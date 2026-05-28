# 공통 OpenAPI 규약 샘플

이 문서는 팀 논의 전 샘플이다. 확정 전까지는 구현 계약이 아니라 API 문서 작성 방향을 맞추기 위한 기준으로만 사용한다.

## 기본 규칙

- OpenAPI 버전은 `3.1.0`을 기본 샘플로 둔다. 팀 도구가 문제없이 처리하면 `3.1.1`도 허용 후보로 둔다.
- 요청과 응답의 기본 `Content-Type`은 `application/json`이다.
- 날짜와 시간은 ISO-8601 형식의 `string` + `format: date-time`을 사용한다. 서버 저장과 이벤트 시간은 UTC 기준을 우선 검토한다.
- ID는 숫자 타입 대신 `string`으로 정의한다. UUID, ULID, 짧은 도메인 ID 중 무엇을 쓸지는 서비스 구현 전 별도 결정한다.
- 인증은 `Authorization: Bearer <JWT>`를 기본으로 한다.
- trace header는 현재 구현 관례와 맞춰 `X-Request-Id`를 우선 사용한다. 외부 tracing 도구와 맞출 필요가 있으면 `X-Trace-Id`를 후보로 둔다.
- 중복 요청 방지가 필요한 생성/변경 API는 `Idempotency-Key` 헤더를 받는다.
- 목록 조회 페이지네이션은 `limit`, `cursor` 쿼리 파라미터를 사용한다.
- 오류 응답은 공통 `ErrorResponse` 스키마를 사용한다.

## Status Code 규칙

- `200 OK`: 조회, 취소/만료 같은 동기 명령이 기존 리소스 상태를 반환할 때 사용한다.
- `201 Created`: 예약 생성처럼 새 리소스가 만들어졌을 때 사용한다.
- `202 Accepted`: 비동기 처리로 넘긴 명령을 접수했지만 아직 완료되지 않았을 때 사용한다.
- `204 No Content`: 성공했지만 반환할 본문이 없을 때 사용한다.
- `400 Bad Request`: 요청 JSON, 쿼리, path parameter 형식이 잘못됐을 때 사용한다.
- `401 Unauthorized`: JWT가 없거나 유효하지 않을 때 사용한다.
- `403 Forbidden`: 인증은 됐지만 해당 리소스나 명령 권한이 없을 때 사용한다.
- `404 Not Found`: 리소스를 찾을 수 없을 때 사용한다.
- `409 Conflict`: 좌석 중복 선점, 이미 취소된 예약 취소 요청처럼 현재 상태와 충돌할 때 사용한다.
- `422 Unprocessable Entity`: 형식은 맞지만 도메인 규칙상 처리할 수 없을 때 사용한다.
- `500 Internal Server Error`: 예측하지 못한 서버 오류에 사용한다.

## ErrorResponse 샘플

```json
{
  "error": {
    "code": "reservation.conflict",
    "message": "Seat is already reserved.",
    "details": {
      "seatId": "seat-001"
    }
  },
  "requestId": "req-01HV6W8ZK2J2J9N9S4V7T3F0CA",
  "occurredAt": "2026-05-28T10:15:30Z"
}
```

`code`는 사람이 읽는 메시지보다 안정적인 식별자 역할을 한다. 클라이언트 분기와 테스트는 `message`가 아니라 `code` 기준으로 작성한다.
