---
id: ADR-0006
title: "서비스 간 Circuit Breaker는 Istio outlierDetection으로 시작한다"
status: accepted
date: 2026-06-08
areas:
  - service-mesh
  - resilience
  - gitops
repos:
  - gitops
  - workspace
decision_drivers:
  - 애플리케이션 코드 변경 없이 서비스 간 장애 전파를 줄여야 한다.
  - Canary Routing과 같은 Istio DestinationRule 경로를 재사용해야 한다.
  - 장애 주입과 rollback 증거를 GitOps manifest로 남겨야 한다.
related:
  - ADR-0003
  - workspace/docs/personal/plans/istio-kong-implementation-plan.md
  - workspace/docs/personal/execution/phase-6-circuit-breaker-runbook.md
links:
  - https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/
  - https://istio.io/latest/docs/reference/config/networking/destination-rule/
  - https://istio.io/latest/docs/reference/config/networking/virtual-service/
supersedes: []
superseded_by: null
---

# ADR 0006: 서비스 간 Circuit Breaker는 Istio outlierDetection으로 시작한다

## 상태

Accepted

## 날짜

2026-06-08

## 배경

공연 티켓 서비스는 Kong을 외부 API Gateway로 두고, Istio를 내부 Service Mesh로 사용하는 구조를 채택했다. 예약, 결제, 티켓 발급, 알림 서비스는 서로 의존하므로 한 서비스의 5xx 오류나 지연이 다른 서비스로 전파될 수 있다.

요구사항에는 `DestinationRule`의 기본 Circuit Breaker와 outlier detection 기반 장애 endpoint ejection 검증이 포함되어 있다. 애플리케이션 코드에 Resilience4j 같은 라이브러리를 넣는 방식도 가능하지만, 현재 목표는 먼저 mesh 레벨에서 공통 정책을 적용하고 GitOps 증거를 확보하는 것이다.

## 결정

서비스 간 Circuit Breaker 1차 구현은 Istio `DestinationRule`의 `connectionPool`과 `outlierDetection`, `VirtualService`의 `timeout`과 `retries`로 구성한다.

첫 적용 대상은 `reservation-service`로 한다.

선택한 기본값:

```text
connectionPool.tcp.maxConnections = 100
connectionPool.http.http1MaxPendingRequests = 100
connectionPool.http.maxRequestsPerConnection = 50
outlierDetection.consecutive5xxErrors = 5
outlierDetection.interval = 10s
outlierDetection.baseEjectionTime = 30s
outlierDetection.maxEjectionPercent = 50
timeout = 2s
retries.attempts = 2
retries.perTryTimeout = 1s
retries.retryOn = 5xx,connect-failure,refused-stream
```

장애 주입은 기본 GitOps 상태에 포함하지 않고, `platform/istio/traffic/reservation/scenarios/` 아래 scenario manifest로 보관한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| 애플리케이션 코드에 Resilience4j 적용 | 서비스별 fallback과 도메인별 부분 응답을 세밀하게 구현할 수 있다. | 서비스 코드 변경과 테스트 범위가 커지고, 언어/프레임워크별 구현 차이가 생긴다. | 2차 보강 후보 |
| Kong upstream circuit breaker 중심 | 외부 진입 트래픽 보호에는 단순하다. | 내부 service-to-service 장애 전파를 직접 제어하기 어렵다. | 채택하지 않음 |
| Istio outlierDetection + retry/timeout | 코드 변경 없이 mesh 내부 공통 정책을 적용하고 Kiali/Prometheus 증거를 남기기 쉽다. | 도메인 fallback 응답 자체는 애플리케이션이 책임져야 한다. | 채택 |
| Circuit Breaker를 아직 구현하지 않음 | 단기 작업량이 줄어든다. | 요구사항 증거와 장애 대응 설계가 부족해진다. | 채택하지 않음 |

## 결과

좋아지는 점:

- 서비스 코드 변경 없이 예약 서비스 대상의 기본 Circuit Breaker 정책을 적용한다.
- Canary Routing에서 이미 사용하는 `DestinationRule` 경로를 재사용한다.
- 장애 주입 scenario와 rollback manifest를 GitOps에 남길 수 있다.
- Kiali/Prometheus에서 장애 시도와 endpoint 상태를 관찰할 수 있는 기반을 만든다.

비용:

- Kong이 mesh 밖에 있는 동안 외부 요청 기준 검증과 mesh 내부 client 기준 검증을 구분해야 한다.
- DB/Kafka 미기동으로 애플리케이션 Pod가 CrashLoopBackOff 상태이면 실제 ejection 검증은 지연된다.
- 도메인 fallback 응답은 mesh 정책만으로 만들 수 없으므로 필요 시 서비스 코드 보강이 필요하다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| done | reservation-service DestinationRule에 connectionPool/outlierDetection 추가 | service | `gitops/platform/istio/traffic/reservation/destination-rule.yaml` |
| done | reservation-service VirtualService에 timeout/retry 추가 | service | `gitops/platform/istio/traffic/reservation/virtual-service-stable.yaml` |
| done | fault-5xx, fault-delay scenario manifest 추가 | service | `gitops/platform/istio/traffic/reservation/scenarios/` |
| todo | DB/Kafka 복구 후 실제 5xx ejection 검증 | service | `workspace/docs/personal/execution/phase-6-circuit-breaker-runbook.md` |
| todo | 도메인 fallback이 필요한 API를 service 코드에서 별도 검토 | service | TBD |
