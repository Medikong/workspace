---
id: kt-cloudnative-study-scenarios
title: KT CloudNative Study 검증 시나리오
type: scenario-index
status: draft
tags: [kt-cloudnative-study, scenarios, validation, dependency, observability, security]
created: 2026-05-27
updated: 2026-05-27
---

# KT CloudNative Study 검증 시나리오

## 문서 목적

이 폴더는 심화 프로젝트에서 "무엇을 만들었는가"가 아니라 "무엇을 검증했는가"를 보여주기 위한 시나리오 문서다.

- `../02-PROJECT_PLAN.md`: 프로젝트 목표와 범위
- `../03-MILESTONES.md`: 5주 일정별 도착점
- `README.md`: 검증할 운영 상황과 성공 기준
- `../../workplans/README.md`: 시나리오를 실제 이슈 후보로 분해하는 작성 규칙

시나리오는 이슈보다 상위 개념이다. 하나의 시나리오는 여러 Epic과 Issue로 나뉘며, 각 Issue 사이에는 선후 의존성이 있다.

## 시나리오 작성 원칙

- 기능 이름이 아니라 검증할 상황으로 작성한다.
- 성공 케이스와 실패 케이스를 함께 생각한다.
- 검증 결과는 로그, 테스트 결과, Grafana 캡처, CI 결과, 정책 차단 결과 중 하나 이상으로 남긴다.
- 발표에서 보여줄 수 없는 작업은 우선순위를 낮춘다.
- 시나리오마다 선행 작업과 후속 작업을 명확히 둔다.

## 시나리오 요약

| ID | 시나리오 | 목표 | 우선순위 | 목표 Sprint | 발표 가치 |
| --- | --- | --- | --- | --- | --- |
| [S0](s0-aws-demo-environment.md) | AWS 시연 환경 준비 | Terraform 기반을 실제 demo/QA 환경으로 연결한다. | P0 | Phase 0-Sprint 1 | 최종 시연의 기반 |
| [S1](s1-happy-path-baseline.md) | 정상 사용자 흐름 검증 | 기준선 E2E 흐름을 확보한다. | P0 | Phase 0-Sprint 1 | 모든 검증의 출발점 |
| [S2](s2-failure-observability.md) | 서비스 장애 관측 | 장애 영향 범위를 지표와 로그로 설명한다. | P0 | Sprint 2-3 | 관측성 핵심 시연 |
| [S3](s3-unauthorized-traffic-block.md) | 비인가 통신 차단 | 서비스 간 접근 경계를 정책으로 강제한다. | P0 | Sprint 2-3 | 보안/통신 통제 시연 |
| [S4](s4-security-scan-gate.md) | 보안 스캔 차단 | 위험한 manifest를 배포 전에 실패시킨다. | P0 | Sprint 2-3 | DevSecOps 시연 |
| [S5](s5-independent-deployment.md) | 독립 배포 검증 | 한 서비스 배포가 전체 흐름을 깨지 않음을 확인한다. | P1 | Sprint 3 | 운영 안정성 시연 |
| [S7](s7-gitops-argocd-release.md) | GitOps/ArgoCD 배포 검증 | 변경 사항이 GitOps 경로로 AWS 환경에 반영되는지 확인한다. | P0 | Sprint 1-2 | 운영 배포 흐름 시연 |
| [S6](s6-presentation-rehearsal.md) | 발표 리허설 | 결과를 하나의 이야기로 설명한다. | P0 | Final Prep | 최종 전달 품질 |

## 시나리오 간 의존성

```text
S0 AWS 시연 환경 준비
├─ S1 정상 사용자 흐름 검증
└─ S7 GitOps/ArgoCD 배포 검증

S1 정상 사용자 흐름 검증
├─ S2 서비스 장애 관측
├─ S3 비인가 통신 차단
└─ S5 독립 배포 검증

S7 GitOps/ArgoCD 배포 검증
├─ S1 정상 사용자 흐름 검증
└─ S5 독립 배포 검증

S4 보안 스캔 차단
└─ S6 발표 리허설

S2 서비스 장애 관측
S3 비인가 통신 차단
S5 독립 배포 검증
S7 GitOps/ArgoCD 배포 검증
└─ S6 발표 리허설
```

## 우선순위 기준

| 우선순위 | 기준 |
| --- | --- |
| P0 | 발표 핵심 시나리오이거나 다른 시나리오의 선행 조건 |
| P1 | 운영 가능성을 강화하지만 일부 축소 가능 |
| P2 | 시간이 남을 때 추가하면 좋은 확장 과제 |

## 이슈 분해 규칙

시나리오를 이슈로 쪼갤 때는 다음 필드를 유지한다.

```text
Scenario:
Epic:
Goal:
Task:
Depends on:
Blocks:
Evidence:
Definition of Done:
```

좋은 이슈는 구현 작업과 검증 작업이 함께 보인다.

```text
Scenario: S2 서비스 장애 관측
Epic: Grafana 장애 관측 대시보드
Task: Grafana에 서비스별 에러율 패널을 구성한다
Depends on: Prometheus가 서비스 메트릭을 수집한다
Evidence: 장애 주입 전후 에러율 패널 캡처
Definition of Done: 장애 주입 후 에러율 변화가 캡처되고 발표 자료에 연결된다
```

## 다음 단계

- `../../workplans/*.yaml`에서 각 시나리오를 실제 이슈 후보로 분해한다.
- GitHub Projects에는 `Scenario`, `Epic`, `Depends on`, `Blocks`, `Evidence` 필드를 반영한다.
- Sprint Planning 때 P0 시나리오부터 이슈를 선택한다.
