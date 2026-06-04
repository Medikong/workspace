---
id: ADR-0004
title: "관측성 신호별 수집 경로와 Trace 처리 기준을 분리한다"
status: accepted
date: 2026-06-04
areas:
  - observability
  - tracing
  - metrics
  - logging
  - audit-log
repos:
  - workspace
  - service
  - gitops
  - infra
decision_drivers:
  - 신호별 수집 목적과 저장소 책임 분리
  - OpenTelemetry 표준과 Collector 구현체 경계 명확화
  - 애플리케이션 로그 노이즈 최소화
  - 로그와 감사 로그의 역할 혼동 방지
  - 고카디널리티 식별자의 비용 통제
related:
  - docs/architecture/observability/README.md
  - docs/architecture/observability/metrics/README.md
  - docs/architecture/observability/tracing/README.md
  - docs/architecture/observability/tracing/tempo-grafana-query.md
  - docs/architecture/observability/tracing/sampling-retention.md
  - docs/architecture/observability/implementation/ROADMAP.md
  - docs/architecture/audit-logs/README.md
  - docs/project_docs/04-scenarios/S7-observability-tracing.md
links:
  - https://github.com/Medikong/workspace/issues/7
  - https://github.com/Medikong/workspace/issues/24
  - https://github.com/Medikong/workspace/issues/25
  - https://github.com/Medikong/service/issues/14
  - https://github.com/Medikong/gitops/issues/18
  - https://github.com/Medikong/gitops/issues/19
supersedes: []
superseded_by: null
---

# ADR 0004: 관측성 신호별 수집 경로와 Trace 처리 기준을 분리한다

## 상태

Accepted

## 날짜

2026-06-04

## 배경

Medikong의 관측성 요구는 단일 데이터 종류가 아니다. 시스템 메트릭, 애플리케이션 메트릭, trace, 애플리케이션 로그, 시스템 로그, 사용자 감사 로그는 생성 위치와 조회 목적이 다르다.

하나의 수집 방식으로 모두 통일하면 다음 문제가 생긴다.

- Prometheus가 잘 처리하는 metric scrape와 OTLP trace export가 불필요하게 섞인다.
- 장애 분석용 애플리케이션 로그와 고객 문의/업무 이력용 감사 로그가 같은 저장소에 섞인다.
- `trace_id`, `request_id`, `user_id`, `payment_id` 같은 고유값이 metric label 또는 Loki label로 올라가 저장 비용과 쿼리 성능 문제가 생긴다.
- OpenTelemetry 표준과 실제 Collector 구현체를 구분하지 않으면 upstream OpenTelemetry Collector와 Grafana Alloy의 역할이 모호해진다.
- 애플리케이션 로그를 과도하게 남기면 장애 대응 근거보다 노이즈가 많아진다.

따라서 관측성은 도구 하나로 통일하지 않고, 신호별 목적에 맞는 경로로 분리한다.

## 결정

상세 운영 기준은 다음 문서로 분리한다.

- Trace 수집 경로와 repo 책임: `../architecture/observability/tracing/README.md`
- Tempo/Grafana 조회 기준: `../architecture/observability/tracing/tempo-grafana-query.md`
- Trace sampling과 retention 기준: `../architecture/observability/tracing/sampling-retention.md`

### 신호별 수집 경로

Medikong의 core observability signal은 다음 6개로 나누고, 수집 경로를 분리한다.

| 신호 | 기본 경로 | 저장/조회 |
| --- | --- | --- |
| 시스템 메트릭 | `node-exporter`, `kube-state-metrics`, `kubelet/cAdvisor` -> Prometheus scrape | Prometheus, Grafana |
| 애플리케이션 메트릭 | FastAPI `/metrics` -> ServiceMonitor -> Prometheus scrape | Prometheus, Grafana |
| Trace | FastAPI OpenTelemetry instrumentation -> OTLP -> OpenTelemetry Collector -> Tempo | Tempo, Grafana |
| 애플리케이션 로그 | stdout/stderr JSON -> Kubernetes container log -> OpenTelemetry Collector filelog receiver | Loki, Grafana |
| 시스템 로그 | node/pod/container log -> OpenTelemetry Collector DaemonSet filelog receiver | Loki, Grafana |
| 사용자 감사 로그 | 업무 이벤트/outbox -> Kafka, Logstash, Beats 등 별도 경로 -> Elasticsearch | Elasticsearch, Kibana |

