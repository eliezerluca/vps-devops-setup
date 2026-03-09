# Estratégias de Deploy

> **Pré-requisito:** Estrutura de diretórios criada conforme [arquitetura multi-app](./multi-app-architecture.md).

---

## 1. Deploy Manual via SSH

O deploy manual é o ponto de partida — simples, sem ferramentas externas, e útil para entender o que acontece por baixo de toda automação.

### Fluxo básico

```bash
# 1. Acessa o servidor
ssh vps

# 2. Navega até o diretório da aplicação
cd /srv/apps/minha-app

# 3. Atualiza o código (se o dockerfile vem do repositório)
git pull origin main

# 4. Reconstrói e recria os containers
docker compose pull       # Para imagens de registry externo
docker compose up -d --build  # Para imagens buildadas localmente

# 5. Verifica o resultado
docker compose ps
docker compose logs --tail=30
```

### Script de deploy manual reutilizável

```bash
# /srv/apps/minha-app/deploy.sh
#!/bin/bash
set -e

APP_DIR="/srv/apps/minha-app"
cd "$APP_DIR"

echo "→ Fazendo pull da imagem mais recente..."
docker compose pull

echo "→ Recriando containers..."
docker compose up -d --remove-orphans

echo "→ Verificando saúde dos containers..."
sleep 5
docker compose ps

echo "✓ Deploy concluído!"
```

```bash
chmod +x /srv/apps/minha-app/deploy.sh
./deploy.sh
```

---

## 2. Deploy via Git (com Webhook)

Este método permite que o servidor faça deploy automaticamente sempre que você fizer push para o repositório.

### 2.1 Configurar Git na VPS

```bash
# Adiciona a chave SSH da VPS ao GitHub/GitLab
ssh-keygen -t ed25519 -C "vps-deploy" -f ~/.ssh/github_deploy
cat ~/.ssh/github_deploy.pub
# Cole esta chave como "Deploy Key" no repositório do GitHub
```

```bash
# Clona o repositório na VPS
cd /srv/apps
git clone git@github.com:sua-org/minha-app.git minha-app
cd minha-app

# Configura git para usar a chave correta
git config core.sshCommand "ssh -i ~/.ssh/github_deploy -F /dev/null"
```

### 2.2 Servidor de Webhook simples

```bash
# Instala o webhook
sudo apt install -y webhook

# Cria arquivo de configuração
sudo mkdir -p /etc/webhook
sudo vim /etc/webhook/hooks.json
```

```json
[
  {
    "id": "deploy-minha-app",
    "execute-command": "/srv/apps/minha-app/deploy.sh",
    "command-working-directory": "/srv/apps/minha-app",
    "trigger-rule": {
      "and": [
        {
          "match": {
            "type": "payload-hash-sha256",
            "secret": "SEU_WEBHOOK_SECRET_AQUI",
            "parameter": {
              "source": "header",
              "name": "X-Hub-Signature-256"
            }
          }
        },
        {
          "match": {
            "type": "value",
            "value": "refs/heads/main",
            "parameter": {
              "source": "payload",
              "name": "ref"
            }
          }
        }
      ]
    }
  }
]
```

```bash
# Inicia o servidor de webhook
webhook -hooks /etc/webhook/hooks.json -port 9000 -verbose &

# No GitHub: Settings → Webhooks → Add webhook
# Payload URL: https://seudominio.com:9000/hooks/deploy-minha-app
# Secret: SEU_WEBHOOK_SECRET_AQUI
# Event: Just the push event
```

---

## 3. Deploy com GitHub Actions

O GitHub Actions permite criar pipelines completos de CI/CD que rodam nos servidores do GitHub e fazem deploy na sua VPS via SSH.

### Estrutura do Workflow

```
.github/
└── workflows/
    ├── ci.yml      # Testes e build
    └── deploy.yml  # Deploy em produção
```

### 3.1 Configurar segredos no GitHub

No repositório: **Settings → Secrets and variables → Actions**

Adicione:

- `VPS_HOST` — IP ou domínio da VPS
- `VPS_USER` — `deploy`
- `VPS_SSH_KEY` — conteúdo da chave privada SSH
- `VPS_PORT` — porta SSH (ex: `22`)
- `DOCKER_USERNAME` — usuário do Docker Hub (se usar)
- `DOCKER_PASSWORD` — senha/token do Docker Hub

