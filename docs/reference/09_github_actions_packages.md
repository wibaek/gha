# GitHub Actions Packages 정리

이 문서는 GitHub Packages와 GitHub Container Registry(GHCR)를 GitHub Actions에서 사용할 때의 권한, 접근 제어, token, 이 repository의 reusable workflow 사용 기준을 정리합니다.
이 repository에서 package라고 말할 때 대부분은 `ghcr.io`에 올라가는 Docker/OCI image를 뜻합니다.

## 한 줄 요약

GHCR image push에는 `packages: write`, pull에는 `packages: read`가 필요합니다.
같은 repository의 workflow가 만든 package는 `GITHUB_TOKEN`으로 다루는 것이 기본이고, 다른 repository나 외부 서버에서 private package를 읽어야 할 때만 package access 부여나 별도 token을 검토합니다.

```text
docker-build-ghcr-push.yaml
  -> GITHUB_TOKEN + packages: write
  -> ghcr.io/<owner>/<repo> image push
  -> image-reference output은 digest reference 우선

ssh-compose-vps-deploy.yaml
  -> GITHUB_TOKEN + packages: read
  -> 서버에서 임시 DOCKER_CONFIG로 ghcr.io login
  -> docker compose pull
  -> logout 후 credential 정리
```

## GitHub Packages와 GHCR

GitHub Packages는 package registry 서비스입니다.
npm, Maven, NuGet 같은 package registry도 포함하지만, 이 repository의 배포 흐름에서는 Container registry인 GHCR을 주로 사용합니다.

GHCR image 이름은 보통 아래 형태입니다.

```text
ghcr.io/<owner>/<image-name>:<tag>
ghcr.io/<owner>/<image-name>@sha256:<digest>
```

이 repository의 `docker-build-ghcr-push.yaml`에서 `image-name: auto`를 쓰면 기본 image 이름은 아래처럼 만들어집니다.

```text
ghcr.io/${{ github.repository }}
```

예를 들어 repository가 `wibaek/my-api`이면 image는 `ghcr.io/wibaek/my-api`입니다.

## Repository와 Package 연결

GHCR package는 user 또는 organization namespace에 속하면서 repository와 연결될 수 있습니다.
workflow에서 `GITHUB_TOKEN`으로 package를 처음 publish하면 해당 workflow repository와 package가 자동으로 연결되는 편이 가장 단순합니다.

연결된 repository는 package에 대한 Actions access를 가질 수 있습니다.
같은 repository의 build job이 image를 push하고, 같은 repository의 deploy job이 image를 pull하는 구조라면 보통 별도 PAT가 필요 없습니다.

command line에서 먼저 push한 package는 repository와 자동으로 연결되지 않을 수 있습니다.
이 경우 `GITHUB_TOKEN`으로 push하려고 할 때 권한 오류가 날 수 있습니다.
package 설정에서 repository를 연결하거나, Docker image에 OCI source label을 추가해서 연결을 명확히 합니다.

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/OWNER/REPO"
```

이 label은 package page에서 source repository를 표시하는 데도 도움이 됩니다.

## Visibility와 access

Package visibility와 repository visibility는 항상 같은 것이 아닙니다.
GHCR package는 package 설정에서 visibility와 access를 따로 관리할 수 있습니다.

| 설정 | 의미 |
| --- | --- |
| Public package | 누구나 pull할 수 있는 image. 민감하지 않은 open source image에 사용 |
| Private package | 허용된 사용자, team, repository, workflow만 읽을 수 있는 image |
| Repository access | 특정 repository의 Actions workflow가 package를 읽거나 쓸 수 있게 부여하는 권한 |
| Inherit access | package가 연결된 repository의 access model을 따르는 방식 |

private application image는 기본적으로 private package로 둡니다.
다른 repository의 workflow가 그 image를 pull해야 하면 package settings에서 해당 repository에 read access를 부여합니다.
이 접근이 가능하면 별도 `GHCR_TOKEN`보다 `GITHUB_TOKEN`과 package access 조합을 우선합니다.

public repository에 private package access를 부여하면 fork workflow와 secret 노출 모델을 따로 검토해야 합니다.
private package가 필요한 job은 trusted branch, internal PR, environment approval이 있는 deploy job으로 제한하는 편이 안전합니다.

## Token과 권한

GitHub Actions에서 GHCR을 다룰 때 기본 token은 `GITHUB_TOKEN`입니다.
workflow에는 필요한 scope를 `permissions`로 명시합니다.

### Push

GHCR에 image를 push하는 job은 `packages: write`가 필요합니다.

```yaml
jobs:
  docker:
    uses: wibaek/gha/.github/workflows/docker-build-ghcr-push.yaml@v1.0
    permissions:
      contents: read
      packages: write
    with:
      image-name: auto
