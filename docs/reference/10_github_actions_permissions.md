# GitHub Actions Permissions 정리

이 문서는 GitHub Actions workflow의 `permissions` 설정을 정리합니다.
`permissions`는 workflow run마다 자동 발급되는 `GITHUB_TOKEN`이 repository에서 무엇을 할 수 있는지 정합니다.

## 한 줄 요약

`GITHUB_TOKEN`은 GitHub Actions가 자동으로 만들어 주는 임시 token입니다.
`permissions`는 그 token의 권한을 job에 필요한 만큼만 열어 주는 설정입니다.

```yaml
permissions:
  contents: read
```

대부분의 CI는 위 설정으로 충분합니다.
release 생성, package push, deployment 생성, OIDC 로그인처럼 GitHub API에 쓰기 작업이 필요한 job에만 권한을 추가합니다.

## `permissions`가 하는 일

workflow의 step은 `github.token` context나 `secrets.GITHUB_TOKEN`을 통해 `GITHUB_TOKEN`을 사용할 수 있습니다.
예를 들어 `gh` CLI, GitHub API 호출, `actions/checkout`, GitHub Packages push/pull 같은 작업이 이 token을 씁니다.

`permissions`는 이 token에 부여할 repository permission을 지정합니다.
사용자 계정 권한을 바꾸는 설정이 아니고, repository settings나 organization permission을 대체하는 설정도 아닙니다.

```yaml
permissions:
  contents: read
  packages: write
```

위 설정은 workflow token이 repository contents는 읽고, GitHub Packages에는 write할 수 있게 합니다.

## 위치

`permissions`는 workflow 전체 또는 특정 job에 둘 수 있습니다.

workflow 최상단에 두면 모든 job의 기본값이 됩니다.

```yaml
name: CI

on:
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm test
```

job 안에 두면 그 job에만 적용됩니다.

```yaml
jobs:
  docker:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v6
      - run: docker build .
```

권한이 다른 job은 job별로 분리하는 편이 좋습니다.
예를 들어 CI job은 `contents: read`, Docker push job은 `packages: write`, release job은 `contents: write`로 나눕니다.

## 중요한 규칙

`permissions`에서 일부 권한을 명시하면, 명시하지 않은 권한은 `none`이 됩니다.

```yaml
permissions:
  contents: read
```

위 설정은 `contents`만 read이고, `packages`, `issues`, `pull-requests`, `deployments` 같은 다른 권한은 없습니다.
그래서 필요한 권한은 모두 같이 적어야 합니다.

```yaml
permissions:
  contents: read
  packages: write
```

전체 read 또는 전체 write도 가능하지만 보통 권장하지 않습니다.

```yaml
permissions: read-all
permissions: write-all
```

모든 권한을 끄려면 빈 객체를 씁니다.

```yaml
permissions: {}
```

단, 이러면 private repository에서 `actions/checkout` 같은 기본 작업도 실패할 수 있습니다.

## 권한 계산 순서

실제 job의 `GITHUB_TOKEN` 권한은 아래 순서로 결정됩니다.

1. enterprise, organization, repository의 기본 Actions 권한 설정이 먼저 적용됩니다.
2. workflow 최상단 `permissions`가 적용됩니다.
3. job-level `jobs.<job_id>.permissions`가 적용됩니다.
4. fork pull request나 Dependabot pull request 같은 제한 조건이 마지막에 적용됩니다.

fork에서 들어온 pull request는 보통 write 권한을 받을 수 없습니다.
관리자가 별도로 허용하지 않는 한 write 권한은 read-only로 낮아집니다.
Dependabot pull request도 fork PR처럼 read-only token으로 실행되고, 일반 repository secret에 접근하지 못합니다.

## 자주 쓰는 권한

| 권한 | 주 용도 |
| --- | --- |
| `contents: read` | checkout, commit 목록 읽기, 일반 CI |
| `contents: write` | release 생성, tag/ref 조작, repository contents 쓰기 |
| `packages: read` | GHCR private image pull |
| `packages: write` | GHCR image push, GitHub Packages publish |
| `id-token: write` | OIDC token 발급. AWS/GCP/Azure role assume |
| `deployments: write` | GitHub Deployment 생성 또는 상태 갱신 |
| `pull-requests: write` | PR 본문, review, label 등 PR 관련 API 작업 |
| `issues: write` | issue comment, issue label, issue 상태 변경 |
| `checks: write` | check run 생성 또는 갱신 |
| `statuses: write` | commit status 생성 또는 갱신 |
| `pages: write` | GitHub Pages 배포 요청 |
| `attestations: write` | artifact attestation 생성 |
| `security-events: write` | code scanning 결과 업로드 |
| `actions: write` | workflow run 취소 등 Actions 자체 조작 |

`write`는 해당 scope의 `read`를 포함합니다.
`id-token`은 `write`와 `none`만 사용합니다.

## 기본 패턴

### CI

```yaml
permissions:
  contents: read
```

lint, test, build처럼 repository를 읽기만 하는 job은 이 정도로 둡니다.

### GHCR image push

```yaml
permissions:
  contents: read
  packages: write
```

