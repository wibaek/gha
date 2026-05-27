# GitHub Actions Worker 정리

GitHub Actions에서 흔히 worker라고 부르는 실행 주체의 공식 명칭은 runner입니다.
runner는 workflow의 job을 실제로 실행하는 머신 또는 컨테이너 실행 환경입니다.

이 문서는 GitHub Actions runner를 기준으로 job 실행 위치, runner 종류, `runs-on` 문법, 네트워크, 보안, 비용, 운영 기준을 정리합니다.

## 핵심 모델

GitHub Actions 실행 단위는 아래 순서로 이해하면 됩니다.

```text
workflow run
  └─ job
      └─ step
```

- workflow는 `.github/workflows/*.yaml` 파일로 정의합니다.
- workflow run은 특정 event 또는 수동 실행으로 생성됩니다.
- job은 runner 하나에 배정되어 실행됩니다.
- step은 같은 job 안에서 순서대로 실행됩니다.
- 여러 job은 기본적으로 병렬 실행됩니다.
- job 실행 순서가 필요하면 `needs`로 의존성을 걸어야 합니다.
- 각 job은 `runs-on`으로 실행할 runner를 선택합니다.

예시:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm test
```

`runs-on: ubuntu-latest`는 이 job을 GitHub-hosted Ubuntu runner에서 실행하겠다는 뜻입니다.

## Runner 종류

GitHub Actions runner는 크게 세 종류로 나뉩니다.

| 종류 | 운영 주체 | 사용 예 |
| --- | --- | --- |
| GitHub-hosted standard runner | GitHub | 일반 CI, 단순 build/test |
| GitHub-hosted larger runner | GitHub | 더 큰 CPU/RAM/disk, 고정 IP, GPU, runner group |
| self-hosted runner | 사용자/회사 | 내부망 접근, 커스텀 하드웨어, 자체 보안 경계 |

### GitHub-hosted standard runner

GitHub가 제공하는 기본 runner입니다.

특징:

- job마다 새 실행 환경을 받습니다.
- Ubuntu, Windows, macOS runner label을 사용할 수 있습니다.
- 기본 도구가 미리 설치되어 있습니다.
- VM image와 설치 도구 목록은 GitHub가 관리하고 주기적으로 갱신합니다.
- runner filesystem은 job이 끝나면 사라진다고 보는 것이 맞습니다.
- cache와 artifact는 runner disk가 아니라 GitHub Actions 저장소 기능으로 따로 보관합니다.

보통은 이걸 기본값으로 씁니다.

```yaml
runs-on: ubuntu-latest
```

권장:

- 일반적인 Node/Python/Docker build는 `ubuntu-latest` 또는 명시 버전인 `ubuntu-24.04`를 우선 사용합니다.
- 재현성이 중요하면 `ubuntu-latest`보다 `ubuntu-24.04`처럼 고정 label을 씁니다.
- macOS와 Windows는 필요한 경우에만 씁니다. 비용과 대기 시간이 커질 수 있습니다.

### Standard runner label과 사양

`runs-on`에 쓰는 workflow label은 실행할 runner image와 사양을 고르는 이름입니다.
같은 `ubuntu-latest` label이라도 public repository와 private repository에서 제공되는 standard runner 사양이 다를 수 있습니다.

아래 표는 문서 작성 시점의 GitHub-hosted standard runner 기준입니다.
정확한 최신 값은 GitHub 공식 runner reference를 확인합니다.

| Workflow label | OS | CPU | RAM | SSD | Architecture |
| --- | --- | ---: | ---: | ---: | --- |
| `ubuntu-slim` | Linux | 1 | 5 GB | 14 GB | x64 |
| `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` | Linux | 2 | 8 GB | 14 GB | x64 |
| `windows-latest`, `windows-2025`, `windows-2022` | Windows | 2 | 8 GB | 14 GB | x64 |
| `ubuntu-24.04-arm`, `ubuntu-22.04-arm` | Linux | 2 | 8 GB | 14 GB | arm64 |
| `windows-11-arm` | Windows | 2 | 8 GB | 14 GB | arm64 |
| `macos-15-intel`, `macos-26-intel` | macOS | 4 | 14 GB | 14 GB | Intel |
| `macos-latest`, `macos-14`, `macos-15`, `macos-26` | macOS | 3 M1 | 7 GB | 14 GB | arm64 |

`ubuntu-slim`은 1 vCPU Linux runner입니다.
가벼운 자동화, 짧은 lint/format, API call, 단순 script에 적합합니다.
무거운 Docker build나 큰 TypeScript/Java/Rust build에는 `ubuntu-latest` 계열이 더 안전합니다.

### GitHub-hosted larger runner

GitHub가 관리하지만 standard runner보다 큰 머신입니다.

사용 이유:

- CPU/RAM/disk가 더 필요함
- 고정 IP가 필요함
- Azure private networking이 필요함
- GPU runner가 필요함
- runner group으로 접근 제어를 하고 싶음
- workflow 동시 실행량을 autoscaling으로 다루고 싶음

일반적인 OSS/소규모 프로젝트에서는 과합니다.
회사 조직에서 빌드가 무겁거나 네트워크 allowlist가 필요할 때 검토합니다.

### Self-hosted runner

사용자가 직접 운영하는 runner입니다.

장점:

- 회사 내부망, private DB, 사내 registry, on-premise 서비스에 접근할 수 있습니다.
- 원하는 OS, 패키지, 하드웨어를 쓸 수 있습니다.
- GitHub-hosted runner에서 제공하지 않는 특수 환경을 구성할 수 있습니다.
- Actions 사용료 관점에서는 self-hosted runner 자체는 무료입니다.

단점:

- OS 업데이트, runner 업데이트, Docker 정리, 디스크 정리, 보안 패치가 모두 운영 책임입니다.
- job마다 깨끗한 VM을 보장하지 않습니다.
- 악성 workflow가 runner에 흔적을 남기거나 secret을 탈취할 수 있습니다.
- public repository에서는 거의 쓰지 않는 것이 안전합니다.
- private/internal repository에서도 fork PR, 권한, environment approval을 신중하게 설계해야 합니다.

권장:

- public repo에는 self-hosted runner를 붙이지 않습니다.
- 내부망 접근이 꼭 필요한 job만 self-hosted runner에서 실행합니다.
- CI 전체를 self-hosted로 옮기기보다, 배포나 내부망 검증 job만 제한적으로 둡니다.
- 가능하면 ephemeral runner 또는 JIT runner처럼 job마다 새 환경을 만드는 방식으로 운영합니다.

## `runs-on` 문법

`runs-on`은 job을 어떤 runner에서 실행할지 정합니다.

### 단일 label

가장 흔한 방식입니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

### 명시 버전 label

재현성을 조금 더 원하면 OS 버전을 고정합니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-24.04
```

