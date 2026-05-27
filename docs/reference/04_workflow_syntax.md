# GitHub Actions workflow syntax

원문: [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)

기준일: 2026-05-27

이 문서는 GitHub Actions workflow YAML 문법을 한국어로 옮긴 참고 문서입니다. 코드 예시는 원문 형태를 유지합니다.

## Workflow란

workflow는 하나 이상의 job으로 구성된 설정 가능한 자동화 프로세스입니다. workflow 설정은 YAML 파일로 정의합니다.

## Workflow YAML 문법

workflow 파일은 YAML 문법을 사용하며, 확장자는 `.yml` 또는 `.yaml`이어야 합니다.

workflow 파일은 repository의 `.github/workflows` 디렉터리에 저장해야 합니다.

## `name`

workflow의 이름입니다. GitHub는 repository의 Actions 탭에서 workflow 이름을 표시합니다. `name`을 생략하면 repository root 기준 workflow 파일 경로가 표시됩니다.

## `run-name`

workflow run의 이름입니다. GitHub는 repository Actions 탭의 workflow run 목록에서 이 이름을 표시합니다.

`run-name`을 생략하거나 공백만 넣으면 event별 기본 정보가 사용됩니다. 예를 들어 `push` 또는 `pull_request`로 실행된 workflow는 commit message나 pull request 제목이 run name이 됩니다.

`run-name`에는 expression을 사용할 수 있고, `github`, `inputs` context를 참조할 수 있습니다.

```yaml
run-name: Deploy to ${{ inputs.deploy_target }} by @${{ github.actor }}
```

## `on`

`on`은 workflow를 실행할 event를 정의합니다.

하나의 event, 여러 event, schedule을 지정할 수 있습니다. 또한 특정 file, tag, branch 변경에 대해서만 workflow가 실행되도록 제한할 수 있습니다.

### 단일 event

```yaml
on: push
```

위 workflow는 repository의 어떤 branch에든 push가 발생하면 실행됩니다.

### 여러 event

```yaml
on: [push, fork]
```

여러 event를 지정하면 그중 하나만 발생해도 workflow가 실행됩니다. 여러 triggering event가 동시에 발생하면 workflow run도 여러 개 생성됩니다.

### Activity type

일부 event는 activity type을 제공합니다. `on.<event_name>.types`로 workflow run을 trigger할 activity type을 지정합니다.

```yaml
on:
  label:
    types:
      - created
```

위 예시는 label이 생성될 때만 실행됩니다.

```yaml
on:
  issues:
    types:
      - opened
      - labeled
```

여러 activity type을 지정하면 그중 하나만 발생해도 workflow가 실행됩니다. 여러 activity type이 동시에 발생하면 workflow run도 여러 개 만들어질 수 있습니다.

### Filter

일부 event는 filter를 제공합니다. 예를 들어 `push` event의 `branches` filter는 지정한 branch pattern에 맞는 push일 때만 workflow를 실행합니다.

```yaml
on:
  push:
    branches:
      - main
      - 'releases/**'
```

### 여러 event에서 activity type과 filter 사용

event마다 설정을 분리해야 합니다. 설정이 없는 event에도 colon(`:`)을 붙입니다.

```yaml
on:
  label:
    types:
      - created
  push:
    branches:
      - main
  page_build:
```

위 workflow는 label 생성, `main` branch push, GitHub Pages branch build event에서 실행됩니다.

## `on.<event_name>.types`

workflow run을 trigger할 activity type입니다. 대부분의 GitHub event는 여러 activity type으로 trigger될 수 있습니다. `types`를 사용하면 workflow 실행 조건을 특정 activity로 좁힐 수 있습니다.

```yaml
on:
  label:
    types: [created, edited]
```

## `on.<pull_request|pull_request_target>.<branches|branches-ignore>`

`pull_request`, `pull_request_target` event에서는 특정 target branch에 대한 pull request에서만 workflow가 실행되도록 설정할 수 있습니다.

- `branches`: 포함할 branch pattern을 지정합니다. include와 exclude를 함께 표현할 때도 사용합니다.
- `branches-ignore`: 제외할 branch pattern만 지정합니다.

같은 event에서 `branches`와 `branches-ignore`를 동시에 사용할 수 없습니다.

`branches` 또는 `branches-ignore`와 `paths` 또는 `paths-ignore`를 모두 정의하면 두 filter가 모두 만족될 때만 workflow가 실행됩니다.

branch filter는 `*`, `**`, `+`, `?`, `!` 같은 glob pattern 문자를 사용할 수 있습니다. branch 이름에 이런 문자가 문자 그대로 들어 있다면 `\`로 escape합니다.

### Branch 포함

```yaml
on:
  pull_request:
    branches:
      - main
      - 'mona/octocat'
      - 'releases/**'
```

위 workflow는 `main`, `mona/octocat`, `releases/10` 같은 branch를 target으로 하는 pull request에서 실행됩니다.

branch filtering, path filtering, commit message 때문에 workflow가 skip되면 관련 check가 Pending 상태로 남을 수 있습니다. 필수 check로 설정된 경우 pull request merge가 막힐 수 있습니다.

### Branch 제외

```yaml
on:
  pull_request:
    branches-ignore:
      - 'mona/octocat'
      - 'releases/**-alpha'
```

위 workflow는 `mona/octocat`이나 `releases/beta/3-alpha` 같은 target branch에서는 실행되지 않습니다.

### Branch 포함과 제외

`branches` 안에서 `!` prefix를 사용해 exclude pattern을 표현할 수 있습니다. `!` pattern을 쓰려면 최소 하나의 positive pattern도 있어야 합니다.

pattern 순서는 중요합니다.

- positive match 뒤에 negative pattern이 match되면 제외됩니다.
- negative match 뒤에 positive pattern이 match되면 다시 포함됩니다.

```yaml
on:
  pull_request:
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.push.<branches|tags|branches-ignore|tags-ignore>`

`push` event에서는 특정 branch 또는 tag에 대해서만 workflow가 실행되도록 설정할 수 있습니다.

- `branches`: 포함할 branch pattern
- `branches-ignore`: 제외할 branch pattern
- `tags`: 포함할 tag pattern
- `tags-ignore`: 제외할 tag pattern

같은 event에서 `branches`와 `branches-ignore`를 동시에 사용할 수 없습니다. `tags`와 `tags-ignore`도 동시에 사용할 수 없습니다.

`tags` 또는 `tags-ignore`만 정의하면 branch push에서는 실행되지 않습니다. `branches` 또는 `branches-ignore`만 정의하면 tag push에서는 실행되지 않습니다. 둘 다 정의하지 않으면 branch와 tag 둘 다에서 실행됩니다.

### Branch와 tag 포함

```yaml
on:
  push:
    branches:
      - main
      - 'mona/octocat'
      - 'releases/**'
    tags:
      - v2
      - v1.*
