# Configuração Inicial do Servidor

> **Pré-requisito:** Você acabou de criar uma VPS com Ubuntu 22.04 ou 24.04 LTS e tem acesso root via SSH.

---

## 1. Primeiro Acesso e Atualização do Sistema

Assim que criar a VPS, acesse-a como root e atualize todos os pacotes imediatamente:

```bash
ssh root@<IP_DA_VPS>
```

```bash
# Atualiza a lista de pacotes e instala as atualizações
apt update && apt upgrade -y

# Remove pacotes desnecessários
apt autoremove -y && apt autoclean -y

# Instala utilitários essenciais
apt install -y \
  curl \
  wget \
  git \
  vim \
  htop \
  unzip \
  gnupg \
  ca-certificates \
  lsb-release \
  software-properties-common \
  ufw \
  fail2ban \
  net-tools
```

> **Por quê atualizar primeiro?** Sistemas recém-criados frequentemente têm pacotes desatualizados com vulnerabilidades conhecidas. Atualizar antes de qualquer outra configuração reduz a janela de exposição.

---

## 2. Configurar o Fuso Horário

```bash
# Verificar fuso horário atual
timedatectl

# Definir para Brasília (ou o seu fuso)
timedatectl set-timezone America/Sao_Paulo

# Verificar
timedatectl status
```

---

## 3. Criar Usuário Não-Root

Nunca opera um servidor como root no dia a dia. Crie um usuário dedicado:

```bash
# Cria o usuário (substitua 'deploy' pelo nome desejado)
adduser deploy

# Adiciona ao grupo sudo
usermod -aG sudo deploy

# Verifica grupos do usuário
groups deploy
```

> **Por que não usar root?** O usuário root tem permissão para tudo — um comando errado pode destruir o sistema. Um usuário com sudo exige confirmação explícita para operações críticas e cria um log de auditoria.

### Verificar acesso sudo

```bash
su - deploy
sudo whoami
# Deve retornar: root
exit
```

---

## 4. Configuração de Autenticação por Chave SSH

Autenticação por chave SSH é muito mais segura do que senha. Configure-a **antes** de desativar o acesso por senha.

### 4.1 Gerar chave SSH na sua máquina local

Execute na sua **máquina local** (não na VPS):

```bash
# Gera par de chaves ED25519 (algoritmo moderno e seguro)
ssh-keygen -t ed25519 -C "deploy@<SEU_DOMINIO>" -f ~/.ssh/vps_deploy

# Exibe a chave pública para copiar
cat ~/.ssh/vps_deploy.pub
```

### 4.2 Instalar a chave pública na VPS

De volta à VPS, como root:

```bash
# Cria o diretório .ssh para o usuário deploy
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh

# Cola a chave pública (substitua pelo conteúdo da sua chave)
echo "ssh-ed25519 AAAA... deploy@seudominio.com" >> /home/deploy/.ssh/authorized_keys

# Ajusta permissões
chmod 600 /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
```

**Alternativa rápida** (execute na sua máquina local):

```bash
ssh-copy-id -i ~/.ssh/vps_deploy.pub deploy@<IP_DA_VPS>
```

### 4.3 Testar o acesso por chave antes de prosseguir

Abra um **novo terminal** e teste:

```bash
ssh -i ~/.ssh/vps_deploy deploy@<IP_DA_VPS>
```

> **IMPORTANTE:** Confirme que o acesso por chave funciona **antes** de desativar a autenticação por senha. Caso contrário, você pode perder acesso ao servidor.

---

## 5. Configuração do SSH (Hardening)

Edite o arquivo de configuração do SSH:

```bash
sudo vim /etc/ssh/sshd_config
```

Localize e altere (ou adicione) as seguintes linhas:

```ini
# Porta SSH — considere mudar para uma porta não padrão (opcional, mas útil)
Port 22

# Desativa login root via SSH
PermitRootLogin no

# Desativa autenticação por senha (somente chaves)
PasswordAuthentication no

# Desativa PAM para autenticação por senha
UsePAM no

# Desativa autenticação por teclado-interativa
ChallengeResponseAuthentication no

# Desativa encaminhamento X11 (desnecessário em servidores)
X11Forwarding no

# Limita tentativas de autenticação por conexão
MaxAuthTries 3

# Fecha conexões ociosas após 10 minutos
ClientAliveInterval 300
ClientAliveCountMax 2

# Somente IPv4 (opcional — remova se precisar de IPv6)
AddressFamily inet

# Usuários permitidos (whitelist explícita)
AllowUsers deploy
```

Salve o arquivo e teste a configuração antes de reiniciar:

```bash
# Testa a sintaxe da configuração
sudo sshd -t

# Se não houver erros, reinicia o serviço
sudo systemctl restart sshd

# Verifica status
sudo systemctl status sshd
```

> **Mudança de porta SSH (opcional mas recomendado):** Alterar a porta padrão 22 para algo como 2222 ou 4422 reduz significativamente o volume de tentativas de brute-force automatizadas nos logs. Se mudar, lembre-se de liberar a nova porta no firewall antes de reiniciar o SSH.

---

## 6. Configuração do Firewall (UFW)

O UFW (Uncomplicated Firewall) é a ferramenta padrão para gerenciar iptables no Ubuntu de forma simples.

```bash
# Verifica status atual
sudo ufw status

# Define política padrão: bloquear tudo que entra, permitir tudo que sai
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Libera SSH (IMPORTANTE: faça isso antes de ativar o UFW)
sudo ufw allow 22/tcp
# Se mudou a porta: sudo ufw allow <SUA_PORTA>/tcp

# Libera HTTP e HTTPS (para o reverse proxy)
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Ativa o firewall
sudo ufw enable

# Verifica regras ativas
sudo ufw status verbose
```

Saída esperada:

```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
443/tcp                    ALLOW IN    Anywhere
```

> **Importante:** Nunca ative o UFW sem liberar a porta SSH primeiro. Caso contrário, você será bloqueado do próprio servidor.

### Comandos úteis do UFW

```bash
# Listar regras com números
sudo ufw status numbered

# Remover uma regra específica
sudo ufw delete <NUMERO>

# Bloquear um IP específico
sudo ufw deny from <IP_MALICIOSO>

# Permitir acesso de um IP específico (útil para ferramentas internas)
sudo ufw allow from <IP_CONFIAVEL> to any port 22
```

---

## 7. Configurar o Hostname

```bash
# Verifica hostname atual
hostnamectl

# Define o hostname da VPS
sudo hostnamectl set-hostname vps-producao

# Adiciona ao /etc/hosts para resolução local
echo "127.0.1.1 vps-producao" | sudo tee -a /etc/hosts
```

---

## 8. Configuração do SSH na Máquina Local (Opcional)

Para facilitar o acesso diário, configure um alias no seu arquivo `~/.ssh/config` local:

```ini
Host vps
  HostName <IP_DA_VPS>
  User deploy
  IdentityFile ~/.ssh/vps_deploy
  Port 22
  ServerAliveInterval 60
```

Agora você pode acessar com apenas:

```bash
ssh vps
```

---

## 9. Verificação Final

Antes de prosseguir, confirme:

- [ ] Sistema atualizado
- [ ] Usuário `deploy` criado e com sudo
- [ ] Acesso por chave SSH funcionando
- [ ] Login root desabilitado no SSH
- [ ] Autenticação por senha desabilitada
- [ ] Firewall UFW ativo com regras corretas
- [ ] Timezone configurado

---

## Próximo Passo

Continue com a **[Segurança do Servidor](./security.md)** para instalar o Fail2ban e configurar atualizações automáticas de segurança.
