# 공연 티켓 서비스 전체 진행 Phase/Task 계획

기준 문서: [requirements-goal-audit-and-task-breakdown.md](./requirements-goal-audit-and-task-breakdown.md)

이 문서는 현재 `service`, `gitops`, `infra` repo 상태를 기준으로 전체 요구사항을 어떤 순서로 진행할지 정리한다. 1차와 2차로 나누지만, 2차도 최종 충족 대상이다.

## 현재 코드 기준 요약

| Repo | 현재 상태 | 계획에 주는 의미 |
| --- | --- | --- |
| `service` | 티켓팅 서비스(`auth`, `concert`, `reservation`, `payment`, `ticket`, `notification`)와 의료 레거시(`patient`, `appointment`, `prescription`)가 함께 존재 | 1차 첫 작업은 테스트/CI/E2E 기준을 티켓팅으로 바꾸는 것 |
| `service` | `contracts/services/*`에는 티켓팅 OpenAPI가 6개 서비스 기준으로 정리됨 | API 계약은 `service/contracts`를 기준으로 삼는 것이 맞음 |
| `service` | `Taskfile.yml`의 `APP_SERVICES`, E2E compose, Newman collection은 아직 의료 도메인 기준 | CI가 실제 티켓팅 요구사항을 검증하지 못함 |
| `service` | Dockerfile은 multi-stage와 non-root는 되어 있으나 UID가 `10001`이고 사용자명이 `app` | 요구사항의 `appuser`, UID `1001`과 불일치 |
| `service` | `payment-service`는 `PaymentEvent`를 DB outbox에 저장하지만 Kafka publish worker가 없음 | `payment-approved -> ticket-issued -> notification` 실제 E2E가 끊길 수 있음 |
| `service` | `payment-approved` payload에 `ticket-service`가 요구하는 `userId`, `seatId`, `producer`, `sourceId`가 빠짐 | 이벤트 계약 정합성 보완이 선행되어야 함 |
| `gitops` | `values/services/*.yaml`과 `argo/applications/aws-dev/services/*.yaml`는 티켓팅 기준으로 전환됨 | GitOps 서비스 배포 기반은 활용 가능 |
| `gitops` | 공통 Helm chart에 `Deployment`, `Service`, `Ingress`, `HPA`, `PDB`, `ServiceMonitor`, `NetworkPolicy`, `RBAC` 템플릿이 있음 | 요구사항 다수를 values 변경과 템플릿 보강으로 충족 가능 |
| `gitops` | `values/env/dev.yaml`은 `hpa`, `pdb`, `serviceMonitor`가 꺼져 있고, `aws-dev/prod`는 켜져 있음 | 로컬 검증과 AWS 검증 기준을 분리해야 함 |
| `gitops` | observability stack은 Prometheus/Grafana/Loki/Alloy/Tempo 기준으로 존재 | 원문 EFK 요구사항과 Loki 기반 현 구조 사이의 선택/ADR 필요 |
| `gitops` | Kong route/plugin/consumer 기반이 존재 | API Gateway/JWT/Rate Limit은 Kong으로 빠르게 검증 가능 |
| `infra` | Terraform ECR repository 목록이 의료 서비스 기준 | AWS image push 전에 반드시 티켓팅 서비스 목록으로 변경 필요 |
| `infra` | `infra/k8s`는 의료 Kustomize 레거시가 많음 | 새 배포 기준은 `gitops`로 두고, `infra/k8s`는 archive 또는 제거 판단 필요 |
| `infra` | kubeadm/observability 설치 자동화는 Prometheus/Grafana/Loki/Tempo 기준으로 존재 | 클러스터와 관측성 설치 기반은 재사용 가능 |

## 전체 Phase 순서

