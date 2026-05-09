# SimpleX Chat Self-Hosted Server — Raspberry Pi 4

**Guia de deployment para comunicações soberanas em mesh**

**Versão:** 2.0 (revista) · **Data:** 2026-05-06 · **Licença:** CC BY-SA 4.0

> 📝 **Nota sobre esta revisão:** versão 2.0 com pass cirúrgica técnica aplicada. Mudanças principais:
>
> - **Secção SSL/TLS reescrita** — o documento original descrevia comportamento Let's Encrypt que **não corresponde** ao modelo de TLS do SimpleX (TOFU/fingerprint, não CA-chain). Ver secção 6.
> - **Docker Compose v2** — substituído `docker-compose` (v1, deprecated) pelo plugin `docker compose` (v2, atual).
> - **Pi OS Bookworm (12+)** — adicionada nota sobre `dhcpcd` ↔ NetworkManager.
> - **Marcadores ⚠️ e ✓** aplicados em todo o documento para sinalizar incertezas e reconstruções confiantes.
> - **Tor hidden service** — secção dedicada adicionada (era recomendação importante em "advanced").
> - **CGNAT** — detecção mais precisa.
>
> ⚠️ Antes de seguir este guia em produção, **valida a configuração específica do SMP server contra a documentação oficial atual** em <https://simplex.chat/docs/server.html> — algumas chaves de configuração e env vars podem ter evoluído entre versões.

---

## Índice

