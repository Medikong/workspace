---
id: TROUBLE-019
title: "HPA scale-out 후 DB connection budget 초과가 다시 병목이 된 문제"
status: open
priority: p1
severity: high
area: service
repos:
  - service
  - gitops
  - workspace
owner: unassigned
created: 2026-06-21
updated: 2026-06-21
resolved: null
tags:
  - loadtest
  - hpa
  - db-pool
  - sqlalchemy
  - concert-service
  - connection-budget
related:
  - TROUBLE-015
  - TROUBLE-018
  - workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/README.md
  - workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/root-cause-analysis.md
links: []
---

# HPA scale-out 후 DB connection budget 초과가 다시 병목이 된 문제

## 문제

`service-hpa-spike-load-test`를 `concert-140rps` 프리셋으로 재실행한 결과, HPA 자체는 정상적으로 동작했다. `concert-service`는 baseline `2` replicas에서 최대 desired `4` replicas까지 올라갔고, scale-out ready까지 약 `227.8s`가 걸렸다.

하지만 k6 threshold는 실패했다. 직접 원인은 HPA 미동작이 아니라 `concert-db` PostgreSQL connection 한계 초과였다. HPA를 늘리는 방향은 맞았지만, 애플리케이션이 DB로 열 수 있는 connection 수가 HPA replica 수와 함께 커지면서 DB pool 문제가 다시 나타났다.

## 증상

- `concert-service` HPA가 `2 -> 4`로 scale-out 했다.
- `concert-140rps`의 overload 구간에서 p95와 p99가 크게 상승했다.
- `/concerts/{id}/calendar` route에서 500 응답이 집중됐다.
- Tempo trace의 DB `connect` span에서 `FATAL: sorry, too many clients already`가 확인됐다.
- 후보 trace `16 / 16`개가 같은 DB connection exhaustion 메시지를 포함했다.

## 확인한 사실

| 항목 | 값 |
| --- | --- |
| loadtest job | `read-api-loadtest-read-manual-20260621121910` |
| scenario | `service-hpa-spike-load-test` |
| preset | `concert-140rps` |
| HPA 결과 | `2 -> 4` scale-out |
| HPA decision seconds | `215.764s` |
| scale-out ready seconds | `227.820s` |
| 대표 실패 route | `/concerts/{id}/calendar` |
| 대표 trace id | `36688b93cf9c569a123884d0a5d9cab1` |
| trace 결론 | DB `connect` span에서 `too many clients already` |

현재 local HPA spike 설정은 다음 connection budget을 만든다.

| 설정 | 값 |
| --- | ---: |
| `SQLALCHEMY_POOL_SIZE` | `35` |
| `SQLALCHEMY_MAX_OVERFLOW` | `10` |
| `UVICORN_WORKERS` | `2` |
| HPA max replicas | `4` |
| `concert-db max_connections` | `200` |

SQLAlchemy engine은 worker process 안에서 생성된다. 따라서 Pod당 connection 상한은 `35 + 10 = 45`가 아니라 `2 * (35 + 10) = 90`이다. HPA가 4 replicas까지 올라가면 애플리케이션 쪽 이론상 최대 connection 수는 `360`이고, 이는 `concert-db max_connections=200`을 넘는다.

## 판단

이번 문제는 이전 `TROUBLE-018`과 성격이 다르다. `TROUBLE-018`은 API process와 background worker 실행 단위가 섞여 capacity-baseline 병목을 키운 문제였고, `cmd/server`와 `cmd/worker` 분리로 해결했다.

이번 문제는 실행 단위 분리 이후에도 남은 HPA 환경의 connection budget 문제다. HPA가 정상 동작하면 Pod 수가 늘고, Pod 수가 늘면 worker process 수와 SQLAlchemy pool 총량도 같이 늘어난다. DB connection budget을 HPA 설계 제약으로 넣지 않으면 scale-out이 처리량 증가가 아니라 DB 접속 실패 증가로 이어질 수 있다.

즉, 해결 방향은 DB pool을 계속 키우는 것이 아니다. 서비스별로 `maxReplicas * UVICORN_WORKERS * (poolSize + maxOverflow)`가 DB connection budget 안에 들어오도록 제한해야 한다.

## aws-dev 영향

