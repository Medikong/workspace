---
id: S1
title: 공연 티켓 예매 S1 좌석 동시성
type: scenario
status: draft
priority: P0
phase: Sprint 1-Sprint 3
tags: [ticketing, scenario, concurrency, seat, k6]
created: 2026-05-27
updated: 2026-05-27
---

## S1: 좌석 동시성

### 검증 질문

같은 공연 회차의 같은 좌석을 여러 사용자가 동시에 예약해도 초기 좌석 수를 초과한 성공 예약과 중복 티켓이 0건인가?

### 목표

티켓 예매 프로젝트의 핵심 정합성을 검증한다. 좌석 경쟁 상황에서 `reservation-service`의 transaction, unique constraint, conflict 응답이 실제로 좌석 중복을 막는지 확인한다.

### 범위

- 동일 좌석 또는 제한된 좌석 풀에 대한 동시 예약 요청
- 성공 예약 수와 `409 Conflict` 수 집계
- 예약, 좌석 lock, 티켓 테이블 최종 상태 조회
- idempotency 정책 적용 여부 확인
- 동시성 테스트 중 5xx 발생 여부 확인

### 성공 기준

- 초기 좌석 수를 초과한 성공 예약이 0건이다.
- 동일 좌석에 대해 유효 티켓이 2건 이상 발행된 사례가 0건이다.
- 충돌 요청은 5xx가 아니라 의도된 conflict 응답으로 처리된다.
- DB 최종 상태와 k6 결과가 같은 결론을 보여준다.

### 증거

- k6 실행 결과와 성공/충돌 수
- DB 최종 상태 조회 결과
- duplicate ticket count query 결과
- reservation-service log
- Grafana의 conflict count와 5xx rate 캡처

### 의존성

```text
S0 정상 예매 E2E
↓
좌석 seed와 회차 기준 확정
↓
reservation-service 좌석 lock/transaction 적용
↓
동시 예약 k6 시나리오 작성
↓
DB 최종 상태 조회 query 준비
↓
동시성 테스트 실행
↓
중복 예약과 중복 티켓 0건 증거 기록
```

### Epic 후보

- 예약 정합성 검증
- 좌석 중복 방지 테스트
- DB 최종 상태 증거 관리

### Issue 후보

- `동시 예약 테스트용 공연 회차와 좌석 seed를 준비한다`
- `동일 좌석 동시 예약 k6 스크립트를 작성한다`
- `성공 예약 수와 conflict 수를 집계한다`
- `중복 티켓 여부를 확인하는 DB query를 작성한다`
- `좌석 동시성 결과를 Grafana와 DB 상태로 캡처한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
