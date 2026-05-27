---
id: kt-cloudnative-study-epics
title: KT CloudNative Study Epic 정의
type: epic-plan
status: draft
tags: [kt-cloudnative-study, epic, issue-backlog, linear, github-projects]
created: 2026-05-27
updated: 2026-05-27
---

# KT CloudNative Study Epic 정의

## 문서 목적

이 문서는 `../plan/05-scenarios/README.md`를 실제 이슈 백로그로 분해하기 전에 Epic 단위를 정의한다.

- Scenario: 검증할 운영 상황
- Epic: 시나리오를 완성하기 위한 큰 작업 묶음
- Issue: 한 사람이 맡아 처리할 수 있는 실제 작업 단위
- Dependency: 이슈 사이의 선후 관계

이번 프로젝트에서는 기존 Linear workplan 스키마를 확장하지 않고, Epic은 `labels`에 `epic:...` 형태로 표현한다.

## Epic 설계 원칙

- Epic은 기능 이름보다 검증 목적을 기준으로 잡는다.
- 하나의 Epic은 여러 Scenario에 걸칠 수 있다.
- 하나의 Issue는 하나의 주요 Epic에 속하게 한다.
- Epic마다 남길 증거와 완료 기준을 둔다.
- 2026-06-19까지 실험 결과를 남길 수 없는 Epic은 범위를 줄이거나 후속 과제로 둔다.

## Epic 요약

| Epic ID | 라벨 | 이름 | 목적 | 주요 Scenario | 목표 Sprint |
| --- | --- | --- | --- | --- | --- |
| E1 | `epic:foundation` | 기준선과 요구사항 정렬 | 정상 흐름과 MVP 범위를 고정한다. | S1, S6 | Phase 0 |
| E2 | `epic:aws-demo-environment` | AWS 시연 환경 준비 | Terraform 기반을 demo/QA 시연 환경으로 연결한다. | S0, S1 | Phase 0-Sprint 1 |
| E3 | `epic:e2e-baseline` | 핵심 사용자 흐름 검증 | 모든 운영 검증의 기준 E2E를 만든다. | S1, S5 | Sprint 1 |
| E4 | `epic:lgtm-observability` | LGTM 장애 관측성 확보 | 장애 영향 범위를 지표, 로그, 가능하면 트레이스로 설명한다. | S2, S6 | Sprint 1-3 |
| E5 | `epic:traffic-policy` | 통신 경계와 Mesh 정책 | 허용/차단 통신을 정책으로 검증한다. | S3 | Sprint 2-3 |
| E6 | `epic:devsecops` | 보안 스캔과 품질 게이트 | 위험한 변경을 배포 전에 차단한다. | S4 | Sprint 2-3 |
| E7 | `epic:delivery-reliability` | 독립 배포와 가용성 | 단일 서비스 배포와 최소 가용성을 검증한다. | S5 | Sprint 3 |
| E8 | `epic:evidence-presentation` | 증거 정리와 발표 | 결과를 하나의 검증 이야기로 만든다. | S6 | Final Prep |
| E9 | `epic:gitops-argocd-release` | GitOps/ArgoCD 배포 검증 | 변경 사항이 GitOps 경로로 AWS 환경에 반영되는지 검증한다. | S7, S5 | Sprint 1-3 |

## E1: 기준선과 요구사항 정렬

### 목적

PRD 요구사항, 5주 일정, 팀의 실제 역량을 맞춰 MVP 범위와 우선순위를 고정한다.

### 포함 범위

- PRD 요구사항 재분류
- Must/Should/Could/Won't 범위 합의
- 핵심 시나리오 우선순위 확정
- 팀 역할과 소유 영역 정리
- 공통 파일 수정 규칙 정리

### 제외 범위

- 실제 기능 구현
- 세부 이슈 전체 생성
- 장기 로드맵 확장

### 연결 Scenario

- S1 정상 사용자 흐름 검증
- S6 발표 리허설

### 완료 기준

- MVP 범위가 팀 단위로 합의되어 있다.
- P0 Scenario가 확정되어 있다.
- Sprint 1에서 처리할 Ready 이슈 후보가 정리되어 있다.

### 증거

- `../plan/02-PROJECT_PLAN.md`
- `../plan/03-MILESTONES.md`
- Sprint Planning 기록

### Issue 후보

- `PRD 요구사항을 Must/Should/Could/Won't로 재분류한다`
- `P0 검증 시나리오를 확정한다`
- `역할과 소유 영역을 정리한다`
- `공통 manifest와 문서 수정 규칙을 정한다`

## E2: AWS 시연 환경 준비

### 목적

Terraform 기반만 있는 상태에서 최종 시연에 사용할 AWS demo/QA 환경을 준비한다.

### 포함 범위

