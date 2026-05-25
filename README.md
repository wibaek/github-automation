# github-automation

여러 프로젝트에서 공통으로 사용할 GitHub Actions reusable workflow 모음입니다.

프로젝트별 repository에서는 workflow 구현을 길게 두지 않고, 여기 있는 workflow를 `jobs.<job_id>.uses`로 호출합니다. `steps` 안에서 호출하는 방식이 아닙니다.

## Workflows

```text
github-automation
└── .github
    └── workflows
        ├── python-ci.yaml
        ├── node-ci.yaml
        ├── cloudflare-pages-deploy.yaml
        ├── cloudflare-workers-deploy.yaml
        ├── docker-build-push.yaml
        ├── release.yaml
        ├── ecs-deploy.yaml
        └── ssh-compose-deploy.yaml
```

| Workflow | 용도 |
| --- | --- |
| `python-ci.yaml` | Python CI |
| `node-ci.yaml` | Node.js CI |
| `cloudflare-pages-deploy.yaml` | Cloudflare Pages deploy |
| `cloudflare-workers-deploy.yaml` | Cloudflare Workers deploy |
| `docker-build-push.yaml` | Docker image build/push |
| `release.yaml` | Release PR, tag, GitHub Release, CHANGELOG |
| `ecs-deploy.yaml` | ECS service deploy |
| `ssh-compose-deploy.yaml` | SSH Docker Compose deploy |

## Recommended Flow

일반적으로 PR에서는 CI만 돌리고, `main` merge 이후에 배포 workflow를 실행합니다.

```mermaid
flowchart TD
  pr["pull_request / feature push"] --> ci_choice{"project type"}
  ci_choice --> node_ci["node-ci.yaml"]
  ci_choice --> python_ci["python-ci.yaml"]
  node_ci --> merge["merge to main"]
  python_ci --> merge

  merge --> deploy_trigger["push main / workflow_dispatch"]
  deploy_trigger --> deploy_choice{"deploy type"}

  deploy_choice --> docker_build["docker-build-push.yaml<br/>build and push image"]
  docker_build --> ssh_deploy["ssh-compose-deploy.yaml<br/>pull image and compose up"]
  docker_build --> ecs_deploy["ecs-deploy.yaml<br/>render task definition and update service"]
```

Docker image 기반 서버 배포는 아래 순서가 기본입니다.

```mermaid
sequenceDiagram
  participant Repo as Project repo
  participant Actions as GitHub Actions
  participant Registry as Container registry
  participant Server as Compose server
  participant ECS as Amazon ECS

  Repo->>Actions: pull_request
  Actions->>Actions: node-ci.yaml or python-ci.yaml
  Repo->>Actions: push main
  Actions->>Registry: docker-build-push.yaml
  alt SSH Docker Compose
    Actions->>Server: ssh-compose-deploy.yaml
    Server->>Registry: docker compose pull
    Server->>Server: docker compose up -d --no-build
    Server->>Server: healthcheck
  else Amazon ECS
    Actions->>ECS: ecs-deploy.yaml
    ECS->>Registry: pull image
    ECS->>ECS: rolling service deployment
  end
```

## How To Use

각 프로젝트 repo의 `.github/workflows/*.yaml`에서 이 repo의 workflow를 job 단위로 호출합니다.

```yaml
jobs:
  node-ci:
    uses: wibaek/github-automation/.github/workflows/node-ci.yaml@v1.0
```

기본 규칙은 다음과 같습니다.

- reusable workflow는 `jobs.<job_id>.uses`로 호출합니다.
- workflow 버전은 `@v1.0`처럼 tag로 고정합니다.
- 기본 권한은 `contents: read`로 시작합니다.
- Docker image push에는 registry에 맞는 권한을 추가합니다. GHCR은 `packages: write`, ECR/ECS는 `id-token: write`가 필요합니다.
- secrets는 호출하는 workflow에서 명시적으로 넘깁니다.
- Docker Compose 배포는 `docker-build-push.yaml` 뒤에 `ssh-compose-deploy.yaml`을 붙입니다.
- ECS 배포는 `docker-build-push.yaml` 뒤에 `ecs-deploy.yaml`을 붙입니다.
- 배포 job은 `docker-build-push.yaml`의 `image-reference` output을 받아서 같은 이미지를 배포합니다.
- dev, stage, prod는 같은 deploy workflow를 쓰고 `environment`, host/path/cluster/service 같은 input만 바꿉니다.
- Cloudflare Pages/Workers 배포는 Docker 라인과 별도 adapter workflow를 사용합니다.
- 릴리즈 관리는 `release.yaml`이 담당하고, Docker build/deploy는 별도 job으로 명시적으로 연결합니다.

구체적인 caller YAML 예시는 [docs/workflow-uses](docs/workflow-uses)를 봅니다.

## Versioning

공용 workflow는 SemVer tag로 배포합니다.

```bash
git tag v1.0
git push origin v1.0
```

이미 배포한 tag는 가능한 한 옮기지 않습니다. 같은 tag가 가리키는 코드가 바뀌면 사용하는 프로젝트의 재현성이 깨집니다.

## Docs

- GitHub Actions 기본 개념: [docs/01_github_actions.md](docs/01_github_actions.md)
- Docker build cache와 고급 예시: [docs/02_github_actions_advanced.md](docs/02_github_actions_advanced.md)
- 프로젝트별 호출 예시: [docs/workflow-uses](docs/workflow-uses)
