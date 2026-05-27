---
id: kt-cloudnative-study-prd-traceability
title: KT CloudNative Study PRD 요구사항 추적표
type: traceability-matrix
status: draft
tags: [kt-cloudnative-study, prd, traceability, requirements, backlog]
created: 2026-05-27
updated: 2026-05-27
---

# KT CloudNative Study PRD 요구사항 추적표

## 문서 목적

이 문서는 `00-PRD.md`의 기본 프로젝트와 심화 프로젝트 요구사항이 현재 마일스톤, 시나리오, Epic, workplan에 어떻게 반영되는지 확인하기 위한 추적표다.

현재 `02-PROJECT_PLAN.md`, `03-MILESTONES.md`, `05-scenarios/README.md`, `../workplans/EPICS.md`는 운영 검증 흐름을 중심으로 정리되어 있다. 따라서 PRD의 모든 필수/선택 항목이 문장 그대로 드러나지는 않는다. 이 문서는 그 사이의 매핑을 보완한다.

## 기준

- 기본 프로젝트는 대부분 완료된 상태로 보고, 심화 프로젝트의 선행 조건이자 승계 검증 대상으로 둔다.
- 기본 프로젝트 중 LGTM/관측성은 아직 미완료로 보고 심화 프로젝트 P0/P1 범위에 포함한다.
- 최종 시연 배포는 AWS에서 진행한다.
- Terraform 기반은 있으나 개발/QA 서버는 아직 없으므로, 5주 일정에서는 AWS demo/QA 단일 환경을 우선한다.
- GitOps와 ArgoCD는 배포 방식과 운영 검증 흐름의 핵심 축으로 명시한다.
- 모든 항목은 구현 여부보다 발표 가능한 증거가 있는지를 기준으로 관리한다.

## 추적 상태 기준

| 상태 | 의미 |
| --- | --- |
| `Covered` | 현재 Scenario/Epic/workplan에서 직접 다룬다. |
| `Inherited` | 기본 프로젝트 완료분으로 승계하되, 증거 또는 기준선 확인이 필요하다. |
| `Gap` | 문서 또는 workplan에 명시가 부족해 추가 반영이 필요하다. |
| `Optional` | 선택 과업이며 P0 완료 후 여력에 따라 다룬다. |
| `Deferred` | 5주 범위에서는 후속 과제로 둔다. |

## 기본 프로젝트 승계 항목

| PRD 영역 | 필수/선택 | 현재 처리 | 연결 Scenario | 연결 Epic/workplan | 상태 |
| --- | --- | --- | --- | --- | --- |
| 마이크로서비스 분할 전략 | 필수 | 도메인별 서비스 경계, 통신 방식, DB per Service 원칙은 기본 프로젝트 완료분으로 승계하고 정상 흐름 기준선에서 확인한다. | S1 | `epic:e2e-baseline`, `10-e2e-baseline.yaml` | Inherited |
| 마이크로서비스 분할 전략 | 선택 | Kafka 이벤트 통신과 Event Sourcing은 발표 핵심이 아니라면 후속 과제로 둔다. | S6 | `epic:evidence-presentation` | Optional |
| 서비스 디스커버리/로드밸런싱 | 필수 | Kubernetes DNS, ClusterIP, Gateway/Ingress 라우팅, 장애 격리 문서를 Sprint 1 기준선으로 확인한다. | S1, S3 | `epic:e2e-baseline`, `epic:traffic-policy` | Covered |
| 서비스 디스커버리/로드밸런싱 | 선택 | Circuit Breaker/Fallback, Gateway Rate Limiting은 장애 관측 결과 확보 후 선택한다. | S2, S3 | `epic:lgtm-observability`, `epic:traffic-policy` | Optional |
| 통합 테스트/운영 관리 | 필수 | pytest/CI, Postman E2E는 기본 검증 루프로 승계하고, Prometheus/Grafana는 심화 관측성 범위로 끌어올린다. | S1, S2 | `epic:e2e-baseline`, `epic:lgtm-observability` | Covered |
| 통합 테스트/운영 관리 | 선택 | Testcontainers, Newman CI, Kafka exporter lag 알림은 P0 증거 확보 후 선택한다. | S2, S6 | `epic:lgtm-observability`, `epic:evidence-presentation` | Optional |

## 심화 프로젝트 항목

