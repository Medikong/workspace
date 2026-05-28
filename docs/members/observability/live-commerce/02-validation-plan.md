# 검증 계획

## 문서 목적

이 문서는 공연 티켓 예매 운영 검증 프로젝트에서 발표 때 보여줄 KPI, SLI/SLO, 검증 시나리오, 성공/실패 케이스, 테스트 목록을 정리한다.

## 발표 KPI

발표 KPI는 예매 기능 자체보다 티켓 오픈 상황에서 핵심 흐름이 안정적으로 유지되는지를 보여주는 지표다.

### 핵심 예매 흐름

- 예매 성공률: 피크 상황에서도 실제 비즈니스 흐름이 유지되는가
- P95/P99 응답시간: 사용자가 체감하는 좌석 점유와 예약 지연이 허용 가능한가
- 최대 처리량: 티켓 오픈 시 어느 정도 트래픽까지 처리할 수 있는가
- 에러율: 부하 또는 장애 상황에서 실패가 얼마나 발생하는가

### 비즈니스 정합성과 장애 격리

- 좌석 정합성: 동시 예매에도 같은 좌석의 유효 티켓이 중복 발행되지 않는가
- 결제 실패/보류율: 외부 의존 서비스 장애를 통제 가능한 상태로 관리하는가
- 티켓 발행 가용성: 결제 승인 이후 티켓 발행과 저장이 안정적으로 완료되는가
- 알림 장애 중 티켓 발행 성공률: 비핵심 알림 기능 장애가 핵심 예매 흐름에 전파되지 않는가

### 운영 안정성

- HPA 스케일아웃 시간: 부하 증가에 인프라가 얼마나 빨리 반응하는가
- 장애 감지 시간: 운영자가 문제를 얼마나 빨리 인지하는가
- MTTR: 장애 발생 후 정상 상태로 복구하는 데 걸리는 시간
- Canary 에러율 비교: 신규 버전을 안전하게 점진 배포할 수 있는가
- 롤백 시간: 문제 있는 배포를 얼마나 빨리 되돌릴 수 있는가

## 운영 KPI

예매 지표는 발표 소재이고, 평가 중심은 Kubernetes 기반 운영 지표다.

### 응답성과 안정성

- P99 응답시간: 부하 상황에서 서비스 응답성이 유지되는가
- 에러율: 장애, 배포, 부하 상황에서 실패가 통제되는가
- 예매 성공률: 핵심 비즈니스 흐름이 유지되는가

### Kubernetes 운영 지표

- HPA scale-out 시간: Kubernetes가 부하에 자동 대응하는가
- Pod restart/recovery time: 장애 발생 후 자동 복구되는가
- Resource usage: 제한된 private cloud 리소스를 효율적으로 쓰는가

### 장애 대응과 배포 안정성

- Alert firing time: 운영자가 장애를 빠르게 인지하는가
- MTTR: 장애 대응 절차가 실제로 복구 시간을 줄이는가
- Canary error delta: 새 버전을 안전하게 배포할 수 있는가
- Rollback time: 장애 버전을 빠르게 되돌릴 수 있는가

## SLI와 SLO 후보

SLO는 실제 운영 계약이 아니라 발표용 성공 기준으로 사용한다. 최종 수치는 구현 환경과 기준선 부하 테스트 결과가 나온 뒤 조정한다. 다만 발표 전까지는 아래 수치를 임시 기준선으로 두고, 같은 시나리오를 개선 전과 개선 후에 반복 실행해 Before/After 비교가 가능하도록 한다.

### P0-1 정상 예매 흐름

- E2E 성공률: 티켓 오픈, 공연/좌석 조회, 좌석 임시 점유, 예약 생성, mock 결제, 티켓 발행, 알림까지 99% 이상 성공한다.
- 예약 API 지연: 정상 부하 10 VU, 2분 실행에서 P95는 500ms 이하, P99는 1.5초 이하로 유지한다.
- 요청 추적성: 실패 또는 보류 예약 1건 이상을 요청 식별자와 예약/티켓 식별자로 로그에서 3분 이내에 추적할 수 있어야 한다.
- 결제 결과 상태: mock 결제 승인은 `paid`, 지연은 `payment_pending`, 실패는 `payment_failed`처럼 구분 가능한 상태로 남아야 한다.

