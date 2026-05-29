---
id: ticketing-domain-based-task-breakdown
title: 도메인별 요구사항 작업 분해와 의존 관계
type: personal-plan
status: draft
tags: [ticketing, requirements, domain-breakdown, dependency]
created: 2026-05-29
updated: 2026-05-29
---

# 도메인별 요구사항 작업 분해와 의존 관계

## 문서 목적

이 문서는 `workspace/docs/project_docs/00-GOAL.md`에 정리된 요구사항을 구현 도메인별로 다시 나누고, 각 도메인 사이의 선행 의존 관계를 정리한다.

이번 프로젝트는 공연 티켓 예매 서비스를 기준으로 한다. 기존에 진행했던 MSA 기본 프로젝트를 기반으로, 다음 영역을 순차적으로 확장하는 방식이 적합하다.

- MSA 서비스 구조
- Kubernetes 배포 기반
- CI/CD와 이미지 보안
- 모니터링
- 로깅
- 대시보드와 알림
- 서비스 메시와 트래픽 관리
- 성능 최적화와 오토스케일링
- DevSecOps와 접근 제어
- 장애 대응과 운영 보고

## 충족 범위 원칙

이 문서는 원문 요구사항 전체를 최종 충족 대상으로 둔다. 단, 여러 프로젝트에 반복 등장하는 같은 성격의 요구사항은 한 번 구현하고 여러 요구사항의 증거로 재사용한다.

예를 들어 다음 항목은 중복 구현하지 않는다.

| 중복 요구사항 | 통합 구현 기준 |
| --- | --- |
| Prometheus + Grafana 기본 관측성 | `kube-prometheus-stack`, ServiceMonitor, Grafana 운영/인프라/성능 대시보드로 통합 |
| API Gateway, JWT, Rate Limit | Kong Gateway로 통합하고, 서비스별 path routing과 JWT/Rate Limit plugin으로 증명 |
| Canary 배포 | Istio `VirtualService`/`DestinationRule` 기반 Canary로 통합하고, ArgoCD Rollouts는 선택 확장 증거로 추가 |
| Circuit Breaker | Istio `DestinationRule` connectionPool/outlierDetection을 기본으로 하고, 서비스 코드 Fallback이 필요한 곳만 Resilience4j 성격의 애플리케이션 fallback으로 보완 |
| NetworkPolicy | Kubernetes NetworkPolicy 하나의 통신 matrix로 MSA/Service Mesh/DevSecOps 보안 요구사항을 함께 충족 |
| SLA, MTTR, 장애 Runbook | Alertmanager, Grafana, Kibana, 장애 주입 결과를 하나의 운영 보고서로 통합 |
| k6 성능 테스트와 HPA 검증 | 같은 k6 시나리오를 baseline, HPA, mesh/canary, 튜닝 전후 비교에 재사용 |

따라서 "선택" 항목도 버리는 것이 아니라 2차 범위로 넣는다. 다만 원문 서비스 예시는 공연 티켓 도메인에 맞게 치환한다.

| 원문 예시 | 공연 티켓 적용 |
| --- | --- |
| 상품/주문/결제/배송 | 공연/예매/결제/티켓/알림 |
| 계좌/이체/대출/알림 | 인증/예매/결제/티켓/알림 |
| 환자기록/예약/처방/알림 | 사용자/예매/공연/티켓/알림 |
| 콘텐츠/스트리밍/사용자/채팅 | 공연/좌석/사용자/예매 이벤트 |

## 1차/2차 충족 계획

작업은 1차와 2차로 나누지만, 2차는 "나중에 할 수도 있는 항목"이 아니라 최종 충족 대상이다. 1차는 모든 요구사항의 기반과 필수 시연 경로를 만들고, 2차는 심화/선택/자동화/보고 증거를 완성한다.

| 차수 | 목표 | 포함 범위 | 완료 기준 |
| --- | --- | --- | --- |
| 1차 | 기본 프로젝트 필수 요구사항과 심화 기반 확보 | MSA 핵심 서비스, Kubernetes 배포, Kong Gateway, GitHub Actions, Trivy image scan, Prometheus/Grafana, Fluentd 또는 Fluent Bit 로그 수집, Alertmanager Slack, HPA, 기본 NetworkPolicy, 기본 Istio Canary | 정상 예매 E2E, 배포 자동화, 기본 관측성, 기본 장애 알림, Canary/rollback 최소 증거 |
| 2차 | 심화 프로젝트와 선택 요구사항까지 최종 충족 | Fluent Bit 전환 비교, Elasticsearch/Splunk ADR, Logstash masking, ILM, severity routing, email/escalation, Kiali, mTLS STRICT, Jaeger/Tempo, outlierDetection, KEDA, Scheduled Scaling, SonarQube, Trivy config, OPA, Falco, ZAP, Chaos Mesh, weekly SLA report, 운영 보고서 | 원문 요구사항 충족표 전체 완료, Before/After 수치, Runbook, 운영 보고서, 발표 증거 캡처 |

## 원문 요구사항 충족 매핑

아래 표는 원문 요구사항이 이 문서의 어느 도메인과 차수에서 충족되는지 정리한 것이다. 상태가 `통합`인 항목은 다른 요구사항과 중복되므로 별도 구현 대신 같은 산출물로 증명한다.

### A. 클라우드 네이티브 모니터링 및 로깅

| 원문 요구사항 | 차수 | 담당 도메인 | 공연 티켓 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| 수집 지표 정의 문서 | 1차 | 모니터링 | CPU, memory, 요청량, 에러율, 응답시간, 예매/결제/티켓 지표 기준서 | 포함 |
| `kube-prometheus-stack` 배포 | 1차 | 모니터링 | `monitoring` namespace Prometheus/Grafana/Alertmanager | 포함 |
| ServiceMonitor 등록 | 1차 | 모니터링 | 6개 서비스 `/metrics` scrape | 포함 |
| 장애 탐지 기준과 대응 프로세스 | 1차 | 장애 대응 | 에러율 5%, P99 2초, CrashLoopBackOff, 탐지-알림-분석-조치-회고 Runbook | 포함 |
| recording rule 5분 평균 에러율 | 2차 | 모니터링 | `service:error_rate_5m` 계열 recording rule | 포함 |
| Namespace ResourceQuota | 1차 | Kubernetes 배포 기반 | application/monitoring/logging namespace quota | 포함 |
| Fluentd DaemonSet 로그 수집 | 1차 | 로깅 | 전 Pod 로그 수집, Elasticsearch 적재 | 포함 |
| 서비스별 로그 인덱스 | 1차 | 로깅 | `reservations-logs-*`, `payments-logs-*`, `tickets-logs-*` | 포함 |
| Kibana 에러 로그 필터/request_id 추적 | 1차 | 로깅 | 서비스별 error saved search, request_id trace view | 포함 |
| Grafana 운영 통합 대시보드 | 1차 | 대시보드와 알림 | 예매 처리량, 결제 성공률, 서비스별 응답시간, threshold | 포함 |
| 결제 로그 별도 인덱스 | 2차 | 로깅 | `payments-logs-*` 보안 감사 인덱스와 masking | 포함 |
| Kibana ILM | 2차 | 로깅 | 7일 hot, 30일 warm, 90일 삭제 | 포함 |
| Alertmanager 알림 규칙 | 1차 | 대시보드와 알림 | PrometheusRule error/latency/CrashLoop | 포함 |
| Slack `#ops-alert` 테스트 알림 | 1차 | 대시보드와 알림 | 테스트 알림 캡처 | 포함 |
| SLA 99.9% 운영 가이드 | 1차 | 장애 대응 | 월간 가용성 계산식과 운영 가이드 | 포함 |
| warning/critical 라우팅 | 2차 | 대시보드와 알림 | Slack, Slack+email severity route | 포함 |
| 업무 시간 외 escalation | 2차 | 장애 대응 | critical on-call/escalation 정책 | 포함 |