Dockerfile과 source를 읽고 GHCR에 image를 push합니다.

### GHCR private image pull

```yaml
permissions:
  contents: read
  packages: read
```

private package pull에는 `contents: read`만으로는 부족합니다.
`packages: read`가 필요합니다.

### GitHub Release 생성

```yaml
permissions:
  contents: write
```

GitHub Release 생성은 repository contents write 작업으로 봅니다.
`gh release create`를 쓰면 `GH_TOKEN`에 자동 발급된 token을 넘깁니다.

```yaml
steps:
  - env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: gh release create "$GITHUB_REF_NAME" --verify-tag --generate-notes
```

### Cloud OIDC 배포

```yaml
permissions:
  contents: read
  id-token: write
```

AWS, GCP, Azure 같은 cloud provider에 long-lived secret 없이 로그인할 때 씁니다.
`id-token: write`는 OIDC token 발급 권한이지 repository write 권한이 아닙니다.

### GitHub Deployment API 사용

```yaml
permissions:
  contents: read
  deployments: write
```

GitHub Deployment object나 deployment status를 직접 만들거나 갱신할 때 씁니다.
단순히 외부 CLI로 배포만 하고 GitHub Deployment API를 쓰지 않으면 필요하지 않을 수 있습니다.

## Reusable workflow

Reusable workflow를 호출할 때도 caller job에 필요한 권한을 명시합니다.

```yaml
jobs:
  docker:
    uses: wibaek/gha/.github/workflows/docker-build-ghcr-push.yaml@v1.0
    permissions:
      contents: read
      packages: write
```

called workflow 내부 step은 caller repository의 workflow run에서 발급된 token을 사용합니다.
따라서 caller가 어떤 job을 호출하는지 보고 필요한 권한을 적어 두는 편이 읽기 쉽습니다.

이 repository의 reusable workflow도 caller 예시에서 권한을 명시합니다.
예를 들어 GHCR build는 `packages: write`, GHCR pull deploy는 `packages: read`, ECS OIDC 배포는 `id-token: write`를 둡니다.

## `GITHUB_TOKEN`과 다른 token

`GITHUB_TOKEN`은 workflow run마다 자동으로 생기는 임시 token입니다.
사용자가 repository secret으로 직접 만들 필요는 없습니다.

하지만 몇 가지 한계가 있습니다.

- 현재 workflow repository 기준 token입니다.
- 다른 repository의 private resource에 항상 접근할 수 있는 organization-wide token이 아닙니다.
- `GITHUB_TOKEN`으로 발생시킨 대부분의 이벤트는 새 workflow run을 다시 만들지 않습니다.
- `workflow_dispatch`, `repository_dispatch`는 예외적으로 workflow run을 만들 수 있습니다.

workflow 안에서 만든 release나 tag가 다른 workflow를 반드시 trigger해야 한다면 GitHub App installation token이나 PAT를 별도로 검토합니다.
일반적인 release 생성 자체는 `GITHUB_TOKEN`과 `contents: write`로 충분합니다.

## 흔한 실수

`permissions`를 아예 생략하면 repository나 organization 기본값에 기대게 됩니다.
기본값이 바뀌면 workflow 동작도 달라질 수 있으므로 명시하는 편이 낫습니다.

모든 job에 `write-all`을 주면 불필요하게 공격면이 넓어집니다.
쓰기 권한이 필요한 job만 따로 분리합니다.

`contents: write`를 주면 GitHub Packages push가 되는 것은 아닙니다.
GHCR push에는 `packages: write`가 필요합니다.

`packages: write`를 주면 release 생성이 되는 것은 아닙니다.
GitHub Release 생성에는 `contents: write`가 필요합니다.

`id-token: write`는 cloud OIDC를 위한 권한입니다.
repository 파일을 쓸 수 있게 만드는 권한이 아닙니다.

fork PR에서 write permission이 그대로 들어온다고 가정하면 안 됩니다.
외부 contributor PR에서는 read-only 기준으로 설계합니다.

## 점검 체크리스트

- workflow나 job에 `permissions`가 명시되어 있는가?
- CI job은 `contents: read`만으로 충분한가?
- write 권한이 필요한 job만 별도 job으로 분리되어 있는가?
- release 생성 job에 `contents: write`가 있는가?
- GHCR push job에 `packages: write`가 있는가?
- GHCR pull job에 `packages: read`가 있는가?
- cloud OIDC job에 `id-token: write`가 있는가?
- GitHub Deployment API를 직접 쓰는 job에만 `deployments: write`가 있는가?
- fork PR이나 Dependabot PR에서 write 권한과 secret을 기대하지 않는가?
- `GITHUB_TOKEN`이 만든 이벤트로 다른 workflow가 자동 실행된다고 가정하지 않는가?

## 참고 문서

- [Workflow syntax: permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#permissions)
- [Workflow syntax: jobs.<job_id>.permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idpermissions)
- [Use GITHUB_TOKEN for authentication in workflows](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
- [Triggering a workflow from a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)
- [Secure use reference](https://docs.github.com/en/actions/reference/security/secure-use)
