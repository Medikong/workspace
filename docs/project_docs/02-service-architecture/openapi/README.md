# OpenAPI 규약 샘플

이 폴더는 팀 논의용 OpenAPI 샘플이다. 아직 확정 규약이 아니며, 서비스 구현 전에 공통 규칙과 파일 구조를 맞추기 위한 초안으로 사용한다.

## 문서 구성

- [common-conventions.md](./common-conventions.md): 서비스 공통 OpenAPI 작성 규약 샘플
- [common/components.yaml](./common/components.yaml): 인증, 공통 헤더, 페이지네이션, 오류 응답 컴포넌트 샘플
- [services/reservation-service/openapi.yaml](./services/reservation-service/openapi.yaml): `reservation-service` OpenAPI 분리 구조 샘플
- [services/reservation-service/paths/](./services/reservation-service/paths/): path 단위 분리 파일 샘플

## 권장 구조

```text
openapi/
  common/
    components.yaml
  services/
    reservation-service/
      openapi.yaml
      paths/
        reservations.yaml
        reservations_me.yaml
        reservations_{id}.yaml
        reservations_{id}_cancel.yaml
        reservations_{id}_expire.yaml
```

서비스별 `openapi.yaml`은 서비스의 `info`, `servers`, `security`, `paths`를 정의하고, 공통 인증/헤더/오류 응답은 `common/components.yaml`을 참조한다.
