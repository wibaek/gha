# Reference index

GitHub Actions와 이 저장소의 reusable workflow를 이해할 때 보는 참고 문서 목록입니다.
짧은 링크 목록만 필요하면 [README.md](README.md)를 봅니다.

## 문서 목록

- [01. GitHub Actions 기본 가이드](01_github_actions.md)
  - workflow, event, job, step, action, runner 같은 기본 용어를 설명합니다.
  - `needs`, matrix, reusable workflow, cache, secret, 권한 관리, 운영 시 자주 하는 실수를 다룹니다.
  - GitHub Actions를 처음 보거나 caller workflow를 직접 고쳐야 할 때 먼저 보면 됩니다.

- [02. Docker build cache와 고급 예시](02_github_actions_advanced.md)
  - Docker 이미지 빌드처럼 시간이 오래 걸리거나 운영 영향이 큰 workflow를 다룹니다.
  - Buildx cache backend, registry pull 기반 VPS 배포, reusable workflow 예시를 정리합니다.
  - 기본 workflow 구조를 알고 있고 Docker build/deploy 최적화를 확인할 때 봅니다.

- [03. GitHub Actions Worker 정리](03_github_actions_workers.md)
  - GitHub Actions에서 job을 실행하는 runner의 종류와 선택 기준을 설명합니다.
  - GitHub-hosted runner, larger runner, self-hosted runner, runner label, filesystem, cache, artifact, network, billing을 다룹니다.
  - `runs-on`을 어떻게 정할지, self-hosted runner를 써도 되는지 판단할 때 봅니다.

- [04. GitHub Actions workflow syntax](04_workflow_syntax.md)
  - GitHub 공식 workflow syntax 문서를 한국어로 옮긴 긴 문법 참고 문서입니다.
  - `on`, `permissions`, `env`, `defaults`, `concurrency`, `jobs`, `steps`, matrix, container, services, reusable workflow 문법을 폭넓게 다룹니다.
  - 특정 YAML keyword의 정확한 의미나 사용 예시를 확인할 때 봅니다.

- [05. GitHub Actions submodule checkout 가이드](05_github_actions_submodules.md)
  - GitHub Actions에서 Git submodule을 checkout하는 권장 패턴을 정리합니다.
  - private submodule을 기준으로 GitHub App token, PAT, SSH key, deploy key 방식을 비교합니다.
  - private repository submodule이 CI에서 실패하거나 credential 방식을 고를 때 봅니다.

- [06. GitHub Actions Environments 정리](06_github_actions_environments.md)
  - GitHub repository의 Settings > Environments에서 만드는 Environment 개념을 설명합니다.
  - 보호 규칙, environment secret과 variable, deployment URL, OIDC 조건, reusable workflow와의 관계를 다룹니다.
  - production 승인, staging/production secret 분리, 배포 UI 연결을 설계할 때 봅니다.

- [07. GitHub Actions Checks 정리](07_github_actions_checks.md)
  - PR과 commit에 붙는 status check, check run, check suite, commit status 차이를 정리합니다.
  - pending check, required status checks, workflow skip과 job skip, merge queue, aggregate required check 패턴을 다룹니다.
  - branch protection이나 merge queue에서 required check가 예상대로 동작하지 않을 때 봅니다.

- [08. GitHub Releases 정리](08_github_releases.md)
  - GitHub repository의 Releases 기능과 tag, release note, asset, draft, prerelease, latest 표시를 설명합니다.
  - release와 git tag의 차이, 자동 생성 release note, asset 업로드, 보안 release, Actions trigger 연결 시 주의점을 다룹니다.
  - GitHub에서 버전 배포 페이지를 만들고 관리하는 기능 자체를 이해할 때 봅니다.

- [09. GitHub Actions Packages 정리](09_github_actions_packages.md)
  - GitHub Packages와 GHCR을 GitHub Actions에서 사용할 때의 권한, visibility, package access를 정리합니다.
  - `packages: write`, `packages: read`, `GITHUB_TOKEN`, `GHCR_TOKEN`, repository-package 연결, digest 기반 배포를 다룹니다.
  - GHCR push/pull 권한 오류나 private image 배포 접근 제어를 판단할 때 봅니다.

- [10. GitHub Actions Permissions 정리](10_github_actions_permissions.md)
  - GitHub Actions workflow의 `permissions` 설정과 자동 발급되는 `GITHUB_TOKEN` 권한 모델을 정리합니다.
  - workflow/job-level 권한, 권한 계산 순서, fork/Dependabot 제한, 자주 쓰는 권한 조합을 다룹니다.
  - release 생성, GHCR push/pull, OIDC 배포, GitHub Deployment API 작업에 어떤 권한이 필요한지 확인할 때 봅니다.