```

위 workflow는 `main`, `mona/octocat`, `releases/10` branch push와 `v2`, `v1.9.1` 같은 tag push에서 실행됩니다.

### Branch와 tag 제외

```yaml
on:
  push:
    branches-ignore:
      - 'mona/octocat'
      - 'releases/**-alpha'
    tags-ignore:
      - v2
      - v1.*
```

### Branch와 tag 포함 및 제외

include와 exclude를 함께 쓰려면 `branches` 또는 `tags` filter에서 `!`를 사용합니다.

```yaml
on:
  push:
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.<push|pull_request|pull_request_target>.<paths|paths-ignore>`

`push`, `pull_request`, `pull_request_target` event에서는 변경된 file path를 기준으로 workflow 실행 여부를 설정할 수 있습니다. tag push에서는 path filter가 평가되지 않습니다.

- `paths`: 포함할 path pattern
- `paths-ignore`: 제외할 path pattern

같은 event에서 `paths`와 `paths-ignore`를 동시에 사용할 수 없습니다. include와 exclude를 함께 쓰려면 `paths` filter에서 `!`를 사용합니다.

pattern 순서는 중요합니다.

- positive match 뒤에 negative pattern이 match되면 path가 제외됩니다.
- negative match 뒤에 positive pattern이 match되면 path가 다시 포함됩니다.

`branches` 또는 `branches-ignore`와 `paths` 또는 `paths-ignore`를 함께 정의하면 두 filter가 모두 만족될 때만 workflow가 실행됩니다.

### Path 포함

```yaml
on:
  push:
    paths:
      - '**.js'
```

하나라도 pattern에 match되는 path가 있으면 workflow가 실행됩니다.

### Path 제외

```yaml
on:
  push:
    paths-ignore:
      - 'docs/**'
```

모든 변경 path가 `paths-ignore` pattern과 match되면 workflow가 실행되지 않습니다. 하나라도 match되지 않는 path가 있으면 workflow는 실행됩니다.

### Path 포함과 제외

```yaml
on:
  push:
    paths:
      - 'sub-project/**'
      - '!sub-project/docs/**'
```

위 workflow는 `sub-project` 아래 파일이 바뀌면 실행되지만, `sub-project/docs` 안의 파일만 바뀌면 실행되지 않습니다.

### Git diff 비교

1,000개가 넘는 commit을 push하거나 GitHub가 timeout 때문에 diff를 만들지 못하면 workflow는 항상 실행됩니다.

GitHub는 변경 파일 목록을 다음 방식으로 만듭니다.

- Pull request: three-dot diff. topic branch 최신 버전과 topic branch가 base branch와 마지막으로 동기화된 commit을 비교합니다.
- 기존 branch push: two-dot diff. head SHA와 base SHA를 직접 비교합니다.
- 새 branch push: push된 가장 깊은 commit의 ancestor parent와 two-dot diff로 비교합니다.

diff는 300개 파일로 제한됩니다. match되어야 하는 파일이 첫 300개 안에 없으면 workflow가 실행되지 않을 수 있습니다.

## `on.schedule`

`on.schedule`은 workflow 실행 시간을 정의합니다.

schedule은 POSIX cron syntax를 사용합니다. 기본적으로 UTC 기준으로 실행됩니다. IANA timezone string으로 timezone-aware scheduling을 지정할 수 있습니다. scheduled workflow는 default branch의 최신 commit에서 실행됩니다. 가장 짧은 실행 간격은 5분입니다.

```text
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of the month (1 - 31)
│ │ │ ┌───────────── month (1 - 12 or JAN-DEC)
│ │ │ │ ┌───────────── day of the week (0 - 6 or SUN-SAT)
│ │ │ │ │
* * * * *
```

| Operator | 설명 | 예시 |
| --- | --- | --- |
| `*` | 모든 값 | `15 * * * *`는 매시간 15분에 실행 |
| `,` | 값 목록 구분 | `2,10 4,5 * * *`는 매일 4시와 5시의 2분, 10분에 실행 |
| `-` | 값 범위 | `30 4-6 * * *`는 4시, 5시, 6시 30분에 실행 |
| `/` | step 값 | `20/15 * * * *`는 20분부터 15분 간격으로 실행 |

```yaml
on:
  schedule:
    - cron: '30 5 * * 1-5'
      timezone: "America/New_York"
```

하나의 workflow는 여러 schedule event로 trigger될 수 있습니다. 어떤 schedule이 trigger했는지는 `github.event.schedule` context로 확인합니다.

```yaml
on:
  schedule:
    - cron: '30 5 * * 1,3'
    - cron: '30 5,17 * * 2,4'

jobs:
  test_schedule:
    runs-on: ubuntu-latest
    steps:
      - name: Not on Monday or Wednesday
        if: github.event.schedule != '30 5 * * 1,3'
        run: echo "This step will be skipped on Monday and Wednesday"
      - name: Every time
        run: echo "This step will always run"
```

## `on.workflow_call`

`on.workflow_call`은 reusable workflow의 inputs, outputs, secrets mapping을 정의합니다.

## `on.workflow_call.inputs`

caller workflow에서 called workflow로 전달할 input을 정의합니다. `workflow_call` input은 `type` parameter가 필요합니다.

`default`가 없으면 boolean은 `false`, number는 `0`, string은 `""`가 기본값입니다.

called workflow에서는 `inputs` context로 input을 참조합니다. caller가 정의되지 않은 input을 넘기면 error가 발생합니다.

```yaml
on:
  workflow_call:
    inputs:
      username:
        description: 'A username passed from the caller workflow'
        default: 'john-doe'
        required: false
        type: string

jobs:
  print-username:
    runs-on: ubuntu-latest
    steps:
      - name: Print the input name to STDOUT
        run: echo The username is ${{ inputs.username }}
```

## `on.workflow_call.inputs.<input_id>.type`

`workflow_call` input을 정의했다면 필수입니다. 가능한 값은 `boolean`, `number`, `string`입니다.

## `on.workflow_call.outputs`

called workflow의 output map입니다. called workflow output은 caller workflow의 downstream job에서 사용할 수 있습니다. output의 `value`는 called workflow 안의 job output을 가리켜야 합니다.

```yaml
on:
  workflow_call:
    outputs:
      workflow_output1:
        description: "The first job output"
        value: ${{ jobs.my_job.outputs.job_output1 }}
      workflow_output2:
        description: "The second job output"
        value: ${{ jobs.my_job.outputs.job_output2 }}
