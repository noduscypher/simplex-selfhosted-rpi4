# SimpleX SMP Server RPi4 — Validation Report

**Para preencher APÓS deployment.** Resultado deste relatório → decisão sobre updates ao guide antes de avançar para EN/ES/JA.

---

## 1. Metadata

**Data do deployment:** `[YYYY-MM-DD]`
**Tester:** `[nome / pseudónimo]`
**Duração total (com pausas):** `[Xh Ym]`
**Duração ativa (excluindo pausas):** `[Xh Ym]`

### Hardware

| Item | Valor |
|---|---|
| Modelo do Pi | `[Raspberry Pi 4 Model B Rev 1.X]` |
| RAM | `[X GB]` |
| SD card | `[capacidade GB / classe]` |
| PSU | `[oficial / outra: ____]` |
| Conexão | `[Ethernet / WiFi]` |
| Localização física | `[ex: gaveta, prateleira, exterior]` |

### Software

| Item | Versão observada |
|---|---|
| Pi OS | `[ex: Raspberry Pi OS Lite Bookworm 12.X 64-bit]` |
| Kernel | `[output de uname -r]` |
| Docker | `[output de docker --version]` |
| Docker Compose | `[output de docker compose version]` |
| SimpleX image tag | `[ex: latest, ou pinned se foi fazer pin]` |
| SimpleX image digest | `[output de docker inspect ... \| grep Image]` |

### Network

| Item | Valor |
|---|---|
| ISP | `[MEO / NOS / Vodafone / NOWO / outro]` |
| Tipo de ligação | `[Fibra / DSL / 4G / Starlink]` |
| IP público | `[anónimo / categoria — não preciso do IP em si]` |
| CGNAT? | `[YES / NO / unsure]` |
| DynDNS provider | `[DuckDNS / domain próprio / nenhum]` |
| Tor hidden service | `[YES / NO]` |
| Router | `[marca/modelo]` |

---

## 2. Outcome geral

Marca **uma**:

- [ ] ✅ **SUCCESS** — server live, clients ligados, mensagens fluem
- [ ] 🟡 **PARTIAL** — server live mas com issues a documentar abaixo
- [ ] ❌ **BLOCKED** — não consegui completar; debug necessário

**Resumo (1-3 frases):**

```
[escreve aqui]
```

---

## 3. Validação dos 5 [VERIFICAR]

### [V1] Volumes paths

**Hipótese do guide:** volumes mapeados em `/var/opt/simplex` (data) e `/etc/opt/simplex` (config).

**Resultado:**
- [ ] ✅ MATCH — paths exatamente como no guide
- [ ] ⚠️ DIFFERENT — paths reais: `[______]`
- [ ] ❌ ERROR — qual: `[______]`

**Outputs capturados:**

```
docker inspect (mount points):
[______]

docker compose exec smp-server ls /var/opt/simplex/:
[______]

docker compose exec smp-server ls /etc/opt/simplex/:
[______]

ls -la ~/simplex-server/data/:
[______]

ls -la ~/simplex-server/config/:
[______]
```

**Action item para o guide:**
- [ ] Manter como está
- [ ] Atualizar paths para `[______]`
- [ ] Adicionar nota sobre comportamento observado

---

### [V2] PASS env var

**Hipótese do guide:** `PASS` controla "basic auth password" para criação de queues.

**Resultado:**
- [ ] ✅ MATCH — controla auth como descrito
- [ ] ⚠️ DIFFERENT — comportamento real: `[______]`
- [ ] ❌ ERROR — `[______]`

**Outputs:**

```
docker compose exec smp-server env | grep -i pass:
[______]

smp-server.ini (se existe):
[______]

Logs com auth/password:
[______]
```

**Teste funcional:**

| Teste | Resultado | Notas |
|---|---|---|
| Add server sem PASS | `[connects/fails]` | `[______]` |
| Add server com PASS no address | `[connects/fails]` | `[______]` |
| Criar contact link no server | `[works/fails]` | `[______]` |
| Sem PASS, criar contact link | `[works/fails]` | `[______]` |

**Conclusão sobre o papel da `PASS`:**

```
[Descreve em 1-2 frases o comportamento observado]
```

**Action item para o guide:**
- [ ] Manter descrição atual
- [ ] Atualizar secção 7.2 com comportamento real
- [ ] Adicionar exemplo de address com auth incluída

---

### [V3] Format dos logs e exposição da fingerprint

**Hipótese do guide:** fingerprint aparece nos logs durante boot, prefixada por "fingerprint" ou similar.

