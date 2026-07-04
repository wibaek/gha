# HeadVer 기반 GitHub CI/CD 정리

이 문서는 HeadVer를 GitHub Actions 기반 CI/CD에 적용할 때의 version 생성, artifact publish, staging/production 배포, package tag, Git tag, GitHub Release, head bump, hotfix, rollback 규칙을 정리합니다.
기본 모델은 일반적인 단일 앱/단일 package repository입니다.

## 핵심 원칙

```text
같은 artifact면 같은 version.
version이 바뀌면 새 artifact.
production 배포는 rebuild가 아니라 artifact promotion.
```

운영 기준은 다음과 같습니다.

```text
version        = {head}.{yearweek}.{build}
release config = release.yaml
head           = release.yaml의 versioning.head
yearweek       = TZ={versioning.timezone} date +%g%V
build          = github.run_number + versioning.build_offset
publish key    = artifact + build
version key    = artifact + version
production     = 기존 artifact를 선택해서 배포, rebuild 금지
head bump      = regular production 배포 성공 후 PR 생성
hotfix         = 팀 정책에 따라 기존 head 유지 또는 새 head 사용 가능
rollback       = 기존 artifact 재배포, 새 build 없음
```

hotfix를 기존 head로 내는 것은 HeadVer가 강제하는 규칙이 아니라 운영 정책입니다.
예를 들어 `5.2601.120`이 production이고 `6.x`를 개발 중일 때 `5.2601.124` hotfix를 낼 수 있습니다.
중요한 것은 같은 full version을 다시 쓰지 않는 것입니다.

```text
허용: 5.2601.120 -> 5.2601.124  # 같은 head, 새 build
금지: 5.2601.120 -> 5.2601.120  # 새 artifact에 같은 full version 재사용
```

## HeadVer 개념

HeadVer 형식은 다음과 같습니다.

```text
{head}.{yearweek}.{build}
```

예:

```text
6.2601.143
```

| Field | 의미 | 생성 방식 |
| --- | --- | --- |
| `head` | 사용자에게 전달되는 release line 또는 release 회차 | 수동 관리 |
| `yearweek` | ISO week-year 2자리 + ISO week number 2자리 | CI에서 자동 계산 |
| `build` | build server가 만든 artifact 번호 | CI에서 자동 증가 |

HeadVer의 핵심은 사람이 관리하는 숫자를 앞자리 `head` 하나로 줄이고, 나머지는 CI가 계산하는 것입니다.
SemVer의 `major.minor.patch`와 모양은 비슷하지만 의미는 다릅니다.

```text
SemVer:  major.minor.patch
HeadVer: head.yearweek.build
```

`head`는 SemVer의 `major`가 아니라 사용자에게 전달되는 release line입니다.

```text
5.2601.120 = v5 release line의 build 120
6.2601.121 = v6 release line의 build 121
5.2601.124 = v5 release line의 hotfix build 124
```

## Version은 artifact identity

이 설계에서 version은 deploy event가 아니라 artifact identity입니다.

```text
artifact = source + build environment + build metadata + package result
version  = artifact identity
```

같은 artifact를 staging에서 production으로 승격해도 version은 바뀌지 않습니다.

```text
staging 검증:    5.2601.120
production 배포: 5.2601.120
```

반대로 version을 바꾸려면 새 artifact를 만들어야 합니다.

```text
5.2601.120 artifact를 QA함
production 직전에 이름만 6.2601.120으로 바꿈
-> 금지
```

production 배포는 rebuild가 아니라 artifact promotion이어야 합니다.

```text
build workflow:
  source checkout
  version 생성
  artifact build
  artifact publish
  staging deploy

deploy workflow:
  version 입력
  artifact 조회
  digest 확인
  target environment 배포
  rebuild 금지
```

## Repository 구조

단일 앱 repository는 root에 `release.yaml`을 둡니다.

```text
.
├── release.yaml
├── scripts
│   ├── generate-headver.sh
│   └── validate-headver-yearweek.sh
├── .github
│   └── workflows
│       ├── ci.yaml
│       ├── build.yaml
│       ├── deploy.yaml
│       ├── bump-head.yaml
│       └── guard-headver.yaml
└── docs
    └── headver-github-cicd.md
```

기본 `release.yaml` 예시:

```yaml
schema_version: 1
artifact: app

versioning:
  scheme: headver
  head: 6
  build_offset: 99
  timezone: Asia/Seoul

release:
  head_bump:
    mode: pull_request
    after_regular_production: true

publish:
  uniqueness:
    primary: artifact_build
    secondary: artifact_version
  rerun_policy: block_publish

deploy:
  staging:
    enabled: true
    auto_on_main: true
    strategy: overwrite

  production:
    environment: production
    require_approval: true
```

`artifact`는 repository가 만드는 배포 단위 이름입니다.
단일 앱 repository라면 `app`, `api`, `frontend`처럼 하나만 두면 됩니다.

`release_kind`는 `release.yaml`에 저장하지 않습니다.
`regular`, `hotfix`, `rollback`은 배포 실행 시점의 의사결정이므로 `deploy.yaml`의 input으로 받습니다.

## Build number

`github.run_number`를 임의로 100부터 시작하게 설정하지 않고 offset을 더합니다.

```text
build = github.run_number + build_offset
```

예:

| `github.run_number` | `build_offset` | 최종 `build` |
| ---: | ---: | ---: |
| 1 | 99 | 100 |
| 2 | 99 | 101 |
| 3 | 99 | 102 |

주의할 점은 `github.run_number`가 workflow별 counter라는 것입니다.
한 artifact는 하나의 artifact-producing workflow에서만 만들어야 합니다.
hotfix도 별도 `build-hotfix.yaml`을 만들지 말고 같은 `build.yaml`을 `workflow_dispatch`로 실행합니다.

## Yearweek

`yearweek`는 calendar year가 아니라 ISO week-year를 사용합니다.

잘못된 예:

```bash
yearweek="$(TZ=Asia/Seoul date +%y%V)"
```

올바른 예:

```bash
yearweek="$(TZ=Asia/Seoul date +%g%V)"
```

| Date | `%y%V` | `%g%V` | Expected |
| --- | ---: | ---: | ---: |
| 2012-01-01 | 1252 | 1152 | 1152 |
| 2010-01-01 | 1053 | 0953 | 0953 |
| 2019-12-31 | 1901 | 2001 | 2001 |
| 2025-12-31 | 2501 | 2601 | 2601 |
| 2016-01-01 | 1653 | 1553 | 1553 |

팀 정책은 아래처럼 고정합니다.

```text
빌드 시간 기준 timezone = Asia/Seoul
주차 계산 방식 = ISO week date
yearweek = %g%V
```

CI에는 연말/연초 경계값을 검증하는 script를 둡니다.

## Version 생성

`scripts/generate-headver.sh`는 `release.yaml`을 읽어서 `VERSION`, `BUILD`, `BUILD_KEY`를 생성합니다.

```bash
scheme="$(yq -r '.versioning.scheme' release.yaml)"
artifact="$(yq -r '.artifact // "app"' release.yaml)"
head="$(yq '.versioning.head' release.yaml)"
build_offset="$(yq '.versioning.build_offset // 0' release.yaml)"
timezone="$(yq -r '.versioning.timezone // "Asia/Seoul"' release.yaml)"

yearweek="$(TZ="$timezone" date +%g%V)"
build="$((GITHUB_RUN_NUMBER + build_offset))"
version="${head}.${yearweek}.${build}"
build_key="${artifact}:${build}"
```

GitHub Actions에서는 output으로 넘깁니다.

```bash
echo "version=${version}" >> "$GITHUB_OUTPUT"
echo "build=${build}" >> "$GITHUB_OUTPUT"
echo "build_key=${build_key}" >> "$GITHUB_OUTPUT"
```

## Publish uniqueness

publish 중복 방지는 full version만 보면 부족합니다.

```text
처음 실행:
  github.run_number = 143
  build = 143
  date = 2026-W01
  version = 6.2601.143

일주일 뒤 rerun:
  github.run_number = 143
  build = 143
  date = 2026-W02
  version = 6.2602.143
```

rerun이면 `github.run_number`는 그대로인데 현재 날짜 기준으로 `yearweek`만 바뀔 수 있습니다.
그래서 primary uniqueness key는 full version이 아니라 `artifact + build`입니다.

```text
primary key   = artifact + build
secondary key = artifact + version
```

정책:

| 대상 | 중복 기준 | 동작 |
| --- | --- | --- |
| artifact publish | `artifact + build` | 이미 있으면 실패 |
| artifact publish | `artifact + version` | 이미 있으면 실패 |
| staging deploy | `artifact + version` | 허용 |
| production deploy | `artifact + version` | 허용, approval 필요 |
| rollback | 기존 `artifact + version` | 허용 |
| redeploy | 기존 `artifact + version` | 허용 |

