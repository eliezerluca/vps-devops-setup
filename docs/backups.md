# Backup e Recuperação

> **Regra de ouro dos backups:** Você só sabe se o backup funciona quando testa a restauração. Automatize backups e teste-os regularmente.

---

## Estratégia 3-2-1

A estratégia **3-2-1** é o padrão da indústria para backups resilientes:

- **3** cópias dos dados
- **2** em mídias/locais diferentes
- **1** cópia fora do servidor (offsite)

```
VPS (produção)
    │
    ├── /srv/apps/minha-app/data/backups/  ← Cópia local (1ª cópia)
    │
    └── Storage externo ─── S3 / Backblaze B2 / Wasabi  ← Offsite (2ª/3ª cópia)
```

---

## O Que Fazer Backup

| Dado                                     | Frequência     | Retenção   |
| ---------------------------------------- | -------------- | ---------- |
| Banco de dados (PostgreSQL/MySQL)        | Diário         | 30 dias    |
| Volumes Docker (uploads, arquivos)       | Diário         | 30 dias    |
| Arquivos de configuração (.env, compose) | A cada mudança | Indefinido |
| Servidor completo (snapshot)             | Semanal        | 4 semanas  |

---

## 1. Backup de Banco de Dados PostgreSQL

### Script de backup do PostgreSQL

```bash
#!/bin/bash
# /srv/scripts/backup-postgres.sh

set -e

# Configuração
APP_NAME="minha-app"
CONTAINER_NAME="${APP_NAME}-db"
DB_USER="app_user"
DB_NAME="minha_app_db"
BACKUP_DIR="/srv/apps/${APP_NAME}/data/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="${BACKUP_DIR}/db_${DATE}.sql.gz"
RETENTION_DAYS=30

# Cria diretório se não existir
mkdir -p "$BACKUP_DIR"

echo "[$(date)] Iniciando backup do banco ${DB_NAME}..."

# Executa o dump e comprime
docker exec "$CONTAINER_NAME" \
  pg_dump -U "$DB_USER" -d "$DB_NAME" --no-password \
  | gzip > "$BACKUP_FILE"

echo "[$(date)] Backup criado: $BACKUP_FILE ($(du -sh $BACKUP_FILE | cut -f1))"

# Remove backups mais antigos que RETENTION_DAYS
find "$BACKUP_DIR" -name "db_*.sql.gz" -mtime +$RETENTION_DAYS -delete
echo "[$(date)] Backups com mais de ${RETENTION_DAYS} dias removidos."
```

### Script de backup do MySQL/MariaDB

```bash
#!/bin/bash
# /srv/scripts/backup-mysql.sh

APP_NAME="minha-app"
CONTAINER_NAME="${APP_NAME}-db"
BACKUP_DIR="/srv/apps/${APP_NAME}/data/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)

mkdir -p "$BACKUP_DIR"

docker exec "$CONTAINER_NAME" \
  mysqldump -u root -p"${MYSQL_ROOT_PASSWORD}" --all-databases \
  | gzip > "${BACKUP_DIR}/db_${DATE}.sql.gz"

find "$BACKUP_DIR" -name "db_*.sql.gz" -mtime +30 -delete
```

---

## 2. Backup de Volumes Docker

### Backup de bind mounts (pastas montadas)

```bash
#!/bin/bash
# /srv/scripts/backup-volumes.sh

APP_NAME="minha-app"
SOURCE_DIR="/srv/apps/${APP_NAME}/data"
BACKUP_DIR="/srv/apps/${APP_NAME}/data/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="${BACKUP_DIR}/volumes_${DATE}.tar.gz"

mkdir -p "$BACKUP_DIR"

# Comprime o diretório de dados (exclui o diretório de backups em si)
tar -czf "$BACKUP_FILE" \
  --exclude="${SOURCE_DIR}/backups" \
  -C "$(dirname $SOURCE_DIR)" \
  "$(basename $SOURCE_DIR)"

echo "Backup criado: $BACKUP_FILE ($(du -sh $BACKUP_FILE | cut -f1))"

# Retém últimos 30 dias
find "$BACKUP_DIR" -name "volumes_*.tar.gz" -mtime +30 -delete
```

