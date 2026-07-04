# GitHub Actions submodule checkout 가이드

이 문서는 GitHub Actions에서 Git submodule을 checkout할 때의 권장 패턴을 정리합니다.
특히 private repository를 submodule로 사용하는 경우를 기준으로 봅니다.

## 한 줄 요약

프로덕션 기준으로는 GitHub App installation token을 만들어 main repository와 private submodule repository에 `Contents: read`만 부여한 뒤, `actions/checkout`에서 `submodules: recursive`로 checkout하는 방식을 우선합니다.

`GITHUB_TOKEN`은 기본적으로 workflow가 실행되는 repository에 scoped 되어 있으므로, 다른 private repository에 있는 submodule을 읽는 용도로는 부족할 수 있습니다.

## 선택 기준

| 상황 | 추천 방식 | 비고 |
| --- | --- | --- |
| public submodule만 있음 | `GITHUB_TOKEN` + `submodules: recursive` | 가장 단순함 |
| 같은 org/team의 private submodule 여러 개 | GitHub App installation token | 가장 추천. repo 단위 scope와 short-lived token을 쓸 수 있음 |
| 빠르게 붙여야 함 | fine-grained PAT 또는 machine user PAT | 단순하지만 장기 credential 관리가 필요함 |
| private submodule 1개, SSH 고정 | Deploy key | read-only 가능. 단, repo 1개당 key 1개 |
| private submodule 여러 개, deploy key 사용 | 가능하지만 비추천 | key, secret, SSH alias 관리가 복잡해짐 |

## 추천: GitHub App token

전제는 다음과 같습니다.

- GitHub App을 생성합니다.
- App을 main repository와 private submodule repository들에 install합니다.
- Repository permissions에서 `Contents`를 read-only로 둡니다.
- `APP_CLIENT_ID`는 repository 또는 organization variable로 저장합니다.
- `APP_PRIVATE_KEY`는 repository 또는 organization secret으로 저장합니다.

```yaml
name: CI

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: GitHub App token 생성
        id: app-token
        uses: actions/create-github-app-token@v3
        with:
          client-id: ${{ vars.APP_CLIENT_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ github.repository_owner }}
          repositories: |
            main-repo
            private-submodule-a
            private-submodule-b
          permission-contents: read

      - name: submodule 포함 checkout
        uses: actions/checkout@v6
        with:
          token: ${{ steps.app-token.outputs.token }}
          submodules: recursive
          fetch-depth: 1
          persist-credentials: false

      - name: submodule 확인
        run: git submodule status --recursive
```

`repositories`에는 token이 접근할 repository만 명시합니다.
private submodule이 nested submodule을 가지고 있으면 nested repository도 포함합니다.

`persist-credentials: false`는 checkout 이후 local git credential을 남기지 않기 위한 기본 보안값으로 권장합니다.
checkout 이후에 `git fetch`, `git submodule update`, `git push` 같은 인증 git 명령을 다시 실행해야 한다면 `persist-credentials: true`를 쓰거나 별도 credential 설정이 필요합니다.

## 간단한 fallback: PAT

GitHub App 설정이 부담되면 사람 계정 PAT보다 machine user 또는 bot 계정의 fine-grained PAT를 우선합니다.
토큰에는 필요한 repository에 대한 `Contents: read`만 부여합니다.

```yaml
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v6
        with:
          token: ${{ secrets.CI_READONLY_PAT }}
          submodules: recursive
          fetch-depth: 1
          persist-credentials: false
```

PAT 접근 대상에는 다음 repository가 모두 포함되어야 합니다.

- workflow가 실행되는 main repository
- 모든 private submodule repository
- nested submodule이 있다면 nested repository

PAT는 장기 credential이므로 rotation, 소유자 계정, 퇴사자 처리, 감사 추적을 같이 고려해야 합니다.

## SSH key 방식

하나의 SSH identity가 main repository와 submodule repository를 모두 읽을 수 있으면 `ssh-key`를 사용할 수 있습니다.

```yaml
steps:
  - uses: actions/checkout@v6
    with:
      ssh-key: ${{ secrets.CI_SSH_KEY }}
      submodules: recursive
      fetch-depth: 1
      persist-credentials: false
```

SSH 기반 checkout을 유지하려면 `ssh-key`를 명시하는 편이 안전합니다.
`ssh-key`가 없으면 checkout 과정에서 SSH URL이 HTTPS로 변환될 수 있습니다.

## Deploy key 방식

Deploy key는 repository 하나에 직접 붙는 SSH key입니다.
private submodule이 하나뿐이고 read-only 접근만 필요하다면 단순하게 쓸 수 있습니다.

