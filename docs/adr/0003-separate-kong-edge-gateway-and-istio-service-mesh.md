---
id: ADR-0003
title: "Kong은 Edge API Gateway로, Istio는 내부 Service Mesh로 분리한다"
status: accepted
date: 2026-05-29
areas:
  - api-gateway
  - service-mesh
  - traffic-management
  - security
  - observability
repos:
  - workspace
  - service
  - gitops
  - infra
decision_drivers:
  - Kong과 Istio의 역할 중복 최소화
  - 외부 API 정책과 내부 메시 정책의 책임 분리
  - 프로젝트 요구사항 증거 산출 용이성
  - mTLS와 JWT 인증의 경계 명확화
related:
  - docs/personal/recommended-role-and-visual-flow.md
  - docs/personal/current-state-and-legacy-inventory.md
  - docs/personal/requirements-goal-audit-and-task-breakdown.md
links:
  - https://developer.konghq.com/kubernetes-ingress-controller/
  - https://developer.konghq.com/kubernetes-ingress-controller/ingress/
  - https://developer.konghq.com/plugins/jwt/
  - https://developer.konghq.com/plugins/rate-limiting/
  - https://developer.konghq.com/plugins/prometheus/
  - https://istio.io/latest/docs/concepts/traffic-management/
  - https://istio.io/latest/docs/concepts/security/
  - https://istio.io/latest/docs/ops/deployment/architecture/
  - https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/
  - https://tanmaybatham.medium.com/using-istio-and-kong-in-kubernetes-cluster-366096dad529
supersedes: []
superseded_by: null
---

# ADR 0003: Kong은 Edge API Gateway로, Istio는 내부 Service Mesh로 분리한다

## 상태

Accepted

## 날짜

2026-05-29

## 배경

공연 티켓 서비스는 기존 의료 정보 플랫폼 기반 MSA 프로젝트에서 확장 중이며, 현재 요구사항에는 API Gateway, JWT 인증, 로드 밸런싱, HPA, 서비스 메시, Canary 라우팅, Circuit Breaker, mTLS, 관측성, 장애 대응이 함께 포함되어 있다.

Kong Gateway와 Istio는 모두 L7 트래픽 라우팅, 정책 적용, 관측성 일부를 처리할 수 있으므로 역할을 명확히 나누지 않으면 같은 기능을 두 계층에 중복 구현하게 된다. 특히 JWT 인증, Rate Limiting, Canary 라우팅, 요청 로그, Prometheus 메트릭은 Kong과 Istio 양쪽에서 모두 다룰 수 있다.

공식 문서 기준으로 Kong Ingress Controller는 Kubernetes의 `Ingress`, `HTTPRoute` 같은 리소스를 Kong Gateway 설정으로 변환해 클러스터 인바운드 요청을 처리한다. Kong의 JWT 플러그인은 HS256/RS256 JWT를 검증하고, Rate Limiting 플러그인은 Route, Service, Consumer 단위로 요청 수를 제한할 수 있다.

Istio 공식 문서는 Istio가 Envoy sidecar를 통해 mesh 내부 서비스 간 트래픽을 제어하며, `VirtualService`, `DestinationRule`로 Canary, A/B 테스트, timeout, retry, circuit breaker 등을 동적으로 구성할 수 있다고 설명한다. Istio 보안 문서는 workload identity, 인증, 인가, mTLS 기반 암호화를 서비스 간 통신 보호의 핵심으로 다룬다.

따라서 둘 중 하나만 선택하는 문제가 아니라, 어느 경계에서 어떤 책임을 맡길지 결정해야 한다.

## 결정

Kong은 외부 클라이언트가 시스템으로 들어오는 Edge API Gateway로 사용하고, Istio는 클러스터 내부 서비스 간 통신을 제어하는 Service Mesh로 사용한다.

채택 구조는 다음과 같다.

```text
Client
  -> Cloud Load Balancer / Kong Proxy Service
  -> Kong Gateway
     - 외부 API route
     - JWT 검증
     - Consumer/Route 기반 Rate Limit
     - Request ID / correlation header
     - Gateway access log와 Prometheus metric
  -> Kubernetes ClusterIP Service
  -> Istio sidecar가 주입된 application Pod
     - service-to-service mTLS
     - VirtualService 기반 내부 라우팅
     - DestinationRule 기반 subset, load balancing, circuit breaker
     - retry, timeout, fault injection
     - Kiali, Prometheus, Grafana 관측성
```

