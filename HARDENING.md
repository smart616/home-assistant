# Host Hardening — smart-home-pc

**OS:** Linux Mint 22.3 (kernel 6.14)  
**Dátum:** 2026-05-08  
**Rozsah:** SSH, UFW, Docker daemon

---

## 1. SSH

Súbor: `/etc/ssh/sshd_config.d/hardening.conf`

```ini
PasswordAuthentication no
X11Forwarding no
```

Efektívna konfigurácia po zmene:

| Parameter | Hodnota |
|---|---|
| `PasswordAuthentication` | `no` |
| `PubkeyAuthentication` | `yes` |
| `X11Forwarding` | `no` |
| `PermitRootLogin` | `without-password` |

Overenie:
```bash
sudo sshd -T | grep -E '^(passwordauthentication|x11forwarding|pubkeyauthentication|permitrootlogin)'
```

---

## 2. UFW (firewall)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow in on tailscale0
echo 'y' | sudo ufw enable
```

Aktívne pravidlá:

```
Default: deny (incoming), allow (outgoing)

22/tcp          ALLOW IN    Anywhere
tailscale0      ALLOW IN    Anywhere
```

Všetky služby (HA port 8123, ESPHome 6052, MQTT 1883, Z2M 8080, Beszel 8090) sú dostupné **iba cez Tailscale** (`tailscale0` interface).

Overenie:
```bash
sudo ufw status verbose
```

---

## 3. Docker daemon

Súbor: `/etc/docker/daemon.json`

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false
}
```

Zmeny sa aplikujú po `sudo systemctl restart docker` — kontajnery s `restart: unless-stopped` sa spustia automaticky, ostatné treba spustiť manuálne.

---

## Budúci hardening (nie je implementovaný)

- `fail2ban` — brute-force ochrana SSH
- `unattended-upgrades` — automatické bezpečnostné aktualizácie
- `sysctl` — kernel network hardening (IP spoofing, ICMP redirect)
- Docker: `no-new-privileges` per kontajner