---

### OpenTelemetry 표준과 Collector

OpenTelemetry는 instrumentation, OTLP, trace/log pipeline의 표준으로 채택한다. Collector 계층의 기본 구현체는 upstream OpenTelemetry Collector로 둔다. Grafana Alloy는 Grafana LGTM 통합 운영 편의가 필요할 때 검토할 수 있는 OpenTelemetry Collector distribution 대안으로 둔다.

```text
OpenTelemetry 표준
-> application instrumentation
-> OTLP
-> upstream OpenTelemetry Collector
-> backend(Prometheus, Tempo, Loki 등)
```

---

### Trace 수집 경로

Trace는 다음 기준으로 처리한다.

```text
FastAPI OpenTelemetry instrumentation
-> OTLP
-> OpenTelemetry Collector OTLP receiver
-> Tempo
-> Grafana
```

---

### Trace repo 책임

Trace 수집 경로의 repo 책임은 다음처럼 나눈다.

```text
workspace
  - ADR
  - 이슈 구조
  - 아키텍처 문서
  - 운영 기준
  - trace sampling 기준
  - retention 기준
  - high-cardinality attribute 금지 기준

service
  - FastAPI OpenTelemetry instrumentation
  - span 생성
  - request context 전파
  - trace context 전파
  - error span/event 기록
  - OTLP endpoint 환경변수 사용
  - service.name 설정
  - deployment.environment 설정

gitops
  - OpenTelemetry Collector 배포
  - OTLP receiver 설정
  - processor 설정
  - Tempo exporter 설정
  - tail sampling 정책
  - memory limiter
  - batch processor
  - Tempo backend 배포
  - Grafana datasource 연결

infra
  - cluster 기반 리소스
  - namespace 경계
  - storage class
  - object storage
  - network policy
  - secret/권한 경계
```

Trace는 서비스 간 호출 흐름과 처리 지연 구간을 보기 위한 신호다. 감사 로그의 장기 증적 저장소가 아니며, metric 집계 저장소도 아니다.

---

### Tempo/Grafana 조회 기준

Tempo/Grafana 조회 기준은 trace 저장 여부가 아니라 운영자가 trace를 어떻게 찾고 다른 신호와 어떻게 이어 보는지를 정의한다.

```text
조회 진입점
  - Grafana Explore
  - Tempo datasource
  - metric dashboard의 장애 시점
  - Loki app log의 trace_id field
  - 감사 로그의 request_id 또는 trace_id

기본 조회 키
  - trace_id
  - span_id
  - request_id
  - service.name
  - deployment.environment
  - http.route
  - http.status_code
  - error.type

장애 분석 확인 순서
  - metric으로 장애 시점과 영향 범위 확인
  - Tempo에서 error trace 또는 slow trace 확인
  - root span 확인
  - error span 확인
  - latency가 큰 span 확인
  - 실패한 외부 의존성 span 확인
  - 필요할 때만 같은 trace_id의 Loki app log 확인

Grafana 연결 설정
  - Tempo datasource 등록
  - Loki datasource 등록
  - Tempo trace-to-logs 설정
  - Loki log field의 trace_id 추출
  - trace_id 기반 Tempo 이동 링크
```

Trace 조회의 기본 흐름은 metric -> trace -> log 순서로 둔다. 로그에서 먼저 원인을 찾는 방식이 아니라, metric으로 장애를 감지하고 trace로 실패 구간을 좁힌 뒤 필요한 경우에만 같은 `trace_id`의 애플리케이션 로그를 확인한다.

---

### Trace sampling

Trace sampling은 tail sampling을 기본 방향으로 둔다. Head sampling은 요청 시작 시점에 저장 여부를 결정하므로 단순하지만, 요청이 끝난 뒤에야 알 수 있는 error, latency, 핵심 업무 흐름 여부를 반영하기 어렵다. Tail sampling은 Collector 계층에서 trace를 일정 시간 모은 뒤 정책을 평가할 수 있으므로 장애 분석에 더 적합하다.

프로젝트 초입 단계에서는 trace를 100% 보존하는 설정으로 시작한다. 이후 실제 요청량, span 수, Tempo 저장량, 조회 비용을 확인한 뒤 prod 환경의 sampling 비율과 정책을 조정한다.

