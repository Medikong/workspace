---
id: ADR-0009
title: "AWS dev NLB는 외부 80 포트로 받고 Kong NodePort 32407로 전달한다"
status: accepted
date: 2026-06-12
areas:
  - aws-networking
  - api-gateway
  - ingress
  - security
repos:
  - workspace
  - infra
  - gitops
decision_drivers:
  - 외부 사용자 URL에서 Kubernetes NodePort 포트 노출을 줄인다.
  - self-managed Kubernetes 환경의 Kong NodePort 구조는 유지한다.
  - HTTPS/TLS 적용은 도메인과 인증서 결정 이후 후속 단계로 분리한다.
  - dev 환경에서는 검증 단순성과 실무 구조 설명 가능성을 함께 만족해야 한다.
related:
  - docs/adr/0003-separate-kong-edge-gateway-and-istio-service-mesh.md
  - docs/members/service/personal/execution/aws-dev-observability-smoke/nlb-kong-entrypoint-check-2026-06-12.md
links: []
supersedes: []
superseded_by: null
---

# AWS dev NLB는 외부 80 포트로 받고 Kong NodePort 32407로 전달한다

## 상태

Accepted

## 날짜

2026-06-12

## 배경

AWS dev 환경은 EKS가 아니라 EC2 기반 self-managed Kubernetes 클러스터다. 따라서 Kubernetes `Service`를 `LoadBalancer` 타입으로 선언한다고 해서 클라우드 로드밸런서가 자동으로 생성되는 구조가 아니다.

현재 Kong Gateway는 Kubernetes 내부에서 NodePort `32407`로 노출되고, Terraform이 별도 AWS NLB를 생성해 worker node의 `32407` 포트로 트래픽을 전달한다.

초기 검증 구조는 다음과 같았다.

```text
Client
  -> NLB:32407
  -> worker node:32407
  -> Kong Proxy NodePort
  -> Kong Gateway
  -> ticketing service ClusterIP
```

이 구조는 동작하지만 외부 URL에 Kubernetes NodePort 번호가 그대로 드러난다.

```text
http://<nlb-dns>:32407/auth/demo-accounts
```

dev 검증에는 사용할 수 있지만, API Gateway를 외부 진입점으로 설명할 때는 표준 HTTP 포트인 `80`을 외부 리스너로 두는 편이 더 자연스럽다.

## 결정

AWS dev NLB의 외부 listener port는 `80`으로 변경한다.

Kong Proxy의 Kubernetes NodePort와 NLB target port는 기존처럼 `32407`을 유지한다.

변경 후 트래픽 흐름은 다음과 같다.

```text
Client
  -> NLB:80
  -> worker node:32407
  -> Kong Proxy NodePort
  -> Kong Gateway
  -> ticketing service ClusterIP
```

사용자 관점의 호출 URL은 다음처럼 단순해진다.

```text
http://<nlb-dns>/auth/demo-accounts
```

Terraform 기준 설정은 다음과 같다.

```hcl
nlb_listener_port = 80
nlb_target_port   = 32407
```

HTTPS `443` 적용은 이번 결정에 포함하지 않는다. 인증서, 도메인, TLS 종료 위치를 별도로 결정해야 하기 때문이다.

## 대안

### 1. NLB 32407 -> NodePort 32407 유지

가장 단순하고 이미 검증된 구조다.

하지만 외부 사용자가 `:32407` 포트를 직접 붙여 호출해야 하므로 Gateway 진입점으로 보기 어색하다. Kubernetes 내부 구현인 NodePort가 외부 계약처럼 보이는 문제도 있다.

### 2. NLB 80 -> NodePort 32407

이번에 선택한 구조다.

외부 사용자는 표준 HTTP 포트로 접근하고, 클러스터 내부 구현은 기존 NodePort를 유지한다. self-managed Kubernetes에서 cloud controller 없이 NLB를 붙이는 현재 구조와도 잘 맞는다.

### 3. NLB 443 -> NodePort 32407

운영 환경에 가장 가까운 구조다.

다만 ACM 인증서, 도메인, TLS termination 위치를 결정해야 한다. NLB에서 TLS를 종료할지, Kong에서 TLS를 종료할지에 따라 인증서 관리 방식과 보안 그룹 정책이 달라진다.

### 4. Kong `Service type=LoadBalancer` 사용

EKS나 cloud provider integration이 구성된 클러스터에서는 자연스러운 방식이다.

현재 클러스터는 EC2 기반 self-managed Kubernetes라서 이 방식을 바로 쓰기 어렵다. 따라서 Terraform으로 NLB를 명시적으로 생성하고 Kong NodePort로 연결하는 방식을 유지한다.

## 결과

Terraform 기본값과 예시 변수에서 `nlb_listener_port`는 `80`이 된다.

실제 AWS dev 환경에 반영하려면 `infra` repo에서 Terraform plan/apply를 수행해야 한다. 코드 변경만으로 기존 NLB listener가 자동 변경되지는 않는다.

Kong NodePort `32407`은 여전히 worker node에서 열려 있어야 한다. NLB target group이 worker node의 `32407`로 health check와 forwarding을 수행하기 때문이다.

보안 그룹 관점에서는 다음을 구분해야 한다.

- 외부 사용자의 진입 포트: NLB listener `80`
- NLB가 worker node로 전달하는 포트: NodePort `32407`
- dev에서 임시로 허용된 직접 접근: `EC2 public IP:32407`

dev 환경에서는 `EC2 public IP:32407` 직접 접근이 남아 있을 수 있다. 운영 기준으로는 사용자가 직접 NodePort로 접근하지 못하도록 NLB 경유만 허용하는 보안 그룹 구성이 필요하다.

## 후속 작업

1. Terraform plan으로 NLB listener가 `32407`에서 `80`으로 바뀌는지 확인한다.
2. Terraform apply 후 `http://<nlb-dns>/auth/demo-accounts` 호출을 검증한다.
3. 기존 `http://<nlb-dns>:32407/...` 호출이 더 이상 외부 계약이 아님을 문서에 반영한다.
4. HTTPS가 필요해지는 시점에 `443`, ACM 인증서, TLS 종료 위치를 별도 ADR로 결정한다.
5. 운영 환경에서는 직접 NodePort 접근을 제한하고 NLB를 유일한 외부 진입점으로 둔다.