### B. 중앙 집중식 로깅 및 실시간 모니터링

| 원문 요구사항 | 차수 | 담당 도메인 | 공연 티켓 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| Elasticsearch vs Splunk ADR | 2차 | 로깅 | 비용, 검색 성능, 확장성 비교 ADR | 포함 |
| Fluentd -> Fluent Bit 전환 비교 | 2차 | 로깅 | DaemonSet memory Before/After | 포함 |
| Logstash masking/error 분류 | 2차 | 로깅 | 결제 민감정보 마스킹, error level 자동 분류 | 포함 |
| Elasticsearch shard/replica 구성 | 2차 | 로깅 | Primary shard 5, replica 1, 노드 장애 조회 검증 | 포함 |
| Sharding/Replication 설계 문서 | 2차 | 로깅 | 대규모 로그 처리 설계 문서 | 포함 |
| 운영/인프라 대시보드 분리 | 1차 | 대시보드와 알림 | Grafana/Kibana 운영 관점, 인프라 관점 분리 | 포함 |
| severity 기반 이중 알림 | 2차 | 대시보드와 알림 | warning/critical route | 통합 |
| 5분/24시간 trend 패널 | 1차 | 대시보드와 알림 | 에러율/응답시간 단기/장기 패널 | 포함 |
| anomaly 알림 | 2차 | 대시보드와 알림 | 에러율 200% 증가 rule | 포함 |
| 로그 볼륨 급증 알림 | 2차 | 로깅 | 5분 평균 대비 3배 초과 alert | 포함 |
| 반복 장애 패턴 2개 Runbook | 2차 | 장애 대응 | 메모리 급증, 배포 직후 에러율 spike Runbook | 포함 |
| Kibana SLA view | 2차 | 장애 대응 | 로그 기반 SLA 준수 view | 포함 |
| 운영 보고서 | 2차 | 장애 대응 | 장애 패턴과 개선 방안 보고서 | 포함 |
| EFK 노드 다운 복구 자동화 | 2차 | 장애 대응 | 노드 1개 다운 복구 스크립트 | 포함 |
| Kibana Canvas 주간 SLA Slack 발송 | 2차 | 장애 대응 | 주간 SLA 리포트 자동 발송 | 포함 |

### C. Kubernetes 배포 자동화와 서비스 메시

| 원문 요구사항 | 차수 | 담당 도메인 | 공연 티켓 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| Jenkins vs GitHub Actions ADR | 1차 | CI/CD와 이미지 보안 | CI 도구 선택 ADR | 포함 |
| GitHub Actions test/build/push/deploy | 1차 | CI/CD와 이미지 보안 | 서비스별 workflow, path filter | 포함 |
| Slack `#deploy-status` | 1차 | CI/CD와 이미지 보안 | 배포 성공/실패 알림 | 포함 |
| 병렬 빌드 시간 개선 | 2차 | CI/CD와 이미지 보안 | matrix build 시간 Before/After | 포함 |
| prod 승인자 확인 | 2차 | CI/CD와 이미지 보안 | GitHub Environment Protection | 포함 |
| multi-stage Dockerfile/non-root | 1차 | CI/CD와 이미지 보안 | `appuser` UID 1001 | 포함 |
| KT클라우드 Container Registry/git-sha tag | 1차 | CI/CD와 이미지 보안 | 서비스별 repo, git-sha tag 정책 | 포함 |
| Trivy HIGH/CRITICAL 차단 | 1차 | CI/CD와 이미지 보안 | push 전 image scan gate | 포함 |
| Trivy PR comment | 2차 | CI/CD와 이미지 보안 | 취약점 결과 PR comment | 포함 |
| untagged image lifecycle | 2차 | CI/CD와 이미지 보안 | registry lifecycle policy | 포함 |
| Dependabot base image PR | 2차 | CI/CD와 이미지 보안 | 주간 base image update PR | 포함 |
| Deployment + HPA | 1차 | Kubernetes 배포 기반 | CPU 70%, min 2, max 10 | 포함 |
| readiness/liveness probe | 1차 | Kubernetes 배포 기반 | `/readyz`, `/healthz` | 포함 |
| Helm + Rolling Update | 1차 | Kubernetes 배포 기반 | Helm chart, rolling strategy | 포함 |
| ArgoCD GitOps | 1차 | Kubernetes 배포 기반 | Git 변경 -> 자동 Sync | 포함 |
| dev/prod values | 1차 | Kubernetes 배포 기반 | 환경별 values/Application | 포함 |
| Canary 배포 | 1차 | 서비스 메시와 트래픽 관리 | Istio canary로 통합 | 통합 |
| Istio vs Linkerd ADR | 1차 | 서비스 메시와 트래픽 관리 | 서비스 메시 선택 ADR | 포함 |
| Istio 설치/sidecar injection | 1차 | 서비스 메시와 트래픽 관리 | namespace auto injection | 포함 |
| VirtualService/DestinationRule Canary | 1차 | 서비스 메시와 트래픽 관리 | 20% -> 50% -> 100% | 포함 |
| mTLS STRICT/Kiali 확인 | 2차 | 서비스 메시와 트래픽 관리 | PeerAuthentication STRICT, Kiali 확인 | 포함 |
| 트래픽 라우팅/Observability 강화 | 2차 | 서비스 메시와 트래픽 관리 | blue-green/canary, telemetry | 포함 |
| Kiali topology | 1차 | 서비스 메시와 트래픽 관리 | service graph, error rate | 포함 |
| Envoy CPU/memory 수집 | 1차 | 서비스 메시와 트래픽 관리 | Prometheus/Grafana Envoy panel | 포함 |
| DestinationRule Circuit Breaker | 1차 | 서비스 메시와 트래픽 관리 | connectionPool | 포함 |
| Jaeger tracing | 2차 | 서비스 메시와 트래픽 관리 | P99 초과 trace 조회 | 포함 |
| outlierDetection 5xx ejection | 2차 | 서비스 메시와 트래픽 관리 | 5회 5xx, 30초 ejection 검증 | 포함 |
| 병목 분석과 개선 | 2차 | 서비스 메시와 트래픽 관리 | trace 기반 병목 분석 | 포함 |
| NetworkPolicy 외부 직접 접근 차단 | 1차 | DevSecOps와 접근 제어 | namespace 내부 허용, 외부 차단 | 포함 |
| Pod 강제 종료/Istio Retry | 1차 | 장애 대응 | 장애 주입 결과 | 포함 |
| rollback Runbook | 1차 | 장애 대응 | ArgoCD revision 또는 VirtualService 즉시 전환 | 포함 |
| 지연/다운 장애 2개 이상 | 2차 | 장애 대응 | fault injection, service down | 포함 |
| 5분 이내 복구 자동화 | 2차 | 장애 대응 | rollback script, MTTR 측정 | 포함 |
| NetworkPolicy 세분화 테스트 | 2차 | DevSecOps와 접근 제어 | 허용/차단 시나리오 | 포함 |

