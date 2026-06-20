# payment-service API 벤치마크

## 대상 API

- 사용자 API: `POST /payments`, `GET /payments/{paymentId}`
- 공급자 API: `GET /provider/concerts/{concertId}/settlement-basis`
- 관리자 API: `GET /admin/concerts/{concertId}/settlement-basis`

## 비대상

- payment event dispatcher와 Kafka publish loop는 API route가 아니라 background outbox 처리라 제외했다.
- `/health`, `/readyz`, `/metrics`는 운영 endpoint라 서비스 API 처리 시간 대상에서 제외했다.

## smoke 결과

| endpoint | method | status | samples | warmup | min ms | p50 ms | p95 ms | p99 ms | max ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| create-payment | POST | 201 | 2 | 1 | 14.025 | 14.025 | 15.923 | 15.923 | 15.923 |
| get-payment | GET | 200 | 2 | 1 | 8.316 | 8.316 | 14.929 | 14.929 | 14.929 |
| provider-settlement-basis | GET | 200 | 2 | 1 | 7.649 | 7.649 | 13.995 | 13.995 | 13.995 |
| admin-settlement-basis | GET | 200 | 2 | 1 | 9.511 | 9.511 | 13.903 | 13.903 | 13.903 |

## 해석

- 결제 생성은 Payment와 PaymentEvent outbox insert를 같은 트랜잭션으로 처리하므로 조회형 API보다 약간 큰 smoke latency를 보였다.
- 인증 토큰 발급은 측정하지 않고 gateway context header만 사용했다.
- 정산 조회는 승인 결제 seed를 미리 넣고 집계 쿼리만 측정했다.
