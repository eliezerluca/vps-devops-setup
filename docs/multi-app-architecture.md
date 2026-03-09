# Arquitetura Multi-App na VPS

> **Pré-requisito:** [Docker instalado e configurado](./docker-installation.md).

---

## 1. Filosofia da Arquitetura

Hospedar múltiplas aplicações em uma única VPS exige organização rigorosa para evitar:

- Conflito de portas entre aplicações
- Mistura de variáveis de ambiente
- Dificuldade de identificar qual arquivo pertence a qual projeto
- Backups inconsistentes
- Dificuldade de remover ou migrar aplicações individualmente

A solução é uma estrutura de diretórios padronizada, onde cada aplicação é completamente isolada e autocontida.

---

## 2. Estrutura de Diretórios Recomendada

```
/srv/
└── apps/
    ├── _infra/                   # Infraestrutura compartilhada
    │   ├── traefik/              # Reverse proxy
    │   │   ├── docker-compose.yml
    │   │   ├── traefik.yml
    │   │   └── acme.json         # Certificados SSL (chmod 600)
    │   ├── monitoring/           # Prometheus + Grafana
    │   │   └── docker-compose.yml
    │   └── portainer/            # Gerenciador visual (opcional)
    │       └── docker-compose.yml
    │
    ├── app1-nome-do-projeto/
    │   ├── docker-compose.yml
    │   ├── .env                  # NÃO vai para o git
    │   ├── .env.example          # Template commitado no git
    │   ├── data/                 # Dados persistentes (bind mounts)
    │   │   ├── uploads/
    │   │   └── backups/
    │   └── logs/                 # Logs específicos da aplicação
    │
    ├── app2-nome-do-projeto/
    │   ├── docker-compose.yml
    │   ├── .env
    │   ├── data/
    │   └── logs/
    │
    ├── app3-nome-do-projeto/
    │   └── ...
    │
    └── app4-nome-do-projeto/
        └── ...
```

### Por que `/srv/apps`?

O diretório `/srv` é definido pelo padrão **FHS (Filesystem Hierarchy Standard)** como o local para dados servidos pelo sistema. É semanticamente correto, fica fora do diretório home, e é fácil de fazer backup de forma direcionada.

---

## 3. Criando a Estrutura Base

```bash
# Cria o diretório raiz
sudo mkdir -p /srv/apps/_infra/{traefik,monitoring,portainer}
sudo chown -R deploy:deploy /srv/apps
sudo chmod -R 750 /srv/apps

# Cria a rede compartilhada do proxy
docker network create proxy

# Verifica
ls -la /srv/apps/
```

### Script auxiliar para criar uma nova aplicação

```bash
# Salve em /usr/local/bin/new-app
sudo tee /usr/local/bin/new-app > /dev/null << 'EOF'
#!/bin/bash
# Uso: new-app <nome-da-app>

if [ -z "$1" ]; then
  echo "Uso: new-app <nome-da-app>"
  exit 1
fi

APP_NAME=$1
APP_DIR="/srv/apps/$APP_NAME"

mkdir -p "$APP_DIR"/{data,logs,data/backups}
touch "$APP_DIR/.env"
touch "$APP_DIR/docker-compose.yml"

echo "✓ Estrutura criada em $APP_DIR"
echo "  Edite $APP_DIR/docker-compose.yml e $APP_DIR/.env para começar."
EOF

sudo chmod +x /usr/local/bin/new-app

# Usar:
new-app meu-novo-projeto
```

---

## 4. Modelo de docker-compose.yml para Aplicação Web

Todo projeto deve seguir este modelo base:

```yaml
# /srv/apps/minha-app/docker-compose.yml
version: "3.9"

services:
  # ── Aplicação principal ──────────────────────────────────
  app:
    image: minha-org/minha-app:${APP_VERSION:-latest}
    container_name: minha-app
    restart: unless-stopped
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./data/uploads:/app/uploads
    labels:
      # Traefik (reverse proxy) — veja o guia de reverse proxy
      - "traefik.enable=true"
      - "traefik.http.routers.minha-app.rule=Host(`minha-app.meudominio.com`)"
      - "traefik.http.routers.minha-app.entrypoints=websecure"
      - "traefik.http.routers.minha-app.tls.certresolver=letsencrypt"
      - "traefik.http.services.minha-app.loadbalancer.server.port=3000"
    networks:
      - internal
      - proxy # Permite que o Traefik roteie tráfego para este container

  # ── Banco de dados ───────────────────────────────────────
  db:
    image: postgres:16-alpine
    container_name: minha-app-db
    restart: unless-stopped
    env_file: .env
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal # Banco de dados NÃO está na rede do proxy

  # ── Cache (opcional) ─────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: minha-app-redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    networks:
      - internal

volumes:
  db_data:
    name: minha-app_db_data
  redis_data:
    name: minha-app_redis_data

networks:
  internal:
    name: minha-app_internal
    driver: bridge
  proxy:
    external: true # Referencia a rede criada pelo Traefik
```