### D. MSA 구조와 보안 아키텍처

| 원문 요구사항 | 차수 | 담당 도메인 | 공연 티켓 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| 이벤트 스토밍 서비스 경계 | 1차 | MSA 서비스 구조 | auth/concert/reservation/payment/ticket/notification 경계 | 포함 |
| REST vs Event 기준 | 1차 | MSA 서비스 구조 | 동기/비동기 통신 기준 문서 | 포함 |
| Database per Service | 1차 | MSA 서비스 구조 | 서비스별 DB 소유권 원칙 | 포함 |
| Kafka StatefulSet/topics | 1차 | MSA 서비스 구조 | reservation/payment/ticket event topics | 포함 |
| Event Sourcing 1개 서비스 | 2차 | MSA 서비스 구조 | payment 또는 reservation event history | 포함 |
| ClusterIP + DNS | 1차 | MSA 서비스 구조 | 내부 통신 검증 | 포함 |
| API Gateway/JWT | 1차 | MSA 서비스 구조 | Kong route/JWT | 통합 |
| 장애 격리 문서 | 1차 | 장애 대응 | 의존 서비스 down 시 부분 응답 설계 | 포함 |
| Fallback 로직 | 2차 | 장애 대응 | notification/payment 장애 graceful degradation | 포함 |
| Rate Limiting | 1차 | 서비스 메시와 트래픽 관리 | Kong Rate Limit | 통합 |
| 단위 테스트/CI | 1차 | CI/CD와 이미지 보안 | pytest/JUnit 성격의 서비스 테스트 자동 실행 | 포함 |
| Postman E2E | 1차 | MSA 서비스 구조 | 예매 -> 결제 -> 티켓 -> 알림 E2E | 포함 |
| Prometheus/Grafana observability | 1차 | 모니터링 | 서비스별 error/latency | 통합 |
| Testcontainers | 2차 | MSA 서비스 구조 | DB/Kafka 통합 테스트 | 포함 |
| Newman CI | 2차 | MSA 서비스 구조 | Postman E2E CI 자동 실행 | 포함 |
| Kafka consumer lag alert | 2차 | 모니터링 | Kafka exporter + lag alert | 포함 |
| 독립 배포 파이프라인 | 1차 | CI/CD와 이미지 보안 | 서비스별 path filter와 deploy job | 포함 |
| API Gateway + Mesh 협업 | 1차 | 서비스 메시와 트래픽 관리 | Kong edge + Istio internal mesh ADR | 포함 |
| PDB 최소 Pod 2개 | 1차 | Kubernetes 배포 기반 | 서비스별 PDB | 포함 |
| KEDA Kafka lag scaling | 2차 | 성능 최적화와 오토스케일링 | ScaledObject | 포함 |
| ArgoCD Rollouts | 2차 | Kubernetes 배포 기반 | 주요 서비스 canary 자동 rollback | 포함 |
| SonarQube quality gate | 2차 | DevSecOps와 접근 제어 | coverage 80%, critical issue 차단 | 포함 |
| Trivy config | 1차 | DevSecOps와 접근 제어 | manifest scan gate | 포함 |
| Slack `#security-report` | 2차 | DevSecOps와 접근 제어 | 보안 스캔 결과 알림 | 포함 |
| OWASP ZAP DAST | 2차 | DevSecOps와 접근 제어 | SARIF 업로드 | 포함 |
| OPA Gatekeeper | 2차 | DevSecOps와 접근 제어 | runAsNonRoot/readOnlyRootFilesystem 강제 | 포함 |
| RBAC 역할 분리 | 1차 | DevSecOps와 접근 제어 | 개발자/운영자/SRE Role | 포함 |
| ServiceAccount 최소 권한 | 1차 | DevSecOps와 접근 제어 | Role + RoleBinding | 포함 |
| NetworkPolicy 접근 제어 | 1차 | DevSecOps와 접근 제어 | 의도하지 않은 통신 차단 | 통합 |
| Incident Response 자동화 | 2차 | 장애 대응 | 탐지-격리-분석-복구-회고 스크립트 | 포함 |
| Falco runtime detection | 2차 | DevSecOps와 접근 제어 | shell 실행 탐지 Slack | 포함 |

### E. 성능 최적화, 트래픽 관리, 장애 대응

