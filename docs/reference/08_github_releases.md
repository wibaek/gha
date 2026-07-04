# GitHub Releases 정리

이 문서는 GitHub repository의 Releases 기능을 설명합니다.

## 한 줄 정의

GitHub Release는 특정 git tag를 기준으로 소프트웨어 버전을 배포 가능한 형태로 묶어 공개하는 GitHub 리소스입니다.
release에는 제목, 설명, release note, source archive 링크, 첨부 asset, prerelease/latest 표시가 붙을 수 있습니다.

```text
git tag
  -> repository history의 특정 commit을 가리킴

GitHub Release
  -> tag를 기준으로 사용자에게 보여 줄 배포 페이지를 만듦
  -> release note, asset, source archive, latest/prerelease 표시를 포함
```

## Release와 tag의 차이

Git tag는 Git object입니다.
특정 commit을 가리키는 version marker이고, GitHub 바깥에서도 `git fetch --tags`, `git checkout v1.2.3`처럼 사용할 수 있습니다.

GitHub Release는 GitHub가 tag 위에 얹는 배포 페이지입니다.
release note를 작성하고, 빌드된 파일을 asset으로 첨부하고, 사용자가 다운로드할 수 있는 형태로 보여 줍니다.

| 구분 | Tag | Release |
| --- | --- | --- |
| 성격 | Git ref | GitHub repository 리소스 |
| 기준 | commit | tag |
| 주요 용도 | version 지점 고정 | 배포 공지, 다운로드, release note |
| 생성 위치 | Git CLI, GitHub UI, API | GitHub UI, GitHub CLI, API |
| 첨부 파일 | 없음 | release asset 첨부 가능 |

release를 만들 때 기존 tag를 선택할 수도 있고, GitHub UI에서 새 tag를 만들 수도 있습니다.
tag 생성일과 release 게시일은 다를 수 있습니다.

## Release에 들어가는 것

release는 보통 아래 요소로 구성됩니다.

| 요소 | 설명 |
| --- | --- |
| Tag | release가 기준으로 삼는 git tag |
| Target | 새 tag를 만들 때 기준이 되는 branch 또는 commit |
| Release title | 사용자에게 보이는 release 제목 |
| Release note | 변경 사항, migration 안내, breaking change, contributor mention |
| Source archive | GitHub가 자동으로 제공하는 source code zip/tar.gz |
| Asset | 직접 업로드한 binary, installer, checksum, SBOM 같은 파일 |
| Prerelease 표시 | production stable이 아닌 사전 배포 표시 |
| Latest 표시 | repository의 최신 release로 표시할지 여부 |

GitHub는 release tag 시점의 source code archive 링크를 자동으로 제공합니다.
빌드 산출물, installer, checksum, SBOM처럼 source archive만으로 부족한 파일은 release asset으로 올립니다.

## Draft, prerelease, latest

release에는 상태와 표시 방식이 있습니다.

| 상태 | 의미 |
| --- | --- |
| Draft | 아직 공개하지 않은 release 초안 |
| Published | 사용자에게 공개된 release |
| Prerelease | production 사용을 권장하지 않는 preview, beta, rc release |
| Latest | repository에서 최신 stable release처럼 강조되는 release |

asset을 모두 붙인 뒤 공개해야 하면 draft로 먼저 만들고 확인 후 publish합니다.
repository에서 immutable releases를 켰다면 draft로 준비한 뒤 publish하는 흐름이 더 중요합니다.

prerelease는 `v2.0.0-beta.1`, `v2.0.0-rc.1` 같은 사전 버전에 붙입니다.
사용자에게 안정 버전이 아니라는 신호를 주고, latest release 계산에서도 stable release와 다르게 취급됩니다.

latest release는 GitHub가 semantic version을 기준으로 자동 판단할 수 있지만, release 생성 화면에서 직접 지정할 수도 있습니다.
문서와 배포 자동화가 latest release를 참조한다면 prerelease와 latest 표시를 분리해서 관리합니다.

## Release note

release note는 사용자가 실제로 읽는 변경 내역입니다.
단순 commit 목록보다 아래 정보를 명확히 적는 편이 좋습니다.

- 무엇이 바뀌었는가
- 사용자가 해야 할 migration이 있는가
- breaking change가 있는가
- 보안 수정이 포함되어 있는가
- 새 asset이나 설치 방법이 바뀌었는가

GitHub UI에는 Generate release notes 기능이 있습니다.
이 기능은 이전 tag 이후 merge된 pull request, contributor, full changelog 링크를 기반으로 초안을 만듭니다.

자동 생성 release note는 label 기준 category와 제외 규칙을 설정할 수 있습니다.
공식 문서 기준 설정 파일명은 `.github/release.yml`입니다.

```yaml
changelog:
  exclude:
    labels:
      - ignore-for-release
  categories:
    - title: Breaking Changes
      labels:
        - breaking-change
    - title: Features
      labels:
        - enhancement
    - title: Other Changes
      labels:
        - "*"
```

자동 생성 결과는 초안입니다.
실제 release 전에는 빠진 migration 안내, breaking change, 보안 영향, 운영 주의사항을 사람이 확인합니다.

## Release asset

release asset은 release에 직접 업로드하는 파일입니다.
예시는 다음과 같습니다.

- macOS, Windows, Linux용 binary
- mobile app archive
- CLI tarball
- checksum 파일
- SBOM
- migration guide PDF

