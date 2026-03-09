# Reverse Proxy — Traefik e Nginx

> **Pré-requisito:** [Gerenciamento de containers](./container-management.md) compreendido. Rede `proxy` criada.

---

## O Que é um Reverse Proxy?

Um reverse proxy recebe todas as requisições da internet na porta 80 (HTTP) e 443 (HTTPS) e as encaminha para o container correto com base no domínio ou subdomínio solicitado.

```
Internet → app1.dominio.com ─────┐
Internet → app2.dominio.com ─────┤
Internet → app3.dominio.com ─────┤──► [Reverse Proxy :80/:443] ──► Containers
Internet → api.dominio.com  ─────┘
```

Sem um reverse proxy, cada aplicação precisaria de uma porta diferente exposta (`:3000`, `:3001`, etc.), impossibilitando o uso de HTTPS com domínio.

---

## Traefik vs Nginx — Qual Usar?

| Característica                    | Traefik v3                       | Nginx Proxy Manager           |
| --------------------------------- | -------------------------------- | ----------------------------- |
| Configuração                      | Via labels no docker-compose.yml | Via interface web ou arquivos |
| HTTPS automático                  | ✅ Nativo com Let's Encrypt      | ✅ Sim                        |
| Detecção automática de containers | ✅ Sim                           | ❌ Manual                     |
| Dashboard visual                  | ✅ Sim                           | ✅ Sim                        |
| Curva de aprendizado              | Moderada                         | Baixa                         |
| Melhor para                       | DevOps intermediário+            | Iniciantes                    |
| Recursos extras                   | Middlewares, rate limiting, auth | Básico                        |

> **Recomendação 2026:** Use **Traefik v3** para ambientes DevOps onde novos containers são adicionados frequentemente. Use **Nginx Proxy Manager** para ambientes mais simples onde você quer uma UI visual.

---

## Opção A — Traefik v3 (Recomendado)

### 1. Estrutura de arquivos

```
/srv/apps/_infra/traefik/
├── docker-compose.yml
├── traefik.yml          # Configuração estática
└── acme.json            # Certificados SSL (criado automaticamente)
```

### 2. Criar o arquivo acme.json

```bash
cd /srv/apps/_infra/traefik

# Cria o arquivo de certificados com permissão correta (OBRIGATÓRIO)
touch acme.json
chmod 600 acme.json
```

### 3. Configuração estática do Traefik

```yaml
# /srv/apps/_infra/traefik/traefik.yml

# Modo de log
log:
  level: INFO

# Painel de administração (acesse em traefik.seudominio.com)
api:
  dashboard: true
  insecure: false

# Pontos de entrada (entrypoints)
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http:
      tls:
        certResolver: letsencrypt

# Provedor de configuração dinâmica (via labels do Docker)
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false # Importante: não expõe containers sem labels explícitas
    network: proxy

# Emissor de certificados SSL via Let's Encrypt
certificatesResolvers:
  letsencrypt:
    acme:
      email: seu@email.com # ← Substitua pelo seu email
      storage: /acme.json
      httpChallenge:
        entryPoint: web
```

### 4. Docker Compose do Traefik

```yaml
# /srv/apps/_infra/traefik/docker-compose.yml
version: "3.9"

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./traefik.yml:/traefik.yml:ro
      - ./acme.json:/acme.json
    labels:
      - "traefik.enable=true"

      # Dashboard do Traefik (protegido por autenticação básica)
      - "traefik.http.routers.traefik-dashboard.rule=Host(`traefik.seudominio.com`)"
      - "traefik.http.routers.traefik-dashboard.entrypoints=websecure"
      - "traefik.http.routers.traefik-dashboard.tls.certresolver=letsencrypt"
      - "traefik.http.routers.traefik-dashboard.service=api@internal"
      - "traefik.http.routers.traefik-dashboard.middlewares=traefik-auth"

      # Autenticação básica para o dashboard (gere com: htpasswd -nb admin senha)
      - "traefik.http.middlewares.traefik-auth.basicauth.users=admin:$$apr1$$HASH_GERADO_AQUI"

    networks:
      - proxy

networks:
  proxy:
    external: true
```

### 5. Gerar senha para o dashboard

```bash
# Instala htpasswd
sudo apt install -y apache2-utils

# Gera o hash da senha (substitua 'admin' e 'sua-senha')
htpasswd -nb admin sua-senha-segura

# Saída exemplo: admin:$apr1$jxxxxxxxxxxxxxxxxxxxxxxxxxxx
# No docker-compose, substitua $ por $$ (escape do YAML)
```

