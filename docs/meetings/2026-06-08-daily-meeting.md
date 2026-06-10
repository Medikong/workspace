---
id: MEETING-2026-06-08-001
title: "데일리 미팅"
date: 2026-06-08
type: meeting
status: draft
areas:
  - planning
  - ci-cd
  - gitops
  - observability
  - service-mesh
  - istio
  - database
repos:
  - workspace
  - service
  - gitops
  - infra
attendees: []
related:
  - docs/meetings/2026-06-04-daily-meeting.md
  - docs/architecture/observability/implementation/README.md
  - docs/adr/0003-separate-kong-edge-gateway-and-istio-service-mesh.md
  - docs/project_docs/04-scenarios/S9-canary-deployment.md
  - docs/trouble/2026-06-08-service-pods-without-database-dependencies.md
links: []
---

# 2026-06-08 데일리 미팅

## 목적

지난주에 각자 진행한 작업을 기준으로 현재 연결 상태를 확인하고, 이번 주에 통합 검증으로 넘길 범위를 정한다.

## 오늘 확인할 일

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| CI 기반에서 Argo CD 목표 구성이 어디까지 준비됐는지 확인 | 석진님 | `docs/project_docs/04-scenarios/S9-canary-deployment.md` | 확인 필요 |
| Grafana, Loki 연동 상태와 남은 설정 확인 | 범휘님 | `docs/architecture/observability/implementation/README.md` | 확인 필요 |
| Istio 서비스 config 적용과 테스트 결과 확인 | 명순님 | `docs/adr/0003-separate-kong-edge-gateway-and-istio-service-mesh.md` | 확인 필요 |
| 서비스 Pod와 DB 의존성 Pod 배포 상태 확인 | 배포 담당 | `docs/trouble/2026-06-08-service-pods-without-database-dependencies.md` | 문제 확인 |

## 현재 문제점

| 문제 | 영향 | 확인 위치 | 상태 |
| --- | --- | --- | --- |
| 서비스 Pod는 배포됐지만 데이터베이스 의존성 Pod가 없는 상태다. | 서비스가 떠 있어도 DB 연결이 필요한 API, readiness, 통합 시나리오 검증이 실패할 수 있다. | `docs/trouble/2026-06-08-service-pods-without-database-dependencies.md` | 확인 필요 |

## 결정 사항

| 결정 | 이유 | 연결 문서 |
| --- | --- | --- |
| 오늘 데일리 미팅은 2026-06-04에 잡은 중간점검으로 진행한다. | 작업별 진행 여부보다 실제로 서로 연결할 수 있는 상태인지 확인해야 한다. | `docs/meetings/2026-06-04-daily-meeting.md` |
| 완료된 작업은 증거 위치를 남기고, 미완료 작업은 다음 확인 지점을 문서에 연결한다. | 통합 검증 때 같은 내용을 다시 구두로 확인하지 않기 위해서다. |  |
| Canary 검증은 Argo CD 배포 상태, Istio traffic split, Grafana 지표를 함께 확인하는 방향으로 잡는다. | 배포와 트래픽 전환, 관측 결과가 같이 남아야 시나리오 증거로 쓸 수 있다. | `docs/project_docs/04-scenarios/S9-canary-deployment.md` |
| 서비스 배포 완료 여부는 DB 의존성까지 함께 본다. | 서비스 Pod만 Running이어도 런타임 의존성이 없으면 실제 기능 검증은 완료로 볼 수 없다. | `docs/trouble/2026-06-08-service-pods-without-database-dependencies.md` |

## 후속 정리 필요

- 석진님 작업은 Argo CD 목표 구성 결과와 남은 배포 연결 지점을 정리한다.
- 범휘님 작업은 Grafana, Loki 연동 결과와 확인 가능한 화면 또는 설정 위치를 정리한다.
- 명순님 작업은 Istio service config 테스트 결과와 다음 적용 대상을 정리한다.
- 서비스별 DB 의존성(PostgreSQL, MongoDB, Redis 후보)이 GitOps 배포 대상에 포함돼 있는지 확인한다.
- DB 의존성 Pod가 별도 namespace나 외부 관리형 서비스로 빠져 있는지 확인하고, 현재 aws-dev 기준의 정답을 정한다.
- 오늘 확인한 내용으로 이번 주 통합 검증 순서를 다시 맞춘다.