### P0-2 티켓 오픈 피크

- 피크 에러율: 티켓 오픈 후 5분 피크 구간에서 예약 API 실패율을 1% 이하로 유지한다.
- 피크 지연: 200 VU, 5분 유지 구간에서 예약 API P99를 1.5초 이하로 유지한다.
- 처리량: 같은 k6 시나리오에서 개선 후 최대 처리량과 안정 처리량을 기준선 대비 수치로 제시한다.
- 좌석 조회 성공률: `concert-service`는 피크 중 `/concerts/{id}/seats` 성공률을 99% 이상으로 유지한다.

### P0-3 좌석 경쟁 조건

- 중복 티켓: 한 공연 회차의 같은 좌석에 500건 동시 예매를 넣어도 유효 티켓은 1장만 발행되어야 한다.
- 좌석 종료 상태: 테스트 종료 후 좌석 상태는 `reserved`, `sold`, `available` 중 하나로 일관되게 남아야 한다.
- 실패 사유: 이미 점유된 좌석 요청은 5xx가 아니라 409 Conflict 또는 동등한 비즈니스 실패로 분류한다.
- 중복 예약: 같은 사용자와 같은 공연/좌석에 대한 중복 예약이 성공 예약으로 2회 이상 기록되지 않아야 한다.

### P0-4 결제 지연/장애

- 예약 유실: `payment-service` 지연, 실패, 중단 중에도 생성된 예약의 유실은 0건이어야 한다.
- 보류 전환: 결제 응답이 3초 이상 지연되고 예약 서비스 timeout이 1초로 설정된 경우, 대상 예약의 95% 이상이 `payment_pending`으로 전환되어야 한다.
- 예약 API 보호: 결제 장애 중 예약 API 5xx 비율은 1% 이하로 유지한다.
- 복구 시간: `payment-service` 복구 후 pending 예약 재처리 또는 수동 보정 절차를 5분 이내에 시작할 수 있어야 한다.

### P0-5 알림 장애 격리

- 티켓 발행 영향도: `notification-service` 중단 또는 2초 지연 중 티켓 발행 성공률은 정상 기준선 대비 95% 이상이어야 한다.
- 예약 지연: 알림 장애 중 예약 API P99는 1.5초 이하 또는 정상 대비 +20% 이내로 유지한다.
- 장애 전파: 알림 장애가 `reservation-service`나 `ticket-service`의 5xx 증가로 전파되지 않아야 하며, 예약 API 5xx 비율은 1% 이하로 유지한다.
- 로그 분리: 알림 장애 로그와 티켓 발행 실패 로그를 서비스명, 요청 식별자, 예약 식별자 기준으로 구분할 수 있어야 한다.

### P0-6 HPA 검증

- Scale-out 시작: CPU 70% 기준 HPA에서 부하 증가 후 60초 이내에 desired replica가 증가해야 한다.
- Replica 범위: `reservation-service`는 min 2, max 10 replica 범위 안에서 동작해야 한다.
- 안정화 효과: HPA 적용 후 같은 k6 피크 시나리오에서 P99 또는 에러율이 기준선 대비 20% 이상 개선되거나 SLO 안으로 들어와야 한다.
- 리소스 기준: `reservation-service` Pod의 CPU request/limit, HPA target, replica 변화 시점을 Grafana 또는 `kubectl describe hpa` 결과로 제시한다.

### P0-7 관측성 검증

- 지표 수집: Prometheus scrape interval 15초 기준으로 예매 성공률, 에러율, P95/P99, CPU, replica 수가 누락 없이 수집되어야 한다.
- 알림 발송: 에러율 5% 초과 2분 지속 또는 P99 2초 초과 2분 지속 시 Alertmanager 알림이 1분 이내에 기록되어야 한다.
- 로그 추적: 결제 장애 또는 좌석 충돌 예약 1건을 요청 식별자나 예약 식별자로 Loki/EFK에서 3분 이내에 추적할 수 있어야 한다.
- 발표 증거: Grafana 캡처, Alertmanager 알림 기록, 로그 쿼리 결과, 장애 Runbook을 한 세트로 묶어 제시한다.

### P1/P2 확장 기준

