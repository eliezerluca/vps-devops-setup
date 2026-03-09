# Monitoramento e Observabilidade

> O monitoramento é o que diferencia uma infraestrutura profissional de um servidor "esperando quebrar". Com ele, você descobre problemas antes dos usuários.

---

## Pilares da Observabilidade

```
Métricas      → "O servidor está consumindo 95% de RAM"
Logs          → "O container de banco caiu às 03:17"
Traces        → "Essa requisição demorou 2s na query X" (avançado)
Alertas       → "Avise-me quando a RAM passar de 85%"
```

Este guia foca em **métricas e alertas**, cobrindo desde opções simples até a stack completa.

---

## Nível 1 — Monitoramento Básico do Servidor (Sem Docker)

Antes de qualquer ferramenta sofisticada, tenha estes comandos na ponta dos dedos:

```bash
# Uso de CPU e memória em tempo real
htop

# Uso de disco
df -h
du -sh /srv/apps/*

# Processos que mais consomem recursos
top -o %MEM
ps aux --sort=-%mem | head -10

# Uso de rede
iftop           # requer: sudo apt install iftop
nethogs         # requer: sudo apt install nethogs

# Conexões de rede abertas
ss -tuln

# Tempo de atividade do sistema
uptime
```

---

## Nível 2 — Monitoramento de Containers com ctop

```bash
# Instala o ctop (top para containers Docker)
sudo wget https://github.com/bcicen/ctop/releases/latest/download/ctop-0.7.7-linux-amd64 \
  -O /usr/local/bin/ctop
sudo chmod +x /usr/local/bin/ctop

# Usa o ctop
ctop
```

Atalhos do ctop:

- `q` — sair
- `s` — ordenar por coluna
- `Enter` — ver detalhes do container selecionado
- `l` — ver logs do container

---

## Nível 3 — Stack Prometheus + Grafana

A stack **Prometheus + Grafana** é o padrão da indústria para monitoramento em 2026. É poderosa, gratuita, e roda perfeitamente em containers.

```
Exporters → Prometheus → Grafana
(coletam)   (armazena)   (visualiza)
```

### Estrutura de arquivos

```
/srv/apps/_infra/monitoring/
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
└── alertmanager/
    └── alertmanager.yml
```

### Docker Compose da Stack

```yaml
# /srv/apps/_infra/monitoring/docker-compose.yml
version: "3.9"

services:
  # ── Prometheus ────────────────────────────────────────
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.seudominio.com`)"
      - "traefik.http.routers.prometheus.entrypoints=websecure"
      - "traefik.http.routers.prometheus.tls.certresolver=letsencrypt"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"
      # Adicione middleware de autenticação básica (veja reverse-proxy.md)
    networks:
      - monitoring
      - proxy

  # ── Grafana ───────────────────────────────────────────
  grafana:
    image: grafana/grafana:10.4.0
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin123}
      - GF_SERVER_ROOT_URL=https://grafana.seudominio.com
    volumes:
      - grafana_data:/var/lib/grafana
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.seudominio.com`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls.certresolver=letsencrypt"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
    networks:
      - monitoring
      - proxy

  # ── Node Exporter (métricas do servidor host) ─────────
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    restart: unless-stopped
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.rootfs=/rootfs"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    networks:
      - monitoring

  # ── cAdvisor (métricas de containers) ─────────────────
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
      - /cgroup:/cgroup:ro
    networks:
      - monitoring

  # ── Alertmanager (envio de alertas) ──────────────────
  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

networks:
  monitoring:
    name: monitoring
    driver: bridge
  proxy:
    external: true
```

### Configuração do Prometheus

```yaml
# /srv/apps/_infra/monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["alertmanager:9093"]

rule_files:
  - "alerts/*.yml"

scrape_configs:
  # Prometheus monitorando a si mesmo
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Métricas do servidor host
  - job_name: "node"
    static_configs:
      - targets: ["node-exporter:9100"]

  # Métricas de containers Docker
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]

  # Adicione aqui endpoints das suas aplicações (se tiverem /metrics)
  # - job_name: "minha-app"
  #   static_configs:
  #     - targets: ["minha-app:3000"]
  #   metrics_path: /metrics
```

### Configuração de Alertas

```yaml
# /srv/apps/_infra/monitoring/prometheus/alerts/node.yml
groups:
  - name: node_alerts
    rules:
      # Alerta quando RAM passa de 85%
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Uso de memória alto no servidor"
          description: "Uso de memória está em {{ humanize $value }}%"

      # Alerta quando disco passa de 80%
      - alert: HighDiskUsage
        expr: (1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})) * 100 > 80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Espaço em disco crítico"
          description: "Disco está {{ humanize $value }}% cheio"

      # Alerta quando la1 de CPU está alto
      - alert: HighCPULoad
        expr: node_load1 > 4.0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alta carga de CPU"
          description: "Load average está em {{ $value }}"
```

### Configuração do Alertmanager (notificação por email)

```yaml
# /srv/apps/_infra/monitoring/alertmanager/alertmanager.yml
global:
  smtp_smarthost: "smtp.gmail.com:587"
  smtp_from: "alertas@seudominio.com"
  smtp_auth_username: "seu@gmail.com"
  smtp_auth_password: "sua-app-password"

route:
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: "email-alertas"

receivers:
  - name: "email-alertas"
    email_configs:
      - to: "voce@email.com"
        subject: "[ALERTA VPS] {{ .GroupLabels.alertname }}"
        body: |
          {{ range .Alerts }}
          Alerta: {{ .Annotations.summary }}
          Descrição: {{ .Annotations.description }}
          {{ end }}
```

### Iniciando a stack de monitoramento

```bash
cd /srv/apps/_infra/monitoring
docker compose up -d

# Verifica os serviços
docker compose ps

# Acessa:
# Prometheus: https://prometheus.seudominio.com
# Grafana:    https://grafana.seudominio.com
```

### Importar dashboards prontos no Grafana

No Grafana: **Dashboards → Import**

IDs de dashboards recomendados:

- `1860` — Node Exporter Full (métricas do servidor)
- `893` — Docker and system monitoring
- `14282` — Traefik 2.x metrics

---

## Nível 4 — Uptime Monitoring com Uptime Kuma

O **Uptime Kuma** monitora se suas URLs estão respondendo e envia alertas quando caem:

```yaml
# /srv/apps/_infra/uptime-kuma/docker-compose.yml
version: "3.9"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    volumes:
      - uptime_data:/app/data
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.uptime.rule=Host(`status.seudominio.com`)"
      - "traefik.http.routers.uptime.entrypoints=websecure"
      - "traefik.http.routers.uptime.tls.certresolver=letsencrypt"
      - "traefik.http.services.uptime.loadbalancer.server.port=3001"
    networks:
      - proxy

volumes:
  uptime_data:

networks:
  proxy:
    external: true
```

Acesse em `https://status.seudominio.com` e adicione os monitores das suas aplicações.

---

## Métricas Essenciais para Monitorar

| Métrica                  | Threshold de Alerta           |
| ------------------------ | ----------------------------- |
| Uso de RAM               | > 85%                         |
| Uso de CPU (load1)       | > número de CPUs × 2          |
| Uso de disco             | > 80%                         |
| Container reiniciando    | Qualquer reinício inesperado  |
| URL respondendo          | Timeout > 5s ou status != 2xx |
| Certificado SSL vencendo | < 14 dias                     |

---

## Próximo Passo

Continue com o **[Gerenciamento de Logs](./logs.md)** para centralizar e analisar logs das suas aplicações.
