# SimpleX SMP Server RPi4 — Deployment Checklist

**Versão:** 1.0
**Para usar com:** `simplex-selfhosted-rpi4-guide.md` v2.0 (1141 linhas)
**Como usar:** segue na ordem. Anota valores nos espaços `[______]`. Tira screenshots dos pontos marcados 📸. Corre comandos exatamente como estão (copy-paste).

> 💡 **Antes de começar:** abre 2 terminais SSH para o Pi. Um para comandos. Outro para `docker compose logs -f` em background.

---

## Section 0 — Pre-deployment (15 min)

### 0.1. Identificação do hardware

- [ ] **Modelo do Pi:** `[______]` (esperado: Raspberry Pi 4)
- [ ] **RAM:** `[______]` GB (mínimo 2 GB, recomendado 4 GB+)
- [ ] **SD card:** `[______]` GB, classe `[______]`
- [ ] **PSU:** oficial 5 V/3 A? `[YES / NO]`
- [ ] **Conexão:** Ethernet ou WiFi? `[______]`

### 0.2. Comandos de inventário inicial

Corre estes e cola o output no validation report (secção `Hardware fingerprint`):

```bash
# Modelo do Pi
cat /proc/device-tree/model
echo

# OS
cat /etc/os-release

# Kernel
uname -a

# Memória
free -h

# Storage
df -h

# Rede
ip -4 addr show
ip route
```

- [ ] Output capturado e colado no report

### 0.3. Detecção de versão Pi OS (CRÍTICO para apêndice C)

```bash
grep VERSION_ID /etc/os-release
```

- [ ] Resultado: `[______]`
  - Se `VERSION_ID="12"` ou superior → Bookworm (NetworkManager, `/boot/firmware/config.txt`)
  - Se `VERSION_ID="11"` ou inferior → Bullseye (`dhcpcd`, `/boot/config.txt`)

### 0.4. CGNAT detection

```bash
# IP público que o ISP te atribuiu
curl -4 ifconfig.me
echo
```

- [ ] **IP público observado:** `[______]`
- [ ] CGNAT? Verifica os intervalos:
  - `100.64.0.0` – `100.127.255.255` → ⚠️ CGNAT específico
  - `10.x.x.x`, `172.16.x.x – 172.31.x.x`, `192.168.x.x` → ⚠️ Privado (também CGNAT se aparece como "público")
  - Outro → ✅ IP público real
- [ ] **Decisão:** se CGNAT, vou usar `[Tor hidden service / VPN tunnel / contactar ISP]`

### 0.5. Router access

- [ ] Login no router funciona (URL: `[______]`)
- [ ] Encontrei a secção de port forwarding
- [ ] Marca/modelo do router: `[______]` (para validation report)

---

## Section 1 — Read-through (5 min)

- [ ] Li a secção 1 (Visão geral) do guide
- [ ] Li a secção 6 (TLS no SimpleX — modelo correto) — **importante, é onde o guide difere de outros tutoriais**
- [ ] Tenho noção das 3 opções de network (domínio / DynDNS / Tor-only)

---

## Section 2 — Requisitos (validação)

- [ ] Hardware OK (secção 0.1 do checklist)
- [ ] Network plan decidido: Opção `[A / B / C]`
- [ ] Skill check: confortável com SSH, nano, docker basics? `[YES / NO]`

---

## Section 3 — DuckDNS / Domínio

> Se escolheste **Opção C (Tor-only)**, salta para secção 5.

### 3.1. Conta DuckDNS

- [ ] Conta criada em duckdns.org
- [ ] Subdomain escolhido: `[______]` (ex: `meu-mesh`)
- [ ] Domain completo: `[______].duckdns.org`
- [ ] Token copiado para sítio seguro
- [ ] IP detectado pelo DuckDNS bate com o de 0.4? `[YES / NO]`

### 3.2. Script auto-update no Pi

