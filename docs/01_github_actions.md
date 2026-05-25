# GitHub Actions 기본 가이드

이 문서는 GitHub Actions를 처음 보거나, CI/CD workflow를 직접 고쳐야 하는 사람을 위한 기본 가이드입니다.
이 레포의 reusable workflow를 사용할 때도 같은 개념을 기준으로 보면 됩니다.

## 한 줄 요약

GitHub Actions는 GitHub 이벤트를 트리거로 받아 runner에서 job을 실행하는 자동화 시스템입니다.
workflow는 자동화 파일이고, job은 실행 단위이며, step은 job 안에서 순서대로 실행되는 명령 또는 action입니다.

```text
event
└── workflow
    ├── job: ci
    │   ├── step: checkout
    │   ├── step: install
    │   └── step: test
    └── job: deploy
        ├── step: checkout
        └── step: deploy
```

## 핵심 용어

| 용어 | 의미 | 예시 |
| --- | --- | --- |
| Workflow | 자동화 전체를 정의하는 YAML 파일 | `.github/workflows/ci.yaml` |
| Event | workflow를 실행시키는 트리거 | `push`, `pull_request`, `workflow_dispatch` |
| Job | 같은 runner에서 실행되는 step 묶음 | `test`, `build`, `deploy` |
| Step | job 안에서 순서대로 실행되는 단위 | `run: pnpm test`, `uses: actions/checkout@v6` |
| Action | step에서 재사용하는 외부 또는 내부 자동화 | `actions/setup-node@v6` |
| Runner | job을 실제로 실행하는 머신 | `ubuntu-latest`, self-hosted runner |
| Matrix | 같은 job을 여러 환경 조합으로 반복 실행 | Node 20, 22, 24 |
| Cache | 다음 실행에서 재사용할 의존성 캐시 | npm, pnpm, pip cache |
| Secret | 로그에 노출되면 안 되는 민감값 | 배포 토큰, API key |

## 기본 workflow 구조

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: Node 설정
        uses: actions/setup-node@v6
        with:
          node-version: "22"
          cache: "pnpm"

      - name: Corepack 활성화
        run: corepack enable

      - name: 의존성 설치
        run: pnpm install --frozen-lockfile

      - name: 테스트
        run: pnpm test
```

중요한 흐름은 다음과 같습니다.

- `on`이 실행 시점을 정합니다.
- `permissions`가 `GITHUB_TOKEN` 권한을 제한합니다.
- `jobs` 아래의 각 job은 기본적으로 병렬 실행됩니다.
- job 사이 순서가 필요하면 `needs`를 사용합니다.
- job 안의 `steps`는 위에서 아래로 순서대로 실행됩니다.
- `uses`는 action을 실행하고, `run`은 shell 명령어를 실행합니다.

## Job과 Step

job은 독립된 실행 단위입니다. 같은 job 안의 step은 같은 runner 파일시스템을 공유하므로 앞 step에서 설치한 의존성이나 생성한 파일을 뒤 step에서 사용할 수 있습니다.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
```

job이 나뉘면 runner도 분리됩니다. 앞 job에서 만든 파일이 뒤 job에 자동으로 남아 있지 않으므로, 같은 파일을 바로 써야 하는 작업은 같은 job 안에 두는 편이 단순합니다.

## `needs`로 실행 순서 제어

job은 기본적으로 병렬입니다. 아래 예시는 `lint`, `test`가 병렬로 실행되고, 둘 다 성공한 뒤 `build`가 실행됩니다.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  build:
    runs-on: ubuntu-latest
    needs:
      - lint
      - test
    steps:
      - run: pnpm build
```

이런 job 의존성 그래프를 DAG라고 부르기도 합니다. DAG는 방향이 있는 비순환 그래프라는 뜻이고, GitHub Actions에서는 `needs`로 표현합니다.

```text
lint ─┐
      ├─ build ─ deploy
test ─┘
```

위 구조는 다음처럼 씁니다.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  build:
    runs-on: ubuntu-latest
    needs:
      - lint
      - test
    steps:
      - run: pnpm build

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: pnpm deploy
```

