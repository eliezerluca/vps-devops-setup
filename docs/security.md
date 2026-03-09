# Segurança do Servidor

> **Pré-requisito:** [Configuração inicial do servidor](./server-initial-setup.md) concluída.

---

## 1. Fail2ban — Proteção contra Brute-Force

O Fail2ban monitora logs do sistema e bane automaticamente IPs que apresentam comportamento suspeito (múltiplas tentativas de login falhas, por exemplo).

### Instalação

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Configuração

O Fail2ban usa um sistema de arquivos `.local` que sobrescreve as configurações padrão sem modificá-las — isso facilita atualizações.

```bash
# Cria arquivo de configuração local
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo vim /etc/fail2ban/jail.local
```

Modifique as seções relevantes:

```ini
[DEFAULT]
# Tempo de banimento em segundos (padrão: 10min → aumentar para 1h)
bantime  = 3600

# Janela de tempo para contar tentativas falhas (em segundos)
findtime  = 600

# Número de falhas antes do banimento
maxretry = 5

# Seu IP de casa ou VPN (nunca será banido)
ignoreip = 127.0.0.1/8 ::1 <SEU_IP_LOCAL>

# Backend de monitoramento
backend = systemd

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 86400
```

```bash
# Reinicia o Fail2ban para aplicar configurações
sudo systemctl restart fail2ban

# Verifica status e jails ativos
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

### Comandos úteis do Fail2ban

```bash
# Ver IPs atualmente banidos
sudo fail2ban-client status sshd

# Desbanir um IP manualmente
sudo fail2ban-client set sshd unbanip <IP>

# Ver logs do Fail2ban
sudo tail -f /var/log/fail2ban.log

# Testar configuração
sudo fail2ban-client -t
```

---

## 2. Atualizações Automáticas de Segurança

Configure o sistema para instalar automaticamente patches de segurança críticos:

```bash
sudo apt install -y unattended-upgrades apt-listchanges

# Ativa as atualizações automáticas de segurança
sudo dpkg-reconfigure -plow unattended-upgrades
```

Edite o arquivo de configuração para personalizar:

```bash
sudo vim /etc/apt/apt.conf.d/50unattended-upgrades
```

```ini
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}";
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
    "${distro_id}ESM:${distro_codename}-infra-security";
};

// Remover dependencias obsoletas automaticamente
Unattended-Upgrade::Remove-Unused-Dependencies "true";

// Reiniciar automaticamente se necessario (CUIDADO: pode derrubar apps)
// Deixe false e cuide de reinícios manualmente
Unattended-Upgrade::Automatic-Reboot "false";

// Notificação por email (opcional)
//Unattended-Upgrade::Mail "voce@seudominio.com";
```

Configure a periodicidade:

```bash
sudo vim /etc/apt/apt.conf.d/20auto-upgrades
```

```ini
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::Unattended-Upgrade "1";
```

```bash
# Testa a execução (modo simulação)
sudo unattended-upgrades --dry-run --debug
```

---

## 3. Proteção com CrowdSec (Alternativa Moderna ao Fail2ban)

O **CrowdSec** é uma alternativa moderna ao Fail2ban que utiliza inteligência coletiva — IPs maliciosos identificados por toda a comunidade são automaticamente bloqueados em todos os servidores participantes.

```bash
# Adiciona repositório oficial
curl -s https://packagecloud.io/install/repositories/crowdsec/crowdsec/script.deb.sh | sudo bash

# Instala o CrowdSec
sudo apt install -y crowdsec

# Instala o bouncer para iptables/nftables
sudo apt install -y crowdsec-firewall-bouncer-iptables

# Verifica cenários ativos
sudo cscli scenarios list

# Verifica decisões (IPs banidos)
sudo cscli decisions list

# Verifica alertas
sudo cscli alerts list
```

> **CrowdSec vs Fail2ban:** Use Fail2ban para simplicidade ou CrowdSec para proteção mais avançada com crowdsourcing de threat intelligence. Para a maioria dos casos, Fail2ban é suficiente.

---

## 4. Monitoramento de Logins e Acessos

### 4.1 Verificar últimos logins

```bash
# Últimos logins no sistema
last -n 20