**Resultado:**
- [ ] ✅ MATCH — apareceu nos logs como esperado
- [ ] ⚠️ DIFFERENT — formato real: `[______]`
- [ ] ❌ NOT FOUND — fingerprint só acessível via `[______]`

**Onde apareceu (qual grep funcionou):**

```
[Comando usado e output que extraiu a fingerprint:]

[______]
```

**Server address completo extraído:**

```
smp://[FINGERPRINT_HASH_ANONIMIZADO]@[host]:[port]
```

**Ficheiros relevantes em `config/` (se foi por aí):**

```
[output de ls + descrição dos ficheiros]
```

**Action item para o guide:**
- [ ] Manter exemplo de extração atual
- [ ] Atualizar secção 7.6 com formato real dos logs
- [ ] Adicionar fallback de extração via ficheiro em `config/`

---

### [V4] Dual address (clearnet + onion)

**Hipótese do guide:** address `smp://FP@clearnet,onion:port` é suportado nativamente.

**Resultado:**
- [ ] ✅ MATCH — formato funciona como descrito
- [ ] ⚠️ DIFFERENT — formato real ou config exigida: `[______]`
- [ ] ❌ NOT SUPPORTED nesta versão
- [ ] N/A — Tor não testado

**Se testado:**

```
Onion hostname: [______].onion
SMP server detectou Tor automaticamente? [______]
Env var/config necessária: [______]
Address combinado tentado: [______]
Client aceita o formato? [YES / NO]
Conexão via Tor funciona? [YES / NO]
```

**Action item para o guide:**
- [ ] Manter secção 8 como está
- [ ] Atualizar com env var específica para onion
- [ ] Marcar [V4] como invalidado se não suportado
- [ ] N/A

---

### [V5] UID interno da imagem

**Hipótese do guide:** imagem corre como UID 1000:1000.

**Resultado:**
- [ ] ✅ MATCH — UID 1000:1000
- [ ] ⚠️ DIFFERENT — UID real: `[______]`
- [ ] ⚠️ ROOT — corre como root (UID 0)

**Outputs:**

```
docker inspect Config.User: [______]
docker compose exec smp-server id: [______]
UID dos ficheiros em data/: [______]
UID dos ficheiros em config/: [______]
```

**Action item para o guide:**
- [ ] Manter `chown -R 1000:1000` no troubleshooting
- [ ] Atualizar para `[______]:[______]`
- [ ] Atualizar nota sobre UID na secção 14.2

---

## 4. Issues encontrados durante deployment

> Listar **todos** os problemas, mesmo os menores. Cada um → potencial update ao guide.

### Issue #1

**Secção do guide afetada:** `[X.Y]`
**Severidade:** `[CRÍTICO / MODERADO / MENOR]`

**Descrição (o que aconteceu):**
```
[descreve o sintoma]
```

**Comando(s) que falharam ou deram output inesperado:**
```
[copy-paste exato]
```

**Output observado:**
```
[copy-paste exato do erro]
```

**Como resolvi:**
```
[se resolvi — comando(s) ou mudança aplicada]
[se não resolvi — descreve estado atual]
```

**Update sugerido ao guide:**
```
[descreve mudança]
```

**Screenshot/log anexado:** `[filename]`

---

### Issue #2

[mesmo formato — copia o template acima]

---

### Issue #N

[mesmo formato]

---

## 5. Updates necessários ao guide (síntese)

### 5.1. Críticos (impedem deployment)

- [ ] Secção `[X.Y]` — mudança: `[______]`
- [ ] ...

### 5.2. Significativos (degradam UX mas não bloqueiam)

- [ ] Secção `[X.Y]` — mudança: `[______]`
- [ ] ...

### 5.3. Menores (typos, clarificações)

- [ ] Secção `[X.Y]` — mudança: `[______]`
- [ ] ...

### 5.4. Marker updates

Quais `[VERIFICAR contra docs SimpleX atuais]` podem virar `✓ Validado em deploy real (data, versão da imagem)`:

- [ ] [V1] → `✓ Validado` ou manter `⚠️`
- [ ] [V2] → `✓ Validado` ou manter `⚠️`
- [ ] [V3] → `✓ Validado` ou manter `⚠️`
- [ ] [V4] → `✓ Validado` ou manter `⚠️` ou `N/A`
- [ ] [V5] → `✓ Validado` ou manter `⚠️`

---

## 6. Client test results

### Device A (primário)