자주 쓰는 패턴은 다음과 같습니다.

### 완전 순차 실행

앞 job이 성공해야 다음 job이 실행됩니다.

```text
lint -> test -> build -> deploy
```

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - run: pnpm test

  build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - run: pnpm build

  deploy:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: pnpm deploy
```

### 병렬 실행 후 합치기

서로 독립적인 검사는 병렬로 돌리고, 모두 성공했을 때만 다음 job을 실행합니다.

```text
lint ─────┐
typecheck ├─ build
test ─────┘
```

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm typecheck

  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  build:
    runs-on: ubuntu-latest
    needs:
      - lint
      - typecheck
      - test
    steps:
      - run: pnpm build
```

### 실패해도 실행해야 하는 job

기본적으로 `needs`에 걸린 job 중 하나라도 실패하면 뒤 job은 실행되지 않습니다. 실패 여부와 상관없이 리포트 정리나 알림 job을 실행해야 하면 `if: ${{ always() }}`를 씁니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  report:
    runs-on: ubuntu-latest
    needs: test
    if: ${{ always() }}
    steps:
      - run: pnpm report
```

앞 job의 결과에 따라 분기할 수도 있습니다.

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - run: pnpm test

  notify-failure:
    runs-on: ubuntu-latest
    needs: test
    if: ${{ failure() }}
    steps:
      - run: echo "테스트 실패"
```

운영 기준은 단순합니다.

- 서로 독립적인 job은 병렬로 둡니다.
- 실제 의존성이 있을 때만 `needs`를 추가합니다.
- 배포 job은 보통 `lint`, `test`, `build`가 모두 성공한 뒤 실행합니다.
- 정리, 리포트, 알림처럼 실패해도 실행해야 하는 job에는 `if: ${{ always() }}`를 명시합니다.

## Matrix

matrix는 같은 job을 여러 버전이나 OS 조합으로 반복 실행할 때 씁니다.

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        os:
          - ubuntu-latest
          - ubuntu-24.04

    steps:
      - uses: actions/checkout@v6
      - run: npm ci
      - run: npm test
```

서비스 프로젝트에서는 운영 버전 하나로 고정하는 편이 CI 신호가 단순합니다. 여러 runtime 지원을 검증해야 하는 라이브러리 프로젝트에서만 matrix를 고려합니다.

## Action과 `uses`

step은 크게 두 종류입니다.

- `run`: runner에서 shell 명령어를 직접 실행합니다.
- `uses`: 이미 만들어진 action을 가져와 실행합니다.

```yaml
steps:
  - name: 명령어 실행
    run: pnpm test

  - name: action 실행
    uses: actions/checkout@v6
```

`actions/checkout@v6`는 다음처럼 읽습니다.

```text
actions/checkout@v6
└────── └─────── └─ 사용할 버전 또는 Git ref
   │       │
   │       └─ action 이름
   └─ GitHub owner 또는 organization
```

즉 `actions/checkout@v6`는 GitHub의 `actions` organization에 있는 `checkout` action을 `v5` 버전으로 사용한다는 뜻입니다.

대표적인 action은 다음과 같습니다.

| Action | 역할 | 이 레포에서 쓰는 위치 |
| --- | --- | --- |
| `actions/checkout@v6` | runner에 현재 repository 코드를 내려받습니다. 대부분의 workflow 첫 step에 필요합니다. | CI/CD 공통 |
| `actions/setup-node@v6` | Node.js를 설치하고 npm, pnpm, yarn cache를 설정합니다. | Node CI/CD |
| `actions/setup-python@v6` | Python을 설치하고 pip, pipenv, poetry cache를 설정합니다. | Python CI/CD |

### `actions/checkout`이 필요한 이유

GitHub Actions runner는 job이 시작될 때 빈 작업 공간에 가깝습니다. repository 코드가 자동으로 들어와 있지 않기 때문에, 보통 먼저 checkout을 해야 합니다.

```yaml
steps:
  - name: 코드 체크아웃
    uses: actions/checkout@v6

  - name: package.json 확인
    run: pnpm install --frozen-lockfile