1. [Visão geral](#1-visão-geral)
2. [Requisitos](#2-requisitos)
3. [Domínio / DynDNS](#3-domínio--dyndns)
4. [Networking setup](#4-networking-setup)
5. [Preparação do Raspberry Pi](#5-preparação-do-raspberry-pi)
6. [TLS no SimpleX — modelo correto](#6-tls-no-simplex--modelo-correto) ← **lê isto**
7. [Deployment do SMP server](#7-deployment-do-smp-server)
8. [Tor hidden service (opcional, recomendado)](#8-tor-hidden-service-opcional-recomendado)
9. [Configuração de clientes](#9-configuração-de-clientes)
10. [Testing](#10-testing)
11. [Monitoring & manutenção](#11-monitoring--manutenção)
12. [Off-grid optimization](#12-off-grid-optimization)
13. [Mesh integration (concept)](#13-mesh-integration-concept)
14. [Troubleshooting](#14-troubleshooting)
15. [Recursos](#15-recursos)
16. [Checklist final](#16-checklist-final)
17. [Apêndices](#17-apêndices)

---

## 1. Visão geral

Este guia configura um **SimpleX Chat SMP server** self-hosted num Raspberry Pi 4, dando controlo total sobre a tua infraestrutura de messaging privado.

**O que vais ter no final:**

- ✅ SimpleX SMP server a correr no teu Raspberry Pi
- ✅ Address `smp://FINGERPRINT@dominio:5223` para configurar nos clientes
- ✅ Persistência via Docker volumes
- ✅ (Opcional) Tor hidden service para acesso `.onion`
- ✅ Zero dependência dos servidores SimpleX Ltd
- ✅ Mensagens relay sob teu controlo total

**Tempo estimado:** 2–3 horas (primeira vez)

**Skill level:** Intermediário (CLI confortável, networking básico)

---

## 2. Requisitos

### 2.1. Hardware

- **Raspberry Pi 4** (2 GB RAM mínimo, **4 GB+ recomendado**)
- **MicroSD card** (16 GB+, Class 10 ou A1/A2)
- **Power supply** (PSU oficial RPi4 — 5 V/3 A USB-C — ou equivalente certificado)
- **Cabo Ethernet** (WiFi funciona mas Ethernet é claramente preferível para servidor)
- **Router com port forwarding** (acesso admin necessário)

### 2.2. Software (vamos instalar)

- Raspberry Pi OS Lite (64-bit), preferencialmente Bookworm 12 ou superior
- Docker Engine + Docker Compose v2 (plugin)
- Imagem oficial `simplexchat/smp-server`
- (Opcional) `tor` para hidden service

### 2.3. Network

- **Opção A:** Domínio próprio (ex.: `smp.teudominio.pt`)
- **Opção B:** DynDNS gratuito (ex.: Duck DNS, No-IP)
- **Opção C:** Tor hidden service apenas (sem necessidade de IP público)
- Acesso a port forwarding no router
- IP público ou workaround para CGNAT

### 2.4. Conhecimento prévio

- CLI básico (ssh, nano/vim, comandos docker)
- Networking básico (IP, port forwarding, DNS)
- Gestão de pacotes Linux (apt)

---

## 3. Domínio / DynDNS

> ⚠️ **Atenção antes de escolher:** o SimpleX **não depende de um nome DNS** para funcionar — os clientes ligam-se por `IP:port` ou `dominio:port` indistintamente, e a verificação de identidade do servidor é feita pelo **fingerprint público** (não pelo CN/SAN do certificado). Domínio é conveniência, não requisito.
>
> Se vais correr **só Tor hidden service** ([secção 8](#8-tor-hidden-service-opcional-recomendado)), podes saltar esta secção inteira.

### 3.1. Opção A — Domínio próprio

**Vantagens:**

- ✅ Profissional (`smp.mesh.pt`)
- ✅ Memorável, brandable
- ✅ Controlo total dos DNS records
- ✅ Reutilizável para outros serviços

**Setup:**

1. Registar domínio (registrar à escolha)
2. Criar A record: `smp.teudominio.pt` → IP público
3. Aguardar propagação DNS (5 min – 24 h)
4. Continuar para deployment

**Custo:** ~€10–15/ano (.pt/.com)

### 3.2. Opção B — DuckDNS (gratuito)

**Vantagens:**

- ✅ 100% gratuito
- ✅ Setup em 5 minutos
- ✅ Auto-update do IP (dynamic IP friendly)

**Setup:**

1. **Regista conta:** vai a <https://www.duckdns.org>, login com GitHub/Reddit/Google.

2. **Cria subdomínio:**
   - Nome: `teu-mesh` (exemplo)
   - Fica: `teu-mesh.duckdns.org`
   - Aponta para o IP público (auto-detect)

3. **Token:** copia o token da página principal (vais precisar para o auto-update).

4. **Script de auto-update no RPi:**

   ```bash
   sudo apt install -y curl

   mkdir -p ~/duckdns
   cd ~/duckdns

   cat > duck.sh << 'EOF'
   #!/bin/bash
   echo url="https://www.duckdns.org/update?domains=teu-mesh&token=SEU_TOKEN_AQUI&ip=" \
     | curl -k -o ~/duckdns/duck.log -K -
   EOF

   chmod 700 duck.sh
   ```

   > ✓ Reconstruído: script DuckDNS com shebang e heredoc (versão segura, recomendada pela documentação oficial DuckDNS).

5. **Crontab (update a cada 5 min):**

   ```bash
   crontab -e
   ```

   Adiciona a linha:

   ```
   */5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
   ```

6. **Testa:**

   ```bash
   ./duck.sh
   cat duck.log
   # Deve mostrar "OK"
   ```

### 3.3. Alternativas DynDNS gratuitas

- **No-IP** (<https://www.noip.com>) — 3 hostnames grátis, mas exige confirmação manual a cada 30 dias
- **Afraid.org / FreeDNS** (<https://freedns.afraid.org>) — múltiplos domínios
- **Dynu** (<https://www.dynu.com>) — 4 hostnames grátis

---

## 4. Networking setup

### 4.1. Descobrir o IP público

```bash
curl -4 ifconfig.me
echo
# alternativa
curl -4 icanhazip.com
```

> ✓ Reconstruído: `curl -4` força IPv4, evitando ambiguidade caso o ISP forneça IPv6 dual-stack.

**Anota este IP** — é o que usas no DNS A record.

### 4.2. Detectar CGNAT

Se o "IP público" detectado acima cai num destes intervalos, estás atrás de **CGNAT (Carrier-Grade NAT)**:

| Intervalo | Designação |
|---|---|
| `100.64.0.0/10` (100.64.0.0 – 100.127.255.255) | **CGNAT específico (RFC 6598)** — sinal claro |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` | Privados RFC 1918 — se aparecem como "público", também é CGNAT |

**Se estás em CGNAT:**

- Port forwarding **não vai funcionar**.
- Soluções:
  - Contacta ISP para "IP público dedicado" (em PT, alguns ISP cobram extra; outros recusam para clientes residenciais)
  - **Tor hidden service** (ver secção 8) — recomendado, contorna CGNAT completamente
  - VPN com IP público fixo (Tailscale Funnel, Cloudflare Tunnel, ZeroTier — fora do scope deste guia)

### 4.3. Port forwarding (router)

**Precisas abrir port 5223 (TCP, SMP protocol):**

> ⚠️ Os passos exatos variam por marca de router — apêndice [B](#apêndice-b--port-forwarding-routers-comuns) cobre os mais comuns em Portugal.

**Passos genéricos:**

1. Login no router (browser para `http://192.168.1.1` ou `http://192.168.0.1`).
2. Encontra a secção: "NAT", "Port Forwarding" ou "Virtual Servers".
3. Cria a regra:

   ```
   Service Name:   SimpleX SMP
   External Port:  5223
   Internal Port:  5223
   Internal IP:    192.168.1.XXX  (IP do Pi)
   Protocol:       TCP
   Enable:         Yes
   ```

4. Save & Apply.

5. **Garante IP fixo para o Pi:**
   - DHCP Reservation no router (recomendado, ver [apêndice C](#apêndice-c--ip-fixo-no-raspberry-pi))
   - Ou IP estático no próprio Pi

**Testar port forwarding (após deployment):**

```bash
# De máquina EXTERNA (não na tua rede):
nc -zv teu-mesh.duckdns.org 5223
```

### 4.4. Firewall no Raspberry Pi

```bash
# Instala ufw se não tiveres
sudo apt install -y ufw

# Permite SSH (não te bloqueies fora!)
sudo ufw allow 22/tcp

# Permite SMP
sudo ufw allow 5223/tcp

# Activa
sudo ufw enable

# Verifica
sudo ufw status verbose
```

> ✓ Reconstruído: comandos `ufw` standard, sintaxe correta para Debian/Ubuntu/Raspberry Pi OS.

> ⚠️ **Nota sobre port 80:** o documento original menciona abrir port 80 para "ACME challenge" do Let's Encrypt. **Não é necessário** com a configuração TLS standard do SMP server (ver [secção 6](#6-tls-no-simplex--modelo-correto)). Não abras port 80 a menos que tenhas razão específica.

---

## 5. Preparação do Raspberry Pi

### 5.1. Sistema operativo

**Raspberry Pi OS Lite (64-bit) — recomendado:**

- Headless (sem GUI), baixo overhead
- 64-bit aproveita a RAM do Pi 4
- Bookworm (12) ou superior

**Flash do SD card:**

1. Descarrega Raspberry Pi Imager: <https://www.raspberrypi.com/software/>
2. Escolhe: "Raspberry Pi OS Lite (64-bit)"
3. Configura settings (OS Customization):
   - Hostname: `simplex-server`
   - Enable SSH (com password ou chave pública)
   - Username/password: a tua escolha
   - WiFi se não usares Ethernet
4. Escreve para o SD card.

**Boot do Pi:**

- Insere o SD, liga energia.
- SSH:

  ```bash
  ssh teu-utilizador@simplex-server.local
  ```

  > ⚠️ O sufixo `.local` (mDNS) **funciona em Linux/macOS por defeito** e em Windows com Bonjour instalado (vem com iTunes ou pode instalar-se à parte). Em Windows sem Bonjour, usa o IP direto: `ssh teu-utilizador@192.168.1.XXX`.

### 5.2. Updates e essentials

```bash
# Atualizar sistema
sudo apt update && sudo apt upgrade -y

# Pacotes essenciais
sudo apt install -y \
  git \
  curl \
  wget \
  vim \
  htop \
  net-tools \
  ufw \
  ca-certificates

# Reboot se houve update do kernel
sudo reboot
```

### 5.3. Docker install

> ⚠️ **Mudança importante vs documento original:** este guia usa **Docker Compose v2** (plugin oficial, atual), não `docker-compose` v1 (Python, deprecated em Junho 2023). A v2 invoca-se como `docker compose` (com espaço, sem hífen).

```bash
# Script oficial Docker (instala Engine + Compose v2 plugin)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Adiciona o teu utilizador ao grupo docker (sem precisar de sudo)
sudo usermod -aG docker $USER

# Logout/login para aplicar o group
exit
# SSH novamente
```

> ✓ Reconstruído: `get.docker.com` é o método oficial recomendado pela Docker Inc. para Raspberry Pi e outras plataformas ARM.

**Verifica a instalação:**

```bash
docker --version
# Esperado: Docker version 24+ ou 25+

docker compose version
# Esperado: Docker Compose version v2.x.x

docker ps
# Lista vazia (sem containers) — confirma que daemon corre
```

> ⚠️ Se vires `docker-compose: command not found` mas `docker compose version` funcionar, está correto. O comando legacy `docker-compose` (v1) **não** é instalado por este script.

---

## 6. TLS no SimpleX — modelo correto

> ⚠️ **CRÍTICO — esta secção corrige uma incorreção significativa do documento original.**

### 6.1. Como o SimpleX faz TLS (na realidade)

O SimpleX SMP server **não usa Let's Encrypt nem CA chains tradicionais**. Em vez disso:

1. **No primeiro arranque**, o container gera **um par de chaves TLS auto-assinado** e calcula o respetivo **fingerprint SHA-256**.

2. Este fingerprint é incorporado no **server address** que vais distribuir aos clientes:

   ```
   smp://FINGERPRINT@teu-mesh.duckdns.org:5223
   ```

3. **Os clientes SimpleX verificam o fingerprint**, não a CA chain. Modelo TOFU (Trust On First Use) — semelhante ao SSH, não ao HTTPS de browser.

4. **Não há renovação periódica** porque não há CA expiry — o cert dura enquanto manténs as chaves persistidas no volume Docker.

### 6.2. Implicações práticas

- ❌ **Não precisas** instalar `certbot`.
- ❌ **Não precisas** abrir port 80 para ACME challenge.
- ❌ **Não precisas** de domínio com DNS público — funciona com IP direto.
- ❌ **Não substitui** os certs com Let's Encrypt — quebra a verificação de fingerprint dos clientes existentes.
- ✅ **Faz backup do volume `./config/`** — perder as chaves significa perder o fingerprint, e todos os clientes existentes deixam de poder validar a identidade do servidor.

### 6.3. O que fazer com as instruções Let's Encrypt do guia original

Ignora-as. Se já fizeste `sudo apt install certbot`, podes desinstalar (`sudo apt remove certbot`). Se já abriste port 80 no router, podes fechar. Se já obtiveste um certificate, **não o montes no container** — vai partir o modelo de fingerprint.

> ✓ Reconstruído com base na documentação oficial do SimpleX SMP server: <https://simplex.chat/docs/server.html> e código da imagem `simplexchat/smp-server` no GitHub. Confirma a versão atual antes de deploy de produção.

### 6.4. Ainda assim quero CA-signed cert?

Há setups onde queres cert público (por exemplo, atrás de proxy reverso para auditoria). Possível mas **fora do scope deste guia** porque:

- Quebra TOFU
- Requer configuração custom do SMP server
- Adiciona surface de ataque (certbot, renovação, dependência de terceiros)

Para deployment standard, **deixa o SimpleX gerar e gerir o cert dele**.

---

## 7. Deployment do SMP server

### 7.1. Estrutura de projeto

```bash
mkdir -p ~/simplex-server
cd ~/simplex-server
```

### 7.2. Ficheiro `.env` (recomendado)

Em vez de hardcodar password no `compose.yml`, usa um `.env` separado (não vai para git, não aparece em logs):

```bash
nano .env
```

Cola e edita:

```bash
# Domínio ou DynDNS
SIMPLEX_ADDR=teu-mesh.duckdns.org

# Password forte para criação de queues novas
# Gera com: openssl rand -base64 32
SIMPLEX_PASS=COLA_AQUI_PASSWORD_GERADA
```

> ✓ Reconstruído: separação `.env` é best practice Docker e prevenção de leaks em logs/git history.

Permissões:

```bash
chmod 600 .env
```

### 7.3. Ficheiro `compose.yml`

> ⚠️ **Nota de naming:** ficheiro `compose.yml` (sem `docker-` no nome) é a convenção atual da Docker Compose v2. `docker-compose.yml` (legacy) ainda funciona mas a forma curta é preferida.

```bash
nano compose.yml
```

Cola:

```yaml
services:
  smp-server:
    image: simplexchat/smp-server:latest
    container_name: simplex_smp
    restart: unless-stopped

    environment:
      - ADDR=${SIMPLEX_ADDR}
      - PASS=${SIMPLEX_PASS}

    volumes:
      # Persistência: configs, certificados (chaves TLS), logs
      - ./data:/var/opt/simplex
      - ./config:/etc/opt/simplex

    ports:
      - "5223:5223"

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

# XFTP server (file transfers self-hosted) — opcional, descomenta para activar.
# Atenção: se usares port 443, conflita com qualquer outro web service no Pi.
#  xftp-server:
#    image: simplexchat/xftp-server:latest
#    container_name: simplex_xftp
#    restart: unless-stopped
#    environment:
#      - ADDR=${SIMPLEX_ADDR}
#      - QUOTA=1gb
#    volumes:
#      - ./xftp-data:/var/opt/simplex-xftp
#      - ./xftp-config:/etc/opt/simplex-xftp
#    ports:
#      - "5443:443"  # Mapeia 5443 externo → 443 interno para evitar conflitos
```

> ⚠️ **[VERIFICAR contra docs SimpleX atuais]** Os volumes `/var/opt/simplex` e `/etc/opt/simplex` são os paths usados pela imagem oficial à data desta revisão. Confirma na documentação atual antes de assumir como definitivo: <https://simplex.chat/docs/server.html>.

> ⚠️ **[VERIFICAR contra docs SimpleX atuais]** O env var `PASS` controla o "basic auth password" para criação de queues novas. Comportamento exato da imagem (exigência obrigatória, formato, hashing) pode variar — confirmar na documentação.

### 7.4. Deploy

```bash
# Sobe os containers em background
docker compose up -d

# Acompanha logs (Ctrl+C para sair)
docker compose logs -f
```

**Procura nos logs:**

- `Server fingerprint: <hash>` — esta é a fingerprint que vai no address do servidor
- `Listening on TCP port 5223` — confirma que está a aceitar ligações
- `Server address: smp://FINGERPRINT@teu-mesh.duckdns.org:5223` — **copia este address inteiro**

> ⚠️ **[VERIFICAR contra docs SimpleX atuais]** O formato exato dos logs e a forma como a fingerprint é exposta podem variar por versão. Se a fingerprint não aparece nos logs, há geralmente um ficheiro em `./config/` que a contém — `cat config/server_pub_hash.bin | xxd` ou similar (verificar documentação atual).

### 7.5. Verificação

```bash
# Container a correr?
docker ps

# Esperado: simplex_smp em STATE "Up"

# Port aberto?
sudo ss -tlnp | grep 5223
# (alternativa moderna a netstat)
```

> ✓ Reconstruído: `ss` é o substituto moderno de `netstat` em Linux atual; ambos funcionam mas `ss` é preferido em distribuições recentes.

---

## 8. Tor hidden service (opcional, recomendado)

Adicionar um endpoint `.onion` ao teu servidor traz vantagens significativas:

- **Bypassa CGNAT** completamente — funciona sem IP público
- **Bypassa port forwarding** — não precisas mexer no router
- **Anonimato adicional** — ISPs e observadores de rede não veem que tens um servidor
- **Resiliência** — se o IP muda, o `.onion` mantém-se

> ✓ Reconstruído: o suporte nativo a `.onion` em SimpleX é uma das vantagens diferenciadoras vs outras messaging platforms — vale a pena ativar.

### 8.1. Instalar e configurar Tor

```bash
sudo apt install -y tor

sudo nano /etc/tor/torrc
```

Adiciona ao final do ficheiro:

```
HiddenServiceDir /var/lib/tor/simplex/
HiddenServicePort 5223 127.0.0.1:5223
```

Restart Tor:

```bash
sudo systemctl restart tor
sudo systemctl enable tor
```

### 8.2. Obter o endereço `.onion`

```bash
sudo cat /var/lib/tor/simplex/hostname
```

Output exemplo:

```
abc123def456...xyz.onion
```

> ⚠️ **[VERIFICAR contra docs SimpleX atuais]** O SimpleX SMP server suporta dual address (clearnet + onion). O server address completo passa a ter ambos:
>
> ```
> smp://FINGERPRINT@teu-mesh.duckdns.org,abc...xyz.onion:5223
> ```
>
> Sintaxe exata e suporte de versão variam — confirmar na documentação atual antes de assumir como funcional.

### 8.3. Cliente sobre Tor

Os clientes SimpleX (Android/iOS/Desktop) suportam transporte sobre Tor:

- **Android:** instala Orbot, ativa "VPN mode" para a app SimpleX
- **iOS:** Onion Browser ou similar (mais limitado)
- **Desktop:** configura SOCKS proxy para `127.0.0.1:9050` nas Settings → Network

---

## 9. Configuração de clientes

### 9.1. Android

1. Abre **SimpleX Chat**
2. **Settings → Network & servers**
3. **Messaging servers → Your SMP servers**
4. **Add server** → cola o address copiado dos logs:

   ```
   smp://FINGERPRINT@teu-mesh.duckdns.org:5223
   ```

5. **Save**

6. **Test:**
   - Settings → Developer tools → Server tests
   - Run connectivity test
   - Esperado: "Connected"

7. **(Opcional) Desactivar servidores default:**
   - Messaging servers → Preset servers → Disable all

   > ⚠️ **Cuidado:** se desativares todos os preset servers e o teu cair, **perdes capacidade de receber mensagens**. Recomendado: deixa pelo menos 1 preset SimpleX como fallback.

### 9.2. iOS / Desktop

Mesmo processo:

- **iOS:** Settings → Network & servers → Add server
- **Desktop:** download em <https://simplex.chat/downloads> → Settings → Network → Add server

---

## 10. Testing

### 10.1. Test 1 — Conectividade externa

De uma máquina **fora** da tua rede (rede móvel do telemóvel, casa de amigo, VPS):

```bash
nc -zv teu-mesh.duckdns.org 5223
```

Esperado: `Connection to teu-mesh.duckdns.org 5223 port [tcp/*] succeeded!`

### 10.2. Test 2 — Fluxo de mensagens

1. **Device A:** adiciona o teu server.
2. **Device B:** adiciona o teu server.
3. **A:** cria new contact link.
4. **B:** connect via link.
5. Envia mensagem A → B.
6. Verifica logs:

   ```bash
   docker compose logs --tail=100
   ```

   > ⚠️ Os logs do SMP server **não expõem conteúdo de mensagens** (correto — é um relay encrypted). Vais ver atividade de queue creation, conexões, mas não payloads. Isto é o comportamento esperado.

### 10.3. Test 3 — Recursos do sistema

```bash
docker stats simplex_smp --no-stream
```

Esperado em idle: <100 MB RAM, <5% CPU.

> ✓ Reconstruído: `--no-stream` faz o comando devolver uma snapshot e sair (sem `--no-stream`, fica em loop contínuo, útil para monitoring mas não para um teste pontual).

---

## 11. Monitoring & manutenção

### 11.1. Logs

```bash
# Tempo real
docker compose logs -f

# Últimas 100 linhas
docker compose logs --tail=100

# Serviço específico (se tiveres XFTP também)
docker compose logs smp-server
```

### 11.2. Restart / stop / start

```bash
cd ~/simplex-server

docker compose restart    # Restart sem destruir containers
docker compose down       # Stop e remove containers (volumes mantêm-se)
docker compose up -d      # Start em background
```

### 11.3. Update do server

```bash
cd ~/simplex-server

docker compose pull              # Pull nova versão da imagem
docker compose up -d             # Recria container com nova imagem
docker compose logs -f           # Verifica que arranca OK
```

> ⚠️ **Antes de update em produção:** faz backup do `./config/` e `./data/` (secção 11.4). Updates major podem mudar formato e fazer rollback é mais fácil com backup.

### 11.4. Backup

> ⚠️ **Importante:** o documento original sugere `tar` com containers a correr. Isto pode capturar estado inconsistente. **Para backup robusto, para o container primeiro:**

```bash
cd ~/simplex-server

# 1. Stop container (1-2 segundos de downtime)
docker compose stop

# 2. Backup
tar -czf simplex-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/ config/ .env compose.yml

# 3. Restart
docker compose start

# 4. Move backup para local seguro (USB, NAS, encrypted cloud)
# IMPORTANTE: o backup contém .env com password e config/ com chaves privadas TLS
# Encripta antes de mover para cloud:
gpg -c simplex-backup-*.tar.gz   # Pede passphrase, gera .gpg
shred -u simplex-backup-*.tar.gz # Apaga o original sem encriptação
```

> ✓ Reconstruído: workflow standard de backup com encriptação simétrica via GPG. `shred` garante que o ficheiro não-encriptado não fica recuperável no SD card.

### 11.5. Disk space

```bash
# Espaço livre
df -h

# Limpeza Docker (cuidado — apaga TUDO o que não está em uso)
docker system prune       # Remove containers parados, networks, build cache
docker system prune -a    # Acima + imagens não usadas (mais agressivo)
```

> ⚠️ `docker system prune -a` apaga **todas** as imagens que não estão em uso por containers a correr. Se tens outros projetos Docker no Pi, podes perder imagens que vais querer usar depois (terás de fazer pull novamente).

### 11.6. Logs do sistema (não Docker)

```bash
# Tor (se ativaste hidden service)
sudo journalctl -u tor -f

# UFW (firewall)
sudo journalctl -u ufw

# Sistema geral
sudo journalctl -xe
```

---

## 12. Off-grid optimization

### 12.1. Consumo de energia

Raspberry Pi 4 + SimpleX server:

- **Idle:** ~3–4 W
- **Carga média:** ~5–7 W
- **Pico (boot, updates):** ~8–10 W

### 12.2. Solar setup mínimo viável

> ⚠️ Os números abaixo são estimativas conservadoras para Portugal continental, latitudes 37–42°N, com 4–6 h de sol pico equivalente por dia. Ajusta para a tua localização e estação.

| Componente | Spec mínima | Notas |
|---|---|---|
| Painel solar | 30 W (não 20 W) | 20 W é justo demais para inverno PT |
| Bateria LiFePO4 | 12.8 V / 20 Ah (não Li-ion) | LiFePO4 = 4000+ ciclos vs 500 Li-ion |
| Charge controller | MPPT 10 A | PWM serve mas perde 15–20% eficiência |
| Conversor 12 V → 5 V/3 A | DC-DC buck regulator | Vital — não uses USB charger automotivo barato |

> ✓ Reconstruído: especificações standard para sistemas solares 12 V de baixa potência; LiFePO4 vs Li-ion é diferença real e não-negligenciável em ciclos de descarga profunda.

### 12.3. Low-power mode

```bash
# Desativa WiFi se usas Ethernet
sudo rfkill block wifi

# Desativa Bluetooth
sudo rfkill block bluetooth

# Reduz GPU memory (sem GUI, não precisas)
sudo nano /boot/firmware/config.txt
```

> ⚠️ **Mudança importante:** em Raspberry Pi OS Bookworm (12+), o config.txt está em `/boot/firmware/config.txt`, **não** em `/boot/config.txt` como em versões anteriores. O guia original menciona `/boot/config.txt` que falha em Bookworm.

Adiciona:

```
gpu_mem=16
```

Reboot:

```bash
sudo reboot
```

Esta mudança poupa ~50 MB de RAM e reduz consumo marginalmente.

---

## 13. Mesh integration (concept)

> ⚠️ **Esta secção é exploratória, não deployment-ready.** O documento original prometia "Bridge messages entre SimpleX queues ↔ Reticulum packets, Requires custom code". Esta promessa é **enganadora** sem qualificações importantes.

### 13.1. Por que não há "bridge" trivial

SimpleX e Reticulum são **protocolos fundamentalmente diferentes**:

| Aspecto | SimpleX | Reticulum |
|---|---|---|
| Transporte | TCP, server-mediated | Datagram, mesh-routed |
| Identidade | Per-queue ephemeral | Per-destination persistent hash |
| Encryption | E2E via DH duplo + queue-side blinding | E2E via DH com transitivos signed |
| Modelo de servidor | Relay obrigatório | Opcional (transports) |
| Latência típica | 100–500 ms | 1–60 s (em LoRa) |
| Bandwidth típica | Mbps | 1 kbps – 50 kbps |

Não há mapeamento natural entre os dois. Para "ligar":

### 13.2. Abordagem realista — bot de aplicação

```
[utilizador SimpleX] ←TCP→ [SMP server] ←app bot↔ [LXMF client] ←mesh→ [outro utilizador]
```

- **Bot de aplicação** (não bridge de protocolo): software custom que tem identidade SimpleX **e** identidade LXMF/Reticulum, e faz proxy semântico entre conversas.
- O utilizador final precisa de **identidade em ambos os sistemas** ou aceitar que o bot é o intermediário.
- **Limitações:**
  - Latência mesh impede UX "real-time" para utilizador SimpleX.
  - Bandwidth LoRa não suporta volumes típicos de SimpleX.
  - Bot vê plaintext dos dois lados (quebra E2E que ambos os protocolos oferecem isoladamente).

### 13.3. Quando faz sentido

- **Casos de emergência:** Internet caiu na zona, mas mesh local está operacional. Bot fica num nó com link Reticulum upstream e relay SimpleX para utilizadores em mesh.
- **Comunidades híbridas:** alguns membros têm Internet, outros só mesh. Bot serve como entrypoint.
- **Status broadcasts:** mensagens curtas, baixa frequência, não-críticas em latência.

### 13.4. Próximos passos

Se queres explorar isto seriamente:

1. **Não comeces pela bridge.** Faz primeiro SimpleX SMP a correr standalone (este guia) e Reticulum a correr standalone (outro doc).
2. Documenta casos de uso concretos antes de escrever código.
3. Considera arquitetura "store-and-forward" em vez de proxy real-time.
4. Há discussões na comunidade Reticulum sobre LXMF gateways — vale a pena fazer pesquisa antes de duplicar esforço.

---

## 14. Troubleshooting

### 14.1. Port 5223 não alcançável externamente

**Verificar:**

1. UFW no Pi: `sudo ufw status` — port 5223/tcp deve aparecer
2. Container: `docker ps` — `simplex_smp` deve estar `Up`
3. Listening: `sudo ss -tlnp | grep 5223` — deve mostrar processo
4. Port forwarding no router: regra ativa, IP interno correto
5. CGNAT? (ver secção 4.2) — se sim, port forwarding **não funciona** independentemente da configuração

**Fix:**

```bash
sudo ufw allow 5223/tcp
docker compose restart
# Re-validar regra de port forwarding no router
```

### 14.2. Container em crash loop

```bash
docker compose logs --tail=50 smp-server
```

**Causas comuns:**

- `ADDR` inválido (domínio não resolve)
- Port 5223 já em uso por outro processo no Pi
- Permissões nos volumes (`config/` ou `data/`)

**Fix:**

```bash
# Validar resolução DNS
nslookup teu-mesh.duckdns.org

# Validar port livre
sudo ss -tlnp | grep 5223

# Permissões
sudo chown -R 1000:1000 data/ config/
docker compose up -d
```

> ⚠️ **[VERIFICAR contra docs SimpleX atuais]** O UID 1000:1000 é assumido com base em convenção comum para imagens Linux user-mode — confirma o UID real usado pela imagem `simplexchat/smp-server` (`docker inspect simplexchat/smp-server:latest | grep -i user`).

### 14.3. SSL / certificate error nos clientes

> Antes de fazer nada: lê a [secção 6](#6-tls-no-simplex--modelo-correto). Erros SSL frequentemente são esperados se tentaste interferir com o cert auto-gerado.

**Causas legítimas:**

- O fingerprint no address que partilhaste não corresponde ao cert atual do servidor (substituíste o cert sem atualizar o address).
- O cliente está a usar uma versão antiga do SimpleX que não suporta o formato atual de address.

**Fix:**

- Se substituíste cert: re-extrai o fingerprint atual e re-distribui o address aos utilizadores.
- Confirma versão da app SimpleX nos clientes.

### 14.4. DuckDNS não atualiza

```bash
# Testa manualmente
~/duckdns/duck.sh
cat ~/duckdns/duck.log
```

Output deve ser `OK`. Se for `KO` ou erro:

- Token errado no script
- Domínio não corresponde ao que registaste em duckdns.org
- IP detectado pelo curl é privado (CGNAT)

### 14.5. Pi reboota sozinho / freezes

Geralmente power supply insuficiente. Sintomas:

- `dmesg | grep -i voltage` mostra "Under-voltage detected"
- LED do Pi a piscar amarelo

**Fix:** PSU oficial 5V/3A USB-C. Cabos baratos têm queda de tensão significativa.

---

## 15. Recursos

### 15.1. Documentação oficial

- **SimpleX Chat:** <https://simplex.chat>
- **SMP Server docs:** <https://simplex.chat/docs/server.html>
- **XFTP Server docs:** <https://simplex.chat/docs/xftp-server.html>
- **GitHub:** <https://github.com/simplex-chat/simplex-chat>
- **Whitepaper SimpleX:** <https://github.com/simplex-chat/simplex-chat/blob/stable/docs/protocol/simplex-messaging.md>

### 15.2. Comunidade

- **Reddit:** r/SimpleXChat
- **Matrix:** `#simplex:matrix.org`
- **In-app:** SimpleX group

### 15.3. Ferramentas relacionadas

- **DuckDNS:** <https://www.duckdns.org>
- **Docker:** <https://docs.docker.com>
- **Tor Project:** <https://www.torproject.org>
- **Reticulum (mesh):** <https://reticulum.network>

---

## 16. Checklist final

### 16.1. Pre-deployment

- [ ] Raspberry Pi 4 com OS atualizado (Bookworm 12+)
- [ ] Docker Engine + Docker Compose v2 instalados
- [ ] Domínio/DynDNS configurado **OU** Tor hidden service planeado
- [ ] Port forwarding 5223/TCP configurado (se não for Tor-only)
- [ ] UFW rules: 22/tcp + 5223/tcp + (opcional) 9050/tcp para Tor
- [ ] DHCP reservation ou IP estático para o Pi
- [ ] CGNAT verificado (se sim, planear Tor ou alternativa)

### 16.2. Deployment

- [ ] `compose.yml` criado, ADDR correto
- [ ] `.env` com PASS forte (gerada com `openssl rand`)
- [ ] `chmod 600 .env`
- [ ] `docker compose up -d` executado sem erros
- [ ] Server fingerprint extraída dos logs
- [ ] Server address completo guardado em local seguro
- [ ] (Opcional) Tor hidden service ativo, hostname `.onion` extraído

### 16.3. Configuração de clientes

- [ ] App SimpleX instalada (Android/iOS/Desktop)
- [ ] Server address adicionado
- [ ] In-app connectivity test passa
- [ ] Test message enviado e recebido entre dois devices

### 16.4. Manutenção

- [ ] Backup do `./config/`, `./data/`, `.env`, `compose.yml` agendado (semanal)
- [ ] Backup encriptado antes de sair do Pi (GPG)
- [ ] Update schedule (mensal: `docker compose pull` + `up -d`)
- [ ] Monitoring de logs (verificação ocasional)
- [ ] DuckDNS auto-update validado (se aplicável)

---

## 17. Apêndices

### Apêndice A — Setup detalhado DuckDNS

Ver [secção 3.2](#32-opção-b--duckdns-gratuito) — já está completo no corpo principal.

### Apêndice B — Port forwarding (routers comuns)

> ⚠️ Estes passos são genéricos por marca. Modelos específicos podem ter UIs diferentes — confirma no manual do teu router.

**TP-Link:**

1. Advanced → NAT Forwarding → Virtual Servers
2. Add → Service Port: 5223, Internal Port: 5223, IP: <IP do Pi>, Protocol: TCP

**Vodafone (Portugal):**

1. Avançado → Firewall → Port Forwarding
2. Nova regra → Porta externa: 5223, Interna: 5223, IP: <Pi>, Protocolo: TCP

**MEO (Portugal):**

1. Firewall → Port Forwarding
2. Add → Aplicação: SimpleX, Porta: 5223, IP: <Pi>

**NOS (Portugal):**

1. Configuração avançada → NAT → Port Forwarding
2. Criar regra → Serviço: SimpleX, Porta: 5223, Dispositivo: <Pi>

### Apêndice C — IP fixo no Raspberry Pi

> ⚠️ **CRÍTICO — depende da versão do Pi OS:**

**Em Pi OS Bullseye (11) e anteriores:** usavam `dhcpcd`, configurável em `/etc/dhcpcd.conf`.

**Em Pi OS Bookworm (12) e posteriores:** **`dhcpcd` foi substituído por NetworkManager.** Editar `/etc/dhcpcd.conf` **não tem efeito**.

#### Método 1: DHCP Reservation no router (recomendado em qualquer versão)

- Abre admin do router → DHCP → Reservation
- Adiciona MAC do Pi → IP fixo (ex.: 192.168.1.100)
- Reboot Pi para aplicar

Vantagem: configuração centralizada, fácil de mudar, agnóstica do OS do Pi.

#### Método 2 (Bookworm 12+) — NetworkManager via `nmtui`

```bash
sudo nmtui
```

Interface texto:

1. Edit a connection
2. Selecionar a interface (eth0 ou wlan0)
3. IPv4 CONFIGURATION → Manual
4. Definir IP, gateway, DNS
5. Save → Quit
6. Reiniciar network: `sudo systemctl restart NetworkManager`

#### Método 3 (Bullseye 11 e anteriores) — `dhcpcd`

```bash
sudo nano /etc/dhcpcd.conf
```

Adiciona ao final:

```
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1 8.8.8.8
```

Reboot:

```bash
sudo reboot
```

> ✓ Reconstruído: separação clara entre versões evita o erro silencioso comum de editar `/etc/dhcpcd.conf` em Bookworm e não funcionar.

---

## 18. Filosofia

```
Plataformas centralizadas:
  Confias nelas
  Controlam os teus dados
  Podem censurar
  Podem desligar

Self-hosted:
  Tu controlas tudo
  Hardware teu, regras tuas
  Resistente a censura
  Comunicação soberana

Mesh integration:
  Off-grid capable
  Infraestrutura resiliente
  Comunidade própria
  Valores cypherpunk
```

**Stack soberana completa:**

```
Internet layer    →  SimpleX self-hosted (este guia)
Mesh layer        →  Reticulum / NomadNet / LXMF
Physical layer    →  Meshtastic LoRa
Power             →  Solar off-grid

= COMPLETE SOVEREIGN COMMS STACK
```

---

☥ walk quietly, ₿ut keep the signal alive ⚡

*Self-sovereign messaging. Mesh-ready. Off-grid capable.*

*Du_arte ☥ rawmesh — Beira Baixa, Portugal*
