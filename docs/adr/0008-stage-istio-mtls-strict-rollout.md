---
id: ADR-0008
title: "Istio mTLS STRICT는 mesh 내부부터 단계적으로 적용한다"
status: accepted
date: 2026-06-08
areas:
  - service-mesh
  - security
  - traffic-management
repos:
  - workspace
  - gitops
decision_drivers:
  - Kong이 외부 API Gateway로 남아 있는 동안 Kong-to-service 경로를 끊지 않아야 한다.
  - 모든 서비스 Pod에 sidecar가 붙기 전 mTLS STRICT를 켜면 정상 트래픽이 차단될 수 있다.
  - mTLS 적용 증거는 Kiali와 kubectl 검증으로 남겨야 한다.
related:
  - ADR-0003
  - ADR-0007
  - workspace/docs/members/service/incident-recovery-runbook.md
links:
  - https://istio.io/latest/docs/concepts/security/
  - https://istio.io/latest/docs/tasks/security/authentication/mtls-migration/
  - https://istio.io/latest/docs/reference/config/security/peer_authentication/
supersedes: []
superseded_by: null
---

# ADR 0008: Istio mTLS STRICT는 mesh 내부부터 단계적으로 적용한다

## 상태

Accepted

## 날짜

2026-06-08

## 배경

공연 티켓 서비스는 Kong을 외부 API Gateway로, Istio를 내부 Service Mesh로 분리해서 사용한다. Kong은 외부 요청의 JWT 인증, Role Guard, Rate Limit, Request ID를 담당하고, Istio는 내부 서비스 간 retry, timeout, canary, circuit breaker, topology 관측을 담당한다.

Istio mTLS `STRICT`를 namespace 전체에 바로 적용하면 sidecar가 없는 workload 또는 mesh 밖에서 들어오는 요청이 차단될 수 있다. 현재 Kong이 mesh 밖에 있으면 Kong에서 service Pod로 들어오는 요청이 mTLS 정책과 충돌할 수 있다. 또한 모든 서비스의 sidecar injection과 API smoke test가 끝나지 않은 상태에서 STRICT를 켜면 장애 원인을 구분하기 어렵다.

## 결정

mTLS는 다음 순서로 적용한다.

```text
1. PERMISSIVE 상태에서 sidecar injection과 API smoke test를 먼저 완료한다.
2. Kiali에서 서비스 간 traffic과 mTLS 상태를 확인한다.
3. mesh 내부 service-to-service 경로부터 STRICT 적용 후보로 검토한다.
4. Kong -> service 경로는 Kong이 mesh에 참여하는지 확인하기 전까지 예외 또는 PERMISSIVE로 둔다.
5. 모든 서비스의 sidecar 적용과 Kong 경유 smoke test가 안정화된 뒤 namespace 단위 STRICT를 검토한다.
```

기본 원칙:

- 외부 사용자의 API 요청은 계속 Kong을 통과한다.
- mTLS STRICT의 첫 적용 대상은 외부 edge가 아니라 mesh 내부 서비스 간 호출이다.
- Kong이 mesh 밖에 있는 동안 Kong-to-service 경로를 끊는 STRICT 정책은 기본값으로 넣지 않는다.
- STRICT 적용 manifest는 기본 GitOps 상태에 바로 포함하지 않고, 검증 scenario 또는 별도 rollout 단계로 관리한다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| 모든 ticketing namespace에 즉시 STRICT 적용 | 요구사항 증거를 빠르게 만들 수 있다. | Kong-to-service, non-sidecar workload, DB/Kafka 경로를 끊을 수 있다. | 채택하지 않음 |
| mTLS를 적용하지 않음 | 단기 장애 위험이 낮다. | 서비스 메시 보안 요구사항 증거가 부족하다. | 채택하지 않음 |
| Kong도 sidecar injection해서 mesh에 포함 | Kong-to-service mTLS 경계가 명확해질 수 있다. | Kong 운영 방식과 sidecar 적용 리스크를 추가 검증해야 한다. | 검토 후보 |
| 내부 mesh traffic부터 단계적으로 STRICT 적용 | 장애 범위를 줄이고 원인 분리가 쉽다. | 최종 STRICT 완료까지 단계가 늘어난다. | 채택 |

## 결과

좋아지는 점:

- Kong Gateway 경로를 갑자기 끊지 않는다.
- sidecar injection과 mTLS 문제를 분리해서 확인할 수 있다.
- Kiali에서 mTLS 적용 상태를 증거로 남기기 쉽다.
- 장애 발생 시 PERMISSIVE 또는 정책 제거로 빠르게 되돌릴 수 있다.

비용:

- mTLS STRICT 요구사항은 즉시 완료가 아니라 단계 적용 상태로 남는다.
- Kong을 mesh에 포함할지, edge만 예외로 둘지 추가 결정이 필요하다.
- NetworkPolicy, AuthorizationPolicy와 함께 적용할 때 차단 원인 분석 Runbook이 필요하다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| todo | Kong이 mesh 밖에 있는지, sidecar injection 대상인지 AWS dev에서 확인 | service | `gitops/platform/kong` |
| todo | sidecar 적용 서비스의 Kiali mTLS 상태 확인 | service | `workspace/docs/members/service/incident-recovery-runbook.md` |
| todo | PERMISSIVE PeerAuthentication scenario 작성 | service | TBD |
| todo | 내부 mesh 대상 STRICT scenario 작성 | service | TBD |
| todo | Kong 경유 API smoke test 후 STRICT 확대 여부 결정 | service | TBD |
