---
id: ticketing-final-scenarios
title: 공연 티켓 예매 검증 시나리오
type: scenario-index
status: draft
tags: [ticketing, scenarios, validation, observability, performance, reliability]
created: 2026-05-27
updated: 2026-05-27
---

# 공연 티켓 예매 검증 시나리오

## 문서 목적

이 폴더는 공연 티켓 예매 프로젝트에서 "무엇을 구현했는가"보다 "무엇을 검증할 것인가"를 먼저 정리하기 위한 시나리오 문서 모음이다.

- `../01-prd.md`: 프로젝트 정의, 핵심 검증 질문, 성공 지표
- `../02-service-architecture.md`: 서비스 구성, API, Kafka 이벤트, 좌석 중복 방지 전략
- `../03-MILESTONES.md`: Phase 0, Sprint 1, Sprint 2, Sprint 3, Final Prep 일정
- `../../members/service/ticketing-final/04-observability-validation.md`: KPI, SLO 후보, 검증 시나리오 표

시나리오는 기능 목록이 아니라 검증 질문이다. 하나의 시나리오는 여러 Epic과 Issue로 나뉠 수 있으며, 각 Issue는 발표에 남길 수 있는 증거를 가져야 한다.

## 작성 원칙

- 티켓 예매 도메인의 실제 운영 상황으로 쓴다.
- 정상 흐름, 동시성, 피크 부하, 비동기 처리, 장애 격리, 배포 안정성을 분리해 검증한다.
- 성공 기준은 수치, 상태, 로그, 캡처처럼 확인 가능한 형태로 둔다.
- 증거는 Newman 결과, k6 리포트, Grafana 캡처, Prometheus query, Loki/Tempo 조회, Kubernetes event/log, DB 최종 상태 중 하나 이상으로 남긴다.
- `../03-MILESTONES.md`의 일정에 맞춰 Phase 0에는 계획과 기준선, Sprint 1에는 핵심 예매 흐름, Sprint 2에는 플랫폼과 운영 검증, Sprint 3에는 최종 실험 결과 고정을 배치한다.

## 시나리오 요약

| ID | 시나리오 | 검증 질문 | 우선순위 | 목표 구간 | 대표 증거 |
| --- | --- | --- | --- | --- | --- |
| [S0](S0-normal-reservation-e2e.md) | 정상 예매 E2E | 로그인부터 티켓 조회까지 핵심 예매 흐름이 재현되는가? | P0 | Phase 0-Sprint 1 | Newman 결과, 대표 요청/응답 |
| [S1](S1-seat-concurrency.md) | 좌석 동시성 | 같은 좌석을 동시에 예약해도 중복 티켓이 생기지 않는가? | P0 | Sprint 1-Sprint 3 | k6 결과, DB 최종 상태 |
| [S2](S2-ticket-open-peak.md) | 티켓 오픈 피크 | 티켓 오픈 5분 피크에서 에러율과 P99가 기준 안에 드는가? | P0 | Sprint 2-Sprint 3 | k6 리포트, Grafana Before/After |
| [S3](S3-kafka-async-processing.md) | Kafka 후속 처리 분리 | 예약 API 응답과 티켓 발행 지연을 분리해 측정할 수 있는가? | P0 | Sprint 1-Sprint 3 | API latency, consumer lag |
| [S4](S4-payment-delay-failure.md) | 결제 지연/장애 | 결제 지연이나 실패 중에도 예약 상태가 유실되지 않는가? | P0 | Sprint 3 | 상태 분포, alert 기록 |
| [S5](S5-notification-failure-isolation.md) | 알림 장애 격리 | 알림 장애가 예매 핵심 흐름을 실패시키지 않는가? | P0 | Sprint 3 | Grafana 캡처, service log |
| [S6](S6-hpa-scale-out.md) | HPA scale-out | 부하 증가 후 60초 이내 desired replica가 증가하는가? | P0 | Sprint 2-Sprint 3 | HPA describe, replica graph |
| [S7](S7-observability-tracing.md) | 관측성 추적 | 실패/지연 요청 1건을 3분 이내 trace와 log로 설명할 수 있는가? | P0 | Sprint 2-Sprint 3 | Tempo trace, Loki query |
| [S8](S8-rate-limiting.md) | Rate Limiting | 과호출은 429로 제한하고 정상 사용자는 유지되는가? | P1 | Sprint 2-Sprint 3 | gateway log, metric |
| [S9](S9-canary-deployment.md) | Canary 배포 | 신규 버전의 에러율 증가가 허용 범위 안에 있는가? | P1 | Sprint 2-Sprint 3 | 버전별 지표, rollout 기록 |
| [S10](S10-canary-rollback.md) | Canary 롤백 | 이상 감지 후 3분 이내 이전 버전으로 복구되는가? | P1 | Sprint 3 | 알림 기록, rollback 이벤트 |
| [S11](S11-security-policy-gate.md) | 보안 스캔과 정책 | 취약점과 비정상 접근을 배포 또는 런타임에서 차단하는가? | P2 | Sprint 2-Sprint 3 | scan report, allow/deny log |

