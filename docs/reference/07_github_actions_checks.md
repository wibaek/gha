# GitHub Actions Checks 정리

이 문서는 GitHub Actions를 PR 보호 규칙이나 merge queue와 같이 사용할 때 자주 보이는 `Checks`, `status checks`, `pending` 상태를 정리합니다.
GitHub UI에서는 비슷하게 보이지만, 실제로는 commit status와 check run이 구분됩니다.

## 한 줄 요약

GitHub Actions의 workflow가 실행되면 GitHub는 PR과 commit에 check 결과를 붙입니다.
이 check는 테스트, lint, build, 배포 검증처럼 merge 전에 확인해야 하는 자동 검사 결과입니다.

```text
pull_request event
└── workflow run: CI
    ├── job: lint      -> check
    ├── job: test      -> check
    └── job: build     -> check

branch protection
└── required status checks
    └── 지정한 check가 success, skipped, neutral 중 하나여야 merge 가능
```

## 용어 구분

| 용어 | 의미 | GitHub Actions와의 관계 |
| --- | --- | --- |
| Status check | PR이나 commit에 붙는 검사 결과를 통칭하는 UI/보호 규칙 용어 | required status checks에서 요구할 수 있음 |
| Check | GitHub Apps 기반의 상세 검사 결과 | GitHub Actions workflow run이 생성하는 방식 |
| Check suite | 같은 commit SHA에 대해 한 앱이 만든 check run 묶음 | workflow 실행 단위에 가깝게 보면 됨 |
| Check run | 실제 검사 하나의 실행 결과 | Actions job 하나가 check로 보이는 경우가 많음 |
| Commit status | 외부 CI가 commit에 단순 상태를 붙이는 오래된 방식 | Actions는 commit status가 아니라 checks를 생성 |

GitHub 공식 문서 기준으로 status checks에는 두 종류가 있습니다.

- `Checks`
- `Commit statuses`

GitHub Actions는 workflow가 실행될 때 commit status가 아니라 checks를 생성합니다.
그래서 PR의 `Checks` 탭에서 job 로그, 실패한 step, annotation, rerun 버튼을 볼 수 있습니다.

## Check 이름

Branch protection이나 ruleset에서 required check를 고를 때는 GitHub UI에 보이는 check 이름을 기준으로 고릅니다.
Actions에서는 보통 job 이름이 check 이름에 반영됩니다.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm install --frozen-lockfile
      - run: pnpm test