### Matrix

여러 OS나 버전에서 같은 job을 돌릴 때 씁니다.

```yaml
jobs:
  test:
    strategy:
      matrix:
        os:
          - ubuntu-24.04
          - macos-latest
          - windows-latest
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - run: npm test
```

### Self-hosted label 조합

self-hosted runner는 label을 조합해서 선택합니다.
배열을 쓰면 모든 label을 만족하는 runner가 선택됩니다.

```yaml
jobs:
  deploy:
    runs-on:
      - self-hosted
      - linux
      - x64
      - prod
```

이 job은 `self-hosted`, `linux`, `x64`, `prod` label을 모두 가진 runner에서만 실행됩니다.

### Runner group

조직이나 엔터프라이즈에서는 runner group으로 접근 범위를 제한할 수 있습니다.

```yaml
jobs:
  deploy:
    runs-on:
      group: production-runners
      labels: linux
```

runner group은 특정 repository나 조직에만 runner를 노출하고 싶을 때 유용합니다.

## Runner 선택 기준

이 저장소의 reusable workflow 기준으로는 아래처럼 고르면 됩니다.

| 상황 | 추천 runner |
| --- | --- |
| 일반 CI | GitHub-hosted `ubuntu-latest` 또는 `ubuntu-24.04` |
| Node/Python lint/test/build | GitHub-hosted Ubuntu |
| Docker image build/push | GitHub-hosted Ubuntu |
| GHCR image를 VM에 stream deploy | GitHub-hosted Ubuntu |
| AWS ECR/ECS OIDC 배포 | GitHub-hosted Ubuntu |
| Cloudflare Pages/Workers 배포 | GitHub-hosted Ubuntu |
| 사내망 DB, private network 접근 | self-hosted 또는 larger runner private networking |
| 고정 outbound IP 필요 | larger runner |
| GPU 필요 | larger runner GPU 또는 self-hosted GPU |