- Terraform 구성 인벤토리
- demo/QA 환경 최소 범위 결정
- AWS 접근 권한과 클러스터 접속 절차
- 이미지 배포 경로와 registry 기준
- 기본 배포 smoke

### 제외 범위

- 완전한 dev/QA/prod 환경 분리
- 장기 운영용 AWS 보안/비용 최적화
- 모든 P0 시나리오를 처음부터 AWS에서만 실행하도록 강제

### 연결 Scenario

- S0 AWS 시연 환경 준비
- S1 정상 사용자 흐름 검증

### 완료 기준

- AWS demo/QA 환경의 최소 범위가 정해져 있다.
- Terraform으로 만들 수 있는 리소스와 부족한 부분이 구분되어 있다.
- AWS 환경 접근과 기본 배포 smoke 결과가 남아 있다.

### 증거

- Terraform plan/apply 결과 또는 검토 기록
- AWS 리소스 목록
- 클러스터 접근 로그
- 배포 smoke 결과
- 환경 리스크와 백업 계획

### Issue 후보

- `현재 Terraform 구성과 생성 가능한 AWS 리소스를 확인한다`
- `AWS demo/QA 환경 최소 범위를 결정한다`
- `AWS 접근 권한과 클러스터 접속 절차를 확인한다`
- `이미지 배포 경로와 registry 기준을 정리한다`
- `AWS 환경 기본 배포 smoke를 실행한다`

## E3: 핵심 사용자 흐름 검증

### 목적

정상 사용자 흐름을 먼저 재현해 장애, 보안, 배포 검증의 기준선을 만든다.

### 포함 범위

- 서비스 DNS 확인
- Gateway/Ingress 라우팅 확인
- 테스트 데이터 준비
- 정상 사용자 흐름 E2E 또는 수동 검증 절차
- 요청 흐름 다이어그램

### 제외 범위

- 전체 서비스의 모든 API 테스트
- 복잡한 부하 테스트
- Canary나 자동 롤백 검증

### 연결 Scenario

- S1 정상 사용자 흐름 검증
- S5 독립 배포 검증

### 완료 기준

- 정상 사용자 흐름이 한 번 이상 성공한다.
- 같은 절차를 다른 팀원이 재현할 수 있다.
- 이후 장애/배포 검증에서 기준선으로 재사용할 수 있다.

### 증거

- E2E 실행 결과
- API 응답 결과
- 서비스 로그
- 요청 흐름 다이어그램

### Issue 후보

- `서비스별 Kubernetes Service와 DNS 이름을 확인한다`
- `Gateway/Ingress 경로 라우팅을 검증한다`
- `정상 사용자 흐름 테스트 데이터를 준비한다`
- `정상 사용자 흐름 E2E 절차를 작성한다`
- `정상 흐름 결과를 캡처하고 문서화한다`

## E4: LGTM 장애 관측성 확보

### 목적

장애 상황에서 어떤 서비스가 실패했고, 영향이 어디까지 퍼졌는지 메트릭, 로그, 가능하면 트레이스로 설명한다.

### 포함 범위

- 서비스 메트릭 노출 확인
- Prometheus scrape 또는 ServiceMonitor 연결
- Grafana 에러율/응답시간 패널
- Loki 로그 조회 경로 또는 대체 로그 중앙화 경로
- Tempo/Jaeger tracing 도입 여부 결정
- 장애 주입 절차
- 장애 전후 지표 변화 캡처

### 제외 범위

- 모든 서비스의 완전한 SLO 체계
- 장기 보관용 로그 운영 설계
- 고급 tracing 전체 도입

### 연결 Scenario

- S2 서비스 장애 관측
- S6 발표 리허설

### 완료 기준

- 장애 전후 에러율 또는 응답시간 변화가 확인된다.
- 장애 시점의 서비스 로그를 함께 확인할 수 있다.
- 실패한 서비스와 영향을 받은 흐름을 설명할 수 있다.
- 발표에서 사용할 캡처가 남아 있다.

### 증거

- Grafana 대시보드 캡처
- Prometheus query 결과
- Loki 로그 조회 결과 또는 대체 로그 조회 기록
- Tempo/Jaeger 도입 여부 결정 기록
- 장애 주입 실행 기록
- 서비스 로그

### Issue 후보

- `서비스별 메트릭 노출 상태를 확인한다`
- `Prometheus가 서비스 메트릭을 수집하도록 연결한다`
- `Grafana에 에러율과 응답시간 패널을 구성한다`
- `Loki 로그 조회 경로 또는 대체 로그 조회 경로를 확인한다`
- `Tempo/Jaeger tracing 도입 여부를 결정한다`
- `장애 주입 절차를 작성한다`
- `장애 전후 지표 변화를 캡처한다`

