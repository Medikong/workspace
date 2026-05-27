---
id: ADR-0002
title: "로컬 Kubernetes 검증은 Docker Desktop을 기준으로 한다"
status: accepted
date: 2026-05-27
areas:
  - local-dev
  - kubernetes
  - validation
  - aws
repos:
  - workspace
  - gitops
  - infra
  - service
decision_drivers:
  - 반복 테스트 속도
  - 크로스 플랫폼 구성 단순화
  - 최종 인프라 연동 검증 분리
related:
  - docs/meetings/2026-05-27-morning-decisions.md
  - docs/projects_plan/plan/02-PROJECT_PLAN.md
  - docs/projects_plan/plan/05-scenarios/s0-aws-demo-environment.md
  - ../gitops/values/env/local-docker-desktop-kubeadm.yaml
  - ../gitops/values/env/local-docker-desktop-kind.yaml
  - ../gitops/docs/architecture/service-release-model.md
links: []
supersedes: []
superseded_by: null
---

# ADR 0002: 로컬 Kubernetes 검증은 Docker Desktop을 기준으로 한다

## 상태

Accepted

## 날짜

2026-05-27

## 배경

기존 로컬 Kubernetes 검증 흐름은 Vagrant + VMware 기반 VM 클러스터를 중심으로 설계되어 있었다. 이 방식은 실제 Kubernetes 노드를 직접 구성해 볼 수 있다는 장점이 있지만, 프로젝트의 반복 검증 루프로 사용하기에는 부담이 크다.

특히 다음 문제가 있다.

- VMware 기반 VM은 로컬 자원 사용량이 크고 실행과 재구성이 느리다.
- 팀원별 OS와 장비 차이에 따라 Vagrant, VMware, provider plugin, SSH key, network 설정을 맞추기 어렵다.
- 서비스, GitOps, 관측성 검증을 자주 반복해야 하는 단계에서는 클러스터 구성 자체보다 빠른 배포와 테스트 피드백이 더 중요하다.
- 최종 발표에서 확인해야 할 인프라 연동은 로컬 VM보다 AWS 배포 환경에서 검증하는 편이 더 직접적이다.

## 결정

로컬 Kubernetes 반복 검증의 기본 경로는 Vagrant + VMware가 아니라 Docker Desktop + Kubernetes로 둔다.

로컬에서는 Docker Desktop에 내장된 Kubernetes 또는 Docker Desktop runtime 기반 구성을 사용해 서비스 배포, GitOps values, Kong route, 관측성, 기본 시나리오를 빠르게 반복 검증한다.

최종 인프라 연동 검증은 AWS 배포 환경에서 수행한다. AWS 환경에서는 Terraform, Ansible, GitOps, ECR, 네트워크, IAM처럼 실제 클라우드 인프라와 연결되는 부분을 확인한다.

따라서 검증 경계는 다음처럼 나눈다.

| 검증 단계 | 기준 환경 | 목적 |
| --- | --- | --- |
| 빠른 로컬 반복 검증 | Docker Desktop + Kubernetes | 서비스 배포, manifest, route, 기본 E2E 흐름을 빠르게 확인 |
| 발표 백업 검증 | Docker Desktop + Kubernetes | AWS 준비가 늦어질 때 시연 가능한 최소 Kubernetes 증거 확보 |
| 최종 인프라 연동 검증 | AWS 배포 환경 | 클라우드 네트워크, IAM, ECR, GitOps, 운영 시나리오 확인 |

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| Vagrant + VMware를 로컬 기본 경로로 유지 | 실제 VM 기반 Kubernetes 구성을 학습하기 좋다. | 무겁고 느리며, 크로스 플랫폼 구성이 복잡하다. 반복 테스트 기본값으로는 부적합하다. | 채택하지 않음 |
| Docker Desktop + Kubernetes를 로컬 기본 경로로 사용 | 설치와 반복 검증이 비교적 단순하고 빠르다. 팀원 간 환경 차이를 줄일 수 있다. | 실제 AWS 인프라와는 차이가 있으므로 최종 인프라 검증을 대체할 수 없다. | 채택 |
| 로컬 Kubernetes 검증을 생략하고 AWS만 사용 | 실제 배포 환경과 가장 가깝다. | 매 반복마다 비용, 시간, 권한, 실패 복구 부담이 커진다. 빠른 개발 루프로 부적합하다. | 채택하지 않음 |

## 결과

좋아지는 점:

- 서비스와 GitOps 변경을 더 빠르게 반복 검증할 수 있다.
- 팀원별 로컬 환경 준비 부담이 줄어든다.
- 로컬 검증과 AWS 검증의 목적이 분리되어 발표 증거를 더 명확하게 구성할 수 있다.
- Vagrant + VMware 설정 문제에 쓰는 시간을 줄이고, 서비스/배포/관측성 시나리오 검증에 집중할 수 있다.

비용:

- Docker Desktop Kubernetes 결과는 AWS 인프라 연동 검증을 대체하지 않는다.
- 기존 Vagrant + VMware 문서와 명령은 퇴출 또는 archive 기준을 정리해야 한다.
- 로컬 Docker Desktop 경로와 AWS 경로의 차이를 발표와 문서에서 명시해야 한다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| todo | Docker Desktop + Kubernetes 기준 로컬 검증 runbook 정리 | 석진 | `../gitops/README.md` |
| todo | Vagrant + VMware 문서와 명령의 유지, archive, 제거 범위 정리 | 석진 | `../infra/infra/cluster/README.md` |
| todo | AWS 인프라 연동 검증 시나리오와 로컬 백업 시나리오 연결 | 범휘 | `docs/projects_plan/plan/05-scenarios/s0-aws-demo-environment.md` |
| todo | 서비스 주제 선정 후 로컬 E2E 검증 기준 연결 | 명수 | `docs/members/service/` |