GitHub는 release 하나에 최대 1000개 asset을 연결할 수 있고, 각 파일은 2GiB보다 작아야 합니다.
release 전체 크기나 bandwidth에는 별도 제한이 없다고 설명합니다.

source code zip/tar.gz는 GitHub가 자동으로 제공합니다.
하지만 빌드 결과물과 source archive는 다릅니다.
사용자가 실행할 파일이 필요하면 release asset으로 별도 업로드합니다.

## 보안 release

release가 보안 취약점을 수정한다면 release note만 쓰고 끝내지 않습니다.
GitHub Security Advisory를 함께 게시할지 검토합니다.

보안 advisory를 게시하면 GitHub가 검토 후 영향을 받는 repository에 Dependabot alert를 보낼 수 있습니다.
library나 public package라면 CVE, 영향 버전, patched version, workaround를 별도로 정리하는 편이 좋습니다.

민감한 취약점은 공개 release note보다 advisory와 coordinated disclosure 절차가 먼저입니다.

## 만드는 방법

GitHub UI에서는 repository 오른쪽의 Releases에서 Draft a new release를 눌러 만듭니다.
기본 흐름은 다음과 같습니다.

1. tag를 선택하거나 새 tag 이름을 입력합니다.
2. 새 tag라면 target branch 또는 commit을 선택합니다.
3. 이전 tag를 확인합니다.
4. release title을 작성합니다.
5. release note를 직접 작성하거나 자동 생성합니다.
6. 필요한 asset을 업로드합니다.
7. prerelease, latest, discussion 연결 여부를 선택합니다.
8. Publish release 또는 Save draft를 선택합니다.

GitHub CLI로도 만들 수 있습니다.

```bash
gh release create v1.2.3 --title "v1.2.3" --notes "Release notes"
```

prerelease 예시는 다음과 같습니다.

```bash
gh release create v2.0.0-beta.1 --title "v2.0.0 beta 1" --notes "Preview release" --prerelease
```

## Actions와 연결할 때

GitHub Release 자체는 Actions 기능이 아닙니다.
다만 release 생성, publish, prerelease 같은 이벤트를 Actions trigger로 사용할 수 있고, Actions에서 release asset을 업로드할 수도 있습니다.

```yaml
on:
  release:
    types:
      - published
```

주의할 점은 자동화 token입니다.
`GITHUB_TOKEN`으로 만든 release 이벤트는 다른 workflow를 다시 trigger하지 않을 수 있습니다.
release publish를 기준으로 별도 배포 workflow를 반드시 실행해야 한다면 token 종류와 trigger 구조를 확인해야 합니다.

이 repository의 `release.yaml`은 release-please 기반 자동화 workflow입니다.
그 문서는 GitHub Release 기능 자체가 아니라 Release PR, tag, GitHub Release 생성을 자동화하는 방법으로 봅니다.

## 운영 기준

- release는 tag를 기준으로 만들고, 이미 공개한 tag는 가능한 한 움직이지 않습니다.
- stable release와 prerelease를 구분합니다.
- release note에는 사용자 영향, migration, breaking change를 사람이 확인해서 남깁니다.
- 실행 파일이나 checksum은 source archive에 기대지 말고 release asset으로 올립니다.
- 보안 수정은 GitHub Security Advisory와 함께 관리할지 검토합니다.
- release 자동화는 GitHub Release 기능 위에 얹는 선택지로 봅니다.
- public project는 release note 품질이 사용자 문서의 일부라고 보고 관리합니다.

## 자주 헷갈리는 점

tag만 만들었다고 release가 생기는 것은 아닙니다.
tag는 Git ref이고, release는 GitHub UI/API에 별도로 생기는 배포 리소스입니다.

release를 삭제해도 tag가 항상 같이 삭제되는 것은 아닙니다.
release와 tag를 별개로 보고 정리해야 합니다.

source code zip/tar.gz는 빌드 산출물이 아닙니다.
컴파일된 binary나 installer가 필요하면 release asset으로 올립니다.

prerelease는 단순히 이름에 `beta`가 들어가는 것과 별개입니다.
GitHub release의 prerelease 표시를 켜야 사용자와 자동화가 명확하게 구분할 수 있습니다.

latest release는 항상 가장 최근 게시된 release와 같다고 가정하면 안 됩니다.
semantic version, prerelease 여부, 수동 latest 지정에 따라 다르게 보일 수 있습니다.

## 점검 체크리스트

- tag가 올바른 commit을 가리키는가?
- release title과 tag version이 일관적인가?
- 이전 tag 기준 changelog가 맞는가?
- breaking change와 migration 안내가 release note에 들어갔는가?
- prerelease라면 prerelease 표시가 켜져 있는가?
- stable release라면 latest 표시가 의도대로 되는가?
- 필요한 binary, checksum, SBOM이 asset으로 올라갔는가?
- 보안 수정이면 Security Advisory가 필요한가?
- Actions가 release 이벤트를 기준으로 동작한다면 token과 trigger 구조가 맞는가?

## 참고 문서

- [About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases)
- [Managing releases in a repository](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository)
- [Viewing your repository's releases and tags](https://docs.github.com/en/repositories/releasing-projects-on-github/viewing-your-repositorys-releases-and-tags)
- [Automatically generated release notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes)
- [REST API endpoints for releases](https://docs.github.com/en/rest/releases/releases)