# Tentativas de login falhas
lastb -n 20

# Quem está logado agora
who
w
```

### 4.2 Configurar alertas de login por email (opcional)

```bash
sudo apt install -y mailutils ssmtp
```

Adicione ao arquivo `/etc/profile.d/login_notify.sh`:

```bash
#!/bin/bash
# Notifica por email em cada login SSH
if [ -n "$SSH_CLIENT" ]; then
    IP=$(echo $SSH_CLIENT | awk '{print $1}')
    DATE=$(date '+%Y-%m-%d %H:%M:%S')
    echo "Login SSH: Usuário $USER conectou de $IP em $DATE" | \
        mail -s "[VPS] Login SSH detectado" seu@email.com
fi
```

```bash
sudo chmod +x /etc/profile.d/login_notify.sh
```

### 4.3 Auditd — Auditoria de sistema (avançado)

```bash
sudo apt install -y auditd audispd-plugins

# Regras básicas de auditoria
sudo auditctl -w /etc/passwd -p wa -k passwd_changes
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
sudo auditctl -w /var/log/auth.log -p r -k auth_log_read

# Persistir regras
sudo vim /etc/audit/rules.d/audit.rules
```

---

## 5. Permissões Corretas de Arquivos

### Permissões críticas do sistema

```bash
# Verifica permissões de arquivos sensíveis
ls -la /etc/passwd /etc/shadow /etc/sudoers

# Corrigi permissões se necessário
sudo chmod 644 /etc/passwd
sudo chmod 600 /etc/shadow
sudo chmod 440 /etc/sudoers

# Verifica arquivos com SUID/SGID (potencial vetor de escalada de privilégios)
find / -perm /6000 -type f 2>/dev/null | grep -v proc
```

### Permissões para o diretório de aplicações

```bash
# Diretório base das aplicações (criado no guia de arquitetura)
sudo mkdir -p /srv/apps
sudo chown -R deploy:deploy /srv/apps
sudo chmod 750 /srv/apps
```

---

## 6. Limite de Processos e Recursos (Opcional)

Configure limites para evitar que um processo consuma todos os recursos:

```bash
sudo vim /etc/security/limits.conf
```

```ini
# Limita número máximo de processos por usuário
deploy    soft    nproc    1024
deploy    hard    nproc    2048

# Limita arquivos abertos por usuário
deploy    soft    nofile   65536
deploy    hard    nofile   131072
```

---

## 7. Desativar Serviços Desnecessários

Reduza a superfície de ataque desativando serviços que não usa:

```bash
# Lista de serviços em execução
systemctl list-units --type=service --state=running

# Exemplos de serviços geralmente desnecessários em VPS
sudo systemctl disable --now snapd
sudo systemctl disable --now bluetooth
sudo systemctl disable --now cups
sudo systemctl disable --now avahi-daemon
```

---

## 8. Verificação de Segurança com Lynis

O Lynis faz uma auditoria completa do servidor e lista recomendações de segurança:

```bash
# Instala o Lynis
sudo apt install -y lynis

# Executa auditoria completa
sudo lynis audit system

# O relatório fica em:
cat /var/log/lynis.log
cat /var/log/lynis-report.dat
```

O Lynis atribui um **Hardening Index** (0-100) e lista itens para melhorar. Alveje um score acima de 70 para servidores de produção.

---

## 9. Checklist de Segurança

- [ ] Fail2ban instalado e configurado
- [ ] Atualizações automáticas de segurança ativas
- [ ] Permissões de arquivos verificadas
- [ ] Serviços desnecessários desativados
- [ ] Logs de acesso monitorados
- [ ] Auditoria com Lynis executada

---

## Próximo Passo

Continue com a **[Instalação do Docker](./docker-installation.md)** para preparar o ambiente de containers.