aws-dev HPA 실험은 이 문제가 해결되기 전에는 실행하지 않는다. 기존 aws-dev service override에 `poolSize=70`, `maxOverflow=20`, HPA max `10`, `UVICORN_WORKERS=2`가 같이 적용되면 서비스별 API Pod만으로도 이론상 `10 * 2 * (70 + 20) = 1800` connection까지 열 수 있다. 이는 aws-dev PostgreSQL values의 `max_connections=300`을 크게 넘는다.

따라서 aws-dev 1차 배포 기준은 `poolSize=8`, `maxOverflow=0`으로 둔다. 이 값이면 API-only 서비스는 최대 `160`, background worker 1개가 있는 서비스는 최대 `168` connection으로 제한된다.

| service | HPA max | API workers | worker replicas | pool | max overflow | worst-case app connections | DB max connections |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| auth | 10 | 2 | 0 | 8 | 0 | 160 | 300 |
| concert | 10 | 2 | 0 | 8 | 0 | 160 | 300 |
| reservation | 10 | 2 | 1 | 8 | 0 | 168 | 300 |
| payment | 10 | 2 | 1 | 8 | 0 | 168 | 300 |
| ticket | 10 | 2 | 1 | 8 | 0 | 168 | 300 |

이 조치는 HPA 실험을 통과시키기 위한 임시 우회가 아니다. aws-dev에서 HPA를 기본 개발환경 정책으로 켜려면 DB connection budget도 같은 배포 기준에 포함되어야 한다.

## 남은 조사

애플리케이션 레벨에서도 connection 사용량이 불필요하게 커지는지 확인해야 한다.

| 상태 | 조사 항목 | 담당 | 링크 |
| --- | --- | --- | --- |
| done | 요청 하나당 DB session 하나만 쓰는지 확인 | service | `services/concert-service/app/database.py` |
| done | repository/service 계층에서 session을 중복 생성하는 경로가 있는지 확인 | service | `services/concert-service/app/services/base.py`, `services/concert-service/app/repositories` |
| done | `/concerts/{id}/calendar`에서 DB round-trip 수와 session 범위를 확인 | service | `services/concert-service/app/services/catalog.py`, `services/concert-service/app/repositories/concerts.py`, `services/concert-service/app/repositories/showtimes.py` |
| todo | SQLAlchemy pool checkout wait, checked_out, overflow, timeout metric 추가 | service/gitops |  |
| todo | PostgreSQL exporter의 `pg_stat_activity_count`가 대시보드에 채워지도록 관측 경로 보강 | gitops |  |

## 애플리케이션 session 조사 결과

`concert-service` 코드 기준으로는 요청 하나가 DB session을 여러 개 만드는 구조는 아니다.

- `app/database.py`의 `get_db()`만 `SessionLocal()`을 호출한다.
- router dependency는 `catalog_service`, `showtime_service`, `seat_service` 등 service factory에서 `Depends(get_db)`로 받은 같은 `Session`을 service 생성자에 넘긴다.
- `ConcertDomainService`는 전달받은 `Session` 하나를 보관하고, 각 repository도 같은 `Session` 객체를 받는다.
- `app/repositories/*`와 `app/services/*` 안에서 `SessionLocal()`을 다시 호출하는 경로는 없다.
- `Depends(..., use_cache=False)`처럼 같은 요청 안에서 `get_db()` 캐시를 끄는 패턴도 없다.

대표 실패 trace `36688b93cf9c569a123884d0a5d9cab1`에서도 `concert.dependency.db.session_create`는 1회, `concert.dependency.db.session_close`는 1회만 확인됐다. 따라서 이번 `too many clients already`를 "요청 하나가 session을 중복 생성해서 생긴 문제"로 보기는 어렵다.

다만 session 하나가 곧 connection 하나를 즉시 의미하지는 않는다. SQLAlchemy `SessionLocal()`은 session 객체를 만들고, 실제 DB connection checkout은 첫 query 시점에 일어난다. 이번 trace에서는 `/concerts/{id}/calendar`가 `has_concert()`의 첫 DB query에서 connection을 얻으려다 실패했다.

`/concerts/{id}/calendar`의 정상 경로는 현재 최소 2개 DB query를 수행한다.

| 순서 | 코드 경로 | 역할 |
| --- | --- | --- |
| 1 | `ConcertRepository.has_concert()` | concert 존재 확인 |
| 2 | `ShowtimeRepository.list_bookable_showtime_starts_between()` | 해당 월의 bookable showtime 날짜 조회 |