- Canary 20% 구간: 신규 버전 에러율은 기존 버전 대비 +1%p 이하로 유지한다.
- Rollback: Canary 이상 감지 후 ArgoCD 또는 Argo Rollouts rollback을 3분 이내에 완료한다.
- Rate Limiting: 단일 사용자 과호출은 429로 제한하고 정상 사용자의 예매 성공률은 99% 이상 유지한다.
- 보안 스캔: Trivy 이미지 스캔에서 CRITICAL 취약점이 나오면 배포 후보에서 제외하고 결과 리포트를 남긴다.

## P0: 발표 필수 시나리오

P0는 발표에서 반드시 증거로 보여줄 검증 항목이다. 각 시나리오는 가능하면 기준선(Before)과 개선 후(After)를 같은 조건으로 반복 실행하고, 수치 변화와 운영 증거를 함께 보여준다.

### 1. 정상 예매 흐름

Postman/Newman과 curl로 티켓 오픈 상태 조회, 공연/좌석 조회, 좌석 임시 점유, 예약, mock 결제, 티켓 발행까지 이어지는 기본 흐름을 반복 실행한다. 실제 PG 결제는 제외하고 payment-service는 승인, 지연, 실패 응답만 제공한다.

성공 기준은 E2E 성공률 99% 이상, 예약 P99 1.5초 이하, 예약/티켓 상태 유실 0건이다. 산출물은 Newman 결과, 대표 요청/응답 로그, Grafana 캡처, 요청/예약/티켓 로그 추적 결과로 둔다.

### 2. 티켓 오픈 피크

k6로 티켓 오픈 직후 200 VU 수준의 예약 피크를 만들고, HPA 적용 전후를 같은 조건으로 비교한다. Prometheus, Grafana, kubectl, Alertmanager로 처리량, 에러율, latency, replica 변화를 본다.

성공 기준은 피크 5분 에러율 1% 이하, 예약 P99 1.5초 이하, scale-out 60초 이내다. 산출물은 k6 리포트, Before/After Grafana 캡처, HPA 관찰 로그다.

### 3. 좌석 경쟁 조건

k6와 DB 조회로 한 공연 회차의 같은 좌석 또는 제한된 좌석 묶음에 500건 수준의 동시 예매를 넣는다. 좌석 lock 또는 unique constraint 적용 전후를 비교해 정합성 개선 여부를 본다.

성공 기준은 중복 티켓 0건, 좌석 상태 일관성 유지, 충돌 요청의 비즈니스 실패 처리다. 산출물은 k6 결과, 좌석 상태 전후 조회, 성공/실패 예약 샘플이다.

### 4. 결제 지연/장애

kubectl과 mock 설정으로 payment-service 지연, 실패, 중단을 주입하고 예약 상태가 보류 또는 실패로 남는지 본다. 예약 부하는 k6/Newman으로 만들고, 장애 구간은 Grafana와 Alertmanager로 관찰한다.

성공 기준은 예약 유실 0건, pending 전환율 95% 이상, 예약 API 5xx 1% 이하, 알림 1분 이내다. 산출물은 예약 상태 분포, 장애 전후 지표, 알림 기록, 결제 장애 Runbook 초안이다.

### 5. 알림 장애 격리

kubectl로 notification-service 지연 또는 중단을 만들고, 동시에 예약 부하와 티켓 발행 흐름을 실행한다. 핵심은 알림 장애가 예약 API나 티켓 발행의 에러율과 지연으로 전파되는지 보는 것이다.

성공 기준은 알림 장애 중 티켓 발행 성공률이 정상 대비 95% 이상, 예약 5xx 1% 이하, 예약 P99 1.5초 이하 또는 정상 대비 +20% 이내다. 산출물은 장애 전후 Grafana 비교와 서비스별 로그 쿼리다.

### 6. HPA 검증

k6로 예약 피크를 만들고, reservation-service HPA가 CPU 70% 기준으로 min 2, max 10 범위에서 반응하는지 본다. metrics-server, kubectl, Prometheus, Grafana로 replica와 성능 변화를 비교한다.

성공 기준은 scale-out 60초 이내 시작, 3분 이내 4개 이상 replica 도달, P99 또는 에러율 20% 이상 개선이다. 산출물은 k6 리포트, HPA describe 결과, CPU/replica/latency 캡처다.

