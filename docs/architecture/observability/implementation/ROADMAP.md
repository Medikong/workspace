# 관측성 구현 로드맵

관련 이슈:

- workspace#8: https://github.com/Medikong/workspace/issues/8
- workspace#13: https://github.com/Medikong/workspace/issues/13
- gitops#4: https://github.com/Medikong/gitops/issues/4

## 결론

관측성 구현은 Prometheus 기반 metric 수집부터 진행한다.

다만 첫 작업은 `kube-prometheus-stack` 설치만이 아니다. `workspace#8`의 범위는 지표 정의와 수집이므로, 서비스가 metric을 어떻게 노출하는지 확인하고 Prometheus가 그 대상을 안정적으로 scrape할 수 있게 만드는 것이 첫 기준이다.

```text
지표 기준 확인
-> 서비스 metric 노출 확인
-> kube-prometheus-stack 배포
-> ServiceMonitor / PodMonitor 연결
-> Prometheus query 확인
-> Grafana dashboard와 PrometheusRule 추가
-> logs / traces 수집으로 확장
```

## 1. 지표 수집 기준 확인

목표:

- 어떤 서비스를 scrape할지 정한다.
- 서비스 metric과 Kubernetes metric을 분리해서 본다.
- metric label에 올리면 안 되는 값을 다시 확인한다.

작업:

- `../metrics/README.md`의 서비스 SLI, 비즈니스 metric, Pod/Container, Kubernetes 상태 기준을 확인한다.
- 서비스별 `/metrics` 노출 여부와 endpoint 경로를 확인한다.
- Prometheus scrape 대상에 들어갈 namespace, service name, port name을 정리한다.

완료 기준:

- 서비스별 metric endpoint 목록이 있다.
- ServiceMonitor/PodMonitor에 필요한 label selector와 port name을 알 수 있다.
- `request_id`, `trace_id`, 주문/결제 ID 같은 고카디널리티 값은 metric label에서 제외하기로 확인한다.

## 2. 서비스 metric 노출 확인

목표:

- Prometheus가 읽을 수 있는 service metric이 실제로 존재하는지 확인한다.

작업 repo:

- `service`

작업:

- 각 FastAPI 서비스가 `/metrics`를 노출하는지 확인한다.
- 이미 `/metrics`가 있다면 Prometheus text format으로 정상 응답하는지 확인한다.
- 없다면 Prometheus client 또는 OpenTelemetry metrics export 중 하나를 정한다.
- HTTP 요청량, 에러율, 응답시간에 필요한 기본 metric 이름을 확인한다.

완료 기준:

- 로컬 또는 클러스터 안에서 서비스별 `/metrics` 응답을 확인할 수 있다.
- Prometheus query에서 사용할 실제 metric 이름 후보가 나온다.
- 서비스 코드 변경이 필요하면 별도 service 이슈로 분리할 수 있다.

## 3. Prometheus 기본 스택 배포

목표:

- Kubernetes 안에 metric 수집 기준점을 만든다.

작업 repo:

- `gitops`

작업:

- `kube-prometheus-stack` Helm chart를 `monitoring` namespace 기준으로 선언한다.
- Prometheus Operator, Prometheus, Alertmanager, Grafana, kube-state-metrics, node-exporter를 포함한다.
- GitOps로 관리할 values 파일과 Argo CD Application 위치를 정한다.
- CRD 설치 순서와 chart sync 순서를 확인한다.

완료 기준:

- Prometheus, Grafana, Alertmanager Pod가 정상 기동한다.
- kube-state-metrics와 node-exporter target이 Prometheus에서 `up`으로 보인다.
- Grafana에서 Prometheus datasource를 조회할 수 있다.

## 4. ServiceMonitor / PodMonitor 연결

목표:

- Prometheus가 각 서비스의 metric endpoint를 자동으로 찾고 scrape하게 한다.

작업 repo:

- `gitops`
- 필요 시 `service`

작업:

- 서비스별 `ServiceMonitor` 또는 `PodMonitor`를 작성한다.
- namespace selector, label selector, endpoint port name을 실제 service manifest와 맞춘다.
- scrape interval은 앱 metric과 인프라 metric을 구분해서 잡는다.
- target down, label mismatch, port name mismatch를 확인한다.

