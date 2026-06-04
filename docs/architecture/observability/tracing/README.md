# Trace 수집 경로와 repo 책임

이 문서는 관측성 신호별 수집 경로와 repo 책임을 정리한다.

관련 결정:

- ADR: `../../../adr/0004-observability-signal-routing-and-trace.md`
- Tempo/Grafana 조회 기준: `tempo-grafana-query.md`
- Sampling/retention 기준: `sampling-retention.md`

## 신호별 수집 경로

```text
시스템 메트릭
  - 생성: Kubernetes node, pod, container, control plane
  - 수집: node-exporter, kube-state-metrics, kubelet/cAdvisor
  - 전달: Prometheus scrape
  - 저장/조회: Prometheus, Grafana

애플리케이션 메트릭
  - 생성: FastAPI /metrics
  - 수집: ServiceMonitor
  - 전달: Prometheus scrape
  - 저장/조회: Prometheus, Grafana

Trace
  - 생성: FastAPI OpenTelemetry instrumentation
  - 전달: OTLP
  - 수집/처리: OpenTelemetry Collector
  - 저장/조회: Tempo, Grafana

애플리케이션 로그
  - 생성: app stdout/stderr JSON
  - 수집: Kubernetes container log
  - 전달: OpenTelemetry Collector filelog receiver
  - 저장/조회: Loki, Grafana

시스템 로그
  - 생성: node, pod, container log
  - 수집: OpenTelemetry Collector DaemonSet filelog receiver
  - 저장/조회: Loki, Grafana

사용자 감사 로그
  - 생성: 업무 이벤트 또는 outbox
  - 전달: Kafka, Logstash, Beats 등 별도 경로
  - 저장/조회: Elasticsearch, Kibana
```

## Trace 경로

```text
FastAPI OpenTelemetry instrumentation
-> OTLP exporter
-> OpenTelemetry Collector OTLP receiver
-> Collector processors
   - resource
   - memory_limiter
   - batch
   - tail_sampling
-> Tempo exporter
-> Tempo
-> Grafana
```

## Repo 책임

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

## 역할 경계

```text
service
  - trace를 만든다.
  - trace context를 전파한다.
  - Collector endpoint를 설정으로 사용한다.

gitops
  - trace를 받는 Collector pipeline을 배포한다.
  - trace를 저장할 Tempo를 배포한다.
  - Grafana에서 Tempo를 조회할 수 있게 연결한다.

infra
  - gitops가 배포할 수 있는 기반 리소스를 제공한다.
  - storage, network, secret, 권한 경계를 준비한다.

workspace
  - 왜 이 경로를 쓰는지 기록한다.
  - repo별 후속 작업과 완료 기준을 정리한다.
```

## 분리 원칙

```text
metric
  - 집계와 알림 기준
  - Prometheus scrape
  - 고유 ID label 금지

trace
  - 실패 위치와 호출 흐름 확인
  - OTLP
  - Tempo 조회

애플리케이션 로그
  - trace를 찾기 위한 보조 정보
  - stdout/stderr JSON
  - Loki 조회
  - 과도한 debug log 금지

감사 로그
  - 고객 문의 대응
  - 업무 이력 조회
  - 증적 보관
  - 시스템 관측성과 별도 저장소
```
