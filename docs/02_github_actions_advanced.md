# GitHub Actions 고급 가이드

이 문서는 기본 workflow 구조를 알고 있다는 전제에서, Docker 이미지 빌드처럼 시간이 오래 걸리거나 운영 영향이 큰 workflow를 다룹니다.

## Docker 이미지 빌드 캐시

GitHub-hosted runner는 매번 새 runner에서 시작하므로 로컬 Docker layer cache가 기본적으로 남아 있지 않습니다.
Docker Buildx의 cache backend를 명시해야 다음 workflow run에서 이전 빌드 레이어를 재사용할 수 있습니다.

가장 많이 쓰는 방식은 두 가지입니다.

| 방식 | 예시 | 특징 |
| --- | --- | --- |
| GitHub Actions cache | `cache-from: type=gha` | 설정이 단순하고 GitHub Actions 안에서 쓰기 좋음 |
| Registry cache | `cache-from: type=registry,ref=...:buildcache` | registry에 cache manifest를 저장해서 더 명시적으로 관리 가능 |

## 이 레포의 Docker reusable workflow

이 레포에서는 Docker 이미지 빌드와 서버 compose, ECS 배포를 reusable workflow로 제공합니다.

| Workflow | 용도 |
| --- | --- |
| `.github/workflows/docker-build-push.yaml` | Docker Buildx 기반 이미지 빌드, cache, registry push |
| `.github/workflows/ssh-compose-deploy.yaml` | SSH 접속 후 `docker compose pull`, `docker compose up -d --no-build`, healthcheck |
| `.github/workflows/ecs-deploy.yaml` | ECS task definition 렌더링 후 service update |

GHCR에 이미지를 push하는 최소 예시는 다음과 같습니다.

```yaml
jobs:
  docker-build-push:
    uses: wibaek/github-automation/.github/workflows/docker-build-push.yaml@v1.0
    with:
      image-name: "ghcr.io/${{ github.repository }}"
      context: "."
      dockerfile: "./Dockerfile"
      cache-type: "gha"
      cache-scope: "app"
```

빌드한 image reference를 서버 배포에 넘기는 예시는 다음과 같습니다.

```yaml
jobs:
  docker-build-push:
    uses: wibaek/github-automation/.github/workflows/docker-build-push.yaml@v1.0
    with:
      image-name: "ghcr.io/${{ github.repository }}"
      cache-scope: "app"

  ssh-compose-deploy:
    needs: docker-build-push
    uses: wibaek/github-automation/.github/workflows/ssh-compose-deploy.yaml@v1.0
    with:
      host: ${{ vars.OCI_HOST }}
      user: ${{ vars.OCI_USER }}
      known-hosts: ${{ vars.OCI_KNOWN_HOSTS }}
      project-directory: "/srv/my-app"
      registry-login: true
      image-reference: ${{ needs.docker-build-push.outputs.image-reference }}
      healthcheck-url: "http://127.0.0.1:8000/health"
    secrets:
      SSH_PRIVATE_KEY: ${{ secrets.OCI_SSH_PRIVATE_KEY }}
      REGISTRY_PASSWORD: ${{ secrets.GHCR_READ_TOKEN }}
```

서버 compose 파일에서는 `IMAGE_TAG`, `IMAGE_DIGEST`, `IMAGE_REF` 환경 변수를 사용할 수 있습니다. 가능하면 `IMAGE_REF` 하나만 쓰는 편이 가장 단순합니다.

## GHCR에 이미지 빌드 및 푸시

GitHub Container Registry에 Docker 이미지를 푸시하면서 GitHub Actions cache를 쓰는 예시입니다.

```yaml
name: Docker 이미지 배포

on:
  push:
    branches:
      - main
    tags:
      - "v*"
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  docker:
    name: Docker 이미지 빌드 및 푸시
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v4

      - name: GHCR 로그인
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker 이미지 메타데이터 생성
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=tag
            type=sha

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v7
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=app
          cache-to: type=gha,mode=max,scope=app
```

핵심은 마지막 step의 `cache-from`, `cache-to`입니다.

```yaml
cache-from: type=gha,scope=app
cache-to: type=gha,mode=max,scope=app
```

- `cache-from`은 이전 workflow run에서 저장한 cache를 가져옵니다.
- `cache-to`는 이번 빌드 결과를 다음 run에서 쓸 수 있게 저장합니다.
- `mode=max`는 가능한 많은 cache metadata를 내보냅니다.
- `scope=app`은 cache 이름입니다. 한 workflow에서 여러 이미지를 빌드하면 이미지마다 다른 scope를 써야 서로 덮어쓰지 않습니다.

## Pull Request에서는 빌드만 하고 push는 막기

외부 PR이나 검증용 workflow에서는 이미지를 registry에 푸시하지 않는 편이 안전합니다.

```yaml
name: Docker 이미지 검사

on:
  pull_request:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:
  docker:
    name: Docker 이미지 빌드 검사
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v4

      - name: GHCR 로그인
        if: ${{ github.event_name != 'pull_request' }}
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker 이미지 메타데이터 생성
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha

      - name: Docker 이미지 빌드
        uses: docker/build-push-action@v7
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha,scope=app
          cache-to: type=gha,mode=max,scope=app,ignore-error=true
```