```

## `on.workflow_call.secrets`

called workflow에서 사용할 secrets map입니다. called workflow 안에서는 `secrets` context로 secret을 참조합니다.

nested reusable workflow로 secret을 전달하려면 `jobs.<job_id>.secrets`를 다시 사용해야 합니다. caller가 정의되지 않은 secret을 넘기면 error가 발생합니다.

```yaml
on:
  workflow_call:
    secrets:
      access-token:
        description: 'A token passed from the caller workflow'
        required: false

jobs:
  pass-secret-to-action:
    runs-on: ubuntu-latest
    steps:
      - name: Pass the received secret to an action
        uses: ./.github/actions/my-action
        with:
          token: ${{ secrets.access-token }}

  pass-secret-to-workflow:
    uses: ./.github/workflows/my-workflow
    secrets:
      token: ${{ secrets.access-token }}
```

## `on.workflow_call.secrets.<secret_id>`

secret과 연결할 string identifier입니다.

## `on.workflow_call.secrets.<secret_id>.required`

secret이 반드시 제공되어야 하는지 지정하는 boolean입니다.

## `on.workflow_run.<branches|branches-ignore>`

`workflow_run` event에서는 triggering workflow가 어떤 branch에서 실행되어야 이 workflow를 trigger할지 지정합니다.

```yaml
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches:
      - 'releases/**'
```

```yaml
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches-ignore:
      - "canary"
```

`branches`와 `branches-ignore`는 함께 사용할 수 없습니다. include와 exclude를 함께 쓰려면 `branches`에서 `!`를 사용합니다.

```yaml
on:
  workflow_run:
    workflows: ["Build"]
    types: [requested]
    branches:
      - 'releases/**'
      - '!releases/**-alpha'
```

## `on.workflow_dispatch`

`workflow_dispatch` event는 workflow를 수동 실행할 때 사용합니다. input을 선택적으로 지정할 수 있습니다.

이 trigger는 workflow file이 default branch에 있을 때만 event를 받습니다.

## `on.workflow_dispatch.inputs`

수동 실행된 workflow는 `inputs` context에서 input을 받습니다. `github.event.inputs`에서도 같은 정보를 받을 수 있지만, `inputs` context는 boolean 값을 boolean으로 유지합니다. `choice` type은 string이 됩니다.

제약:

- top-level input은 최대 25개입니다.
- input payload는 최대 65,535자입니다.

```yaml
on:
  workflow_dispatch:
    inputs:
      logLevel:
        description: 'Log level'
        required: true
        default: 'warning'
        type: choice
        options:
          - info
          - warning
          - debug
      print_tags:
        description: 'True to print to STDOUT'
        required: true
        type: boolean
      tags:
        description: 'Test scenario tags'
        required: true
        type: string
      environment:
        description: 'Environment to run tests against'
        type: environment
        required: true

jobs:
  print-tag:
    runs-on: ubuntu-latest
    if: ${{ inputs.print_tags }}
    steps:
      - name: Print the input tag to STDOUT
        run: echo  The tags are ${{ inputs.tags }}
```

## `on.workflow_dispatch.inputs.<input_id>.required`

input이 필수인지 지정하는 boolean입니다.

## `on.workflow_dispatch.inputs.<input_id>.type`

input type입니다. 가능한 값은 `boolean`, `choice`, `number`, `environment`, `string`입니다.

## `permissions`

`permissions`는 `GITHUB_TOKEN`의 기본 권한을 조정합니다. 필요한 최소 권한만 허용하는 것이 원칙입니다.

workflow top-level에 둘 수도 있고 job 안에 둘 수도 있습니다. job 안에 두면 그 job에서 `GITHUB_TOKEN`을 사용하는 action과 run command가 지정한 권한을 받습니다.

`write`는 `read`를 포함합니다. 하나라도 permission을 명시하면 명시하지 않은 permission은 모두 `none`이 됩니다.

`pull_request_target` event는 public fork에서 온 경우에도 read/write repository permission을 받을 수 있어 보안상 주의해야 합니다.

| Permission | 의미 |
| --- | --- |
| `actions` | GitHub Actions 작업. 예: workflow run 취소 |
| `artifact-metadata` | artifact metadata 작업 |
| `attestations` | artifact attestation 생성 |
| `checks` | check run, check suite 작업 |
| `code-quality` | code coverage report upload 같은 code quality 작업 |
| `contents` | repository contents 작업. `contents: read`는 commit list, `contents: write`는 release 생성 가능 |
| `deployments` | deployment 생성 같은 deployment 작업 |
| `discussions` | GitHub Discussions 작업 |
| `id-token` | OIDC token 발급. `id-token: write` 필요 |
| `issues` | issue comment 작성 같은 issue 작업 |
| `models` | GitHub Models inference API 사용 |
| `packages` | GitHub Packages upload/publish 작업 |
| `pages` | GitHub Pages build 요청 같은 Pages 작업 |
| `pull-requests` | pull request label 추가 같은 PR 작업 |
| `security-events` | code scanning alert 조회/상태 업데이트 |
| `statuses` | commit status 조회/작성 |
| `vulnerability-alerts` | Dependabot alert 조회. `read`, `none`만 지원 |

```yaml
permissions:
  actions: read|write|none
  artifact-metadata: read|write|none
  attestations: read|write|none
  checks: read|write|none
  code-quality: read|write|none
  contents: read|write|none
  deployments: read|write|none
  id-token: write|none
  issues: read|write|none
  models: read|none
  discussions: read|write|none
  packages: read|write|none
  pages: read|write|none
  pull-requests: read|write|none
  security-events: read|write|none
  statuses: read|write|none
  vulnerability-alerts: read|none
```

모든 권한을 read 또는 write로 열 수도 있습니다.

```yaml
permissions: read-all
```

```yaml
permissions: write-all
```

모든 권한을 비활성화하려면 빈 map을 사용합니다.

```yaml
permissions: {}
```

fork repository에서는 일반적으로 write access를 부여할 수 없습니다. 예외는 admin이 pull request workflow에 write token 전송을 허용한 경우입니다.

## Workflow job permission 계산

`GITHUB_TOKEN` permission은 enterprise, organization, repository 기본 설정에서 시작합니다. 이후 workflow level, job level 설정 순서로 조정됩니다. fork repository에서 온 pull request event는 `pull_request_target`이 아닌 한 write permission이 read-only로 조정됩니다.

```yaml
name: "My workflow"

on: [ push ]

permissions: read-all

jobs:
  ...
```

Dependabot pull request가 trigger한 workflow run은 fork repository에서 온 것처럼 read-only `GITHUB_TOKEN`을 사용하고 secrets에 접근할 수 없습니다.

## `env`

workflow, job, step에서 사용할 environment variable map입니다. 같은 이름이 여러 위치에 있으면 가장 구체적인 값이 사용됩니다.

`env` map 안의 variable은 같은 map 안의 다른 variable을 기준으로 정의할 수 없습니다.

```yaml
env:
  SERVER: production
