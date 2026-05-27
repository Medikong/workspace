# Workplans

이 폴더는 `EPICS.md`의 Epic 기준으로 Linear/GitHub Projects 업로드 후보 작업을 YAML로 나눈 곳이다. 작업 계획의 편집 원본은 이 폴더의 YAML 파일이다.

## 파일 구성

| 파일 | Epic | 범위 |
| --- | --- | --- |
| `00-foundation.yaml` | `epic:foundation` | 요구사항 정렬, 시나리오 우선순위, 역할/소유 영역, Ready 기준 |
| `05-aws-demo-environment.yaml` | `epic:aws-demo-environment` | Terraform 인벤토리, demo/QA 환경 범위, AWS 접근, 배포 smoke |
| `10-e2e-baseline.yaml` | `epic:e2e-baseline` | 정상 사용자 흐름, DNS/Gateway, 테스트 데이터, 기준 E2E |
| `15-gitops-argocd-release.yaml` | `epic:gitops-argocd-release` | GitOps/ArgoCD 구성 gap, sync smoke, rollback runbook |
| `20-observability.yaml` | `epic:lgtm-observability` | 메트릭 수집, Prometheus/Grafana, 로그 조회, tracing 판단, 장애 캡처 |
| `30-traffic-policy.yaml` | `epic:traffic-policy` | 호출 매트릭스, Gateway/Mesh 경계, NetworkPolicy, 차단 검증 |
| `40-devsecops.yaml` | `epic:devsecops` | Trivy scan, 위험 manifest 샘플, 실패 결과, 보안 증거 |
| `50-delivery-reliability.yaml` | `epic:delivery-reliability` | 독립 배포 대상, probes/PDB, 재배포, E2E 재검증 |
| `60-evidence-presentation.yaml` | `epic:evidence-presentation` | 발표 스토리라인, 다이어그램, 캡처 정리, 리허설 |

## 작성 규칙

- 각 파일은 `linear-workplan.schema.json` 스키마를 따른다.
- `local_id`는 모든 파일을 통틀어 유일해야 한다.
- `depends_on`은 다른 workplan 파일의 `local_id`를 참조할 수 있다.
- 스키마는 확장하지 않고 `labels`에 `scenario:*`, `epic:*`를 넣어 추적한다.
- `done_when`과 `evidence`를 반드시 채워 구현 완료와 검증 완료를 분리한다.
- 선택 과업은 `priority: 4` 또는 `related_to`로 연결하고, P0 흐름을 막지 않게 둔다.

## 라벨 규칙

```yaml
labels:
  - scenario:s2
  - epic:lgtm-observability
  - observability
```

## 로컬 그래프 뷰어

이 폴더의 YAML 파일이 workplan의 단일 진실 원천이다. 의존성 그래프는 다음 명령으로 정적 HTML을 생성해서 확인한다.

```bash
node docs/projects_plan/scripts/render-workplan-graph.mjs
```

- 출력 파일: `docs/projects_plan/.generated/workplan-graph.html`
- 그래프 노드 클릭 시 `local_id`, `source_file`, `milestone`, `type`, `priority`, `estimate`, 직접 선행/후행 작업 수를 확인한다.
- `depends_on`, `blocks`, `related_to`, `done_when`, `evidence`, `metrics`는 상세 패널에서 확인한다.
- HTML은 읽기 전용 생성물이며, 이슈 구조 변경은 YAML을 수정한 뒤 다시 생성한다.

## 검토 순서

1. `00-foundation.yaml`에서 P0 시나리오와 Ready 기준을 확정한다.
2. `05-aws-demo-environment.yaml`로 AWS demo/QA 시연 환경의 최소 접근과 배포 smoke를 확인한다.
3. `15-gitops-argocd-release.yaml`로 ArgoCD sync 또는 단기 대체 배포 경로를 확인한다.
4. `10-e2e-baseline.yaml`로 정상 흐름 기준선을 만든다.
5. Sprint 2에서는 `20`, `30`, `40` 중 P0 시나리오를 우선 처리하고 2026-06-12 중간 발표에 초기 결과를 공유한다.
6. Sprint 3에서는 2026-06-19까지 `20`, `30`, `40`, 필요 시 `50`의 실험 결과를 freeze한다.
7. Final Prep에서는 `60`으로 증거, 발표 흐름, 리허설, 백업 캡처를 고정한다.