환경별 기본 기준은 다음처럼 둔다.

| 환경 | 기준 |
| --- | --- |
| `dev` | trace 100% 보존 |
| `staging` | trace 100% 보존 |
| `prod` 초기 | trace 100% 보존 후 실제 볼륨과 비용 측정 |
| `prod` 안정화 | error, slow, 핵심 업무 flow는 보존하고 정상 일반 요청은 낮은 비율로 보존 |

Prod 안정화 이후에도 다음 trace는 가능한 한 보존한다.

- error span이 포함된 trace
- 5xx 응답 trace
- 예매, 결제, 티켓 발급 같은 핵심 업무 flow trace
- latency threshold를 초과한 slow trace

다음 trace는 drop 또는 낮은 비율 보존 후보로 둔다.

- `/healthz`, `/readyz`, `/metrics`
- heartbeat, polling, readiness/liveness
- 정상 2xx 일반 조회 요청
- 반복성이 높은 consumer 처리 trace

---

### 장애 대응 기준

장애 대응의 1차 근거는 metric과 trace로 둔다. Metric은 장애 감지와 영향 범위 판단에 사용하고, trace는 실패 또는 지연이 발생한 처리 구간을 좁히는 데 사용한다. 애플리케이션 로그는 주요 판단 근거가 아니라 예외적 맥락을 보강하는 보조 신호로 둔다.

장애 발생 위치와 당시 실행 컨텍스트는 trace span에 남긴다. 실패한 처리 구간의 span은 error 상태를 가져야 하며, 예외가 발생한 경우 span event 또는 span attribute로 다음 정보를 확인할 수 있어야 한다.

- `exception.type`
- 제한된 `exception.message`
- `exception.stacktrace`
- `service.name`
- `http.route`
- `http.status_code`
- 관련 외부 의존성 이름 또는 operation 이름

따라서 stack trace 전체를 애플리케이션 로그에 항상 출력하지 않는다. 로그에는 같은 요청을 trace로 이어 볼 수 있는 `trace_id`, `span_id`, `request_id`와 제한된 error 요약을 남기고, 상세한 실패 위치와 호출 컨텍스트는 Tempo trace에서 확인하는 구조로 둔다.

---

### 애플리케이션 로그 최소화

애플리케이션 로그는 다음 정보로 제한한다.

- 요청 완료 요약: `service.name`, `http.route`, `http.status_code`, `duration_ms`, `trace_id`, `request_id`
- error/exception: `error.type`, 제한된 `error.message`, trace 연결 필드
- 외부 의존성 실패: DB, Kafka, 결제, 알림 같은 외부 시스템 실패
- service lifecycle: startup, shutdown, 필수 config 누락, consumer 시작/중단

애플리케이션 로그에는 다음 정보를 기본으로 남기지 않는다.

- 요청/응답 body 전체
- 정상 요청의 세부 단계 debug log
- trace span으로 이미 확인 가능한 duration breakdown 중복 로그
- consumer heartbeat 같은 반복성 높은 정상 상태 로그
- 개인정보, 결제 민감 정보, 인증 토큰

## 필드와 연결 기준

`trace_id`, `span_id`, `request_id`는 metric, log, trace, audit log를 이어 보기 위한 공통 연결 키로 둔다.

- Trace에는 `trace_id`, `span_id`, `service.name`, `service.version`, `deployment.environment`를 포함한다.
- 애플리케이션 로그에는 `trace_id`, `span_id`, `request_id`, `service.name`을 구조화 필드로 남긴다.
- 감사 로그에는 `trace_id`, `request_id`를 포함해 시스템 관측성과 연결할 수 있게 한다.
- `trace_id`, `span_id`, `request_id`, `user_id`, `reservation_id`, `payment_id`, raw URL path는 metric label로 사용하지 않는다.

Loki label은 낮은 cardinality를 가진 bounded 값으로 제한한다. 요청마다 값이 바뀌거나 사용자/업무 객체마다 값이 달라지는 식별자는 Loki label로 올리지 않고 JSON log field 또는 structured metadata로 둔다.

Loki label로 사용할 수 있는 값:

- `service.name`
- `deployment.environment`
- Kubernetes namespace
- Kubernetes workload 또는 app name
- container name
- `severity` 또는 `severity_text`
- bounded `component` 값, 예: `api`, `consumer`, `worker`