```

## `defaults`

workflow의 모든 job에 적용할 default setting map입니다. job level에서도 설정할 수 있고, 같은 이름이면 더 구체적인 값이 우선합니다.

## `defaults.run`

workflow의 모든 `run` step에 기본 `shell`, `working-directory`를 제공합니다. context나 expression은 사용할 수 없습니다.

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./scripts
```

## `defaults.run.shell`

step에 사용할 shell을 지정합니다.

| Platform | `shell` | 내부 실행 command |
| --- | --- | --- |
| Linux/macOS | unspecified | `bash -e {0}` |
| All | `bash` | `bash --noprofile --norc -eo pipefail {0}` |
| All | `pwsh` | `pwsh -command ". '{0}'"` |
| All | `python` | `python {0}` |
| Linux/macOS | `sh` | `sh -e {0}` |
| Windows | `cmd` | `%ComSpec% /D /E:ON /V:OFF /S /C "CALL "{0}""` |
| Windows | `pwsh` | `pwsh -command ". '{0}'"` |
| Windows | `powershell` | `powershell -command ". '{0}'"` |

## `defaults.run.working-directory`

`run` step의 작업 directory입니다. 지정한 directory는 runner에 존재해야 합니다.

## `concurrency`

같은 concurrency group을 사용하는 job 또는 workflow가 동시에 하나만 실행되도록 제한합니다.

workflow level `concurrency` expression은 `github`, `inputs`, `vars` context만 사용할 수 있습니다. job level에서는 `needs`, `strategy`, `matrix`도 사용할 수 있습니다.

동작:

- 같은 group에 running run이 있으면 새 run은 `pending`이 됩니다.
- 기본값에서는 같은 group에 pending run이 하나만 유지됩니다.
- `cancel-in-progress: true`면 running run도 취소합니다.
- `queue: max`면 최대 100개 pending run이 queue에 남을 수 있습니다.
- `queue: max`와 `cancel-in-progress: true`는 함께 사용할 수 없습니다.
- group name은 case-insensitive입니다.

```yaml
on:
  push:
    branches:
      - main

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
jobs:
  job-1:
    runs-on: ubuntu-latest
    concurrency:
      group: example-group
      cancel-in-progress: true
```

```yaml
concurrency:
  group: production-deploy
  queue: max
```

```yaml
concurrency:
  group: ${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ !contains(github.ref, 'release/')}}
```

## `jobs`

workflow run은 하나 이상의 job으로 구성됩니다. job은 기본적으로 병렬 실행됩니다. 순서가 필요하면 `jobs.<job_id>.needs`를 사용합니다.

각 job은 `runs-on`으로 지정한 runner environment에서 실행됩니다.

## `jobs.<job_id>`

job의 unique identifier입니다. letter 또는 `_`로 시작해야 하며 alphanumeric, `-`, `_`만 포함할 수 있습니다.

```yaml
jobs:
  my_first_job:
    name: My first job
  my_second_job:
    name: My second job
```

## `jobs.<job_id>.name`

GitHub UI에 표시되는 job 이름입니다.

## `jobs.<job_id>.permissions`

특정 job의 `GITHUB_TOKEN` 권한을 조정합니다.

```yaml
jobs:
  stale:
    runs-on: ubuntu-latest
    permissions:
      issues: write
      pull-requests: write
    steps:
      - uses: actions/stale@v10
```

## `jobs.<job_id>.needs`

이 job이 실행되기 전에 성공해야 하는 job을 지정합니다. string 또는 string array를 사용할 수 있습니다.

```yaml
jobs:
  job1:
  job2:
    needs: job1
  job3:
    needs: [job1, job2]
```

dependency job이 실패하거나 skip되면 이 job도 skip됩니다. 실패와 관계없이 실행하려면 `always()`를 사용합니다.

```yaml
jobs:
  job1:
  job2:
    needs: job1
  job3:
    if: ${{ always() }}
    needs: [job1, job2]
```

## `jobs.<job_id>.if`

job 실행 조건입니다. `strategy.matrix`가 적용되기 전에 평가됩니다. `if`에서는 보통 `${{ }}`를 생략할 수 있지만, expression이 `!`로 시작하면 YAML과 충돌하므로 `${{ }}`를 사용합니다.

```yaml
if: ${{ ! startsWith(github.ref, 'refs/tags/') }}
```

```yaml
name: example-workflow
on: [push]
jobs:
  production-deploy:
    if: github.repository == 'octo-org/octo-repo-prod'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '14'
      - run: npm install -g bats
```

## `jobs.<job_id>.runs-on`

job을 실행할 machine type을 정의합니다. GitHub-hosted runner, larger runner, self-hosted runner를 사용할 수 있습니다.

가능한 형태:

- 단일 string
- string variable
- string 또는 string variable의 array
- `group`, `labels` key를 쓰는 map

```yaml
runs-on: [self-hosted, linux, x64, gpu]
```

```yaml
on:
  workflow_dispatch:
    inputs:
      chosen-os:
        required: true
        type: choice
        options:
          - Ubuntu
          - macOS

jobs:
  test:
    runs-on: [self-hosted, "${{ inputs.chosen-os }}"]
    steps:
      - run: echo Hello world!
```

### GitHub-hosted runner

GitHub-hosted runner를 사용하면 각 job은 지정한 runner image의 fresh instance에서 실행됩니다.

대표 label:

| Repository type | OS | CPU | RAM | Storage | Architecture | Label |
| --- | --- | --- | --- | --- | --- | --- |
| Public | Linux slim | 1 | 5 GB | 14 GB | x64 | `ubuntu-slim` |
| Public | Linux | 4 | 16 GB | 14 GB | x64 | `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` |
| Public | Windows | 4 | 16 GB | 14 GB | x64 | `windows-latest`, `windows-2025`, `windows-2022` |
| Public | Linux | 4 | 16 GB | 14 GB | arm64 | `ubuntu-24.04-arm`, `ubuntu-22.04-arm` |
| Public | Windows | 4 | 16 GB | 14 GB | arm64 | `windows-11-arm` |
| Public | macOS Intel | 4 | 14 GB | 14 GB | Intel | `macos-15-intel`, `macos-26-intel` |
| Public | macOS arm64 | 3 | 7 GB | 14 GB | arm64 | `macos-latest`, `macos-14`, `macos-15`, `macos-26` |
| Private | Linux slim | 1 | 5 GB | 14 GB | x64 | `ubuntu-slim` |
| Private | Linux | 2 | 8 GB | 14 GB | x64 | `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` |
| Private | Windows | 2 | 8 GB | 14 GB | x64 | `windows-latest`, `windows-2025`, `windows-2022` |
| Private | Linux | 2 | 8 GB | 14 GB | arm64 | `ubuntu-24.04-arm`, `ubuntu-22.04-arm` |
| Private | Windows | 2 | 8 GB | 14 GB | arm64 | `windows-11-arm` |
| Private | macOS Intel | 4 | 14 GB | 14 GB | Intel | `macos-15-intel`, `macos-26-intel` |
| Private | macOS arm64 | 3 | 7 GB | 14 GB | arm64 | `macos-latest`, `macos-14`, `macos-15`, `macos-26` |