### 6. Iniciar o Traefik

```bash
cd /srv/apps/_infra/traefik
docker compose up -d
docker compose logs -f
```

### 7. Configurar uma aplicação com Traefik

Adicione estas labels ao seu `docker-compose.yml` de cada aplicação:

```yaml
services:
  app:
    image: minha-app:1.0
    restart: unless-stopped
    labels:
      - "traefik.enable=true"

      # Rota principal HTTPS
      - "traefik.http.routers.minha-app.rule=Host(`minha-app.seudominio.com`)"
      - "traefik.http.routers.minha-app.entrypoints=websecure"
      - "traefik.http.routers.minha-app.tls.certresolver=letsencrypt"

      # Porta interna do container
      - "traefik.http.services.minha-app.loadbalancer.server.port=3000"
    networks:
      - internal
      - proxy

networks:
  internal:
    driver: bridge
  proxy:
    external: true
```

### 8. Middlewares úteis do Traefik

```yaml
# Rate limiting — limita requisições por IP
- "traefik.http.middlewares.rate-limit.ratelimit.average=100"
- "traefik.http.middlewares.rate-limit.ratelimit.burst=50"

# Headers de segurança
- "traefik.http.middlewares.security-headers.headers.stsSeconds=31536000"
- "traefik.http.middlewares.security-headers.headers.stsIncludeSubdomains=true"
- "traefik.http.middlewares.security-headers.headers.browserXssFilter=true"
- "traefik.http.middlewares.security-headers.headers.contentTypeNosniff=true"
- "traefik.http.middlewares.security-headers.headers.frameDeny=true"

# Aplica os middlewares na rota
- "traefik.http.routers.minha-app.middlewares=rate-limit,security-headers"
```

---

## Opção B — Nginx Proxy Manager (Interface Visual)

O **Nginx Proxy Manager** oferece uma interface web amigável para gerenciar proxies e certificados SSL sem precisar editar arquivos de configuração manualmente.

### 1. Docker Compose do Nginx Proxy Manager

```yaml
# /srv/apps/_infra/nginx-proxy-manager/docker-compose.yml
version: "3.9"

services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81" # Interface web de administração
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    networks:
      - proxy

networks:
  proxy:
    external: true
```

```bash
cd /srv/apps/_infra/nginx-proxy-manager
docker compose up -d
```

Acesse: `http://<IP_DA_VPS>:81`

- Email padrão: `admin@example.com`
- Senha padrão: `changeme`

> **IMPORTANTE:** Altere as credenciais padrão imediatamente após o primeiro acesso.

### 2. Configurar uma rota no Nginx Proxy Manager

1. Acesse a interface web na porta 81
2. Vá em **Proxy Hosts** → **Add Proxy Host**
3. Configure:
   - **Domain Names:** `minha-app.seudominio.com`
   - **Scheme:** `http`
   - **Forward Hostname / IP:** nome do container (ex: `minha-app`)
   - **Forward Port:** porta interna do container (ex: `3000`)
4. Na aba **SSL**, selecione **Let's Encrypt** e marque **Force SSL**
5. Salve — o certificado é emitido automaticamente

> **Para o NPM alcançar os containers das apps**, as apps devem estar na rede `proxy`:
>
> ```yaml
> networks:
>   proxy:
>     external: true
> ```

---

## Configuração de DNS

Antes de qualquer certificado SSL funcionar, aponte seus domínios para o IP da VPS:

```
Tipo A:  app1.seudominio.com    → <IP_DA_VPS>
Tipo A:  app2.seudominio.com    → <IP_DA_VPS>
Tipo A:  api.seudominio.com     → <IP_DA_VPS>
Tipo A:  traefik.seudominio.com → <IP_DA_VPS>
```

Usando wildcard (simplifica gerenciamento):

```
Tipo A:  seudominio.com         → <IP_DA_VPS>
Tipo A:  *.seudominio.com       → <IP_DA_VPS>
```

---

## Verificar Certificados SSL

```bash
# Verifica certificado de um domínio
echo | openssl s_client -servername minha-app.seudominio.com \
  -connect minha-app.seudominio.com:443 2>/dev/null | \
  openssl x509 -noout -dates

# Verifica com curl
curl -vI https://minha-app.seudominio.com 2>&1 | grep -E "SSL|certificate|expire"
```

---

## Próximo Passo

Continue com as **[Ferramentas de Deploy](./deploy-tools.md)** para conhecer alternativas que simplificam o gerenciamento de aplicações.
