# Workflow 사용 예시

이 디렉터리는 이 레포의 reusable workflow를 각 프로젝트에서 호출하는 예시 YAML을 보관합니다.
각 파일은 사용하는 프로젝트 레포의 `.github/workflows/*.yaml`에 복사해서 쓸 수 있는 형태입니다.

| 예시 파일 | 호출하는 workflow | 용도 |
| --- | --- | --- |
| `node-ci.yaml` | `.github/workflows/node-ci.yaml` | Node.js, Next.js CI |
| `python-ci.yaml` | `.github/workflows/python-ci.yaml` | Python CI |
| `cloudflare-pages-deploy.yaml` | `.github/workflows/cloudflare-pages-deploy.yaml` | Cloudflare Pages 배포 |
| `cloudflare-workers-deploy.yaml` | `.github/workflows/cloudflare-workers-deploy.yaml` | Cloudflare Workers 배포 |
| `docker-build-ghcr-push.yaml` | `.github/workflows/docker-build-ghcr-push.yaml` | GHCR Docker 이미지 빌드 및 push |
| `docker-build-ecr-push.yaml` | `.github/workflows/docker-build-ecr-push.yaml` | ECR Docker 이미지 빌드 및 push |
| `docker-build-docker-hub-push.yaml` | `.github/workflows/docker-build-docker-hub-push.yaml` | Docker Hub 이미지 빌드 및 push |
| `release.yaml` | `.github/workflows/release.yaml` | Release PR, tag, GitHub Release, CHANGELOG |
| `ssh-compose-deploy.yaml` | `.github/workflows/ssh-compose-deploy.yaml` | compose/env를 업로드하고 서버가 registry에서 직접 pull하는 SSH Docker Compose 배포 |
| `ssh-compose-image-load-deploy.yaml` | `.github/workflows/ssh-compose-image-load-deploy.yaml` | compose/env와 이미지를 runner에서 서버에 전송하는 SSH Docker Compose 배포 |
| `docker-ghcr-ssh-deploy.yaml` | `docker-build-ghcr-push.yaml`, `ssh-compose-image-load-deploy.yaml` | GHCR 이미지 빌드 후 runner에서 pull하고 서버에는 image와 compose file 전송 |
| `ecs-ecr-deploy.yaml` | `docker-build-ecr-push.yaml`, `ecs-deploy.yaml` | ECR 이미지 빌드 후 ECS service 배포 |

예시의 `@v1.0`은 배포된 태그에 맞게 바꿔서 사용합니다.
배포 secret 이름은 프로젝트별 GitHub repository 또는 environment secret에 맞춰 조정합니다.
