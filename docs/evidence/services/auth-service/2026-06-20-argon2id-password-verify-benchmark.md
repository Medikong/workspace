# Auth Service Argon2id Password Verify Benchmark And Decision

## Summary

2026-06-20 기준 auth-service에 Argon2id password hash 구현과 verifier를 추가하고, 기존 PBKDF2-SHA256 방식과 성능을 비교했다. 검토 결과 Argon2id는 구현은 유지하되 신규 password hash 기본값으로 채택하지 않는다. auth-service의 기본 저장 방식은 기존 `pbkdf2_sha256`으로 되돌린다.

FastAPI 공식 문서는 `pwdlib[argon2]`와 `PasswordHash.recommended()` 기반 Argon2 사용을 안내하고, RFC 9106은 Argon2id 프로파일을 권장한다. 다만 Medikong은 티켓팅 특성상 특정 시점에 로그인과 재로그인이 몰릴 수 있다. Argon2id 64MiB 프로파일은 verify 1건마다 약 64MiB 작업 메모리를 쓰므로, 동시 로그인 수백 건이 들어오는 상황에서는 컨테이너 메모리와 node allocatable memory를 빠르게 압박한다.

따라서 이번 결정은 `Argon2id 지원 코드는 보존`, `기본 신규 hash는 PBKDF2 유지`, `향후 rate limit, backoff, verify 동시성 제한, 운영 부하테스트가 준비된 뒤 재검토`다.

## Code Surface

| 항목 | 위치 | 결정 |
| --- | --- | --- |
| 기본 hash 생성 | `service/services/auth-service/app/security.py` | `hash_password()`가 `pbkdf2_sha256` hash를 생성 |
| Argon2id hash 생성 | `service/services/auth-service/app/security.py` | `hash_password_argon2id()`로 명시 호출할 때만 Argon2id PHC hash를 생성 |
| legacy PBKDF2 생성 | `service/services/auth-service/app/security.py` | `hash_password_legacy_pbkdf2()`로 분리 |
| 통합 검증 | `service/services/auth-service/app/security.py` | hash scheme을 식별한 뒤 Argon2id 또는 PBKDF2 verifier로 분기 |
| 로그인 trace | `service/services/auth-service/app/main.py` | scheme과 비용 파라미터만 남기고 password/hash/token 값은 남기지 않음 |
| 의존성 | `service/services/auth-service/pyproject.toml` | `pwdlib[argon2]==0.3.0` 추가 |

## Hash Settings

| 구분 | scheme | 설정 |
| --- | --- | --- |
| current default | `pbkdf2_sha256` | `AUTH_PASSWORD_ITERATIONS=210000`, SHA-256, 16-byte salt |
| supported candidate | `argon2id` | memory `65536 KiB`, time cost `3`, parallelism `4`, hash length `32`, salt length `16` |
| RFC 9106 high-memory | `argon2id` | memory `2097152 KiB`, time cost `1`, parallelism `4` |

RFC 9106은 2GiB 프로파일을 첫 번째 권장안으로 제시하지만, auth-service의 로그인 경로에 그대로 적용하면 verify 1건마다 약 2GiB 작업 메모리가 필요하다. 64MiB 프로파일도 동시 로그인 수가 커지면 컨테이너 메모리 산정에 직접 영향을 준다. 그래서 이번 변경에서는 Argon2id를 기본 저장 방식으로 채택하지 않는다.

## Benchmark

실행 명령:

```bash
cd service/services/auth-service
AUTH_PASSWORD_BENCHMARK_SAMPLES=9 uv run pytest -s tests/test_password_verify_benchmark.py
```

실행 환경:

| 항목 | 값 |
| --- | --- |
| date | 2026-06-20 |
| python | `3.13.4` |
| platform | `macOS-26.2-arm64-arm-64bit-Mach-O` |
| pytest | `9.0.2` |
| samples | `9` |

결과:

| scheme | mean ms | median ms | min ms | max ms |
| --- | ---: | ---: | ---: | ---: |
| `pbkdf2_sha256` | 48.206 | 48.082 | 47.616 | 49.394 |
| `argon2id` | 34.606 | 32.191 | 31.115 | 47.294 |

