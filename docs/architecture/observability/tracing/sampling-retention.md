# Trace sampling과 retention 기준

이 문서는 trace sampling과 retention의 기본 정책을 정리한다.

관련 문서:

- Trace 수집 경로와 repo 책임: `README.md`
- Tempo/Grafana 조회 기준: `tempo-grafana-query.md`
- ADR: `../../../adr/0004-observability-signal-routing-and-trace.md`

## 기본 결정

```text
sampling 방식
  - tail sampling

초기 정책
  - dev trace 100% 보존
  - staging trace 100% 보존
  - prod 초기 trace 100% 보존

조정 시점
  - 실제 요청량 확인 후
  - span 생성량 확인 후
  - Tempo 저장량 확인 후
  - 조회 비용 확인 후
```

프로젝트 초입에는 trace를 먼저 충분히 남긴다. 초기에는 어떤 span이 실제 장애 분석에 필요한지 알기 어렵기 때문에, prod도 100% 보존으로 시작한 뒤 실제 볼륨과 비용을 보고 sampling 정책을 줄인다.

## 환경별 기준

```text
dev
  - trace 100% 보존
  - 개발 중 span 누락 확인
  - instrumentation 검증

staging
  - trace 100% 보존
  - 배포 전 end-to-end 흐름 검증
  - Grafana 조회 흐름 검증

prod 초기
  - trace 100% 보존
  - 실제 traffic 기준 비용 측정
  - Tempo 저장량 측정
  - slow/error trace 분석 기준 수립

prod 안정화
  - error trace 보존
  - slow trace 보존
  - 핵심 업무 flow 보존
  - 정상 일반 요청은 낮은 비율 보존
```

## 보존 우선 trace

```text
항상 보존 후보
  - error span이 포함된 trace
  - 5xx 응답 trace
  - latency threshold 초과 trace
  - 예매 flow trace
  - 결제 flow trace
  - 티켓 발급 flow trace
  - 외부 의존성 실패 trace

낮은 비율 보존 후보
  - 정상 2xx 일반 조회 요청
  - 반복성이 높은 consumer 처리 trace
  - polling trace

drop 후보
  - /healthz
  - /readyz
  - /metrics
  - liveness probe
  - readiness probe
  - heartbeat
```

## Tail sampling 정책 방향

```text
Collector 정책 기준
  - status_code 기준 error 보존
  - latency 기준 slow trace 보존
  - http.route 기준 probe drop
  - service.name 기준 서비스별 정책 적용
  - 핵심 업무 flow 기준 보존

Collector 운영 기준
  - decision wait 설정
  - expected new traces per sec 측정
  - memory limiter 설정
  - batch processor 설정
  - dropped span 수 모니터링
```

Tail sampling은 Collector가 trace를 일정 시간 모은 뒤 보존 여부를 결정한다. 그래서 `decision wait`, Collector 메모리, span 유실 여부를 함께 관리해야 한다.

## Retention 기준

```text
초기 기준
  - 최종 retention 기간은 이 ADR에서 고정하지 않는다.
  - prod 초기 100% 보존 기간에는 Tempo 저장량을 측정한다.
  - retention은 비용과 장애 분석 필요 기간을 보고 결정한다.

결정 입력값
  - 일일 trace 수
  - 일일 span 수
  - 평균 span size
  - Tempo 저장량 증가율
  - Grafana 조회 빈도
  - 장애 분석에 필요한 과거 조회 기간

후속 결정 대상
  - dev retention
  - staging retention
  - prod retention
  - error/slow trace 별도 retention 여부
```

## High-cardinality attribute 금지 기준

```text
trace attribute 기본 금지
  - user_id
  - reservation_id
  - payment_id
  - ticket_id
  - seat_id
  - raw URL path
  - query string
  - request body
  - response body
  - 인증 토큰
  - 개인정보 원문
  - 결제 민감 정보 원문

trace attribute 허용 후보
  - service.name
  - service.version
  - deployment.environment
  - http.route
  - http.method
  - http.status_code
  - error.type
  - exception.type
  - 외부 의존성 이름
  - 제한된 업무 flow 구분자
```

업무 객체 ID는 감사 로그 검색 필드로는 중요하지만, trace attribute에 기본으로 대량 추가하지 않는다. 특정 핵심 업무 flow에서 연결이 꼭 필요하면 마스킹되거나 제한된 형태로 별도 검토한다.

## 완료 기준

```text
sampling/retention 기준 완료
  - tail sampling 사용 기준이 문서화되어 있다.
  - dev/staging/prod 초기 기준이 문서화되어 있다.
  - prod 안정화 이후 조정 방향이 문서화되어 있다.
  - 보존 우선 trace와 drop 후보가 분리되어 있다.
  - retention 최종값을 별도 후속 결정으로 둔다.
  - high-cardinality attribute 금지 기준이 문서화되어 있다.
```