Comandos exatos (copy-paste, substitui `SEU_TOKEN_AQUI` e `meu-mesh`):

```bash
sudo apt install -y curl
mkdir -p ~/duckdns
cd ~/duckdns

cat > duck.sh << 'EOF'
#!/bin/bash
echo url="https://www.duckdns.org/update?domains=meu-mesh&token=SEU_TOKEN_AQUI&ip=" \
  | curl -k -o ~/duckdns/duck.log -K -
EOF

chmod 700 duck.sh
./duck.sh
cat duck.log
```

- [ ] Output de `cat duck.log`: `[______]`
- [ ] Esperado: `OK`. Se `KO`, troubleshoot na quick-ref.

### 3.3. Crontab

```bash
crontab -e
# Adiciona:
# */5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
crontab -l | grep duckdns
```

- [ ] Linha aparece em `crontab -l`

📸 Screenshot 1: `crontab -l` output

---

## Section 4 — Networking

### 4.1. Port forwarding no router

- [ ] Regra criada: external 5223 → internal `[IP_DO_PI]:5223` TCP
- [ ] Regra ativa (enabled)
- [ ] Save & apply confirmado

📸 Screenshot 2: regra de port forwarding visível no router

### 4.2. IP fixo do Pi

Escolhe método (com base em 0.3):

**Método 1: DHCP Reservation no router (recomendado)**
- [ ] MAC do Pi: `[______]` (`ip link show eth0` mostra)
- [ ] Reservation criada: MAC → `[______]`

**Método 2 (Bookworm 12+): NetworkManager**
- [ ] `sudo nmtui` aberto, IPv4 manual configurado
- [ ] Reboot: `sudo systemctl restart NetworkManager`

**Método 3 (Bullseye 11-): dhcpcd**
- [ ] `/etc/dhcpcd.conf` editado
- [ ] Reboot: `sudo reboot`

Validar:
```bash
ip -4 addr show
```
- [ ] IP do Pi: `[______]` (deve ser fixo agora)

### 4.3. UFW

```bash
sudo apt install -y ufw
sudo ufw allow 22/tcp
sudo ufw allow 5223/tcp
sudo ufw enable
sudo ufw status verbose
```

- [ ] Output `sudo ufw status verbose` mostra:
  - `22/tcp ALLOW`
  - `5223/tcp ALLOW`
  - Default: `deny (incoming)`

📸 Screenshot 3: `sudo ufw status verbose` output

---

## Section 5 — Pi preparation

### 5.1. Updates

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget vim htop net-tools ca-certificates
```

- [ ] Updates concluídos sem erros
- [ ] Reboot se kernel atualizou: `sudo reboot`

### 5.2. Docker install

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
exit  # logout
# SSH novamente
```

Após SSH novo:
```bash
docker --version
docker compose version
docker ps
```

- [ ] `docker --version`: `[______]`
- [ ] `docker compose version`: `[______]`
- [ ] `docker ps` corre sem erro de permissão (grupo aplicado)

> ⚠️ Se `docker ps` der "permission denied", grupo não aplicou — `exit` + SSH novo, ou `newgrp docker`.

📸 Screenshot 4: outputs das três versões

---

## Section 6 — TLS model (read-only, sem ações)

- [ ] Confirmei que **NÃO** vou instalar `certbot`
- [ ] Confirmei que **NÃO** vou abrir port 80
- [ ] Entendi que o cert é gerado pelo container e o fingerprint é o que vai no address dos clientes

---

## Section 7 — Deployment

### 7.1. Estrutura de projeto

```bash
mkdir -p ~/simplex-server
cd ~/simplex-server
```

### 7.2. `.env`

```bash
# Gera password forte
openssl rand -base64 32
```

- [ ] **Password gerada:** `[______]` (guarda no password manager — vais precisar para clients e backup)

```bash
nano .env
```

Cola e edita:
```
SIMPLEX_ADDR=meu-mesh.duckdns.org
SIMPLEX_PASS=COLA_PASSWORD_AQUI
```

