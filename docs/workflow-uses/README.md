# Workflow 사용 예시

이 디렉터리는 이 레포의 reusable workflow를 각 프로젝트에서 호출하는 예시 YAML을 보관합니다.
각 파일은 사용하는 프로젝트 레포의 `.github/workflows/*.yaml`에 복사해서 쓸 수 있는 형태입니다.

| 예시 파일 | 호출하는 workflow | 용도 |
| --- | --- | --- |
| `node-ci.yaml` | `.github/workflows/node-ci.yaml` | Node.js, Next.js CI |
| `python-ci.yaml` | `.github/workflows/python-ci.yaml` | Python CI |
| `cloudflare-pages-deploy.yaml` | `.github/workflows/cloudflare-pages-deploy.yaml` | Cloudflare Pages 배포 |
| `cloudflare-workers-deploy.yaml` | `.github/workflows/cloudflare-workers-deploy.yaml` | Cloudflare Workers 배포 |
| `docker-build-push.yaml` | `.github/workflows/docker-build-push.yaml` | Docker 이미지 빌드 및 registry push |
| `release.yaml` | `.github/workflows/release.yaml` | Release PR, tag, GitHub Release, CHANGELOG |
| `ssh-compose-deploy.yaml` | `.github/workflows/ssh-compose-deploy.yaml` | SSH 기반 Docker Compose 배포 |
| `docker-ghcr-ssh-deploy.yaml` | `docker-build-push.yaml`, `ssh-compose-deploy.yaml` | GHCR 이미지 빌드 후 서버 배포 |
| `ecs-ecr-deploy.yaml` | `docker-build-push.yaml`, `ecs-deploy.yaml` | ECR 이미지 빌드 후 ECS service 배포 |

예시의 `@v1.0`은 배포된 태그에 맞게 바꿔서 사용합니다.
배포 secret 이름은 프로젝트별 GitHub repository 또는 environment secret에 맞춰 조정합니다.
