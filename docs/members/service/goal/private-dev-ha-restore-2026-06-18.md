# private-dev HA 검증용 리소스 복구 기록

## 1. 목적

private-dev에서 서비스별 최소 Pod 2개 유지와 PDB 동작을 검증하기 위해, 임시 smoke 기준으로 낮춰져 있던 서비스 replica/HPA 값을 HA 검증 기준으로 복구한다.

이번 변경은 대규모 부하 테스트가 아니라 다음 항목을 확인하기 위한 것이다.

- 서비스별 기본 Pod 2개 유지
- HPA `minReplicas` 2 유지
- 제한적 자동 확장 가능성 확보
- PDB `minAvailable: 1` 기준에서 voluntary disruption 허용량 확인
- private-dev 클러스터 리소스 과사용 방지

## 2. 적용 기준

```yaml
deployment:
  replicas: 2

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 4
  targetCPUUtilizationPercentage: 70

pdb:
  enabled: true
  minAvailable: 1
```

## 3. maxReplicas를 4로 둔 이유

`maxReplicas: 10`은 운영 또는 대규모 부하 대응 검증에는 의미가 있지만, 현재 private-dev 검증 목적에는 과하다.

private-dev에서 이번에 확인하려는 것은 "얼마나 크게 확장되는가"가 아니라 "최소 2개 Pod 기준과 PDB가 실제로 적용되는가"이다. 따라서 `minReplicas: 2`, `maxReplicas: 4` 조합이 더 적절하다.

- 평상시: 서비스별 Pod 2개 유지
- 부하 발생 시: 최대 4개까지 확장 가능
- 서비스 수가 많아졌을 때 전체 Pod/sidecar 증가량 제한
- PDB 검증에 필요한 기본 replica 수 확보

## 4. GitOps 변경 위치

기본 private-dev 환경값:

- `gitops/values/env/private-dev.yaml`

실제 ArgoCD Application에서 마지막으로 적용되는 HA 검증 override:

- `gitops/values/overrides/private-dev-ha-stable.yaml`

private-dev 서비스 Application 참조 변경:

- `gitops/argo/applications/private-dev/services/auth.yaml`
- `gitops/argo/applications/private-dev/services/concert.yaml`
- `gitops/argo/applications/private-dev/services/dashboard.yaml`
- `gitops/argo/applications/private-dev/services/frontend.yaml`
- `gitops/argo/applications/private-dev/services/notification.yaml`
- `gitops/argo/applications/private-dev/services/payment.yaml`
- `gitops/argo/applications/private-dev/services/reservation.yaml`
- `gitops/argo/applications/private-dev/services/ticket.yaml`

## 5. 왜 override 파일을 새로 만들었나

private-dev의 서비스별 ArgoCD Application은 Helm valueFiles를 순서대로 병합한다. 뒤에 있는 파일이 앞의 값을 덮어쓴다.

기존 일부 서비스는 마지막에 다음 파일을 참조하고 있었다.

```yaml
- $values/values/overrides/private-dev-smoke-stable.yaml
```

이 파일은 smoke 테스트용으로 `replicas: 1`, `hpa.minReplicas: 1`, `hpa.maxReplicas: 1`을 강제한다. 따라서 `values/env/private-dev.yaml`만 수정하면 일부 서비스는 여전히 1개 Pod 기준으로 남을 수 있다.

이를 피하기 위해 HA 검증용 override를 별도 생성하고 모든 private-dev 서비스 Application이 마지막 valueFile로 참조하게 했다.

## 6. 로컬 검증 결과

수행한 확인:

```powershell
rg -n "private-dev-smoke-stable|private-dev-ha-stable|replicas: 2|minReplicas: 2|maxReplicas: 4|minAvailable: 1" argo/applications/private-dev/services values/env/private-dev.yaml values/overrides
```

확인 결과:

- `values/env/private-dev.yaml`에 `replicas: 2`, `minReplicas: 2`, `maxReplicas: 4`, `minAvailable: 1` 존재
- `values/overrides/private-dev-ha-stable.yaml`에 동일한 HA 검증 기준 존재
- private-dev 서비스 Application 8개가 `private-dev-ha-stable.yaml`을 참조
- 기존 `private-dev-smoke-stable.yaml`은 삭제하지 않고 보존

로컬 Windows 환경에는 `helm`이 PATH에 없어 `helm template` 검증은 수행하지 못했다. 서버 또는 Helm이 설치된 환경에서 ArgoCD sync 전후로 실제 렌더링/런타임 검증이 필요하다.

## 7. ArgoCD 적용 후 확인 명령어

ArgoCD sync 후 다음을 확인한다.

```bash
kubectl get deploy,hpa,pdb -A | grep -E 'ticketing-|NAME'
kubectl get pods -A | grep -E 'ticketing-|NAME'
kubectl top nodes
kubectl top pods -A --sort-by=memory | head -40
```

서비스별로 기대하는 상태:

- Deployment desired/current: 2
- HPA min/max: 2/4
- PDB minAvailable: 1
- replicas 2 상태에서 PDB allowed disruptions: 보통 1

## 8. 과제 충족 판단

이 변경만으로 과제 검증이 끝나는 것은 아니다. 이 변경은 private-dev에서 과제 검증을 할 수 있도록 기준값을 복구하는 준비 작업이다.

과제 충족 근거로 쓰려면 다음 런타임 증거가 추가로 필요하다.

- `kubectl get deploy` 결과에서 서비스별 replica 2 확인
- `kubectl get hpa` 결과에서 min 2, max 4 확인
- `kubectl get pdb` 결과에서 PDB 상태와 allowed disruptions 확인
- `kubectl top nodes/pods` 결과에서 private-dev 리소스 여유 확인
- 필요 시 부하를 주고 HPA가 2에서 3 또는 4로 확장 가능한지 확인