```bash
chmod 600 .env
ls -la .env
```

- [ ] `.env` existe com permissões `-rw-------`

### 7.3. `compose.yml`

```bash
nano compose.yml
```

Cola exatamente o que está na secção 7.3 do guide. Verifica:

- [ ] `services.smp-server.image: simplexchat/smp-server:latest`
- [ ] Volumes `./data:/var/opt/simplex` e `./config:/etc/opt/simplex`
- [ ] Port `5223:5223`
- [ ] `environment` usa `${SIMPLEX_ADDR}` e `${SIMPLEX_PASS}` (referência ao `.env`)
- [ ] **NÃO** tem montagem de `/etc/letsencrypt`

### 7.4. Deploy

```bash
cd ~/simplex-server
docker compose up -d
docker compose ps
```

- [ ] Container `simplex_smp` está `Up`

📸 Screenshot 5: `docker compose ps` output

### 7.5. Capturar logs iniciais (CRÍTICO para validation)

```bash
# No outro terminal SSH:
docker compose logs --tail=200 > ~/simplex-deploy-logs.txt 2>&1
cat ~/simplex-deploy-logs.txt
```

- [ ] Logs guardados em `~/simplex-deploy-logs.txt`
- [ ] **Anexar este ficheiro ao validation report**

### 7.6. Extrair server fingerprint dos logs

```bash
grep -iE "fingerprint|server address|smp://" ~/simplex-deploy-logs.txt
```

- [ ] **Server address completo:** `[______]`
- [ ] Formato esperado: `smp://FINGERPRINT@meu-mesh.duckdns.org:5223`

> ⚠️ **Se grep não retorna nada:** ver secção [V3] na quick-ref. Há fallbacks.

📸 Screenshot 6: server address visível

### 7.7. Verificar port aberto

```bash
sudo ss -tlnp | grep 5223
```

- [ ] Port 5223 aparece em LISTEN, com docker-proxy ou similar

---

## VALIDATION CRÍTICA — 5 [VERIFICAR]

> Esta é a parte mais importante. Cada um dos 5 pontos vai gerar uma decisão sobre se o guide está correto ou precisa de update.

### [V1] Volumes paths

**Hipótese a validar:** os volumes `/var/opt/simplex` (data) e `/etc/opt/simplex` (config) são os paths corretos usados pela imagem.

**Procedimento:**

```bash
cd ~/simplex-server

# Método 1: olhar dentro do container
docker compose exec smp-server ls -la /var/opt/simplex/ 2>&1
echo "---"
docker compose exec smp-server ls -la /etc/opt/simplex/ 2>&1
echo "---"

# Método 2: olhar para os mount points
docker inspect simplex_smp --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
echo "---"

# Método 3: olhar para o que está no host (deve refletir o container)
ls -la ~/simplex-server/data/ 2>&1
echo "---"
ls -la ~/simplex-server/config/ 2>&1
```

> ⚠️ Se `docker compose exec` falhar com "executable file not found" → a imagem pode não ter shell. Tentar:
>
> ```bash
> docker compose exec smp-server /bin/sh
> # ou
> docker run --rm -it simplexchat/smp-server:latest /bin/sh
> ```
>
> Se nenhum funciona, a imagem é distroless — usar só Método 2 e 3.

**Capturar:**

- [ ] **Paths observados nos mount points:** `[______]`
- [ ] **Conteúdo de `/var/opt/simplex/`:** `[______]`
- [ ] **Conteúdo de `/etc/opt/simplex/`:** `[______]`
- [ ] **Conteúdo de `~/simplex-server/data/`:** `[______]`
- [ ] **Conteúdo de `~/simplex-server/config/`:** `[______]`

**Resultado [V1]:**
- [ ] ✅ MATCH (paths são exatamente como no guide)
- [ ] ⚠️ DIFFERENT (quais são os paths reais? `[______]`)
- [ ] ❌ ERROR (qual o erro? `[______]`)

