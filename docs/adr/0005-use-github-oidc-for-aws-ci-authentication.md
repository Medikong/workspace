---
id: ADR-0005
title: "GitHub Actions의 AWS 인증은 OIDC 기반 IAM Role Assume을 사용한다"
status: accepted
date: 2026-06-04
areas:
  - ci-cd
  - aws
  - iam
  - security
  - delivery
repos:
  - workspace
  - service
  - gitops
  - infra
decision_drivers:
  - 장기 AWS access key 공유 제거
  - 팀원별 secret 관리 부담 축소
  - CI 권한의 repo와 환경 단위 제한
  - ECR push 자동화의 보안 기준 정리
related:
  - docs/issues/2026-05-29-service-cicd-pipeline.yaml
  - docs/projects_plan/reference/AWS_INFRA_SIZING.md
  - service/.github/workflows/image-publish.yml
  - gitops/.github/workflows/observability-image-mirror.yml
links:
  - https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
  - https://github.com/aws-actions/configure-aws-credentials
  - https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html
supersedes: []
superseded_by: null
---

# ADR 0005: GitHub Actions의 AWS 인증은 OIDC 기반 IAM Role Assume을 사용한다

## 상태

Accepted

## 날짜

2026-06-04

## 배경

Medikong의 서비스 이미지 publish와 관측성 이미지 mirror workflow는 AWS ECR에 접근해야 한다. 현재 후보 흐름은 GitHub Actions에서 이미지를 빌드한 뒤 ECR에 push하고, 이후 GitOps values 또는 배포 선언이 해당 image tag/digest를 참조하는 구조다.

정적 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`를 GitHub Secrets에 등록하면 초기 설정은 빠르지만, 팀 운영에서는 다음 문제가 생긴다.

- 누가 키를 소유하고 회전할지 정해야 한다.
- repo별, 환경별 권한을 키 단위로 분리하기 어렵다.
- 키가 유출되면 만료 전까지 계속 사용할 수 있다.
- 팀원이 늘거나 권한이 바뀔 때 secret 재배포와 폐기가 번거롭다.

팀 논의 결과, 서비스 repo에 먼저 적용해보고 결과를 공유하기로 했다. AWS 쪽 초기 작업은 Terraform으로 관리하며, 이미 준비된 `terraform-admin` 사용자는 Terraform 적용 주체로만 사용한다. `terraform-admin` access key를 GitHub Actions runtime credential로 등록하지 않는다.

## 결정

GitHub Actions에서 AWS에 접근할 때 정적 AWS access key를 workflow secret으로 주입하지 않는다. 대신 GitHub OIDC token을 AWS IAM Role trust policy와 연결하고, workflow 실행 시점에 `sts:AssumeRoleWithWebIdentity`로 짧은 수명의 임시 credential을 발급받는다.

적용 우선순위는 다음과 같다.

1. `service` repo의 image publish workflow에 먼저 적용한다.
2. 적용 결과가 확인되면 `gitops` repo의 observability image mirror workflow로 확장한다.
3. AWS OIDC provider, IAM role, permission policy는 `infra` repo의 Terraform으로 관리한다.

기본 구조는 다음과 같다.

```text
GitHub Actions workflow
  -> GitHub OIDC token 요청
  -> AWS IAM Role trust policy 검증
  -> STS AssumeRoleWithWebIdentity
  -> 임시 AWS credential 발급
  -> ECR login / image push / 필요한 AWS API 호출
