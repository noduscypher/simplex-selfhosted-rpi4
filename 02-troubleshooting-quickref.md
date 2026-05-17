# SimpleX SMP RPi4 — Troubleshooting Quick-Ref

**Para ter aberto durante deployment.** Issues por ordem de probabilidade.

---

## 🔴 Issue 1 — Container não arranca / crash loop

### Sintomas
- `docker compose ps` mostra `Restarting` ou `Exited`
- Logs imediatos param e reiniciam ciclicamente

### Diagnóstico

```bash
docker compose logs --tail=100 smp-server
docker compose logs --tail=100 smp-server | grep -iE "error|fatal|cannot|failed"
```

### Causas prováveis e fix

| Causa | Sintoma nos logs | Fix |
|---|---|---|
| `ADDR` inválido | `getaddrinfo: Name or service not known` | Verifica `cat .env`, valida `nslookup $SIMPLEX_ADDR` |
| Port 5223 em uso | `bind: address already in use` | `sudo ss -tlnp \| grep 5223`. Mata processo ou troca port no compose |
| Permissões volume | `Permission denied` em `data/` ou `config/` | `sudo chown -R 1000:1000 data/ config/` (ou UID real de [V5]) |
| Imagem corrupta | `manifest unknown` ou `pull access denied` | `docker compose pull` para refazer |
| `.env` não carrega | env vars vazias nos logs | Confirma `.env` está no mesmo dir que `compose.yml` |

### Reset nuclear (último recurso)

```bash
cd ~/simplex-server
docker compose down
sudo rm -rf data/ config/  # ⚠️ apaga tudo, perdes fingerprint atual
docker compose up -d
```

---

## 🔴 Issue 2 — Fingerprint não aparece nos logs

### Sintomas
- `grep -i fingerprint ~/simplex-deploy-logs.txt` retorna vazio
- Não tens server address para dar aos clients

### Diagnóstico

```bash
# Tenta vários greps
grep -iE "fingerprint|server address|smp://|hash|^pub" ~/simplex-deploy-logs.txt

# Ver primeiras 50 linhas (boot completo)
docker compose logs --tail=200 smp-server | head -100

# Ver ficheiros do volume config (onde a fingerprint pode estar guardada)
ls -la ~/simplex-server/config/
```

### Fallbacks para extrair fingerprint

```bash
# Se há ficheiro tipo "server_pub_hash.bin" ou similar
for f in ~/simplex-server/config/*; do
  echo "=== $f ==="
  file "$f"
  if file "$f" | grep -qE "ASCII|text"; then
    cat "$f"
  else
    xxd "$f" | head -3
  fi
done

# Ver dentro do container
docker compose exec smp-server ls /etc/opt/simplex/
docker compose exec smp-server cat /etc/opt/simplex/server_pub_hash.bin 2>&1 | xxd | head
```

### Se mesmo assim não encontras

> ⚠️ Pode ser que esta versão da imagem mostre a fingerprint só ao primeiro arranque, num ficheiro temporário ou stderr. Tenta:
>
> ```bash
> # Para o container, apaga config (cuidado), recomeça e captura tudo
> docker compose down
> rm -rf config/
> docker compose up 2>&1 | tee ~/simplex-first-boot.log
> # Para com Ctrl+C depois de 30 segundos
> grep -i "smp://" ~/simplex-first-boot.log
> ```

---

## 🟡 Issue 3 — Port 5223 não alcançável externamente

### Sintomas
- `nc -zv $ADDR 5223` de fora da rede falha
- Clients dão "Network error"

### Diagnóstico (sequência)

```bash
# 1. Container a listening localmente?
sudo ss -tlnp | grep 5223
# Esperado: 0.0.0.0:5223 ou *:5223 com docker-proxy

# 2. UFW a bloquear?
sudo ufw status verbose | grep 5223
# Esperado: 5223/tcp ALLOW IN

# 3. Localhost responde?
nc -zv 127.0.0.1 5223
# Esperado: succeeded

# 4. Pi responde no LAN?
# (de outra máquina na mesma rede:)
nc -zv 192.168.X.X 5223  # IP do Pi
# Esperado: succeeded

# 5. Externo responde?
# (de rede DIFERENTE — telemóvel com 4G via tethering, ou VPS:)
nc -zv $ADDR 5223
# Aqui pode falhar
```