막아야 하는 것은 같은 build number로 새 artifact를 publish하는 것입니다.
허용해야 하는 것은 기존 artifact를 같은 version으로 다시 deploy하는 것입니다.

## Rerun 정책

기본 정책은 단순하게 둡니다.

```text
github.run_attempt != 1 이면 artifact publish 금지
```

build가 실패했고 새 artifact가 필요하면 rerun으로 같은 build number를 재사용하지 말고 새 workflow run을 만들어 새 build number를 받습니다.

```yaml
- name: Block artifact publish on rerun
  if: github.run_attempt != '1'
  run: |
    echo "::error title=Artifact publish blocked on rerun::Start a new workflow run to get a new build number."
    exit 1
```

작은/중간 규모 팀에서는 claim manifest를 별도로 설계하는 것보다 `run_attempt != 1` publish 차단이 더 단순하고 안전합니다.

## Artifact manifest

모든 build artifact에는 추적 가능한 metadata를 남깁니다.

```json
{
  "artifact": "app",
  "version": "6.2601.143",
  "head": 6,
  "yearweek": "2601",
  "build": 143,
  "build_key": "app:143",
  "git_sha": "abc123",
  "git_ref": "refs/heads/main",
  "github_run_id": "1234567890",
  "github_run_number": 44,
  "github_run_attempt": 1,
  "package": "ghcr.io/org/repo:6.2601.143",
  "package_digest": "sha256:...",
  "built_at": "2026-01-02T12:34:56+09:00"
}
```

운영 중에는 version에서 바로 역추적할 수 있어야 합니다.

```text
version -> manifest
manifest -> git_sha
manifest -> GitHub Actions run
manifest -> package digest
manifest -> deployment history
```

## Package, Deployment, Git tag, GitHub Release

역할을 분리합니다.

```text
Package / registry tag = 모든 build artifact에 붙임
GitHub Deployment      = staging/production 배포 이력
Git tag                = production에 실제 공개된 version marker
GitHub Release         = Git tag와 1:1인 사람용 release 기록
```

| 대상 | 언제 생성? | 이름 예시 | 의미 |
| --- | --- | --- | --- |
| Package tag | build 성공 시 | `ghcr.io/org/repo:6.2601.143` | 배포 가능한 artifact |
| Package build tag | build 성공 시 | `ghcr.io/org/repo:build-143` | `artifact + build` 중복 방지용 |
| GitHub Deployment | staging/production deploy 시 | `environment=staging` | 환경 배포 이력 |
| Git tag | production deploy 성공 후 | `v6.2601.143` | 사용자 공개 release marker |
| GitHub Release | Git tag 생성 후 | `v6.2601.143` | release note/manifest 기록 |
| Head bump PR | regular production 성공 후 | `head: 6 -> 7` | 다음 release line 시작 |

권장 package tag:

```text
{version}
build-{build}
```

비추천 tag:

```text
latest
staging
production
```

mutable tag는 deploy source of truth로 쓰지 않습니다.
deploy는 `version` 또는 가능하면 `digest` 기준으로 합니다.

Git tag는 production에 실제 공개된 regular/hotfix release에만 만듭니다.
staging build마다 Git tag를 만들지 않습니다.

```text
6.2601.143 staging    -> tag 없음
6.2601.143 production -> v6.2601.143 tag 생성
```

Git tag가 가리키는 commit은 현재 main HEAD가 아니라 artifact manifest의 `git_sha`여야 합니다.

GitHub Release는 Git tag와 1:1로 둡니다.
웹/서버 container라면 binary를 반드시 첨부할 필요는 없고, manifest와 digest만 남겨도 충분합니다.

## Workflow 역할

### `ci.yaml`

PR 검증 전용입니다.

```text
does:
  lint
  test
  typecheck
  build check if needed

does not:
  version 생성
  artifact publish
  staging deploy
  production deploy
```

### `build.yaml`

artifact 생성 workflow입니다.

```text
trigger:
  push to main
  workflow_dispatch

does:
  release.yaml 읽기
  HeadVer 생성
  artifact build
  artifact publish
  manifest publish
  main이면 staging 자동 배포

does not:
  production deploy
  head bump
```

### `deploy.yaml`

기존 artifact를 배포하는 workflow입니다.