### 7. 관측성 검증

티켓 오픈 피크, 결제 장애, 알림 장애 중 최소 1개 이상을 골라 Prometheus, Grafana, Alertmanager, Loki/EFK로 운영자가 원인을 설명할 수 있는지 본다.

성공 기준은 에러율·P99·replica 변화가 대시보드에 보이고, 알림이 1분 이내 기록되며, 실패 예약 로그를 3분 이내 추적하는 것이다. 산출물은 대시보드 캡처, 알림 기록, 로그 쿼리, Runbook이다.

## P1: 발표 강화 시나리오

P1은 발표 설득력을 높이는 보조 검증 항목이다. P0가 완료된 뒤 시간과 구현 여력에 따라 선택한다.

### Rate Limiting

k6로 단일 사용자의 과도한 좌석 조회/예약 호출과 정상 사용자 예매 부하를 함께 실행한다.
성공 기준은 과호출은 429로 제한하고 정상 사용자 예매 성공률은 99% 이상 유지하는 것이며, 산출물은 차단 건수, 정상 사용자 지표, gateway 로그다.

### Canary 배포

ArgoCD와 Istio 또는 Argo Rollouts로 reservation-service 신규 버전을 20%, 50%, 100% 순서로 전환한다.
성공 기준은 신규 버전 에러율이 기존 대비 +1%p 이하, P99가 +20% 이내인 것이며, 산출물은 버전별 지표와 rollout 기록이다.

### Canary 롤백

Canary 구간에서 신규 버전 오류를 주입하고 Alertmanager 알림 이후 rollback을 실행한다.
성공 기준은 이상 감지 후 3분 이내 이전 버전으로 복구하고 5xx를 1% 이하로 낮추는 것이며, 산출물은 알림 기록과 rollback 이벤트 로그다.

### 티켓 저장 장애 격리

ticket-service의 S3 업로드 지연 또는 실패 중에도 예약/결제 상태가 유실되지 않는지 본다.
성공 기준은 예약 성공률 99% 이상, 티켓 저장 실패가 재시도 큐나 실패 로그로 분리되는 것이며, 산출물은 예약 지표와 티켓 artifact 실패 로그다.

### 로그 추적

실패 예약이나 pending 예약 1건을 골라 gateway, reservation, payment, ticket 로그를 한 흐름으로 추적한다.
성공 기준은 실패 원인을 3분 이내 설명하는 것이며, 산출물은 로그 쿼리와 요청 흐름 캡처다.

## P2: 선택 시나리오

P2는 시간이 남을 때 확장하는 선택 항목이다. 발표 핵심 메시지를 흐리지 않는 범위에서만 포함한다.

### 장시간 안정성

k6로 100 VU 예매 부하를 30분 유지하고 Prometheus/Grafana로 리소스 추세를 본다.
성공 기준은 에러율 1% 이하, 메모리 증가 안정화, Pod restart 0건이며, 산출물은 장시간 부하 리포트와 리소스 캡처다.

### 대기열 상태 장애

waiting-room 또는 티켓 오픈 상태 API의 응답을 지연 또는 중단해 예약 흐름과 분리되는지 본다.
성공 기준은 대기열 상태 조회 실패가 이미 시작된 예약 성공률에 영향을 주지 않는 것이며, 산출물은 상태 조회 에러율, 예약 영향도, 서비스별 로그다.

### 좌석 조회 폭주

좌석 조회 폭주와 예약 부하를 동시에 실행해 로그·메트릭 증가가 예약 흐름에 미치는 영향을 본다.
성공 기준은 예약 API 에러율 1% 이하이며, 산출물은 좌석 조회 처리량, 예약 P99, 로그 볼륨 변화다.

### 이벤트 기반 확장

Kafka consumer lag을 의도적으로 키우고 KEDA 또는 Prometheus Adapter 기반 worker scale-out을 본다.
성공 기준은 scale-out 60초 이내 시작, 5분 이내 lag 감소 전환이며, 산출물은 queue lag 그래프와 scale-out 이벤트다.

### 보안 정책 검증