### Diagnóstico → causa

| Onde falha primeiro | Causa | Fix |
|---|---|---|
| Step 1 | Container não está a listening | Issue 1 |
| Step 2 | UFW bloqueia | `sudo ufw allow 5223/tcp` |
| Step 3 | Container não fez bind correto | Issue 1, verifica `ports:` em compose.yml |
| Step 4 | Firewall do Pi (não UFW) ou IP diferente | Verifica `ip addr`, confirma IP usado |
| Step 5 | Port forwarding do router OU CGNAT | Volta ao router, ou ativa Tor |

---

## 🟡 Issue 4 — DuckDNS não atualiza ou retorna KO

### Sintomas
- `cat ~/duckdns/duck.log` mostra `KO` em vez de `OK`
- Domínio não resolve para o IP atual

### Causas

```bash
# Validar token e domínio
cat ~/duckdns/duck.sh
# Confirma que `domains=` tem o teu subdomain (sem .duckdns.org)
# Confirma que `token=` tem o token real

# Testar manualmente
curl -k "https://www.duckdns.org/update?domains=meu-mesh&token=TOKEN&ip="
```

### Output → causa

- `OK` → tudo bem, esperar propagação (até 5 min)
- `KO` → token errado, ou domain não te pertence
- timeout → DuckDNS down (raro) ou rede sem saída

### Validar resolução DNS

```bash
nslookup meu-mesh.duckdns.org
# Esperado: Address: <teu IP público>

dig +short meu-mesh.duckdns.org
```

---

## 🟡 Issue 5 — `docker compose exec` falha com "executable file not found"

### Sintomas
- Erro: `OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH`

### Causa
A imagem é distroless (sem shell). Não é problema, é design.

### Workarounds para validation

```bash
# Em vez de exec, usa inspect
docker inspect simplex_smp --format '{{json .Config}}' | jq

# Para ler ficheiros, copia para fora do container
docker cp simplex_smp:/etc/opt/simplex/. /tmp/simplex-config-snapshot/
ls -la /tmp/simplex-config-snapshot/

# Para ver env vars
docker inspect simplex_smp --format '{{range .Config.Env}}{{println .}}{{end}}'
```

> Ajusta os procedimentos de [V1], [V2], [V5] para usar `docker cp` e `docker inspect` em vez de `exec`.

---

## 🟢 Issue 6 — Client conecta mas mensagens não chegam

### Sintomas
- Server tests no client passam
- `nc -zv` externo passa
- Mas mensagem entre A e B fica "Sending..." indefinidamente

### Diagnóstico

```bash
# Ambos clients usam o teu server?
# Settings → Network & servers em ambos. Confirma.

# Logs do server mostram queue creation?
docker compose logs --tail=100 smp-server | grep -iE "queue|connect"

# Server cheio? (improvável em teste, mas...)
docker stats simplex_smp --no-stream
# RAM > 80% = problema
```

### Causas comuns

- **Client B nunca usou o server:** mesmo que esteja na lista, se não criou nenhuma queue lá, mensagens vão por preset SimpleX. Solução: B precisa também desativar presets ou aceitar que ambos usam o teu server.
- **Address diferente nos clients:** verifica que ambos têm exatamente o mesmo `smp://...` (incluindo fingerprint exato)
- **Connection link foi criado antes do server estar adicionado:** SimpleX usa "queues" persistentes. Cada contact link aponta para um server específico. Recria o contact link **depois** do teu server estar adicionado em ambos.

### Teste limpo

1. Adiciona o teu server **em ambos** os clients
2. Em ambos, marca o teu server como "preferred for new conversations"
3. Em A: cria **novo** contact link (não reusa antigo)
4. Em B: connect via novo link
5. Mensagem A → B

---

## 🟢 Issue 7 — Pi reboota sozinho / freezes

