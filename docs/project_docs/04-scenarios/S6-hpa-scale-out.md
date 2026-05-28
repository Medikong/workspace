---
id: S6
title: 공연 티켓 예매 S6 HPA scale-out
type: scenario
status: draft
priority: P0
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, hpa, kubernetes, scale-out]
created: 2026-05-27
updated: 2026-05-27
---

## S6: HPA scale-out

### 검증 질문

티켓 오픈 부하가 증가한 뒤 60초 이내에 Kubernetes HPA의 desired replica가 증가하는가?

### 목표

트래픽 폭발 대응을 "pod가 늘었다"가 아니라 측정 가능한 운영 증거로 검증한다. 부하 증가, metric 수집, HPA 판단, replica 증가, latency 변화가 같은 시간축에서 설명되어야 한다.

### 범위

- 예매 API 또는 reservation-service 대상 부하
- CPU, memory 또는 custom metric 기반 HPA 확인
- desired/current replica 변화 관측
- scale-out 전후 P95/P99와 5xx rate 비교
- Kubernetes event와 HPA describe 기록

### 성공 기준

- 부하 증가 후 60초 이내 desired replica가 증가한다.
- replica 증가 시점이 Grafana 또는 Kubernetes event로 확인된다.
- scale-out 이후 P99와 5xx rate 변화가 기록된다.
- scale-out이 좌석 정합성을 깨지 않는다.

### 증거

- `kubectl describe hpa` 출력
- replica graph Grafana 캡처
- Kubernetes event/log
- k6 부하 테스트 결과
- Prometheus CPU, memory, request latency query 결과
- DB 좌석 정합성 확인 결과

### 의존성

```text
S0 정상 예매 E2E
↓
S2 티켓 오픈 피크
↓
서비스별 resource request/limit 정리
↓
HPA manifest 또는 Helm values 준비
↓
metric-server와 Prometheus 수집 확인
↓
부하 실행
↓
replica 변화와 latency 변화 기록
```

### Epic 후보

- Kubernetes 자동 확장 검증
- HPA metric과 event 증거 관리
- scale-out 전후 성능 비교

### Issue 후보

- `reservation-service 또는 gateway 경로의 HPA 기준을 정한다`
- `HPA describe와 event 수집 절차를 작성한다`
- `scale-out 유도 k6 부하를 실행한다`
- `replica 변화와 P99 변화를 같은 시간축으로 캡처한다`
- `scale-out 후 좌석 정합성 결과를 확인한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
