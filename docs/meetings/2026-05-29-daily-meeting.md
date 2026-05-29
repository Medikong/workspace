---
id: MEETING-2026-05-29-001
title: "데일리 미팅"
date: 2026-05-29
type: meeting
status: draft
areas:
  - planning
  - service
  - gitops
  - infra
repos:
  - workspace
  - service
  - gitops
  - infra
attendees: []
related:
  - docs/meetings/2026-05-28-morning-decisions.md
  - docs/project_docs/00-GOAL.md
  - docs/issues/2026-05-29-service-cicd-pipeline.yaml
  - docs/projects_plan/
links: []
---

# 2026-05-29 데일리 미팅

## 목적

전일 진행 상황, 오늘 집중할 작업, 막힘과 조율이 필요한 항목을 정리한다.

## 전일 진행

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
|  |  |  | todo |

## 오늘 할 일

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| `00-GOAL.md`의 서비스 도메인과 기술 항목 자료 조사 | 전체 | `docs/project_docs/00-GOAL.md`, `docs/members/` | todo |
| 작업 계획을 세우기 전에 조사 결과를 도메인별·기술별로 정리 | 전체 | `docs/members/`, `docs/projects_plan/` | todo |
| 새 서비스 도메인 기준으로 서비스 CI 대상과 파이프라인 수정 | 서비스 담당 | `docs/issues/2026-05-29-service-cicd-pipeline.yaml` | todo |

## 막힘 / 리스크

| 항목 | 영향 | 필요한 결정 또는 도움 |
| --- | --- | --- |
| 예전 의료 도메인 기준과 새 티켓팅 도메인 기준이 CI에 함께 남아 있음 | 테스트, 이미지 빌드, GitOps 배포 대상이 서로 어긋날 수 있음 | 새 서비스 목록과 dashboard 포함 여부 확정 |
| 자료 조사 없이 바로 작업 계획을 만들면 범위가 흐려짐 | 도메인 이해와 기술 선택 근거가 약해질 수 있음 | 조사 결과를 먼저 남긴 뒤 작업 단위로 분해 |

## 결정 사항

| 결정 | 이유 | 연결 문서 |
| --- | --- | --- |
|  |  |  |

## 후속 정리 필요

- 자료 조사 결과를 담당 영역 문서 또는 프로젝트 계획 문서에 반영한다.
- CI 수정 후 실행 결과와 남은 불일치를 작업계획에 연결한다.