| Item | Valor |
|---|---|
| Plataforma | `[Android / iOS / Desktop Linux/Mac/Win]` |
| App version | `[______]` |
| Conexão ao server | `[SUCCESS / FAIL]` |
| Connectivity test in-app | `[PASS / FAIL]` |
| Mensagem enviada | `[YES / NO]` |
| Mensagem recebida | `[YES / NO]` |
| Latência aproximada | `[X segundos]` |
| Notas | `[______]` |

### Device B

| Item | Valor |
|---|---|
| Plataforma | `[______]` |
| App version | `[______]` |
| Conexão ao server | `[SUCCESS / FAIL]` |
| Mensagem A → B | `[YES / NO]` |
| Mensagem B → A | `[YES / NO]` |
| Notas | `[______]` |

### Tor client (se testado)

| Item | Valor |
|---|---|
| Plataforma | `[______]` |
| Tor mode | `[Orbot VPN / SOCKS proxy / nativo]` |
| Conexão via .onion | `[SUCCESS / FAIL]` |
| Latência vs clearnet | `[similar / X% mais lento]` |

---

## 7. Resource usage observado

### Idle (sem clients ativos)

```
docker stats output (idle):
CPU%:        [______]
MEM USAGE:   [______]
NET I/O:     [______]
BLOCK I/O:   [______]
```

### Sob load (durante teste de mensagens A↔B)

```
docker stats output (load):
CPU%:        [______]
MEM USAGE:   [______]
NET I/O:     [______]
```

### Sistema (Pi como um todo)

```
free -h:
[______]

df -h (volumes Docker):
[______]

uptime (load average):
[______]
```

---

## 8. Uptime test (24h, opcional)

- [ ] Server deixado ligado 24h
- [ ] **Status após 24h:** `[Up / Restarted X times / Crashed]`
- [ ] **Erros nos logs (24h window):** `[count + descrição]`
- [ ] **DuckDNS atualizou consistentemente:** `[YES / NO]`
- [ ] **Mensagens entre A e B continuam a funcionar:** `[YES / NO]`

---

## 9. Recommendations

### 9.1. Antes de partilhar o guide

- [ ] Atualizar com mudanças críticas e significativas (secção 5)
- [ ] Atualizar markers `[VERIFICAR]` validados → `✓ Validado`
- [ ] Re-revisão final por ti
- [ ] (Opcional) Teste de validação em segundo Pi para reproducibilidade

### 9.2. Antes de avançar para EN/ES/JA

- [ ] Versão PT estabilizada
- [ ] Não há mudanças críticas pendentes
- [ ] Documentação de "version" no guide reflete o que foi testado

### 9.3. Para deployment futuro / produção

```
[escreve recomendações aprendidas — ex: "usar pinned image tag em vez de :latest",
 "sempre fazer backup antes de docker compose pull", "Tor hidden service vale o esforço",
 "RAM 2GB chega para uso pessoal, 4GB para 5+ utilizadores", etc.]
```

### 9.4. Para o ecosystem mais alargado

```
[insights que afectam outros guides ou planeamento — ex: "SimpleX + RPi4 → eat 4GB RAM",
 "Reticulum bridge é mais complicado que pensava — adiar", etc.]
```

---

## 10. Anexos

Lista de ficheiros gerados durante deployment:

- [ ] `simplex-deploy-logs.txt` — logs iniciais
- [ ] `simplex-final-state.txt` — estado capturado no fim
- [ ] `simplex-first-boot.log` — primeiro arranque (se foi capturado)
- [ ] `simplex-debug-*.txt` — debug snapshots (se houve)
- [ ] Screenshots: `[lista numerada]`
- [ ] Backup encriptado: `[localização]`

---

## 11. Sign-off

**Confidence level pós-deployment:**

- [ ] **🟢 Deployment-ready** — guide pode ser publicado/partilhado tal como ficou após updates
- [ ] **🟡 Quase pronto** — falta validar `[X coisas específicas]` num segundo deployment
- [ ] **🔴 Precisa mais trabalho** — `[descreve]`

**Próxima ação:**

- [ ] Aplicar updates ao guide PT, partilhar com Claude para revisar pass cirúrgica
- [ ] Avançar para traduções EN/ES/JA
- [ ] Outro: `[______]`

---

⚡ **Lightning:** `trustyflame02@zeuspay.com`

No sats? Carry the signal further: share it, mirror it, translate it, remix it.

---

## ☥ License

[CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

---

*☥ Walk in silence, ₿ keep the signal alive.*

*Self-sovereign messaging. Mesh-ready. Off-grid capable.*

*₿uilt with love in cooperation with nature.* · 🜁 🜂 ☰ ☱ ☲ ☳ ☴ ☵ ☶ ☷ 🜃 🜄