📸 Screenshot 7: outputs dos 3 métodos

---

### [V2] PASS env var — comportamento

**Hipótese a validar:** a env var `PASS` controla auth para criação de queues novas. Sem PASS correto, clients não conseguem criar queues no servidor.

**Procedimento:**

```bash
# Confirmar que PASS está exportada para o container
docker compose exec smp-server env 2>&1 | grep -i pass
echo "---"

# Ver a config gerada (se existir)
ls ~/simplex-server/config/ 2>&1
cat ~/simplex-server/config/smp-server.ini 2>/dev/null || echo "smp-server.ini não existe"
echo "---"

# Procurar nos logs por menção a auth/password
grep -iE "auth|password|pass|basic" ~/simplex-deploy-logs.txt | head -20
```

**Capturar:**

- [ ] **`PASS` aparece em `env`?** `[YES / NO]`. Valor visível: `[NO — está mascarada / YES — vejo a password]` (esperado: visível, env vars não são mascaradas)
- [ ] **Há `smp-server.ini`?** `[YES / NO]`. Se SIM, contém `auth_password` ou similar? `[______]`
- [ ] **Logs mencionam basic auth?** `[YES / NO]`. Citação relevante: `[______]`

**Teste funcional:**

Da app SimpleX (cliente), tenta adicionar o teu servidor **sem** password:

- [ ] Adicionei `smp://FINGERPRINT@meu-mesh.duckdns.org:5223` (sem auth) no client
- [ ] **Client consegue conectar?** `[YES / NO]`
- [ ] **Client consegue criar queue (criar contact link)?** `[YES / NO]`

Se YES YES → `PASS` não é exigida para uso público. É opcional, controla quem pode criar queues administrativamente.
Se YES NO → `PASS` é exigida para criação de queues. Client precisa do PASS no address.

**Resultado [V2]:**
- [ ] ✅ MATCH (comportamento como descrito no guide)
- [ ] ⚠️ DIFFERENT (descreve: `[______]`)
- [ ] ❌ ERROR (`[______]`)

---

### [V3] Format dos logs e exposição da fingerprint

**Hipótese a validar:** a fingerprint do server aparece nos logs no boot, em formato extraível.

**Procedimento:**

```bash
# Vários greps para apanhar diferentes formatos possíveis
grep -iE "fingerprint" ~/simplex-deploy-logs.txt
echo "---fingerprint---"
grep -iE "server address" ~/simplex-deploy-logs.txt
echo "---server address---"
grep -iE "^smp://" ~/simplex-deploy-logs.txt
echo "---smp://---"
grep -iE "hash|sha" ~/simplex-deploy-logs.txt | head -10
echo "---hash/sha---"

# Se grep nos logs não retornar nada útil, procurar em ficheiros do volume config/
ls ~/simplex-server/config/
echo "---ficheiros em config/---"

# Procurar ficheiros que pareçam conter fingerprint
for f in ~/simplex-server/config/*; do
  echo "=== $f ==="
  file "$f"
  if [[ $(file "$f") == *"ASCII"* ]] || [[ $(file "$f") == *"text"* ]]; then
    head -5 "$f"
  fi
done
```

**Capturar:**

- [ ] **Fingerprint apareceu em qual grep?** `[______]`
- [ ] **Server address completo encontrado:** `[______]`
- [ ] **Ficheiros em `config/`:** `[______]`
- [ ] Se a fingerprint só estava num ficheiro (não nos logs), qual? `[______]`

**Resultado [V3]:**
- [ ] ✅ MATCH (logs expõem fingerprint claramente)
- [ ] ⚠️ DIFFERENT (formato é `[______]` — guide precisa atualizar exemplo)
- [ ] ❌ NOT FOUND (fingerprint só acessível via `[______]`)

📸 Screenshot 8: comando que extraiu fingerprint + output

---

### [V4] Dual address (clearnet + onion)