```

checkout 없이 `pnpm install`, `pytest`, `docker build .` 같은 명령을 실행하면 필요한 소스 파일이 없어서 실패합니다.

checkout은 옵션으로 특정 ref나 submodule 처리도 설정할 수 있습니다.

```yaml
- uses: actions/checkout@v6
  with:
    ref: main
    fetch-depth: 0
    submodules: recursive
```

자주 쓰는 옵션은 다음과 같습니다.

| 옵션 | 의미 | 언제 쓰나 |
| --- | --- | --- |
| `fetch-depth: 0` | 전체 Git history를 가져옵니다. | 변경 이력, 태그, semantic release가 필요할 때 |
| `ref` | checkout할 branch, tag, SHA를 지정합니다. | 특정 ref 기준으로 빌드해야 할 때 |
| `submodules` | Git submodule도 같이 가져옵니다. | submodule을 사용하는 레포 |
| `persist-credentials: false` | checkout 후 Git credential을 남기지 않습니다. | 보안 민감 workflow |

### `setup-node`와 `setup-python`

언어 runtime을 직접 설치하지 않고 setup action을 쓰면 버전과 cache 설정을 표준화할 수 있습니다.

```yaml
- uses: actions/setup-node@v6
  with:
    node-version: "22"
    cache: "pnpm"
    cache-dependency-path: "pnpm-lock.yaml"
```

```yaml
- uses: actions/setup-python@v6
  with:
    python-version: "3.12"
    cache: "pip"
```

setup action은 runtime만 준비합니다. 의존성 설치는 별도 step에서 실행해야 합니다.

```yaml
- run: pnpm install --frozen-lockfile
- run: uv sync --frozen
```

### Action 버전 고정

외부 action은 항상 버전을 붙여서 사용합니다.

```yaml
- uses: actions/checkout@v6
```

다음처럼 `@main`을 쓰면 action 코드가 바뀔 때 workflow 동작도 예고 없이 바뀔 수 있습니다.

```yaml
- uses: actions/checkout@main
```

일반 workflow는 major version tag를 고정하는 방식이 흔하고, 보안이 중요한 workflow는 commit SHA로 더 강하게 고정할 수 있습니다.

## Reusable Workflow

reusable workflow는 여러 레포에서 공통 workflow를 job 단위로 재사용하는 방식입니다.

호출되는 workflow에는 `on.workflow_call`이 있어야 합니다.

```yaml
name: Node.js CI

on:
  workflow_call:
    inputs:
      node-version:
        required: false
        type: string
        default: "22"
```

호출하는 쪽에서는 `steps` 안이 아니라 `jobs.<job_id>.uses`로 호출합니다.

```yaml
jobs:
  node-ci:
    uses: wibaek/github-automation/.github/workflows/node-ci.yaml@v1.0
    with:
      node-version: "22"
      package-manager: "pnpm"
```

secrets가 필요하면 호출하는 쪽에서 명시적으로 넘깁니다.

```yaml
jobs:
  deploy:
    uses: wibaek/github-automation/.github/workflows/ssh-compose-deploy.yaml@v1.0
    with:
      host: ${{ vars.DEPLOY_HOST }}
      user: ${{ vars.DEPLOY_USER }}
      project-directory: "/srv/my-app"
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_PRIVATE_KEY }}
```

## CI 예시

Node 프로젝트에서 pull request와 main push마다 검사하는 예시입니다.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  node-ci:
    uses: wibaek/github-automation/.github/workflows/node-ci.yaml@v1.0
    with:
      node-version: "22"
      package-manager: "pnpm"
      pnpm-version: "10"
      lint-command: "auto"
      typecheck-command: "auto"
      test-command: "auto"
      build-command: "auto"
```

Python 프로젝트에서 uv를 사용하는 예시입니다.