`-latest` image는 GitHub가 제공하는 최신 stable image라는 뜻이며, OS vendor의 최신 version이라는 뜻은 아닙니다.

```yaml
runs-on: ubuntu-latest
```

### Self-hosted runner

self-hosted runner를 사용하려면 `runs-on`에 self-hosted runner label을 지정합니다. 일반적으로 array는 `self-hosted`로 시작하고 필요한 label을 추가합니다.

```yaml
runs-on: [self-hosted, linux]
```

Actions Runner Controller는 `self-hosted` label을 지원하지 않습니다.

### Runner group

runner group을 target할 수 있고, group과 label을 함께 사용할 수도 있습니다.

```yaml
name: learn-github-actions
on: [push]
jobs:
  check-bats-version:
    runs-on:
      group: ubuntu-runners
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '14'
      - run: npm install -g bats
      - run: bats -v
```

```yaml
name: learn-github-actions
on: [push]
jobs:
  check-bats-version:
    runs-on:
      group: ubuntu-runners
      labels: ubuntu-24.04-16core
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: '14'
      - run: npm install -g bats
      - run: bats -v
```

## `jobs.<job_id>.snapshot`

custom image를 생성할 때 사용합니다. `snapshot` keyword가 있는 job이 성공할 때마다 image version이 생성됩니다. 하나의 image만 만들려면 모든 step을 하나의 job에 넣습니다.

## `jobs.<job_id>.environment`

job이 참조할 GitHub Environment를 정의합니다. environment는 name만 지정하거나 `name`, `url` object로 지정할 수 있습니다.

environment를 참조하는 job은 deployment protection rule을 통과해야 runner로 전송됩니다.

```yaml
environment: staging_environment
```

```yaml
environment:
  name: production_environment
  url: https://github.com
```

```yaml
environment:
  name: production_environment
  url: ${{ steps.step_id.outputs.url_output }}
```

```yaml
environment:
  name: ${{ github.ref_name }}
```

deployment object를 만들지 않고 environment secrets와 variables만 쓰려면 `deployment: false`를 설정합니다.

```yaml
environment:
  name: testing
  deployment: false
```

`deployment: false`는 custom deployment protection rule과 호환되지 않습니다.

## `jobs.<job_id>.concurrency`

job level concurrency입니다. workflow level concurrency와 같은 개념이지만, job level에서는 `needs`, `strategy`, `matrix` context도 사용할 수 있습니다.

## `jobs.<job_id>.outputs`

job output map입니다. downstream job은 `needs` context로 output을 사용할 수 있습니다.

job output은 job당 최대 1 MB, workflow run 전체에서 최대 50 MB입니다. secret으로 보이는 output은 redacted되어 GitHub Actions로 전송되지 않을 수 있습니다.

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      output1: ${{ steps.step1.outputs.test }}
      output2: ${{ steps.step2.outputs.test }}
    steps:
      - id: step1
        run: echo "test=hello" >> "$GITHUB_OUTPUT"
      - id: step2
        run: echo "test=world" >> "$GITHUB_OUTPUT"
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - env:
          OUTPUT1: ${{needs.job1.outputs.output1}}
          OUTPUT2: ${{needs.job1.outputs.output2}}
        run: echo "$OUTPUT1 $OUTPUT2"
```

matrix job output은 matrix 안의 모든 job output이 결합됩니다. matrix job 실행 순서는 보장되지 않으므로 output name은 unique해야 합니다.

## `jobs.<job_id>.env`

job의 모든 step에서 사용할 environment variable map입니다.

```yaml
jobs:
  job1:
    env:
      FIRST_NAME: Mona
```

## `jobs.<job_id>.defaults`

job의 모든 step에 적용할 default setting map입니다.

## `jobs.<job_id>.defaults.run`

job의 모든 `run` step에 기본 `shell`, `working-directory`를 제공합니다.

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    defaults:
      run:
        shell: bash
        working-directory: ./scripts
```

## `jobs.<job_id>.defaults.run.shell`

job의 `run` step 기본 shell입니다.

## `jobs.<job_id>.defaults.run.working-directory`

job의 `run` step 기본 working directory입니다. directory는 runner에 존재해야 합니다.

## `jobs.<job_id>.steps`

job은 `steps`라는 task sequence를 포함합니다. step은 command를 실행하거나 setup task를 수행하거나 action을 실행합니다.

각 step은 runner environment 안에서 별도 process로 실행됩니다. 따라서 environment variable 변경은 다음 step에 자동 보존되지 않습니다.

```yaml
name: Greeting from Mona

on: push

jobs:
  my-job:
    name: My Job
    runs-on: ubuntu-latest
    steps:
      - name: Print a greeting
        env:
          MY_VAR: Hi there! My name is
          FIRST_NAME: Mona
          MIDDLE_NAME: The
          LAST_NAME: Octocat
        run: |
          echo $MY_VAR $FIRST_NAME $MIDDLE_NAME $LAST_NAME.
```

## `jobs.<job_id>.steps[*].id`

step의 unique identifier입니다. context에서 step을 참조할 때 사용합니다.

## `jobs.<job_id>.steps[*].if`

step 실행 조건입니다. expression이 `!`로 시작하면 `${{ }}`를 사용하거나 escape해야 합니다.

```yaml
if: ${{ ! startsWith(github.ref, 'refs/tags/') }}
```

```yaml
steps:
  - name: My first step
    if: ${{ github.event_name == 'pull_request' && github.event.action == 'unassigned' }}
    run: echo This event is a pull request that had an assignee removed.
```

```yaml
steps:
  - name: My first step
    uses: octo-org/action-name@main
  - name: My backup step
    if: ${{ failure() }}
    uses: actions/heroku@1.0.0
```

secret은 `if:`에서 직접 참조할 수 없습니다. job-level env로 옮긴 뒤 env를 조건에 사용하는 방식을 고려합니다.

