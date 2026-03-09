# Gerenciamento de Containers

> **Pré-requisito:** [Arquitetura multi-app](./multi-app-architecture.md) configurada.

---

## 1. Políticas de Reinício (restart policy)

Sempre configure a política de reinício nos seus serviços para garantir que os containers voltem automaticamente após uma reinicialização do servidor ou falha:

```yaml
services:
  app:
    image: minha-app:1.0
    restart: unless-stopped # ← Recomendado para produção
```

| Política         | Comportamento                                               |
| ---------------- | ----------------------------------------------------------- |
| `no`             | Nunca reinicia automaticamente (padrão)                     |
| `always`         | Sempre reinicia, mesmo se parado manualmente                |
| `on-failure`     | Reinicia apenas se o container saiu com erro                |
| `unless-stopped` | Reinicia sempre, exceto se parado manualmente pelo operador |

> **Recomendação:** Use `unless-stopped` para todos os serviços de produção. Ele garante reinício automático após reboot da VPS sem reiniciar serviços que você parou intencionalmente para manutenção.

---

## 2. Operações do Dia a Dia

### Gerenciar uma aplicação específica

```bash
cd /srv/apps/minha-app

# Iniciar todos os serviços
docker compose up -d

# Parar todos os serviços (sem remover containers)
docker compose stop

# Reiniciar todos os serviços
docker compose restart

# Parar e remover containers (mantém volumes)
docker compose down

# Ver status dos containers
docker compose ps

# Ver logs em tempo real
docker compose logs -f

# Ver logs dos últimos 100 linhas
docker compose logs --tail=100

# Ver logs de um serviço específico
docker compose logs -f app
```

### Executar comandos dentro de containers

```bash
# Abre shell interativo no container da aplicação
docker compose exec app sh

# Abre shell interativo no PostgreSQL
docker compose exec db psql -U app_user -d minha_app_db

# Executa um comando único sem abrir shell
docker compose exec app node -e "console.log(process.env.NODE_ENV)"

# Como root dentro do container (útil para debug)
docker compose exec -u root app sh
```

---

## 3. Atualização Segura de Containers

### Processo de atualização sem downtime

```bash
cd /srv/apps/minha-app

# 1. Faz pull das novas imagens sem derrubar o serviço ainda
docker compose pull

# 2. Verifica o que mudou (opcional)
docker compose images

# 3. Recria os containers com as novas imagens (downtime mínimo)
docker compose up -d --remove-orphans

# 4. Verifica se todos os containers estão saudáveis
docker compose ps

# 5. Verifica logs por erros nos primeiros minutos
docker compose logs -f --tail=50
```

### Usando tags específicas para controle de versão

```yaml
# docker-compose.yml — sempre use tags específicas em produção
services:
  app:
    image: minha-org/minha-app:1.5.2 # ← tag fixa
```

Para atualizar, mude a tag no arquivo e execute:

```bash
# Muda a versão e aplica
sed -i 's/minha-app:1.5.2/minha-app:1.5.3/' docker-compose.yml
docker compose up -d
```

---

## 4. Rollback de Aplicações

### Rollback por tag de imagem

```bash
cd /srv/apps/minha-app

# Volta para a versão anterior (substitua pela tag correta)
sed -i 's/minha-app:1.5.3/minha-app:1.5.2/' docker-compose.yml

# Aplica o rollback
docker compose up -d

# Verifica se está funcionando
docker compose ps
docker compose logs -f app
```

### Manter imagens anteriores localmente para rollback rápido

```bash
# Lista imagens disponíveis localmente
docker images minha-org/minha-app

# Faz pull de uma versão específica para tê-la disponível
docker pull minha-org/minha-app:1.5.2

# Rollback instantâneo sem precisar baixar
docker compose up -d  # (com a tag já ajustada no compose file)
```

### Script de rollback automatizado

```bash
# /srv/apps/minha-app/rollback.sh
#!/bin/bash
set -e

if [ -z "$1" ]; then
  echo "Uso: ./rollback.sh <versao>"
  echo "Exemplo: ./rollback.sh 1.5.2"
  exit 1
fi

TARGET_VERSION=$1
COMPOSE_FILE="docker-compose.yml"

echo "→ Iniciando rollback para versão $TARGET_VERSION..."

# Faz backup do arquivo compose atual
cp $COMPOSE_FILE ${COMPOSE_FILE}.bak

# Atualiza a versão
sed -i "s/minha-app:[0-9.]*/minha-app:$TARGET_VERSION/" $COMPOSE_FILE

# Aplica
docker compose up -d

echo "✓ Rollback para $TARGET_VERSION concluído."
echo "  Verifique: docker compose logs -f app"
```

---

## 5. Healthchecks

Defina healthchecks para que o Docker saiba quando um serviço está realmente pronto:

```yaml
services:
  app:
    image: minha-app:1.0
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s # verifica a cada 30s
      timeout: 10s # considera falha após 10s sem resposta
      retries: 3 # 3 falhas consecutivas = unhealthy
      start_period: 40s # aguarda 40s antes da primeira verificação

  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

```bash
# Verifica status de saúde dos containers
docker compose ps

# Inspeciona healthcheck de um container específico
docker inspect --format='{{json .State.Health}}' minha-app | jq
```

---

## 6. Gerenciamento de Logs

### Logs com Docker Compose

```bash
# Todos os serviços
docker compose logs

# Com timestamp
docker compose logs -t

# Seguir em tempo real
docker compose logs -f

# Somente erros (filtrando)
docker compose logs app 2>&1 | grep -i error
```

### Configuração de rotação de logs por serviço

```yaml
services:
  app:
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "5"
```

### Localização dos logs no host

```bash
# Encontra o arquivo de log de um container específico
docker inspect --format='{{.LogPath}}' minha-app

# Lê o log diretamente
sudo tail -f $(docker inspect --format='{{.LogPath}}' minha-app)
```

---

## 7. Limpeza de Recursos

Containers, imagens e volumes antigos acumulam espaço em disco. Execute regularmente:

```bash
# Remove containers parados, imagens não utilizadas, e redes ociosas
docker system prune -f

# Inclui volumes não utilizados por NENHUM container (CUIDADO!)
docker system prune -f --volumes

# Versão mais seletiva e segura:
docker container prune -f    # Remove containers parados
docker image prune -f        # Remove imagens dangling (sem tag)
docker network prune -f      # Remove redes não utilizadas

# Ver uso de disco pelo Docker
docker system df
docker system df -v          # Versão detalhada
```

### Automação com cronjob semanal

```bash
# Adiciona limpeza automática, sem afetar containers rodando
(crontab -l 2>/dev/null; echo "0 3 * * 0 docker system prune -f >> /var/log/docker-prune.log 2>&1") | crontab -
```

---

## 8. Inspecionar e Debugar Containers

```bash
# Inspeciona todas as configurações de um container
docker inspect minha-app

# Verifica variáveis de ambiente de um container rodando
docker exec minha-app env

# Verifica uso de recursos em tempo real
docker stats

# Uso de recursos de containers específicos
docker stats minha-app minha-app-db

# Processos rodando dentro de um container
docker top minha-app

# Diferença entre imagem e container atual
docker diff minha-app
```

---

## Próximo Passo

Continue com a configuração do **[Reverse Proxy](./reverse-proxy.md)** para rotear tráfego para cada aplicação com HTTPS automático.
