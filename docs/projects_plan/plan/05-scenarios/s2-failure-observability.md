---
id: kt-cloudnative-study-scenario-s2
title: KT CloudNative Study S2 서비스 장애 관측
type: scenario
status: draft
tags: [kt-cloudnative-study, scenario, s2, validation]
created: 2026-05-27
updated: 2026-05-27
---

## S2: 서비스 장애 관측

### 검증 질문

예약 또는 처방 서비스 장애가 발생했을 때 영향 범위를 Grafana, 로그, 메트릭으로 설명할 수 있는가?

### 목표

관측성 도입의 가치를 보여준다. 단순히 Prometheus와 Grafana를 설치하는 것이 아니라, LGTM 관점에서 장애 상황의 메트릭, 로그, 가능하면 트레이스를 연결해 어떤 서비스가 실패했고 영향이 어디까지 퍼졌는지 설명한다.

### 범위

- 서비스별 `/metrics` 또는 Prometheus scrape 대상 확인
- ServiceMonitor 또는 scrape config 연결
- Grafana 에러율/응답시간 패널 구성
- Loki 로그 조회 경로 또는 대체 로그 중앙화 경로 확인
- Tempo/Jaeger tracing 도입 여부 결정
- 장애 주입 절차 작성
- 장애 전후 지표 변화 캡처

### 성공 기준

- 장애 전후 에러율 또는 응답시간 변화가 확인된다.
- 장애 시점의 서비스 로그를 함께 조회할 수 있다.
- 실패한 서비스와 영향을 받은 호출 흐름을 설명할 수 있다.
- 발표에 사용할 Grafana 캡처 또는 로그가 남는다.

### 증거

- Grafana 대시보드 캡처
- Prometheus query 결과
- Loki 로그 조회 결과 또는 대체 로그 조회 기록
- Tempo/Jaeger 도입 여부 결정 기록
- 장애 주입 명령 또는 절차
- 서비스 로그

### 의존성

```text
S1 정상 사용자 흐름 검증
↓
서비스 메트릭 노출
↓
Prometheus 수집 연결
↓
Grafana 패널 구성
↓
로그 조회 경로 확인
↓
Tracing 도입 여부 결정
↓
장애 주입 절차 작성
↓
장애 실행
↓
지표/로그 변화 캡처
```

### Epic 후보

- 서비스 메트릭 수집
- Grafana 장애 관측 대시보드
- 로그 중앙화 또는 조회 경로
- Tracing 도입 판단
- 장애 주입 검증

### Issue 후보

- `서비스별 메트릭 노출 상태를 확인한다`
- `Prometheus가 서비스 메트릭을 수집하도록 연결한다`
- `Grafana에 에러율과 응답시간 패널을 구성한다`
- `Loki 로그 조회 경로 또는 대체 로그 조회 경로를 확인한다`
- `Tempo/Jaeger tracing 도입 여부를 결정한다`
- `예약 또는 처방 서비스 장애 주입 절차를 작성한다`
- `장애 전후 지표 변화를 캡처한다`
- `장애 영향 범위 설명 문서를 작성한다`

## 관련 문서

- [시나리오 인덱스](README.md)
- [프로젝트 실행 계획](../02-PROJECT_PLAN.md)
- [마일스톤](../03-MILESTONES.md)
- [Workplans](../../workplans/README.md)