| 원문 요구사항 | 차수 | 담당 도메인 | 공연 티켓 산출물 | 상태 |
| --- | --- | --- | --- | --- |
| k6 baseline | 1차 | 성능 최적화와 오토스케일링 | P99, 최대 처리량, 에러율 | 포함 |
| `tests/performance/` 관리 | 1차 | 성능 최적화와 오토스케일링 | k6 script repo 관리 | 포함 |
| 병목 분석 문서 | 1차 | 성능 최적화와 오토스케일링 | CPU/memory/network I/O 분석 | 포함 |
| HPA scale-out 검증 | 1차 | 성능 최적화와 오토스케일링 | 동일 k6 시나리오 비교 | 포함 |
| Scheduled Scaling | 2차 | 성능 최적화와 오토스케일링 | 티켓 오픈 전 replica 사전 확장 | 포함 |
| KEDA queue scaling | 2차 | 성능 최적화와 오토스케일링 | Kafka lag 기반 scaling | 통합 |
| JMX Exporter | 2차 | 모니터링 | JVM 서비스가 있을 경우 GC metric, Python 서비스면 런타임 metric으로 대체 근거 문서화 | 조건부 포함 |
| 서비스 custom metric | 1차 | 모니터링 | 동시 예매, 결제 성공률, 티켓 발행량 | 포함 |
| 1초 갱신 Grafana dashboard | 2차 | 대시보드와 알림 | 실시간 성능 dashboard | 포함 |
| 성능 임계치 Slack alert | 1차 | 대시보드와 알림 | P99/error alert | 포함 |
| 비즈니스 KPI dashboard | 2차 | 대시보드와 알림 | 예매 전환율, 결제 성공률, 티켓 발행량 | 포함 |
| Grafana Annotation | 2차 | 대시보드와 알림 | 배포 이벤트 annotation | 포함 |
| Before/After 표 | 1차 | 성능 최적화와 오토스케일링 | 튜닝 전후 수치 | 포함 |
| 최적화 결과 문서/점진 적용 가이드 | 1차 | 성능 최적화와 오토스케일링 | 운영 적용 가이드 | 포함 |
| 2차 튜닝 사이클 | 2차 | 성능 최적화와 오토스케일링 | 미달성 항목 재검증 | 포함 |
| 월 1회 성능 리뷰 | 2차 | 성능 최적화와 오토스케일링 | 정기 리뷰 프로세스 | 포함 |
| 30일 Prometheus 트래픽 분석 | 2차 | 서비스 메시와 트래픽 관리 | 실제 30일 데이터가 없으면 부하 테스트/시뮬레이션 데이터 기준 명시 | 조건부 포함 |
| Scheduled vs Event-driven 전략 | 2차 | 성능 최적화와 오토스케일링 | 트래픽 패턴별 scaling 전략 | 포함 |
| Istio weighted routing | 1차 | 서비스 메시와 트래픽 관리 | canary routing | 통합 |
| API Gateway 또는 Istio Rate Limit | 1차 | 서비스 메시와 트래픽 관리 | Kong Rate Limit | 통합 |
| Locality-aware LB | 2차 | 서비스 메시와 트래픽 관리 | 가능 환경에서 latency 개선 측정 | 포함 |
| 트래픽 정책 정기 리뷰 | 2차 | 서비스 메시와 트래픽 관리 | VirtualService/Rate Limit 리뷰 절차 | 포함 |
| Graceful Degradation | 2차 | 장애 대응 | 의존 서비스 장애 시 fallback | 포함 |
| Slack `#incident` 채널 자동 생성 | 2차 | 장애 대응 | Alertmanager webhook 연동 | 포함 |
| 장애 Runbook 검증 | 1차 | 장애 대응 | 실제 장애 시나리오 실행 | 포함 |
| Alertmanager webhook autoscale | 2차 | 장애 대응 | HPA min replica 자동 확장 | 포함 |
| Chaos Mesh | 2차 | 장애 대응 | Pod kill/network delay 정기 실행 | 포함 |
| 안정성 Before/After | 2차 | 장애 대응 | error rate, latency stddev, SLA 준수율 비교 | 포함 |
| MTTR 개선 수치 | 2차 | 장애 대응 | 복구 시간 Before/After | 포함 |
| 팀 발표 자료 | 2차 | 장애 대응 | 효과 분석 발표 자료 | 포함 |
| 인수인계 문서 | 2차 | 장애 대응 | 최적 설정값과 근거 문서 | 포함 |

## 전체 진행 순서

아래 순서대로 진행하면 의존 관계가 가장 자연스럽다.

| 순서 | 도메인 | 이유 |
| --- | --- | --- |
| 1 | MSA 서비스 구조 | 실제 metric, log, trace, 장애 시나리오를 만들 서비스가 먼저 필요하다. |
| 2 | Kubernetes 배포 기반 | 모니터링, 로깅, 서비스 메시, HPA, NetworkPolicy는 Kubernetes 위에서 검증된다. |
| 3 | CI/CD와 이미지 보안 | 서비스와 배포 구조가 잡힌 뒤 자동 빌드, 배포, 스캔을 붙인다. |
| 4 | 모니터링 | `/metrics`, ServiceMonitor, PrometheusRule, HPA 판단 근거가 된다. |
| 5 | 로깅 | 장애 분석, request_id 추적, 운영 보고서의 근거가 된다. |
| 6 | 대시보드와 알림 | 모니터링/로깅 데이터가 있어야 Grafana, Kibana, Alertmanager를 구성할 수 있다. |
| 7 | 서비스 메시와 트래픽 관리 | Kubernetes 배포와 기본 관측성이 있어야 Canary, Retry, Circuit Breaker를 검증할 수 있다. |
| 8 | 성능 최적화와 오토스케일링 | 모니터링과 배포가 안정된 뒤 k6, HPA, Before/After 측정을 수행한다. |
| 9 | DevSecOps와 접근 제어 | 배포 파이프라인과 Kubernetes 리소스가 정리된 뒤 정책을 강제한다. |
| 10 | 장애 대응과 운영 보고 | 앞 단계의 지표, 로그, 알림, 트래픽 정책 결과를 종합해 Runbook과 보고서를 만든다. |

## 도메인 1: MSA 서비스 구조

### 목표

공연 티켓 예매 서비스를 독립 배포 가능한 마이크로서비스로 나누고, 서비스 간 통신과 데이터 소유권을 명확히 한다.

### 포함 요구사항

- 도메인별 서비스 경계를 이벤트 스토밍으로 도출한다.
- 서비스 간 통신 구조를 REST API와 비동기 이벤트로 나누어 설계한다.
- Database per Service 원칙을 적용한다.
- Kubernetes ClusterIP + DNS로 내부 서비스 디스커버리를 검증한다.
- API Gateway 또는 Ingress로 경로 라우팅과 JWT 인증 필터를 구성한다.
- JUnit 단위 테스트와 Postman E2E 테스트를 작성한다.

### 공연 티켓 도메인 적용

| 기존 예시 | 공연 티켓 서비스 적용 |
| --- | --- |
| 환자기록 서비스 | `user-service` 또는 `auth-service` |
| 예약 서비스 | `reservation-service` |
| 처방 서비스 | `concert-service` 또는 `ticket-service` |
| 알림 서비스 | `notification-service` |
| 환자 예약 → 예약 확정 이벤트 → 알림 발송 | 좌석 선택 → 예매 생성 → 결제 승인 → 티켓 발행 → 알림 발송 |

### 주요 작업

- `auth-service`, `concert-service`, `reservation-service`, `payment-service`, `ticket-service`, `notification-service` 경계를 확정한다.
- 좌석 중복 예매 방지 책임을 `reservation-service`에 둔다.
- 결제 승인 이후 티켓 발행과 알림 발송은 Kafka 이벤트로 분리한다.
- 각 서비스별 DB를 분리하고 직접 DB 공유를 금지한다.
- 서비스별 `/healthz`, `/readyz`, `/metrics` 엔드포인트를 제공한다.
- Postman Collection으로 핵심 예매 E2E를 만든다.

### 선행 의존

- 없음. 가장 먼저 해야 하는 기반 작업이다.

### 후속 의존

- 모니터링은 `/metrics`가 필요하다.
- Kubernetes Probe는 `/healthz`, `/readyz`가 필요하다.
- 로깅 추적은 서비스 간 `request_id` 전달이 필요하다.
- 서비스 메시 장애 검증은 실제 서비스 간 호출이 필요하다.
- 성능 테스트는 정상 예매 E2E가 먼저 있어야 한다.

## 도메인 2: Kubernetes 배포 기반

### 목표

서비스를 Kubernetes에 배포하고, 운영 검증이 가능한 기본 실행 환경을 만든다.

