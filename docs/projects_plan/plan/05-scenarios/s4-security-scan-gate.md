---
id: kt-cloudnative-study-scenario-s4
title: KT CloudNative Study S4 보안 스캔 차단
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s4, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S4: 보안 스캔 차단

### 검증 질문

privileged 컨테이너, root 실행, 위험한 securityContext 같은 manifest 위반을 배포 전에 차단할 수 있는가?

### 목표

DevSecOps 파이프라인의 가치를 보여준다. 보안 검사는 체크리스트가 아니라, 위험 설정이 들어왔을 때 실제로 실패하는 자동화로 증명한다.

### 범위

- Trivy config scan 기준 정의
- 정상 manifest 스캔
- 위험 manifest 샘플 작성
- 실패 케이스 확인
- CI 또는 로컬 실행 절차 정리

### 성공 기준

- 정상 manifest는 통과한다.
- 위험 manifest는 실패한다.
- 실패 원인과 수정 방향을 설명할 수 있다.

### 증거

- Trivy 실행 결과
- 위험 manifest 샘플
- CI 로그 또는 로컬 실행 로그
- 수정 전후 비교

### 의존성

```text
Kubernetes manifest 위치 확인
↓
Trivy config scan 실행 방식 결정
↓
정상 manifest 스캔
↓
위험 manifest 샘플 작성
↓
실패 케이스 검증
↓
결과 기록
```

### Epic 후보

- Kubernetes manifest 보안 스캔
- CI 보안 게이트
- 보안 결과 보고

### Issue 후보

- `Trivy config scan 실행 방식을 정한다`
- `현재 Kubernetes manifest를 스캔한다`
- `privileged 컨테이너 실패 샘플을 만든다`
- `보안 스캔 실패 결과를 캡처한다`
- `스캔 결과를 README 또는 발표 자료에 정리한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