### Backup de volumes Docker nomeados

```bash
#!/bin/bash
# Backup de um volume Docker nomeado

VOLUME_NAME="minha-app_db_data"
BACKUP_DIR="/srv/backups/volumes"
DATE=$(date +%Y-%m-%d_%H-%M-%S)

mkdir -p "$BACKUP_DIR"

# Usa um container temporário para acessar o volume
docker run --rm \
  -v "${VOLUME_NAME}:/volume:ro" \
  -v "${BACKUP_DIR}:/backup" \
  alpine \
  tar -czf "/backup/${VOLUME_NAME}_${DATE}.tar.gz" -C /volume .

echo "Backup do volume ${VOLUME_NAME} concluído."
```

---

## 3. Envio para Storage Externo (S3/Backblaze)

### Usando rclone para sincronizar com qualquer cloud storage

```bash
# Instala o rclone
curl https://rclone.org/install.sh | sudo bash

# Configura o rclone (interactive)
rclone config
# Siga as instruções para adicionar seu provider (S3, Backblaze B2, Wasabi, etc.)
```

Exemplo de configuração para Backblaze B2 (`~/.config/rclone/rclone.conf`):

```ini
[b2-backups]
type = b2
account = SUA_ACCOUNT_ID
key = SUA_APPLICATION_KEY
```

Script de upload automático:

```bash
#!/bin/bash
# /srv/scripts/upload-backups.sh

BACKUP_DIR="/srv/apps/minha-app/data/backups"
REMOTE="b2-backups:meu-bucket-backups/minha-app"

# Upload e sincroniza (mantém apenas os 30 últimos dias no remoto também)
rclone sync "$BACKUP_DIR" "$REMOTE" \
  --min-age 0 \
  --log-level INFO \
  --log-file /var/log/rclone-backup.log

echo "[$(date)] Upload de backups para $REMOTE concluído."
```

---

## 4. Script de Backup Completo por Aplicação

```bash
#!/bin/bash
# /srv/scripts/backup-app.sh <nome-da-app>
# Uso: ./backup-app.sh minha-app

set -e

APP_NAME="${1:-minha-app}"
APP_DIR="/srv/apps/${APP_NAME}"
BACKUP_BASE="${APP_DIR}/data/backups"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
LOG_FILE="/var/log/backup-${APP_NAME}.log"

# Carrega variáveis do .env da aplicação
if [ -f "${APP_DIR}/.env" ]; then
  export $(grep -v '^#' "${APP_DIR}/.env" | xargs)
fi

echo "[${DATE}] === Iniciando backup de ${APP_NAME} ===" | tee -a "$LOG_FILE"

# 1. Backup do banco (PostgreSQL)
if docker ps --format '{{.Names}}' | grep -q "${APP_NAME}-db"; then
  BACKUP_FILE="${BACKUP_BASE}/db_${DATE}.sql.gz"
  docker exec "${APP_NAME}-db" \
    pg_dump -U "${POSTGRES_USER}" -d "${POSTGRES_DB}" \
    | gzip > "$BACKUP_FILE"
  echo "[${DATE}] ✓ Banco de dados: $(du -sh $BACKUP_FILE | cut -f1)" | tee -a "$LOG_FILE"
fi

# 2. Backup de arquivos/uploads
if [ -d "${APP_DIR}/data/uploads" ]; then
  UPLOADS_FILE="${BACKUP_BASE}/uploads_${DATE}.tar.gz"
  tar -czf "$UPLOADS_FILE" -C "${APP_DIR}/data" uploads/
  echo "[${DATE}] ✓ Uploads: $(du -sh $UPLOADS_FILE | cut -f1)" | tee -a "$LOG_FILE"
fi

# 3. Backup das configurações
CONFIG_FILE="${BACKUP_BASE}/config_${DATE}.tar.gz"
tar -czf "$CONFIG_FILE" \
  -C "${APP_DIR}" \
  docker-compose.yml \
  .env.example \
  2>/dev/null || true

# 4. Upload para storage externo
if command -v rclone &> /dev/null; then
  rclone sync "${BACKUP_BASE}" "b2-backups:meu-bucket/${APP_NAME}" \
    --log-level INFO >> "$LOG_FILE" 2>&1
  echo "[${DATE}] ✓ Upload para cloud concluído" | tee -a "$LOG_FILE"
fi

# 5. Limpa backups locais antigos (> 30 dias)
find "$BACKUP_BASE" -name "*.gz" -mtime +30 -delete
echo "[${DATE}] ✓ Limpeza de backups antigos concluída" | tee -a "$LOG_FILE"

echo "[${DATE}] === Backup de ${APP_NAME} finalizado ===" | tee -a "$LOG_FILE"
```