### 포함 요구사항

- Deployment, Service, HPA를 구성한다.
- Readiness Probe와 Liveness Probe를 설정한다.
- Helm Chart로 배포 설정을 관리한다.
- Rolling Update 배포 전략을 적용한다.
- Namespace, ResourceQuota, PDB를 구성한다.
- dev/prod values 분리와 ArgoCD Application 구성을 확장한다.

### 주요 작업

- 서비스별 Deployment, Service, ConfigMap, Secret 참조를 만든다.
- HPA 기준은 CPU 70%, 최소 2개, 최대 10개 replica로 둔다.
- Probe 경로는 `/readyz`, `/healthz`로 통일한다.
- Helm Chart를 공통 chart 또는 서비스별 chart로 정리한다.
- ArgoCD Application으로 Git 변경 → 자동 Sync 흐름을 구성한다.
- PDB로 유지보수 중 최소 Pod 수 2개를 보장한다.
- ResourceQuota로 `monitoring`, `logging`, application namespace를 격리한다.

### 선행 의존

- MSA 서비스 구조
- 서비스별 Docker image
- health/readiness endpoint

### 후속 의존

- Prometheus ServiceMonitor는 Kubernetes Service가 필요하다.
- Fluent Bit/Fluentd는 Pod 로그가 실제로 발생해야 한다.
- Istio sidecar injection은 namespace와 Deployment가 있어야 한다.
- HPA 검증은 Deployment와 metric 수집이 필요하다.
- NetworkPolicy와 RBAC은 namespace, service account 구조가 있어야 한다.

## 도메인 3: CI/CD와 이미지 보안

### 목표

서비스별 변경 사항만 빌드, 테스트, 스캔, 배포되도록 자동화하고, 취약한 이미지는 배포 전에 차단한다.

### 포함 요구사항

- Jenkins와 GitHub Actions를 비교하고 ADR을 작성한다.
- GitHub Actions로 test → Docker build → registry push → Kubernetes deploy 파이프라인을 구성한다.
- path filter로 변경된 서비스만 빌드/배포한다.
- 배포 결과를 Slack `#deploy-status`로 보낸다.
- Dockerfile을 multi-stage build와 non-root user 기준으로 작성한다.
- 이미지 태그를 `git-sha`로 관리한다.
- Trivy로 HIGH/CRITICAL CVE 발견 시 push를 차단한다.

### 주요 작업

- GitHub Actions 선택 ADR을 작성한다.
- 서비스별 workflow 또는 matrix workflow를 구성한다.
- path filter를 서비스 디렉토리 단위로 적용한다.
- Dockerfile을 build stage와 runtime stage로 분리한다.
- `appuser`, UID `1001`로 non-root 실행을 강제한다.
- Trivy image scan을 registry push 이전 단계에 둔다.
- 배포 성공/실패 Slack notification을 구성한다.
- prod 배포는 Environment Protection Rule로 승인자를 요구한다.

### 선행 의존

- MSA 서비스 구조
- 서비스별 Dockerfile
- Container Registry
- Kubernetes 배포 대상

### 후속 의존

- GitOps 배포 자동화는 image tag와 values 업데이트 흐름이 필요하다.
- Grafana Annotation은 배포 이벤트를 받아야 한다.
- DevSecOps는 Trivy/SonarQube 결과를 CI 품질 게이트로 사용한다.
- 운영 보고서에는 배포 성공률, 실패 원인, 보안 스캔 결과가 근거로 들어간다.

## 도메인 4: 모니터링

### 목표

서비스와 인프라 상태를 Prometheus로 수집하고, 장애 탐지와 성능 분석의 기본 지표를 만든다.

### 포함 요구사항

- 모니터링 대상 서비스와 수집 지표를 정의한다.
- CPU, 메모리, 요청량, 에러율, 응답시간 수집 기준 문서를 작성한다.
- `kube-prometheus-stack`을 `monitoring` namespace에 배포한다.
- 각 서비스 `/metrics`를 ServiceMonitor로 scrape 대상에 등록한다.
- 장애 기준을 정의한다. 에러율 5% 초과, P99 2초 초과, Pod CrashLoopBackOff.
- PrometheusRule로 에러율, 응답 지연, Pod CrashLoop 알림 규칙을 정의한다.
- recording rule로 서비스별 5분 평균 에러율을 사전 계산한다.

### 공연 티켓 핵심 지표

| 지표 | 의미 |
| --- | --- |
| `ticket_reservation_requests_total` | 예매 요청 수 |
| `ticket_reservation_success_total` | 예매 성공 수 |
| `ticket_reservation_conflict_total` | 좌석 중복 시도 또는 선점 실패 수 |
| `payment_success_total` | 결제 성공 수 |
| `payment_failed_total` | 결제 실패 수 |
| `ticket_issued_total` | 티켓 발행 수 |
| `http_server_requests_seconds` | 서비스별 API 응답시간 |
| `kafka_consumer_lag` | 이벤트 처리 지연 |

### 주요 작업

- metric naming convention을 정한다.
- 서비스별 `/metrics` 노출을 확인한다.
- Prometheus stack을 설치한다.
- ServiceMonitor를 서비스별로 만든다.
- PrometheusRule을 warning/critical severity로 나눈다.
- recording rule로 5분 평균 에러율을 만든다.
- CrashLoopBackOff, P99 latency, error rate alert를 검증한다.

### 선행 의존

- MSA 서비스 구조
- Kubernetes 배포 기반
- `/metrics` endpoint
- Service label/namespace 규칙

### 후속 의존

- Grafana dashboard는 Prometheus metric이 필요하다.
- Alertmanager는 PrometheusRule이 필요하다.
- HPA와 성능 분석은 CPU, memory, latency metric이 필요하다.
- Istio Envoy metric 수집도 Prometheus 기반으로 이어진다.
- SLA 산출과 운영 보고서는 metric 저장 결과가 필요하다.

## 도메인 5: 로깅

### 목표

전체 Pod 로그를 중앙에서 수집하고, 서비스별 에러 로그와 request_id 기반 요청 추적이 가능하게 한다.

### 포함 요구사항

- Fluentd를 DaemonSet으로 배포해 전체 Pod 로그를 수집한다.
- Elasticsearch에 서비스별 인덱스로 적재한다.
- Kibana에서 서비스별 에러 로그 필터와 request_id 기반 추적 뷰를 구성한다.
- Elasticsearch와 Splunk를 비교하고 ADR을 작성한다.
- Fluentd를 Fluent Bit으로 교체하고 메모리 사용량 절감 수치를 문서화한다.
- Logstash에 민감 데이터 마스킹과 에러 레벨 자동 분류 규칙을 추가한다.
- 결제 로그를 별도 인덱스로 분리한다.
- ILM 정책으로 7일 hot, 30일 warm, 90일 삭제를 자동화한다.

### 공연 티켓 로그 인덱스 예시

