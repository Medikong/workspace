---
id: kt-cloudnative-study-scenario-s3
title: KT CloudNative Study S3 비인가 통신 차단
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s3, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S3: 비인가 통신 차단

### 검증 질문

허용되지 않은 서비스 간 직접 호출을 NetworkPolicy 또는 Service Mesh 정책으로 차단할 수 있는가?

### 목표

서비스 간 통신 경계를 정책으로 관리할 수 있음을 보여준다. MSA에서 모든 서비스가 모든 서비스에 접근 가능한 상태를 줄이고, 의도한 호출 경로만 허용한다.

### 범위

- 허용할 호출 관계 정의
- 차단할 호출 관계 정의
- NetworkPolicy 또는 Mesh AuthorizationPolicy 작성
- 허용 호출 성공 확인
- 비인가 호출 실패 확인

### 성공 기준

- 허용된 서비스 간 호출은 성공한다.
- 허용되지 않은 서비스 간 호출은 실패한다.
- 실패 결과가 로그 또는 명령 결과로 남는다.

### 증거

- 정책 manifest
- 허용 호출 결과
- 차단 호출 결과
- Kubernetes event 또는 서비스 로그

### 의존성

```text
S1 정상 사용자 흐름 검증
↓
서비스 호출 관계 정리
↓
허용/차단 매트릭스 작성
↓
NetworkPolicy 또는 Mesh 정책 작성
↓
허용 호출 검증
↓
차단 호출 검증
↓
결과 기록
```

### Epic 후보

- 서비스 호출 경계 설계
- NetworkPolicy 적용
- Service Mesh 통신 정책 검증

### Issue 후보

- `서비스 간 허용/차단 호출 매트릭스를 작성한다`
- `NetworkPolicy 기본 정책을 작성한다`
- `허용된 서비스 호출이 성공하는지 확인한다`
- `비인가 서비스 호출이 실패하는지 확인한다`
- `정책 적용 전후 결과를 발표 자료로 정리한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
