# GitHub Actions Environments 정리

이 문서는 GitHub repository의 `Settings > Environments`에서 만드는 GitHub Environment를 설명합니다.
shell의 `env`, workflow의 `env`, `.env` 파일, Docker runtime environment와는 다른 개념입니다.

예시 환경 이름은 이 repository의 문서 규칙에 맞춰 `development`, `staging`, `production`으로 통일합니다.
짧은 이름을 선호하는 프로젝트에서는 `dev`, `stg`, `prod`를 한 세트로 사용합니다.

## 한 줄 정의

GitHub Environment는 GitHub Actions job이 배포 대상으로 참조할 수 있는 repository 단위 리소스입니다.
job에 `environment`를 붙이면 그 job은 해당 environment의 보호 규칙, secret, variable, deployment 표시와 연결됩니다.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://example.com
    steps:
      - run: ./deploy.sh
        env:
          API_URL: ${{ vars.API_URL }}
          API_TOKEN: ${{ secrets.API_TOKEN }}
```

## 영향 범위

Environment는 `jobs.<job_id>.environment`를 지정한 job에만 영향을 줍니다.
workflow 전체, repository 전체, runner 전체에 자동 적용되지 않습니다.

영향을 받는 것:

- 해당 job의 실행 전 보호 규칙
- 해당 job에서 접근 가능한 environment secret
- 해당 job에서 `vars` context로 접근 가능한 environment variable
- deployment 기록, deployment status, environment URL 표시
- OIDC token의 subject claim에 들어가는 environment 조건

영향을 받지 않는 것:

- `on.push`, `on.pull_request` 같은 workflow trigger 자체
- `environment`가 없는 다른 job
- branch protection rule 자체
- 서버나 컨테이너의 runtime `.env` 파일
- self-hosted runner의 격리 수준

## 실행 순서

Environment를 참조하는 job은 대략 아래 순서로 처리됩니다.

1. workflow가 event로 trigger됩니다.
2. job의 `if`, `needs`, matrix 등이 평가됩니다.
3. job이 참조하는 environment가 확인됩니다.
4. required reviewers, wait timer, branch/tag rule, custom protection rule을 통과해야 합니다.
5. 보호 규칙을 통과한 뒤 runner로 job이 전달됩니다.
6. job 안에서 environment secret과 environment variable을 사용할 수 있습니다.

승인 규칙이 있는 environment secret은 승인 전에는 job에서 접근할 수 없습니다.

## 보호 규칙

Environment에는 배포 전에 통과해야 하는 규칙을 붙일 수 있습니다.

| 규칙 | 의미 |
| --- | --- |
| Required reviewers | 지정한 사용자나 팀 중 1명이 승인해야 job 진행 |
| Prevent self-review | workflow를 실행한 사용자가 직접 승인하지 못하게 제한 |
| Wait timer | 1분부터 30일까지 배포 대기 |
| Deployment branches and tags | 특정 branch 또는 tag만 해당 environment로 배포 허용 |
| Admin bypass | 관리자가 보호 규칙을 우회할 수 있는지 설정 |
| Custom protection rules | GitHub App으로 외부 승인 시스템, 품질 게이트, 변경 관리 시스템 연결 |

branch/tag 제한은 workflow 실행 자체를 막는 기능이 아닙니다.
workflow는 trigger될 수 있고, environment를 참조하는 job이 해당 제한에 걸려 진행되지 않는 구조입니다.

## Secret과 variable

Environment secret은 해당 environment를 참조한 job에서만 사용할 수 있습니다.
환경별로 같은 secret 이름을 유지하면 workflow YAML을 단순하게 유지할 수 있습니다.

예를 들어 `development`, `staging`, `production` environment에 모두 `DATABASE_URL` secret을 만들면 workflow는 아래처럼 공통으로 쓸 수 있습니다.

```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

Environment variable도 해당 environment를 참조한 job에서만 `vars` context로 사용할 수 있습니다.

```yaml
env:
  API_URL: ${{ vars.API_URL }}
```

같은 이름이 organization, repository, environment에 모두 있으면 더 좁은 범위의 값이 우선합니다.
실무에서는 배포 대상별로 값이 달라지는 것은 environment에 두고, 모든 환경에서 같은 값은 repository나 organization에 둡니다.

민감한 값은 secret에 둡니다.
variable은 로그에 그대로 찍힐 수 있는 평문 설정값으로 봐야 합니다.

## Deployment와 URL

`environment`를 지정한 job은 기본적으로 GitHub Deployment와 연결됩니다.
여기서 Environment와 Deployment는 비슷하게 보이지만 역할이 다릅니다.

