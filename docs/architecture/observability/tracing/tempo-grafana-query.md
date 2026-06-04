# Tempo/Grafana 조회 기준

이 문서는 Tempo에 저장된 trace를 Grafana에서 어떻게 찾고, metric/log/audit log와 어떻게 이어 볼지 정리한다.

관련 문서:

- Trace 수집 경로와 repo 책임: `README.md`
- Sampling/retention 기준: `sampling-retention.md`
- ADR: `../../../adr/0004-observability-signal-routing-and-trace.md`

## 기본 원칙

```text
장애 분석 기본 흐름
  - metric
  - trace
  - log

metric
  - 장애 감지
  - 영향 범위 확인
  - 시간대 확인

trace
  - 실패 위치 확인
  - 지연 구간 확인
  - 외부 의존성 실패 확인

log
  - 필요한 경우에만 보조 맥락 확인
  - trace_id 또는 request_id로 연결
```

로그에서 먼저 원인을 찾는 방식을 기본으로 두지 않는다. Metric으로 장애 시점과 범위를 잡고, Tempo trace로 실패 구간을 좁힌 뒤, 필요한 경우에만 같은 `trace_id`의 애플리케이션 로그를 확인한다.

## 조회 진입점

```text
Grafana Explore
  - Tempo datasource 선택
  - trace_id 직접 조회
  - service.name, http.route, status 기준 탐색

Metric dashboard
  - 장애 시점 확인
  - error rate, latency, traffic 확인
  - 같은 시간대의 error trace 또는 slow trace 확인

Loki app log
  - log field의 trace_id 확인
  - trace_id 기반 Tempo 이동
  - error 요약에서 trace로 이동

Audit log
  - request_id 확인
  - trace_id 확인
  - 시스템 관측성 화면으로 이동
```

## 기본 조회 키

```text
공통 연결 키
  - trace_id
  - span_id
  - request_id

서비스 기준
  - service.name
  - service.version
  - deployment.environment

HTTP 기준
  - http.route
  - http.method
  - http.status_code

오류 기준
  - error.type
  - exception.type
  - 제한된 exception.message
```

`user_id`, `reservation_id`, `payment_id`, `ticket_id` 같은 업무 객체 ID는 trace 조회의 기본 키로 두지 않는다. 업무 이력 조회가 필요하면 감사 로그에서 먼저 찾고, 필요한 경우 `request_id` 또는 `trace_id`로 시스템 관측성 화면에 연결한다.

## 장애 분석 확인 순서

```text
1. metric 확인
   - 장애 시작 시점
   - 영향 받은 service
   - error rate
   - latency
   - traffic 변화

2. Tempo trace 확인
   - error trace
   - slow trace
   - root span
   - error span
   - latency가 큰 span
   - 실패한 외부 의존성 span

3. Loki app log 확인
   - 같은 trace_id
   - 같은 request_id
   - 제한된 error.message
   - service lifecycle event
   - 외부 의존성 실패 요약

4. 감사 로그 확인
   - 고객 문의
   - 업무 이력
   - 결제/예매/티켓 발급 결과
   - 증적 보관 대상 이벤트
```

## Grafana 연결 설정

```text
datasource
  - Prometheus
  - Tempo
  - Loki

Tempo
  - Tempo datasource 등록
  - trace 검색 활성화
  - trace-to-logs 설정

Loki
  - Loki datasource 등록
  - JSON log field 파싱
  - trace_id field 추출
  - trace_id 기반 Tempo 이동 링크

dashboard
  - service.name 기준 필터
  - deployment.environment 기준 필터
  - http.route 기준 필터
  - status/error 기준 필터
```

## 완료 기준

```text
Tempo/Grafana 조회 기준 완료
  - Grafana에서 Tempo datasource를 조회할 수 있다.
  - trace_id로 trace를 직접 조회할 수 있다.
  - metric dashboard에서 장애 시점과 대상 service를 확인할 수 있다.
  - Loki app log에서 trace_id field를 확인할 수 있다.
  - Loki trace_id에서 Tempo trace로 이동할 수 있다.
  - 장애 분석 흐름이 metric -> trace -> log 순서로 문서화되어 있다.
```
