# reservation-service API 벤치마크

## 대상 API

- 사용자 API: `POST /reservations`, `GET /reservations/me`, `GET /reservations/{id}`, `POST /reservations/{id}/cancel`, `POST /reservations/{id}/expire`
- 관리자 판매 API: `POST /admin/concerts/{concertId}/sales/start`, `pause`, `resume`, `GET /admin/concerts/{concertId}/sales`
- 공급자 판매 API: `GET /provider/concerts/{concertId}/sales`, `GET /provider/showtimes/{showtimeId}/sales`
- 관리자 정책 API: `PUT /admin/concerts/{concertId}/queue-policy`, `PUT /admin/concerts/{concertId}/traffic-policy`

## 비대상

- `POST /admin/.../queue-policy`, `POST /admin/.../traffic-policy`는 OpenAPI에서 숨긴 호환용 alias라 canonical `PUT`만 측정했다.
- `/health`, `/readyz`, `/metrics`는 운영 endpoint라 서비스 API 처리 시간 대상에서 제외했다.

## smoke 결과

| endpoint | method | status | samples | warmup | min ms | p50 ms | p95 ms | p99 ms | max ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| create-reservation | POST | 201 | 2 | 1 | 6.314 | 6.314 | 9.056 | 9.056 | 9.056 |
| list-my-reservations | GET | 200 | 2 | 1 | 3.602 | 3.602 | 4.229 | 4.229 | 4.229 |
| get-reservation | GET | 200 | 2 | 1 | 3.307 | 3.307 | 3.708 | 3.708 | 3.708 |
| cancel-reservation | POST | 200 | 2 | 1 | 5.812 | 5.812 | 8.106 | 8.106 | 8.106 |
| expire-reservation | POST | 200 | 2 | 1 | 4.765 | 4.765 | 12.938 | 12.938 | 12.938 |
| admin-start-sales | POST | 200 | 2 | 1 | 4.995 | 4.995 | 7.207 | 7.207 | 7.207 |
| admin-pause-sales | POST | 200 | 2 | 1 | 4.372 | 4.372 | 5.748 | 5.748 | 5.748 |
| admin-resume-sales | POST | 200 | 2 | 1 | 6.561 | 6.561 | 6.644 | 6.644 | 6.644 |
| admin-get-sales | GET | 200 | 2 | 1 | 4.972 | 4.972 | 6.688 | 6.688 | 6.688 |
| provider-concert-sales | GET | 200 | 2 | 1 | 5.327 | 5.327 | 5.429 | 5.429 | 5.429 |
| provider-showtime-sales | GET | 200 | 2 | 1 | 2.713 | 2.713 | 2.794 | 2.794 | 2.794 |
| admin-queue-policy | PUT | 200 | 2 | 1 | 5.295 | 5.295 | 6.827 | 6.827 | 6.827 |
| admin-traffic-policy | PUT | 200 | 2 | 1 | 4.520 | 4.520 | 5.101 | 5.101 | 5.101 |

## 해석

- smoke 표본에서는 만료 처리의 p99가 가장 컸지만, sample 2라 기준값으로 보기는 어렵다.
- 생성/취소/만료는 매 iteration마다 다른 예약 ID나 좌석 ID를 사용해 중복 충돌을 측정에 섞지 않았다.
- Kafka producer는 dependency override로 `None` 처리해 외부 브로커 비용은 제외했다.