| 개념 | 의미 |
| --- | --- |
| Environment | `production`, `staging` 같은 배포 대상 설정. 보호 규칙, secret, variable을 가집니다. |
| Deployment | 특정 ref 또는 commit을 특정 environment에 배포하려는 한 번의 시도와 기록입니다. |
| Deployment status | 그 배포 시도의 현재 상태입니다. 예를 들어 `queued`, `in_progress`, `success`, `failure`, `error` 같은 상태가 붙습니다. |

Actions job에 `environment`를 붙이면 GitHub는 보통 해당 job을 배포로 보고 Deployment 기록을 만듭니다.
job이 시작되기 전에는 environment 보호 규칙을 먼저 확인하고, 통과한 뒤 runner에 job을 보냅니다.
job이 진행되면 GitHub UI에서 workflow run, PR, repository의 Deployments 화면에 배포 상태가 연결되어 보입니다.

즉 Environment는 "어디로 배포하는가"에 대한 설정이고, Deployment는 "이번 commit을 그곳에 배포했다"는 실행 기록입니다.
같은 `production` environment라도 commit이 바뀌거나 workflow run이 다시 실행되면 Deployment 기록은 새로 생길 수 있습니다.

`url`을 지정하면 배포된 서비스 링크로 표시됩니다.
이 값은 Deployments API의 `environment_url`로 매핑되고, repository의 Deployments 화면이나 PR의 deployment 상태에서 사용자가 눌러 이동할 수 있는 링크가 됩니다.

```yaml
environment:
  name: production
  url: https://example.com
```

preview URL처럼 step 결과로 만들어지는 주소도 사용할 수 있습니다.

```yaml
jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    environment:
      name: preview
      url: ${{ steps.deploy.outputs.preview-url }}
    steps:
      - id: deploy
        run: ./deploy-preview.sh
```

GitHub Deployment는 Actions 전용 개념은 아닙니다.
외부 배포 시스템도 GitHub Deployments API로 deployment와 deployment status를 만들 수 있습니다.
이 경우 GitHub는 배포 기록과 상태를 보여 주고, 실제 배포 작업은 외부 시스템이 수행합니다.

deployment object 없이 environment secret과 variable만 쓰고 싶으면 `deployment: false`를 사용할 수 있습니다.
예를 들어 테스트용 environment secret은 필요하지만 repository의 배포 이력에는 남기고 싶지 않은 job에서 쓸 수 있습니다.

```yaml
environment:
  name: production
  deployment: false
```

단, custom deployment protection rule은 deployment object가 필요하므로 `deployment: false`와 함께 쓰면 안 됩니다.
production 배포처럼 이력을 남기고 PR이나 repository 화면에서 배포 상태를 추적해야 하는 job에는 기본값처럼 deployment object를 만드는 편이 맞습니다.

## OIDC와 cloud 권한

OIDC를 쓰는 배포에서는 environment가 권한 조건으로 유용합니다.
job이 environment를 참조하면 OIDC token의 subject claim에 environment 정보가 들어갈 수 있습니다.

따라서 cloud provider의 trust policy에서 아래처럼 제한할 수 있습니다.

- `repo:<owner>/<repo>:environment:production`만 production role assume 허용
- `repo:<owner>/<repo>:environment:staging`은 staging role만 허용
- branch 조건과 environment 조건을 같이 사용

이 방식은 long-lived cloud secret을 GitHub Secret에 저장하지 않고, 승인된 environment job에서만 임시 credential을 발급받게 만드는 데 쓰입니다.

## Reusable workflow와 Environment

Reusable workflow를 호출하는 job은 일반 job과 달리 사용할 수 있는 keyword가 제한됩니다.
호출 job에는 `environment`를 직접 붙일 수 없습니다.

따라서 reusable workflow에서 GitHub Environment를 쓰려면 called workflow 내부의 job이 `environment`를 선언해야 합니다.
이 repository의 일부 reusable workflow는 caller가 environment 이름과 URL을 input으로 넘기고, called workflow 내부 job에서 사용합니다.