완료 기준:

- Prometheus target 화면에서 서비스별 target이 `up`으로 보인다.
- `up`, HTTP request count, latency bucket 같은 기본 query가 동작한다.
- metric 이름이 문서의 후보와 다르면 실제 노출 이름을 기준으로 문서를 갱신한다.

## 5. Grafana 기본 dashboard 작성

목표:

- metric이 실제 운영 화면에서 읽히는지 확인한다.

작업 repo:

- `gitops`
- 기준 정리는 `workspace`

작업:

- 서비스별 request rate, error rate, latency 패널을 만든다.
- Pod restart, Ready, CPU, memory 패널을 만든다.
- dashboard JSON을 GitOps repo에서 관리한다.
- datasource UID/name을 고정해 dashboard import 후에도 query가 흔들리지 않게 한다.

완료 기준:

- Grafana에서 서비스 metric과 Kubernetes metric을 같은 화면에서 볼 수 있다.
- dashboard JSON이 GitOps 관리 대상에 들어간다.
- 발표나 검증에 쓸 수 있는 캡처 기준이 생긴다.

## 6. PrometheusRule과 Alertmanager 연결

목표:

- metric 수집을 운영 알림 기준으로 확장한다.

작업 repo:

- `gitops`

작업:

- PrometheusRule로 기본 alert rule을 작성한다.
- Alertmanager routing과 notification target을 연결한다.
- 처음에는 scrape 실패, target down, Pod restart, Ready false처럼 명확한 운영 alert부터 둔다.
- 서비스별 error rate와 latency alert는 실제 metric 이름을 확인한 뒤 추가한다.

완료 기준:

- Prometheus에서 alert rule이 로드된다.
- Alertmanager에서 firing/pending 상태를 확인할 수 있다.
- 테스트용 alert가 notification target까지 전달된다.

## 7. Logs와 traces 수집으로 확장

목표:

- metric 기준점이 잡힌 뒤 로그와 trace를 같은 운영 화면에 연결한다.

작업 repo:

- `service`
- `gitops`

작업:

- Python/FastAPI 로그를 stdout/stderr 한 줄 JSON으로 정리한다.
- OTel Collector Agent의 filelog receiver로 container log를 수집한다.
- Loki를 기본 log backend로 연결한다.
- OpenTelemetry trace context를 HTTP/Kafka 구간에 전파한다.
- Tempo datasource와 Grafana trace-to-logs 연결을 설정한다.

완료 기준:

- 실패 요청 1건을 metric에서 찾고, 같은 시점의 Loki log와 Tempo trace로 이어볼 수 있다.
- 로그와 trace에도 `service.name`, `trace_id`, `span_id`, `request_id`가 남는다.
- 로그 수집 경로와 metric 수집 경로가 섞이지 않는다.

## 작업 순서 요약

| 순서 | 중심 repo | 작업 | 완료 신호 |
|---|---|---|---|
| 1 | `workspace` | 지표 기준 확인 | scrape 대상과 metric label 기준이 정리됨 |
| 2 | `service` | `/metrics` 노출 확인 | 서비스별 metric endpoint가 확인됨 |
| 3 | `gitops` | `kube-prometheus-stack` 배포 | Prometheus/Grafana/Alertmanager가 기동함 |
| 4 | `gitops` | ServiceMonitor/PodMonitor 연결 | 서비스 target이 `up`으로 보임 |
| 5 | `gitops` | Grafana dashboard 작성 | 기본 운영 화면에서 metric 조회 가능 |
| 6 | `gitops` | PrometheusRule/Alertmanager 연결 | 기본 alert가 전달됨 |
| 7 | `service`, `gitops` | logs/traces 확장 | metric에서 log/trace로 이어볼 수 있음 |

## 지금 바로 시작할 작업

1. `service` repo에서 각 서비스의 `/metrics` 노출 여부를 확인한다.
2. `gitops#4`에서 `kube-prometheus-stack` 배포 위치와 values 파일 위치를 정한다.
3. 첫 ServiceMonitor 대상은 실제 metric endpoint가 확인된 서비스로 제한한다.
4. Prometheus에서 노출된 실제 metric 이름을 확인한 뒤 Grafana query와 문서를 맞춘다.
