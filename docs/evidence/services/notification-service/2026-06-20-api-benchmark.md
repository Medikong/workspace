# notification-service API 벤치마크

## 대상 API

- 사용자 API: `GET /notifications`, `GET /notifications/{notification_id}`

## 비대상

- Kafka consumer와 business event handler는 API route가 아니라 background/event 처리 경로라 제외했다.
- `/health`, `/readyz`, `/metrics`는 운영 endpoint라 서비스 API 처리 시간 대상에서 제외했다.

## smoke 결과

| endpoint | method | status | samples | warmup | min ms | p50 ms | p95 ms | p99 ms | max ms |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| list-notifications | GET | 200 | 2 | 1 | 5.501 | 5.501 | 6.996 | 6.996 | 6.996 |
| get-notification | GET | 200 | 2 | 1 | 2.696 | 2.696 | 5.106 | 5.106 | 5.106 |

## 해석

- 알림 서비스는 MongoDB 비동기 경로라 `httpx.AsyncClient`와 ASGI transport로 같은 이벤트 루프에서 측정했다.
- 목록 조회는 `user_id`, `_id` index를 생성한 뒤 첫 페이지 전체 목록 경로를 측정했다.
- business event 생성 비용은 API 조회 시간이 아니므로 포함하지 않았다.
