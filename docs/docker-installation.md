# Instalação do Docker

> **Pré-requisito:** [Segurança do servidor](./security.md) configurada. Execute como usuário `deploy` (com sudo).

---

## 1. Instalação do Docker Engine (Método Oficial)

Nunca instale Docker via `apt install docker.io` — esse pacote é desatualizado e mantido pela distribuição, não pela Docker Inc. Use sempre o repositório oficial.

```bash
# Remove versões antigas se existirem
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Instala dependências necessárias
sudo apt update
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# Adiciona a chave GPG oficial da Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Adiciona o repositório oficial da Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Atualiza a lista de pacotes e instala o Docker
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

### Verificar a instalação

```bash
# Verifica versão instalada
docker --version
docker compose version

# Testa com container Hello World (como root por enquanto)
sudo docker run hello-world
```

---

## 2. Configurar Docker sem Root (Rootless Mode de Grupo)

Por padrão, o Docker requer sudo. Adicione o usuário `deploy` ao grupo `docker`:

```bash
sudo usermod -aG docker deploy

# Aplica a mudança na sessão atual sem fazer logout
newgrp docker

# Verifica que funciona sem sudo
docker run hello-world
```

> **Atenção de segurança:** Usuários no grupo `docker` têm efetivamente privilégios equivalentes ao root, pois containers podem montar o sistema de arquivos do host. Adicione ao grupo apenas usuários confiáveis.

---

## 3. Configurar o Docker Engine

Crie ou edite o arquivo de configuração do daemon Docker:

```bash
sudo vim /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true
}
```

Explicação das opções:

| Opção                     | Descrição                                                         |
| ------------------------- | ----------------------------------------------------------------- |
| `log-driver: json-file`   | Driver de log padrão com rotação automática                       |
| `max-size: 10m`           | Cada arquivo de log pode ter no máximo 10 MB                      |
| `max-file: 3`             | Mantém no máximo 3 arquivos de log por container                  |
| `live-restore: true`      | Containers continuam rodando se o daemon reiniciar                |
| `userland-proxy: false`   | Melhora performance de redes de containers                        |
| `no-new-privileges: true` | Impede processos dentro de containers de ganhar novos privilégios |

```bash
# Reinicia o Docker para aplicar configurações
sudo systemctl restart docker

# Verifica status
sudo systemctl status docker

# Habilita o Docker para iniciar com o sistema
sudo systemctl enable docker
```

---

## 4. Configurar Redes Docker Personalizadas

Para isolar aplicações e permitir comunicação seletiva entre containers, crie redes dedicadas:

```bash
# Rede para o reverse proxy (Traefik/Nginx) se comunicar com os apps
docker network create proxy

# Verifica redes existentes
docker network ls
```

Cada aplicação também terá sua própria rede interna (definida no docker-compose.yml).

---

## 5. Boas Práticas de Segurança com Docker

### 5.1 Nunca execute containers como root

No seu `Dockerfile`, sempre defina um usuário não-root:

```dockerfile
FROM node:20-alpine

# Cria usuário não-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY --chown=appuser:appgroup . .

# Troca para o usuário não-root
USER appuser

CMD ["node", "server.js"]
```

### 5.2 Use imagens verificadas e com tags fixas

```yaml
# ❌ Ruim — versão pode mudar sem aviso
image: postgres:latest

# ✅ Bom — versão fixada, previsível
image: postgres:16.2-alpine3.19
```

### 5.3 Secrets — nunca passe senhas como variáveis de ambiente simples

Use Docker Secrets ou arquivos `.env` fora do repositório:

```yaml
# docker-compose.yml
services:
  app:
    image: minha-app:1.0
    env_file:
      - .env # Arquivo NÃO commitado no git
```

```bash
# .gitignore deve sempre conter:
echo ".env" >> .gitignore
echo "*.env" >> .gitignore
```

### 5.4 Limitar recursos de containers

```yaml
services:
  app:
    image: minha-app:1.0
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 256M
```

### 5.5 Scan de vulnerabilidades em imagens

```bash
# Instala o Trivy (scanner de vulnerabilidades)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Escaneia uma imagem
trivy image postgres:16-alpine

# Escaneia apenas vulnerabilidades críticas e altas
trivy image --severity HIGH,CRITICAL minha-app:1.0
```

---

## 6. Docker Compose — Revisão de Conceitos

O `docker compose` (v2, integrado ao Docker CLI) é a ferramenta para orquestrar múltiplos containers definidos em um arquivo `docker-compose.yml`.

### Exemplo básico — Aplicação Node.js com PostgreSQL

```yaml
# docker-compose.yml
version: "3.9"

services:
  app:
    build: .
    restart: unless-stopped
    env_file: .env
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - internal
      - proxy

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    env_file: .env
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $POSTGRES_USER"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - internal

volumes:
  db_data:

networks:
  internal:
    driver: bridge
  proxy:
    external: true
```

### Comandos essenciais do Docker Compose

```bash
# Inicia todos os containers em background
docker compose up -d

# Para e remove containers (mantém volumes)
docker compose down

# Para e remove containers E volumes
docker compose down -v

# Ver status dos containers
docker compose ps

# Ver logs de todos os serviços
docker compose logs -f

# Ver logs de um serviço específico
docker compose logs -f app

# Executa um comando dentro de um container rodando
docker compose exec app sh

# Faz pull das imagens mais recentes
docker compose pull

# Reconstrói as imagens e recria containers
docker compose up -d --build

# Escala um serviço (múltiplas instâncias)
docker compose up -d --scale app=3
```

---

## 7. Verificação da Instalação Completa

```bash
# Versões instaladas
docker --version
docker compose version
docker buildx version

# Informações do sistema Docker
docker info

# Containers e imagens atuais
docker ps -a
docker images

# Volumes e redes
docker volume ls
docker network ls
```

---

## Próximo Passo

Continue com a **[Arquitetura Multi-App](./multi-app-architecture.md)** para organizar seus projetos de forma profissional.
