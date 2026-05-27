---
id: kt-cloudnative-study-scenario-s1
title: KT CloudNative Study S1 정상 사용자 흐름 검증
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s1, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S1: 정상 사용자 흐름 검증

### 검증 질문

환자 조회, 예약 생성, 알림 발송 같은 핵심 사용자 흐름이 기준 환경에서 정상 동작하는가?

### 목표

모든 운영 시나리오의 기준선을 만든다. 장애, 보안, 배포 검증은 정상 흐름이 먼저 재현되어야 의미가 있다.

### 범위

- 서비스 DNS 확인
- Gateway/Ingress 라우팅 확인
- 핵심 API 요청 흐름 확인
- 테스트 데이터 준비
- E2E 또는 수동 검증 절차 작성

### 성공 기준

- 핵심 사용자 흐름이 한 번 이상 성공한다.
- 요청 경로와 관련 서비스가 문서화된다.
- 같은 절차를 다른 팀원이 따라 실행할 수 있다.

### 증거

- API 응답 결과
- 서비스 로그
- E2E 실행 결과 또는 수동 검증 기록
- 요청 흐름 다이어그램

### 의존성

```text
S0 AWS 시연 환경 준비
↓
서비스 배포 manifest 준비
↓
서비스 DNS/ClusterIP 확인
↓
Gateway/Ingress 라우팅 확인
↓
테스트 데이터 준비
↓
정상 사용자 흐름 실행
↓
결과 기록
```

### Epic 후보

- 기준 배포 환경 점검
- 핵심 사용자 흐름 E2E 구성
- 발표용 요청 흐름 정리

### Issue 후보

- `서비스별 Kubernetes Service와 DNS 이름을 확인한다`
- `Gateway/Ingress 경로 라우팅을 검증한다`
- `정상 사용자 흐름 테스트 데이터를 준비한다`
- `정상 사용자 흐름 E2E 절차를 작성한다`
- `정상 흐름 결과를 캡처하고 문서화한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
