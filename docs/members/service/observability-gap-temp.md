# service 문서 보강 위치별 후보

`docs/members/observability/live-commerce` 기준으로 `docs/members/service` 문서에 추가하면 좋은 항목을, 서비스 문서의 어느 부분에 보충하면 좋은지 기준으로 정리한 임시 문서다.

## 서비스 목표와 성공 기준에 보충

현재 서비스 문서에는 중복 티켓 방지, Kafka 분리, 장애 격리, mTLS, 관측성, S3 저장 같은 목표가 있다. 여기에 발표용 정량 기준을 더 붙이면 목표가 더 검증 가능해진다.

- 정상 예매 E2E 성공률 99% 이상
- 예약 API P95 500ms 이하, P99 1.5초 이하
- 티켓 오픈 피크 구간 예약 API 실패율 1% 이하
- 좌석 조회 API 성공률 99% 이상
- HPA scale-out 60초 이내 시작
- Alertmanager 알림 1분 이내 기록
- 장애 발생 후 MTTR 측정
- Canary 이상 감지 후 rollback 3분 이내 완료
- 실패 또는 보류 예약 1건을 요청 ID, 예약 ID, 티켓 ID로 3분 이내 추적

## 추천 서비스 구성에 보충

현재 서비스 구성은 도메인 서비스 중심으로 잘 정리되어 있다. 운영 검증 관점에서는 gateway와 각 서비스의 장애 상태 표현을 더 명확히 두면 좋다.

- `gateway-service`: 인증, 라우팅, Rate Limiting, 429 응답 검증
- `payment-service`: 승인, 지연, 실패, 중단을 재현하는 mock 결제 서비스
- `reservation-service`: 결제 지연 시 `payment_pending`, 결제 실패 시 `payment_failed` 상태 전환
- `ticket-service`: S3 업로드 지연 또는 실패 시 예약/결제 상태와 분리
- `notification-service`: 장애 중에도 핵심 예약/티켓 발행 흐름에 5xx를 전파하지 않음

## Kafka 이벤트 설계에 보충

현재 이벤트 topic과 producer/consumer는 정의되어 있다. 검증까지 연결하려면 lag, delay, scale-out 관찰 항목을 추가하면 좋다.

- Kafka consumer lag을 주요 운영 지표로 추가
- `payment-approved` 이후 ticket issue delay 측정
- `ticket-issued` 이후 notification delay 측정
- Kafka exporter를 통한 lag 수집 후보
- KEDA 또는 Prometheus Adapter 기반 worker scale-out 후보
- 이벤트 지연 증가 시 Grafana에서 lag 감소와 scale-out 이벤트 확인

## 핵심 데이터 모델에 보충

현재 좌석 lock과 예약 모델은 들어가 있다. 장애/보정 상태를 데이터 모델에 반영하면 검증 시나리오와 연결하기 쉽다.

- 예약 상태에 `payment_pending`, `payment_failed` 추가
- 결제 장애 중 생성된 예약 유실 0건 검증 가능하도록 상태 전이 기록
- pending 예약 재처리 또는 수동 보정 여부를 추적할 필드 후보 추가
- 티켓 저장 실패 상태 또는 artifact upload failure 로그 연동
- 요청 ID, 예약 ID, 티켓 ID를 로그/trace와 연결할 correlation field 명시

## 좌석 중복 방지 구현 전략에 보충

현재 unique constraint, Redis lock, scheduler 후보가 있다. 관측성 문서 기준으로는 테스트 조건과 성공 기준을 더 구체화하면 좋다.

- 같은 좌석 500건 동시 예매 테스트
- 중복 티켓 0건
- 이미 점유된 좌석 요청은 5xx가 아니라 409 Conflict 또는 동등한 비즈니스 실패로 처리
- 테스트 종료 후 좌석 상태가 `reserved`, `sold`, `available` 중 하나로 일관되게 남는지 확인
- k6 결과, 좌석 상태 전후 조회, 성공/실패 예약 샘플을 증거로 남김

## API 설계 초안에 보충

현재 예약/결제/티켓 조회 API 예시는 있다. 장애와 Rate Limiting 검증을 위해 응답 상태 예시를 추가하면 좋다.

- 결제 지연 시 예약 상태 `payment_pending` 응답 예시
- 결제 실패 시 `payment_failed` 응답 예시
- 좌석 충돌 시 409 Conflict 응답 예시
- Rate Limiting 적용 시 429 Too Many Requests 응답 예시
- 요청 추적을 위한 `X-Request-ID` 또는 동등한 correlation header 후보

## 배포 구조에 보충

현재 namespace, Kong, Istio, observability namespace는 들어가 있다. 운영 검증 관점에서는 HPA, rollout, 정책 리소스를 배포 구조에 포함하면 좋다.

- `reservation-service` HPA: CPU 70%, min 2, max 10 replica
- metrics-server
- NetworkPolicy
- ServiceAccount/RBAC
- ArgoCD rollback 대상 리소스
- Argo Rollouts 또는 Istio traffic split 기반 Canary 후보
- `kubectl describe hpa`와 Grafana로 replica 변화 시점 기록

