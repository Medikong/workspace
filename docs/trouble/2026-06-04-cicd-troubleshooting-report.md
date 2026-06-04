# Image Publish CI/CD 트러블슈팅 보고서

## 개요

GitHub Actions `image-publish.yml` 워크플로우가 ECR에 이미지를 push하지 못하는 문제가 반복 발생했다.
총 3가지 원인이 순차적으로 발견되었으며, 각 원인을 해결한 후 최종적으로 Image Publish #11이 성공했다.

---

## 문제 1 — 로컬 레지스트리로 push되는 문제

### 증상
이미지가 ECR에 저장되지 않고 GitHub Actions 내부 임시 레지스트리로만 push됨

### 원인
```yaml
# 기존 설정
services:
  registry:
    image: registry:2       ← GitHub Actions 내부에 임시 레지스트리 띄움
    ports:
      - 5000:5000

env:
  IMAGE_REGISTRY: localhost:5000  ← ECR이 아닌 로컬 주소
```

기존 코드가 로컬 Vagrant 환경 기준으로 작성되어 있어 ECR 연결이 없었다.

### 해결
```yaml
# 수정 후
env:
  IMAGE_REGISTRY: 941141115079.dkr.ecr.ap-northeast-2.amazonaws.com

# ECR 로그인 step 추가
- name: ECR 로그인
  uses: aws-actions/amazon-ecr-login@v2
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    AWS_REGION: ap-northeast-2
```

로컬 레지스트리 서비스 및 준비 확인 step을 제거하고 ECR URL과 로그인 step을 추가했다.

---

## 문제 2 — auth-service Docker 빌드 컨텍스트 오류

### 증상
```
ERROR: "/services/auth-service/app": not found
ERROR: "/packages/server": not found
```

### 원인
`Taskfile.yml`의 `case` 문에서 `auth-service`가 `*` 케이스로 빠져 빌드 컨텍스트가 잘못 설정됨

```bash
# 기존 Taskfile.yml
case "${service}" in
  concert-service|reservation-service|payment-service|ticket-service|notification-service)
    docker build -f "services/${service}/Dockerfile" -t "..." .  ← repo 루트 기준
    ;;
  *)  ← auth-service가 여기로 빠짐
    docker build -t "..." "services/${service}"  ← services/auth-service/ 기준
    ;;
esac
```

`auth-service`가 `*` 케이스로 처리되어 빌드 컨텍스트가 `services/auth-service/`로 잡히면서
상위 디렉토리인 `packages/server`를 찾지 못했다.

### 해결
```bash
# 수정 후 Taskfile.yml
case "${service}" in
  auth-service|concert-service|reservation-service|payment-service|ticket-service|notification-service)
    docker build -f "services/${service}/Dockerfile" -t "..." .  ← 모두 repo 루트 기준
    ;;
```

`auth-service`를 다른 서비스들과 같은 케이스에 추가하여 빌드 컨텍스트를 repo 루트로 통일했다.

---

## 문제 3 — ECR 저장소 이름 불일치

### 증상
```
name unknown: The repository with name 'auth-service' does not exist in the registry with id '941141115079'
```

### 원인
Terraform으로 생성된 ECR 저장소 이름과 GitHub Actions에서 push하는 이름이 달랐다.

```
ECR 실제 이름: medikong-default-auth-service
push 시도 이름: auth-service
```

Terraform `main.tf`에서 ECR 이름에 prefix가 붙어있었다.

```hcl
# 기존
name = "${local.name_prefix}-${each.key}"
# → medikong-default-auth-service
```

### 해결
```hcl
# 수정 후
name = "${each.key}"
# → auth-service
```

Terraform으로 기존 ECR 6개를 삭제하고 prefix 없는 이름으로 재생성했다.

```
auth-service
concert-service
reservation-service
payment-service
ticket-service
notification-service
```

---

## 최종 결과

Image Publish #11 성공 (1분 31초 소요)

```
✅ 이미지 선택
✅ 이미지 publish (auth-service)
✅ 이미지 publish (concert-service)
✅ 이미지 publish (reservation-service)
✅ 이미지 publish (payment-service)
✅ 이미지 publish (ticket-service)
✅ 이미지 publish (notification-service)
✅ deploy plan 조립 및 gitops 업데이트
```

6개 서비스 이미지가 ECR에 정상 push되었으며 gitops repo의 이미지 태그도 자동 업데이트되었다.

---

## 수정된 파일 목록

| 파일 | 수정 내용 |
|------|----------|
| `.github/workflows/image-publish.yml` | ECR URL 변경, ECR 로그인 추가, 로컬 레지스트리 제거, gitops 업데이트 추가 |
| `Taskfile.yml` | auth-service 빌드 컨텍스트 수정 |
| `terraform/main.tf` | ECR 저장소 이름 prefix 제거 |