| 인덱스 | 대상 |
| --- | --- |
| `reservations-logs-*` | 예매 생성, 좌석 선점, 예매 취소 |
| `payments-logs-*` | 결제 승인, 결제 실패, PG mock 응답 |
| `tickets-logs-*` | 티켓 발행, QR/PDF 생성 |
| `notifications-logs-*` | 알림 발송, 알림 실패 |
| `gateway-logs-*` | 인증, 라우팅, rate limit |

### 주요 작업

- 모든 서비스 로그에 `request_id`, `user_id`, `reservation_id`를 포함한다.
- Fluentd 또는 Fluent Bit DaemonSet을 배포한다.
- Elasticsearch index naming rule을 정한다.
- Kibana index pattern과 saved search를 만든다.
- request_id로 전체 예매 요청 흐름을 추적하는 뷰를 만든다.
- 결제 로그의 민감 정보 마스킹 규칙을 적용한다.
- Fluentd와 Fluent Bit의 DaemonSet memory 사용량을 비교한다.
- Elasticsearch vs Splunk ADR을 작성한다.

### 선행 의존

- MSA 서비스 구조
- Kubernetes 배포 기반
- 서비스 로그 포맷 표준화
- request_id propagation

### 후속 의존

- 장애 원인 분석은 중앙 로그가 필요하다.
- Kibana SLA view는 로그와 timestamp가 필요하다.
- 반복 장애 패턴 식별은 누적 로그가 필요하다.
- 보안 감사 추적은 결제 로그 분리와 masking이 필요하다.
- 운영 보고서는 로그 분석 결과를 근거로 사용한다.

## 도메인 6: 대시보드와 알림

### 목표

운영자가 서비스 상태를 한 화면에서 판단하고, 임계치 초과 시 Slack으로 즉시 알림을 받게 한다.

### 포함 요구사항

- Grafana에 예매 처리량, 결제 성공률, 서비스별 응답시간 대시보드를 구성한다.
- threshold 초과 시 패널 색상이 변하도록 설정한다.
- 운영 관점과 인프라 관점으로 대시보드를 분리한다.
- 단기 5분, 장기 24시간 트렌드 패널을 추가한다.
- Alertmanager와 Slack `#ops-alert`를 연동한다.
- warning은 Slack, critical은 Slack + email 등 severity 기반 라우팅을 구성한다.
- 서비스 SLA 99.9%와 운영 가이드라인을 문서화한다.

### 대시보드 분리

| 대시보드 | 주요 패널 |
| --- | --- |
| 운영 대시보드 | 예매 성공률, 예매 처리량, 결제 성공률, 좌석 conflict 수, API P95/P99 |
| 인프라 대시보드 | Pod CPU, memory, network, restart count, HPA replica |
| 로그 대시보드 | 서비스별 error log, request_id trace, 결제 실패 로그 |
| 서비스 메시 대시보드 | mTLS 상태, Envoy CPU/memory, service graph, 5xx rate |
| 성능 대시보드 | k6 결과, 처리량, latency trend, error rate Before/After |

### 주요 작업

- Grafana datasource를 Prometheus와 Elasticsearch/Loki에 연결한다.
- 운영 대시보드와 인프라 대시보드를 분리한다.
- threshold color rule을 패널별로 설정한다.
- 5분/24시간 trend를 같은 지표에 대해 함께 보여준다.
- Alertmanager route를 severity 기준으로 나눈다.
- Slack test alert를 발송하고 캡처를 남긴다.
- SLA 99.9% 기준과 계산 방법을 문서화한다.

### 선행 의존

- 모니터링
- 로깅
- PrometheusRule
- Slack webhook 또는 app 설정

### 후속 의존

- 성능 최적화 Before/After 비교는 dashboard가 있으면 증거화가 쉽다.
- 장애 대응 Runbook은 alert 이름과 dashboard 링크를 기준으로 작성된다.
- 운영 보고서는 dashboard 캡처와 alert 이력을 사용한다.
- 서비스 메시 운영 metric도 Grafana에 합쳐진다.

## 도메인 7: 서비스 메시와 트래픽 관리

### 목표

Istio를 도입해 서비스 간 통신을 관찰하고, Canary, Retry, Circuit Breaker, mTLS, fault injection을 검증한다.

### 포함 요구사항

- Istio와 Linkerd를 비교하고 ADR을 작성한다.
- Istio를 설치하고 서비스 namespace에 sidecar auto injection을 활성화한다.
- VirtualService와 DestinationRule로 Canary 라우팅을 적용한다.
- Kiali로 서비스 topology와 에러율을 시각화한다.
- Envoy sidecar CPU/memory를 Prometheus로 수집한다.
- DestinationRule connectionPool 기반 Circuit Breaker를 구성한다.
- PeerAuthentication mTLS STRICT를 활성화한다.
- Jaeger 또는 Tempo로 분산 트레이싱을 수집한다.
- fault injection, retry, outlierDetection 장애 시나리오를 검증한다.

### 공연 티켓 적용 시나리오

| 기능 | 적용 대상 |
| --- | --- |
| Canary | `reservation-service` 신규 버전 20% → 50% → 100% |
| Retry | `reservation-service` → `payment-service` 호출 실패 |
| Circuit Breaker | `payment-service` 과부하 또는 5xx 연속 발생 |
| mTLS | ticket namespace 내부 서비스 전체 |
| Fault injection | `payment-service` 200ms 지연 또는 5xx 주입 |
| Rollback | VirtualService weight를 이전 버전 100%로 즉시 전환 |

### 주요 작업

- Istio vs Linkerd ADR을 작성한다.
- Istio control plane을 설치한다.
- application namespace에 sidecar injection label을 추가한다.
- Kiali를 배포한다.
- VirtualService와 DestinationRule을 서비스별로 작성한다.
- Canary weight 전환 절차를 문서화한다.
- Retry, timeout, connectionPool, outlierDetection 정책을 구성한다.
- mTLS STRICT 적용 여부를 Kiali에서 확인한다.
- 장애 주입 후 alert, log, dashboard 변화를 기록한다.

### 선행 의존

- Kubernetes 배포 기반
- 모니터링
- 대시보드와 알림
- 서비스 간 실제 호출 흐름

### 후속 의존

- 트래픽 관리 정책 평가가 가능해진다.
- 장애 대응 Runbook의 rollback 절차가 구체화된다.
- MTTR 측정에 retry/canary/rollback 결과가 들어간다.
- 성능 테스트에서 service mesh overhead를 비교할 수 있다.

## 도메인 8: 성능 최적화와 오토스케일링

### 목표

티켓 오픈 상황의 부하를 재현하고, 병목을 식별한 뒤 HPA/KEDA/Scheduled Scaling 등으로 개선 효과를 측정한다.

### 포함 요구사항

