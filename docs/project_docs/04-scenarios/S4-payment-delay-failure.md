---
id: S4
title: 공연 티켓 예매 S4 결제 지연과 장애
type: scenario
status: draft
priority: P0
phase: Sprint 3
tags: [ticketing, scenario, payment, failure, grafana]
created: 2026-05-27
updated: 2026-05-27
---

## S4: 결제 지연과 장애

### 검증 질문

결제 승인 지연이나 실패가 발생해도 예약이 유실되지 않고, 상태가 `pending`, `paid`, `failed` 중 하나로 설명되는가?

### 목표

외부 결제 의존성이 느리거나 실패하는 상황을 예매 도메인 안에서 검증한다. 사용자는 예약 상태를 확인할 수 있어야 하고, 운영자는 결제 지연이 예매 API 5xx 폭증으로 번지는지 판단할 수 있어야 한다.

### 범위

- payment-service 지연/실패 모드
- 예약 상태 분포 조회
- pending 전환율과 5xx rate 측정
- 결제 실패 이벤트와 예약 상태 전환 확인
- Alertmanager 또는 Grafana alert 기록 확인

### 성공 기준

- 결제 지연/장애 중 예약 유실이 0건이다.
- 지연 요청의 pending 전환율이 95% 이상이다.
- 주문 또는 결제 API 5xx 에러율이 1% 이하이다.
- 장애 시점과 회복 시점이 Grafana, log, alert 기록으로 설명된다.

### 증거

- 주문/결제 상태 분포 query 결과
- Grafana payment latency와 5xx rate 캡처
- Alertmanager alert history
- payment-service와 reservation-service log
- k6 실행 결과
- Kubernetes event/log

### 의존성

```text
S0 정상 예매 E2E
↓
S3 Kafka 후속 처리 분리
↓
payment-service 지연/실패 모드 준비
↓
예약 상태 전환 기준 확정
↓
k6 또는 kubectl 기반 장애 실행
↓
상태 분포와 alert 기록 수집
↓
예약 유실 여부 확인
```

### Epic 후보

- 결제 장애 시나리오 검증
- 예약 상태 전환 관측
- 결제 장애 alert 구성

### Issue 후보

- `payment-service 지연/실패 모드를 실행하는 절차를 작성한다`
- `pending, paid, failed 상태 분포 query를 준비한다`
- `결제 지연 중 예약 유실 여부를 확인한다`
- `결제 장애 alert 조건과 firing 기록을 남긴다`
- `장애 전후 Grafana 캡처를 발표 자료에 연결한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