kubectl, curl, NetworkPolicy, RBAC, Trivy로 결제 서비스 직접 접근과 취약한 배포 후보를 차단하는지 본다.
성공 기준은 비정상 접근 차단과 CRITICAL 취약점 배포 차단이며, 산출물은 허용/차단 로그, 정책 manifest, Trivy 리포트다.

## 테스트 목록

테스트 목록은 구현 완료 여부가 아니라 발표 증거를 만들기 위한 실행 단위다.

### P0-1 정상 예매 E2E

- 연결 시나리오: P0-1 정상 예매 흐름.
- 실행 도구: Postman/Newman, curl, Prometheus, Grafana, Loki 또는 EFK.
- 만드는 증거: E2E 성공률, 예약/티켓 상태 전이, 결제 pending 전환, 요청/예약/티켓 로그 추적.

### P0-2 티켓 오픈 피크 부하

- 연결 시나리오: P0-2 티켓 오픈 피크, P0-6 HPA 검증.
- 실행 도구: k6, Prometheus, Grafana, kubectl, Alertmanager.
- 만드는 증거: 처리량, k6 실패율, P95/P99, HPA scale-out 시간, Before/After 비교.

### P0-3 좌석 경쟁 정합성

- 연결 시나리오: P0-3 좌석 경쟁 조건.
- 실행 도구: k6, Newman, DB 조회 스크립트, Loki 또는 EFK.
- 만드는 증거: 성공 예약 수, 좌석 충돌 응답 수, 중복 티켓 0건, 최종 좌석 상태, 중복 예약 차단.

### P0-4 결제 장애 주입

- 연결 시나리오: P0-4 결제 지연/장애, P0-7 관측성 검증.
- 실행 도구: kubectl, k6, Newman, Prometheus, Grafana, Alertmanager, Loki 또는 EFK.
- 만드는 증거: 예약 유실 0건, pending 전환율, 장애 감지 시간, MTTR, Runbook 실행 기록.

### P0-5 알림 장애 격리

- 연결 시나리오: P0-5 알림 장애 격리.
- 실행 도구: kubectl, k6, Prometheus, Grafana, Loki 또는 EFK.
- 만드는 증거: 알림 장애 중 티켓 발행 성공률, 예약 P99 변화, 장애 영향 서비스 범위, 서비스별 로그.

### P0-6 HPA 동작 기록

- 연결 시나리오: P0-2 티켓 오픈 피크, P0-6 HPA 검증.
- 실행 도구: k6, kubectl, metrics-server, Prometheus, Grafana.
- 만드는 증거: desired/current replica 변화, CPU 사용률, scale-out 시작 시간, 성능 개선율.

### P0-7 관측성 증거 수집

- 연결 시나리오: P0-4 결제 지연/장애, P0-5 알림 장애 격리, P0-7 관측성 검증.
- 실행 도구: Prometheus, Grafana, Alertmanager, Loki 또는 EFK, kubectl.
- 만드는 증거: 대시보드 캡처, alert history, 로그 쿼리, 장애 원인 설명, Runbook.

### P1 Rate Limiting

- 연결 시나리오: P1 Rate Limiting.
- 실행 도구: k6, gateway 설정, Prometheus, Grafana.
- 만드는 증거: 429 차단 건수, 정상 사용자 성공률, gateway 로그.

### P1 Canary와 Rollback

- 연결 시나리오: P1 Canary 배포, P1 Canary 롤백.
- 실행 도구: ArgoCD, Istio 또는 Argo Rollouts, Prometheus, Alertmanager, kubectl.
- 만드는 증거: 버전별 에러율, latency 비교, rollback 시간, ArgoCD sync/rollback 기록.

### P2 장시간 안정성과 이벤트 확장

- 연결 시나리오: P2 장시간 안정성, P2 이벤트 기반 확장.
- 실행 도구: k6, Kafka exporter, KEDA 또는 Prometheus Adapter, Prometheus, Grafana.
- 만드는 증거: 장시간 에러율, resource trend, consumer lag, worker scale-out 이벤트.

### P2 보안과 네트워크 정책

- 연결 시나리오: P2 보안 정책 검증.
- 실행 도구: kubectl, curl, NetworkPolicy, ServiceAccount/RBAC, Trivy.
- 만드는 증거: 허용/차단 로그, 정책 manifest, Trivy 리포트.
