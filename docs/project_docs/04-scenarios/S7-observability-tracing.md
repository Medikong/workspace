---
id: S7
title: 공연 티켓 예매 S7 관측성 추적
type: scenario
status: draft
priority: P0
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, observability, loki, tempo, grafana]
created: 2026-05-27
updated: 2026-05-27
---

## S7: 관측성 추적

### 검증 질문

실패하거나 지연된 예매 요청 1건을 3분 이내에 metric, log, trace로 설명할 수 있는가?

### 목표

관측성 도구 설치 여부가 아니라 장애 판단 능력을 검증한다. 운영자는 correlationId를 기준으로 gateway, service, Kafka producer, consumer 흐름을 따라가며 병목이나 실패 지점을 설명할 수 있어야 한다.

### 범위

- HTTP metric, business metric, Kafka metric 확인
- service log의 `service`, `correlationId`, `userId`, `reservationId`, `eventId`, `status` 필드 확인
- Loki log query
- Tempo trace query
- Grafana dashboard와 alert history 확인
- 실패/지연 요청 1건의 조사 절차 기록

### 성공 기준

- 실패/지연 요청 1건을 3분 이내에 추적한다.
- metric에서 이상 시점과 영향을 받은 service를 확인한다.
- log에서 같은 correlationId의 요청과 이벤트를 찾는다.
- trace에서 gateway부터 후속 consumer까지의 구간별 duration을 확인한다.

### 증거

- Grafana dashboard 캡처
- Loki query 결과
- Tempo trace 화면 또는 trace id
- Prometheus query 결과
- Alertmanager alert history
- 조사 절차와 소요 시간 기록

### 의존성

```text
S0 정상 예매 E2E
↓
서비스 metric/log/trace 필드 확정
↓
Prometheus 수집 확인
↓
Loki 로그 조회 확인
↓
Tempo trace 조회 확인
↓
실패 또는 지연 요청 생성
↓
3분 이내 원인 설명과 증거 기록
```

### Epic 후보

- 예매 흐름 관측성 검증
- correlationId 기반 추적
- 발표용 장애 조사 흐름 정리

### Issue 후보

- `service log 공통 필드를 점검한다`
- `예매 API metric과 business metric Prometheus query를 정리한다`
- `Loki에서 reservationId 또는 correlationId로 조회한다`
- `Tempo에서 예매 요청 trace를 확인한다`
- `실패/지연 요청 1건의 조사 시간을 기록한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