```yaml
jobs:
  deploy:
    uses: wibaek/gha/.github/workflows/cloudflare-pages-deploy.yaml@v1.0
    permissions:
      contents: read
    with:
      environment: production
      environment-url: https://example.com
      project-name: my-pages
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

이 repository에서 `environment` input을 받는 reusable workflow:

- `.github/workflows/cloudflare-pages-deploy.yaml`
- `.github/workflows/cloudflare-workers-deploy.yaml`
- `.github/workflows/ecs-deploy.yaml`
- `.github/workflows/ssh-compose-image-load-deploy.yaml`

`docker-build-*.yaml` 계열은 image build/push가 목적이므로 runtime environment를 붙이지 않습니다.
배포 environment는 build job이 아니라 deploy job에 붙이는 편이 맞습니다.

`ssh-compose-vps-deploy.yaml`처럼 environment input이 없는 workflow는 GitHub Environment 보호 규칙과 deployment UI에 직접 연결되지 않습니다.
필요하면 called workflow에 `environment` input과 job-level `environment`를 추가해야 합니다.

## 설정 방법

Repository에서 환경을 만들 때는 아래 순서로 설정합니다.

1. `Settings > Environments`로 이동합니다.
2. `New environment`를 눌러 `development`, `staging`, `production` 같은 이름을 만듭니다.
3. `production`에는 required reviewers와 branch/tag 제한을 우선 설정합니다.
4. 환경별 secret과 variable을 추가합니다.
5. workflow의 deploy job에서 `environment`를 참조합니다.
6. 배포 job이 expected environment approval과 secret을 사용하는지 workflow run에서 확인합니다.

GitHub CLI로 secret과 variable을 관리할 수도 있습니다.

```bash
gh secret set --env production API_TOKEN
gh secret list --env production

gh variable set API_URL --env production
gh variable list --env production
```

## 권장 기준

이 repository의 reusable workflow를 쓰는 caller repository에서는 아래 기준을 기본으로 둡니다.

- CI job에는 environment를 붙이지 않습니다.
- Docker image build/push job에는 runtime secret을 넣지 않습니다.
- 배포 job에만 environment를 붙입니다.
- `production`은 required reviewers와 branch/tag 제한을 둡니다.
- `development`, `staging`은 필요하면 wait timer 없이 자동 배포를 허용합니다.
- runtime secret은 build-args가 아니라 deploy job secret으로 전달합니다.
- environment별 secret 이름은 같게 유지하고 값만 다르게 둡니다.
- cloud 권한은 가능하면 OIDC와 environment 조건으로 제한합니다.
- 실제 배포 이력을 GitHub UI에서 추적해야 하는 job은 `deployment: false`를 쓰지 않습니다.

## 흔한 오해

Environment를 만들었다고 workflow가 자동으로 그 environment를 쓰지는 않습니다.
job에 `environment`를 명시해야 합니다.

Environment variable은 shell environment variable과 다릅니다.
workflow에서는 `vars.NAME`으로 읽고, 필요하면 `env`에 다시 매핑합니다.

Environment secret은 runner를 안전한 격리 공간으로 만들어 주지 않습니다.
self-hosted runner는 environment를 써도 격리 컨테이너에서 실행되는 것이 아니므로 repository secret과 같은 수준으로 다뤄야 합니다.

Reusable workflow 호출 job에 `environment`를 붙일 수 없습니다.
중앙 reusable workflow가 environment를 지원해야 caller에서 `with.environment` 형태로 제어할 수 있습니다.

branch/tag 제한은 trigger filter가 아닙니다.
push나 pull request workflow가 시작된 뒤, 해당 environment를 참조하는 job의 진행 여부를 제한합니다.

Environment와 Deployment는 같은 것이 아닙니다.
Environment는 repository 설정이고, Deployment는 특정 ref를 해당 environment에 배포한 이력입니다.
배포 이력을 남기지 않으려면 `deployment: false`를 명시해야 합니다.

## 점검 체크리스트

- deploy job에만 `environment`가 붙어 있는가?
- `production` environment에 required reviewers가 있는가?
- `production` environment가 `main` 또는 release tag만 허용하는가?
- runtime secret이 image build job이나 Dockerfile build arg로 들어가지 않는가?
- environment secret 이름이 `development`, `staging`, `production`에서 동일한가?
- reusable workflow가 environment input을 실제 job-level `environment`로 연결하는가?
- 배포 이력이 필요한 job에서 Deployment URL이 올바르게 표시되는가?
- 배포 이력이 필요 없는 job에서만 `deployment: false`를 쓰는가?
- OIDC를 쓰는 cloud role이 repository와 environment 조건을 모두 확인하는가?

## 참고 문서

- [Deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [Managing environments for deployment](https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments)
- [Reusing workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)
- [Variables reference](https://docs.github.com/en/actions/reference/workflows-and-actions/variables)
- [OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)
- [Workflow syntax: `jobs.<job_id>.environment`](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#jobsjob_idenvironment)
- [REST API: Deployments](https://docs.github.com/en/rest/deployments/deployments)
- [REST API: Deployment statuses](https://docs.github.com/en/rest/deployments/statuses)
