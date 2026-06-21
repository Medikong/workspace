# Service HPA Spike payment 150RPS

## Summary

| 항목 | 값 |
| --- | --- |
| service | `payment-service` |
| scenario | `service-hpa-spike-load-test` |
| preset | `payment-150rps` |
| run id | `read-api-loadtest-read-manual-20260621102039-tvll7` |
| k6 status | `PASS` |
| CPU request | `1000m` |
| HPA target | `70%` |
| min / max replicas | `1 / 4` |
| overload target | `150 RPS` |
| report type | `service_hpa_spike` |
| 판단 | `RPS 부족` |

## Conclusion

payment-service는 `1000m`, HPA target `70%`, `150 RPS`에서 HPA가 상승하지 않았다. post-run snapshot은 `horizontalpodautoscaler.autoscaling/payment-service Deployment/payment-service cpu: 42%/70% 1 4 1 70m`로 CPU 70%에 닿지 않았고, stage 결과도 latency/error/checks 한계 후보를 만들지 못했다. 따라서 실패라기보다 현재 RPS가 scale-out 유발 부하가 아니며, 다음 실험에서는 spike/overload RPS 또는 duration을 올려야 한다.

## 특이사항

- `150 RPS`에서는 post-run CPU가 `42%/70%`라 HPA target에 닿지 않았다.
- latency/error/checks는 모두 안정적이어서 실패가 아니라 scale-out 유발 부하 부족으로 판단한다.
- 재시도 preset은 `payment-250rps`로 올렸고 baseline/spike/overload/cooldown duration도 `90s`로 늘렸다.

## HPA Result

| 항목 | 값 |
| --- | ---: |
| baseline replicas | `1` |
| max desired replicas | `1` |
| HPA decision seconds from test start | `-` |
| HPA decision seconds from spike start | `-` |
| scale-out ready seconds from test start | `-` |
| ready after decision | `-` |

Post-run HPA snapshot: `horizontalpodautoscaler.autoscaling/payment-service Deployment/payment-service cpu: 42%/70% 1 4 1 70m`

## Stage Summary

| stage role | target RPS | max p95 ms | max p99 ms | max error | min checks | 해석 |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| warmup | 40 | 6.9 | 11.1 | 0.00% | 100.00% | 판단 제외 / excluded |
| baseline | 80 | 4.8 | 10.1 | 0.00% | 100.00% | 안정 기준 / ok |
| spike | 120 | 5.3 | 13.3 | 0.00% | 100.00% | HPA decision 관찰 / ok |
| overload | 150 | 4.8 | 14.9 | 0.00% | 100.00% | 한계 후보 확인 / ok |
| cooldown | 80 | 5.1 | 14.2 | 0.00% | 100.00% | recovery 관찰 / ok |

## Stage Result

| stage role | step | target RPS | p95 ms | p99 ms | error | checks | status / reasons |
| --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| warmup | payment.create | 40 | 6.9 | 11.1 | 0.00% | 100.00% | excluded |
| baseline | payment.create | 80 | 4.8 | 10.1 | 0.00% | 100.00% | ok |
| spike | payment.create | 120 | 5.3 | 13.3 | 0.00% | 100.00% | ok |
| overload | payment.create | 150 | 4.8 | 14.9 | 0.00% | 100.00% | ok |
| cooldown | payment.create | 80 | 5.1 | 14.2 | 0.00% | 100.00% | ok |

Warmup은 판단에서 제외한다. 원본 보고서의 observed RPS 필드는 현재 `null`이라 이 문서에서는 preset target RPS로 비교했다.

## Limit And Recovery

| 항목 | 값 |
| --- | --- |
| first limit candidate | `none` |
| recovery observations | `payment.create payment_cooldown_rps_80: p95 5.1ms, p99 14.2ms, error 0.00%, checks 100.00%, recovered=true` |
| cooldown quality | `recovered_or_stable` |

## Raw Result

- 원본 JSON: `loadtest-run-report.json`
- 서비스 JSON: `loadtest-run-report-service.json`
- k6 로그: `k6.log`
- HPA/Pod snapshot: `kubectl-get-hpa-A.txt`, `kubectl-get-hpa-pods.txt`