```text
trigger:
  workflow_dispatch

inputs:
  version
  environment
  release_kind
  force_redeploy

does:
  artifact 조회
  digest 확인
  target environment 배포
  production approval
  regular production 성공 시 head bump PR 생성
  regular/hotfix production 성공 시 Git tag + GitHub Release 생성

does not:
  source build
  version 재생성
  artifact 덮어쓰기
```

### `bump-head.yaml`

다음 head를 여는 PR을 만듭니다.
main에 직접 commit하지 않습니다.

```text
regular production deploy 성공
-> deployed_head 파싱
-> next_head = deployed_head + 1
-> release.yaml 수정 PR 생성
-> 사람이 merge
```

### `guard-headver.yaml`

head bump 누락을 막습니다.

```text
release.yaml의 versioning.head < NEXT_HEAD이면 CI 실패
```

처음에는 warning으로 시작할 수 있지만 실제 운영에서는 error가 낫습니다.

## Release kind

`deploy.yaml`은 `release_kind`를 받습니다.

| release_kind | 의미 | head bump | Git tag / GitHub Release |
| --- | --- | --- | --- |
| `regular` | 정규 사용자 release | production 성공 후 다음 head PR 생성 | 생성 |
| `hotfix` | 기존 production release line에 대한 긴급 수정 | 없음 | 생성 |
| `rollback` | 이전에 배포했던 artifact로 되돌림 | 없음 | 생성 안 함 |

rollback은 새 build가 아닙니다.
이미 존재하는 artifact를 다시 production에 배포하는 것입니다.

## Regular release flow

```text
1. main merge
2. build.yaml 실행
3. version 생성
4. artifact publish
5. staging 자동 배포
6. QA / smoke test
7. deploy.yaml 수동 실행
   version=<version>
   environment=production
   release_kind=regular
8. production approval
9. production 배포 성공
10. Git tag 생성
11. GitHub Release 생성
12. NEXT_HEAD 설정
13. bump-head PR 자동 생성
14. PR merge
15. 다음 head의 baseline artifact 생성
```

head bump 이후 첫 build는 다음 release line의 baseline artifact입니다.

## Hotfix flow

기본 정책은 기존 production head를 유지하고 hotfix build를 만드는 것입니다.

```text
5.2601.120 -> 5.2601.124
```

정확한 흐름:

```text
1. production에 배포된 version의 git_sha 확인
2. 해당 commit에서 hotfix branch 생성
3. release.yaml의 versioning.head는 기존 head 유지
4. hotfix patch 적용
5. build.yaml을 workflow_dispatch로 hotfix branch에서 실행
6. 새 version 생성
7. staging에 임시 배포
8. smoke test
9. deploy.yaml 실행
   environment=production
   release_kind=hotfix
10. production 배포
11. Git tag 생성
12. GitHub Release 생성
13. head bump 없음
14. hotfix patch만 main에 cherry-pick
15. main build로 staging을 최신 head로 복구
```

hotfix를 main에 반영할 때 hotfix branch를 통째로 merge하지 않습니다.
`release.yaml`의 `versioning.head`가 예전 값으로 되돌아갈 수 있기 때문입니다.
hotfix patch commit만 cherry-pick하거나 PR에서 `release.yaml` 변경을 제외합니다.

## Staging 정책

기본 정책은 staging을 하나만 두는 것입니다.

```text
staging = latest main build
```

hotfix 때는 staging을 잠깐 점유합니다.

```text
평상시:
  staging = 6.x latest

hotfix:
  staging lock
  staging = 5.x hotfix artifact
  smoke test
  production deploy
  hotfix commit cherry-pick to main
  main build
  staging = 6.x latest
  staging unlock
```

긴급 장애라면 local 검증, automated test, 빠른 rollback 준비로 production에 갈 수 있지만 기본 정책은 production 전에 최소 한 번은 실제 artifact를 실행해 보는 것입니다.

## Production approval

production 배포는 GitHub Environment를 사용합니다.

```yaml
environment: production
```

권장 정책:

```text
staging:
  auto deploy allowed

production:
  manual workflow_dispatch
  required reviewers
  prevent self-review 권장
  rebuild forbidden
  version/digest required
```

## GitHub token 정책

`bump-head.yaml`이 PR을 만들 때 기본 `GITHUB_TOKEN`만 쓰면 후속 workflow trigger가 제한될 수 있습니다.
자동화 PR을 일반 PR처럼 다루고 CI가 자연스럽게 돌게 하려면 GitHub App installation token을 권장합니다.

추천 순서:

```text
1. GitHub App installation access token
2. fine-grained PAT
3. classic PAT
```

장수 PAT는 가능하면 피합니다.

