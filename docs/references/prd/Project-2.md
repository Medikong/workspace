## 🔹 프로젝트 개요

- **기본 프로젝트**: CI/CD 파이프라인 구축 및 서비스 배포 자동화
- **심화 프로젝트**: GitOps 도입 및 무중단 배포(Blue/Green, Canary) 고도화
- **서비스 예시**: 수동 배포 프로세스 → 완전 자동화된 GitOps 파이프라인으로 전환

## 🔹 요약 (빠르게 보기)

| 구분 | 영역 | 필수 과업 | 선택 과업 |
| --- | --- | --- | --- |
| **기본** | **파이프라인** | GitHub Actions/Jenkins 연동, 빌드/테스트 자동화 | 이미지 스캔(Trivy 등) 단계 추가 |
|  | **배포관리** | 환경별(Dev/Prod) 설정 분리, 기초 롤링 업데이트 | 배포 승인 프로세스(Approval) 구축 |
|  | **인프라 연동** | K8s 매니페스트 관리, ConfigMap/Secret 활용 | Vault 등 외부 비밀번호 관리 도구 연동 |
| **심화** | **GitOps** | ArgoCD 도입 및 선언적 배포 환경 구축 | GitOps 기반 인프라 변경 관리(IaC 통합) |
|  | **무중단배포** | Blue/Green 또는 Canary 배포 전략 구현 | 배포 후 자동 롤백(Metric 기반) 연동 |
|  | **최적화** | 파이프라인 실행 시간 단축, 캐싱 전략 최적화 | 다중 클러스터 배포 자동화 파이프라인 |

## 🔹 상세 구현 기준

### **📖 기본 프로젝트 —** 쿠버네티스 기반 애플리케이션 배포 자동화

> **서비스 예시**: 핀테크 결제 서비스 (계좌·이체·대출·알림 서비스)
> 

### (1) CI/CD 파이프라인 설계

**`필수`**

- Jenkins와 GitHub Actions의 학습 비용·연동 용이성·병렬 처리 지원을 비교하고 선택 근거를 ADR 문서로 작성한다.
- GitHub Actions로 서비스별 단위 테스트 → Docker 빌드 → Container Registry 푸시 → Kubernetes 배포 단계의 파이프라인을 구성한다. path filter를 적용하여 변경된 서비스만 빌드·배포되도록 한다.
- 배포 성공·실패 결과를 Slack `#deploy-status` 채널에 자동 발송한다.

**`선택`**

- 4개 서비스의 빌드 Job을 병렬로 실행하여 전체 배포 시간을 단축하고 개선 수치를 측정한다.
- prod 환경 배포는 GitHub Environment Protection Rule로 승인자 확인을 강제한다.

---

### (2) 컨테이너 이미지 빌드 및 레지스트리

**`필수`**

- 각 서비스의 Dockerfile을 빌드 스테이지와 런타임 스테이지로 분리하는 멀티스테이지 빌드로 작성하고, 비루트 사용자(`appuser`, UID 1001) 실행으로 설정한다.
- KT클라우드 Container Registry에 서비스별 저장소를 분리하고 이미지 태그를 `git-sha`로 관리한다.
- Trivy를 CI 파이프라인에 통합하여 HIGH/CRITICAL CVE 발견 시 레지스트리 푸시를 차단한다.

**`선택`**

- Trivy 취약점 스캔 결과를 GitHub PR 코멘트로 자동 게시한다.
- untagged 이미지 자동 삭제 수명 주기 정책을 레지스트리에 설정한다.
- Dependabot으로 베이스 이미지 최신화 자동화 PR을 주간으로 생성한다.

---

### (3) 쿠버네티스 배포 자동화

**`필수`**