```

위 예시에서 PR에는 `Test`라는 check가 보입니다.
`name`을 생략하면 `test` 같은 job id가 표시될 수 있습니다.
matrix를 쓰면 Node 버전이나 OS 조합이 check 이름에 붙어 여러 check로 보일 수 있습니다.

```yaml
jobs:
  test:
    name: Test node-${{ matrix.node-version }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [20, 22]
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with:
          node-version: ${{ matrix.node-version }}
      - run: pnpm test
```

이런 경우 required check 이름도 matrix별로 나뉠 수 있습니다.
보호 규칙에 등록할 check 이름은 실제 PR 화면에서 확인한 뒤 고르는 편이 안전합니다.

## 상태와 결론

Check run에는 실행 중 상태인 `status`와 완료 후 결과인 `conclusion`이 있습니다.

| 구분 | 값 | 의미 |
| --- | --- | --- |
| status | `requested` | check run이 생성됐지만 아직 queue에 들어가지 않음 |
| status | `queued` | 실행 queue에 들어감 |
| status | `in_progress` | 실행 중 |
| status | `pending` | queue 맨 앞에 있지만 concurrency 제한 때문에 아직 시작하지 못함 |
| status | `waiting` | environment protection rule 같은 배포 보호 규칙을 기다림 |
| status | `completed` | 실행이 끝났고 conclusion이 있음 |
| conclusion | `success` | 성공 |
| conclusion | `failure` | 실패 |
| conclusion | `cancelled` | 취소됨 |
| conclusion | `timed_out` | 제한 시간 초과 |
| conclusion | `skipped` | skip됨 |
| conclusion | `neutral` | 중립 결과 |
| conclusion | `action_required` | 사용자의 추가 조치가 필요함 |
| conclusion | `stale` | 너무 오래되어 GitHub가 stale로 표시함 |

Required status check에서 merge 가능한 성공 상태는 `success`, `skipped`, `neutral`입니다.
반대로 `failure`, `cancelled`, `timed_out`, 계속 남아 있는 `pending`은 merge를 막을 수 있습니다.

## Pending check가 보이는 경우

`pending`은 한 가지 상황만 뜻하지 않습니다.
실무에서는 아래 케이스를 나눠서 봐야 합니다.

### 1. 정상적인 대기 상태

workflow가 실제로 생성됐고 job이 실행 대기 중이면 잠시 pending으로 보일 수 있습니다.
대표 원인은 다음과 같습니다.

- GitHub-hosted runner가 아직 배정되지 않음
- self-hosted runner가 비어 있지 않음
- `concurrency` group 제한 때문에 앞 실행이 끝나기를 기다림
- environment protection rule 승인 대기

이 경우 Actions 탭에 workflow run이 보입니다.
run 상세 화면에서 queue, runner, environment approval, concurrency 상태를 확인합니다.

### 2. Required check가 보고되지 않아 pending으로 남는 상태

더 위험한 케이스는 workflow run 자체가 생성되지 않았는데 branch protection이 해당 check를 기다리는 경우입니다.
GitHub 공식 문서에 따르면 workflow가 아래 이유로 skip되면 그 workflow와 연결된 check가 `Pending` 상태로 남을 수 있습니다.

- `paths` 또는 `paths-ignore` 필터
- `branches` 또는 `branches-ignore` 필터
- commit message의 skip 지시어

예를 들어 `scripts/**` 변경에만 CI를 돌리도록 만들고, 이 workflow의 `build` check를 required로 걸었다고 가정합니다.

```yaml
name: ci

on:
  pull_request:
    paths:
      - "scripts/**"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm build
```

PR이 `README.md`만 바꾸면 workflow가 트리거되지 않습니다.
하지만 branch protection은 여전히 `build` check 성공을 기다릴 수 있습니다.
이때 PR에는 `Waiting for status to be reported`처럼 보이고 merge가 막힙니다.

## Workflow skip과 job skip의 차이

workflow 전체가 event/filter 단계에서 skip되는 것과, workflow 안의 job이 조건문으로 skip되는 것은 다릅니다.

| 상황 | 결과 |
| --- | --- |
| `on.pull_request.paths` 때문에 workflow 자체가 실행되지 않음 | required check가 pending으로 남을 수 있음 |
| commit message로 workflow run skip | required check가 pending으로 남을 수 있음 |
| workflow는 실행됐고 `jobs.<job_id>.if` 조건 때문에 job만 skip | skipped job은 success처럼 취급됨 |

그래서 required check로 쓸 workflow는 `paths` 필터로 workflow 전체를 skip하지 않는 편이 안전합니다.
무거운 작업만 조건부로 줄이고 싶다면 workflow는 항상 실행시키고 job이나 step 내부에서 조건을 거는 구조가 merge blocking 문제를 덜 만듭니다.

## Required check 설계 패턴

### 단순한 CI

가장 단순한 구조는 PR마다 항상 CI를 실행하고, 해당 job을 required check로 등록하는 것입니다.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

jobs:
  ci:
    name: ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
```

이 경우 branch protection에서는 `ci` check를 required로 지정합니다.

### 무거운 job은 조건부 실행

특정 파일이 바뀔 때만 무거운 검사를 돌리고 싶다면, required check 자체를 path-filtered workflow에 두는 것보다 항상 실행되는 가벼운 required job을 두는 편이 낫습니다.

```yaml
name: CI

on:
  pull_request:

jobs:
  required:
    name: required
    runs-on: ubuntu-latest
    steps:
      - run: echo "required check reported"

  docker-build:
    name: docker-build
    if: ${{ contains(github.event.pull_request.title, '[docker]') }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: docker build .
```

이 예시는 단순화를 위해 PR title 조건을 썼습니다.
실제 프로젝트에서는 변경 파일 감지 action이나 스크립트로 조건을 만들 수 있습니다.
핵심은 required로 등록한 `required` check가 항상 보고되게 하는 것입니다.

### 여러 job 결과를 하나의 required check로 합치기

여러 job을 각각 required로 걸면 matrix나 이름 변경 때 보호 규칙 관리가 번거로울 수 있습니다.
이럴 때는 마지막에 aggregate job을 두고 그 job만 required로 걸 수 있습니다.

```yaml
name: CI

on:
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  required:
    name: required
    runs-on: ubuntu-latest
    needs:
      - lint
      - test
    if: ${{ always() }}
    steps:
      - name: Check required job results
        run: |
          test "${{ needs.lint.result }}" = "success"
          test "${{ needs.test.result }}" = "success"
```

`if: ${{ always() }}`가 중요합니다.
공식 문서에서는 선행 job이 실패했을 때 dependent job이 skip되어 실패를 보고하지 않는 상황을 피하려면 `needs`와 함께 `always()`를 쓰라고 안내합니다.

## Branch protection에서 required status checks 설정 시 주의점

Required status checks는 protected branch나 ruleset에서 설정합니다.
일반적으로 `main` 또는 release branch에 걸고, PR merge 전에 필요한 check가 성공했는지 강제합니다.

주의할 점은 다음과 같습니다.

- required check는 최신 commit SHA에 대해 통과해야 합니다.
- 성공으로 인정되는 결론은 `success`, `skipped`, `neutral`입니다.
- required check로 선택하려면 해당 check가 선택한 repository에서 최근 7일 안에 성공적으로 완료된 적이 있어야 합니다.
- 같은 이름의 check와 commit status가 둘 다 있으면, 그 이름을 required로 선택했을 때 둘 다 필요할 수 있습니다.
- check 이름은 가능한 고유하게 둡니다. 여러 workflow에서 같은 job name을 쓰면 보호 규칙과 UI 해석이 헷갈릴 수 있습니다.
- branch protection에서 "Require branches to be up to date before merging"를 켜면 base branch 최신 코드와 함께 테스트된 상태가 필요합니다.

## Merge queue를 쓰는 경우

Merge queue를 사용하는 repository에서는 required check workflow에 `merge_group` event를 추가해야 합니다.
`pull_request`와 `push`만 있으면 PR에서는 check가 통과했더라도 merge queue에 들어간 merge group에서는 required check가 보고되지 않아 merge가 실패할 수 있습니다.

```yaml
on:
  pull_request:
  merge_group:
    types:
      - checks_requested
```

`merge_group`은 `pull_request`나 `push`와 별개의 event입니다.
Required check가 merge queue에서도 다시 보고되어야 하므로 CI workflow에는 함께 넣는 편이 안전합니다.

## 직접 check run을 만드는 경우

일반적인 Actions job은 GitHub가 자동으로 check를 만듭니다.
직접 Checks API를 호출해서 check run을 만들거나 수정하는 action을 작성하는 경우에는 권한을 별도로 신경 써야 합니다.

```yaml
permissions:
  contents: read
  checks: write
```

다만 대부분의 CI workflow에서는 `checks: write`를 직접 줄 필요가 없습니다.
테스트, lint, build를 `run` step으로 실행하는 일반적인 workflow는 job 결과가 자동으로 check에 반영됩니다.

## 문제 해결 체크리스트

PR에서 required check가 pending으로 남으면 아래 순서로 봅니다.

1. Actions 탭에 해당 workflow run이 생성됐는지 확인합니다.
2. workflow run이 있으면 runner queue, concurrency, environment approval, 실패한 선행 job을 확인합니다.
3. workflow run이 없으면 `paths`, `branches`, `commit message skip`, `workflow_dispatch` 전용 workflow인지 확인합니다.
4. required check 이름이 실제 PR에 뜨는 check 이름과 같은지 확인합니다.
5. merge queue를 쓰면 `merge_group` event가 있는지 확인합니다.
6. dependent aggregate job을 required로 쓴다면 `if: ${{ always() }}`로 결과를 항상 보고하는지 확인합니다.
7. 오래된 required check라면 최근 7일 안에 같은 repository에서 성공한 이력이 있는지 확인합니다.

## 공식 문서

- [GitHub Actions 이해하기](https://docs.github.com/en/actions/get-started/understand-github-actions)
- [Status checks 설명](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/about-status-checks)
- [Required status checks 문제 해결](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/collaborating-on-repositories-with-code-quality-features/troubleshooting-required-status-checks)
- [Workflow syntax](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Events that trigger workflows: merge_group](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#merge_group)
- [REST API: Check runs](https://docs.github.com/en/rest/checks/runs)
- [REST API: Commit statuses](https://docs.github.com/en/rest/commits/statuses)