Loki label로 사용하지 않는 값:

- `trace_id`
- `span_id`
- `request_id`
- `user_id`
- `reservation_id`
- `payment_id`
- `ticket_id`
- `seat_id`
- raw URL path
- query string
- exception message
- request/response body 값

위 값들은 검색과 연결에 필요할 수 있으므로 로그 본문 필드로는 남길 수 있다. 다만 label/indexing 대상이 되지 않게 한다.

Trace, 애플리케이션 로그, 감사 로그는 같은 `trace_id`와 `request_id`를 공유할 수 있지만 저장 목적은 다르다.

| 구분 | 주 목적 | 저장할 정보 | 저장하지 않을 정보 |
| --- | --- | --- | --- |
| Trace | 실패 위치, 지연 구간, 서비스 간 호출 흐름 확인 | span name, status, duration, `exception.*`, `service.name`, `http.route`, `http.status_code`, 외부 의존성 operation, 제한된 업무 flow 구분자 | 요청/응답 body, 개인정보, 결제 민감 정보, 대용량 payload, 사용자별 고유값 기반 대량 attribute |
| 애플리케이션 로그 | trace를 찾기 위한 보조 단서와 제한된 error 맥락 | `trace_id`, `span_id`, `request_id`, `service.name`, route, status, duration, 제한된 `error.type`/`error.message`, lifecycle event | stack trace 전체 상시 출력, 정상 요청 debug 단계 로그, request/response body, heartbeat 반복 로그, 감사 이벤트 전체 |
| 감사 로그 | 고객 문의 대응, 업무 이력 조회, 증적 보관 | `event_id`, `event_type`, `occurred_at`, `user_id`, `actor_type`, `reservation_id`, `payment_id`, `ticket_id`, result, `request_id`, `trace_id` | 장애 분석용 debug log, 시스템 로그 원문, span duration breakdown, 개인정보/결제 민감 원문 |

Trace attribute는 장애 분석에 필요한 범위에서만 추가한다. `http.route`, `service.name`, `error.type`, 외부 의존성 이름처럼 값의 종류가 제한된 필드를 우선 사용한다. `user_id`, `reservation_id`, `payment_id` 같은 업무 객체 ID는 감사 로그 검색 필드로는 중요하지만, trace attribute에는 기본으로 대량 추가하지 않는다. 특정 핵심 업무 flow에서 연결이 꼭 필요하면 마스킹되거나 제한된 형태로 별도 검토한다.

애플리케이션 로그는 감사 로그를 대체하지 않는다. 고객 문의나 업무 이력 조회에 필요한 사실은 감사 로그 이벤트로 남기고, 앱 로그는 장애 분석을 위한 기술적 보조 정보로 제한한다.

## 포함 범위

이 ADR은 다음 결정을 포함한다.

- metric은 Prometheus scrape를 기본으로 둔다.
- trace는 OTLP를 통해 Collector 계층으로 보내고 Tempo에 저장한다.
- Tempo/Grafana 조회 기준은 metric -> trace -> log 흐름과 trace-to-logs 연결 기준으로 둔다.
- trace sampling은 tail sampling을 기본 방향으로 둔다.
- 프로젝트 초입에는 trace 100% 보존으로 시작하고, 실제 볼륨과 비용을 확인한 뒤 prod sampling 정책을 조정한다.
- Collector 계층의 기본 구현체는 upstream OpenTelemetry Collector로 둔다.
- 애플리케이션 로그는 장애 분석 보조 신호로 두고 최소 필드만 stdout/stderr JSON으로 출력한다.
- 시스템 로그는 stdout/stderr와 filelog receiver 기반으로 수집한다.
- Loki label은 낮은 cardinality의 bounded 값으로 제한하고, 요청/사용자/업무 객체별 고유값은 label로 사용하지 않는다.
- Trace, 애플리케이션 로그, 감사 로그는 저장 목적과 필드 범위를 분리한다.
- 사용자 감사 로그는 Loki가 아니라 Elasticsearch/Kibana 계열의 별도 검색 스택으로 보낸다.
- Grafana는 시스템 관측성 조회 화면이고, 감사 로그 조회 화면은 별도 영역으로 둔다.

## 제외 범위

다음은 이 ADR에서 최종 결정하지 않는다.