## 구현 순서에 보충

현재 구현 순서는 서비스 구현 후 Kafka, S3, Kong, Istio, Observability, 부하 테스트로 이어진다. 검증 중심 프로젝트라면 테스트와 증거 수집 단계를 더 앞쪽부터 병행하면 좋다.

- P0/P1/P2 테스트 우선순위 확정
- 테스트 시나리오별 성공 기준과 증거 목록 먼저 정의
- 정상 예매 E2E를 Newman/curl로 먼저 고정
- k6 피크 부하 테스트를 HPA 적용 전/후로 반복 실행
- payment-service 장애 주입과 Runbook 작성
- notification-service 장애 주입과 로그 분리 검증
- Rate Limiting, Canary, Rollback은 P1 강화 시나리오로 분리

## Load Test and Goal Evidence에 보충

현재 `k6/Locust`, Kafka delay, 장애 주입, 보안 스캔이 있다. 관측성 문서의 테스트 체계를 옮기면 발표 증거가 더 명확해진다.

- P0-1 정상 예매 E2E: Newman, curl, Grafana, Loki/EFK
- P0-2 티켓 오픈 피크: k6 200 VU, Prometheus, Grafana, kubectl, Alertmanager
- P0-3 좌석 경쟁 정합성: k6, Newman, DB 조회, Loki/EFK
- P0-4 결제 장애 주입: kubectl, k6, Newman, Prometheus, Grafana, Alertmanager
- P0-5 알림 장애 격리: kubectl, k6, Prometheus, Grafana, Loki/EFK
- P0-6 HPA 동작 기록: k6, kubectl, metrics-server, Prometheus, Grafana
- P0-7 관측성 증거 수집: Grafana 캡처, alert history, 로그 쿼리, Runbook

## Service Mesh와 Gateway 검증에 보충

현재 mTLS와 traffic split은 들어가 있다. gateway 보호와 배포 안정성 시나리오를 함께 붙이면 좋다.

- 단일 사용자의 과도한 좌석 조회/예약 호출 시나리오
- gateway에서 과호출을 429로 제한
- 정상 사용자 예매 성공률 99% 이상 유지
- `reservation-service` 신규 버전을 20%, 50%, 100% 순서로 전환
- 신규 버전 에러율이 기존 대비 +1%p 이하인지 검증
- 신규 버전 P99가 기존 대비 +20% 이내인지 검증
- Canary 이상 감지 후 3분 이내 rollback
- 버전별 에러율, latency, ArgoCD sync/rollback 기록을 증거로 남김

## Observability and Tracing에 보충

현재 Prometheus, Grafana, Loki, Tempo, OpenTelemetry는 들어가 있다. 빠진 것은 Alertmanager와 실제 장애 추적 기준이다.

- Alertmanager 추가
- 에러율 5% 초과 2분 지속 또는 P99 2초 초과 2분 지속 시 알림
- 알림 1분 이내 기록
- 결제 장애 또는 좌석 충돌 예약 1건을 Loki/EFK에서 3분 이내 추적
- gateway -> reservation -> payment -> ticket -> notification 흐름을 trace로 확인
- Grafana 캡처, Alertmanager 기록, 로그 쿼리, trace 캡처를 한 세트로 보관

## Cloud Data and Storage에 보충

현재 RDS/S3/VPC/Terraform은 들어가 있다. S3 장애 격리와 복구 기준을 추가하면 더 운영 검증답다.

- `ticket-service`의 S3 업로드 지연 또는 실패 시나리오
- S3 장애 중에도 예약/결제 상태 유실 없음
- 티켓 저장 실패가 재시도 큐 또는 실패 로그로 분리되는지 확인
- 예약 성공률, 티켓 저장 실패 로그, artifact 상태를 증거로 남김
- RDS snapshot, S3 version restore 절차를 실제 검증 항목과 연결

## DevSecOps와 보안 검증에 보충

현재 Trivy/SonarQube 결과는 들어가 있다. 네트워크와 권한 정책 검증을 추가하면 보안 항목이 더 구체적이다.

- NetworkPolicy로 허용된 서비스 간 통신만 가능한지 검증
- ServiceAccount/RBAC 권한 범위 검증
- 결제 서비스 직접 접근 차단 시나리오
- Trivy 이미지 스캔에서 CRITICAL 취약점이 있으면 배포 후보 제외
- 허용/차단 로그, 정책 manifest, Trivy 리포트를 증거로 남김

## 발표 산출물에 보충

현재 최종 산출물은 API/event contract, manifest, dashboard, Terraform, 테스트 결과로 정리되어 있다. 발표용 증거 묶음을 더 명시하면 좋다.

- Before/After 비교표
- k6 리포트
- Newman 결과
- Grafana 캡처
- Alertmanager 기록
- Loki/EFK 로그 쿼리
- HPA describe 결과
- ArgoCD sync/rollback 기록
- 장애 Runbook 실행 기록
- 발표용 증거 인덱스