```bash
# Torna o script executável
chmod +x /srv/scripts/backup-app.sh

# Testa manualmente
/srv/scripts/backup-app.sh minha-app
```

---

## 5. Automação com Cron

```bash
# Edita o crontab do usuário deploy
crontab -e
```

```cron
# Backup do banco de dados: todo dia às 02:00
0 2 * * * /srv/scripts/backup-app.sh minha-app >> /var/log/backup-minha-app.log 2>&1

# Backup de outra app: todo dia às 02:30
30 2 * * * /srv/scripts/backup-app.sh outra-app >> /var/log/backup-outra-app.log 2>&1

# Limpa logs antigos do docker system todo domingo às 03:00
0 3 * * 0 docker system prune -f >> /var/log/docker-prune.log 2>&1
```

```bash
# Verifica se os cronjobs estão corretos
crontab -l

# Monitora execução dos backups
tail -f /var/log/backup-minha-app.log
```

---

## 6. Restauração de Backup

### Restaurar banco de dados PostgreSQL

```bash
# Descomprime e restaura
gunzip < /srv/apps/minha-app/data/backups/db_2024-01-15_02-00-00.sql.gz | \
  docker exec -i minha-app-db \
  psql -U app_user -d minha_app_db

# Verifica se a restauração foi bem-sucedida
docker exec minha-app-db \
  psql -U app_user -d minha_app_db -c "\dt"
```

### Restaurar volumes/arquivos

```bash
# Para a aplicação antes de restaurar arquivos
cd /srv/apps/minha-app
docker compose stop app

# Restaura os arquivos
tar -xzf data/backups/uploads_2024-01-15_02-00-00.tar.gz -C data/

# Reinicia a aplicação
docker compose start app
```

---

## 7. Snapshot da VPS (Backup do Servidor Inteiro)

A maioria dos provedores de VPS oferece snapshot via painel ou API:

```bash
# DigitalOcean (com API)
curl -X POST \
  -H "Authorization: Bearer $DO_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.digitalocean.com/v2/droplets/<DROPLET_ID>/actions" \
  -d '{"type":"snapshot","name":"backup-manual-2024-01-15"}'

# Hetzner Cloud
hcloud server create-image --type snapshot --description "backup-semanal" <SERVER_ID>
```

> **Snapshots vs Backups de dados:** Snapshots capturam o servidor inteiro (disco completo), mas são lentos e caros. Use-os semanalmente e complemente com backups de dados diários.

---

## Checklist de Backup

- [ ] Script de backup de banco de dados criado e testado
- [ ] Script de backup de volumes criado e testado
- [ ] Rclone configurado e apontando para storage externo
- [ ] Cronjobs configurados para execução automática
- [ ] Restauração testada com sucesso pelo menos uma vez
- [ ] Snapshots semanais configurados no painel do provedor

---

## Próximo Passo

Finalize com as **[Boas Práticas DevOps](./devops-best-practices.md)** para manter a infraestrutura organizada e escalável ao longo do tempo.