```yaml
name: Run a step if a secret has been set
on: push
jobs:
  my-jobname:
    runs-on: ubuntu-latest
    env:
      super_secret: ${{ secrets.SuperSecret }}
    steps:
      - if: ${{ env.super_secret != '' }}
        run: echo 'This step will only run if the secret has a value set.'
      - if: ${{ env.super_secret == '' }}
        run: echo 'This step will only run if the secret does not have a value set.'
```

## `jobs.<job_id>.steps[*].name`

GitHub UI에 표시되는 step 이름입니다.

## `jobs.<job_id>.steps[*].uses`

step에서 실행할 action을 선택합니다. action은 같은 repository, public repository, Docker registry에서 가져올 수 있습니다.

action은 Git ref, SHA, Docker tag 등으로 version을 지정하는 것이 강하게 권장됩니다.

- release action의 commit SHA가 안정성과 보안 면에서 가장 안전합니다.
- major version tag는 호환성을 유지하면서 critical fix와 security patch를 받을 수 있습니다.
- default branch는 편하지만 breaking change에 취약합니다.

```yaml
steps:
  - uses: actions/checkout@8f4b7f84864484a7bf31766abe9204da3cbe65b3
  - uses: actions/checkout@v6
  - uses: actions/checkout@v6.2.0
  - uses: actions/checkout@main
```

public action:

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/heroku@main
      - name: My second step
        uses: actions/aws@v2.0.1
```

public action subdirectory:

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/aws/ec2@main
```

같은 repository의 action:

```yaml
jobs:
  my_first_job:
    runs-on: ubuntu-latest
    steps:
      - name: My first step - check out repository
        uses: actions/checkout@v6
      - name: Use local hello-world-action
        uses: ./.github/actions/hello-world-action
```

Docker Hub action:

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://alpine:3.8
```

GHCR public image:

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://ghcr.io/OWNER/IMAGE_NAME
```

private repository action을 직접 참조할 수 없다면 repository를 checkout한 뒤 local action으로 참조합니다. PAT 대신 GitHub App을 쓰면 token owner 퇴사 같은 문제를 줄일 수 있습니다.

```yaml
jobs:
  my_first_job:
    steps:
      - name: Check out repository
        uses: actions/checkout@v6
        with:
          repository: octocat/my-private-repo
          ref: v1.0
          token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          path: ./.github/actions/my-private-repo
      - name: Run my action
        uses: ./.github/actions/my-private-repo/my-action
```

## `jobs.<job_id>.steps[*].run`

shell에서 command-line program을 실행합니다. command는 21,000자를 넘을 수 없습니다. `name`이 없으면 `run` command text가 step name이 됩니다.

각 `run` keyword는 새 process와 shell입니다. multi-line command는 같은 shell에서 실행됩니다.

```yaml
- name: Install Dependencies
  run: npm install
```

```yaml
- name: Clean install dependencies and build
  run: |
    npm ci
    npm run build
```

## `jobs.<job_id>.steps[*].working-directory`

command를 실행할 working directory입니다.

```yaml
- name: Clean temp directory
  run: rm -rf *
  working-directory: ./temp
```

## `jobs.<job_id>.steps[*].shell`

step의 shell을 override합니다.

```yaml
steps:
  - name: Display the path
    shell: bash
    run: echo $PATH
```

```yaml
steps:
  - name: Display the path
    shell: cmd
    run: echo %PATH%
```

```yaml
steps:
  - name: Display the path
    shell: pwsh
    run: echo ${env:PATH}
```

```yaml
steps:
  - name: Display the path
    shell: python
    run: |
      import os
      print(os.environ['PATH'])
```

custom shell은 `command [options] {0} [more_options]` 형태로 지정합니다.

```yaml
steps:
  - name: Display the environment variables and their values
    shell: perl {0}
    run: |
      print %ENV
```

shell별 기본 실패 처리:

- `bash`/`sh`: 기본적으로 `set -e`가 적용됩니다. `shell: bash`는 `-o pipefail`도 적용합니다.
- `pwsh`/`powershell`: `$ErrorActionPreference = 'stop'`이 앞에 붙고 `$LASTEXITCODE` 처리가 뒤에 붙습니다.
- `cmd`: 완전한 fail-fast가 기본 제공되지 않으므로 script에서 error code를 직접 확인해야 합니다.

## `jobs.<job_id>.steps[*].with`

action input parameter map입니다. input은 `INPUT_` prefix와 upper case로 변환된 environment variable로 action에 전달됩니다.

```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/hello_world@main
        with:
          first_name: Mona
          middle_name: The
          last_name: Octocat
```

## `jobs.<job_id>.steps[*].with.args`

Docker container action input을 정의하는 string입니다. GitHub는 container start 시 `args`를 `ENTRYPOINT`에 전달합니다.

```yaml
steps:
  - name: Explain why this job ran
    uses: octo-org/action-name@main
    with:
      entrypoint: /bin/echo
      args: The ${{ github.event_name }} event triggered this step.
```

## `jobs.<job_id>.steps[*].with.entrypoint`

Dockerfile의 `ENTRYPOINT`를 override합니다. 실행할 executable을 나타내는 단일 string만 허용합니다.

```yaml
steps:
  - name: Run a custom command
    uses: octo-org/action-name@main
    with:
      entrypoint: /a/different/executable
```

## `jobs.<job_id>.steps[*].env`

step에서 사용할 environment variable을 설정합니다. secret이나 token은 `secrets` context를 사용합니다.

```yaml
steps:
  - name: My first action
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      FIRST_NAME: Mona
      LAST_NAME: Octocat
```

## `jobs.<job_id>.steps[*].continue-on-error`

step이 실패해도 job이 실패하지 않게 합니다.

## `jobs.<job_id>.steps[*].timeout-minutes`

step 최대 실행 시간입니다. 최대 360분이고 positive integer여야 합니다.

## `jobs.<job_id>.timeout-minutes`

job 최대 실행 시간입니다. 기본값은 360분입니다. `GITHUB_TOKEN`은 job 종료 시 또는 최대 24시간 후 만료됩니다.

## `jobs.<job_id>.strategy`

matrix strategy를 사용할 때 사용합니다. 하나의 job definition으로 여러 variable combination job을 생성할 수 있습니다.

## `jobs.<job_id>.strategy.matrix`

matrix는 workflow run당 최대 256개 job을 생성합니다. matrix variable은 `matrix` context의 property가 됩니다.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        version: [10, 12, 14]
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [ubuntu-22.04, ubuntu-24.04]
        version: [10, 12, 14]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.version }}
```

object array도 사용할 수 있습니다.

```yaml
matrix:
  os:
    - ubuntu-latest
    - macos-latest
  node:
    - version: 14
    - version: 20
      env: NODE_OPTIONS=--openssl-legacy-provider