```yaml
name: CI

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  python-ci:
    uses: wibaek/github-automation/.github/workflows/python-ci.yaml@v1.0
    with:
      python-version: "3.12"
      python-package-tool: "uv"
      lint-command: "uv run ruff check ."
      test-command: "auto"
      cache-dependency-path: "uv.lock"
```

## CD 예시

배포 workflow는 보통 main 브랜치 push 또는 수동 실행으로 제한합니다.

```yaml
name: CD

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  docker-build-push:
    uses: wibaek/github-automation/.github/workflows/docker-build-push.yaml@v1.0
    with:
      image-name: "ghcr.io/${{ github.repository }}"

  ssh-compose-deploy:
    needs: docker-build-push
    uses: wibaek/github-automation/.github/workflows/ssh-compose-deploy.yaml@v1.0
    with:
      host: ${{ vars.DEPLOY_HOST }}
      user: ${{ vars.DEPLOY_USER }}
      environment: "prod"
      environment-url: "https://example.com"
      project-directory: "/srv/my-app"
      image-reference: ${{ needs.docker-build-push.outputs.image-reference }}
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_SSH_PRIVATE_KEY }}
```

dev, stage, prod는 같은 reusable workflow를 호출하되 `environment`, host, path, cluster, service 같은 input만 환경별로 바꿉니다.

같은 environment에 여러 배포가 겹치면 같은 concurrency group으로 묶습니다. prod는 중간 취소보다 순서대로 처리하는 쪽이 보수적이고, dev/stage는 최신 실행만 남기고 이전 실행을 취소해도 됩니다.

```yaml
concurrency:
  group: deploy-${{ github.repository }}-prod
  cancel-in-progress: false
```

## Cache

의존성은 `setup-node`, `setup-python`의 cache 옵션을 먼저 고려합니다.

```yaml
- uses: actions/setup-node@v6
  with:
    node-version: "22"
    cache: "pnpm"
    cache-dependency-path: "pnpm-lock.yaml"
```

## Secrets와 Variables

민감값은 코드에 쓰지 말고 GitHub Actions secret으로 관리합니다.

```yaml
env:
  API_TOKEN: ${{ secrets.API_TOKEN }}
```

일반 설정값은 secret이 아니라 repository variable 또는 workflow input으로 두는 편이 낫습니다.

```yaml
env:
  NODE_ENV: production
  PUBLIC_API_URL: ${{ vars.PUBLIC_API_URL }}
```

주의할 점은 다음과 같습니다.

- secret 값을 `echo`로 출력하지 않습니다.
- JSON처럼 구조화된 값을 secret에 넣으면 마스킹이 어긋날 수 있으므로 가능하면 작은 값으로 나눕니다.
- fork PR에서는 secret 접근이 제한될 수 있습니다.
- 배포 secret은 environment secret으로 분리하고 승인 규칙을 붙이는 것이 좋습니다.

## 권한 관리

기본 원칙은 최소 권한입니다.

```yaml
permissions:
  contents: read
```

PR 코멘트 작성이 필요한 job만 `pull-requests: write`를 추가합니다. 배포 job도 외부 CLI만 실행한다면 보통 `contents: read`로 충분하고, GitHub Deployment API를 직접 만들거나 갱신하는 job에만 `deployments: write`를 추가합니다.

```yaml
jobs:
  deploy:
    permissions:
      contents: read
      deployments: write
```

권한을 workflow 전체에 크게 열어두면 실수나 악성 PR이 영향을 줄 수 있습니다. 쓰기 권한은 필요한 job에만 좁게 주는 쪽을 기본값으로 둡니다.

## 보안 Best Practice

- `permissions`는 명시하고, 기본은 `contents: read`로 둡니다.
- 외부 action은 `actions/checkout@v6`처럼 버전을 고정합니다. 보안 민감 workflow는 SHA pinning도 고려합니다.
- secret을 로그에 출력하지 않습니다.
- untrusted input을 shell에 직접 꽂지 않습니다.
- `pull_request_target`은 신뢰할 수 없는 PR 코드와 함께 쓰지 않습니다.
- 배포에는 environment protection rule을 붙여 승인, branch 제한, environment secret을 활용합니다.
- 클라우드 인증은 가능하면 장기 secret보다 OIDC 기반 단기 토큰을 사용합니다.
- self-hosted runner는 외부 기여자 PR에 노출하지 않는 것이 안전합니다.