`ignore-error=true`는 cache export 실패가 전체 빌드 실패로 이어지지 않게 합니다. 빌드 결과 자체가 중요한 workflow에서는 cache를 최적화로만 취급하는 편이 운영상 안전합니다.

## Registry cache 사용

GitHub Actions cache 대신 registry에 cache manifest를 저장할 수도 있습니다.
cache를 이미지와 같은 registry에서 관리하고 싶거나, cache를 GitHub Actions cache 제한과 분리하고 싶을 때 쓸 수 있습니다.

```yaml
name: Docker 이미지 배포

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:
  docker:
    name: Registry cache로 Docker 이미지 빌드
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v4

      - name: GHCR 로그인
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache
          cache-to: type=registry,ref=ghcr.io/${{ github.repository }}:buildcache,mode=max
```

Registry cache는 `buildcache` 같은 별도 tag를 사용합니다. 이 tag는 실행용 이미지 tag가 아니라 빌드 cache 저장용입니다.

## Docker Hub로 푸시하는 경우

Docker Hub를 쓰면 login 정보만 바뀝니다.

```yaml
name: Docker 이미지 배포

on:
  push:
    branches:
      - main

permissions:
  contents: read

jobs:
  docker:
    name: Docker Hub 이미지 빌드 및 푸시
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v4

      - name: Docker Hub 로그인
        uses: docker/login-action@v4
        with:
          username: ${{ vars.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v7
        with:
          context: .
          push: true
          tags: ${{ vars.DOCKERHUB_USERNAME }}/my-app:latest
          cache-from: type=gha,scope=my-app
          cache-to: type=gha,mode=max,scope=my-app
```

Docker Hub 토큰은 password가 아니라 access token을 secret으로 저장해서 쓰는 편이 좋습니다.

## Multi-platform 이미지

`linux/amd64`, `linux/arm64`를 같이 빌드하려면 QEMU와 Buildx를 설정하고 `platforms`를 넘깁니다.

```yaml
name: Docker 멀티 플랫폼 이미지 배포

on:
  push:
    branches:
      - main

permissions:
  contents: read
  packages: write

jobs:
  docker:
    name: Docker 멀티 플랫폼 이미지 빌드 및 푸시
    runs-on: ubuntu-latest

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v6

      - name: QEMU 설정
        uses: docker/setup-qemu-action@v4

      - name: Docker Buildx 설정
        uses: docker/setup-buildx-action@v4

      - name: GHCR 로그인
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker 이미지 빌드 및 푸시
        uses: docker/build-push-action@v7
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha,scope=app-multi-platform
          cache-to: type=gha,mode=max,scope=app-multi-platform
```

멀티 플랫폼 빌드는 단일 플랫폼보다 오래 걸립니다. 필요한 플랫폼만 지정하고, 정말 느리면 플랫폼별 job matrix로 나눈 뒤 manifest를 합치는 구조를 검토합니다.

## Dockerfile도 cache 친화적으로 작성하기

GitHub Actions cache를 켜도 Dockerfile 순서가 나쁘면 cache hit가 잘 나지 않습니다.
자주 바뀌는 소스 전체를 먼저 복사하면 의존성 설치 layer가 매번 깨집니다.

좋은 패턴은 lockfile과 manifest를 먼저 복사하고 의존성을 설치한 뒤, 그 다음 소스 파일을 복사하는 방식입니다.

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app

COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile

FROM node:22-alpine AS build
WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN corepack enable && pnpm build
```

`.dockerignore`도 같이 관리합니다.

```gitignore
.git
node_modules
dist
.next
.env
```

## 선택 기준

처음에는 GitHub Actions cache를 추천합니다.

```yaml
cache-from: type=gha,scope=app
cache-to: type=gha,mode=max,scope=app
```

다음 상황이면 registry cache를 검토합니다.

- 여러 workflow나 runner에서 같은 cache를 더 명시적으로 공유해야 합니다.
- GitHub Actions cache 제한이나 eviction 영향을 줄이고 싶습니다.
- cache를 이미지 registry의 lifecycle policy로 관리하고 싶습니다.

## 주의할 점

- GitHub Actions cache backend는 Buildx/BuildKit이 필요합니다.
- GitHub-hosted runner에서 `docker/build-push-action`을 쓰면 Buildx/BuildKit은 보통 최신 상태입니다.
- self-hosted runner에서는 Buildx, BuildKit, Docker Engine 버전을 직접 관리해야 합니다.
- cache는 성능 최적화일 뿐입니다. cache miss가 나도 빌드가 성공해야 정상입니다.
- secret이나 `.env`를 Docker build context에 넣지 않도록 `.dockerignore`를 관리합니다.
- PR에서는 push를 막고 build 검증만 하는 구성을 기본으로 둡니다.

## 참고 문서

- [Docker Build GitHub Actions](https://docs.docker.com/build/ci/github-actions/)
- [Docker cache management with GitHub Actions](https://docs.docker.com/build/ci/github-actions/cache/)
- [Docker GitHub Actions cache backend](https://docs.docker.com/build/cache/backends/gha/)
- [Docker multi-platform image with GitHub Actions](https://docs.docker.com/build/ci/github-actions/multi-platform/)
- [GitHub Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
