# Service HPA Spike concert 140RPS

## Summary

| 항목 | 값 |
| --- | --- |
| service | `concert-service` |
| scenario | `service-hpa-spike-load-test` |
| preset | `concert-140rps` |
| run id | `read-api-loadtest-read-manual-20260621094621-hgnfb` |
| k6 status | `PASS` |
| CPU request | `1000m` |
| HPA target | `70%` |
| min / max replicas | `1 / 4` |
| overload target | `140 RPS` |
| report type | `service_hpa_spike` |
| 판단 | `RPS 부족 / baseline 2 재실험 필요` |

## Conclusion

concert-service는 이번 `140 RPS` 실행에서 baseline이 이미 `2` replicas였고 max desired도 `2`였다. post-run snapshot도 `horizontalpodautoscaler.autoscaling/concert-service Deployment/concert-service cpu: 33%/70% 1 4 2 54m`라서 추가 scale-out이 필요할 만큼 CPU 70%를 넘기지 않았다. 따라서 이 실행은 안정성 확인으로는 유효하지만 `1 -> 2` HPA decision 검증은 아니며, 다음에는 시작 전 replica를 1로 안정화한 뒤 재실행해야 한다.

## 특이사항

- 이 run은 시작 시점 baseline이 이미 `2` replicas였으므로 `1 -> 2` HPA 검증으로 쓰면 안 된다.
- `140 RPS`에서 k6는 PASS였고 HPA도 `2 -> 2`로 유지되어, 이번 결과는 안정성 확인에 가깝다.
- 재시도 전 concert-service replica가 `1`로 안정화됐는지 먼저 확인해야 한다.

## HPA Result

| 항목 | 값 |
| --- | ---: |
| baseline replicas | `2` |
| max desired replicas | `2` |
| HPA decision seconds from test start | `-` |
| HPA decision seconds from spike start | `-` |
| scale-out ready seconds from test start | `-` |
| ready after decision | `-` |

Post-run HPA snapshot: `horizontalpodautoscaler.autoscaling/concert-service Deployment/concert-service cpu: 33%/70% 1 4 2 54m`

## Stage Summary

| stage role | target RPS | max p95 ms | max p99 ms | max error | min checks | 해석 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| warmup | 20 | 14.4 | 54.7 | 0.00% | 100.00% | 판단 제외 / excluded |
| baseline | 80 | 9.8 | 32.3 | 0.00% | 100.00% | 안정 기준 / ok |
| spike | 120 | 19.3 | 36.9 | 0.00% | 100.00% | HPA decision 관찰 / ok |
| overload | 140 | 22.5 | 39.6 | 0.00% | 100.00% | 한계 후보 확인 / ok |
| cooldown | 80 | 13.6 | 37.5 | 0.00% | 100.00% | recovery 관찰 / ok |

## Stage Result

| stage role | step | target RPS | p95 ms | p99 ms | error | checks | status / reasons |
| --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| warmup | concert.recommended | 20 | 12.5 | 54.7 | 0.00% | 100.00% | excluded |
| warmup | concert.detail | 20 | 14.3 | 18.6 | 0.00% | 100.00% | excluded |
| warmup | concert.calendar | 20 | 12.2 | 17 | 0.00% | 100.00% | excluded |
| warmup | concert.date_performances | 20 | 10.7 | 12.3 | 0.00% | 100.00% | excluded |
| warmup | concert.seat_map | 20 | 14.4 | 22.7 | 0.00% | 100.00% | excluded |
| baseline | concert.recommended | 80 | 8.4 | 32.3 | 0.00% | 100.00% | ok |
| baseline | concert.detail | 80 | 9.3 | 22.3 | 0.00% | 100.00% | ok |
| baseline | concert.calendar | 80 | 6.3 | 13.6 | 0.00% | 100.00% | ok |
| baseline | concert.date_performances | 80 | 6.8 | 19.1 | 0.00% | 100.00% | ok |
| baseline | concert.seat_map | 80 | 9.8 | 26 | 0.00% | 100.00% | ok |
| spike | concert.recommended | 120 | 8.5 | 33.3 | 0.00% | 100.00% | ok |
| spike | concert.detail | 120 | 9.2 | 24.9 | 0.00% | 100.00% | ok |
| spike | concert.calendar | 120 | 6.5 | 19.3 | 0.00% | 100.00% | ok |
| spike | concert.date_performances | 120 | 8.8 | 22.6 | 0.00% | 100.00% | ok |
| spike | concert.seat_map | 120 | 19.3 | 36.9 | 0.00% | 100.00% | ok |
| overload | concert.recommended | 140 | 11.5 | 34.6 | 0.00% | 100.00% | ok |
| overload | concert.detail | 140 | 11.6 | 32.2 | 0.00% | 100.00% | ok |
| overload | concert.calendar | 140 | 7.1 | 20.3 | 0.00% | 100.00% | ok |
| overload | concert.date_performances | 140 | 13 | 32.6 | 0.00% | 100.00% | ok |
| overload | concert.seat_map | 140 | 22.5 | 39.6 | 0.00% | 100.00% | ok |
| cooldown | concert.recommended | 80 | 9.3 | 34 | 0.00% | 100.00% | ok |
| cooldown | concert.detail | 80 | 12.5 | 33.9 | 0.00% | 100.00% | ok |
| cooldown | concert.calendar | 80 | 6.3 | 19.3 | 0.00% | 100.00% | ok |
| cooldown | concert.date_performances | 80 | 10.5 | 27.6 | 0.00% | 100.00% | ok |
| cooldown | concert.seat_map | 80 | 13.6 | 37.5 | 0.00% | 100.00% | ok |

Warmup은 판단에서 제외한다. 원본 보고서의 observed RPS 필드는 현재 `null`이라 이 문서에서는 preset target RPS로 비교했다.

## Limit And Recovery

| 항목 | 값 |
| --- | --- |
| first limit candidate | `none` |
| recovery observations | `concert.recommended concert_recommended_cooldown_rps_80: p95 9.3ms, p99 34ms, error 0.00%, checks 100.00%, recovered=true`<br>`concert.detail concert_detail_cooldown_rps_80: p95 12.5ms, p99 33.9ms, error 0.00%, checks 100.00%, recovered=true`<br>`concert.calendar concert_calendar_cooldown_rps_80: p95 6.3ms, p99 19.3ms, error 0.00%, checks 100.00%, recovered=true`<br>`concert.date_performances concert_date_performances_cooldown_rps_80: p95 10.5ms, p99 27.6ms, error 0.00%, checks 100.00%, recovered=true`<br>`concert.seat_map concert_seat_map_cooldown_rps_80: p95 13.6ms, p99 37.5ms, error 0.00%, checks 100.00%, recovered=true` |
| cooldown quality | `recovered_or_stable` |

## Raw Result

- 원본 JSON: `loadtest-run-report.json`
- 서비스 JSON: `loadtest-run-report-service.json`
- k6 로그: `k6.log`
- HPA/Pod snapshot: `kubectl-get-hpa-A.txt`, `kubectl-get-hpa-pods.txt`