### 3.2 Workflow de CI (Testes)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      - name: Usa Node.js 20
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Instala dependências
        run: npm ci

      - name: Executa testes
        run: npm test

      - name: Build
        run: npm run build
```

### 3.3 Workflow de Deploy em Produção

```yaml
# .github/workflows/deploy.yml
name: Deploy em Produção

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: [] # Adicione 'test' aqui se quiser rodar após os testes

    steps:
      - name: Checkout código
        uses: actions/checkout@v4

      # Opção A: Publica imagem no Docker Hub e deploy na VPS
      - name: Login no Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build e push da imagem
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/minha-app:latest
            ${{ secrets.DOCKER_USERNAME }}/minha-app:${{ github.sha }}

      # Deploy na VPS via SSH
      - name: Deploy na VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          port: ${{ secrets.VPS_PORT }}
          script: |
            cd /srv/apps/minha-app

            # Atualiza a tag da imagem no compose file
            sed -i 's|minha-app:.*|minha-app:${{ github.sha }}|g' docker-compose.yml

            # Pull e recria os containers
            docker compose pull
            docker compose up -d --remove-orphans

            # Verifica saúde
            sleep 10
            docker compose ps

            # Limpa imagens antigas
            docker image prune -f
```

### 3.4 Deploy com Zero Downtime (Blue-Green simplificado)

```yaml
# Adicione nos scripts de deploy
- name: Deploy com health check
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USER }}
    key: ${{ secrets.VPS_SSH_KEY }}
    script: |
      cd /srv/apps/minha-app

      # Pull nova imagem
      docker compose pull

      # Recria um serviço por vez (reduz downtime)
      docker compose up -d --no-deps --build app

      # Aguarda o healthcheck ficar saudável
      TIMEOUT=60
      ELAPSED=0
      while [ $ELAPSED -lt $TIMEOUT ]; do
        STATUS=$(docker inspect --format='{{.State.Health.Status}}' minha-app 2>/dev/null || echo "unknown")
        if [ "$STATUS" = "healthy" ]; then
          echo "✓ Container saudável após ${ELAPSED}s"
          break
        fi
        echo "Aguardando... status: $STATUS (${ELAPSED}s)"
        sleep 5
        ELAPSED=$((ELAPSED + 5))
      done

      if [ "$STATUS" != "healthy" ]; then
        echo "✗ Container não ficou saudável — revertendo..."
        # Rollback automático
        git stash
        docker compose up -d --no-deps app
        exit 1
      fi
```

---

## 4. Deploy com GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: registry.gitlab.com/$CI_PROJECT_PATH

test:
  stage: test
  image: node:20-alpine
  script:
    - npm ci
    - npm test
  only:
    - merge_requests
    - main

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHA .
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHA $DOCKER_IMAGE:latest
    - docker push $DOCKER_IMAGE:$CI_COMMIT_SHA
    - docker push $DOCKER_IMAGE:latest
  only:
    - main

deploy_production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - mkdir -p ~/.ssh
    - echo "$VPS_SSH_KEY" > ~/.ssh/id_ed25519
    - chmod 600 ~/.ssh/id_ed25519
    - ssh-keyscan -H $VPS_HOST >> ~/.ssh/known_hosts
  script:
    - |
      ssh -p $VPS_PORT $VPS_USER@$VPS_HOST "
        cd /srv/apps/minha-app
        sed -i 's|registry.gitlab.com/.*:.*|$DOCKER_IMAGE:$CI_COMMIT_SHA|g' docker-compose.yml
        docker compose pull
        docker compose up -d --remove-orphans
        docker image prune -f
      "
  environment:
    name: production
    url: https://minha-app.seudominio.com
  only:
    - main
```

---

## 5. Resumo das Estratégias

| Estratégia      | Complexidade | Automação   | Indicado para            |
| --------------- | ------------ | ----------- | ------------------------ |
| Deploy manual   | Baixa        | Nenhuma     | Aprendizado, emergências |
| Script + SSH    | Baixa        | Semi-auto   | Projetos pequenos        |
| Webhook         | Média        | Auto (push) | Times pequenos           |
| GitHub Actions  | Média        | Total       | Qualquer projeto         |
| GitLab CI/CD    | Média        | Total       | Projetos no GitLab       |
| Coolify/Dokploy | Baixa        | Total       | Quem quer abstração      |

---

## Próximo Passo

Continue com o **[Monitoramento e Observabilidade](./monitoring.md)** para saber o que está acontecendo na sua infraestrutura.