기본은 GitHub-hosted runner입니다.
self-hosted runner는 네트워크나 하드웨어 이유가 있을 때만 씁니다.

## Runner filesystem

runner의 주요 경로는 대략 이렇게 이해합니다.

| 경로 | 의미 |
| --- | --- |
| `GITHUB_WORKSPACE` | repository가 checkout되는 작업 디렉터리 |
| `HOME` | runner user home |
| `RUNNER_TEMP` | job 중 임시 파일 경로 |

주의:

- `actions/checkout`을 실행하지 않으면 repository 파일이 자동으로 있는 것이 아닙니다.
- job이 다르면 workspace도 공유되지 않습니다.
- job 간 파일 전달은 artifact, cache, registry, 외부 storage를 사용해야 합니다.
- runner local disk에 남긴 파일은 다음 job에서 신뢰하지 않습니다.

## Cache와 artifact

Runner local disk는 job 생명주기와 묶여 있습니다.
반복 사용이 필요한 데이터는 GitHub Actions 기능을 써야 합니다.

### Cache

의존성 cache에 사용합니다.

예:

- pnpm store
- pip/uv cache
- Docker Buildx cache

cache는 “있으면 빠르고 없어도 동작해야 하는 값”에만 씁니다.
배포 artifact의 유일한 원본으로 쓰면 안 됩니다.

### Artifact

workflow run 결과물을 다운로드하거나 다음 job에서 쓰기 위해 올립니다.

예:

- test report
- coverage report
- build output
- release bundle

Docker image 배포에서는 artifact보다 registry push 또는 `docker save | ssh docker load`가 더 명확합니다.

## Docker build와 runner

GitHub-hosted Ubuntu runner에서 Docker build/push를 실행할 수 있습니다.
이 저장소는 registry별 workflow를 분리합니다.

```text
docker-build-ghcr-push.yaml
docker-build-ecr-push.yaml
docker-build-docker-hub-push.yaml
```

주의:

- job이 끝나면 runner local Docker image cache는 사라집니다.
- 다음 job에서 같은 image를 쓰려면 registry에 push하거나 artifact/tar로 넘겨야 합니다.
- registry별 표준 build/push workflow는 단일 platform만 지원합니다.
- multi-platform build가 필요하면 표준 workflow가 아니라 별도 Buildx workflow로 분리합니다.

## GHCR private image와 VM 배포

GHCR private image를 단일 VM에 배포할 때 서버가 GHCR에 직접 로그인하게 만들지 않는 것이 깔끔합니다.

이 저장소의 권장 흐름:

```text
GitHub-hosted runner
  -> GITHUB_TOKEN으로 GHCR login
  -> image pull
  -> local tag 생성
  -> docker save | ssh docker load

VM
  -> docker load
  -> docker compose up --pull never
```

이 구조에서는 VM에 GitHub PAT, deploy bot, GHCR credential이 필요 없습니다.

관련 workflow:

```text
docker-build-ghcr-push.yaml
ssh-compose-image-load-deploy.yaml
```

compose file은 아래 형태를 권장합니다.

```yaml
services:
  app:
    image: ${IMAGE_REFERENCE:?IMAGE_REFERENCE is required}
    pull_policy: never
    env_file:
      - ./app.env
```

