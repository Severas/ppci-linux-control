# 🏁 PPCI Linux Lite — Tutorial Completo

## 🎯 Objetivo

Criar um sistema onde o computador:
- Liga
- Baixa regras de um servidor
- Bloqueia a internet
- Aplica configurações automaticamente

---

## 🧠 Explicação Simples

- Servidor = professor (manda regras)
- Computador = aluno ou CLIENTE (recebe regras)
- Script = cérebro
- Config = regras

---

## 📁 PASSO 1 — Servidor

Crie a estrutura:

```
https://maratona.td.utfpr.edu.br/ppci-linux-lite/
```

Arquivos necessários:

* `bloqueio.sh`
* `maratona.conf`
* `wallpaper_ppci/wallpaper.png`

---

## 🧾 PASSO 2 — Arquivo de Configuração

Crie `maratona.conf`:

```ini
IPS="200.134.10.20"
PORTA=443
WALLPAPER_URL="https://maratona.td.utfpr.edu.br/ppci-linux-lite/wallpaper_ppci/wallpaper.png"
VERSION=1
```

---

## ⚙️ PASSO 3 — Script Principal (bloqueio.sh)

```bash
#!/bin/bash

set -e

URL_CONF="https://maratona.td.utfpr.edu.br/ppci-linux-lite/maratona.conf"
CACHE_CONF="/var/cache/maratona.conf"
TMP_CONF="/tmp/maratona.conf"
LOG="/var/log/maratona.log"

log(){
    echo "[ $(date) ] $1" | tee -a $LOG
}

log "Baixando configuração..."

if curl -fsSL "$URL_CONF" -o "$TMP_CONF"; then
    cp "$TMP_CONF" "$CACHE_CONF"
    log "Config atualizada"
else
    log "Erro download, usando cache"
    if [ -f "$CACHE_CONF" ]; then
        cp "$CACHE_CONF" "$TMP_CONF"
    else
        log "Sem config disponível!"
        exit 1
    fi
fi

source "$TMP_CONF"

if [ -z "$IPS" ] || [ -z "$PORTA" ]; then
    log "Variáveis inválidas"
    exit 1
fi

log "Aplicando firewall..."

iptables -F
iptables -t nat -F
iptables -X

iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

for ip in $IPS; do
    iptables -A OUTPUT -p tcp -d $ip --dport $PORTA -j ACCEPT
    iptables -A INPUT -p tcp -s $ip --sport $PORTA -j ACCEPT
done

log "Firewall aplicado"

# Wallpaper (para LXQt)

if [ ! -z "$WALLPAPER_URL" ]; then
    TMP_WALL="/tmp/wallpaper.png"
    DEST="/usr/share/backgrounds/ppci/wallpaper.png"

    if curl -fsSL "$WALLPAPER_URL" -o "$TMP_WALL"; then
        mkdir -p /usr/share/backgrounds/ppci
        cp "$TMP_WALL" "$DEST"
        log "Wallpaper baixado"

        (
        sleep 5

        USER_REAL=$(logname 2>/dev/null || echo $SUDO_USER)

        if [ ! -z "$USER_REAL" ]; then
            su - $USER_REAL -c "DISPLAY=:0 pcmanfm-qt --set-wallpaper=$DEST"
            log "Wallpaper aplicado (LXQt)"
        else
            log "Usuário não detectado"
        fi
        ) &

    else
        log "Erro wallpaper"
    fi
fi
```

---

## ⚙️ PASSO 4 — Loader (CLIENTE)

Crie:

```bash
sudo nano /usr/local/bin/maratona-loader.sh
```

Cole:

```bash
#!/bin/bash

URL="https://maratona.td.utfpr.edu.br/ppci-linux-lite/bloqueio.sh"
TMP="/tmp/bloqueio.sh"
LOCAL="/usr/local/bin/bloqueio-local.sh"
LOG="/var/log/maratona.log"

log(){
    echo "[ $(date) ] $1" | tee -a $LOG
}

log "Baixando script..."

if curl -fsSL "$URL" -o "$TMP"; then
    chmod +x "$TMP"
    cp "$TMP" "$LOCAL"
    log "Executando remoto"
    bash "$TMP"
else
    log "Erro download, usando local"
    if [ -f "$LOCAL" ]; then
        bash "$LOCAL"
    else
        log "Sem fallback!"
        exit 1
    fi
fi
```

Permissão:

```bash
sudo chmod +x /usr/local/bin/maratona-loader.sh
```

---

## ⚙️ PASSO 5 — systemd (CLIENTE)

```bash
sudo nano /etc/systemd/system/maratona.service
```

Conteúdo:

```ini
[Unit]
Description=Maratona Loader
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/maratona-loader.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Ativar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable maratona.service
```

---

## 🚀 PASSO 6 — Fluxo

Boot → Loader → Script → Config → Firewall → Wallpaper

---

## 🧪 PASSO 7 — Testes

```bash
curl https://maratona.td.utfpr.edu.br
```

```bash
curl https://google.com
```

---

## 📜 Logs

```bash
cat /var/log/maratona.log
```

---

## 🏁 Resultado Final

- Todas máquinas iguais ✔️
- Atualização centralizada ✔️
- Bloqueio funcionando ✔️
- Wallpaper automático ✔️