### Sintomas
- SSH cai aleatoriamente
- LED amarelo a piscar
- Logs do kernel mencionam `Under-voltage detected`

### Diagnóstico

```bash
dmesg | grep -iE "voltage|throttle"
vcgencmd get_throttled
# Output:
# 0x0      → tudo OK
# 0x50000  → throttled in past
# qualquer outro != 0 → problema
```

### Fix

- PSU oficial 5 V/3 A USB-C
- Cabo curto (cabos longos têm queda de tensão)
- Se está num USB hub, remove e liga direto

---

## 🟢 Issue 8 — Backup tar fica corrompido ou inconsistente

### Sintomas
- `tar tzf backup.tar.gz` dá erro
- Restore falha

### Causa
Backup feito com container a correr (race conditions em ficheiros).

### Fix

```bash
cd ~/simplex-server
docker compose stop  # ESPERA o stop completo
sleep 3
tar -czf simplex-backup-$(date +%Y%m%d-%H%M%S).tar.gz data/ config/ .env compose.yml
docker compose start

# Validar backup
tar tzf simplex-backup-*.tar.gz | head
```

---

## 🟢 Issue 9 — Tor hidden service não aparece

### Sintomas
- `sudo cat /var/lib/tor/simplex/hostname` dá "No such file or directory"

### Diagnóstico

```bash
sudo systemctl status tor
sudo journalctl -u tor --since "5 minutes ago" | tail -50

# Tor está a ler o /etc/tor/torrc?
sudo grep -E "HiddenService" /etc/tor/torrc
```

### Causas

- Tor parou imediatamente após restart (config inválida) → ver journalctl
- Permissões erradas em `/var/lib/tor/simplex/` → `sudo chown -R debian-tor:debian-tor /var/lib/tor/`
- Tor não fez restart real → `sudo systemctl restart tor && sleep 10`

---

## 🟢 Issue 10 — Apt update falha em Pi OS recente

### Sintoma
- `sudo apt update` dá erros de chave GPG ou repositórios inacessíveis

### Diagnóstico

```bash
sudo apt update 2>&1 | grep -iE "warn|err"
```

### Fix comum

```bash
# Atualizar chaves Raspberry Pi
sudo apt install -y --reinstall raspberrypi-archive-keyring debian-archive-keyring

# Limpar cache
sudo apt clean
sudo apt update
```

---

## Comandos de emergência (cheatsheet)

```bash
# Tudo morreu — começar de zero localmente (sem perder o backup)
cd ~/simplex-server
docker compose down
docker compose pull
docker compose up -d
docker compose logs -f

# Container vivo mas comportamento estranho
docker compose restart
docker compose logs --tail=100

# Ver recursos do Pi
free -h         # RAM
df -h           # Disk
top -bn1 | head # CPU

# Network
ip -4 addr
ip route
ping -c 3 1.1.1.1
ping -c 3 google.com

# Capturar tudo para análise
{
  date
  uname -a
  free -h
  df -h
  docker ps -a
  docker compose logs --tail=200
  sudo ufw status
  sudo ss -tlnp
  ip -4 addr
} > ~/simplex-debug-$(date +%s).txt 2>&1
```

---

## Quando reportar issue ao guide vs corrigir localmente

| Tipo | Ação |
|---|---|
| Erro de comando no guide (typo, syntax errada) | 📝 Reportar — atualizar guide |
| Comportamento da imagem SimpleX diferente do documentado | 📝 Reportar — atualizar [VERIFICAR] no guide |
| Erro do teu setup específico (CGNAT, ISP, hardware) | 🔧 Anotar para troubleshooting do guide |
| Erro intermitente que resolve com restart | 🔧 Anotar — pode ser bug upstream |

Anota tudo no `validation-report-template.md`.


---

⚡ **Lightning:** `trustyflame02@zeuspay.com`

No sats? Carry the signal further: share it, mirror it, translate it, remix it.

---

## ☥ License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

---

*☥ Walk in silence, ₿ keep the signal alive.*


*₿uilt with love in cooperation with nature.* · 🜁 🜂 ☰ ☱ ☲ ☳ ☴ ☵ ☶ ☷ 🜃 🜄