## E5: 통신 경계와 Mesh 정책

### 목적

서비스 간 통신을 모두 열어두는 대신, 허용된 호출만 가능하도록 정책으로 통제한다.

### 포함 범위

- 서비스 호출 관계 정리
- 허용/차단 매트릭스 작성
- NetworkPolicy 적용
- Service Mesh 도입 여부와 책임 경계 결정
- 허용 호출과 차단 호출 검증

### 제외 범위

- 모든 서비스에 완전한 mTLS 강제
- 복잡한 traffic splitting
- 장기 운영용 mesh 표준화

### 연결 Scenario

- S3 비인가 통신 차단

### 완료 기준

- 허용된 서비스 간 호출은 성공한다.
- 허용되지 않은 직접 호출은 실패한다.
- NetworkPolicy와 Service Mesh의 책임 경계가 설명되어 있다.

### 증거

- 호출 허용/차단 매트릭스
- NetworkPolicy 또는 Mesh policy manifest
- 허용/차단 테스트 결과
- 정책 적용 전후 비교

### Issue 후보

- `서비스 간 허용/차단 호출 매트릭스를 작성한다`
- `Gateway와 Service Mesh 책임 경계를 결정한다`
- `NetworkPolicy 기본 정책을 작성한다`
- `허용된 서비스 호출이 성공하는지 확인한다`
- `비인가 서비스 호출이 실패하는지 확인한다`

## E6: 보안 스캔과 품질 게이트

### 목적

위험한 manifest나 품질 기준 위반이 배포 전에 실패하도록 DevSecOps 검증 흐름을 만든다.

### 포함 범위

- Trivy config scan 실행
- 위험 manifest 샘플
- 정상/실패 케이스 확인
- CI 또는 로컬 검증 절차
- 필요 시 SonarQube 품질 게이트 검토

### 제외 범위

- 모든 보안 도구 동시 도입
- 운영 수준의 취약점 관리 프로세스
- GitHub Security 탭/SARIF 전체 연동

### 연결 Scenario

- S4 보안 스캔 차단

### 완료 기준

- 정상 manifest는 통과한다.
- 위험 manifest는 실패한다.
- 실패 원인과 수정 방향을 설명할 수 있다.

### 증거

- Trivy 실행 결과
- CI 실패 예시 또는 로컬 실행 로그
- 위험 manifest 샘플
- 수정 전후 비교

### Issue 후보

- `Trivy config scan 실행 방식을 정한다`
- `현재 Kubernetes manifest를 스캔한다`
- `privileged 컨테이너 실패 샘플을 만든다`
- `보안 스캔 실패 결과를 캡처한다`
- `스캔 결과를 발표 자료에 정리한다`

## E7: 독립 배포와 가용성

### 목적

한 서비스의 배포가 전체 흐름을 깨뜨리지 않고, 주요 서비스가 최소 가용성을 유지하는지 확인한다.

### 포함 범위

- 독립 배포 대상 서비스 선정
- 단일 서비스 재배포 절차
- 핵심 E2E 재검증
- readiness/liveness 확인
- PDB 최소 가용성 기준

### 제외 범위

- 완성형 Canary 배포
- 모든 서비스의 별도 배포 파이프라인 완성
- KEDA/HPA 전체 운영 자동화

### 연결 Scenario

- S5 독립 배포 검증

### 완료 기준

- 대상 서비스만 재배포된다.
- 재배포 후 정상 사용자 흐름이 다시 통과한다.
- 배포 중 문제가 생기면 영향 범위를 설명할 수 있다.

### 증거

- `kubectl rollout status` 결과
- 배포 로그
- E2E 재검증 결과
- Grafana 지표 변화

### Issue 후보

- `독립 배포 검증 대상 서비스를 선택한다`
- `대상 서비스의 readiness/liveness 설정을 확인한다`
- `PDB 적용 기준을 정리한다`
- `대상 서비스만 재배포하는 절차를 작성한다`
- `재배포 후 정상 사용자 흐름을 재검증한다`

## E8: 증거 정리와 발표

### 목적

프로젝트 기간 동안 만든 결과를 기능 목록이 아니라 운영 능력 검증 이야기로 정리한다.

### 포함 범위

- 발표 스토리라인
- 아키텍처 다이어그램
- 검증 결과 캡처 정리
- 회고와 후속 과제
- 발표 리허설

### 제외 범위

- 새 기능 추가
- 미검증 기능을 핵심 성과로 포장
- 발표 직전 대규모 구조 변경

### 연결 Scenario

- S6 발표 리허설
- S1, S2, S3, S4, S5의 최종 증거

### 완료 기준

