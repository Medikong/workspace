---
id: S3
title: 공연 티켓 예매 S3 Kafka 후속 처리 분리
type: scenario
status: draft
priority: P0
phase: Sprint 1-Sprint 3
tags: [ticketing, scenario, kafka, async-processing, ticket]
created: 2026-05-27
updated: 2026-05-27
---

## S3: Kafka 후속 처리 분리

### 검증 질문

예약 API 응답시간과 결제 이후 티켓 발행 지연을 분리해 측정할 수 있는가?

### 목표

예약 API가 후속 티켓 발행과 알림 저장을 기다리지 않고 응답하는지 확인한다. Kafka 이벤트 기반 처리의 목적은 "모든 일을 빠르게 끝낸다"가 아니라 핵심 요청 경로와 후속 처리 지연을 분리해 운영자가 각각 관측할 수 있게 만드는 것이다.

### 범위

- `reservation-created`, `payment-approved`, `payment-failed`, `ticket-issued` 이벤트 흐름
- reservation API p95 latency 측정
- ticket issue delay와 Kafka consumer lag 측정
- consumer idempotency와 중복 이벤트 처리 확인
- 이벤트별 correlationId 전달 확인

### 성공 기준

- 예약 API p95와 티켓 발행 delay가 별도 지표로 측정된다.
- 결제 승인 이후 `payment-approved` 이벤트가 ticket-service로 전달된다.
- ticket-service가 티켓을 발행하고 `ticket-issued` 이벤트 또는 상태를 남긴다.
- Kafka consumer lag가 Grafana 또는 Prometheus query로 확인된다.

### 증거

- reservation API latency metric
- Kafka consumer lag metric
- ticket issue delay metric
- 이벤트별 service log와 correlationId
- Kafka topic 또는 consumer group 조회 결과
- ticket DB 최종 상태

### 의존성

```text
S0 정상 예매 E2E
↓
Kafka topic과 event schema 확정
↓
payment-service producer 준비
↓
ticket-service consumer 준비
↓
consumer lag와 ticket issue delay metric 노출
↓
부하 중 예약 API latency와 후속 처리 지연 측정
↓
결과 기록
```

### Epic 후보

- Kafka 이벤트 계약 검증
- 후속 처리 지연 관측
- consumer idempotency 검증

### Issue 후보

- `핵심 Kafka 이벤트 schema를 문서화한다`
- `payment-approved 이벤트 발행과 수신 로그를 연결한다`
- `reservation API p95 metric을 Prometheus에서 조회한다`
- `ticket issue delay와 consumer lag metric을 수집한다`
- `중복 이벤트 처리 결과를 DB 상태로 확인한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
