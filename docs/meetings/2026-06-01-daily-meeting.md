---
id: MEETING-2026-06-01-001
title: "데일리 미팅"
date: 2026-06-01
type: meeting
status: draft
areas:
  - planning
  - observability
  - gitops
repos:
  - workspace
  - gitops
attendees: []
related:
  - docs/project_docs/00-GOAL.md
  - docs/architecture/observability/
  - docs/architecture/observability/metrics/README.md
links:
  - https://github.com/Medikong/workspace/issues/8
  - https://github.com/Medikong/workspace/issues/13
  - https://github.com/Medikong/gitops/issues/4
---

# 2026-06-01 데일리 미팅

## 목적

오늘 진행할 관측성 조사 범위와 `workspace#8` 작업의 연결 지점을 정리한다.

## 전일 진행

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| 관측성 기술 조사 진행 | 관측성 담당 | `docs/architecture/observability/` | 진행 중 |

## 오늘 할 일

| 항목 | 담당 | 기록 위치 | 상태 |
| --- | --- | --- | --- |
| `workspace#8` 지표 정의와 수집 작업 이어서 진행 | 관측성 담당 | https://github.com/Medikong/workspace/issues/8 | 진행 중 |
| 어떤 서비스에서 어떤 지표를 볼지 조사 | 관측성 담당 | `docs/architecture/observability/metrics/README.md` | 진행 중 |
| Prometheus, ServiceMonitor, `/metrics` 수집 경로 정리 | 관측성 담당 | `docs/architecture/observability/`, https://github.com/Medikong/gitops/issues/4 | todo |

## 결정 사항

| 결정 | 이유 | 연결 문서 |
| --- | --- | --- |
| 관측성 작업은 `workspace#8` 지표 정의와 수집 범위부터 정리한다. | 무엇을 수집할지 먼저 잡아야 Prometheus와 ServiceMonitor 설정도 같은 기준으로 맞출 수 있다. | https://github.com/Medikong/workspace/issues/8 |

## 후속 정리 필요

- 조사한 내용을 관측성 담당 문서와 지표 기준 문서에 옮긴다.
- 지표 기준을 정리한 뒤 `gitops#4` 구현 범위와 연결한다.