따라서 현재 애플리케이션 레벨의 1차 문제는 session 중복 생성이 아니라, 많은 동시 요청이 각자 하나의 session/connection을 정상적으로 요구하고, 그 총량이 HPA scale-out 후 DB connection budget을 넘는 구조다.

## 대응 방향

| 우선순위 | 조치 | 이유 |
| --- | --- | --- |
| 1 | 환경별 SQLAlchemy pool 예산을 전체 서비스 values에 반영한다 | HPA replicas와 worker Deployment까지 포함해 DB connection budget을 넘지 않게 한다 |
| 2 | `maxOverflow`를 거의 0에 가깝게 제한한다 | overflow가 순간 부하 흡수가 아니라 DB 접속 폭증으로 작동한다 |
| 3 | GitOps 또는 CI에서 connection budget 계산을 검증한다 | 위험한 HPA/pool 조합이 배포 전에 드러나야 한다 |
| 4 | endpoint별 DB 진입 동시성 제한을 검토한다 | `/calendar` 같은 집중 route가 DB를 한 번에 밀어붙이지 않게 한다 |
| 5 | PgBouncer 도입을 검토한다 | 앱 client connection 수와 PostgreSQL backend connection 수를 분리한다 |

## GitOps 반영

2026-06-21 기준으로 local/dev 계열과 aws-dev의 SQLAlchemy pool 값을 connection budget 안으로 낮췄다. `maxOverflow`는 모든 대상에서 `0`으로 제한했다. overflow는 순간 부하를 흡수하기보다 HPA 환경에서 DB connection 폭증으로 이어질 수 있기 때문이다.

예산 계산은 API Deployment와 worker Deployment를 분리해서 본다.

```text
api_connection_budget = hpa_max_replicas * uvicorn_workers * (poolSize + maxOverflow)
worker_connection_budget = worker_replicas * (poolSize + maxOverflow)
service_connection_budget = api_connection_budget + worker_connection_budget
```

| 환경 | DB max_connections | 적용값 | 최대 API-only 서비스 | 최대 worker 포함 서비스 |
| --- | ---: | --- | ---: | ---: |
| dev/local non-HPA | `200` | `poolSize=15`, `maxOverflow=0` | `60` | `75` |
| local-hpa-spike | `200` | `poolSize=15`, `maxOverflow=0` | `120` | `135` |
| aws-dev | `300` | `poolSize=10`, `maxOverflow=0` | `200` | `210` |

반영 파일:

- `gitops/values/env/dev.yaml`
- `gitops/values/overrides/local-hpa-spike/auth.yaml`
- `gitops/values/overrides/local-hpa-spike/concert.yaml`
- `gitops/values/overrides/local-hpa-spike/notification.yaml`
- `gitops/values/overrides/local-hpa-spike/reservation.yaml`
- `gitops/values/overrides/local-hpa-spike/payment.yaml`
- `gitops/values/overrides/local-hpa-spike/ticket.yaml`
- `gitops/values/services/aws-dev/auth.yaml`
- `gitops/values/services/aws-dev/concert.yaml`
- `gitops/values/services/aws-dev/notification.yaml`
- `gitops/values/services/aws-dev/reservation.yaml`
- `gitops/values/services/aws-dev/payment.yaml`
- `gitops/values/services/aws-dev/ticket.yaml`

## 증거

- `workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/README.md`
- `workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/root-cause-analysis.md`
- `workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/root-cause-evidence/tempo-trace-summary.json`
- `workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/root-cause-evidence/tempo-traces/36688b93cf9c569a123884d0a5d9cab1.json`
- `workspace/docs/evidence/loadtest/hpa-spike-test/reports/service-hpa-spike-concert-140rps/root-cause-evidence/runtime-db-settings.txt`

## 결론

HPA scale-out 자체는 유효했다. 문제는 HPA가 늘린 애플리케이션 동시성이 DB connection budget 안에서 통제되지 않았다는 점이다.

앞으로 HPA max replicas, Uvicorn worker 수, SQLAlchemy pool size, max overflow, DB max connections는 따로 조정하지 않는다. 하나의 connection budget으로 계산하고, 그 예산 안에서만 scale-out을 허용한다.