`ssh-compose-image-load-deploy.yaml`은 서버에서 `docker compose up --pull never`를 강제합니다.

## Network와 IP

GitHub-hosted standard runner는 outbound network로 외부 서비스에 접근합니다.
하지만 고정 IP가 필요하면 standard runner만으로는 운영이 불편할 수 있습니다.

선택 기준:

- 단순 API 호출, Docker registry push, SSH 배포: standard runner로 충분한 경우가 많습니다.
- firewall allowlist에 고정 IP가 필요함: larger runner 또는 self-hosted runner 검토
- private network 안의 DB/API 접근: self-hosted runner 또는 larger runner private networking 검토

SSH 배포에서는 `known_hosts`를 고정해서 MITM 위험을 줄입니다.

## Secrets와 token

Runner는 job 실행 중 secret과 token을 다룰 수 있습니다.
따라서 runner 선택은 보안 경계 선택이기도 합니다.

기본 원칙:

- `permissions`를 job에 필요한 최소 권한으로 줄입니다.
- `contents: read`를 기본으로 둡니다.
- GHCR push는 `packages: write`가 필요합니다.
- GHCR pull은 `packages: read`가 필요합니다.
- AWS OIDC는 `id-token: write`가 필요합니다.
- secret은 로그에 출력하지 않습니다.
- untrusted PR에서 secret을 쓰는 job을 실행하지 않습니다.

예:

```yaml
permissions:
  contents: read
  packages: write
```

## Self-hosted runner 보안 기준

self-hosted runner는 회사 infra 안에서 실행되므로 강력하지만 위험합니다.

최소 기준:

- public repository에는 붙이지 않습니다.
- repository-level runner보다 organization-level runner group을 우선합니다.
- runner group으로 접근 repository를 제한합니다.
- label을 세분화합니다.
- 배포 runner와 CI runner를 분리합니다.
- root 권한이 필요한 작업을 줄입니다.
- Docker socket 접근을 신중하게 다룹니다.
- job 후 workspace와 Docker cache를 정리합니다.
- 가능하면 ephemeral runner를 사용합니다.

특히 Docker socket을 열어둔 self-hosted runner는 사실상 host root 권한에 가깝게 취급해야 합니다.

## Concurrency

GitHub Actions는 기본적으로 여러 workflow run과 여러 job을 동시에 실행할 수 있습니다.
배포 workflow는 동시 실행을 제한하는 편이 안전합니다.

예:

```yaml
concurrency:
  group: deploy-${{ github.repository }}-prod
  cancel-in-progress: false
```

운영 기준:

- production 배포는 `cancel-in-progress: false`로 순서대로 처리합니다.
- dev/stage 검증은 `cancel-in-progress: true`로 최신 실행만 남겨도 됩니다.
- 같은 서버나 같은 ECS service를 건드리는 배포는 같은 concurrency group을 씁니다.

## Timeout

job은 무한히 실행되면 안 됩니다.
workflow마다 적절한 `timeout-minutes`를 둡니다.

예:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15
```

기준:

- lint/test: 10-20분
- Docker build: 30-60분
- 배포: 10-30분
- release 자동화: 10-20분

## Billing과 limits

비용과 제한은 runner 종류와 GitHub plan에 따라 달라집니다.

기본 원칙:

- public repository의 standard GitHub-hosted runner는 일반적으로 무료로 사용할 수 있습니다.
- private repository는 plan별 무료 minute과 storage quota가 있습니다.
- larger runner는 더 많은 기능을 제공하지만 비용이 더 큽니다.
- self-hosted runner는 Actions 사용료 관점에서는 무료지만, 서버 운영 비용과 보안 비용이 있습니다.
- 동시 실행량, queue, artifact/cache storage, run duration에는 제한이 있습니다.

정확한 숫자는 자주 바뀔 수 있으므로 GitHub 공식 billing과 limits 문서를 확인합니다.

### Workflow label과 billing SKU

Workflow label은 실행 환경을 고르는 이름이고, billing SKU는 GitHub billing 시스템에서 쓰는 과금 bucket입니다.

```text
workflow label
  -> runner OS / CPU / architecture 선택
  -> billing SKU 매핑
  -> SKU별 per-minute rate 적용
