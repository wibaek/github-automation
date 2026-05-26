# SSH 배포 서버 준비

`ssh-compose-deploy.yaml`과 `ssh-compose-image-load-deploy.yaml`은 SSH로 서버에 접속해서 Docker Compose를 실행합니다.
서버에는 Docker와 Docker Compose plugin이 설치되어 있다고 가정합니다.

## 서버 유저 생성

먼저 서버에서 배포 전용 유저와 프로젝트 디렉터리를 만듭니다.

```bash
sudo useradd --create-home --shell /bin/bash --user-group deploy
sudo usermod -aG docker deploy

sudo install -d -m 755 -o deploy -g deploy /srv/my-app
sudo install -d -m 700 -o deploy -g deploy /home/deploy/.ssh
sudo touch /home/deploy/.ssh/authorized_keys
sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

`docker` 그룹 권한은 사실상 root 권한에 가깝습니다.
배포용 key는 이 서버와 이 앱에만 쓰고, GitHub Environment 승인 규칙을 붙이는 편이 안전합니다.

## SSH Key 생성

로컬 머신에서 GitHub Actions가 사용할 SSH key를 생성합니다.

```bash
ssh-keygen -t ed25519 -C "github-actions my-app deploy" -f ./github-actions-my-app-deploy -N ""
```

생성한 public key를 서버의 배포 유저에 등록합니다.
아래 예시의 `ubuntu@example.com`은 서버에 sudo 권한으로 접속할 수 있는 기존 관리자 계정으로 바꿉니다.

```bash
cat ./github-actions-my-app-deploy.pub | ssh ubuntu@example.com \
  'sudo tee -a /home/deploy/.ssh/authorized_keys >/dev/null && sudo chown deploy:deploy /home/deploy/.ssh/authorized_keys && sudo chmod 600 /home/deploy/.ssh/authorized_keys'
```

접속과 Docker 권한을 확인합니다.

```bash
ssh -i ./github-actions-my-app-deploy deploy@example.com \
  'docker ps >/dev/null && test -w /srv/my-app'
```

## GitHub 값 등록

GitHub repository variable과 secret을 등록합니다.
아래는 repository-level 값 기준입니다.

```bash
ssh-keyscan -p 22 -H example.com > known_hosts

gh variable set HOST --body "example.com"
gh variable set USER --body "deploy"
gh variable set KNOWN_HOSTS < known_hosts
gh secret set SSH_PRIVATE_KEY < ./github-actions-my-app-deploy
gh secret set ENV_FILE_CONTENT < ./.env.prod
```

환경별로 값을 분리하려면 `--env prod`를 붙입니다.

```bash
gh variable set HOST --env prod --body "example.com"
gh variable set USER --env prod --body "deploy"
gh variable set KNOWN_HOSTS --env prod < known_hosts
gh secret set SSH_PRIVATE_KEY --env prod < ./github-actions-my-app-deploy
gh secret set ENV_FILE_CONTENT --env prod < ./.env.prod
```

`ENV_FILE_CONTENT`는 workflow가 `env-file` input의 경로로 업로드합니다.
예를 들어 caller workflow에서 `env-file: ".env"`를 넘기면 서버의 `/srv/my-app/.env`가 배포 시점마다 이 secret 내용으로 갱신됩니다.
`.env` 파일은 repository에 커밋하지 않습니다.

`ssh-compose-deploy.yaml`에서 서버가 private registry를 직접 pull해야 한다면 registry password도 secret으로 등록합니다.
`ssh-compose-image-load-deploy.yaml`처럼 runner가 이미지를 pull해서 서버에 load하는 방식이면 서버의 registry login은 필요하지 않습니다.

```bash
gh secret set REGISTRY_PASSWORD < ./registry-password.txt
```