```

이 repository의 `docker-build-ghcr-push.yaml`은 내부에서도 아래 권한을 선언합니다.

```yaml
permissions:
  contents: read
  packages: write
```

caller job에서도 권한을 명시해 두면 reusable workflow를 읽는 사람이 의도를 바로 알 수 있습니다.

### Pull

GHCR private image를 pull하는 job은 `packages: read`가 필요합니다.

```yaml
jobs:
  deploy:
    uses: wibaek/gha/.github/workflows/ssh-compose-vps-deploy.yaml@v1.0
    permissions:
      contents: read
      packages: read
    with:
      ghcr-login: true
      image-reference: ${{ needs.docker.outputs.image-reference }}
```

같은 repository와 연결된 private package는 이 권한만으로 pull되는 경우가 많습니다.
다른 repository의 package라면 package settings에서 workflow repository에 read access를 부여합니다.

### PAT와 override token

`GHCR_TOKEN`은 기본값이 아니라 예외 처리용입니다.
다음 상황에서만 검토합니다.

- package access를 workflow repository에 부여할 수 없음
- 다른 owner namespace의 private package를 pull해야 함
- 외부 서버나 local CLI에서 직접 private package를 pull해야 함
- 기존 package가 repository와 연결되지 않아 `GITHUB_TOKEN`으로 접근할 수 없음

GitHub Packages 인증에는 classic PAT가 필요할 수 있습니다.
pull만 필요하면 `read:packages`, push가 필요하면 `write:packages`를 최소 권한으로 둡니다.
가능하면 넓은 `repo` scope를 피합니다.

## 이 repository의 GHCR build workflow

`docker-build-ghcr-push.yaml`은 Docker image를 빌드하고 GHCR에 push합니다.
runtime secret은 다루지 않습니다.

핵심 input과 output은 다음과 같습니다.

| 구분 | 이름 | 의미 |
| --- | --- | --- |
| input | `image-name` | `auto`이면 `ghcr.io/${{ github.repository }}` |
| input | `tags` | `docker/metadata-action`에 전달할 tag 규칙 |
| input | `cache-type` | `gha`, `registry`, `none` |
| input | `registry-cache-ref` | registry cache를 쓸 때의 cache image ref |
| secret | `GHCR_TOKEN` | push용 override token. 비우면 `github.token` |
| output | `image-tags` | 생성된 tag 목록 |
| output | `image-primary-tag` | 첫 번째 tag |
| output | `image-reference` | 배포에 사용할 digest reference 우선 값 |
| output | `image-digest` | 빌드된 image digest |

배포에는 tag보다 digest reference를 우선합니다.
tag는 나중에 같은 이름으로 다시 push될 수 있지만, digest는 특정 image content를 가리킵니다.
이 repository의 workflow도 `image-reference` output에서 digest reference를 우선 만들어 줍니다.

```yaml
deploy:
  needs: docker
  uses: wibaek/gha/.github/workflows/ssh-compose-vps-deploy.yaml@v1.0
  with:
    image-reference: ${{ needs.docker.outputs.image-reference }}
