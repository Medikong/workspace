# Service HPA Spike Summary 2026-06-21

## Summary

서비스별 `service-hpa-spike-load-test`를 한 번에 하나의 preset만 실행했다. 기준은 모든 서비스 `1000m`, HPA target `70%`, min/max replicas `1/4`다.

| service | preset | k6 | HPA | decision after spike s | ready after decision s | cooldown | 판단 | first limit |
| --- | --- | --- | --- | ---: | ---: | --- | --- | --- |
| auth-service | auth-30rps | FAIL | 1 -> 2 | 28.4 | 11.4 | 회복/안정 | HPA 유효 | auth_spike_rps_25 p99 810.8ms |
| concert-service | concert-140rps | PASS | 2 -> 2 | - | - | 회복/안정 | RPS 부족 / baseline 2 재실험 필요 | none |
| reservation-service | reservation-140rps | PASS | 1 -> 2 | 127.7 | 12.1 | 회복/안정 | HPA 유효 | none |
| payment-service | payment-150rps | PASS | 1 -> 1 | - | - | 회복/안정 | RPS 부족 | none |
| ticket-service | ticket-75rps | PASS | - | - | - | 회복/안정 | RPS 부족 | none |
| notification-service | notification-400rps | FAIL | 1 -> 2 | 58.6 | 11.9 | 회복/안정 | HPA 유효 | notification_spike_rps_320 p99 341.7ms |

## Conclusion

- `auth-service`, `reservation-service`, `notification-service`는 이번 RPS에서 HPA가 상승했고 cooldown 품질도 회복 또는 안정 상태였다. 이 셋은 CPU 기반 HPA가 유효하다.
- `payment-service`, `ticket-service`는 post-run CPU가 각각 42%, 54% 수준이라 `1000m` 기준 CPU 70%를 넘기지 못했다. 현재 RPS는 scale-out 유발 부하가 아니다.
- `concert-service`는 실행 시작 기준 baseline이 이미 2 replicas였고 140 RPS에서도 추가 scale-out이 없었다. 이번 결과는 안정성 확인으로만 쓰고, `1 -> 2` 검증은 replica 안정화 후 재실행해야 한다.
- k6 FAIL은 `auth-30rps`, `notification-400rps`에서 p99 SLO 후보가 발생했기 때문이다. 두 run 모두 error/checks는 0%/100%를 유지했고 cooldown에서 p99가 정상화됐다.

## Next Experiments

| service | 판단 | 다음 조정 |
| --- | --- | --- |
| auth-service | HPA 유효 | 같은 조건에서 duration을 늘려 decision/ready와 p99 회복을 재검증한다. |
| concert-service | RPS 부족 / baseline 2 재실험 필요 | 시작 전 replica를 1로 안정화한 뒤 재실행하고, 필요하면 160~180 RPS로 올린다. |
| reservation-service | HPA 유효 | 같은 조건에서 duration을 늘려 decision/ready와 p99 회복을 재검증한다. |
| payment-service | RPS 부족 | overload RPS 또는 overload duration을 올려 CPU 70% 유지 구간을 만든다. |
| ticket-service | RPS 부족 | overload RPS 또는 overload duration을 올려 CPU 70% 유지 구간을 만든다. |
| notification-service | HPA 유효 | 같은 조건에서 duration을 늘려 decision/ready와 p99 회복을 재검증한다. |

## Raw Result Paths

- `auth-30rps`: `reports/service-hpa-spike-auth-30rps/loadtest-run-report.json`, `reports/service-hpa-spike-auth-30rps/README.md`
- `concert-140rps`: `reports/service-hpa-spike-concert-140rps/loadtest-run-report.json`, `reports/service-hpa-spike-concert-140rps/README.md`
- `reservation-140rps`: `reports/service-hpa-spike-reservation-140rps/loadtest-run-report.json`, `reports/service-hpa-spike-reservation-140rps/README.md`
- `payment-150rps`: `reports/service-hpa-spike-payment-150rps/loadtest-run-report.json`, `reports/service-hpa-spike-payment-150rps/README.md`
- `ticket-75rps`: `reports/service-hpa-spike-ticket-75rps/loadtest-run-report.json`, `reports/service-hpa-spike-ticket-75rps/README.md`
- `notification-400rps`: `reports/service-hpa-spike-notification-400rps/loadtest-run-report.json`, `reports/service-hpa-spike-notification-400rps/README.md`

## 특이사항

- `auth`, `reservation`, `notification`은 HPA 유효로 판단했다.
- `payment`, `ticket`은 RPS 부족으로 판단했고 retry preset을 각각 `payment-250rps`, `ticket-110rps`로 추가했다.
- `concert`는 baseline이 이미 2 replicas였으므로 시작 전 replica 안정화 후 같은 `concert-140rps`로 재실행한다.


# 그다음 수행할 실험

```
SCENARIO=service-hpa-spike-load-test PRESET=concert-140rps task --dir /Users/danghamo/Documents/gituhb/medikong/gitops dev:loadtest
SCENARIO=service-hpa-spike-load-test PRESET=payment-250rps task --dir /Users/danghamo/Documents/gituhb/medikong/gitops dev:loadtest
SCENARIO=service-hpa-spike-load-test PRESET=ticket-110rps task --dir /Users/danghamo/Documents/gituhb/medikong/gitops dev:loadtest
```

# 실험별 시간 구간

| preset | start | end |
| --- | --- | --- |
| `auth-30rps` | `2026-06-21T09:24:03.619Z` | `2026-06-21T09:28:33Z` |
| `concert-140rps` | `2026-06-21T09:46:23.916Z` | `2026-06-21T10:09:54.025Z` |
| `reservation-140rps` | `2026-06-21T10:13:01.542Z` | `2026-06-21T10:17:41.136Z` |
| `payment-250rps` | `2026-06-21T10:20:41.313Z` | `2026-06-21T10:25:20.941Z` |
| `ticket-110rps` | `2026-06-21T10:28:15.069Z` | `2026-06-21T10:32:54.792Z` |
| `notification-400rps` | `2026-06-21T10:35:55.885Z` | `2026-06-21T10:40:36Z` |