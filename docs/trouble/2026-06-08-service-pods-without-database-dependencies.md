---
id: TROUBLE-002
title: "서비스 Pod 배포 후 DB 의존성 Pod 부재"
status: open
priority: p1
severity: high
area: deployment
repos:
  - workspace
  - service
  - gitops
  - infra
owner: unassigned
created: 2026-06-08
updated: 2026-06-08
resolved: null
tags:
  - aws-dev
  - deployment
  - database
  - postgresql
  - mongodb
  - redis
  - dependency
related:
  - docs/meetings/2026-06-08-daily-meeting.md
  - docs/project_docs/02-service-architecture.md
  - docs/project_docs/00-GOAL.md
links: []
---

# 서비스 Pod 배포 후 DB 의존성 Pod 부재

## Context

2026-06-08 데일리 미팅에서 aws-dev 배포 상태를 확인하던 중, 서비스 Pod는 배포됐지만 데이터베이스 의존성 Pod가 없는 상태가 확인됐다.

현재 서비스 아키텍처 문서 기준으로 `auth-service`, `concert-service`, `reservation-service`, `payment-service`, `ticket-service`는 PostgreSQL에 의존하고, `notification-service`는 MongoDB에 의존한다. `reservation-service`는 Redis를 후보 의존성으로도 둔다.

## Symptoms

- 관찰된 현상:
  - 서비스 Pod는 배포된 상태다.
  - 데이터베이스 의존성 Pod가 보이지 않는다.
- 재현 조건:
  - aws-dev 환경에서 서비스 배포 상태와 의존성 Pod 상태를 함께 확인한다.
- 기대 동작:
  - 서비스 Pod와 함께 필요한 PostgreSQL, MongoDB, Redis 후보 등 런타임 의존성이 준비돼 있다.
  - 서비스 readiness와 주요 API 검증이 DB 연결 상태를 기준으로 통과한다.
- 실제 동작:
  - 서비스 Pod는 있으나 DB 의존성 Pod가 없는 상태라 실제 기능 검증을 완료로 볼 수 없다.

## Impact

- 영향 범위:
  - DB 연결이 필요한 API와 readiness 검증이 실패할 수 있다.
  - 정상 예매 E2E, 좌석 동시성, 결제, 티켓 발급, 알림 저장 검증이 막힐 수 있다.
  - Canary 배포나 관측성 검증에서도 애플리케이션 오류가 배포 문제인지 의존성 부재인지 구분하기 어려워진다.
- 우선 처리 이유:
  - 서비스 Pod Running 상태만으로 통합 검증 준비를 판단하면 실제 사용자 흐름 검증에서 바로 막힌다.
  - DB 의존성이 GitOps 관리 대상인지, 외부 관리형 서비스인지, 아직 미구성인지 먼저 확정해야 한다.
- 우회 방법:
  - 임시 DB Pod를 수동으로 올릴 수는 있지만, aws-dev 배포 기준으로는 GitOps 또는 infra 책임 경계가 먼저 정해져야 한다.

## Investigation

| 시간 | 확인 내용 | 결과 |
| --- | --- | --- |
| 2026-06-08 | 데일리 미팅에서 서비스 Pod 배포 상태 확인 | 서비스 Pod는 배포됨 |
| 2026-06-08 | 데일리 미팅에서 데이터베이스 의존성 Pod 존재 여부 확인 | DB 의존성 Pod 없음 |
| 2026-06-08 | 서비스 아키텍처 문서의 의존성 확인 | PostgreSQL, MongoDB, Redis 후보가 서비스별 의존성으로 정의됨 |

## Decision

서비스 Pod 배포만으로 배포 완료를 판단하지 않는다. aws-dev의 서비스 검증 준비 상태는 서비스 Pod와 데이터베이스 의존성 준비 상태를 함께 확인한다.

먼저 다음 세 가지 중 현재 구조가 무엇인지 확정한다.

- DB를 클러스터 내부 Pod로 GitOps에서 배포한다.
- DB를 AWS 관리형 서비스로 두고 secret, endpoint, network만 GitOps/infra에서 연결한다.
- 현재는 DB 배포 범위가 빠져 있어 별도 작업으로 추가해야 한다.

## Actions

| 상태 | 작업 | 담당 | 링크 |
| --- | --- | --- | --- |
| todo | aws-dev에서 DB 의존성 Pod 또는 외부 DB endpoint가 실제로 준비돼 있는지 확인 | gitops/infra |  |
| todo | 서비스별 DB 의존성 목록과 namespace, secret, endpoint 기준을 정리 | workspace | `docs/project_docs/02-service-architecture.md` |
| todo | DB를 클러스터 내부에 둘지 AWS 관리형 서비스로 둘지 결정 | infra |  |
| todo | 결정된 방식에 맞춰 GitOps 배포 대상 또는 infra 리소스를 추가 | gitops/infra |  |
| todo | 서비스 readiness와 주요 API가 DB 연결까지 포함해 통과하는지 확인 | service |  |

## Resolution

미해결. 서비스 Pod와 데이터베이스 의존성이 함께 준비된 뒤 정상 예매 E2E 또는 최소 readiness 검증으로 닫는다.
