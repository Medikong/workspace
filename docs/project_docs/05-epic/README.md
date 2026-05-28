---
id: ticketing-final-epics
title: 공연 티켓 예매 서비스 Epic 목록
type: epic-list
status: draft
tags: [ticketing, epic]
created: 2026-05-28
updated: 2026-05-28
---

- Epic = 프로젝트의 큰 작업 묶음
- Issue = 한 사람이 맡아 처리할 수 있는 작업
- Task = Issue 안에서 나누는 세부 할 일

# 공연 티켓 예매 서비스 Epic 목록

## Epic 1. API Contract & Common Rules

- 공통 개발 규약
  - 브랜치 전략 정의
  - 커밋 메시지 규칙 정의
  - PR 규칙 정의
  - 코드 리뷰 규칙 정의
  - 로그 포맷 정의
  - traceId/requestId 규칙 정의
  - 환경변수 규칙 정의
  - Dockerfile 규칙 정의
- API 공통 계약
  - 서비스 구현 전 OpenAPI 파일 정의
  - API 응답 포맷 정의
  - 공통 ErrorResponse 정의
  - 에러 코드 규칙 정의
  - 공통 JWT claim 정의
  - 공통 Header 규칙 정의
  - 공통 schema/component 파일 작성
- 서비스별 OpenAPI
  - auth-service OpenAPI 작성
  - concert-service OpenAPI 작성
  - reservation-service OpenAPI 작성
  - payment-service OpenAPI 작성
  - ticket-service OpenAPI 작성
  - notification-service OpenAPI 작성
- Kafka 이벤트 계약
  - 서비스 구현 전 Kafka 이벤트 파일 정의
  - AsyncAPI 이벤트 스펙 작성
  - Kafka topic 목록 정의
  - Kafka 이벤트 payload schema 작성
  - 이벤트 version/correlationId 규칙 정의

## Epic 2. Local Development Environment

- docker-compose 데이터베이스 및 메시징/캐시 의존성 구성
- 로컬 Kubernetes 공통 Helm chart 템플릿 작성
- 로컬 Kubernetes values 템플릿 작성
- 로컬 Kubernetes namespace 구성
- 로컬 Kubernetes 배포 스크립트 작성
- 각 서비스 local env 구성

## Epic 3. Service Implementation

### 공통 제공 항목

- `/healthz` 제공
- `/readyz` 제공
- `/metrics` 제공
- OpenAPI 문서 제공
- Dockerfile 작성
- 서비스별 Helm values 작성
- unit test 작성
- integration test 작성
- structured JSON log 적용

### auth-service

- 로그인 API 구현
- refresh API 구현
- JWT role claim 구현
- auth OpenAPI contract test 작성

### concert-service

- 공연 목록 조회 구현
- 공연 상세 조회 구현
- 회차 조회 구현
- 좌석 배치 조회 구현
- seed data 작성

### reservation-service

- 좌석 lock 구현
- 예약 생성 구현
- 예약 조회 구현
- 예약 취소 구현
- 예약 만료 처리 구현
- 동일 좌석 동시 예약 테스트 작성

### payment-service

- 결제 mock API 구현
- 성공/실패/지연 시뮬레이션 구현
- 결제 이벤트 발행 구현

### ticket-service

- 티켓 발행 API 구현
- QR 생성 mock 구현
- PDF 생성 mock 구현
- S3 저장 구현
- 티켓 조회 구현

### notification-service

- Kafka 이벤트 소비 구현
- 알림 저장 구현
- 알림 조회 구현

## Epic 4. Event Flow

- ReservationCreated 이벤트 정의
- PaymentApproved 이벤트 정의
- PaymentFailed 이벤트 정의
- TicketIssued 이벤트 정의
- NotificationCreated 이벤트 정의
- Kafka topic 생성 스크립트 작성
- producer/consumer 공통 규약 작성
- 이벤트 재처리 전략 정의

## Epic 5. AWS Infrastructure

- DEV/QA 환경 분리 기준 정의
- Terraform env/workspace 구성
- Terraform state backend 구성
- VPC/Subnet/Security Group 구성
- IAM role/policy 구성
- ECR repository 부트스트랩
- EBS 등 고정 스토리지 자원 부트스트랩
- S3 bucket 또는 object storage 자원 구성
- EC2 instance 생성
- instance profile/key pair 구성
- Ansible inventory 생성
- 인스턴스 기본 패키지 및 런타임 구성
- kubeadm 기반 클러스터 노드 구성
- AWS 환경 teardown 스크립트 작성

## Epic 6. Kubernetes / GitOps

- GitOps repository 구조 구성
- DEV/QA 환경별 Helm values 구성
- namespace 선언
- ConfigMap/Secret 선언
- Deployment/Service/Ingress 선언
- readiness/liveness probe 설정
- GitHub Actions image build workflow 연결
- ECR image tag 참조 규칙 구성
- Argo CD app 구성
- Argo CD sync 정책 구성
- image tag update 전략 구성
- GitOps manifest render/validate 스크립트 작성

## Epic 7. Observability

- JSON logging 적용
- requestId/traceId 전파
- OpenTelemetry SDK 적용
- Prometheus metrics endpoint 구현
- 파일 기반 운영 Grafana dashboard 구성
- 파일 기반 성능/부하 테스트 Grafana dashboard 구성
- Loki log 수집 구성
- Tempo/Jaeger trace 수집 구성
- Kafka consumer lag 모니터링 구성

## Epic 8. Test Automation

- Testcontainers 기반 서비스 테스트 자동화
- API contract test 자동화
- E2E test 자동화
- k6 load test 자동화
- 파일 기반 시나리오 테스트 실행기 작성
- 장애/복구 시나리오 테스트 자동화
- PR/main/release branch CI 테스트 workflow 구성

## Epic 9. Scenario Validation

- 로컬 환경 검증
  - S0 정상 예매 E2E 검증
  - S1 동일 좌석 동시 예약 검증
  - S3 Kafka 후속 처리 분리 검증

- AWS DEV/QA 환경 검증
  - S2 티켓 오픈 피크 검증
  - S6 HPA scale-out 검증
  - S8 Rate Limiting 검증
  - S9 Canary 배포 검증
  - S10 Canary 롤백 검증
  - S11 보안 스캔과 정책 검증

- 장애/복구 검증
  - S4 결제 지연/장애 검증
  - S5 알림 장애 격리 검증
  - 티켓 발급 실패 검증
  - notification-service 장애 후 재처리 검증

- 증거 수집
  - Newman 결과 수집
  - k6 리포트 수집
  - Grafana 캡처 수집
  - Prometheus query 결과 수집
  - Loki 로그 캡처 수집
  - Tempo/Jaeger trace 캡처 수집
  - Kubernetes event/log 수집
  - DB 최종 상태 수집

- 검증 결과 정리
  - 시나리오별 성공/실패 결과 정리
  - 미완료 시나리오 사유 정리
  - 발표용 증거 목록 작성