> Só validar se ativaste Tor hidden service (secção 8). Se não, marcar `N/A`.

**Hipótese a validar:** o SMP server suporta address com clearnet + onion separados por vírgula: `smp://FP@clearnet,onion:port`.

**Procedimento:**

Após configurar Tor hidden service (secção 8 do guide):

```bash
# Hostname onion
sudo cat /var/lib/tor/simplex/hostname
echo "---"

# Verificar se o SMP server detectou o onion (precisa env var ou config)
grep -iE "onion|tor|hidden" ~/simplex-deploy-logs.txt
echo "---"

# Procurar se há env var específica para onion
docker compose exec smp-server env 2>&1 | grep -iE "onion|tor"
```

**Capturar:**

- [ ] **Onion hostname:** `[______].onion`
- [ ] **SMP server detectou Tor automaticamente?** `[YES / NO]`
- [ ] **Há env var dedicada para onion address?** `[______]`

> ⚠️ **Provável outcome:** o SMP server **não** detecta Tor automaticamente. Precisas configurá-lo explicitamente (env var como `ONION_ADDR=...` ou config no `smp-server.ini`). Documentação atual em <https://simplex.chat/docs/server.html> deve clarificar.

**Teste:** tenta adicionar no client o address combinado:
```
smp://FINGERPRINT@meu-mesh.duckdns.org,abc...xyz.onion:5223
```

- [ ] **Client aceita o formato?** `[YES / NO]`
- [ ] **Conexão funciona via Tor?** (testar com Orbot ativo, sem WiFi/dados nativos para evitar fallback) `[YES / NO]`

**Resultado [V4]:**
- [ ] ✅ MATCH (dual address funciona como descrito)
- [ ] ⚠️ DIFFERENT (formato real é `[______]` ou requer config diferente)
- [ ] ❌ NOT SUPPORTED nesta versão
- [ ] N/A (Tor não testado neste deployment)

---

### [V5] UID interno da imagem

**Hipótese a validar:** a imagem `simplexchat/smp-server:latest` corre como UID 1000:1000 (assumido no guide para troubleshooting de permissões).

**Procedimento:**

```bash
# Inspecionar config da imagem
docker inspect simplexchat/smp-server:latest --format '{{.Config.User}}'
echo "---"

# UID dentro do container
docker compose exec smp-server id 2>&1
echo "---"
# Fallback se sem shell:
docker compose exec smp-server whoami 2>&1
echo "---"

# UID dos ficheiros criados nos volumes (no host)
ls -la ~/simplex-server/data/
echo "---"
ls -la ~/simplex-server/config/
```

**Capturar:**

- [ ] **`Config.User` da imagem:** `[______]` (vazio = root, ou UID:GID, ou nome de user)
- [ ] **`id` dentro do container:** `[______]`
- [ ] **UID dos ficheiros em `data/`:** `[______]`
- [ ] **UID dos ficheiros em `config/`:** `[______]`

**Resultado [V5]:**
- [ ] ✅ MATCH (UID é 1000:1000)
- [ ] ⚠️ DIFFERENT (UID real é `[______]`)
- [ ] ⚠️ ROOT (container corre como root — guide precisa update sobre permissões)

---

## Section 8 — Tor hidden service (opcional)

Só preencher se decidiste ativar.

```bash
sudo apt install -y tor
sudo nano /etc/tor/torrc
```

Adicionar:
```
HiddenServiceDir /var/lib/tor/simplex/
HiddenServicePort 5223 127.0.0.1:5223
```

```bash
sudo systemctl restart tor
sudo systemctl enable tor
sleep 5
sudo cat /var/lib/tor/simplex/hostname
```

- [ ] Hostname `.onion`: `[______]`
- [ ] [V4] validation feita (acima)

---

## Section 9 — Client configuration

### 9.1. Android (primário)