- k6로 P99 응답시간, 최대 처리량, 에러율을 측정한다.
- k6 script를 `tests/performance/`에 코드로 관리한다.
- Prometheus metric으로 CPU, memory, network I/O 병목을 식별한다.
- HPA 설정 후 동일 k6 시나리오로 오토스케일링을 검증한다.
- custom metric을 추가한다.
- 성능 관점과 리소스 관점 dashboard를 분리한다.
- 튜닝 전후를 같은 k6 시나리오로 비교한다.
- Before/After 표로 개선 수치를 정리한다.

### 성능 시나리오

| 시나리오 | 목적 |
| --- | --- |
| 티켓 오픈 피크 | 짧은 시간에 좌석 조회/예매 요청 집중 |
| 동일 좌석 경쟁 | 중복 예매 방지와 conflict 처리 확인 |
| 결제 지연 | 결제 지연이 예매 흐름 전체에 미치는 영향 확인 |
| Kafka consumer 지연 | 티켓 발행/알림 지연과 consumer lag 확인 |
| HPA scale-out | replica 증가 후 P99와 error rate 개선 확인 |

### 주요 작업

- k6 baseline script를 작성한다.
- 기준 성능을 측정한다.
- Prometheus/Grafana에서 병목 지표를 확인한다.
- HPA 적용 전후를 같은 시나리오로 비교한다.
- 필요하면 KEDA로 Kafka consumer lag 기반 scaling을 검증한다.
- Scheduled Scaling은 피크 시간 사전 replica 증가 시나리오로 문서화한다.
- Before/After 표를 작성한다.
- 미달성 항목은 2차 튜닝 방향으로 분리한다.

### 선행 의존

- MSA 서비스 구조
- Kubernetes 배포 기반
- 모니터링
- 대시보드
- 정상 예매 E2E

### 후속 의존

- 운영 보고서의 성능 개선 근거가 된다.
- SLA 준수율과 error budget 판단에 사용된다.
- HPA/KEDA 정책은 트래픽 관리 전략으로 이어진다.
- 장애 대응 시 자동 scale-out 정책의 근거가 된다.

## 도메인 9: DevSecOps와 접근 제어

### 목표

코드, 이미지, Kubernetes manifest, 런타임 권한을 배포 전후로 통제한다.

### 포함 요구사항

- SonarQube 정적 분석을 CI에 통합한다.
- coverage 80% 미만 또는 Critical issue 발견 시 pipeline을 중단한다.
- SonarQube 결과를 PR comment로 게시한다.
- Trivy config로 Kubernetes manifest 보안 위반을 차단한다.
- Slack `#security-report`로 보안 스캔 결과를 전송한다.
- RBAC을 개발자, 운영자, SRE 역할로 분리한다.
- ServiceAccount를 서비스별로 분리한다.
- Role + RoleBinding을 사용하고 ClusterRole 사용을 최소화한다.
- NetworkPolicy로 의도하지 않은 통신을 차단한다.
- OPA Gatekeeper, Falco, OWASP ZAP은 선택 확장으로 둔다.

### 주요 작업

- SonarQube quality gate를 정의한다.
- CI에 coverage threshold를 설정한다.
- Trivy image scan과 Trivy config scan을 분리한다.
- privileged container, root 실행, hostPath 사용 등을 배포 전에 차단한다.
- RBAC role matrix를 작성한다.
- 서비스별 ServiceAccount와 RoleBinding을 만든다.
- NetworkPolicy 허용/차단 시나리오를 테스트한다.
- security report Slack 알림을 검증한다.

### 선행 의존

- CI/CD와 이미지 보안
- Kubernetes 배포 기반
- 서비스별 namespace/service account 구조
- manifest 관리 방식

### 후속 의존

- 보안 운영 보고서의 근거가 된다.
- 장애 대응에서 격리 단계의 정책 근거가 된다.
- NetworkPolicy는 서비스 메시와 함께 통신 제어 검증에 사용된다.
- 발표에서 DevSecOps 파이프라인 완성도를 보여주는 근거가 된다.

## 도메인 10: 장애 대응과 운영 보고

### 목표

장애를 탐지, 알림, 분석, 조치, 회고까지 연결하고, 실제 장애 시나리오 결과를 운영 문서로 남긴다.

### 포함 요구사항

- 장애 대응 프로세스를 탐지 → 알림 → 분석 → 조치 → 회고로 문서화한다.
- 반복 장애 패턴 2개 이상을 로그 분석으로 식별한다.
- 각 장애 패턴에 대한 재발 방지 Runbook을 작성한다.
- 로그 분석 기반 SLA 준수 Kibana view를 구성한다.
- 장애 패턴 식별 결과와 개선 방안을 운영 보고서로 작성한다.
- Pod 강제 종료 시나리오에서 Istio Retry 동작을 확인한다.
- 이전 버전 rollback 절차를 Runbook으로 작성한다.
- MTTR 개선 수치를 정량화한다.

### 추천 장애 시나리오

| 시나리오 | 검증 대상 |
| --- | --- |
| `payment-service` 5xx 증가 | Alertmanager, Circuit Breaker, 결제 실패 처리 |
| `reservation-service` Pod 강제 종료 | Retry, HPA, Pod 재시작, 예매 흐름 영향 |
| Kafka consumer lag 증가 | 알림/티켓 발행 지연 탐지 |
| 신규 버전 error rate 증가 | Canary 중단, rollback, MTTR 측정 |
| 네트워크 지연 200ms 주입 | P99 증가, timeout, retry 정책 |

### 주요 작업

- 장애 탐지 기준과 alert 이름을 정리한다.
- alert 발생 시 볼 dashboard와 log query를 Runbook에 연결한다.
- 장애 시나리오별 실행 절차와 복구 절차를 작성한다.
- 실제 실행 결과를 캡처와 수치로 남긴다.
- MTTR을 측정한다.
- 반복 장애 패턴과 재발 방지책을 운영 보고서에 정리한다.
- SLA 99.9% 준수 여부 산출 방식을 문서화한다.

### 선행 의존

- 모니터링
- 로깅
- 대시보드와 알림
- 서비스 메시와 트래픽 관리
- Kubernetes 배포 기반

### 후속 의존

- 최종 발표 자료
- 운영 가이드라인
- 회고 문서
- 다음 프로젝트 인수인계 문서

## 도메인 간 핵심 의존 관계

```text
MSA 서비스 구조
  -> Kubernetes 배포 기반
    -> CI/CD와 이미지 보안
    -> 모니터링
      -> 대시보드와 알림
      -> 성능 최적화와 오토스케일링
    -> 로깅
      -> 장애 대응과 운영 보고
    -> 서비스 메시와 트래픽 관리
      -> 장애 대응과 운영 보고
    -> DevSecOps와 접근 제어
      -> 장애 대응과 운영 보고
```

## 병렬 진행 가능한 부분

아래 작업은 선행 계약만 맞으면 병렬로 진행할 수 있다.

