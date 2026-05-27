# Medikong project plan

이 폴더는 Medikong 프로젝트의 일정 계획, workplan, 참고 자료를 한곳에 묶는 프로젝트 계획 공간이다.

## 읽는 순서

1. `plan/00-PRD.md`
2. `plan/01-PRD_TRACEABILITY.md`
3. `plan/02-PROJECT_PLAN.md`
4. `plan/03-MILESTONES.md`
5. `plan/04-SCHEDULE.md`
6. `plan/05-scenarios/README.md`
7. `workplans/README.md`
8. `workplans/EPICS.md`
9. `workplans/*.yaml`
10. `reference/*.md`

## 폴더 역할

| 경로 | 역할 |
| --- | --- |
| `plan/` | PRD, 일정, 마일스톤, 검증 시나리오처럼 프로젝트 방향과 계획을 둔다. |
| `workplans/` | Epic과 실제 이슈 후보 YAML을 둔다. YAML은 작업 계획의 편집 원본이다. |
| `reference/` | AWS 산정, 조사, 참고표처럼 계획을 보조하는 문서를 둔다. |
| `scripts/` | 문서 보조 스크립트를 둔다. |
| `.generated/` | 스크립트가 만든 읽기 전용 산출물을 둔다. |

## 운영 원칙

- 프로젝트 전체 계획은 `plan/`에서 시작한다.
- 실제 작업 단위와 의존관계는 `workplans/*.yaml`에서 수정한다.
- 생성된 그래프 HTML은 직접 수정하지 않고, YAML을 고친 뒤 다시 생성한다.
- workspace 수준의 ADR, repo 경계, 온보딩 문서는 `docs/adr`, `docs/architecture`, `docs/onboarding`에 둔다.
- 각 repo의 구현, 배포, 테스트 상세 명령은 `service`, `gitops`, `infra` repo 문서를 기준으로 한다.
