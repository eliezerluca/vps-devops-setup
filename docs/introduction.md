# Guia Completo: VPS Linux para Múltiplas Aplicações com Docker (2026)

## Introdução

Este guia foi criado para desenvolvedores iniciantes e intermediários que desejam configurar uma VPS Linux profissional capaz de hospedar entre **5 e 10 aplicações em produção**, utilizando Docker e as melhores práticas modernas de DevOps.

Ao final deste guia, você terá um servidor:

- **Seguro** — hardening de SSH, firewall, proteção contra brute-force
- **Organizado** — estrutura de diretórios clara para múltiplos projetos
- **Escalável** — arquitetura baseada em containers com reverse proxy
- **Observável** — monitoramento de métricas, logs e alertas
- **Automatizado** — deploy via Git/CI-CD com rollback

---

## Pré-requisitos

| Requisito | Descrição                                            |
| --------- | ---------------------------------------------------- |
| VPS       | Ubuntu 22.04 LTS ou Ubuntu 24.04 LTS (recomendado)   |
| RAM       | Mínimo 2 GB (recomendado 4 GB ou mais)               |
| Disco     | Mínimo 40 GB SSD                                     |
| Acesso    | Root via SSH                                         |
| Domínio   | Um domínio ou subdomínios apontando para o IP da VPS |

> **Nota sobre distribuições:** Ubuntu 24.04 LTS é a escolha recomendada para 2026 por ter suporte ativo até 2029. Debian 12 também é uma alternativa sólida.

---

## Arquitetura Geral

```
Internet
    │
    ▼
[Firewall UFW]
    │
    ▼
[Reverse Proxy - Traefik ou Nginx]
    │
    ├──► [App 1 - Docker Compose] → app1.dominio.com
    ├──► [App 2 - Docker Compose] → app2.dominio.com
    ├──► [App 3 - Docker Compose] → app3.dominio.com
    ├──► [App 4 - Docker Compose] → app4.dominio.com
    └──► [App 5+ - Docker Compose] → app5.dominio.com

[Volumes Docker]       → /srv/apps/<app>/data
[Banco de Dados]       → container isolado por app
[Logs]                 → Loki + Grafana (ou simples)
[Backups]              → cronjob + armazenamento externo
```

---

## Estrutura do Guia

| Arquivo                                                       | Conteúdo                                        |
| ------------------------------------------------------------- | ----------------------------------------------- |
| [Configuração Inicial do Servidor](./server-initial-setup.md) | SSH, usuários, firewall, hardening              |
| [Segurança](./security.md)                                    | Fail2ban, atualizações, permissões              |
| [Instalação do Docker](./docker-installation.md)              | Docker Engine, Docker Compose, boas práticas    |
| [Arquitetura Multi-App](./multi-app-architecture.md)          | Estrutura de diretórios, variáveis, organização |
| [Gerenciamento de Containers](./container-management.md)      | Compose, logs, reinício, atualização, rollback  |
| [Reverse Proxy](./reverse-proxy.md)                           | Traefik ou Nginx, HTTPS, múltiplos domínios     |
| [Ferramentas de Deploy](./deploy-tools.md)                    | Coolify, Dokploy, CapRover, Portainer           |
| [Estratégias de Deploy](./deployment-strategies.md)           | Manual, Git, GitHub Actions, GitLab CI          |
| [Monitoramento](./monitoring.md)                              | Prometheus, Grafana, métricas de containers     |
| [Gerenciamento de Logs](./logs.md)                            | Logs de containers, centralização, ferramentas  |
| [Backup e Recuperação](./backups.md)                          | Volumes, banco de dados, automação              |
| [Boas Práticas DevOps](./devops-best-practices.md)            | Organização, variáveis, versionamento           |

---

## Filosofia Deste Guia

### Por que Docker?

Em 2026, Docker se consolidou como o padrão de fato para empacotar e executar aplicações em produção. Os benefícios incluem:

- **Isolamento**: cada aplicação roda em seu próprio ambiente
- **Portabilidade**: o mesmo container funciona em qualquer lugar
- **Reprodutibilidade**: o ambiente de produção é idêntico ao de desenvolvimento
- **Facilidade de atualização e rollback**: trocar versões é questão de mudar uma tag

### Por que múltiplas aplicações em uma VPS?

Para projetos menores, startups ou portfólios, consolidar múltiplas aplicações em uma única VPS bem configurada é economicamente eficiente. Com Docker e um reverse proxy, é possível rodar 10+ aplicações em uma VPS de 4 GB de RAM sem conflito de portas ou dependências.

### Abordagem progressiva

O guia segue uma ordem lógica de configuração — do mais crítico (segurança SSH) ao mais avançado (CI/CD e monitoramento). Siga na sequência para garantir uma base sólida antes de avançar.

---

## Convencões Usadas Neste Guia

```bash
# Comandos executados como root ou com sudo
sudo apt update

# Comandos executados como o usuário deploy (não-root)
$ docker compose up -d

# Variáveis que você deve substituir pelo seu valor
<SEU_USUARIO>
<SEU_DOMINIO>
<IP_DA_VPS>
```

> **Blocos de atenção** destacam informações importantes ou pontos de cuidado.

---

## Próximo Passo

Comece pela **[Configuração Inicial do Servidor](./server-initial-setup.md)** para preparar a base segura da sua VPS.
