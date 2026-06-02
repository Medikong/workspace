---
id: MEETING-2026-06-02-002
title: "데일리 미팅"
date: 2026-06-02
type: meeting
status: draft
areas:
  - infra
  - aws
  - deployment
  - ci-cd
repos:
  - workspace
  - infra
  - gitops
  - service
attendees: []
related:
  - docs/meetings/2026-06-02-aws-infra-questions.md
  - docs/projects_plan/reference/AWS_INFRA_SIZING.md
  - docs/project_docs/04-scenarios/S9-canary-deployment.md
  - docs/project_docs/04-scenarios/S10-canary-rollback.md
links: []
---

# 2026-06-02 데일리 미팅

## 목적

개발 환경용 AWS 인프라를 하나 구성하고, CI/CD로 지속 배포해서 팀이 함께 쓰는 검증 환경으로 운영하는 방향을 정리한다.

## 전일 진행

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| AWS 인프라 구성과 배포 전략 관련 질문 정리 | 인프라 담당 | `docs/meetings/2026-06-02-aws-infra-questions.md` | 완료 |

## 오늘 할 일

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| 개발 환경용 AWS 인프라 범위 정리 | 인프라 담당 | `docs/projects_plan/reference/AWS_INFRA_SIZING.md` | todo |
| CI/CD를 통한 지속 배포 흐름 정리 | 배포 담당 | `docs/project_docs/04-scenarios/S9-canary-deployment.md` | todo |
| 공용 검증 환경으로 사용할 때 필요한 운영 기준 정리 | 인프라 담당 | `docs/project_docs/04-scenarios/S10-canary-rollback.md` | todo |

## 결정 사항

| 결정 | 이유 | 연결 문서 |
| --- | --- | --- |
| AWS 인프라는 개발 환경용으로 하나 구성한다. | 각자 따로 환경을 만들기보다 한 환경을 공용으로 쓰는 편이 비용과 관리 부담을 줄일 수 있다. | `docs/projects_plan/reference/AWS_INFRA_SIZING.md` |
| 공용 AWS 환경은 CI/CD로 지속 배포한다. | 배포 결과를 같은 환경에서 확인해야 서비스 간 연동과 운영 기준을 꾸준히 맞출 수 있다. | `docs/project_docs/04-scenarios/S9-canary-deployment.md` |

## 후속 정리 필요

- AWS 개발 환경의 최소 구성과 예상 비용을 정리한다.
- CI/CD가 어느 repo 변경을 기준으로 공용 환경에 배포할지 정리한다.
- 공용 환경에서 배포 성공과 롤백 필요 여부를 판단할 기준을 정리한다.
