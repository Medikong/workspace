# Auth Service PBKDF2 Concurrency RPS Benchmark

## Summary

2026-06-20 기준 auth-service의 현재 기본 password hash 방식인 `pbkdf2_sha256` verify 구간만 분리해서 동시성별 처리량과 latency를 측정했다.

결론은 hash-only 기준으로 동시성 `4`, 약 `50-55 RPS`를 보수적인 상한 후보로 보는 것이다. 이 구간은 p50 `68.07ms`, p95 `83.15ms`, p99 `91.83ms`로 100ms 안에 들어왔다. 동시성 `8`은 처리량이 `74.64 RPS`까지 올라가지만 p95 `158.22ms`, p99 `177.32ms`로 tail latency가 커진다. 동시성 `16`은 처리량이 더 오르지 않고 p99 `386.17ms`까지 악화되어 적정 구간이 아니다.

이 benchmark는 DB user lookup, refresh token insert, audit insert, commit, trace/metric 비용을 제외한 password verify 구간만 측정한다. 실제 `/auth/login` 운영 RPS는 이 수치보다 낮게 잡아야 한다.

## Benchmark Command

```bash
cd service/services/auth-service
AUTH_PBKDF2_CONCURRENCY_BENCHMARK=1 \
AUTH_PBKDF2_BENCHMARK_REQUESTS=120 \
uv run pytest -s tests/test_pbkdf2_verify_concurrency_benchmark.py
```

테스트 파일:

```text
service/services/auth-service/tests/test_pbkdf2_verify_concurrency_benchmark.py
```

기본 동시성:

```text
1,2,4,8,16
```

## Environment

| 항목 | 값 |
| --- | --- |
| date | 2026-06-20 |
| python | `3.13.4` |
| platform | `macOS-26.2-arm64-arm-64bit-Mach-O` |
| scheme | `pbkdf2_sha256` |
| iterations | `210000` |
| requests per concurrency | `120` |

## Results

| concurrency | requests | throughput rps | p50 ms | p95 ms | p99 ms | mean ms | max ms |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 1 | 120 | 21.15 | 47.02 | 48.85 | 50.88 | 47.26 | 52.62 |
| 2 | 120 | 34.87 | 56.01 | 68.53 | 85.97 | 57.18 | 86.70 |
| 4 | 120 | 57.44 | 68.07 | 83.15 | 91.83 | 69.36 | 95.95 |
| 8 | 120 | 74.64 | 104.37 | 158.22 | 177.32 | 105.49 | 193.43 |
| 16 | 120 | 71.49 | 218.03 | 323.92 | 386.17 | 217.09 | 397.33 |

## Reading

| 관점 | 해석 |
| --- | --- |
| p50 | 동시성 `4`까지는 p50이 `68.07ms`로 유지된다. 동시성 `8`부터 `104.37ms`로 100ms를 넘는다. |
| p95 | 동시성 `4`까지는 p95가 `83.15ms`다. 동시성 `8`은 p95 `158.22ms`로 tail latency가 뚜렷하게 커진다. |
| p99 | 동시성 `4`는 p99 `91.83ms`다. 동시성 `8`은 `177.32ms`, 동시성 `16`은 `386.17ms`로 급격히 나빠진다. |
| throughput | 최대 처리량은 동시성 `8`의 `74.64 RPS`지만 latency 기준을 함께 보면 적정 운영 후보로 보기 어렵다. |
| saturation | 동시성 `16`은 처리량이 `71.49 RPS`로 줄고 latency만 커져서 이미 포화 구간으로 본다. |

## RPS Recommendation

| 기준 | 후보 RPS | 판단 |
| --- | ---: | --- |
| p95 `< 100ms` hash-only | `50-55 RPS` | 동시성 `4` 결과를 기준으로 한 보수적 상한 |
| p99 `< 100ms` hash-only | `50-55 RPS` | 동시성 `4`의 p99 `91.83ms`까지만 통과 |
| 최대 처리량 우선 | `70-75 RPS` | 동시성 `8`에서 가능하지만 p95/p99가 커져 로그인 UX 기준으로는 위험 |
| 실제 `/auth/login` 운영 후보 | `30-40 RPS`부터 재검증 | DB write, audit, token 발급, trace/metric 비용을 포함하면 hash-only보다 낮게 잡아야 함 |

현재 auth-service의 capacity 기준은 password verify만으로 정하지 않는다. 이 benchmark는 password verification 구간의 상한을 보는 자료다. 실제 운영 후보는 `/auth/login` API 전체를 같은 RPS ladder로 재실행해서 정해야 한다.

## Next Validation

1. `/auth/login` 전체 API로 `10 -> 20 -> 30 -> 40 -> 50 RPS` ladder를 실행한다.
2. `auth.password.verify` span p50/p95/p99와 `/auth/login` p50/p95/p99를 같이 비교한다.
3. DB connection pool, audit insert, refresh token insert가 p95/p99에 더하는 비용을 분리한다.
4. 운영 기본값은 p95, p99, error rate, readiness 흔들림이 모두 통과한 마지막 RPS로 잡는다.

## Verification

| 명령 | 결과 |
| --- | --- |
| `AUTH_PBKDF2_CONCURRENCY_BENCHMARK=1 AUTH_PBKDF2_BENCHMARK_REQUESTS=120 uv run pytest -s tests/test_pbkdf2_verify_concurrency_benchmark.py` | `1 passed` |
