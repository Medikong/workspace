---
id: TROUBLE-003
title: "서비스 이미지 publish 빌드 시간 증가"
status: triaged
priority: p2
severity: medium
area: service
repos:
  - service
  - workspace
owner: unassigned
created: 2026-06-08
updated: 2026-06-08
resolved: null
tags:
  - ci
  - docker
  - buildx
  - arm64
  - image-publish
related: []
links:
  - https://github.com/Medikong/service/actions/runs/27116104899
  - https://github.com/Medikong/service/actions/runs/26939331791
---

# 서비스 이미지 publish 빌드 시간 증가

## Context

2026-06-08 기준 `service` repo의 `Image Publish` workflow에서 `concert-service`와 `reservation-service`는 2분 안팎으로 끝났지만, `auth-service`, `payment-service`, `ticket-service`, `notification-service`는 5분 이상 걸렸다.

AWS 실행 환경은 ARM64이므로 ARM64 이미지는 반드시 필요하다. 따라서 `linux/amd64`만 빌드하는 방식은 이 트러블의 해결책이 아니다.

## Symptoms

- 관찰된 현상: 일부 서비스 이미지 publish job이 5분에서 10분 가까이 소요된다.
- 재현 조건: GitHub hosted `ubuntu-latest` runner에서 `docker buildx build --platform linux/amd64,linux/arm64 --push`를 수행한다.
- 기대 동작: 모든 서비스 publish job이 기존처럼 1분 안팎 또는 2분 내외로 완료된다.
- 실제 동작: `concert-service`와 `reservation-service`만 2분 내외이고, pip 기반 서비스들이 5분 이상 걸린다.

최근 느린 실행:

| 항목 | 값 |
| --- | --- |
| GitHub Actions run | `27116104899` |
| URL | https://github.com/Medikong/service/actions/runs/27116104899 |
| commit | `25f4c4d3e2e7d6933353d412e26ba4b4ba89cce2` |
| 실행 시간 | 2026-06-08 13:30:03 KST -> 13:40:17 KST |

| Job | Duration |
| --- | ---: |
| `이미지 publish (concert-service)` | 115s |
| `이미지 publish (reservation-service)` | 111s |
| `이미지 publish (auth-service)` | 487s |
| `이미지 publish (payment-service)` | 494s |
| `이미지 publish (notification-service)` | 533s |
| `이미지 publish (ticket-service)` | 599s |

비교용 빠른 실행:

| 항목 | 값 |
| --- | --- |
| GitHub Actions run | `26939331791` |
| URL | https://github.com/Medikong/service/actions/runs/26939331791 |
| commit | `1547341d018bdc66bb4c9c6cf187f92f53956321` |
| 실행 시간 | 2026-06-04 17:08:40 KST -> 17:10:02 KST |

| Job | Duration |
| --- | ---: |
| `이미지 publish (concert-service)` | 44s |
| `이미지 publish (reservation-service)` | 45s |
| `이미지 publish (auth-service)` | 54s |
| `이미지 publish (payment-service)` | 54s |
| `이미지 publish (notification-service)` | 56s |
| `이미지 publish (ticket-service)` | 62s |

## Impact

- 영향 범위: `service` repo의 image publish workflow와 ARM64 이미지 빌드 경로.
- 우선 처리 이유: main publish 시간이 느려지고, 전체 배포 피드백 시간이 가장 느린 서비스 job에 묶인다.
- 우회 방법: ARM64 이미지는 필요하므로 amd64-only 전환은 우회 방법으로 보지 않는다. 수동으로 특정 이미지만 publish하는 것은 가능하지만 근본 해결은 아니다.

## Investigation

| 시간 | 확인 내용 | 결과 |
| --- | --- | --- |
| 2026-06-08 13:30 KST | 최신 `Image Publish` run의 job별 소요 시간 확인 | `ticket-service` 599s, `notification-service` 533s, `payment-service` 494s, `auth-service` 487s |
| 2026-06-08 13:30 KST | 같은 run의 step별 소요 시간 확인 | 병목은 `Docker 이미지 빌드` step이며 OIDC, ECR login, digest 수집, gitops 업데이트는 몇 초 수준 |
| 2026-06-04 17:08 KST | 과거 빠른 run과 비교 | 모든 서비스가 44s~62s에 완료됨 |
| 2026-06-08 | `ticket-service` build log 확인 | `linux/amd64` pip 설치는 약 24초, `linux/arm64` pip 설치는 약 497.7초 |
| 2026-06-08 | `reservation-service` build log 확인 | `linux/arm64` `uv sync --frozen --no-dev --no-install-project --no-editable` 단계는 약 67.1초 |
| 2026-06-08 | 서비스별 Dockerfile과 의존성 파일 확인 | 빠른 서비스는 `uv.lock` 기반, 느린 서비스는 `requirements.txt`와 여러 번의 `pip install` 기반 |

