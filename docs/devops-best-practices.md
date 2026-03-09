# Boas Práticas DevOps para Pequenos Servidores

> Este documento consolida as melhores práticas para manter uma infraestrutura saudável, organizada e profissional ao longo do tempo.

---

## 1. Organização de Múltiplos Repositórios

### Padrão de nomenclatura de repositórios

```
# Prefixo por tipo de projeto
app-loja-ecommerce       → aplicação principal
api-serviço-pagamentos   → API/backend
web-landing-page         → frontend estático
infra-vps-configuracao   → configurações de infraestrutura
```

### Repositório de infraestrutura (GitOps)

Versione toda a sua infraestrutura como código em um repositório dedicado:

```
infra-vps/
├── README.md
├── apps/
│   ├── minha-app/
│   │   ├── docker-compose.yml
│   │   └── .env.example       ← Template (SEM valores reais)
│   └── outra-app/
│       ├── docker-compose.yml
│       └── .env.example
├── _infra/
│   ├── traefik/
│   │   ├── docker-compose.yml
│   │   └── traefik.yml
│   └── monitoring/
│       ├── docker-compose.yml
│       └── prometheus/prometheus.yml
└── scripts/
    ├── backup-app.sh
    ├── deploy.sh
    └── setup-server.sh
```

> **Princípio:** Tudo que pode ser versionado, deve ser. A única exceção são os valores secretos (senhas, chaves de API) — esses ficam apenas no servidor, nunca em repositórios.

---

## 2. Gerenciamento de Variáveis de Ambiente e Segredos

### Hierarquia de segredos

```
Nível 1 (mais simples):   .env no servidor
Nível 2 (mais seguro):    Docker Secrets
Nível 3 (avançado):       HashiCorp Vault ou similar
```

Para a maioria das VPS pequenas, o **Nível 1** com boas práticas é suficiente.

### Boas práticas para arquivos .env

```bash
# 1. Nunca commite .env no git
echo ".env" >> .gitignore
echo "*.env" >> .gitignore

# 2. Sempre mantenha um .env.example atualizado
# Este arquivo tem apenas as CHAVES, sem os valores
cat .env.example
# APP_SECRET=
# POSTGRES_PASSWORD=
# REDIS_PASSWORD=

# 3. Use nomes descritivos e padronizados
# ❌ Ruim
SECRET=abc123
DB_PASS=abc123

# ✅ Bom
APP_JWT_SECRET=<gerar com: openssl rand -hex 32>
POSTGRES_PASSWORD=<gerar com: openssl rand -hex 20>

# 4. Gere senhas fortes para todos os serviços
openssl rand -hex 32   # 64 caracteres hexadecimais
openssl rand -base64 32  # 44 caracteres base64
```

### Gerenciar segredos com sops (avançado)

