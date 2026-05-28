---
id: S2
title: 공연 티켓 예매 S2 티켓 오픈 피크
type: scenario
status: draft
priority: P0
phase: Sprint 2-Sprint 3
tags: [ticketing, scenario, load-test, k6, grafana]
created: 2026-05-27
updated: 2026-05-27
---

## S2: 티켓 오픈 피크

### 검증 질문

티켓 오픈 직후 5분 동안 예매 요청이 몰려도 5xx 에러율 1% 이하와 P99 1.5초 이하를 유지하는가?

### 목표

공연 티켓 예매 도메인의 대표적인 피크 상황을 수치로 검증한다. 좌석 조회와 예약 요청이 동시에 증가할 때 API latency, error rate, throughput, pod scale-out, Kafka lag가 어떻게 변하는지 기록한다.

### 범위

- k6 기반 200 VU 5분 부하
- 로그인, 공연 조회, 좌석 조회, 예약 요청의 혼합 시나리오
- Prometheus query와 Grafana dashboard 확인
- 피크 전후 latency, 5xx, RPS 비교
- HPA와 Kafka lag에 대한 보조 관측

### 성공 기준

- 피크 5분 구간의 5xx 에러율이 1% 이하이다.
- 핵심 예매 API의 P99 응답시간이 1.5초 이하이다.
- 좌석 동시성 기준을 깨는 중복 티켓이 발생하지 않는다.
- Grafana에서 Before/After 비교가 가능한 캡처가 남는다.

### 증거

- k6 summary와 HTML/JSON 리포트
- Grafana Before/After 캡처
- Prometheus latency/error query 결과
- Kubernetes pod, HPA, event 기록
- DB 최종 좌석과 예약 상태

### 의존성

```text
S0 정상 예매 E2E
↓
S1 좌석 동시성
↓
k6 피크 부하 스크립트 준비
↓
Prometheus metric 수집 확인
↓
Grafana dashboard 준비
↓
티켓 오픈 피크 실행
↓
에러율, P99, 정합성 증거 기록
```

### Epic 후보

- 티켓 오픈 부하 테스트
- 예매 API 성능 지표 수집
- 피크 전후 운영 지표 비교

### Issue 후보

- `티켓 오픈 피크 k6 시나리오를 작성한다`
- `예매 API P95/P99와 5xx rate Prometheus query를 정리한다`
- `Grafana에 티켓 오픈 피크 Before/After 패널을 구성한다`
- `부하 테스트 후 DB 좌석 정합성을 확인한다`
- `피크 테스트 결과를 발표용 캡처로 고정한다`

### 관련 문서

- [시나리오 인덱스](README.md)
- [PRD](../01-prd.md)
- [서비스 아키텍처](../02-service-architecture.md)
- [마일스톤](../03-MILESTONES.md)
- [관측성과 검증 계획](../../members/service/ticketing-final/04-observability-validation.md)