| Phase | 차수 | 목표 | 핵심 산출물 |
| --- | --- | --- | --- |
| 0 | 1차-A | 기준선 정리 | 레거시 영향 범위 제거, 티켓팅 기준 CI/E2E 대상 확정 |
| 1 | 1차-A | 서비스 계약과 핵심 E2E 완성 | 예매 -> 결제 -> 티켓 -> 알림 흐름 |
| 2 | 1차-A | 로컬 Kubernetes/GitOps 배포 안정화 | `task dev:platform`, `task dev:services` 성공 |
| 3 | 1차-A | CI/CD와 이미지 보안 기본선 | path filter, Docker build/push/deploy, Trivy image scan |
| 4 | 1차-A | 기본 모니터링 | Prometheus, ServiceMonitor, 기본 Grafana |
| 5 | 1차-A | 기본 로깅과 요청 추적 | 로그 수집, request_id trace |
| 6 | 1차-B | 대시보드/알림/SLA | Alertmanager Slack, SLA 기준, 운영 대시보드 |
| 7 | 1차-B | Kong + Istio 통합 | Kong edge, Istio mesh, Canary, rollback |
| 8 | 1차-B | 성능/HPA 검증 | k6 baseline, HPA scale-out, Before/After |
| 9 | 1차-B | 보안/접근 제어 | RBAC, ServiceAccount, NetworkPolicy, Trivy config |
| 10 | 2차 | 심화 로깅/추적/운영 자동화 | Fluent Bit 비교, ES/Splunk ADR, ILM, tracing, runbook |
| 11 | 2차 | 고급 장애 대응/최종 보고 | Chaos, KEDA, incident automation, 운영 보고서 |

## Phase 0. 기준선 정리

목표는 "티켓팅 프로젝트인데 의료 레거시가 CI와 배포 판단을 흔드는 상태"를 먼저 끊는 것이다.