- prod 안정화 이후 정상 요청 sampling 비율의 최종 값
- trace retention 기간의 최종 값
- Elasticsearch 대신 OpenSearch를 사용할지 여부
- 감사 로그 전송 구현체를 Kafka, Logstash, Beats, Vector 중 무엇으로 둘지
- Profiling, Synthetic/E2E Probe, 배포 이벤트 annotation의 세부 구현

위 항목은 후속 gitops, service, infra 이슈에서 구체화한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| 모든 신호를 OpenTelemetry/OTLP로 통일 | 수집 프로토콜 설명이 단순하다. | Kubernetes/system metric은 Prometheus scrape 생태계가 더 자연스럽고, 로그까지 OTLP로 직접 보내면 stdout 수집과 중복될 수 있다. | 채택하지 않음 |
| metric은 Prometheus, trace는 OTLP, log는 stdout/filelog로 분리 | 각 신호의 일반적인 운영 방식과 잘 맞고 현재 repo 구조와도 맞다. | 수집 경로가 여러 개라 문서와 GitOps 책임 경계를 명확히 해야 한다. | 채택 |
| Head sampling 사용 | 구현이 단순하고 초기에 데이터량을 줄이기 쉽다. | 요청 종료 후에 알 수 있는 error, slow trace, 핵심 업무 flow 여부를 반영하기 어렵다. | 채택하지 않음 |
| Tail sampling 사용 | error, slow, 핵심 업무 flow를 더 잘 보존할 수 있다. 장애 분석에 필요한 trace를 남기기 쉽다. | Collector 계층에서 trace를 모아야 하므로 메모리와 구성이 더 필요하다. | 채택 |
| 애플리케이션 로그를 상세하게 남김 | 요청별 맥락을 로그만으로 많이 볼 수 있다. | 정상 요청 로그가 노이즈가 되고 저장 비용이 증가한다. trace와 metric으로 충분한 정보를 중복 저장하게 된다. | 채택하지 않음 |
| 애플리케이션 로그를 최소화하고 metric/trace 중심으로 분석 | 장애 감지와 원인 구간 분석을 구조화된 신호에 맡기고 로그 노이즈를 줄일 수 있다. | 일부 상세 맥락은 trace span attribute 또는 제한된 error log 설계가 필요하다. | 채택 |
| Loki label에 `trace_id`, `request_id`, 업무 ID를 포함 | 특정 요청이나 업무 객체를 label query로 바로 찾을 수 있다. | label cardinality가 급증해 저장 비용과 쿼리 성능 문제가 생긴다. | 채택하지 않음 |
| Loki label은 bounded 값으로 제한하고 고유값은 log field로 둠 | Loki 저장 비용과 쿼리 성능을 통제하면서 trace/log 연결은 필드 검색으로 유지할 수 있다. | 특정 ID 조회는 label query보다 느릴 수 있다. | 채택 |
| Trace, 앱 로그, 감사 로그에 같은 상세 정보를 모두 저장 | 어디서든 같은 정보를 볼 수 있다. | 중복 저장, 개인정보/민감 정보 노출, 저장 비용 증가, 책임 경계 혼란이 생긴다. | 채택하지 않음 |
| Trace, 앱 로그, 감사 로그의 필드 역할 분리 | 각 저장소의 목적이 명확하고 비용과 노이즈를 통제할 수 있다. | 서로 연결하기 위한 공통 키 표준을 지켜야 한다. | 채택 |
| Collector 계층에 upstream OpenTelemetry Collector 사용 | 표준 문서와 레퍼런스가 가장 넓고 vendor-neutral 설명이 쉽다. Grafana 외 backend로 확장하기 쉽다. | Grafana LGTM 전용 편의 기능은 Alloy보다 직접 구성해야 한다. | 채택 |
| Collector 계층에 Grafana Alloy 사용 | Grafana LGTM, Prometheus pipeline, Loki, Pyroscope 연동이 편하다. | upstream Collector와 설정/운영 표면이 완전히 같지 않고 Grafana 생태계 의존도가 커진다. | 대안으로 유지 |
| 감사 로그를 Loki에 함께 저장 | 운영 화면을 하나로 줄일 수 있다. | 고객 문의, 업무 이력, 장기 보관, 접근 제어 요구와 맞지 않는다. 시스템 로그와 감사 증적이 섞인다. | 채택하지 않음 |

## 결과

좋아지는 점:

