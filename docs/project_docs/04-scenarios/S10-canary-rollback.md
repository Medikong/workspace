---
id: S10
title: 공연 티켓 예매 S10 Canary 롤백
type: scenario
status: draft
priority: P1
phase: Sprint 3
tags: [ticketing, scenario, rollback, alertmanager, argocd]
created: 2026-05-27
updated: 2026-05-27
---

## S10: Canary 롤백

### 검증 질문

Canary 버전 이상을 감지한 뒤 3분 이내에 이전 버전으로 복구되는가?

### 목표

배포 안정성의 마지막 판단 기준을 검증한다. 신규 버전의 에러율이나 latency가 기준을 넘었을 때 alert가 발생하고, rollback 이벤트와 복구 후 정상 상태를 증거로 남긴다.

### 범위

- canary 실패 조건 정의
- Alertmanager 또는 rollout 분석 조건 확인
- rollback 실행과 이벤트 기록
- 복구 후 정상 예매 E2E 확인
- rollback time 측정

### 성공 기준

- 이상 감지 후 3분 이내 이전 버전으로 복구된다.
- Alertmanager 기록 또는 rollout 분석 실패 기록이 남는다.
- rollback 후 정상 예매 E2E가 성공한다.
- canary 버전 트래픽이 제거되거나 stable로 되돌아간 상태가 확인된다.

### 증거

- alert firing과 resolved 기록
- Argo CD 또는 rollout rollback 이벤트
- Kubernetes event/log
- rollback 전후 Grafana 캡처
- rollback 후 Newman 결과

### 의존성

```text
S9 Canary 배포
↓
S7 관측성 추적
↓
canary 실패 조건 준비
↓
이상 버전 배포 또는 실패 metric 주입
↓
alert/analysis 실패 확인
↓
rollback 실행
↓
복구 시간과 정상 예매 결과 기록
```

### Epic 후보

- Canary rollback 검증
- 배포 이상 감지와 알림
- 복구 시간 증거 관리

### Issue 후보

- `canary 실패 기준과 rollback 조건을 정의한다`
- `rollback 전후 버전별 metric을 캡처한다`
- `Alertmanager firing과 resolved 기록을 확보한다`
- `rollback event와 소요 시간을 기록한다`
- `복구 후 정상 예매 E2E를 실행한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