이번 로컬 단건 verify 기준으로는 Argon2id 64MiB 프로파일이 기존 PBKDF2 210000회보다 낮은 평균 지연시간을 보였다. 다만 Argon2id는 CPU만 쓰는 PBKDF2와 달리 메모리 비용이 핵심이다. 운영 판단은 단건 평균만 보지 말고, 동시 로그인 수, Pod memory limit, replica 수, 실패 로그인 rate limit을 함께 봐야 한다.

## Quick Stress Check

추가로 DB, token 발급, audit insert를 제외하고 password verify 함수만 동시성별로 눌러 보았다. 이 값은 `/auth/login` 전체 처리량이 아니라 password verification 구간만의 로컬 참고치다.

실행 방식:

```bash
cd service/services/auth-service
uv run python - <<'PY'
from concurrent.futures import ThreadPoolExecutor, as_completed
import statistics
import time
from app.security import hash_password_argon2id, hash_password_legacy_pbkdf2, verify_password

PASSWORD = "benchmark-password-1234"
SAMPLES_BY_CONCURRENCY = {1: 20, 2: 24, 4: 32, 8: 40}
HASHES = {
    "pbkdf2_sha256": hash_password_legacy_pbkdf2(PASSWORD),
    "argon2id": hash_password_argon2id(PASSWORD),
}

def verify_once(password_hash: str) -> float:
    started = time.perf_counter_ns()
    ok = verify_password(PASSWORD, password_hash)
    elapsed_ms = (time.perf_counter_ns() - started) / 1_000_000
    if not ok:
        raise RuntimeError("verification failed")
    return elapsed_ms

for scheme, password_hash in HASHES.items():
    for concurrency, total in SAMPLES_BY_CONCURRENCY.items():
        started = time.perf_counter()
        durations = []
        with ThreadPoolExecutor(max_workers=concurrency) as pool:
            futures = [pool.submit(verify_once, password_hash) for _ in range(total)]
            for future in as_completed(futures):
                durations.append(future.result())
        elapsed_s = time.perf_counter() - started
        print(
            scheme,
            concurrency,
            round(total / elapsed_s, 2),
            round(statistics.fmean(durations), 2),
            round(statistics.quantiles(durations, n=20)[18], 2),
            round(max(durations), 2),
        )
PY
```

결과:

| scheme | concurrency | requests | throughput rps | mean ms | median ms | p95 ms | max ms |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `pbkdf2_sha256` | 1 | 20 | 20.26 | 49.32 | 48.64 | 54.65 | 54.79 |
| `pbkdf2_sha256` | 2 | 24 | 34.18 | 58.39 | 53.29 | 77.48 | 77.57 |
| `pbkdf2_sha256` | 4 | 32 | 54.19 | 71.08 | 71.46 | 85.10 | 86.16 |
| `pbkdf2_sha256` | 8 | 40 | 43.50 | 178.52 | 152.71 | 347.08 | 360.38 |
| `argon2id` | 1 | 20 | 29.11 | 34.31 | 33.91 | 40.77 | 41.07 |
| `argon2id` | 2 | 24 | 44.26 | 44.94 | 43.00 | 55.74 | 55.88 |
| `argon2id` | 4 | 32 | 34.97 | 112.49 | 115.13 | 163.70 | 166.61 |
| `argon2id` | 8 | 40 | 44.63 | 172.25 | 171.95 | 252.37 | 254.64 |

해석:

- Argon2id는 동시성 `1-2`에서는 기존 PBKDF2보다 빠르게 나왔다.
- 동시성 `4`부터는 64MiB 메모리 비용이 겹치면서 평균 latency가 `112.49ms`까지 올라갔다.
- 동시성 `8`에서는 두 방식 모두 tail latency가 커졌고, Argon2id는 p95 `252.37ms`, PBKDF2는 p95 `347.08ms`였다.
- 실제 `/auth/login` stress test에서는 여기에 DB user lookup, refresh token insert, audit insert, commit, trace/metric 비용이 더해진다.

## Operational Reading

