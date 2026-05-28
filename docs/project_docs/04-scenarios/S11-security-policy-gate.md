---
id: S11
title: 공연 티켓 예매 S11 보안 스캔과 정책
type: scenario
status: draft
priority: P2
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, security, trivy, networkpolicy, rbac]
created: 2026-05-27
updated: 2026-05-27
---

## S11: 보안 스캔과 정책

### 검증 질문

Critical 취약점 또는 비정상 접근을 배포 전후 단계에서 차단하고, 차단 근거를 증거로 남길 수 있는가?

### 목표

보안은 기능 완성 후 덧붙이는 항목이 아니라 배포와 런타임의 품질 게이트로 다룬다. Trivy 스캔, NetworkPolicy, RBAC 중 발표 가능한 범위를 정하고 정상 접근과 비정상 접근의 차이를 기록한다.

### 범위

- Trivy image 또는 manifest scan
- Critical 취약점 배포 차단 기준
- NetworkPolicy allow/deny 검증
- RBAC 권한 범위 확인
- 정상 예매 흐름에 대한 영향 확인

### 성공 기준

- Critical 취약점이 있는 artifact는 배포 또는 품질 게이트에서 차단된다.
- 허용되지 않은 service 간 접근은 실패한다.
- 정상 예매 흐름은 정책 적용 후에도 성공한다.
- scan report와 allow/deny log가 발표 증거로 남는다.

### 증거

- Trivy scan report
- CI 또는 품질 게이트 실패 기록
- NetworkPolicy allow/deny log
- RBAC 권한 확인 결과
- 정책 적용 후 Newman 정상 예매 결과
- Kubernetes event/log

### 의존성

```text
S0 정상 예매 E2E
↓
배포 manifest와 image 기준 확정
↓
Trivy 스캔 기준 정리
↓
NetworkPolicy 또는 RBAC 정책 적용
↓
정상 접근과 비정상 접근 테스트
↓
차단 결과와 정상 예매 영향 기록
```

### Epic 후보

- 보안 스캔 품질 게이트
- NetworkPolicy 접근 통제 검증
- 정책 적용 후 정상 흐름 회귀 검증

### Issue 후보

- `Trivy scan 기준과 실패 예시를 준비한다`
- `Critical 취약점 차단 결과를 CI 로그로 남긴다`
- `허용되지 않은 service 간 접근 테스트를 작성한다`
- `NetworkPolicy allow/deny 결과를 기록한다`
- `정책 적용 후 정상 예매 E2E를 다시 실행한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
