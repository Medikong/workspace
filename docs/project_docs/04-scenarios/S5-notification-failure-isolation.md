---
id: S5
title: 공연 티켓 예매 S5 알림 장애 격리
type: scenario
status: draft
priority: P0
phase: Sprint 3
tags: [ticketing, scenario, notification, isolation, kafka]
created: 2026-05-27
updated: 2026-05-27
---

## S5: 알림 장애 격리

### 검증 질문

notification-service 장애 중에도 예약, 결제, 티켓 발행으로 이어지는 핵심 예매 흐름이 유지되는가?

### 목표

알림은 사용자 경험에 중요하지만 결제 완료와 티켓 발행을 막아서는 안 된다. notification-service 장애를 주입해 Kafka 후속 처리 분리와 장애 격리가 실제로 동작하는지 확인한다.

### 범위

- notification-service scale-down, crash, 응답 지연 중 하나의 장애 방식
- 장애 중 core flow 성공률 측정
- notification retry 또는 consumer lag 확인
- 장애 회복 후 누락된 알림 처리 여부 확인
- 관련 service log와 Grafana 패널 캡처

### 성공 기준

- notification 장애 중 core flow 성공률이 정상 대비 95% 이상이다.
- 예약, 결제, 티켓 발행은 성공 상태로 남는다.
- notification 실패는 retry, lag, error log 중 하나로 관측된다.
- 장애 회복 후 처리된 알림 또는 미처리 알림의 상태가 설명된다.

### 증거

- 장애 전후 Grafana core flow success rate 캡처
- notification-service log와 consumer lag
- reservation, payment, ticket DB 최종 상태
- Kubernetes event/log
- k6 결과
- Loki query 결과

### 의존성

```text
S0 정상 예매 E2E
↓
S3 Kafka 후속 처리 분리
↓
notification 장애 주입 방식 확정
↓
core flow success rate metric 준비
↓
장애 중 예매 흐름 실행
↓
장애 회복과 retry 상태 확인
↓
격리 결과 기록
```

### Epic 후보

- 알림 장애 격리 검증
- notification retry와 lag 관측
- core flow 성공률 비교

### Issue 후보

- `notification 장애 주입 절차를 작성한다`
- `core flow 성공률과 notification error metric을 구분한다`
- `장애 중 예매 흐름 k6 시나리오를 실행한다`
- `장애 회복 후 알림 처리 상태를 확인한다`
- `Grafana와 Loki 증거를 발표용으로 고정한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