## Rollback flow

rollback은 새 build가 아닙니다.

```text
deploy.yaml 실행
  version=5.2601.120
  environment=production
  release_kind=rollback
```

결과:

```text
5.2601.120 production rollback
```

하지 않는 것:

```text
no build
no head bump
no version rewrite
no Git tag
no GitHub Release
```

rollback은 예전에 만든 artifact를 다시 배포하는 행위입니다.

## Monorepo 팁

기본 모델은 단일 repository입니다.
monorepo에서는 앱별 설정과 workflow를 분리합니다.

```text
apps/web/release.yaml
apps/ios/release.yaml
apps/android/release.yaml
```

GitHub Actions의 `github.run_number`는 workflow별 counter입니다.
앱별 version stream이 독립적이면 앱별 build workflow를 분리합니다.

```text
build-web.yaml      -> web 전용 run_number
build-ios.yaml      -> ios 전용 run_number
build-android.yaml  -> android 전용 run_number
```

규칙:

```text
하나의 앱은 하나의 artifact-producing workflow만 가져야 한다.
```

monorepo에서는 app prefix를 붙인 tag를 권장합니다.

```text
web/v6.2601.143
ios/v12.2601.88
android/v9.2601.52
```

GitHub Release의 latest는 repository 전체 기준입니다.
monorepo에서는 앱별 latest source of truth를 별도 release index나 manifest로 둡니다.

## Do / Do Not

### Do

```text
일반 repository에서는 root release.yaml을 둔다.
main merge 시 build하고 staging에 자동 배포한다.
production에는 기존 artifact만 배포한다.
regular production 배포 성공 후 head bump PR을 자동 생성한다.
head bump 누락은 CI guard로 막는다.
hotfix는 production commit에서 branch를 딴다.
hotfix도 같은 build workflow로 빌드한다.
hotfix 배포는 release_kind=hotfix로 한다.
rollback은 기존 artifact를 재배포한다.
build number는 github.run_number + build_offset으로 만든다.
yearweek는 %g%V로 계산한다.
publish 중복 방지는 artifact + build를 primary key로 한다.
production regular/hotfix release에만 Git tag와 GitHub Release를 만든다.
```

### Do Not

```text
production 배포 시 rebuild하지 않는다.
같은 artifact에 다른 version을 붙이지 않는다.
새 artifact에 같은 full version을 재사용하지 않는다.
main에 head bump를 직접 자동 commit하지 않는다.
한 artifact를 여러 build workflow에서 만들지 않는다.
hotfix branch를 main에 통째로 merge해서 head를 되돌리지 않는다.
이미 publish된 artifact를 덮어쓰지 않는다.
rerun으로 artifact publish를 재시도하지 않는다.
yearweek를 %y%V로 계산하지 않는다.
staging build마다 Git tag/GitHub Release를 만들지 않는다.
rollback에서 새 Git tag/GitHub Release를 만들지 않는다.
```

## 최종 요약

```text
version:
  {head}.{yearweek}.{build}

release config:
  release.yaml

head:
  release.yaml의 versioning.head

yearweek:
  TZ={versioning.timezone} date +%g%V

build:
  github.run_number + versioning.build_offset

publish uniqueness:
  primary   = artifact + build
  secondary = artifact + version

package tag:
  ghcr.io/org/repo:{version}
  ghcr.io/org/repo:build-{build}

Git tag:
  v{version}
  production regular/hotfix 성공 후에만 생성
  target은 manifest.git_sha

GitHub Release:
  v{version}
  Git tag와 1:1
  manifest/digest/release_kind 기록

regular release:
  main build
  staging auto deploy
  production manual deploy
  success 후 Git tag/GitHub Release 생성
  success 후 head bump PR auto create

hotfix:
  production commit에서 branch
  팀 정책에 따라 기존 head 유지
  workflow_dispatch로 build
  staging 임시 점유
  production deploy with release_kind=hotfix
  Git tag/GitHub Release 생성
  patch만 main에 cherry-pick

rollback:
  기존 artifact 재배포
  rebuild 없음
  head bump 없음
  새 Git tag/GitHub Release 없음
```

## 참고 문서

- [HeadVer Specification](https://github.com/line/headver)
- [GitHub Actions contexts](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [GitHub Actions GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
- [GitHub Actions deployments and environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments)
- [GitHub Releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- [GNU libc date format specifiers](https://sourceware.org/glibc/manual/html_node/Formatting-Calendar-Time.html)
