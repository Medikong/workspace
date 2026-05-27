---
id: kt-cloudnative-study-scenario-s5
title: KT CloudNative Study S5 독립 배포 검증
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s5, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S5: 독립 배포 검증

### 검증 질문

한 서비스만 재배포했을 때 다른 서비스와 핵심 사용자 흐름이 계속 정상 동작하는가?

### 목표

MSA의 독립 배포 가능성을 보여준다. 서비스별 배포가 전체 시스템 장애로 이어지지 않도록 배포 단위와 검증 흐름을 분리한다.

### 범위

- 대상 서비스 1개 선택
- 단일 서비스 이미지 또는 manifest 변경
- 재배포 절차 실행
- 핵심 E2E 재검증
- PDB 또는 readiness/liveness 설정 확인

### 성공 기준

- 대상 서비스만 재배포된다.
- 재배포 후 정상 사용자 흐름이 다시 통과한다.
- 배포 중 장애가 발생하면 원인과 영향 범위를 설명할 수 있다.

### 증거

- 배포 로그
- `kubectl rollout status` 결과
- E2E 재검증 결과
- Grafana 지표 변화

### 의존성

```text
S1 정상 사용자 흐름 검증
↓
대상 서비스 선택
↓
readiness/liveness 또는 PDB 확인
↓
단일 서비스 재배포
↓
rollout 상태 확인
↓
정상 사용자 흐름 재검증
↓
결과 기록
```

### Epic 후보

- 서비스별 독립 배포 흐름
- 배포 안정성 검증
- PDB/readiness 운영 기준

### Issue 후보

- `독립 배포 검증 대상 서비스를 선택한다`
- `대상 서비스의 readiness/liveness 설정을 확인한다`
- `대상 서비스만 재배포하는 절차를 작성한다`
- `재배포 후 정상 사용자 흐름을 재검증한다`
- `배포 결과와 영향 범위를 정리한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
