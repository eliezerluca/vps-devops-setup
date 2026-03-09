# Gerenciamento de Logs

> Uma boa estratégia de logs permite investigar problemas rapidamente, auditar comportamentos e identificar padrões de erro antes que virem incidentes.

---

## 1. Logs de Containers Docker

### Como o Docker armazena logs

Por padrão, o Docker captura tudo que os containers escrevem em `stdout` e `stderr` e armazena em arquivos JSON no host:

```bash
# Localização padrão dos logs de um container
/var/lib/docker/containers/<container-id>/<container-id>-json.log

# Encontra o arquivo de log de um container específico
docker inspect --format='{{.LogPath}}' minha-app
```

### Comandos essenciais para logs

```bash
# Ver logs de um serviço (dentro de um docker-compose)
docker compose logs app

# Seguir em tempo real
docker compose logs -f app

# Últimas 100 linhas
docker compose logs --tail=100 app

# Com timestamps
docker compose logs -t app

# Filtrar por tempo
docker compose logs --since="2024-01-15T10:00:00" app
docker compose logs --since="1h" app
docker compose logs --until="30m" app

# Combinando filtros
docker compose logs --since="2h" --tail=200 -f app
```

---

## 2. Configuração de Rotação de Logs

Sem rotação, os logs crescem indefinidamente e podem encher o disco. Configure a política de log no `daemon.json` (configuração global) ou no `docker-compose.yml` (por serviço).

### Configuração global (recomendado)

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

### Configuração por serviço (sobrescreve o global)

```yaml
services:
  app:
    image: minha-app:1.0
    logging:
      driver: "json-file"
      options:
        max-size: "20m" # Máximo 20 MB por arquivo
        max-file: "5" # Mantém os 5 últimos arquivos
        compress: "true" # Comprime arquivos antigos
```

---

## 3. Stack de Centralização de Logs — Loki + Grafana

O **Loki** (da Grafana Labs) é um sistema de agregação de logs leve e altamente eficiente. A combinação **Loki + Promtail + Grafana** é a solução moderna mais indicada para VPS em 2026, especialmente se você já usa Grafana para métricas.

```
Containers → Promtail → Loki → Grafana
(produzem)   (coleta)  (armazena)  (consulta)
```

### Estrutura de arquivos

```
/srv/apps/_infra/logging/
├── docker-compose.yml
├── loki/
│   └── loki-config.yml
└── promtail/
    └── promtail-config.yml
```

### Docker Compose — Loki + Promtail

```yaml
# /srv/apps/_infra/logging/docker-compose.yml
version: "3.9"

services:
  # ── Loki (servidor de logs) ───────────────────────────
  loki:
    image: grafana/loki:2.9.4
    container_name: loki
    restart: unless-stopped
    volumes:
      - ./loki/loki-config.yml:/etc/loki/local-config.yaml:ro
      - loki_data:/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring

  # ── Promtail (agente coletor de logs) ─────────────────
  promtail:
    image: grafana/promtail:2.9.4
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml:ro
      - /var/log:/var/log:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring

volumes:
  loki_data:

networks:
  monitoring:
    external: true
```

### Configuração do Loki

```yaml
# /srv/apps/_infra/logging/loki/loki-config.yml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d # Retém logs por 30 dias

compactor:
  working_directory: /loki/compactor
  retention_enabled: true
```

### Configuração do Promtail

```yaml
# /srv/apps/_infra/logging/promtail/promtail-config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # Coleta logs de todos os containers Docker
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      # Usa o nome do container como label
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: container
      # Usa a imagem como label
      - source_labels: ["__meta_docker_container_image"]
        target_label: image
      # Usa o compose project como label
      - source_labels: ["__meta_docker_compose_project"]
        target_label: compose_project
      # Usa o compose service como label
      - source_labels: ["__meta_docker_compose_service"]
        target_label: compose_service

  # Coleta logs do sistema (syslog, auth.log, etc.)
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*.log
```

### Inicia a stack de logging

```bash
cd /srv/apps/_infra/logging
docker compose up -d

# Verifica
docker compose ps
docker compose logs -f
```

### Configurar o Grafana para usar Loki como datasource

1. Acesse o Grafana
2. **Configuration → Data Sources → Add data source**
3. Selecione **Loki**
4. URL: `http://loki:3100`
5. Salve e teste

### Consultando logs no Grafana (LogQL)

```logql
# Todos os logs do container 'minha-app'
{container="minha-app"}

# Apenas erros
{container="minha-app"} |= "error"

# Container de banco de dados, filtra por termo
{compose_service="db"} |~ "FATAL|ERROR"

# Últimas 1 hora, container específico, conta erros
count_over_time({container="minha-app"} |= "error" [1h])

# Extrai campo JSON dos logs estruturados
{container="minha-app"} | json | status >= 400
```

---

## 4. Logs do Sistema Operacional

```bash
# Ver logs do sistema (systemd journal)
journalctl -xe

# Logs do SSH (tentativas de login)
journalctl -u sshd -f

# Logs do Docker daemon
journalctl -u docker -f

# Logs de autenticação
tail -f /var/log/auth.log

# Logs do kernel
journalctl -k

# Filtrar por prioridade (err, warning, info)
journalctl -p err

# Logs das últimas 2 horas
journalctl --since "2 hours ago"

# Logs de uma data específica
journalctl --since "2024-01-15" --until "2024-01-16"
```

---

## 5. Logs Estruturados nas Aplicações

Configure suas aplicações para emitir logs em formato JSON — isso facilita filtragem e análise no Loki/Grafana.

### Exemplo Node.js com Pino

```javascript
// logger.js
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  base: {
    service: "minha-app",
    env: process.env.NODE_ENV,
  },
});

// Uso:
logger.info({ userId: 123, action: "login" }, "Usuário autenticado");
logger.error({ err: error, requestId }, "Erro ao processar pagamento");
```

Saída JSON:

```json
{
  "level": "info",
  "time": "2024-01-15T10:30:00.000Z",
  "service": "minha-app",
  "env": "production",
  "userId": 123,
  "action": "login",
  "msg": "Usuário autenticado"
}
```

---

## 6. Checklist de Logs

- [ ] Rotação de logs configurada (evita disco cheio)
- [ ] Stack Loki + Promtail instalada
- [ ] Grafana configurado com datasource Loki
- [ ] Aplicações emitindo logs estruturados (JSON)
- [ ] Alertas configurados para erros frequentes

---

## Próximo Passo

Continue com a estratégia de **[Backup e Recuperação](./backups.md)** para proteger os dados das suas aplicações.