```

## `jobs.<job_id>.strategy.matrix.include`

matrix combination에 추가 값을 넣거나 새 combination을 추가합니다.

```yaml
jobs:
  example_matrix:
    strategy:
      matrix:
        os: [windows-latest, ubuntu-latest]
        node: [14, 16]
        include:
          - os: windows-latest
            node: 16
            npm: 6
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - if: ${{ matrix.npm }}
        run: npm install -g npm@${{ matrix.npm }}
      - run: npm --version
```

```yaml
jobs:
  includes_only:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - site: "production"
            datacenter: "site-a"
          - site: "staging"
            datacenter: "site-b"
```

## `jobs.<job_id>.strategy.matrix.exclude`

matrix configuration을 제외합니다. partial match만으로도 제외됩니다. `include`는 `exclude` 이후 처리되므로 제외한 조합을 다시 추가할 수 있습니다.

## `jobs.<job_id>.strategy.fail-fast`

matrix 전체 실패 처리입니다. 기본값은 `true`입니다. matrix job 하나가 실패하면 진행 중이거나 queued된 다른 matrix job이 취소됩니다.

`continue-on-error`는 단일 job에 적용됩니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    continue-on-error: ${{ matrix.experimental }}
    strategy:
      fail-fast: true
      matrix:
        version: [6, 7, 8]
        experimental: [false]
        include:
          - version: 9
            experimental: true
```

## `jobs.<job_id>.strategy.max-parallel`

동시에 실행할 matrix job 수를 제한합니다. 기본적으로 GitHub는 runner availability에 따라 가능한 많은 job을 병렬 실행합니다.

## `jobs.<job_id>.continue-on-error`

job이 실패해도 workflow run이 실패하지 않게 할 수 있습니다.

```yaml
runs-on: ${{ matrix.os }}
continue-on-error: ${{ matrix.experimental }}
strategy:
  fail-fast: false
  matrix:
    node: [13, 14]
    os: [macos-latest, ubuntu-latest]
    experimental: [false]
    include:
      - node: 15
        os: ubuntu-latest
        experimental: true
```

## `jobs.<job_id>.container`

job step을 container 안에서 실행합니다. container action, job container, service container는 Linux runner가 필요합니다.

container 안의 `run` step default shell은 `bash`가 아니라 `sh`입니다.

```yaml
name: CI
on:
  push:
    branches: [ main ]
jobs:
  container-test-job:
    runs-on: ubuntu-latest
    container:
      image: node:18
      env:
        NODE_ENV: development
      ports:
        - 80
      volumes:
        - my_docker_volume:/volume_mount
      options: --cpus 1
    steps:
      - name: Check for dockerenv file
        run: (ls /.dockerenv && echo Found dockerenv) || (echo No dockerenv)
```

```yaml
jobs:
  container-test-job:
    runs-on: ubuntu-latest
    container: node:18
```

## `jobs.<job_id>.container.image`

job container image입니다. Docker Hub image name 또는 registry name을 사용할 수 있습니다.

## `jobs.<job_id>.container.credentials`

private container image pull credential입니다. `docker login`에 넣는 값과 같습니다.

```yaml
container:
  image: ghcr.io/owner/image
  credentials:
     username: ${{ github.actor }}
     password: ${{ secrets.github_token }}
```

## `jobs.<job_id>.container.env`

container 안의 environment variable map입니다.

## `jobs.<job_id>.container.ports`

container에서 expose할 port array입니다.

## `jobs.<job_id>.container.volumes`

container에서 사용할 volume array입니다.

```yaml
volumes:
  - my_docker_volume:/volume_mount
  - /data/my_data
  - /source/directory:/destination/directory
```

## `jobs.<job_id>.container.options`

추가 Docker container resource option입니다. `--network`, `--entrypoint`는 지원되지 않습니다.

## `jobs.<job_id>.services`

job에 service container를 붙입니다. database, Redis 같은 service를 테스트에서 사용할 때 유용합니다.

job이 container 안에서 실행되면 service label name을 hostname으로 사용해 접근할 수 있습니다. job이 runner host에서 직접 실행되면 필요한 port를 host로 mapping해야 합니다.

```yaml
services:
  nginx:
    image: nginx
    ports:
      - 8080:80
  redis:
    image: redis
    ports:
      - 6379/tcp
steps:
  - run: |
      echo "Redis available on 127.0.0.1:${{ job.services.redis.ports['6379'] }}"
      echo "Nginx available on 127.0.0.1:${{ job.services.nginx.ports['80'] }}"
```

## `jobs.<job_id>.services.<service_id>.image`

service container image입니다. 빈 string이면 service가 시작되지 않습니다.

```yaml
services:
  nginx:
    image: ${{ options.nginx == true && 'nginx' || '' }}
```

## `jobs.<job_id>.services.<service_id>.credentials`

service container pull credential입니다.

```yaml
services:
  myservice1:
    image: ghcr.io/owner/myservice1
    credentials:
      username: ${{ github.actor }}
      password: ${{ secrets.github_token }}
  myservice2:
    image: dockerhub_org/myservice2
    credentials:
      username: ${{ secrets.DOCKER_USER }}
      password: ${{ secrets.DOCKER_PASSWORD }}
```

## `jobs.<job_id>.services.<service_id>.env`

service container environment variable map입니다.

## `jobs.<job_id>.services.<service_id>.ports`

service container port array입니다.

## `jobs.<job_id>.services.<service_id>.volumes`

service container volume array입니다.

## `jobs.<job_id>.services.<service_id>.options`

service container 추가 option입니다. `--network`는 지원되지 않습니다.

## `jobs.<job_id>.services.<service_id>.command`

Docker image의 기본 `CMD`를 override합니다.

```yaml
services:
  mysql:
    image: mysql:8
    command: --sql_mode=STRICT_TRANS_TABLES --max_allowed_packet=512M
    env:
      MYSQL_ROOT_PASSWORD: test
    ports:
      - 3306:3306
```

## `jobs.<job_id>.services.<service_id>.entrypoint`

Docker image의 기본 `ENTRYPOINT`를 override합니다.

```yaml
services:
  etcd:
    image: quay.io/coreos/etcd:v3.5.17
    entrypoint: etcd
    command: >-
      --listen-client-urls http://0.0.0.0:2379
      --advertise-client-urls http://0.0.0.0:2379
    ports:
      - 2379:2379
```

## `jobs.<job_id>.uses`

reusable workflow file을 job으로 호출할 때 사용합니다.

가능한 syntax:

- `{owner}/{repo}/.github/workflows/{filename}@{ref}`
- `./.github/workflows/{filename}`

