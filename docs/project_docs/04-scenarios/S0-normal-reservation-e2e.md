---
id: S0
title: 공연 티켓 예매 S0 정상 예매 E2E
type: scenario
status: draft
priority: P0
phase: Phase 0-Sprint 1
tags: [ticketing, scenario, e2e, newman, reservation]
created: 2026-05-27
updated: 2026-05-27
---

## S0: 정상 예매 E2E

### 검증 질문

사용자가 로그인하고 공연을 조회한 뒤 좌석을 선택해 예약, 결제, 티켓 조회까지 완료할 수 있는가?

### 목표

Docker Compose로 서비스들을 함께 실행한 뒤, Newman으로 정상 예매 E2E 테스트를 자동화한다. S0이 통과해야 좌석 동시성, 티켓 오픈 피크, 결제 장애 같은 후속 시나리오를 의미 있게 검증할 수 있다.

### 범위

- 로그인과 JWT 발급
- 공연 목록, 공연 상세, 좌석 상태 조회
- 좌석 lock과 예약 생성
- mock 결제 승인
- 결제 승인 이벤트 이후 티켓 발행
- 내 티켓 조회
- Newman collection과 실행 환경
- 정상 예매 1건의 DB 최종 상태 확인

### 성공 기준

- Docker Compose로 S0에 필요한 서비스와 의존 런타임을 실행할 수 있다.
- Newman collection이 로그인, 공연 조회, 좌석 조회, 예약, 결제, 티켓 조회를 순서대로 실행한다.
- Newman 실행 결과가 통과한다.
- 정상 예매 1건의 reservation, payment, ticket 최종 상태가 DB에서 확인된다.
- 같은 절차를 다른 팀원이 local 환경에서 다시 실행할 수 있다.

### 증거

- Newman 실행 결과와 collection/export 파일
- 대표 요청/응답 예시
- 정상 예매 1건의 reservation, payment, ticket 최종 상태
- Docker Compose 실행 결과
- seed data 적용 방법

### 의존성

```text
S0 API 요청/응답 계약 확정
↓
테스트 사용자와 공연/좌석 seed data 정의
↓
auth-service 로그인/JWT API 구현
↓
concert-service 공연/좌석 조회 API 구현
↓
reservation-service 예약 생성 API 구현
↓
payment-service mock 결제 승인 API 구현
↓
ticket-service 티켓 발행/조회 API 구현
↓
Docker Compose 통합 실행
↓
Newman E2E 실행
↓
결과와 증거 기록
```

### Epic 후보

- 정상 예매 E2E 구현
- Docker Compose 통합 실행
- Newman 테스트 자동화

### Issue 후보

- `auth-service 로그인/JWT API`
- `concert-service 공연/좌석 조회 API`
- `reservation-service 예약 생성 API`
- `payment-service mock 결제 승인 API`
- `ticket-service 티켓 발행/조회 API`
- `S0 Docker Compose 실행 환경`
- `S0 seed data`
- `S0 Newman collection`
- `S0 DB 최종 상태 확인`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