---

## 5. Modelo de Arquivo .env

```bash
# /srv/apps/minha-app/.env
# NUNCA commit este arquivo no git!

# ── Aplicação ────────────────────────────────────────────
APP_VERSION=1.2.3
NODE_ENV=production
APP_SECRET=sua-chave-secreta-aqui-muito-longa-e-aleatoria

# ── Banco de Dados ───────────────────────────────────────
POSTGRES_DB=minha_app_db
POSTGRES_USER=app_user
POSTGRES_PASSWORD=senha-forte-do-banco

DATABASE_URL=postgresql://app_user:senha-forte-do-banco@db:5432/minha_app_db

# ── Redis ────────────────────────────────────────────────
REDIS_PASSWORD=senha-forte-redis
REDIS_URL=redis://:senha-forte-redis@redis:6379

# ── Email ────────────────────────────────────────────────
SMTP_HOST=smtp.exemplo.com
SMTP_PORT=587
SMTP_USER=seu@email.com
SMTP_PASS=senha-smtp
```

### .env.example (commitado no git)

```bash
# /srv/apps/minha-app/.env.example
# Copie este arquivo: cp .env.example .env
# Preencha com valores reais antes de rodar

APP_VERSION=latest
NODE_ENV=production
APP_SECRET=

POSTGRES_DB=minha_app_db
POSTGRES_USER=app_user
POSTGRES_PASSWORD=

DATABASE_URL=postgresql://app_user:@db:5432/minha_app_db

REDIS_PASSWORD=
REDIS_URL=redis://:@redis:6379

SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
```

---

## 6. Separação de Redes — Arquitetura de Segurança

```
Internet
    │
    ▼
[Traefik] ──── rede: proxy ────►  [app] (minha-app)
                                    │
                               rede: internal
                                    │
                          ┌─────────┴──────────┐
                          ▼                    ▼
                       [db]                [redis]
               (NÃO exposto)          (NÃO exposto)
```

**Princípio de menor privilégio para redes:**

- O banco de dados e o Redis **nunca** são conectados à rede `proxy`
- Somente o container da aplicação tem acesso ao banco internamente
- O Traefik só enxerga a aplicação, não o banco

---

## 7. Nomenclatura de Containers e Volumes

Use um padrão consistente para facilitar identificação e operação:

```
Containers:   <app-name>-<role>
              ex: loja-app, loja-db, loja-redis, loja-worker

Volumes:      <app-name>_<role>_data
              ex: loja_db_data, loja_redis_data, loja_uploads

Redes:        <app-name>_internal
              ex: loja_internal
```

---

## 8. Exemplo Completo — 5 Aplicações na Mesma VPS

```
/srv/apps/
├── _infra/
│   ├── traefik/      → reverse proxy + HTTPS automático
│   └── monitoring/   → Prometheus + Grafana
│
├── loja-ecommerce/   → app.lojaexemplo.com.br
│   ├── docker-compose.yml (Next.js + PostgreSQL)
│   ├── .env
│   └── data/uploads/
│
├── blog-marketing/   → blog.empresa.com.br
│   ├── docker-compose.yml (Ghost ou WordPress)
│   ├── .env
│   └── data/
│
├── api-interna/      → api.empresa.com.br
│   ├── docker-compose.yml (FastAPI + PostgreSQL + Redis)
│   ├── .env
│   └── data/
│
├── painel-admin/     → admin.empresa.com.br
│   ├── docker-compose.yml (React SPA servido por Nginx)
│   └── .env
│
└── webhook-service/  → hooks.empresa.com.br
    ├── docker-compose.yml (Node.js simples)
    └── .env
```

Consumo estimado para 5 aplicações leves:

| Recurso         | Estimativa                     |
| --------------- | ------------------------------ |
| RAM total usada | ~2-3 GB                        |
| CPU idle        | < 10%                          |
| Disco           | ~20 GB (dados + imagens)       |
| VPS recomendada | 4 GB RAM / 2 vCPUs / 40 GB SSD |

---

## Próximo Passo

Continue com o **[Gerenciamento de Containers](./container-management.md)** para aprender a operar e manter as aplicações no dia a dia.
