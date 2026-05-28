---
id: S8
title: 공연 티켓 예매 S8 Rate Limiting
type: scenario
status: draft
priority: P1
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, rate-limiting, kong, gateway]
created: 2026-05-27
updated: 2026-05-27
---

## S8: Rate Limiting

### 검증 질문

과호출 사용자는 429로 제한하고 정상 사용자의 예매 성공률은 99% 이상으로 유지되는가?

### 목표

티켓 오픈 상황에서 일부 사용자의 과도한 요청이 전체 예매 흐름을 망가뜨리지 않도록 gateway 단계의 backpressure를 검증한다. Rate Limiting은 단순 차단이 아니라 정상 사용자 보호를 증명해야 한다.

### 범위

- Kong 또는 gateway rate limit 정책
- 정상 사용자와 과호출 사용자를 분리한 k6 시나리오
- 429 응답 비율과 정상 사용자 성공률 측정
- gateway log와 metric 확인
- 예매 API 5xx와 latency 영향 확인

### 성공 기준

- 과호출 요청은 429로 제한된다.
- 정상 사용자 예매 성공률은 99% 이상이다.
- rate limit 적용 중 핵심 예매 API 5xx가 증가하지 않는다.
- gateway log와 metric으로 제한 근거가 확인된다.

### 증거

- k6 사용자 그룹별 결과
- gateway access log
- Kong 또는 gateway metric
- Grafana rate limit 패널 캡처
- 정상 사용자 대표 요청/응답

### 의존성

```text
S0 정상 예매 E2E
↓
Gateway/Kong route 준비
↓
rate limit 정책 기준 확정
↓
정상 사용자와 과호출 사용자 k6 시나리오 작성
↓
rate limit 적용 전후 비교
↓
429와 정상 성공률 증거 기록
```

### Epic 후보

- Gateway 과호출 제어 검증
- 정상 사용자 보호 지표 수집
- Rate Limiting 발표 증거 정리

### Issue 후보

- `예매 API route에 rate limit 정책을 적용한다`
- `정상 사용자와 과호출 사용자를 분리한 k6 시나리오를 작성한다`
- `429 비율과 정상 사용자 성공률을 집계한다`
- `gateway log와 metric을 Grafana에 연결한다`
- `rate limit 적용 전후 결과를 비교한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