```

## Deploy에서 pull하는 방식

GHCR private image를 서버에 배포하는 방식은 두 가지입니다.

| 방식 | 서버 GHCR credential | 특징 |
| --- | --- | --- |
| `ssh-compose-vps-deploy.yaml` | 배포 중 임시 login | 서버가 직접 `docker compose pull` |
| `ssh-compose-image-load-deploy.yaml` | 필요 없음 | runner가 image를 pull하고 서버에는 `docker save/load` 형태로 전달 |

`ssh-compose-vps-deploy.yaml`은 서버에 임시 `DOCKER_CONFIG`를 만들고 GHCR에 로그인한 뒤, 배포가 끝나면 logout과 credential 정리를 수행합니다.
개인 VPS에서 일반적인 pull 기반 배포를 할 때 단순합니다.

`ssh-compose-image-load-deploy.yaml`은 서버가 registry에 로그인하지 않습니다.
runner가 image를 pull한 뒤 서버에 image와 compose file을 전송합니다.
서버에 registry credential을 남기고 싶지 않거나 서버에서 registry 접근이 제한된 경우에 사용합니다.

## Registry cache와 package

`cache-type: registry`를 쓰면 Buildx cache manifest를 registry에 저장합니다.
예를 들어 아래 값은 실행용 image tag가 아니라 cache 저장용 ref입니다.

```yaml
cache-type: registry
registry-cache-ref: ghcr.io/owner/app:buildcache
```

registry cache도 GHCR에 push되므로 `packages: write`가 필요합니다.
`buildcache` tag는 배포 대상 image로 쓰지 않습니다.
cache lifecycle을 명시적으로 관리하고 싶거나 GitHub Actions cache 제한과 분리하고 싶을 때만 registry cache를 검토합니다.

일반 프로젝트는 기본값인 `cache-type: gha`로 시작하는 편이 단순합니다.

## 권장 패턴

- GHCR image 이름은 기본적으로 `ghcr.io/${{ github.repository }}`를 사용합니다.
- build job에는 `contents: read`, `packages: write`를 명시합니다.
- deploy job에는 `contents: read`, `packages: read`를 명시합니다.
- 같은 repository package는 `GITHUB_TOKEN`으로 push/pull합니다.
- 다른 repository package는 먼저 package access를 workflow repository에 부여합니다.
- PAT나 `GHCR_TOKEN` override는 package access로 해결되지 않을 때만 사용합니다.
- 서버에는 long-lived GHCR credential을 두지 않습니다.
- 배포에는 tag보다 digest reference를 사용합니다.
- public image가 아니라면 package visibility를 private으로 둡니다.
- Dockerfile 또는 metadata에 `org.opencontainers.image.source`를 남겨 source repository 연결을 명확히 합니다.

## 자주 헷갈리는 점

`packages: write`는 repository contents write 권한이 아닙니다.
GitHub Packages에 upload/publish할 수 있는 권한입니다.
code push나 branch 수정 권한과는 별개입니다.

`contents: read`만으로 GHCR private image를 pull할 수 없습니다.
private package pull에는 `packages: read`가 필요합니다.

package가 repository 이름과 같은 namespace에 있어도 자동으로 같은 repository 권한을 쓰는 것은 아닙니다.
package가 repository에 연결되어 있는지, workflow repository에 Actions access가 있는지 확인해야 합니다.

`GITHUB_TOKEN`은 현재 workflow repository에 설치된 GitHub App token입니다.
다른 repository의 private package를 무조건 읽을 수 있는 organization-wide token이 아닙니다.

`latest` tag는 고정된 배포 단위가 아닙니다.
배포 재현성이 필요하면 `ghcr.io/owner/app@sha256:...` 형태의 digest reference를 사용합니다.

## 문제 해결 체크리스트

- build job에 `packages: write`가 있는가?
- deploy job에 `packages: read`가 있는가?
- image 이름이 `ghcr.io/<owner>/<image>` 형태로 맞는가?
- package가 workflow repository에 연결되어 있는가?
- private package라면 workflow repository에 read 또는 write access가 있는가?
- command line으로 먼저 push한 package라면 repository 연결이나 OCI source label을 확인했는가?
- 다른 repository package를 pull한다면 package settings에서 Actions access를 부여했는가?
- `GHCR_TOKEN`을 쓴다면 필요한 최소 scope만 있는가?
- 서버에 long-lived registry credential이 남지 않는가?
- deploy job이 `image-reference` digest output을 쓰는가?

## 참고 문서

- [Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [About permissions for GitHub Packages](https://docs.github.com/en/packages/learn-github-packages/about-permissions-for-github-packages)
- [Configuring package access control and visibility](https://docs.github.com/en/packages/learn-github-packages/configuring-a-packages-access-control-and-visibility)
- [Publishing and installing a package with GitHub Actions](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions)
- [Connecting a repository to a package](https://docs.github.com/en/packages/learn-github-packages/connecting-a-repository-to-a-package)
- [GITHUB_TOKEN](https://docs.github.com/en/actions/security-guides/automatic-token-authentication)