직접 원인은 AWS 인증이나 ECR push가 아니라 `docker buildx build --platform linux/amd64,linux/arm64 --push`에서 ARM64 이미지를 빌드하는 동안 발생한 pip 의존성 설치 지연이다.

GitHub hosted `ubuntu-latest` runner는 기본적으로 `linux/amd64` 환경이다. 이 runner에서 `linux/arm64` 이미지를 빌드하면 QEMU 에뮬레이션 경로를 타고, pip 기반 의존성 설치가 크게 느려졌다.

서비스별 차이는 다음과 같다.

- `concert-service`, `reservation-service`: `pyproject.toml`/`uv.lock` 기반, `uv sync --frozen --no-dev --no-install-project --no-editable` 사용.
- `auth-service`, `payment-service`, `ticket-service`, `notification-service`: `requirements.txt` 기반, 공통 패키지와 서비스 요구사항을 여러 번 `pip install`.

`ticket-service` 로그에서는 `packages/server`가 `sqlalchemy>=2.0.41,<3.0.0`을 가져온 뒤 서비스 `requirements.txt`의 `sqlalchemy==2.0.41`에 맞추며 재설치 흐름이 나타났다.

조사 당시 `service` repo 로컬 워킹트리에는 다음 개선 시도가 미커밋 상태로 있었다.

- `.github/workflows/image-build.yml`: `type=gha` build cache 전달
- `.github/workflows/image-publish.yml`: `type=gha` build cache 전달
- `Taskfile.yml`: `docker buildx build`에 cache 인자 조립
- `services/*/Dockerfile`: pip/uv cache mount 추가

느린 실행 `27116104899` 기준으로는 아직 `task app-image-build` 호출에 `DOCKER_BUILD_CACHE_FROM`/`DOCKER_BUILD_CACHE_TO`가 전달되지 않았다. 따라서 cache 개선은 GitHub Actions에서 검증된 상태가 아니다.

## Decision

ARM64 이미지는 유지한다. 우선순위는 다음과 같다.

1. pip 기반 서비스도 `concert-service`/`reservation-service`처럼 `uv.lock` 기반으로 전환한다.
2. workflow -> `Taskfile.yml` -> `docker buildx build`까지 `type=gha` build cache가 실제로 전달되게 한다.
3. 멀티 플랫폼 빌드를 유지하면 cache scope를 서비스와 아키텍처 단위로 분리하는 방식을 검토한다.
4. AWS 운영 환경이 ARM64인 점을 고려해 Graviton 기반 ARM64 native runner를 장기 개선안으로 검토한다.

## Actions

| 상태 | 작업 | 담당 | 링크 |
| --- | --- | --- | --- |
| todo | `service` repo의 cache 관련 미커밋 변경을 검토하고 workflow에서 buildx까지 전달되는지 확인 | unassigned |  |
| todo | `auth-service`, `payment-service`, `ticket-service`, `notification-service`의 `uv.lock` 전환 가능성 검토 | unassigned |  |
| todo | `ticket-service`의 SQLAlchemy 재설치/다운그레이드 흐름 정리 | unassigned |  |
| todo | GitHub Actions에서 같은 workflow를 최소 2회 실행해 cache 저장과 cache hit를 확인 | unassigned |  |
| todo | ARM64 native runner 또는 ARM64-only publish 운영 가능 여부 확인 | unassigned |  |

## Resolution

미해결. 현재 원인은 triaged 상태다.

완료 시 다음을 확인해야 한다.

- ARM64 이미지 publish가 유지된다.
- `concert-service`와 `reservation-service`의 빌드 시간이 악화되지 않는다.
- pip 기반 서비스의 ARM64 의존성 설치 시간이 줄어든다.
- digest 수집과 deploy plan 생성이 계속 성공한다.