- [ ] App SimpleX instalada (versão: `[______]`)
- [ ] Settings → Network & servers → Your SMP servers → Add server
- [ ] Server address colado: `[______]`
- [ ] Save → server aparece na lista
- [ ] Settings → Developer tools → Server tests → Run
- [ ] Resultado do test: `[CONNECTED / FAILED — qual erro?]`

📸 Screenshot 9: server tests output (success ou failure)

### 9.2. Segundo device (para teste end-to-end)

- [ ] Device B identificado: `[Android / iOS / Desktop]`
- [ ] Mesmo server address adicionado
- [ ] Connectivity test passa

---

## Section 10 — Testing

### 10.1. Test 1 — Connectivity externa

> Importante: faz isto de uma rede DIFERENTE (rede móvel do telemóvel via tethering, ou casa de amigo).

```bash
nc -zv meu-mesh.duckdns.org 5223
```

- [ ] Output: `[______]`
- [ ] Esperado: `succeeded` ou `Connected to`

### 10.2. Test 2 — Message flow

- [ ] Device A: criar new contact link
- [ ] Device B: connect via link
- [ ] Mensagem A → B: `[ENVIADA / FALHOU]`
- [ ] Mensagem B → A: `[ENVIADA / FALHOU]`
- [ ] Latência aproximada: `[______]` segundos

📸 Screenshot 10: conversa funcional entre A e B

### 10.3. Test 3 — Resources

```bash
docker stats simplex_smp --no-stream
```

- [ ] **CPU%:** `[______]`
- [ ] **MEM USAGE:** `[______]`
- [ ] **NET I/O:** `[______]`
- [ ] Esperado idle: <100 MB RAM, <5% CPU

📸 Screenshot 11: docker stats output

### 10.4. Test 4 — Uptime (24h, opcional)

- [ ] Server deixado a correr 24h
- [ ] Após 24h: `docker compose ps` ainda mostra `Up`
- [ ] Logs sem erros críticos: `docker compose logs --since 24h | grep -iE "error|fatal"`
- [ ] Output: `[______]`

---

## Section 11 — Backup test

```bash
cd ~/simplex-server
docker compose stop

tar -czf simplex-backup-$(date +%Y%m%d-%H%M%S).tar.gz \
  data/ config/ .env compose.yml

ls -la simplex-backup-*.tar.gz

docker compose start
docker compose ps
```

- [ ] Backup criado, tamanho: `[______]`
- [ ] Container reinicia OK após backup
- [ ] **Mover backup para sítio seguro fora do Pi**

---

## Section 12 — Final state

### 12.1. Capturar estado final

```bash
# Tudo num só comando para o report
{
  echo "=== docker compose ps ==="
  docker compose ps
  echo
  echo "=== docker stats (snapshot) ==="
  docker stats simplex_smp --no-stream
  echo
  echo "=== ufw status ==="
  sudo ufw status verbose
  echo
  echo "=== ss -tlnp ==="
  sudo ss -tlnp | grep -E "5223|9050"
  echo
  echo "=== logs últimas 50 linhas ==="
  docker compose logs --tail=50
} > ~/simplex-final-state.txt 2>&1

cat ~/simplex-final-state.txt
```

- [ ] `~/simplex-final-state.txt` capturado
- [ ] **Anexar ao validation report**

### 12.2. Documentação para o futuro-tu

Anota num sítio seguro (password manager + backup encriptado):

- [ ] **Server address completo:** `[______]`
- [ ] **PASS gerada:** `[______]`
- [ ] **Onion hostname (se aplicável):** `[______]`
- [ ] **IP fixo do Pi:** `[______]`
- [ ] **Localização do backup inicial:** `[______]`

---

## Done

- [ ] Validation report preenchido (próximo ficheiro)
- [ ] Screenshots organizadas numa pasta
- [ ] Logs anexados (`simplex-deploy-logs.txt`, `simplex-final-state.txt`)
- [ ] Issues encontrados anotados durante deployment

**Próximo:** preenche `validation-report-template.md` com os achados.