private submodule이 여러 개면 deploy key 방식은 보통 비추천합니다.
같은 deploy key를 여러 repository에 재사용할 수 없으므로 submodule repository마다 별도 key와 SSH alias를 관리해야 합니다.

```yaml
steps:
  - name: main repository만 checkout
    uses: actions/checkout@v6
    with:
      submodules: false
      persist-credentials: false

  - name: private submodule SSH 설정
    env:
      SUBMODULE_A_KEY: ${{ secrets.SUBMODULE_A_DEPLOY_KEY }}
    run: |
      set -euo pipefail

      mkdir -p ~/.ssh
      chmod 700 ~/.ssh

      printf '%s\n' "$SUBMODULE_A_KEY" > ~/.ssh/submodule_a
      chmod 600 ~/.ssh/submodule_a

      ssh-keyscan github.com >> ~/.ssh/known_hosts

      cat >> ~/.ssh/config <<'EOF'
      Host github.com-submodule-a
        HostName github.com
        User git
        IdentityFile ~/.ssh/submodule_a
        IdentitiesOnly yes
      EOF

      git submodule set-url libs/foo git@github.com-submodule-a:my-org/private-submodule-a.git
      git submodule update --init --recursive libs/foo
```

이 방식은 submodule 수가 늘수록 secret, key rotation, SSH alias 관리가 빠르게 복잡해집니다.
private submodule이 2개 이상이면 GitHub App token 쪽을 먼저 검토합니다.

## `.gitmodules` 권장

HTTPS token 방식이면 `.gitmodules`는 HTTPS URL로 두는 편이 단순합니다.

```gitconfig
[submodule "libs/foo"]
  path = libs/foo
  url = https://github.com/my-org/private-submodule-a.git
```

SSH 중심 조직이면 SSH URL을 사용할 수 있습니다.

```gitconfig
[submodule "libs/foo"]
  path = libs/foo
  url = git@github.com:my-org/private-submodule-a.git
```

CI와 local 개발 환경 모두에서 예측 가능하게 운영하려면 한 가지 URL 형태로 통일합니다.
token 기반 checkout을 기본으로 삼는다면 HTTPS로 통일하는 쪽이 보통 더 단순합니다.

## Fork PR 주의

Fork에서 들어온 pull request workflow에는 repository secret이 전달되지 않을 수 있습니다.
또한 `GITHUB_TOKEN` 권한도 제한됩니다.

private submodule이 필요한 job은 다음 중 하나로 제한하는 편이 안전합니다.

- trusted branch의 `push`
- 같은 repository 내부 pull request
- `workflow_dispatch`
- environment approval을 거친 배포 job

외부 fork PR에서 private submodule이 꼭 필요하다면 별도 threat model을 먼저 정리합니다.

## 재현성 기준

CI에서는 `git submodule update --remote`를 기본값처럼 쓰지 않습니다.
submodule은 main repository에 기록된 commit SHA에 pinning된 상태로 checkout합니다.

CI가 submodule branch의 최신 head를 따라가면 같은 commit의 workflow가 날짜에 따라 다르게 동작할 수 있습니다.
submodule update는 별도 PR에서 commit SHA 변경으로 명시합니다.

## 보안 체크리스트

- workflow 상단에 `permissions: contents: read`를 명시합니다.
- submodule 접근용 credential은 read-only로 둡니다.
- 빌드와 테스트에 write 권한 PAT를 쓰지 않습니다.
- private submodule이 여러 개면 GitHub App token을 우선 검토합니다.
- checkout 이후 인증 git 작업이 필요 없으면 `persist-credentials: false`를 사용합니다.
- submodule은 commit SHA에 pinning하고 CI에서 임의로 최신 branch head를 따라가지 않습니다.
- fork PR에서 private submodule checkout을 실행할지 별도로 판단합니다.

## 기본 추천 조합

대부분의 사내 private repository 구조라면 아래 조합을 기본값으로 둡니다.

```text
actions/create-github-app-token@v3
+ actions/checkout@v6
+ submodules: recursive
+ permission-contents: read
+ persist-credentials: false
```

이 방식은 PAT보다 권한 범위와 감사 추적이 좋고, deploy key보다 여러 private submodule 관리가 단순합니다.

## 참고 문서

- [actions/checkout README](https://github.com/actions/checkout/blob/main/README.md)
- [actions/create-github-app-token README](https://github.com/actions/create-github-app-token)
- [GitHub Docs: GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
- [GitHub Docs: Use GITHUB_TOKEN in workflows](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
- [GitHub Docs: Managing deploy keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys)
- [GitHub Docs: Events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