- 문제, 목표, 설계, 검증, 결과, 회고가 한 흐름으로 연결된다.
- 핵심 주장이 로그, 지표, 캡처, 테스트 결과 중 하나로 뒷받침된다.
- 미완성 항목은 후속 과제로 분류되어 있다.

### 증거

- 발표 스토리라인
- 아키텍처 다이어그램
- 검증 결과 캡처 모음
- 회고 문서

### Issue 후보

- `발표 흐름을 문제-목표-검증-결과-회고 순서로 작성한다`
- `핵심 아키텍처 다이어그램을 작성한다`
- `검증 결과 캡처를 최신 상태로 정리한다`
- `10분 발표 리허설을 진행한다`
- `미완성 항목과 후속 과제를 정리한다`

## E9: GitOps/ArgoCD 배포 검증

### 목적

서비스 변경 사항이 GitOps 경로를 통해 AWS demo/QA 환경에 반영되고, ArgoCD sync 상태와 rollback 기준으로 운영 배포 흐름을 설명할 수 있게 한다.

### 포함 범위

- 현재 GitOps/ArgoCD 구성과 gap 확인
- AWS demo/QA 환경의 ArgoCD Application 범위 결정
- 이미지 tag와 manifest update 기준
- ArgoCD sync 또는 단기 대체 배포 smoke
- rollback runbook

### 제외 범위

- 완성형 progressive delivery 플랫폼 구축
- 모든 서비스의 개별 ApplicationSet 표준화
- ArgoCD Rollouts Canary 완성

### 연결 Scenario

- S7 GitOps/ArgoCD 배포 검증
- S5 독립 배포 검증

### 완료 기준

- GitOps/ArgoCD 배포 경로의 현재 상태와 gap이 정리되어 있다.
- AWS 환경에 변경 사항을 반영하는 절차가 검증되어 있다.
- rollback 또는 수동 복구 기준이 문서화되어 있다.

### 증거

- GitOps 경로 인벤토리
- ArgoCD Application/sync 상태 캡처 또는 대체 배포 로그
- manifest diff 또는 image tag 변경 기록
- 배포 smoke 결과
- rollback runbook

### Issue 후보

- `현재 GitOps/ArgoCD 구성과 gap을 확인한다`
- `AWS demo/QA 환경의 ArgoCD Application 범위를 결정한다`
- `이미지 tag와 manifest update 기준을 정리한다`
- `ArgoCD sync 또는 대체 배포 smoke를 실행한다`
- `rollback 기준과 운영 runbook을 작성한다`

## Scenario와 Epic 매핑

| Scenario | 관련 Epic | 설명 |
| --- | --- | --- |
| S0 AWS 시연 환경 준비 | E2 | Terraform 기반을 demo/QA 시연 환경으로 연결한다. |
| S1 정상 사용자 흐름 검증 | E1, E2, E3 | 범위 합의, AWS 시연 환경, 정상 흐름 기준선을 만든다. |
| S2 서비스 장애 관측 | E3, E4 | 정상 흐름 위에 장애 주입과 LGTM 관측성을 얹는다. |
| S3 비인가 통신 차단 | E3, E5 | 서비스 호출 관계를 기준으로 허용/차단 정책을 검증한다. |
| S4 보안 스캔 차단 | E6 | 배포 전 보안 위반 차단을 검증한다. |
| S5 독립 배포 검증 | E3, E7, E9 | 정상 흐름과 GitOps 경로를 기준으로 단일 서비스 재배포를 검증한다. |
| S6 발표 리허설 | E8 | 각 시나리오의 증거를 발표 흐름으로 연결한다. |
| S7 GitOps/ArgoCD 배포 검증 | E2, E9 | AWS 환경에 변경 사항을 반영하는 GitOps 운영 흐름을 검증한다. |

## 향후 workplan 폴더 분리안

기존 Linear workplan 스키마를 유지한다면, 다음 단계의 YAML은 폴더로 나누는 편이 좋다.

```text
workplans/
├── 00-foundation.yaml
├── 05-aws-demo-environment.yaml
├── 10-e2e-baseline.yaml
├── 15-gitops-argocd-release.yaml
├── 20-observability.yaml
├── 30-traffic-policy.yaml
├── 40-devsecops.yaml
├── 50-delivery-reliability.yaml
└── 60-evidence-presentation.yaml
```

각 YAML의 task에는 `scenario:s1`, `epic:foundation` 같은 라벨을 붙인다.

```yaml
labels:
  - scenario:s2
  - epic:lgtm-observability
  - observability
```

## 다음 단계

- `README.md`에서 백로그 작성 규칙을 정한다.
- `workplans/*.yaml`을 만들 때 이 Epic ID와 label을 그대로 사용한다.
- `depends_on`은 `local_id` 기준으로 연결하고, `blocks`는 필요할 때만 보조로 적는다.
