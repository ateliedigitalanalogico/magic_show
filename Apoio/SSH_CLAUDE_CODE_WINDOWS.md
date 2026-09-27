# SSH + Claude Code no Windows 11 — controlar o TD remotamente

Objetivo: deixar o PC Windows (que vai rodar o TouchDesigner com os 2 Kinects) acessível
por SSH, com o **Claude Code** instalado nele, pra dar pra conectar de fora (Mac, outra
máquina) e controlar o TD através do MCP `twozero` que já roda dentro do próprio TD.
Pesquisado/escrito em 2026-09-27.

## 1. Ativar o OpenSSH Server no Windows 11

Abrir **PowerShell como Administrador** e rodar:

```powershell
# instala o componente do servidor SSH (só precisa uma vez)
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# inicia o serviço agora
Start-Service sshd

# deixa o serviço subindo sozinho toda vez que o Windows ligar
Set-Service -Name sshd -StartupType 'Automatic'

# confirma que a regra de firewall pra porta 22 existe (o instalador já cria sozinho)
Get-NetFirewallRule -Name 'OpenSSH-Server-In-TCP'
```

Se a regra de firewall não existir por algum motivo, criar manualmente:

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True `
  -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

**Achar o IP local do Windows** (pra conectar depois): `ipconfig` → anotar o
"Endereço IPv4" da rede que o PC vai usar (Wi-Fi ou cabo, o que for usar na sala).

## 2. Gerar a chave SSH (na máquina de onde você vai conectar — o Mac)

No Mac (ou em qualquer máquina cliente), se ainda não tiver uma chave:

```bash
ssh-keygen -t ed25519 -C "controle-td-windows"
```

Aceitar o caminho padrão (`~/.ssh/id_ed25519`). Isso gera um par de chaves: a privada
(fica só no Mac, nunca compartilhar) e a pública (`id_ed25519.pub`, essa sim vai pro
Windows).

## 3. Copiar a chave pública pro Windows

**Atenção**: se a conta de usuário do Windows for **Administrador** (o mais comum), o
OpenSSH do Windows usa um arquivo DIFERENTE do padrão Linux/Mac — não é
`~/.ssh/authorized_keys`, é `C:\ProgramData\ssh\administrators_authorized_keys`
(compartilhado entre todos os admins da máquina). Usar o caminho errado faz cair sempre
em autenticação por senha, sem erro claro.

**Se a conta do Windows for Administrador** (rodar no Mac, troca `USUARIO` e `IP`):

```bash
type_content=$(cat ~/.ssh/id_ed25519.pub)
ssh USUARIO@IP "powershell -Command \"'$type_content' | Out-File -Append -Encoding ascii C:\ProgramData\ssh\administrators_authorized_keys\""
```

Depois, **no Windows** (PowerShell como Admin), ajustar a permissão do arquivo (o
OpenSSH recusa a chave se a permissão estiver aberta demais):

```powershell
icacls.exe "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r `
  /grant "Administrators:F" /grant "SYSTEM:F"
```

**Se a conta do Windows NÃO for Administrador** (conta padrão), aí sim é o caminho
normal, direto do Mac:

```bash
cat ~/.ssh/id_ed25519.pub | ssh USUARIO@IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

## 4. Testar a conexão

Do Mac:

```bash
ssh USUARIO@IP
```

Se conectar SEM pedir senha, a chave está funcionando. Se pedir senha, revisar o passo
3 (caminho errado do arquivo ou permissão da ACL) antes de continuar.

**Dica de conveniência** — criar um atalho no `~/.ssh/config` do Mac pra não digitar o
IP toda vez:

```
Host td-windows
    HostName IP_DO_WINDOWS
    User USUARIO
```

Depois disso, só `ssh td-windows`.

## 5. Instalar o Claude Code no Windows

Dentro da sessão SSH já conectada no Windows (ou direto no PowerShell do próprio
Windows), rodar:

```powershell
irm https://claude.ai/install.ps1 | iex
```

Isso instala o binário nativo (não precisa Node.js pra esse caminho), adiciona ao PATH
e não precisa rodar como Admin. Fechar e abrir o terminal de novo (ou a sessão SSH) pra
o PATH atualizar, depois confirmar:

```powershell
claude --version
```

Se der erro de "não pode rodar scripts" (política de execução do PowerShell), rodar
antes: `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`.

**Alternativa** (se preferir o caminho via npm): instalar Node.js 22+ primeiro
(nodejs.org, marcar "Add to PATH" no instalador) + Git for Windows (o Claude Code usa
o Git Bash internamente), depois `npm install -g @anthropic-ai/claude-code` — **nunca**
com `sudo`/admin elevado, isso é a causa mais comum de erro de permissão.

## 6. Autenticar

Primeira vez rodando `claude` no Windows, ele vai pedir login
(`claude auth login` ou fluxo interativo na primeira execução) — mesma conta/plano
usado no Mac.

## 7. Fluxo completo pra controlar o TD remotamente

1. TouchDesigner aberto no Windows, com o `twozero.tox` (pasta `Tox/` deste projeto)
   carregado no `.toe` — é ele que sobe o servidor MCP local que o Claude Code
   conversa.
2. Do Mac (ou de onde for), `ssh td-windows`.
3. Dentro da sessão SSH, `cd` até a pasta do projeto e rodar `claude`.
4. A partir daí, o Claude Code rodando NO Windows enxerga o MCP `twozero_td` local
   (mesma máquina do TD) e controla o TD normalmente — a única diferença é que você
   está "entrando" nessa sessão via SSH em vez de sentar na frente do PC.

## 8. Notas de segurança

- **Não expor a porta 22 direto pra internet** (sem VPN/port-forward público) — deixar
  só acessível na rede local da instalação, a não ser que se saiba o que está fazendo
  com hardening adicional (fail2ban equivalente, mudar a porta, etc.).
- Preferir **só autenticação por chave** — depois que a chave estiver funcionando,
  desativar login por senha no SSH editando `C:\ProgramData\ssh\sshd_config`
  (`PasswordAuthentication no`) e reiniciando o serviço (`Restart-Service sshd`).
- Se for mesmo precisar acessar de fora da rede local (ex: controlar remoto no dia do
  show de outro lugar), preferir uma VPN (Tailscale é o caminho mais simples hoje,
  zero configuração de roteador) em vez de abrir a porta 22 no roteador.

## Fontes

- [Get started with OpenSSH Server for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)
- [Key-Based Authentication in OpenSSH for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)
- [openssh server configuration | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_server_configuration)
- [Enable OpenSSH Server in Windows 11: Complete Setup Guide](https://www.itechguides.com/how-to-enable-openssh-server-in-windows-11/)
- [Setting Permissions for SSH Key — Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5758779/setting-permissions-for-ssh-key)
