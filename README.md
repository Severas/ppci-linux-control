# 🏁 PPCI Linux Lite — Complete Tutorial

## 🎯 Objective

Create a system where the computer:

- Boots up
- Downloads rules from a server
- Blocks internet access
- Applies configurations automatically

---

## 🧠 Simple Explanation

- Server = teacher (sends the rules)
- Computer = student or CLIENT (receives the rules)
- Script = brain
- Config = rules

---

## 📦 STEP 0 — Dependencies (CLIENT)

Before anything else, install the required dependencies:

```
sudo apt update
sudo apt install -y curl iptables iptables-legacy pcmanfm-qt lxqt-core network-manager
```

Make this important adjustment:

```
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
```

---

## 📁 STEP 1 — Server

Create the structure:

```
https://maratona.td.utfpr.edu.br/ppci-linux-lite/
```

Required files:

- `bloqueio.sh`
- `maratona.conf`
- `wallpaper_ppci/wallpaper.png`

---

## 🧾 STEP 2 — Configuration File

Create `maratona.conf`:

```
IPS="200.134.10.20"
PORTA=443
WALLPAPER_URL="https://maratona.td.utfpr.edu.br/ppci-linux-lite/wallpaper_ppci/wallpaper.png"
VERSION=1
```

---

## ⚙️ STEP 3 — Main Script (bloqueio.sh)

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

log "Downloading configuration..."

if curl -fsSL "$URL_CONF" -o "$TMP_CONF"; then
    cp "$TMP_CONF" "$CACHE_CONF"
    log "Config updated"
else
    log "Download error, using cache"
    if [ -f "$CACHE_CONF" ]; then
        cp "$CACHE_CONF" "$TMP_CONF"
    else
        log "No config available!"
        exit 1
    fi
fi

source "$TMP_CONF"

if [ -z "$IPS" ] || [ -z "$PORTA" ]; then
    log "Invalid variables"
    exit 1
fi

log "Applying firewall..."

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

log "Firewall applied"

# Wallpaper (for LXQt)

if [ ! -z "$WALLPAPER_URL" ]; then
    TMP_WALL="/tmp/wallpaper.png"
    DEST="/usr/share/backgrounds/ppci/wallpaper.png"

    if curl -fsSL "$WALLPAPER_URL" -o "$TMP_WALL"; then
        mkdir -p /usr/share/backgrounds/ppci
        cp "$TMP_WALL" "$DEST"
        log "Wallpaper downloaded"

        (
        sleep 5

        USER_REAL=$(logname 2>/dev/null || echo $SUDO_USER)

        if [ ! -z "$USER_REAL" ]; then
            su - $USER_REAL -c "DISPLAY=:0 pcmanfm-qt --set-wallpaper=$DEST"
            log "Wallpaper applied (LXQt)"
        else
            log "User not detected"
        fi
        ) &

    else
        log "Wallpaper error"
    fi
fi
```

---

## ⚙️ STEP 4 — Loader (CLIENT)

Create:

```
sudo nano /usr/local/bin/maratona-loader.sh
```

Paste:

```bash
#!/bin/bash

URL="https://maratona.td.utfpr.edu.br/ppci-linux-lite/bloqueio.sh"
TMP="/tmp/bloqueio.sh"
LOCAL="/usr/local/bin/bloqueio-local.sh"
LOG="/var/log/maratona.log"

log(){
    echo "[ $(date) ] $1" | tee -a $LOG
}

log "Downloading script..."

if curl -fsSL "$URL" -o "$TMP"; then
    chmod +x "$TMP"
    cp "$TMP" "$LOCAL"
    log "Running remote"
    bash "$TMP"
else
    log "Download error, using local"
    if [ -f "$LOCAL" ]; then
        bash "$LOCAL"
    else
        log "No fallback!"
        exit 1
    fi
fi
```

Permission:

```
sudo chmod +x /usr/local/bin/maratona-loader.sh
```

---

## ⚙️ STEP 5 — systemd (CLIENT)

```
sudo nano /etc/systemd/system/maratona.service
```

Content:

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

Enable:

```
sudo systemctl daemon-reload
sudo systemctl enable maratona.service
```

---

## 🚀 STEP 6 — Flow

Boot → Loader → Script → Config → Firewall → Wallpaper

---

## 🧪 STEP 7 — Tests

```
curl https://maratona.td.utfpr.edu.br
```

```
curl https://google.com
```

---

## 📜 Logs

```
cat /var/log/maratona.log
```

---

## 🏁 Final Result

- All machines identical ✔️
- Centralized updates ✔️
- Blocking working ✔️
- Automatic wallpaper ✔️
