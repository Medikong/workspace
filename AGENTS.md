# AGENTS.md

이 repo는 Medikong polyrepo의 보조 진입점이다. `workspace`는 monorepo가 아니며, `service`, `gitops`, `infra`를 내부에 포함하거나 대신 관리하지 않는다. 공통 문서, 온보딩, `repos.env`, `task help/list/doctor/bootstrap/status`, VS Code workspace 같은 빠른 작업공간 구성만 담당한다.

VS Code 기준 폴더 구조는 `medikong.code-workspace`를 따른다. `medikong/` 자체는 git repo가 아니며, Windows에서는 VS Code 기본 터미널을 Git Bash로 쓰는 것을 기준으로 한다.

- `medikong/workspace`: 현재 repo
- `medikong/service`: `../service`
- `medikong/gitops`: `../gitops`
- `medikong/infra`: `../infra`