| 병렬 작업 | 필요한 공통 계약 |
| --- | --- |
| 서비스 API 구현과 Helm chart 초안 | 서비스 이름, port, health endpoint |
| Prometheus ServiceMonitor와 Grafana dashboard 초안 | metric 이름, label 규칙 |
| Fluent Bit pipeline과 Kibana view 초안 | log field, request_id 규칙 |
| GitHub Actions와 Dockerfile 정리 | 서비스 디렉토리 구조, image 이름 |
| Istio VirtualService 초안과 ArgoCD Application | namespace, service name, version label |
| k6 script와 Postman E2E | API path, auth 방식, seed data |
| RBAC/NetworkPolicy와 ServiceAccount | namespace, service account 이름, 통신 matrix |

## 차수별 완료 범위

아래 목록은 우선순위가 아니라 완료 순서다. 1차와 2차 모두 최종 충족 범위에 포함된다.

### 1차-A: 기반과 기본 필수 완료

- 정상 예매 E2E
- 좌석 중복 예매 방지
- Kubernetes 배포
- `/metrics`, `/healthz`, `/readyz`
- Prometheus + Grafana 기본 대시보드
- Fluent Bit/Fluentd 로그 수집
- GitHub Actions test/build/deploy
- Trivy image scan
- Alertmanager Slack 알림

### 1차-B: 통합 시연 필수 완료

- Istio Canary와 rollback
- Kiali service topology
- k6 Before/After 성능 비교
- HPA scale-out 검증
- request_id 기반 로그 추적
- SonarQube quality gate
- NetworkPolicy 통신 차단/허용 검증
- 장애 Runbook과 MTTR 측정

### 2차: 심화와 선택 요구사항 최종 완료

- Elasticsearch ILM
- warning/critical email 이중 알림
- 업무 시간 외 critical escalation
- Elasticsearch shard/replica와 노드 다운 검증
- Fluentd와 Fluent Bit memory 비교
- Logstash 민감정보 masking과 error level 자동 분류
- 로그 볼륨 급증 alert
- KEDA consumer lag scaling
- Scheduled Scaling
- ArgoCD Rollouts 자동 rollback
- Trivy PR comment
- untagged image lifecycle
- Dependabot base image update
- OPA Gatekeeper
- Falco
- OWASP ZAP
- Jaeger 또는 Tempo tracing
- Istio outlierDetection
- Istio locality-aware load balancing
- Alertmanager webhook 기반 HPA min replica 자동 확장
- Chaos Mesh 장애 주입
- Kibana Canvas 주간 SLA 리포트
- Grafana Annotation
- 실시간 1초 갱신 dashboard
- 월 1회 성능 리뷰 프로세스
- Incident Response 자동화 스크립트
- 주간 SLA 리포트 자동 발송
- 대규모 shard/replica 장애 검증
- 팀 발표 자료
- 인수인계 문서

## 문서와 실행 이슈 정리 방식

1. `00-GOAL.md`는 요구사항 원문을 보존하고, 이 문서는 도메인별 실행 계획과 충족 매핑으로 사용한다.
2. 각 도메인별 Epic에는 선행 의존, 완료 기준, 검증 증거를 명시한다.
3. 실제 구현 이슈는 1차-A, 1차-B, 2차로 나누되, 2차도 최종 완료 대상임을 issue description에 명시한다.
4. 중복 요구사항은 같은 산출물을 여러 요구사항의 증거로 연결한다.
5. 최종 발표 범위는 1차 전체를 최소 기준으로 하고, 2차 항목은 최종 제출 전까지 완료 증거를 축적한다.

## 현재 코드 기준 보완

현재 repo 상태를 보면 도메인 작업에 들어가기 전에 기준선 정리가 먼저 필요하다. 상세 인벤토리는 `workspace/docs/personal/current-state-and-legacy-inventory.md`에 둔다.

### 우선 정리해야 하는 불일치

| 영역 | 현재 불일치 | 작업 영향 |
| --- | --- | --- |
| 서비스 목록 | `workspace` 목표는 티켓팅 6개 서비스지만 `service`에는 의료 서비스가 함께 남아 있음 | 테스트, 이미지 빌드, 배포 대상이 흔들림 |
| OpenAPI | `service/contracts`는 6개 티켓팅 서비스, `workspace/project_docs/openapi`는 일부 서비스만 존재 | API 계약 기준을 하나로 정해야 함 |
| E2E | `service/tests/e2e`는 의료 도메인 collection과 compose를 사용 | S0 정상 예매 E2E를 새로 만들어야 함 |
| GitOps Argo | 2026-05-29 pull 이후 AWS dev Argo Application은 `concert`, `reservation`, `payment`, `ticket` 중심으로 전환됨. 다만 환경별 values, prod override, 문서의 잔여 의료 표현은 추가 확인 필요 | AWS/GitOps 배포 검증 전에 잔여 레거시 확인 필요 |
| Infra Terraform | ECR repository 기본 목록이 의료 서비스 기준 | image push/deploy 흐름 전에 수정 필요 |
| Dashboard | 정적 화면이 의료 도메인 기준 | 티켓팅 데모 검증에는 재작성 필요 |

### 도메인 작업 전 선행 기준선

1. `service`의 기본 테스트 대상과 이미지 빌드 대상을 티켓팅 서비스 목록으로 통일한다.
2. 티켓팅 정상 예매 E2E compose와 Newman collection을 만든다.
3. `gitops/argo/applications/aws-dev/services`는 최신 pull로 티켓팅 기준 전환이 진행됐으므로, `auth`, `concert`, `reservation`, `payment`, `ticket`, `notification`, `dashboard`가 실제 values와 모두 연결되는지 검증한다.
4. `infra/terraform`의 ECR repository 목록을 티켓팅 서비스로 맞춘다.
5. `/healthz`, `/readyz`, `/metrics`를 최종 운영 endpoint 기준으로 확정하고 Helm probe/ServiceMonitor 문서와 맞춘다.

### 최신 GitOps 변경 반영

2026-05-29 pull 이후 `gitops` local dev 흐름이 다음처럼 분리됐다.

| Task | 도메인 작업에서의 의미 |
| --- | --- |
| `task dev:platform` | Kubernetes namespace, data, Kong 기반을 먼저 올리는 선행 작업 |
| `task dev:services` | 모든 티켓팅 서비스를 병렬 Helm 배포하고 rollout을 기다리는 작업 |
| `task dev:service SERVICE=<name>` | 특정 도메인 서비스 하나만 build/push/deploy하는 작업 |
| `task dev:restart SERVICE=<name>` | image 변경 없이 단일 서비스만 재시작하는 작업 |

따라서 앞으로 각 도메인 작업의 검증 단위는 다음처럼 잡는 것이 좋다.

- 서비스 구현 변경: `service` 수정 후 `gitops`에서 `task dev:service SERVICE=<service>`로 검증
- platform/data/Kong 변경: `task dev:platform`으로 먼저 검증
- 전체 통합 확인: `task dev:services` 또는 `task dev`
- 배포 이슈 발행: `workspace/docs/issues/README.md` 기준으로 부모 이슈는 `workspace`, 실제 작업 이슈는 `service`/`gitops`/`infra`에 배치