구체적인 책임 경계는 다음처럼 둔다.

| 영역 | Kong 담당 | Istio 담당 |
| --- | --- | --- |
| 외부 진입점 | Client-facing API Gateway, LoadBalancer 뒤의 단일 진입점 | 기본 public ingress로 사용하지 않음 |
| API 라우팅 | `/api/auth`, `/api/concerts`, `/api/reservations` 같은 외부 path routing | 내부 서비스 간 host/subset routing |
| 인증 | JWT 토큰 존재와 서명 검증, Consumer 매핑, 인증 실패 차단 | 서비스 간 workload identity, mTLS peer authentication |
| 인가 | 공통 API 진입 정책 수준까지만 담당 | 서비스 간 접근 정책 또는 AuthorizationPolicy 적용 시 담당 |
| Rate Limiting | 외부 사용자/route/consumer 기준 제한 | 내부 mesh rate limit은 이번 기본 범위에서 제외 |
| Canary | 외부 API 자체의 단순 route 전환은 가능하나 기본 담당 아님 | `VirtualService`/`DestinationRule`로 v1/v2 20% -> 50% -> 100% 전환 |
| Circuit Breaker | Gateway upstream 보호 보조 수단 | 내부 의존 서비스 과부하 방지의 주 담당 |
| Retry/Timeout | 외부 요청의 gateway timeout 정도만 제한 | 서비스 간 retry/timeout 정책의 주 담당 |
| Observability | Gateway 요청량, 인증 실패, rate limit, latency | 서비스 간 topology, error rate, latency, Envoy resource |
| 보안 | 외부 API 인증과 edge 정책 | 내부 통신 암호화, 서비스 신원, namespace/service policy |

JWT 인증과 mTLS는 같은 보안 기능이 아니다. JWT는 외부 사용자 또는 클라이언트의 API 호출 자격을 검증하는 수단이고, mTLS는 서비스 workload 간 통신을 암호화하고 workload 신원을 검증하는 수단이다. 따라서 JWT는 Kong에서 먼저 검증하고, 서비스 내부 통신은 Istio mTLS로 보호한다. 도메인별 권한 검사는 각 서비스가 최종 책임을 가진다.

Istio `STRICT` mTLS를 적용하는 경우에는 Kong에서 application service로 들어가는 트래픽도 mesh 정책과 충돌하지 않게 별도 검증한다. 기본 방향은 Kong data plane이 mesh에 참여하도록 구성하는 것이다. 이것이 어려운 환경에서는 edge 구간만 `PERMISSIVE`로 두는 예외를 문서화하고, 내부 service-to-service 구간은 `STRICT`를 목표로 한다.

이번 프로젝트에서는 Istio IngressGateway를 별도 public ingress로 추가하지 않는다. Kong과 Istio IngressGateway를 둘 다 외부 진입점으로 두면 발표와 운영 문서에서 책임 경계가 흐려지고 디버깅 지점이 늘어난다. Istio IngressGateway는 추후 Kong을 대체하거나 multi-gateway ingress 구조가 필요할 때 별도 ADR로 검토한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| Kong만 사용 | 기존 작업과 이어지고 API Gateway, JWT, Rate Limit 구현이 단순하다. | 서비스 메시 요구사항인 mTLS, 내부 Canary, Envoy 기반 circuit breaker, Kiali topology 증거를 만들기 어렵다. | 채택하지 않음 |
| Istio만 사용하고 Kong 제거 | Istio Gateway와 VirtualService만으로 ingress와 mesh를 단일 기술로 구성할 수 있다. | 기존 Kong/GitOps 자산을 버리게 되고, API Gateway 플러그인 기반 JWT/Consumer/Rate Limit 증거가 약해진다. | 채택하지 않음 |
| Kong과 Istio를 역할 분리 없이 모두 사용 | 기능 선택지가 많다. | JWT, routing, canary, telemetry가 중복되어 장애 분석과 발표 설명이 어려워진다. | 채택하지 않음 |
| Kong -> Istio IngressGateway -> Services | Kong의 edge 기능과 Istio ingress 기능을 모두 보여줄 수 있다. | ingress 계층이 2개가 되어 timeout, TLS, header, access log, 장애 지점이 증가한다. 현재 요구사항 대비 복잡하다. | 현재는 채택하지 않음 |
| Kong edge + Istio internal mesh | 기존 Kong 자산을 유지하면서 서비스 메시 요구사항을 충족한다. 외부 API 정책과 내부 복원력 정책의 증거가 분리된다. | mTLS STRICT 적용 시 Kong-to-service 경계 검증이 필요하다. 두 제품의 운영 문서를 모두 관리해야 한다. | 채택 |

