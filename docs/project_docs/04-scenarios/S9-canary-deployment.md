---
id: S9
title: 공연 티켓 예매 S9 Canary 배포
type: scenario
status: draft
priority: P1
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, canary, argocd, rollout]
created: 2026-05-27
updated: 2026-05-27
---

## S9: Canary 배포

### 검증 질문

신규 버전으로 일부 트래픽을 보냈을 때 에러율 증가가 기존 버전 대비 +1%p 이하인가?

### 목표

티켓 오픈 서비스에서 배포 안정성을 검증한다. 신규 버전이 전체 사용자에게 한 번에 노출되지 않고, 제한된 트래픽에서 latency와 error를 비교한 뒤 계속 진행할 수 있는지 확인한다.

### 범위

- Argo CD와 Istio 또는 Argo Rollouts 기반 canary 경로
- stable/canary 버전별 metric 분리
- 트래픽 비율 변화 기록
- canary error delta와 latency 비교
- rollout 상태와 이벤트 기록

### 성공 기준

- canary 버전 에러율이 stable 대비 +1%p 이하이다.
- 버전별 latency와 5xx rate가 Grafana에서 구분된다.
- traffic split ratio와 rollout 진행 상태가 기록된다.
- canary 중 정상 예매 E2E가 유지된다.

### 증거

- 버전별 Grafana metric 캡처
- Argo CD sync 또는 rollout 기록
- traffic split 설정과 이벤트
- Newman 정상 예매 결과
- Kubernetes event/log

### 의존성

```text
S0 정상 예매 E2E
↓
배포 환경과 GitOps 경로 준비
↓
stable/canary 버전 구분 label 준비
↓
traffic split 또는 rollout 설정
↓
canary 중 정상 예매와 metric 비교
↓
진행 상태와 증거 기록
```

### Epic 후보

- Canary 배포 검증
- 버전별 운영 지표 비교
- GitOps 배포 증거 관리

### Issue 후보

- `stable/canary 버전 label과 metric 구분 기준을 정한다`
- `Canary traffic split 설정을 준비한다`
- `Canary 중 정상 예매 Newman 결과를 기록한다`
- `버전별 error rate와 latency를 Grafana에서 비교한다`
- `rollout 진행 상태와 event를 캡처한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