| 선택지 | 판단 | 이유 |
| --- | --- | --- |
| PBKDF2 유지 | 선택 | 티켓팅 피크 시간대에 동시 로그인 수가 커질 수 있고, 현재 운영 보호장치 없이 Argon2id 64MiB를 기본값으로 두면 컨테이너 메모리 부담이 커짐 |
| Argon2id 2GiB | 운영 기본값 제외 | verify 1건당 메모리 비용이 커서 컨테이너와 CI 환경에서 동시 처리 여유가 급격히 줄어듦 |
| Argon2id 64MiB | 구현 유지, 기본값 제외 | RFC 9106 low-memory 권장안이고 단건 지연시간은 좋지만, 동시 verify 수백 건에서 메모리 비용이 수 GiB 단위로 커짐 |

## Decision Record

결정: auth-service의 기본 password hash 생성 방식은 `pbkdf2_sha256`으로 유지한다. Argon2id 구현, verifier, benchmark는 남겨서 이후 재검토할 수 있게 한다.

사유:

- 티켓팅 서비스는 이벤트 오픈, 세션 만료, 재시도 상황에서 로그인 요청이 짧은 시간에 몰릴 수 있다.
- Argon2id 64MiB 프로파일은 verify 1건당 약 64MiB 작업 메모리를 사용한다.
- 동시 verify 100건은 이론상 약 6.4GiB, 200건은 약 12.8GiB 작업 메모리 압박으로 이어질 수 있다.
- 현재 auth-service에는 password verify 동시성 제한, 로그인 전용 rate limit, 실패 backoff가 아직 없다.
- 따라서 Argon2id를 기본값으로 켜면 보안 강도는 좋아지지만, 피크 로그인 상황에서 auth-service 안정성을 먼저 해칠 수 있다.

재검토 조건:

- IP, account, IP+account 기준 로그인 rate limit을 적용한다.
- 실패 로그인 backoff 또는 일시 잠금을 적용한다.
- Pod 내부 password verify 동시성 제한을 둔다.
- auth-login 전용 부하테스트에서 memory working set, OOMKilled, restart, p95/p99를 함께 본다.
- 위 조건을 만족한 뒤 Argon2id memory cost를 낮은 값부터 다시 실험한다.

## Compatibility And Migration

기존 사용자 hash는 `pbkdf2_sha256$...` 포맷으로 식별해 PBKDF2 verifier에서 검증한다. 신규 가입과 demo seed도 현재는 `hash_password()`를 통해 PBKDF2 hash를 저장한다. Argon2id hash가 이미 저장된 사용자가 생긴 경우에도 verifier는 `$argon2id$...` PHC 포맷을 인식해 검증할 수 있다.

PBKDF2 hash로 로그인 성공 시 즉시 Argon2id로 재해시하는 방식은 이번 변경에 넣지 않았다. 로그인 경로에서 password hash를 갱신하려면 같은 요청 안에서 refresh token 발급, audit insert, commit 순서와 실패 복구 범위를 다시 정해야 한다. 이번 단계에서는 읽기 호환성을 먼저 보장하고, 재해시는 별도 마이그레이션 작업으로 분리하는 편이 부작용이 작다.

마이그레이션 전략:

1. 신규 hash는 현재 PBKDF2로 저장한다.
2. Argon2id hash가 저장되어 있어도 로그인 시 계속 허용한다.
3. unknown hash scheme은 silent fallback 없이 명시적으로 실패시킨다.
4. Argon2id 재도입 또는 재해시를 도입할 때는 로그인 성공 후 같은 DB transaction 안에서 hash update 여부와 audit/token 발급 순서를 별도 테스트한다.

## Verification

```bash
cd service/services/auth-service
uv run pytest tests/test_auth.py
AUTH_PASSWORD_BENCHMARK_SAMPLES=9 uv run pytest -s tests/test_password_verify_benchmark.py
```

검증 결과:

| 명령 | 결과 |
| --- | --- |
| `uv run pytest tests/test_auth.py` | `12 passed`, warnings `3` |
| `AUTH_PASSWORD_BENCHMARK_SAMPLES=9 uv run pytest -s tests/test_password_verify_benchmark.py` | `1 passed` |
| `uv run pytest` | `13 passed`, warnings `3` |

## References

- [FastAPI OAuth2 with Password, Bearer with JWT tokens](https://fastapi.tiangolo.com/tutorial/security/oauth2-jwt/)
- [RFC 9106: Argon2 Memory-Hard Function for Password Hashing and Proof-of-Work Applications](https://datatracker.ietf.org/doc/rfc9106/)