## 결과

좋아지는 점:

- Kong 작업은 API Gateway, JWT, Rate Limit, request ID, external route 검증으로 명확해진다.
- Istio 작업은 내부 Canary, retry, timeout, circuit breaker, mTLS, Kiali topology로 명확해진다.
- 발표에서 "왜 둘 다 쓰는가"를 외부 진입 정책과 내부 메시 정책의 차이로 설명할 수 있다.
- 기존 GitOps의 Kong route/plugin 자산을 유지하면서 심화 프로젝트의 서비스 메시 요구사항을 추가할 수 있다.
- 장애 분석 시 gateway 단계 문제인지, service mesh 단계 문제인지 구분하기 쉬워진다.

비용:

- Kong, Istio, Kubernetes NetworkPolicy가 함께 적용되므로 트래픽 차단 원인을 구분하는 runbook이 필요하다.
- mTLS STRICT 적용 전 Kong-to-service 경로가 mesh에 참여하는지 반드시 검증해야 한다.
- JWT 인증은 Kong에서 끝내지 않고 서비스의 도메인 권한 검사와 연결해야 한다.
- Canary 라우팅을 Kong과 Istio 양쪽에 동시에 구현하지 않도록 배포 책임 경계를 지켜야 한다.

## 적용 원칙

1. 외부 사용자가 호출하는 HTTP API는 Kong을 통과한다.
2. 서비스 간 내부 HTTP 호출의 라우팅, retry, timeout, circuit breaker는 Istio에서 관리한다.
3. Kafka 이벤트 흐름은 API Gateway의 대상이 아니며, 필요 시 Kafka exporter와 consumer lag 관측성으로 다룬다.
4. 인증은 Kong JWT 검증으로 1차 차단하고, 서비스별 권한 검사는 서비스 코드에서 수행한다.
5. 서비스 mesh 정책을 추가할 때는 `auth`, `concert`, `reservation`, `payment`, `ticket`, `notification` 순으로 영향도를 작게 나누어 검증한다.
6. Canary 전환은 Istio `VirtualService`와 `DestinationRule`을 기준으로 구현하고, Kong route는 같은 service host를 계속 바라보게 둔다.
7. mTLS STRICT를 켜기 전후로 Kong -> service, service -> service, service -> Kafka/PostgreSQL/MongoDB 경로를 각각 테스트한다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| todo | GitOps에 Istio 설치 및 application namespace sidecar injection 기준 추가 | unassigned | `gitops/` |
| todo | Kong data plane의 mesh 참여 여부 또는 mTLS edge 예외 정책 검증 | unassigned | `docs/personal/recommended-role-and-visual-flow.md` |
| todo | `reservation` 서비스 v1/v2 Canary 예제 작성: 20% -> 50% -> 100% | unassigned | `gitops/values/services/reservation*.yaml` |
| todo | `payment` 또는 `reservation` 의존 호출에 retry/timeout/circuit breaker 정책 적용 | unassigned | `gitops/` |
| todo | Kiali topology, Envoy resource metric, Grafana dashboard 증거 캡처 기준 작성 | unassigned | `workspace/docs/project_docs/` |
| todo | Kong JWT/Rate Limit/Request ID 검증 시나리오를 E2E 또는 runbook으로 정리 | unassigned | `service/contracts/` |
| todo | Kong, Istio, NetworkPolicy가 함께 차단할 때의 장애 분석 runbook 작성 | unassigned | `workspace/docs/project_docs/` |