O [sops](https://github.com/getsops/sops) permite encriptar arquivos `.env` e commitar com segurança:

```bash
# Instala o sops
sudo apt install -y age
curl -Lo /usr/local/bin/sops https://github.com/getsops/sops/releases/latest/download/sops-v3.9.0.linux.amd64
sudo chmod +x /usr/local/bin/sops

# Gera chave age
age-keygen -o ~/.config/sops/age/keys.txt

# Encripta um arquivo .env
sops --age $(cat ~/.config/sops/age/keys.txt | grep 'public key' | awk '{print $4}') \
  -e .env > .env.enc

# Agora .env.enc pode ser commitado no git com segurança
# Para decriptar na VPS:
sops -d .env.enc > .env
```

---

## 3. Versionamento de Infraestrutura (IaC)

### Princípios de Infrastructure as Code

1. **Idempotência** — rodar o script duas vezes não muda nada se já está configurado
2. **Versionamento** — todo arquivo de configuração fica no git
3. **Documentação embutida** — comentários explicam o "por quê", não o "o quê"
4. **Revisão por PR** — mudanças em produção passam por revisão (se tiver time)

### Script de setup inicial idempotente

```bash
#!/bin/bash
# /srv/scripts/setup-server.sh
# Idempotente: pode ser rodado múltiplas vezes sem efeitos colaterais

set -e

DEPLOY_USER="deploy"
APPS_DIR="/srv/apps"

# ── Verifica se usuário existe, cria se não ────────────
if ! id "$DEPLOY_USER" &>/dev/null; then
  adduser --disabled-password --gecos "" "$DEPLOY_USER"
  usermod -aG sudo,docker "$DEPLOY_USER"
  echo "✓ Usuário $DEPLOY_USER criado"
else
  echo "✓ Usuário $DEPLOY_USER já existe"
fi

# ── Cria estrutura de diretórios ──────────────────────
mkdir -p "$APPS_DIR"/{_infra/{traefik,monitoring,portainer},scripts}
chown -R "$DEPLOY_USER:$DEPLOY_USER" "$APPS_DIR"
chmod 750 "$APPS_DIR"
echo "✓ Estrutura de diretórios criada"

# ── Cria rede proxy se não existir ──────────────────
if ! docker network ls | grep -q "^.*proxy"; then
  docker network create proxy
  echo "✓ Rede proxy criada"
else
  echo "✓ Rede proxy já existe"
fi

echo "Setup concluído."
```

---

## 4. Processo de Mudança em Produção

### Checklist antes de qualquer mudança

```markdown
## Checklist de Deploy

- [ ] Mudança testada em ambiente local ou staging
- [ ] Backup recente do banco de dados (< 24h)
- [ ] docker-compose.yml revisado e salvo no git
- [ ] Variáveis de ambiente verificadas no .env da VPS
- [ ] Plano de rollback conhecido
- [ ] Horário de menor tráfego (se possível)
```

### Janela de manutenção

Para mudanças de maior risco, comunique usuários previamente:

```bash
# Cria página de manutenção temporária no Traefik
# Adiciona label no serviço para redirecionar para página de manutenção
- "traefik.http.routers.minha-app.middlewares=maintenance@file"
```

---

## 5. Rotinas de Manutenção

### Rotina semanal

```bash
#!/bin/bash
# /srv/scripts/manutencao-semanal.sh
# Agende com: 0 4 * * 0 /srv/scripts/manutencao-semanal.sh >> /var/log/manutencao.log 2>&1

echo "=== Manutenção semanal $(date) ==="

# 1. Atualiza pacotes do sistema
sudo apt update && sudo apt upgrade -y

# 2. Limpa recursos Docker não utilizados
docker system prune -f
docker volume prune -f --filter "label!=keep"

# 3. Verifica containers com status não-esperado
UNHEALTHY=$(docker ps --filter "health=unhealthy" --format "{{.Names}}")
if [ -n "$UNHEALTHY" ]; then
  echo "ATENÇÃO: Containers não saudáveis: $UNHEALTHY"
fi

# 4. Verifica uso de disco
DISK_USAGE=$(df -h / | tail -1 | awk '{print $5}' | tr -d '%')
if [ "$DISK_USAGE" -gt 70 ]; then
  echo "ATENÇÃO: Disco em ${DISK_USAGE}% de uso"
fi

echo "=== Manutenção concluída ==="
```

### Rotina diária de verificação rápida

```bash
#!/bin/bash
# /srv/scripts/health-check.sh

echo "=== Status dos Containers ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.RunningFor}}"

echo ""
echo "=== Uso de Recursos ==="
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

echo ""
echo "=== Disco ==="
df -h / | tail -1
du -sh /srv/apps/*/ 2>/dev/null | sort -h

echo ""
echo "=== Últimos Logins ==="
last -n 5
```

---

## 6. Controle por Ambiente

Separe claramente as configurações de desenvolvimento e produção:

```
.env.development  → valores locais (não vai pro git)
.env.production   → valores de produção na VPS (não vai pro git)
.env.example      → template (vai pro git)
```

No `docker-compose.yml`:

```yaml
services:
  app:
    env_file:
      - .env # Valor do ambiente atual
    environment:
      # Valores que podem ser controlados por variável de ambiente do pipeline
      - APP_VERSION=${APP_VERSION:-dev}
      - DEPLOYMENT_DATE=${DEPLOYMENT_DATE:-unknown}
```

---

## 7. Documentação Interna

Cada aplicação deve ter um `README.md` no diretório `/srv/apps/<app-name>/`:

````markdown
# minha-app

**Repositório:** https://github.com/org/minha-app
**URL de produção:** https://minha-app.seudominio.com
**Responsável:** Nome do Dev

## Serviços

- `app` — Aplicação Node.js na porta 3000
- `db` — PostgreSQL 16
- `redis` — Redis 7 (cache de sessões)

## Deploy

```bash
# Deploy manual
docker compose pull && docker compose up -d --remove-orphans

# Ou via GitHub Actions (automático ao push em main)
```
````

## Rollback

```bash
# Edite a tag no docker-compose.yml e execute:
docker compose up -d --no-deps app
```

## Backup

- Banco: cronjob diário às 02:00, retém 30 dias
- Arquivos: incluídos no backup de volumes diário
- Storage remoto: Backblaze B2, bucket `meus-backups`

## Variáveis de Ambiente

Consulte `.env.example` para a lista completa de variáveis necessárias.

```

```

---

## 8. Checklist Final da Infraestrutura

### Segurança

- [ ] Usuário não-root com SSH por chave configurado
- [ ] Login root desabilitado
- [ ] Autenticação por senha desabilitada
- [ ] Firewall UFW ativo
- [ ] Fail2ban configurado
- [ ] Atualizações automáticas de segurança ativas

### Docker e Aplicações

- [ ] Docker Engine instalado pelo método oficial
- [ ] Rede `proxy` criada
- [ ] Reverse proxy (Traefik ou Nginx) configurado
- [ ] HTTPS automático funcionando
- [ ] Todas as apps com `restart: unless-stopped`
- [ ] Todas as apps com healthcheck definido
- [ ] Rotação de logs configurada

### Observabilidade

- [ ] Prometheus + Grafana instalados (ou Uptime Kuma no mínimo)
- [ ] Loki + Promtail para centralização de logs
- [ ] Alertas de RAM, disco e CPU configurados
- [ ] Uptime de todas as URLs monitorado

### Backup

- [ ] Backup de banco de dados automatizado
- [ ] Backup de volumes automatizado
- [ ] Envio para storage externo configurado (rclone)
- [ ] Restauração testada pelo menos uma vez

### DevOps

- [ ] Repositório de infraestrutura criado e atualizado
- [ ] Arquivos .env fora do git
- [ ] .env.example atualizado para cada app
- [ ] Pipeline CI/CD configurado para apps principais
- [ ] README de operação em cada app

---

## Conclusão

Você agora tem uma infraestrutura profissional, segura e pronta para produção. Os principais pontos são:

1. **Segurança em camadas**: SSH seguro + Firewall + Fail2ban + atualizações automáticas
2. **Isolamento por containers**: cada app em seu próprio compose com rede isolada
3. **Reverse proxy centralizado**: Traefik gerencia HTTPS e roteamento automaticamente
4. **Observabilidade**: métricas, logs e alertas para detectar problemas proativamente
5. **Backups automatizados**: proteção dos dados com estratégia 3-2-1
6. **Infraestrutura como código**: tudo versionado, documentado e reproduzível

Para evoluir a infraestrutura no futuro, considere:

- **Kubernetes** quando uma única VPS não for mais suficiente
- **Terraform** para provisionar servidores automaticamente
- **Ansible** para configurar múltiplos servidores com um único comando
- **ArgoCD** para GitOps em ambientes Kubernetes

Mas antes de escalar, certifique-se de que o básico está bem feito — e com este guia, está. ✓
