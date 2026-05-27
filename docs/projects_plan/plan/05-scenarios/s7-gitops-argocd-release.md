---
id: kt-cloudnative-study-scenario-s7
title: KT CloudNative Study S7 GitOps/ArgoCD 배포 검증
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s7, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S7: GitOps/ArgoCD 배포 검증

### 검증 질문

서비스 변경 사항을 GitOps 경로로 AWS demo/QA 환경에 반영하고, ArgoCD 상태와 rollback 기준을 설명할 수 있는가?

### 목표

기본 프로젝트에서 만든 배포 기반을 심화 프로젝트의 운영 배포 흐름으로 연결한다. 단순히 manifest를 적용하는 것이 아니라, Git 변경, ArgoCD sync, 배포 smoke, rollback 기준이 하나의 운영 절차로 보이게 한다.

### 범위

- 현재 GitOps/ArgoCD 구성 위치와 gap 확인
- AWS demo/QA 환경에서 사용할 ArgoCD Application 또는 대체 배포 경로 결정
- 이미지 tag와 manifest update 기준 정리
- ArgoCD sync 또는 대체 배포 실행
- rollback 기준과 runbook 작성

### 성공 기준

- GitOps/ArgoCD 배포 경로의 현재 상태와 gap이 정리된다.
- AWS 환경에 변경 사항을 반영하는 절차가 확인된다.
- ArgoCD sync 상태 또는 단기 대체 배포 결과가 남는다.
- 실패 시 rollback 또는 수동 복구 기준이 정리된다.

### 증거

- GitOps 경로 인벤토리
- ArgoCD Application/sync 상태 캡처 또는 대체 배포 로그
- manifest diff 또는 image tag 변경 기록
- 배포 smoke 결과
- rollback runbook

### 의존성

```text
S0 AWS 시연 환경 준비
↓
GitOps/ArgoCD 현재 구성 확인
↓
AWS demo/QA 배포 경로 결정
↓
이미지 tag와 manifest update 기준 정리
↓
ArgoCD sync 또는 대체 배포 실행
↓
배포 smoke
↓
rollback 기준 정리
```

### Epic 후보

- GitOps 배포 경로 정렬
- ArgoCD sync 검증
- rollback runbook

### Issue 후보

- `현재 GitOps/ArgoCD 구성과 gap을 확인한다`
- `AWS demo/QA 환경의 ArgoCD Application 범위를 결정한다`
- `이미지 tag와 manifest update 기준을 정리한다`
- `ArgoCD sync 또는 대체 배포 smoke를 실행한다`
- `rollback 기준과 운영 runbook을 작성한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