- 각 서비스에 Deployment + HPA(CPU 70% 기준, 최소 2·최대 10 Replica)를 구성한다.
- Readiness Probe(`/health/ready`, DB 연결 확인)와 Liveness Probe(`/health`, 프로세스 생존 확인)를 설정하여 배포 중 트래픽 단절이 없도록 한다.
- Helm Chart로 서비스 배포 설정을 관리하고 Rolling Update 배포 전략을 적용한다.

**`선택`**

- ArgoCD를 클러스터에 배포하고 Helm Chart 저장소와 연결하여 기본 GitOps 사이클(Git 변경 → 자동 Sync)을 구성한다.
- dev/prod 환경을 values 파일로 분리하고 ArgoCD Application으로 환경별 배포를 관리한다.
- 이체 서비스에 Canary 배포(신규 버전 20% → 100%)를 추가로 구현한다.

### **📖 심화 프로젝트 —** 고급 네트워크 통합 및 서비스 메시 구현

> **서비스 예시**: 기본 프로젝트의 서비스를 이어서 고도화
> 

### (1) 서비스 메시 도입 전략

**`필수`**

- Istio와 Linkerd의 기능·리소스 사용량·학습 난이도를 비교 분석하고 선택 근거를 ADR 문서로 작성한다.
- Istio를 클러스터에 설치하고 서비스 Namespace에 사이드카 자동 주입을 활성화한다.
- VirtualService와 DestinationRule을 정의하여 이체 서비스에 Canary 라우팅(신규 버전 20% → 50% → 100% 단계적 전환)을 적용한다.

**`선택`**

- PeerAuthentication으로 Namespace 전체에 mTLS STRICT 모드를 활성화하여 서비스 간 통신을 암호화하고 Kiali에서 mTLS 적용 현황을 시각적으로 확인한다.
- 트래픽 라우팅(블루-그린, 카나리) 및 관찰성(Observability) 강화 설정을 추가로 구성한다.

---

### (2) 서비스 메시 운영 및 모니터링

**`필수`**

- Kiali를 배포하여 서비스 토폴로지(서비스 간 트래픽 흐름, 에러율)를 시각화한다.
- Istio 사이드카 프록시(Envoy)의 메모리·CPU 사용량을 Prometheus로 수집하고 Grafana에서 모니터링한다.
- DestinationRule의 기본 Circuit Breaker(connectionPool 설정)를 구성하여 서비스 과부하를 방지한다.

**`선택`**

- Jaeger를 배포하고 Istio와 연동하여 서비스 간 요청의 분산 트레이싱을 수집한다. P99 응답 시간 초과 요청에 대한 Trace를 즉시 조회할 수 있도록 샘플링 전략을 설정한다.
- outlierDetection으로 연속 5회 5xx 응답 시 Pod를 30초 ejection하는 Circuit Breaker를 구성하고 실제 장애 주입으로 동작을 검증한다.
- 분산 트레이싱(Jaeger/Zipkin) 연동 후 서비스 간 병목 구간을 정량적으로 분석하고 개선 방안을 도출한다.

---

### (3) 네트워크 정책 및 장애 복구

**`필수`**

- Kubernetes NetworkPolicy를 정의하여 Namespace 내부 서비스 간 통신만 허용하고 외부 직접 접근을 차단한다.
- Pod 강제 종료 장애 시나리오 1가지를 수행하고 Istio Retry 정책의 동작을 확인한다.
- 이전 버전으로의 롤백 절차(ArgoCD 이전 Revision 복원 또는 VirtualService 가중치 즉시 전환)를 Runbook으로 작성한다.

**`선택`**

- 네트워크 지연(200ms 주입, Istio fault injection)·서비스 다운 등 2가지 이상 추가 장애 시나리오를 수행하고 Istio Circuit Breaker 동작 결과를 측정·기록한다.
- 장애 발생 시 롤백 절차를 자동화 스크립트로 구현하여 5분 이내 복구를 목표로 검증한다.
- Pod 간 통신 차단·허용 시나리오를 NetworkPolicy로 세분화하여 테스트한다.