위험한 예시입니다.

```yaml
- name: 위험한 입력 사용
  run: echo "${{ github.event.pull_request.title }}"
```

PR 제목 같은 값은 외부 사용자가 조작할 수 있습니다. shell 명령으로 넘겨야 한다면 환경 변수로 분리하고 quoting을 신중하게 처리합니다.

```yaml
- name: 입력을 환경 변수로 전달
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: |
    printf '%s\n' "$PR_TITLE"
```

## 운영 Best Practice

- CI와 CD workflow를 분리합니다.
- CI는 `pull_request`와 `push`에서 빠르게 돌리고, CD는 `main` push 또는 `workflow_dispatch`로 제한합니다.
- job 이름은 UI에서 원인을 바로 알 수 있게 `lint`, `test`, `build`, `deploy`처럼 짧고 명확하게 둡니다.
- `working-directory`는 monorepo에서 명시적으로 지정합니다.
- 실패 원인이 잘 보이도록 하나의 step에 너무 많은 일을 넣지 않습니다.
- 반복되는 설치, 검증, 배포 흐름은 reusable workflow로 뺍니다.
- 배포 job에는 `concurrency`를 설정해 같은 환경에 동시에 배포되지 않게 합니다.
- 오래 걸리는 job에는 `timeout-minutes`를 둡니다.
- cron schedule은 UTC 기준이므로 한국 시간 기준 작업은 시간 변환을 명시합니다.

## 자주 하는 실수

### Reusable workflow를 step에서 호출함

잘못된 예시입니다.

```yaml
steps:
  - uses: wibaek/github-automation/.github/workflows/node-ci.yaml@v1.0
```

reusable workflow는 job에서 호출해야 합니다.

```yaml
jobs:
  node-ci:
    uses: wibaek/github-automation/.github/workflows/node-ci.yaml@v1.0
```

### job 사이에서 파일이 이어진다고 생각함

job이 바뀌면 runner도 바뀝니다. 같은 파일을 바로 써야 하는 작업은 같은 job 안에서 처리하는 편이 단순합니다.

### 모든 workflow에 쓰기 권한을 줌

대부분의 CI는 `contents: read`면 충분합니다. 쓰기 권한은 필요한 job에만 추가합니다.

### 브랜치와 태그를 고정하지 않음

공용 workflow는 `@main`보다 `@v1.0`처럼 버전 태그를 고정해서 재현성을 확보합니다.

## 이 레포 기준 추천 패턴

이 레포는 여러 프로젝트에서 공통으로 사용할 reusable workflow를 모아두는 레포입니다.

- 프로젝트별 레포에는 얇은 workflow만 둡니다.
- 실제 설치, 검증, 빌드, 배포 로직은 이 레포의 reusable workflow에 둡니다.
- 호출은 항상 `jobs.<job_id>.uses`로 합니다.
- 공용 workflow 참조는 SemVer 태그로 고정합니다.
- 배포 방식이 다르면 Docker 이미지 빌드, SSH 배포처럼 목적별 workflow로 분리합니다.

## 참고 문서

- [Understanding GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)
- [Workflows](https://docs.github.com/actions/concepts/workflows-and-actions/workflows)
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Reuse workflows](https://docs.github.com/en/actions/how-tos/sharing-automations/reusing-workflows)
- [Dependency caching](https://docs.github.com/en/actions/concepts/workflows-and-actions/dependency-caching)
- [Using secrets in GitHub Actions](https://docs.github.com/en/actions/how-tos/administering-github-actions/sharing-workflows-secrets-and-runners-with-your-organization)
- [Security for GitHub Actions](https://docs.github.com/en/actions/how-tos/security-for-github-actions)