`{ref}`는 SHA, release tag, branch name일 수 있습니다. release tag와 branch 이름이 같으면 release tag가 우선합니다. 안정성과 보안을 위해 commit SHA가 가장 안전합니다.

같은 repository의 workflow를 `./.github/workflows/{filename}`로 호출하면 caller workflow와 같은 commit의 workflow를 사용합니다. 이 keyword에는 context나 expression을 사용할 수 없습니다.

```yaml
jobs:
  call-workflow-1-in-local-repo:
    uses: octo-org/this-repo/.github/workflows/workflow-1.yml@172239021f7ba04fe7327647b213799853a9eb89
  call-workflow-2-in-local-repo:
    uses: ./.github/workflows/workflow-2.yml
  call-workflow-in-another-repo:
    uses: octo-org/another-repo/.github/workflows/workflow.yml@v1
```

## `jobs.<job_id>.with`

reusable workflow로 전달할 input map입니다. called workflow에 정의된 input spec과 일치해야 합니다.

`steps[*].with`와 달리 `jobs.<job_id>.with` input은 called workflow에서 environment variable이 아니라 `inputs` context로 참조합니다.

```yaml
jobs:
  call-workflow:
    uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
    with:
      username: mona
```

## `jobs.<job_id>.with.<input_id>`

input identifier와 value의 pair입니다. identifier는 called workflow의 `on.workflow_call.inputs.<input_id>`와 일치해야 합니다.

허용되는 expression context는 `github`, `needs`입니다.

## `jobs.<job_id>.secrets`

reusable workflow로 전달할 secrets map입니다. called workflow에 정의된 secret 이름과 일치해야 합니다.

```yaml
jobs:
  call-workflow:
    uses: octo-org/example-repo/.github/workflows/called-workflow.yml@main
    secrets:
      access-token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
```

## `jobs.<job_id>.secrets.inherit`

calling workflow가 접근할 수 있는 organization, repository, environment secrets 전체를 called workflow로 전달합니다.

```yaml
on:
  workflow_dispatch:

jobs:
  pass-secrets-to-workflow:
    uses: ./.github/workflows/called-workflow.yml
    secrets: inherit
```

```yaml
on:
  workflow_call:

jobs:
  pass-secret-to-action:
    runs-on: ubuntu-latest
    steps:
      - name: Use a repo or org secret from the calling workflow.
        run: echo ${{ secrets.CALLING_WORKFLOW_SECRET }}
```

## `jobs.<job_id>.secrets.<secret_id>`

secret identifier와 value의 pair입니다. identifier는 called workflow의 `on.workflow_call.secrets.<secret_id>`와 일치해야 합니다.

허용되는 expression context는 `github`, `needs`, `secrets`입니다.

## Filter pattern cheat sheet

path, branch, tag filter에서 사용할 수 있는 special character입니다.

| Character | 의미 |
| --- | --- |
| `*` | 0개 이상의 문자와 match. `/`는 match하지 않음 |
| `**` | 0개 이상의 모든 문자와 match |
| `?` | 앞 문자가 0개 또는 1개인 경우 match |
| `+` | 앞 문자가 1개 이상인 경우 match |
| `[]` | bracket 안에 나열되거나 range에 포함된 alphanumeric 문자 하나와 match |
| `!` | pattern 시작에 있으면 이전 positive pattern을 negate |

YAML에서 `*`, `[`, `!`는 special character입니다. pattern이 이 문자들로 시작하면 quote로 감싸야 합니다.

```yaml
# Valid
paths:
  - '**/README.md'

# Invalid
paths:
  - **/README.md

# Valid
branches: [ main, 'release/v[0-9].[0-9]' ]

# Invalid
branches: [ main, release/v[0-9].[0-9] ]
```

### Branch와 tag pattern

| Pattern | 설명 | Match 예시 |
| --- | --- | --- |
| `feature/*` | slash(`/`)를 제외한 모든 문자와 match | `feature/my-branch`, `feature/your-branch` |
| `feature/**` | slash(`/`)를 포함한 모든 문자와 match | `feature/beta-a/my-branch`, `feature/mona/the/octocat` |
| `main`, `releases/mona-the-octocat` | 정확한 branch 또는 tag 이름과 match | `main`, `releases/mona-the-octocat` |
| `'*'` | slash(`/`)가 없는 모든 branch/tag 이름과 match | `main`, `releases` |
| `'**'` | 모든 branch/tag 이름과 match | `all/the/branches`, `every/tag` |
| `'*feature'` | `feature`로 끝나는 이름과 match | `mona-feature`, `feature`, `ver-10-feature` |
| `v2*` | `v2`로 시작하는 이름과 match | `v2`, `v2.0`, `v2.9` |
| `v[12].[0-9]+.[0-9]+` | major version 1 또는 2인 semantic version과 match | `v1.10.1`, `v2.0.0` |

### File path pattern

path pattern은 전체 path와 match하며 repository root에서 시작합니다.

| Pattern | 설명 | Match 예시 |
| --- | --- | --- |
| `'*'` | repository root file만 match | `README.md`, `server.rb` |
| `'*.jsx?'` | `page.js`, `page.jsx` 같은 이름과 match | `page.js`, `page.jsx` |
| `'**'` | 모든 file과 match | `all/the/files.md` |
| `'*.js'` | repository root의 `.js` file과 match | `app.js`, `index.js` |
| `'**.js'` | repository 전체 `.js` file과 match | `index.js`, `js/index.js`, `src/js/app.js` |
| `docs/*` | root `docs` 바로 아래 file만 match | `docs/README.md`, `docs/file.txt` |
| `docs/**` | root `docs`와 그 하위 file match | `docs/README.md`, `docs/mona/octocat.txt` |
| `docs/**/*.md` | `docs` 안의 `.md` file match | `docs/README.md`, `docs/a/markdown/file.md` |
| `'**/docs/**'` | 어디든 `docs` directory 안 file match | `docs/hello.md`, `dir/docs/my-file.txt` |
| `'**/README.md'` | 어디든 README.md file match | `README.md`, `js/README.md` |
| `'**/*src/**'` | `src` suffix folder 안 file match | `a/src/app.js`, `my-src/code/js/app.js` |
| `'**/*-post.md'` | `-post.md` suffix file match | `my-post.md`, `path/their-post.md` |
| `'**/migrate-*.sql'` | `migrate-` prefix와 `.sql` suffix file match | `migrate-10909.sql`, `db/migrate-v1.0.sql` |
| `'*.md'` + `'!README.md'` | negative pattern으로 README.md 제외 | `hello.md` match, `README.md` 제외 |
| `'*.md'` + `'!README.md'` + `README*` | 뒤 positive pattern이 다시 포함 가능 | `hello.md`, `README.md`, `README.doc` |