### Task 0.1 서비스 목록을 티켓팅 기준으로 고정

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/Taskfile.yml`, `service/README.md`, `service/tests/README.md` |
| 현재 상태 | `APP_SERVICES`가 `auth`, `patient`, `appointment`, `prescription`, `notification`, `payment` 기준 |
| 작업 | `APP_SERVICES`를 `auth-service concert-service reservation-service payment-service ticket-service notification-service`로 변경 |
| 작업 | `install` task의 requirements 설치 목록도 티켓팅 서비스 기준으로 변경 |
| 작업 | README의 서비스 설명을 의료 도메인에서 공연 티켓 도메인으로 변경 |
| 완료 기준 | `task test-unit`이 티켓팅 서비스만 대상으로 실행됨 |
| 의존성 | 없음 |

### Task 0.2 의료 레거시 보존/제거 정책 결정

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `infra` |
| 대상 파일 | `service/services/patient-service`, `service/services/appointment-service`, `service/services/prescription-service`, `infra/k8s/**` |
| 현재 상태 | 레거시 코드가 active tree에 남아 있음 |
| 작업 | 당장 삭제하지 않을 경우 `archive/` 또는 문서상 legacy로 명시 |
| 작업 | CI, 이미지 빌드, E2E, GitOps에서 레거시가 실행되지 않도록 제거 |
| 완료 기준 | `rg "patient|appointment|prescription|medical" service/Taskfile.yml service/.github service/tests gitops/values gitops/argo infra/terraform` 결과가 의도된 archive/문서 외에는 없음 |
| 의존성 | Task 0.1 |

### Task 0.3 Dockerfile non-root 요구사항 정렬

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/services/*-service/Dockerfile`, `service/dashboard/Dockerfile` |
| 현재 상태 | multi-stage는 대체로 적용, non-root UID는 `10001`, 사용자명은 `app` |
| 요구사항 | `appuser`, UID `1001` |
| 작업 | 티켓팅 6개 서비스 Dockerfile을 `appuser`, UID `1001` 기준으로 통일 |
| 작업 | dashboard는 `nginxinc/nginx-unprivileged` 사용 근거를 문서화하거나 동일 기준으로 조정 |
| 완료 기준 | `docker run --rm <image> id`에서 UID 1001 또는 예외 근거 확인 |
| 의존성 | Task 0.1 |

### Task 0.4 Infra ECR repository 목록 수정

| 항목 | 내용 |
| --- | --- |
| repo | `infra` |
| 대상 파일 | `infra/terraform/variables.tf`, `infra/terraform/terraform.tfvars.example` |
| 현재 상태 | `patient-service`, `appointment-service`, `prescription-service`가 남아 있음 |
| 작업 | `auth-service`, `concert-service`, `reservation-service`, `payment-service`, `ticket-service`, `notification-service`, `dashboard`로 변경 |
| 완료 기준 | Terraform plan에서 티켓팅 ECR repository만 생성 대상 |
| 의존성 | Task 0.1 |

## Phase 1. 서비스 계약과 핵심 E2E 완성

목표는 이후 모니터링, 로깅, 성능, 장애 실험에 쓸 실제 업무 흐름을 만드는 것이다.

### Task 1.1 OpenAPI 기준 확정

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `workspace` |
| 대상 파일 | `service/contracts/services/*/openapi.yaml`, `workspace/docs/project_docs/openapi/**` |
| 현재 상태 | `service/contracts`가 더 최신이고 6개 티켓팅 서비스를 포함 |
| 작업 | OpenAPI 기준 위치를 `service/contracts`로 확정 |
| 작업 | `workspace` 쪽 OpenAPI 문서는 mirror/reference인지 archive인지 결정 |
| 완료 기준 | 서비스별 route와 OpenAPI path가 1차 E2E에 필요한 범위에서 일치 |
| 의존성 | Phase 0 |

### Task 1.2 이벤트 계약 정합성 수정

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `payment-service`, `ticket-service`, `notification-service` schemas/tests |
| 현재 상태 | `payment-service` outbox payload에는 `userId`, `seatId`, `producer`, `sourceId`가 빠짐. `ticket-service`는 이를 필수로 기대 |
| 작업 | `payment-approved` event payload 표준 필드 확정: `eventId`, `eventType`, `sourceId`, `paymentId`, `reservationId`, `concertId`, `seatId`, `userId`, `producer`, `occurredAt`, `correlationId` |
| 작업 | `CreatePaymentRequest`에 `seatId`가 없으면 reservation 조회 API 또는 payment 요청 payload에 포함시키는 방식 결정 |
| 작업 | `ticket-issued` event payload도 notification이 요구하는 필드와 맞춤 |
| 완료 기준 | payment unit test, ticket consumer unit test, notification event test가 같은 fixture를 공유 |
| 의존성 | Task 1.1 |

### Task 1.3 Kafka publish worker 구현

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/services/payment-service/app/**` |
| 현재 상태 | `PaymentEvent`는 DB에 저장되지만 Kafka로 발행하는 worker/producer가 없음 |
| 작업 | outbox publisher를 payment-service 내부 background task 또는 별도 worker로 구현 |
| 작업 | 발행 성공 시 outbox status/processed timestamp를 저장 |
| 작업 | 실패 시 retry 가능한 상태로 남김 |
| 완료 기준 | `payment-approved` topic에 event가 실제 발행되고 `ticket-service` consumer가 처리 |
| 의존성 | Task 1.2 |

### Task 1.4 티켓팅 Docker Compose E2E 작성

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/tests/e2e/docker-compose.yml`, `postgres-init`, `postman`, `newman`, `wait-for-services.sh` |
| 현재 상태 | 의료 E2E: patient -> appointment -> prescription |
| 작업 | auth, concert, reservation, payment, ticket, notification, postgres, mongo, kafka 기준 compose로 변경 |
| 작업 | DB 이름을 `auth_db`, `concert_db`, `reservation_db`, `payment_db`, `ticket_db`, `notification_db`로 맞춤 |
| 작업 | Kafka topic을 `reservation-created`, `reservation-expired`, `payment-approved`, `payment-failed`, `ticket-issued`로 변경 |
| 완료 기준 | `task test-e2e`가 예매 -> 결제 승인 -> 티켓 발행 -> 알림 조회를 검증 |
| 의존성 | Task 1.2, Task 1.3 |

### Task 1.5 단위/통합 테스트 확장

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/services/*/tests/**` |
| 작업 | 좌석 중복 예매 conflict 테스트 |
| 작업 | payment idempotency 테스트 |
| 작업 | ticket duplicate event idempotency 테스트 |
| 작업 | notification duplicate event idempotency 테스트 |
| 작업 | `/healthz`, `/readyz`, `/metrics` smoke test를 6개 서비스에 통일 |
| 완료 기준 | `task test-unit`과 `task test-e2e` 성공 |
| 의존성 | Task 1.4 |

## Phase 2. 로컬 Kubernetes/GitOps 배포 안정화

목표는 `gitops`의 최신 local dev 흐름을 기준으로 전체 서비스를 Kubernetes에 반복 배포할 수 있게 만드는 것이다.

### Task 2.1 Helm values와 서비스 코드 포트/환경변수 일치 확인

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service` |
| 대상 파일 | `gitops/values/services/*.yaml`, 서비스 `config.py` |
| 현재 상태 | 서비스별 values는 티켓팅 기준이나 payment/ticket event payload와 env는 추가 보완 필요 |
| 작업 | DB URL, Kafka bootstrap, topic 이름, JWT secret/issuer를 서비스 코드와 대조 |
| 작업 | `notification-service`, `ticket-service` Kafka consumer가 local k8s에서 시작 가능한지 확인 |
| 완료 기준 | `task helm:template:all` 성공, 모든 서비스 `/readyz` 성공 |
| 검증 명령 | `task --dir gitops validate` |
| 의존성 | Phase 1 |

### Task 2.2 Probe 경로를 운영 기준으로 통일

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 대상 파일 | `gitops/values/base.yaml`, `gitops/charts/medikong-service/values.yaml` |
| 현재 상태 | 기본 probe path가 `/health` |
| 요구사항 | Readiness `/health/ready`, Liveness `/health`로 제시되어 있으나 현재 서비스는 `/readyz`, `/healthz`, `/health` |
| 결정 | 실제 코드 기준으로 Liveness `/healthz`, Readiness `/readyz`를 표준화하고 요구사항 문서에 mapping |
| 작업 | Helm 기본 readiness/liveness path를 `/readyz`, `/healthz`로 변경 |
| 완료 기준 | rolling update 중 readiness가 DB 연결 상태를 반영 |
| 의존성 | Task 1.5 |

### Task 2.3 Local dev platform 배포 검증

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 대상 파일 | `gitops/Taskfile.yml`, `platform/namespaces`, `platform/data`, `platform/kong` |
| 현재 상태 | `task dev:platform`, `dev:data`, `dev:kong` 존재 |
| 작업 | Docker Desktop Kubernetes 기준 namespace/data/Kong 배포 |
| 검증 명령 | `task --dir gitops dev:platform` |
| 완료 기준 | Postgres/Mongo/Kafka StatefulSet rollout, Kong proxy localhost 접근 |
| 의존성 | Task 2.1 |

### Task 2.4 전체 서비스 배포 검증

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `gitops` |
| 현재 상태 | `gitops`가 `service` Taskfile의 `app-images-push`를 호출 |
| 작업 | `task --dir gitops dev:services` 또는 `task --dir gitops dev`로 전체 배포 |
| 작업 | 실패 시 `task --dir gitops dev:service SERVICE=<name>` 단위로 분리 |
| 완료 기준 | 6개 서비스와 dashboard Deployment rollout 성공 |
| 의존성 | Task 2.3 |

### Task 2.5 Kong route/JWT/Rate Limit smoke 검증

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service` |
| 대상 파일 | `gitops/platform/kong/plugins/*.yaml`, `gitops/values/services/*.yaml` |
| 현재 상태 | Kong plugins와 ingress annotations 존재 |
| 작업 | `/auth`는 JWT 없이 접근 가능, 나머지는 JWT 필요하도록 검증 |
| 작업 | Rate Limit 과다 요청 시 429 확인 |
| 작업 | correlation-id가 서비스 로그까지 전달되는지 확인 |
| 완료 기준 | curl/Newman smoke script와 캡처 |
| 의존성 | Task 2.4 |

## Phase 3. CI/CD와 이미지 보안 기본선

### Task 3.1 GitHub Actions path filter 재구성

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/.github/workflows/ci.yml`, `service/.github/workflows/e2e.yml` |
| 현재 상태 | push/PR 때 전체 `task test-unit`, `task test-e2e` 실행 |
| 작업 | 서비스별 path filter 또는 matrix 구성 |
| 작업 | 변경된 서비스만 unit test, image build 대상이 되도록 구성 |
| 완료 기준 | PR에서 변경 서비스 job만 실행 |
| 의존성 | Phase 1 |

### Task 3.2 Docker build/push workflow 구성

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `gitops`, `infra` |
| 현재 상태 | service Taskfile에 image build/push target 존재, infra ECR 목록은 수정 필요 |
| 작업 | git-sha tag로 image build |
| 작업 | KT Cloud 또는 AWS/ECR registry 로그인 step 구성 |
| 작업 | 성공 시 GitOps values image tag 업데이트 또는 deploy job 실행 |
| 완료 기준 | 단일 서비스 변경 -> image push -> k8s deploy 흐름 |
| 의존성 | Task 0.4, Task 3.1 |

### Task 3.3 Trivy image scan gate

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 작업 | Docker push 전에 Trivy HIGH/CRITICAL 차단 |
| 작업 | 2차에서 PR comment 자동 게시 추가 |
| 완료 기준 | 취약 이미지 push 차단 로그 |
| 의존성 | Task 3.2 |

### Task 3.4 배포 결과 Slack 알림

| 항목 | 내용 |
| --- | --- |
| repo | `service` 또는 `gitops` |
| 작업 | `#deploy-status` 성공/실패 알림 |
| 작업 | 실패 시 service, image tag, workflow run URL 포함 |
| 완료 기준 | 테스트 알림 캡처 |
| 의존성 | Task 3.2 |

## Phase 4. 기본 모니터링

### Task 4.1 kube-prometheus-stack 설치 경로 확정

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `infra` |
| 현재 상태 | `gitops/cluster/stacks/observability`와 `infra/infra/cluster/stacks/observability`에 Prometheus/Grafana/Loki/Alloy/Tempo values 존재 |
| 결정 | 서비스 배포와 함께 관리할 기준은 `gitops/cluster/stacks/observability`로 둠 |
| 작업 | `monitoring` 또는 `observability` namespace 명칭을 요구사항에 맞춰 결정. 원문은 `monitoring` |
| 완료 기준 | kube-prometheus-stack 설치, Grafana 접속 |
| 의존성 | Phase 2 |

### Task 4.2 ServiceMonitor 활성화

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 대상 파일 | `gitops/values/env/dev.yaml`, `aws-dev.yaml`, `aws-prod.yaml`, `values/services/*.yaml` |
| 현재 상태 | dev는 `serviceMonitor.enabled: false`, aws는 true |
| 작업 | 1차 로컬 검증에는 dev에서도 ServiceMonitor를 켜거나 별도 `local-observability` env values 생성 |
| 완료 기준 | Prometheus targets에 6개 서비스가 나타남 |
| 의존성 | Task 4.1 |

### Task 4.3 서비스 metric 표준화

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | 각 서비스 `observability.py`, `main.py` |
| 현재 상태 | `/metrics` endpoint는 존재. 도메인 custom metric은 제한적 |
| 작업 | 예매 요청/성공/conflict, 결제 성공/실패, 티켓 발행, notification 처리량 metric 추가 |
| 작업 | 공통 label: `service`, `method`, `path`, `status`, `error_code` |
| 완료 기준 | Grafana에서 공연 티켓 업무 지표 조회 가능 |
| 의존성 | Phase 1 |

### Task 4.4 PrometheusRule/recording rule

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | 에러율 5%, P99 2초, CrashLoopBackOff alert |
| 작업 | 5분 평균 에러율 recording rule |
| 완료 기준 | PrometheusRule CRD 적용, 테스트 alert 발생 |
| 의존성 | Task 4.2, Task 4.3 |

## Phase 5. 기본 로깅과 요청 추적

### Task 5.1 로깅 스택 선택 정리

| 항목 | 내용 |
| --- | --- |
| repo | `workspace`, `gitops` |
| 현재 상태 | GitOps는 Loki/Alloy 중심. 원문은 Fluentd/Elasticsearch/Kibana, 심화는 Fluent Bit/Logstash/Elasticsearch |
| 결정 필요 | 원문을 엄격히 만족하려면 EFK/ELK를 추가해야 함. Loki는 보조 또는 현재 구현 증거로만 사용 |
| 작업 | Elasticsearch vs Splunk ADR 작성 |
| 작업 | 1차는 Fluentd 또는 Fluent Bit + Elasticsearch + Kibana 최소 구성 |
| 완료 기준 | `reservations-logs-*`, `payments-logs-*` 인덱스 생성 |
| 의존성 | Phase 2 |

### Task 5.2 request_id propagation

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `gitops` |
| 현재 상태 | Kong correlation-id plugin 존재. 서비스 logging은 일부 구현 |
| 작업 | `X-Request-Id` 또는 Kong correlation id를 모든 서비스 로그에 포함 |
| 작업 | Kafka event payload에도 `correlationId` 포함 |
| 완료 기준 | 하나의 예매 요청을 gateway log, service log, event log에서 같은 id로 추적 |
| 의존성 | Task 2.5, Task 1.2 |

### Task 5.3 Logstash masking/error 분류

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | 결제 로그 민감정보 masking |
| 작업 | status/error_code 기반 error level 자동 분류 |
| 완료 기준 | Kibana에서 masking된 결제 로그 확인 |
| 의존성 | Task 5.1 |

## Phase 6. 대시보드/알림/SLA

### Task 6.1 Grafana dashboard 구성

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 현재 상태 | `local-kubernetes-overview.json`만 있음 |
| 작업 | 운영 dashboard: 예매 처리량, 결제 성공률, 티켓 발행량, P95/P99, 에러율 |
| 작업 | 인프라 dashboard: CPU, memory, network, restart, HPA replicas |
| 작업 | 성능 dashboard: k6 결과와 Before/After |
| 완료 기준 | dashboard JSON이 repo에 저장되고 Grafana sidecar로 반영 |
| 의존성 | Phase 4 |

### Task 6.2 Alertmanager Slack 연동

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | `#ops-alert` 기본 알림 |
| 작업 | warning/critical severity route |
| 작업 | 2차에서 critical Slack + email, 업무 시간 외 escalation |
| 완료 기준 | 테스트 알림 캡처 |
| 의존성 | Task 4.4 |

### Task 6.3 SLA/운영 가이드

| 항목 | 내용 |
| --- | --- |
| repo | `workspace` |
| 작업 | 월간 가용성 99.9% 계산식 |
| 작업 | alert 발생 시 dashboard/log query/runbook 연결 |
| 완료 기준 | SLA 산출 예시와 운영 가이드 문서 |
| 의존성 | Phase 4, Phase 5 |

## Phase 7. Kong + Istio 통합

### Task 7.1 Istio/Linkerd ADR

| 항목 | 내용 |
| --- | --- |
| repo | `workspace` |
| 기준 | Kong/Istio 역할 분리는 ADR-0003에 이미 기록 |
| 작업 | Istio vs Linkerd 비교 ADR 작성 |
| 완료 기준 | 기능, 리소스 사용량, 학습 난이도, 선택 근거 |
| 의존성 | Phase 2 |

### Task 7.2 Istio 설치와 sidecar injection

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | Istio base/istiod 설치 manifest 또는 Helm values 추가 |
| 작업 | `ticketing-*` namespace에 injection label 적용 |
| 작업 | Kong -> service 경로가 mTLS 정책에 막히지 않는지 확인 |
| 완료 기준 | sidecar 주입된 Pod에서 기존 Kong API 호출 성공 |
| 의존성 | Task 2.5 |

### Task 7.3 Canary routing

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service` |
| 대상 | `reservation-service` |
| 작업 | v1/v2 image tag 또는 deployment subset 준비 |
| 작업 | DestinationRule subset `v1`, `v2` |
| 작업 | VirtualService weight 20% -> 50% -> 100% |
| 완료 기준 | 요청 분포가 weight와 비슷하게 관측, rollback 100% v1 가능 |
| 의존성 | Task 7.2 |

### Task 7.4 Circuit Breaker/Retry/Timeout

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service` |
| 대상 | `reservation-service -> payment-service` 호출 또는 테스트용 내부 호출 |
| 현재 이슈 | 현재 payment는 외부 API 중심이라 명확한 내부 HTTP 호출 시나리오가 약함 |
| 작업 | 내부 호출이 필요한 경로를 만들거나 장애 시나리오 전용 호출을 정의 |
| 작업 | DestinationRule connectionPool, outlierDetection |
| 작업 | VirtualService retry/timeout |
| 완료 기준 | payment 5xx/지연 주입 시 retry/circuit breaker 결과 캡처 |
| 의존성 | Task 7.3 |

### Task 7.5 Kiali/mTLS/Tracing

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | Kiali 설치 |
| 작업 | PeerAuthentication PERMISSIVE -> STRICT 단계 적용 |
| 작업 | Tempo 또는 Jaeger trace 연결 |
| 완료 기준 | Kiali topology, mTLS 표시, P99 초과 trace 조회 |
| 의존성 | Task 7.2 |

## Phase 8. 성능/HPA 검증

### Task 8.1 k6 baseline

| 항목 | 내용 |
| --- | --- |
| repo | `service` |
| 대상 파일 | `service/tests/performance/**` |
| 작업 | login, concert 조회, reservation 생성, payment 승인, ticket 조회 시나리오 |
| 작업 | 동일 좌석 경쟁 시나리오 |
| 완료 기준 | P99, 최대 처리량, 에러율 baseline 기록 |
| 의존성 | Phase 1, Phase 2 |

### Task 8.2 HPA 값 요구사항 정렬

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 현재 상태 | `aws-dev` min 2 max 5 CPU 60, `aws-prod` max 8 CPU 60 |
| 요구사항 | CPU 70%, min 2, max 10 |
| 작업 | 요구사항 충족용 scenario values 또는 env values에서 70/2/10 적용 |
| 완료 기준 | HPA manifest가 요구사항 수치와 일치 |
| 의존성 | Task 4.2 |

### Task 8.3 HPA scale-out Before/After

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service`, `workspace` |
| 작업 | HPA off baseline, HPA on 결과를 같은 k6 script로 비교 |
| 작업 | P99, throughput, error rate 표 작성 |
| 완료 기준 | Before/After 표와 Grafana 캡처 |
| 의존성 | Task 8.1, Task 8.2 |

### Task 8.4 KEDA/Scheduled Scaling

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | Kafka consumer lag 기반 ScaledObject |
| 작업 | 티켓 오픈 전 replica 사전 확장 Scheduled Scaling |
| 완료 기준 | lag 증가 또는 예약 시간 도래 시 replica 변화 |
| 의존성 | Task 8.3 |

## Phase 9. 보안/접근 제어

### Task 9.1 RBAC/ServiceAccount 최소 권한

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 현재 상태 | Helm chart에 ServiceAccount, Role, RoleBinding 템플릿 존재 |
| 작업 | 개발자/운영자/SRE 역할 matrix 문서화 |
| 작업 | 서비스별 ServiceAccount 사용 |
| 작업 | ClusterRole 대신 Role + RoleBinding 원칙 적용 |
| 완료 기준 | `kubectl auth can-i` 결과 캡처 |
| 의존성 | Phase 2 |

### Task 9.2 NetworkPolicy 통신 matrix

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 현재 상태 | 서비스 ingress는 Kong namespace만 허용. DB/Kafka와 서비스 간 정책은 추가 확인 필요 |
| 작업 | Kong -> API, service -> DB, service -> Kafka, monitoring -> metrics 허용 |
| 작업 | 외부 직접 접근 차단 |
| 작업 | Istio sidecar/mTLS와 충돌하지 않는지 검증 |
| 완료 기준 | 허용/차단 시나리오 테스트 |
| 의존성 | Phase 7 |

### Task 9.3 Trivy config/SonarQube/OPA

| 항목 | 내용 |
| --- | --- |
| repo | `service`, `gitops` |
| 작업 | `trivy config`로 privileged/root/hostPath 차단 |
| 작업 | SonarQube coverage 80%, critical issue gate |
| 작업 | OPA Gatekeeper로 `runAsNonRoot`, `readOnlyRootFilesystem` 강제 |
| 완료 기준 | 정책 위반 manifest 배포 거부 증거 |
| 의존성 | Phase 3, Phase 9.1 |

### Task 9.4 Runtime/DAST

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `service` |
| 작업 | Falco shell 실행 탐지 Slack |
| 작업 | OWASP ZAP staging DAST, SARIF 업로드 |
| 완료 기준 | 보안 리포트와 Slack `#security-report` |
| 의존성 | Task 9.3 |

## Phase 10. 심화 로깅/추적/운영 자동화

### Task 10.1 Fluentd vs Fluent Bit 비교

| 항목 | 내용 |
| --- | --- |
| repo | `gitops`, `workspace` |
| 작업 | Fluentd DaemonSet 배포 후 memory 측정 |
| 작업 | Fluent Bit DaemonSet 전환 후 memory 측정 |
| 작업 | 절감률 문서화 |
| 완료 기준 | Before/After 표 |
| 의존성 | Phase 5 |

### Task 10.2 Elasticsearch ILM/shard/replica

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | ILM 7일 hot -> 30일 warm -> 90일 삭제 |
| 작업 | primary shard 5, replica 1 |
| 작업 | 노드 1개 다운 시 조회 검증 |
| 완료 기준 | Kibana/Elasticsearch API 결과 캡처 |
| 의존성 | Phase 5 |

### Task 10.3 Anomaly/log volume alert

| 항목 | 내용 |
| --- | --- |
| repo | `gitops` |
| 작업 | 에러율 전 시간 대비 200% 증가 alert |
| 작업 | 로그 볼륨 5분 평균 대비 3배 초과 alert |
| 완료 기준 | 테스트 alert |
| 의존성 | Phase 4, Phase 5 |

## Phase 11. 장애 대응/최종 보고

### Task 11.1 장애 시나리오 실행

| 시나리오 | 기대 증거 |
| --- | --- |
| `reservation-service` Pod 강제 종료 | Istio retry, Kubernetes restart, alert |
| `payment-service` 5xx 증가 | error alert, circuit breaker, graceful degradation |
| payment 200ms 지연 | P99 상승, timeout/retry |
| Kafka consumer lag 증가 | lag alert, KEDA scale-out |
| canary v2 error 증가 | weight rollback, MTTR 측정 |

### Task 11.2 Runbook 작성

| 항목 | 내용 |
| --- | --- |
| repo | `workspace` |
| 작업 | 탐지 -> 알림 -> 분석 -> 조치 -> 회고 |
| 작업 | alert 이름, Grafana panel, Kibana query, kubectl/istioctl 명령 연결 |
| 완료 기준 | 장애 시나리오 2개 이상에 대해 재발 방지책 포함 |
| 의존성 | Phase 6, Phase 7, Phase 10 |

### Task 11.3 SLA/MTTR 운영 보고서

| 항목 | 내용 |
| --- | --- |
| repo | `workspace` |
| 작업 | SLA 99.9% 준수 여부 산출 |
| 작업 | MTTR Before/After 정량화 |
| 작업 | 성능/보안/장애/운영 개선 결과 통합 |
| 완료 기준 | 최종 발표용 운영 보고서와 인수인계 문서 |
| 의존성 | 모든 Phase |

## 바로 시작해야 하는 작업 순서

아래 순서가 현재 코드 상태에서 가장 실용적이다.

1. `service/Taskfile.yml`을 티켓팅 서비스 기준으로 수정한다.
2. `payment-approved` event payload와 `ticket-service` consumer schema를 맞춘다.
3. `payment-service` outbox를 Kafka로 발행하는 worker를 만든다.
4. 티켓팅 Docker Compose E2E와 Newman collection을 만든다.
5. `infra/terraform/variables.tf`의 ECR repository 목록을 티켓팅 기준으로 바꾼다.
6. `gitops`에서 `task validate`와 `task dev:platform`을 먼저 통과시킨다.
7. `task dev:service SERVICE=auth`, `concert`, `reservation`, `payment`, `ticket`, `notification` 순서로 개별 배포한다.
8. Kong route/JWT/Rate Limit smoke 검증을 한다.
9. Prometheus ServiceMonitor와 dashboard를 켠다.
10. Istio canary는 `reservation-service` 하나로 시작한다.

## 담당 역할 추천

사용자가 기존에 Kong, API 설계, JWT 인증, LoadBalancer, HPA를 맡았다면 이번에는 다음 묶음이 가장 적합하다.

| 역할 | 포함 Task |
| --- | --- |
| Platform Traffic & Delivery Reliability | Phase 2, 3, 7, 8 중심 |
| Gateway 책임 | Kong route, JWT, Rate Limit, request id |
| Mesh 책임 | Istio sidecar, canary, retry, circuit breaker, rollback |
| 배포 책임 | GitOps values, HPA/PDB/ServiceMonitor, image tag |
| 증거 책임 | k6/HPA 결과, Grafana/Kiali 캡처, rollback runbook |

다만 Phase 1의 이벤트 계약이 완성되지 않으면 이후 phase가 모두 흔들리므로, 처음 1~2일은 서비스 담당자와 함께 `payment-approved -> ticket-issued -> notification` 흐름을 먼저 고정해야 한다.
