# Ferramentas Modernas de Deploy para VPS

> Este documento apresenta as principais ferramentas de PaaS self-hosted disponíveis em 2026 para simplificar o gerenciamento de aplicações em VPS.

---

## Visão Geral

Se você não quer gerenciar cada `docker-compose.yml` manualmente, existem ferramentas que adicionam uma camada de abstração com interface web, semelhante ao Heroku ou Railway, mas rodando na sua própria VPS.

```
Abordagem Manual        vs       Plataforma Self-Hosted
─────────────────────────────────────────────────────────
SSH + docker compose              Interface Web
Editar arquivos .env              Formulários visuais
Labels do Traefik                 Domínio com 1 clique
CI/CD manual                      Deploy via webhook/Git
Logs dispersos                    Logs centralizados
```

---

## 1. Coolify

**Site:** [coolify.io](https://coolify.io) | **Open Source** | **Auto-hospedado**

Coolify é a opção mais completa e popular em 2026 para PaaS self-hosted. Nasceu como alternativa ao Heroku e Railway.

### Funcionalidades

- Deploy de aplicações, bancos de dados e serviços com interface visual
- Suporte nativo a Docker, Docker Compose, Nixpacks e Dockerfile
- Integração com GitHub, GitLab, Bitbucket e qualquer repositório Git
- HTTPS automático com Let's Encrypt (usa Traefik por baixo)
- Gerenciamento de variáveis de ambiente com segredos criptografados
- Deploy automático via push (webhooks)
- Preview environments (deploy de PRs automaticamente)
- Suporte a múltiplos servidores remotos
- Backup automático de bancos de dados
- Logs em tempo real por aplicação

### Instalação

```bash
# Requisito: Docker instalado, Ubuntu 22.04+ ou Debian 12+
# Execute como root na VPS

curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Após a instalação, acesse: `http://<IP_DA_VPS>:8000`

### Quando usar Coolify

✅ Quando você quer uma experiência tipo Heroku/Railway na sua própria VPS  
✅ Quando você tem um time e quer dar acesso sem expor SSH  
✅ Quando você tem múltiplos servidores para gerenciar  
✅ Para projetos com CI/CD frequente e múltiplos repositórios  
❌ Quando você precisa de controle total e granular da infraestrutura

---

## 2. Dokploy

**Site:** [dokploy.com](https://dokploy.com) | **Open Source** | **Auto-hospedado**

Dokploy é uma alternativa mais leve ao Coolify, focada em simplicidade e velocidade de setup.

### Funcionalidades

- Deploy de aplicações Docker e Docker Compose
- Interface minimalista e intuitiva
- Integração com GitHub/GitLab via webhooks
- HTTPS automático com Traefik integrado
- Gerenciamento de variáveis de ambiente
- Logs de deploy em tempo real
- Suporte a múltiplos domínios por aplicação

### Instalação

```bash
# Execute como root ou com sudo
curl -sSL https://dokploy.com/install.sh | sh
```

Acesse: `http://<IP_DA_VPS>:3000`

### Quando usar Dokploy

✅ Quando quer algo mais simples e leve que o Coolify  
✅ Para projetos pessoais ou times pequenos  
✅ Quando o tempo de setup é uma prioridade  
❌ Para ambientes que precisam de preview environments ou múltiplos servidores

---

## 3. CapRover

**Site:** [caprover.com](https://caprover.com) | **Open Source** | **Auto-hospedado**

CapRover é uma das plataformas mais maduras do segmento, com foco em simplicidade e uma loja de "one-click apps" para instalação rápida de serviços populares.

### Funcionalidades

- One-click apps (mais de 100 apps pré-configuradas: WordPress, Mattermost, Nextcloud, etc.)
- CLI própria para deploy a partir do terminal
- Suporte a clusters com múltiplos servidores (via Docker Swarm)
- HTTPS automático com Let's Encrypt (usa Nginx por baixo)
- Nginx customizável por aplicação
- Webhook para CI/CD
- Interface web administrativa

### Instalação

```bash
# Requisito: Docker instalado
docker run -e "ACCEPTED_TERMS=true" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /captain:/captain \
  -p 80:80 -p 443:443 -p 3000:3000 \
  caprover/caprover

# Instala a CLI localmente (na sua máquina)
npm install -g caprover

# Configura o servidor
caprover serversetup
```

### CLI do CapRover

```bash
# Login no servidor
caprover login

# Deploy de uma aplicação
caprover deploy --appName minha-app

# Lista aplicações
caprover api --path /v2/user/apps/appDefinitions --method GET
```

### Quando usar CapRover

✅ Quando quer instalar serviços pré-configurados rapidamente (one-click apps)  
✅ Quando precisa de um cluster com Docker Swarm  
✅ Quando prefere usar CLI para deploys  
❌ Para times maiores que precisam de colaboração via interface web

---

## 4. Portainer

**Site:** [portainer.io](https://portainer.io) | **Community Edition gratuita**

O Portainer não é um PaaS — é um gerenciador visual de Docker. Ele não faz deploy a partir de repositórios Git, mas oferece visibilidade total sobre containers, imagens, volumes e redes.

### Funcionalidades

- Interface visual para gerenciar containers, imagens, volumes, redes
- Suporte a Docker Standalone e Docker Swarm
- Templates de stacks para criar serviços rapidamente
- Acesso a logs e terminal dos containers pela interface web
- Controle de acesso por usuário e time
- Suporte a múltiplos ambientes (hosts remotos)
- Edge Agents para ambientes isolados

### Instalação

```bash
# Cria volume para dados persistentes
docker volume create portainer_data

# Inicia o Portainer
docker run -d \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Acesse: `https://<IP_DA_VPS>:9443`

Ou via Docker Compose, integrado com Traefik:

```yaml
# /srv/apps/_infra/portainer/docker-compose.yml
version: "3.9"

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - portainer_data:/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.portainer.rule=Host(`portainer.seudominio.com`)"
      - "traefik.http.routers.portainer.entrypoints=websecure"
      - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
      - "traefik.http.services.portainer.loadbalancer.server.port=9000"
    networks:
      - proxy

volumes:
  portainer_data:

networks:
  proxy:
    external: true
```

### Quando usar Portainer

✅ Como complemento a qualquer outra ferramenta, para visibilidade  
✅ Quando você quer inspecionar containers sem usar o terminal  
✅ Para times com membros não técnicos que precisam ver logs  
❌ Como substituto de deploy automatizado — ele não tem essa função

---

## Comparativo Final

| Característica        | Coolify                     | Dokploy          | CapRover            | Portainer      |
| --------------------- | --------------------------- | ---------------- | ------------------- | -------------- |
| Deploy via Git        | ✅                          | ✅               | ✅                  | ❌             |
| Interface visual      | ✅                          | ✅               | ✅                  | ✅             |
| One-click apps        | ✅                          | ❌               | ✅                  | ✅ (templates) |
| Preview environments  | ✅                          | ❌               | ❌                  | ❌             |
| Multi-servidor        | ✅                          | ❌               | ✅ (Swarm)          | ✅             |
| Complexidade de setup | Média                       | Baixa            | Média               | Baixa          |
| Melhor para           | Equipes, múltiplos projetos | Projetos simples | Clusters, one-click | Visibilidade   |

---

## Recomendação para Este Guia

Para uma VPS com 5-10 aplicações sem ferramentas adicionais, a abordagem manual deste guia (Docker Compose + Traefik) é perfeitamente adequada e dá mais controle. Se quiser simplificar, adote:

- **Coolify** — melhor escolha geral para self-hosted PaaS
- **Portainer** — ótimo complemento visual independente da abordagem escolhida

---

## Próximo Passo

Continue com as **[Estratégias de Deploy](./deployment-strategies.md)** para automatizar o processo de entrega de novas versões.
