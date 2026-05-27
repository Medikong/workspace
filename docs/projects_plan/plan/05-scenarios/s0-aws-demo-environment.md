---
id: kt-cloudnative-study-scenario-s0
title: KT CloudNative Study S0 AWS 시연 환경 준비
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s0, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S0: AWS 시연 환경 준비

### 검증 질문

Terraform 기반만 있는 상태에서 최종 시연에 사용할 AWS demo/QA 환경을 준비할 수 있는가?

### 목표

최종 발표의 실제 시연 배포 환경을 AWS로 고정한다. 개발 서버와 QA 서버가 아직 없으므로, 5주 일정에서는 분리된 dev/QA 체계보다 demo/QA 단일 환경을 먼저 만든다.

### 범위

- 현재 Terraform 구성 범위 확인
- demo/QA 환경 최소 토폴로지 결정
- AWS 클러스터 접근 방법 정리
- 이미지 배포 경로 확인
- 기본 배포 smoke 실행

### 성공 기준

- Terraform 기반으로 만들 수 있는 AWS 리소스 범위가 확인된다.
- demo/QA 환경에 접근할 수 있다.
- 핵심 서비스 또는 최소 smoke 대상이 AWS 환경에 배포된다.
- 실패 시 로컬 검증을 백업으로 사용할 기준이 정리된다.

### 증거

- Terraform plan/apply 결과 또는 검토 기록
- AWS 리소스 목록
- 클러스터 접근 로그
- 배포 smoke 결과
- 환경 리스크와 백업 계획

### 의존성

```text
Terraform 구성 인벤토리
↓
demo/QA 환경 범위 결정
↓
AWS 접근/권한 확인
↓
클러스터 또는 서버 접근 확인
↓
이미지 배포 경로 확인
↓
기본 smoke 실행
↓
AWS 시연 가능 여부 기록
```

### Epic 후보

- AWS 시연 환경 준비
- 배포 smoke 검증
- 환경 리스크 관리

### Issue 후보

- `현재 Terraform 구성과 생성 가능한 AWS 리소스를 확인한다`
- `AWS demo/QA 환경 최소 범위를 결정한다`
- `AWS 접근 권한과 클러스터 접속 절차를 확인한다`
- `이미지 배포 경로와 registry 기준을 정리한다`
- `AWS 환경 기본 배포 smoke를 실행한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