```

예:

```yaml
runs-on: ubuntu-latest
```

private repository 기준으로는 대략 아래처럼 해석합니다.

```text
ubuntu-latest
  -> Linux 2-core x64 standard runner
  -> actions_linux
  -> Linux 2-core x64 rate
```

현재 runner pricing 문서 기준 SKU는 아래처럼 이해하면 됩니다.

| Workflow label | Billing SKU | Rate |
| --- | --- | ---: |
| `ubuntu-slim` | `actions_linux_slim` | `$0.002/min` |
| `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04` | `actions_linux` | `$0.006/min` |
| `ubuntu-24.04-arm`, `ubuntu-22.04-arm` | `actions_linux_arm` | `$0.005/min` |
| `windows-latest`, `windows-2025`, `windows-2022` | `actions_windows` | `$0.010/min` |
| `windows-11-arm` | `actions_windows_arm` | `$0.010/min` |
| `macos-latest`, `macos-14`, `macos-15`, `macos-26`, `macos-15-intel`, `macos-26-intel` | `actions_macos` | `$0.062/min` |

GitHub는 job의 partial minute을 job별로 올림 처리합니다.
예를 들어 10초 실행된 job도 billing 계산에서는 최소 1분 단위로 잡힐 수 있습니다.

중요한 점:

- `ubuntu-latest`, `ubuntu-24.04`, `ubuntu-22.04`는 label은 다르지만 같은 `actions_linux` SKU입니다.
- `ubuntu-slim`은 Linux이지만 1-core runner라 `actions_linux_slim` SKU입니다.
- Windows와 macOS는 Linux보다 rate가 높습니다.
- public repository의 standard GitHub-hosted runner usage는 무료로 취급됩니다.
- private repository의 standard GitHub-hosted runner usage는 plan에 포함된 minutes와 billing 설정의 영향을 받습니다.
- billing UI와 usage metrics가 보여주는 값은 raw runtime과 최종 과금 반영값이 다르게 보일 수 있습니다.

GitHub Pro의 `3,000 minutes`는 private repository에서 standard GitHub-hosted runner를 쓸 때 적용되는 월간 included usage입니다.
문서에 “3,000분은 정확히 actions_linux 기준 $18 credit”이라고 직접 적힌 것은 아니므로, 이 문서에서는 다음처럼 표현합니다.

```text
GitHub Pro included usage
  -> private repository standard runner usage에 적용
  -> ubuntu-latest 계열 Linux 2-core x64에서는 3,000 wall-clock minutes로 이해하기 쉽다
  -> 다른 OS/SKU는 billing rate가 다르므로 billing UI에서 최종 소진/비용을 확인한다