## 시나리오 간 의존성

```text
S0 정상 예매 E2E
├─ S1 좌석 동시성
├─ S3 Kafka 후속 처리 분리
├─ S7 관측성 추적
└─ S8 Rate Limiting

S1 좌석 동시성
└─ S2 티켓 오픈 피크

S3 Kafka 후속 처리 분리
├─ S4 결제 지연/장애
└─ S5 알림 장애 격리

S2 티켓 오픈 피크
├─ S6 HPA scale-out
├─ S9 Canary 배포
└─ S10 Canary 롤백

S7 관측성 추적
├─ S4 결제 지연/장애
├─ S5 알림 장애 격리
└─ S10 Canary 롤백

S11 보안 스캔과 정책
└─ Final Prep 발표 근거 정리
```

## 우선순위 기준

| 우선순위 | 기준 | 일정 배치 |
| --- | --- | --- |
| P0 | PRD의 핵심 검증 질문에 직접 연결되거나 다른 검증의 선행 조건이다. | Phase 0부터 Sprint 3까지 반드시 추적한다. |
| P1 | 운영 안정성을 강화하지만 P0 결과가 확보된 뒤 범위를 조절할 수 있다. | Sprint 2 이후 실제 배포 경로가 준비되면 진행한다. |
| P2 | 보안과 정책 품질을 높이는 확장 검증이다. | Sprint 2 이후 여력이 있을 때 실행하고 Final Prep에서 후속 과제로도 설명할 수 있다. |

## 이슈 분해 규칙

시나리오를 이슈로 나눌 때는 구현 작업과 검증 작업이 함께 보이도록 작성한다.

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

좋은 이슈는 "기능을 만든다"에서 끝나지 않고, 어떤 증거가 남아야 완료인지까지 포함한다.

```text
Scenario: S1 좌석 동시성
Epic: 예약 정합성 검증
Task: 동일 좌석에 대한 동시 예약 k6 시나리오를 작성한다
Depends on: reservation-service가 좌석 lock과 unique constraint를 적용한다
Evidence: 성공/409 충돌 수, DB의 최종 좌석과 티켓 상태
Definition of Done: 초기 좌석 수를 초과한 성공 예약과 중복 티켓이 0건임을 기록한다
```

## 다음 단계

- Phase 0에서 S0-S7의 검증 질문과 증거 형식을 팀 기준으로 확정한다.
- Sprint 1에서 S0, S1, S3의 local E2E와 데이터 기준을 먼저 만든다.
- Sprint 2에서 S2, S6, S7, S8, S9, S11을 배포 환경과 연결한다.
- Sprint 3에서 S2-S7과 S10의 최종 캡처, 로그, 리포트를 고정한다.
- Final Prep에서는 새 실험을 추가하지 않고 시나리오별 증거 색인과 발표 흐름을 정리한다.