```

Role trust policy는 최소한 다음 조건을 포함한다.

- `token.actions.githubusercontent.com:aud`는 `sts.amazonaws.com`으로 제한한다.
- `token.actions.githubusercontent.com:sub`는 허용 repo와 branch 또는 environment로 제한한다.
- `service`와 `gitops`는 서로 다른 IAM Role을 사용한다.
- `prod` 성격의 workflow는 GitHub Environment protection rule과 함께 사용한다.

예상 Role 분리는 다음과 같다.

| Role | 허용 주체 | 주요 권한 | 비고 |
| --- | --- | --- | --- |
| `medikong-service-image-publish` | `repo:Medikong/service`의 허용 branch 또는 environment | 서비스 이미지 ECR push | 우선 적용 대상 |
| `medikong-gitops-observability-mirror` | `repo:Medikong/gitops`의 허용 branch 또는 environment | 관측성 이미지 ECR pull 확인, push, 필요 시 repository 생성 | 후속 적용 대상 |

`terraform-admin`은 이 Role과 OIDC provider를 만드는 bootstrap 권한으로만 사용한다. CI/CD workflow는 `terraform-admin`을 직접 assume하거나 해당 사용자의 access key를 secret으로 보관하지 않는다.

## 대안

| 대안 | 장점 | 단점 | 판단 |
| --- | --- | --- | --- |
| repo secret에 AWS access key 저장 | 설정이 빠르고 기존 workflow 수정이 적다. | 장기 키 공유, 회전, 폐기, 권한 분리 부담이 크다. 유출 시 위험이 오래 지속된다. | 채택하지 않음 |
| organization secret에 AWS access key 저장 | 여러 repo에 같은 값을 반복 등록하지 않아도 된다. | 장기 키 문제는 그대로 남고 repo별 최소 권한 분리가 어렵다. | 채택하지 않음 |
| GitHub OIDC + IAM Role | 장기 키를 저장하지 않고 repo/branch/environment 단위로 assume 조건을 제한할 수 있다. 권한 변경은 IAM Role과 Terraform에서 관리한다. | 초기 IAM/OIDC 구성이 필요하고 trust policy를 잘못 넓히면 위험하다. | 채택 |
| 자체 hosted runner의 instance profile 사용 | runner 인프라에 권한을 붙일 수 있다. | runner 운영 부담이 생기고 GitHub hosted runner 기준의 단순한 파이프라인과 맞지 않는다. | 현재는 채택하지 않음 |

## 결과

좋아지는 점:

- 팀원이 AWS access key와 secret access key를 공유하지 않아도 된다.
- workflow 실행마다 임시 credential을 사용하므로 장기 키 유출 위험이 줄어든다.
- `service`, `gitops`, `infra`의 책임 경계가 명확해진다.
- repo, branch, environment 단위로 AWS 접근 조건을 제한할 수 있다.
- AWS 권한 변경과 감사는 IAM Role/Policy/Terraform 변경 이력으로 추적할 수 있다.

비용:

- `infra` repo에서 OIDC provider, IAM Role, trust policy, permission policy를 먼저 준비해야 한다.
- Role trust policy가 넓어지지 않도록 repo, branch, environment 조건을 계속 검토해야 한다.
- workflow마다 `permissions: id-token: write`와 `aws-actions/configure-aws-credentials` 설정을 추가해야 한다.
- 초기 전환 기간에는 기존 secret 기반 workflow와 OIDC workflow의 차이를 팀에 설명해야 한다.

## 적용 원칙

1. GitHub Actions에는 AWS long-lived access key를 등록하지 않는다.
2. `terraform-admin`은 Terraform apply용 관리자 주체로만 사용하고 CI runtime credential로 사용하지 않는다.
3. `service` image publish와 `gitops` image mirror는 별도 Role을 사용한다.
4. ECR 권한은 필요한 repository와 작업 단위로 좁힌다.
5. `prod` 배포성 작업은 GitHub Environment protection rule을 함께 사용한다.
6. trust policy에 wildcard를 사용할 때는 repo 전체 허용이 필요한지 먼저 문서화한다.
7. Role 생성, 정책 변경, provider 등록은 `infra` Terraform 변경으로 남긴다.

## 후속 작업

| 상태 | 작업 | 담당 | 연결 문서 |
| --- | --- | --- | --- |
| todo | `infra` Terraform에 GitHub OIDC provider를 추가한다. | infra 담당 | `infra/` |
| todo | `service` image publish용 IAM Role과 ECR push policy를 추가한다. | infra 담당 | `infra/` |
| todo | `service` image publish workflow에서 정적 AWS secret env를 제거하고 OIDC assume role로 전환한다. | service 담당 | `service/.github/workflows/image-publish.yml` |
| todo | service workflow 실행 결과를 확인하고 팀에 공유한다. | service 담당 | `service/.github/workflows/image-publish.yml` |
| todo | 검증 후 `gitops` observability image mirror workflow도 별도 Role로 전환한다. | gitops 담당 | `gitops/.github/workflows/observability-image-mirror.yml` |
| todo | prod 성격의 workflow에 GitHub Environment protection rule을 연결한다. | unassigned | GitHub repository settings |