- metric, trace, log, audit log의 목적과 저장소가 명확해진다.
- 현재 이미 준비된 `/metrics`, ServiceMonitor, Prometheus 흐름을 그대로 살릴 수 있다.
- trace 작업은 `workspace#24`, `workspace#25`, `gitops#18`, `gitops#19`, `service#14`로 독립 추적할 수 있다.
- upstream OpenTelemetry Collector 문서와 예제를 기준으로 gitops 구성을 작성할 수 있다.
- 애플리케이션 로그 노이즈와 저장 비용을 줄이고, 장애 판단 근거를 metric과 trace로 모을 수 있다.
- 감사 로그는 시스템 관측성 로그와 분리되어 검색, 보관, 접근 제어 기준을 별도로 세울 수 있다.

비용:

- Grafana, Prometheus, Tempo, Loki, Elasticsearch/Kibana의 역할을 문서와 runbook에서 계속 분리해야 한다.
- `trace_id`와 `request_id`가 여러 저장소에 존재하므로 필드명 표준을 지켜야 한다.
- OpenTelemetry Collector, Tempo, Loki, Elasticsearch 같은 backend 운영 기준이 repo별 이슈로 나뉜다.
- prod 초기 100% trace 보존 기간에는 Tempo 저장량과 비용을 주기적으로 확인해야 한다.
- tail sampling은 Collector 메모리, decision wait, trace completeness 기준을 별도로 튜닝해야 한다.
- retention 정책을 별도 결정하지 않으면 저장 비용을 예측하기 어렵다.
- 로그를 최소화하는 대신 trace span attribute와 error log 필드 기준을 더 신중하게 정해야 한다.

## 적용 원칙

1. Metric은 Prometheus scrape 경로를 기본으로 둔다.
2. Trace는 OTLP 경로를 기본으로 두고 Tempo에서 조회한다.
3. Trace sampling은 tail sampling을 기본 방향으로 둔다.
4. 프로젝트 초입에는 trace 100% 보존으로 시작한다.
5. Prod 안정화 이후에는 error, slow, 핵심 업무 flow를 보존하고 정상 일반 요청은 비용에 맞춰 낮은 비율로 보존한다.
6. Collector 계층의 기본 구현체는 upstream OpenTelemetry Collector로 둔다.
7. Grafana Alloy는 Grafana LGTM 통합 편의가 필요할 때 검토하는 대안으로 둔다.
8. 애플리케이션 로그는 장애 분석 보조 신호로 두고, 대응에 필요한 최소 정보만 stdout/stderr JSON으로 출력한다.
9. 로그 전송은 애플리케이션이 backend로 직접 보내지 않고 Collector filelog receiver가 Kubernetes container log를 읽는 구조로 둔다.
10. OTLP logs exporter는 기본으로 켜지 않는다.
11. Loki label은 `service.name`, environment, namespace, workload, severity처럼 낮은 cardinality 값으로 제한한다.
12. `trace_id`, `request_id`, 사용자 ID, 업무 객체 ID, raw path, exception message는 Loki label로 사용하지 않는다.
13. Trace는 실패 위치와 호출 흐름을, 애플리케이션 로그는 제한된 기술 맥락을, 감사 로그는 업무 증적을 담당한다.
14. 감사 로그는 시스템 관측성 로그로 대체하지 않는다.
15. 고유 식별자는 label이 아니라 structured field 또는 검색 필드로 둔다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| todo | trace 수집 기준과 조회 기준 ADR 검토 | unassigned | `workspace#25` |
| todo | Tempo trace backend와 Grafana datasource 배포 기준 선언 | unassigned | `gitops#19` |
| todo | OpenTelemetry Collector OTLP receiver와 Tempo exporter 구성 | unassigned | `gitops#18` |
| todo | FastAPI OpenTelemetry trace instrumentation과 context 전파 적용 | unassigned | `service#14` |
| todo | trace sampling 정책과 prod 비용 측정 기준 정리 | unassigned | `workspace#25` |
| todo | metric과 trace를 기준으로 Grafana 조회 흐름 검증 | unassigned | `workspace#24` |
| todo | 애플리케이션/시스템 로그 수집 기준 정리 | unassigned | `workspace#9` |
| todo | 감사 로그 검색 파이프라인은 시스템 관측성과 별도 설계 | unassigned | `workspace#21` |
