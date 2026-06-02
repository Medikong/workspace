# 감사 로그 아키텍처

감사 로그는 시스템 관측성과 분리한다.

시스템 관측성은 장애 탐지와 처리 과정 확인을 위한 영역이고, 감사 로그는 운영자/CS가 고객 문의와 업무 이력을 확인하기 위한 영역이다.

## 목적

감사 로그는 "어떤 고객 또는 업무 객체에 어떤 일이 있었는가"를 찾기 위한 기록이다.

예매 시스템에서는 다음 질문에 답해야 한다.

```text
특정 사용자가 어떤 예매를 시도했는가?
좌석 선점은 언제 성공하거나 실패했는가?
결제 승인은 어떤 요청과 연결되는가?
티켓은 언제 발급되었는가?
알림은 어떤 채널로 발송되었는가?
운영자/CS가 특정 request_id, user_id, reservation_id, payment_id로 이력을 찾을 수 있는가?
```

## 시스템 관측성과의 차이

```text
시스템 관측성
-> 장애 탐지, 처리 지연, 에러율, 서비스 간 호출, trace 확인
-> Prometheus, Loki, Tempo, Grafana

감사 로그
-> 고객 문의 대응, 업무 이력 조회, 특정 ID 기반 조건 검색, 장기 로그 분석
-> Elasticsearch, Kibana
```

두 영역은 같은 요청에서 만들어질 수 있지만 저장 목적과 조회 방식이 다르다.

## 기본 전달 경로

```text
서비스 도메인 이벤트 / 감사 로그
-> stdout/stderr JSON 또는 별도 event publisher
-> 수집기
-> Elasticsearch
-> Kibana
```

수집기는 특정 도구로 고정하지 않는다. Elastic Agent, Filebeat, Logstash, OpenTelemetry Collector, Vector 등은 구현 시점에 비교한다.

중요한 기준은 애플리케이션이 감사 로그를 구조화된 이벤트로 남기고, 검색 스택이 그 필드를 안정적으로 색인하는 것이다.

## 감사 로그 예시

```json
{
  "event_id": "event_001",
  "event_type": "reservation_confirmed",
  "occurred_at": "2026-06-02T12:34:56Z",
  "observed_at": "2026-06-02T12:34:57Z",
  "service.name": "reservation-service",
  "service.version": "1.2.3",
  "deployment.environment": "prod",
  "user_id": "user_123",
  "actor_type": "customer",
  "reservation_id": "reservation_456",
  "seat_id": "seat_A-10",
  "payment_id": "payment_789",
  "request_id": "req_abc",
  "trace_id": "trace_xyz",
  "result": "success"
}
```

## 공통 필드 계약

감사 로그와 시스템 관측성 로그는 저장 목적이 다르지만 같은 요청을 서로 이어 보기 위한 공통 필드가 필요하다.

```text
감사 로그에서 user_id, reservation_id, payment_id로 업무 이력을 찾는다.
-> 같은 request_id 또는 trace_id로 Grafana/Tempo/Loki에서 처리 과정을 확인한다.
```

공통 필드는 다음 기준으로 둔다.

| 구분 | 필드 | 설명 |
|---|---|---|
| 시간 | `occurred_at` | 업무 사건이 실제로 발생한 시각 |
| 시간 | `observed_at` | 수집기가 관측한 시각 |
| 이벤트 | `event_id` | 감사 로그 한 건의 고유 ID |
| 이벤트 | `event_type` | 업무 사건 이름 |
| 서비스 | `service.name` | 사건을 만든 서비스 |
| 서비스 | `service.version` | 사건을 만든 서비스 버전 |
| 서비스 | `deployment.environment` | `local`, `dev`, `staging`, `prod` 같은 실행 환경 |
| 연결 | `request_id` | API 요청 단위 식별자 |
| 연결 | `trace_id` | 시스템 관측성과 연결할 trace 식별자 |
| 연결 | `span_id` | 처리 구간을 좁힐 span 식별자 |
| 주체 | `user_id` | 고객 또는 사용자 식별자 |
| 주체 | `actor_type` | `customer`, `operator`, `system` 구분 |
| 업무 객체 | `reservation_id` | 예매 식별자 |
| 업무 객체 | `seat_id` | 좌석 식별자 |
| 업무 객체 | `payment_id` | 결제 식별자 |
| 업무 객체 | `ticket_id` | 티켓 식별자 |
| 결과 | `result` | `success`, `failed`, `rejected`, `pending` |
| 결과 | `failure_reason` | 실패 또는 거절 사유 |

`user_id`, `reservation_id`, `payment_id`, `ticket_id`는 Elasticsearch에서는 검색 필드로 관리할 수 있다. 반면 Loki에서는 label로 올리지 않고 log field 또는 structured metadata로 둔다.

`request_id`와 `trace_id`는 감사 로그와 시스템 관측성을 연결하는 핵심 키다.

## 주요 이벤트

```text
reservation_requested
seat_hold_succeeded
seat_hold_failed
payment_approved
payment_failed
ticket_issued
notification_requested
notification_sent
notification_failed
```

이 목록은 시스템 로그 레벨이 아니라 업무 사건을 기준으로 확장한다.

## 검색 기준

Elasticsearch에서는 다음 필드를 우선 검색 대상으로 본다.

```text
event_id
event_type
occurred_at
service.name
service.version
deployment.environment
user_id
actor_type
reservation_id
seat_id
payment_id
ticket_id
notification_id
request_id
trace_id
result
failure_reason
```

`request_id`와 `trace_id`는 시스템 관측성과 연결하기 위한 공통 키다. 감사 로그에서 특정 업무 이력을 찾은 뒤, 같은 `trace_id`로 Grafana/Tempo/Loki에서 처리 과정을 확인할 수 있다.

## 보관 기준

감사 로그는 장애 분석 로그보다 더 오래 보관될 수 있다. 보관 기간은 고객 문의 대응, 운영 보고, 개인정보 처리 기준을 함께 보고 정한다.

주의할 점:

- 개인정보와 결제 민감 정보는 그대로 저장하지 않는다.
- 조회 권한은 운영자/CS 역할에 맞춰 제한한다.
- 고객에게 직접 노출할 이력은 감사 로그를 그대로 보여주지 않고 별도 API 응답 모델로 가공한다.
- 장애 분석용 debug log를 감사 로그로 대체하지 않는다.

## Repo별 책임

| Repo | 책임 |
|---|---|
| `workspace` | 감사 로그 기준, 이벤트 목록, 검색 기준, 보관 정책 문서화 |
| `service` | 업무 이벤트 생성, 필드명, 민감 정보 제거, `request_id`/`trace_id` 포함 |
| `gitops` | Elasticsearch/Kibana 또는 선택된 검색 스택 배포 선언 |
| `infra` | 장기 저장소, 접근 제어, 백업, 네트워크 기반 |