| PRD 영역 | 필수/선택 | 현재 처리 | 연결 Scenario | 연결 Epic/workplan | 상태 |
| --- | --- | --- | --- | --- | --- |
| 마이크로서비스 확장성 설계 | 필수 | 서비스별 독립 배포, Gateway/Mesh 협업 구조, PDB는 Sprint 2~3의 배포 안정성 검증으로 다룬다. | S5, S7 | `epic:delivery-reliability`, `epic:gitops-argocd-release` | Covered |
| 마이크로서비스 확장성 설계 | 선택 | KEDA와 ArgoCD Rollouts Canary는 선택 과업이다. 단, ArgoCD 자체 GitOps 동기화는 배포 기반으로 별도 P0/P1에 둔다. | S7 | `epic:gitops-argocd-release`, `epic:delivery-reliability` | Optional |
| DevSecOps 파이프라인 강화 | 필수 | SonarQube, Trivy, Slack 보안 리포트 중 Trivy를 P0로 두고 SonarQube/Slack은 일정에 따라 P1로 둔다. | S4 | `epic:devsecops` | Covered |
| DevSecOps 파이프라인 강화 | 선택 | OWASP ZAP, OPA Gatekeeper는 5주 범위에서는 후속 또는 부록 후보로 둔다. | S4, S6 | `epic:devsecops` | Optional |
| 보안 아키텍처 재설계 | 필수 | RBAC, ServiceAccount, NetworkPolicy는 통신/보안 정책 검증으로 다룬다. | S3, S4 | `epic:traffic-policy`, `epic:devsecops` | Covered |
| 보안 아키텍처 재설계 | 선택 | Incident Response 자동화와 Falco는 후속 과제로 둔다. | S6 | `epic:evidence-presentation` | Deferred |

## PRD 외 실행 필수 항목

| 항목 | 필요한 이유 | 연결 Scenario | 연결 Epic/workplan | 상태 |
| --- | --- | --- | --- | --- |
| AWS demo/QA 환경 | 최종 시연이 AWS에서 진행되므로 모든 검증의 실행 기반이 된다. | S0 | `epic:aws-demo-environment`, `05-aws-demo-environment.yaml` | Covered |
| GitOps/ArgoCD 배포 경로 | 기본 프로젝트에서 심화 프로젝트로 이어지는 운영 배포 흐름을 설명해야 한다. | S7 | `epic:gitops-argocd-release`, `15-gitops-argocd-release.yaml` | Covered |
| LGTM 관측성 | 기본 프로젝트에서 미완료된 관측성 축이며 심화 프로젝트의 핵심 증거가 된다. | S2 | `epic:lgtm-observability`, `20-observability.yaml` | Covered |
| 요구사항 커버리지 관리 | PRD의 필수/선택 항목이 누락되지 않도록 Sprint Planning 기준으로 관리해야 한다. | S6 | `epic:foundation`, `00-foundation.yaml` | Covered |
| 발표 증거 인덱스 | 구현 결과를 발표 가능한 주장과 증거로 연결해야 한다. | S6 | `epic:evidence-presentation`, `60-evidence-presentation.yaml` | Covered |

## 우선순위 재정의

### P0

- AWS demo/QA 환경 접근과 기본 smoke
- GitOps/ArgoCD 동기화 경로 또는 명확한 대체 배포 경로
- 정상 사용자 흐름 E2E 기준선
- Prometheus/Grafana 기반 메트릭 관측
- 로그 중앙화 또는 최소 로그 조회 경로
- NetworkPolicy 또는 Mesh 정책 기반 통신 차단
- Trivy manifest scan 실패 케이스
- 발표용 증거 캡처

### P1

- Tempo/Jaeger 분산 추적
- SonarQube 품질 게이트
- Slack 알림
- PDB와 독립 배포 검증
- Circuit Breaker/Fallback

### P2 또는 후속

- KEDA
- ArgoCD Rollouts Canary
- OWASP ZAP
- OPA Gatekeeper
- Falco
- Incident Response 자동화
- Event Sourcing

## Sprint Planning 사용법

1. 이 문서에서 `Covered`가 아닌 `Gap`을 먼저 확인한다.
2. `Inherited` 항목은 완료 여부가 아니라 발표 증거가 있는지 확인한다.
3. P0 항목은 2026-06-19(금)까지 실행 결과 또는 후속 과제 판단을 끝낸다.
4. 선택 과업은 P0 증거가 확보된 뒤에만 스프린트에 올린다.
5. 최종 발표에서는 완료 기능, 검증 결과, 미완성 후속 과제를 분리해서 설명한다.