```

운영 판단:

| 상황 | 판단 |
| --- | --- |
| 일반 private Linux CI | `ubuntu-latest` 기준으로 생각하면 가장 덜 헷갈림 |
| 매우 가벼운 job | `ubuntu-slim` 실험 가치 있음 |
| Docker build / 무거운 compile | `ubuntu-latest` 이상 권장 |
| Windows 전용 테스트 | 비용 증가 감수 |
| macOS/iOS build | 비용이 커서 job 수와 trigger를 엄격히 제한 |
| 정확한 비용 추적 | billing dashboard와 usage export 기준으로 확인 |

## 이 저장소의 기본 방침

이 저장소의 reusable workflow는 다음 기준을 따릅니다.

1. 기본 runner는 `ubuntu-latest`로 둡니다.
2. 필요하면 caller가 `runs-on` input으로 runner label을 바꿉니다.
3. Docker build/push는 registry별 workflow를 우선 사용합니다.
4. 개인 VPS 기본 배포는 `ssh-compose-vps-deploy.yaml`을 사용합니다.
5. VM에 registry credential을 두지 않을 때만 `ssh-compose-image-load-deploy.yaml`을 사용합니다.
6. 배포 workflow는 concurrency group을 제공합니다.
7. production 배포는 진행 중인 배포를 취소하지 않는 것을 기본값으로 둡니다.
8. self-hosted runner는 내부망 접근이 꼭 필요한 경우에만 사용합니다.

## 자주 헷갈리는 점

### runner와 workflow는 같은 것이 아닙니다

workflow는 정의 파일이고, runner는 job을 실행하는 머신입니다.

### runner와 repository checkout은 별개입니다

runner가 할당되어도 repository 파일은 자동으로 준비되지 않습니다.
대부분의 job은 `actions/checkout`이 필요합니다.

### job 간 local filesystem은 공유되지 않습니다

같은 workflow 안에서도 job이 다르면 다른 runner에서 실행될 수 있습니다.
파일 전달은 artifact/cache/registry/외부 storage를 써야 합니다.

### `GITHUB_TOKEN`은 장기 VM 배포 credential이 아닙니다

`GITHUB_TOKEN`은 Actions job 안에서 쓰는 token입니다.
서버에 저장해 두고 장기 credential처럼 쓰지 않습니다.
`ssh-compose-vps-deploy.yaml`처럼 배포 job 실행 중에만 임시 `DOCKER_CONFIG`로 GHCR pull에 쓰고 정리하는 방식은 가능합니다.
이 경우 caller deploy job에 `packages: read` 권한이 필요하고, private package가 workflow repository에 read access를 가져야 합니다.

### self-hosted runner는 깨끗한 VM이 아닐 수 있습니다

GitHub-hosted runner와 달리 self-hosted runner는 job 후에도 상태가 남을 수 있습니다.
untrusted code를 실행하면 runner가 오염될 수 있습니다.

# Reference

- [GitHub Docs: Choosing the runner for a job](https://docs.github.com/en/actions/how-tos/write-workflows/choose-where-workflows-run/choose-the-runner-for-a-job?apiVersion=2022-11-28)
- [GitHub Docs: GitHub-hosted runners](https://docs.github.com/actions/how-tos/using-github-hosted-runners/using-github-hosted-runners/about-github-hosted-runners)
- [GitHub Docs: GitHub-hosted runners reference](https://docs.github.com/en/actions/reference/runners/github-hosted-runners)
- [GitHub Docs: Larger runners](https://docs.github.com/en/actions/using-github-hosted-runners/using-larger-runners/about-larger-runners)
- [GitHub Docs: Self-hosted runners](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners?platform=linux)
- [GitHub Docs: Using labels with self-hosted runners](https://docs.github.com/actions/how-tos/manage-runners/self-hosted-runners/apply-labels)
- [GitHub Docs: Workflow syntax for GitHub Actions](https://docs.github.com/actions/reference/workflow-syntax-for-github-actions)
- [GitHub Docs: Concurrency](https://docs.github.com/en/actions/concepts/workflows-and-actions/concurrency)
- [GitHub Docs: Billing and usage](https://docs.github.com/actions/learn-github-actions/usage-limits-billing-and-administration)
- [GitHub Docs: GitHub Actions billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)
- [GitHub Docs: Actions runner pricing](https://docs.github.com/billing/reference/actions-minute-multipliers)
- [GitHub Docs: Actions limits](https://docs.github.com/actions/reference/limits)
- [GitHub Docs: Secure use reference](https://docs.github.com/en/actions/how-tos/security-for-github-actions/security-guides/security-hardening-for-github-actions?learn=getting_started)
- [GitHub Changelog: Reduced pricing for GitHub-hosted runners usage](https://github.blog/changelog/2026-01-01-reduced-pricing-for-github-hosted-runners-usage/)
- [GitHub Changelog: 1 vCPU Linux runner generally available](https://github.blog/changelog/2026-01-22-1-vcpu-linux-runner-now-generally-available-in-github-actions/)
- [GitHub Changelog: arm64 standard runners in private repositories](https://github.blog/changelog/2026-01-29-arm64-standard-runners-are-now-available-in-private-repositories)
