# Homelab projekt — kontext pre Claude Code

> Referenčný dokument pre Claude Code pri budovaní a údržbe homelabu.
> Ulož ako `CLAUDE.md` v koreni repa — Claude Code ho číta automaticky.

## 1. Hardvér a OS

- **Host**: MacBook Air 2013 (Haswell, Intel HD 5000, 4–8 GB RAM)
- **OS**: Linux Mint
- **Sieť**: USB-Ethernet → router (statická IP / DHCP rezervácia)
- **VPN overlay**: Tailscale (nainštalované, free Personal plán)
- **Klienti**: Android TV box, HY300 projektor (Android 9/10), iPhone/Android mobily
- **Plánovaný disk**: USB 3.0 HDD 4–8 TB pre `/mnt/media`
- **Pripojený disk**: ADATA SU800 256 GB SSD → `/mnt/immich` (btrfs, Immich library)

## 2. Hardvérové obmedzenia (KRITICKÉ)

- GPU (HD 5000): iba H.264 8-bit hardvérový encode. ŽIADNE HEVC/AV1/10-bit/tone-mapping/Dolby Vision.
- CPU (ULV Haswell): nezvládne softvérový transcoding nad 1080p.
- **Doktrína: direct-play only**. Médiá prekódovať raz cez HandBrake na inom stroji.
- **Cieľový formát**: H.264 High Profile L4.1 8-bit + AAC stereo + MP4/MKV + externé `.srt`.
- **Vyhnúť sa**: 4K obsah, JVM appky, Nextcloud, GitLab, Elasticsearch.

## 3. Tech stack — finálne rozhodnutia

| Vrstva | Voľba | Poznámka |
|---|---|---|
| Container runtime | Docker + Compose | jeden compose súbor na službu |
| Compose UI | Dockge | bez lock-inu, číta súbory z disku |
| CLI helper | lazydocker | rýchla inšpekcia |
| Ad blocking | AdGuard Home | `network_mode: host` |
| Torrent | qBittorrent + VueTorrent | cez Gluetun namespace |
| Indexery | Prowlarr | NIE Jackett |
| Automatizácia | Sonarr + Radarr + Bazarr | NIE Readarr/Lidarr |
| VPN | ProtonVPN (WireGuard) | NAT-PMP port forwarding |
| VPN container | Gluetun | `VPN_PORT_FORWARDING=on` |
| Media server | Jellyfin | s VAAPI |
| Fotky | Immich v2.x | ML voliteľné podľa RAM |
| Úložisko (Drive) | Seafile CE 12.x | náhrada Google Drive, dokumenty/general files |
| Vzdialený prístup | Tailscale Serve | žiadny reverse proxy |
| Monitoring | Beszel | ~50 MB RAM |
| Uptime monitoring | Uptime Kuma | externý — Hetzner CX11 |
| Dashboard | Homepage (gethomepage) | YAML config |
| Backup | Restic → Hetzner Storage Box | 3,49 €/mes. za 1 TB |
| Reverse proxy | NEPOUŽÍVAŤ | Tailscale Serve to rieši |
| All-in-one panel | NEPOUŽÍVAŤ | žiadne CasaOS/Umbrel |
| Auto-update | NEPOUŽÍVAŤ | iba notify mode (Diun) |
| FS na externom HDD | ext4 | NIE btrfs |
| FS na Immich SSD | btrfs | bitrot ochrana pre fotky/videá |

## 4. Štruktúra priečinkov

```
/opt/stacks/                    # compose + appdata (git repo)
├── adguardhome/
│   ├── compose.yaml
│   ├── .env                    # gitignored
│   └── .env.example
├── arr/                        # Sonarr+Radarr+Prowlarr+Bazarr+qBit+Gluetun
├── jellyfin/
├── immich/
├── homeassistant/              # vlastný update cyklus
├── monitoring/                 # Beszel + Homepage
├── seafile/                    # Seafile CE 12.x (Drive replacement)
└── dockge/

/mnt/immich/                    # ADATA SU800 256 GB SSD, btrfs, label: immich
├── library/                    # Immich UPLOAD_LOCATION
└── seafile/                    # Seafile data (/shared bind mount)

/mnt/media/                     # USB 3.0 HDD, ext4 (plánovaný)
└── data/
    ├── torrents/{movies,tv}
    └── media/{movies,tv}

/var/lib/docker/                # Docker volumes na internom SSD
```

**KRITICKÉ pre *arr stack**: jeden mount `/mnt/media/data:/data` do KAŽDÉHO kontajnera. Nikdy nemapovať `/downloads` a `/movies` separately — rozbije hardlinky a atomic moves.

## 5. Sieťová topológia

```
Router (DHCP, statická IP pre MacBook 192.168.1.53)
   ↓ DNS pointing na 192.168.1.53
MacBook
   ├─ AdGuard Home (host network, port 53)
   ├─ Tailscale (overlay, MagicDNS, HTTPS certs)
   ├─ Docker bridge networks (homelab + per-stack default)
   └─ Gluetun namespace (qBit zdieľa)

Tailnet:
   - MacBook = global nameserver → AGH blokuje reklamy aj na mobile vonku
   - Tailscale Serve vystaví Jellyfin/Immich na *.ts.net (Let's Encrypt)
```

## 6. Build order (8 krokov)

1. **Harden host**: SSH key-only, `ufw allow in on tailscale0`, `ufw allow 22/tcp`, `ufw enable`. Vypnúť password auth.
2. **Docker + Dockge**: nainštaluj Docker Engine (NIE Desktop), spusti Dockge → zvyšok riaď z prehliadača.
3. **AdGuard Home**: vyrieš `systemd-resolved` konflikt (sekcia 7), nasaď, zmeň router DNS, otestuj na jednom zariadení predtým než prepneš celú sieť.
4. **Jellyfin**: VAAPI na `/dev/dri/renderD128`, vypni HEVC/AV1/VP9 encodery, weekly restart cron (memory leak #12838).
5. **arr stack**: Gluetun + ProtonVPN WireGuard, qBit s `network_mode: "service:gluetun"`. Sonarr/Radarr/Prowlarr/Bazarr na default bridge. Konfig podľa TRaSH Guides.
6. **Immich**: ak 4 GB → zakomentuj `immich-machine-learning`. Inak `MACHINE_LEARNING_MAX_BATCH_SIZE__FACIAL_RECOGNITION=1`, `concurrency=1`.
7. **Monitoring**: Beszel + Homepage lokálne, Uptime Kuma na Hetzner CX11 VPS (Falkenstein).
8. **Backup** ✓: Restic → Hetzner Storage Box BX11 (`u591423.your-storagebox.de`), nočný cron 02:30, štvrťročný restore test.

## 7. Kritické gotchas

### Linux Mint + AGH: konflikt s systemd-resolved
`systemd-resolved` drží `127.0.0.53:53`. **NEvypínaj službu** (Docker ju potrebuje). Riešenie:
```ini
# /etc/systemd/resolved.conf.d/adguardhome.conf
[Resolve]
DNSStubListener=no
DNS=1.1.1.1 9.9.9.9
```
Potom `ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf` a restart.

### DNS: nikdy sekundárna na 1.1.1.1
OS-y race-ujú oba servery a obídu blokovanie. Buď jeden (AGH), alebo dva oba na AGH.

### qBittorrent v Gluetun namespace
V Sonarr/Radarr Download Client config hostname **`gluetun`** (NIE `qbittorrent`) — qBit nemá vlastnú IP.

### *arr volumes — single mount
Mapuj `/data` (NIE `/downloads` + `/movies`). Inak prídeš o hardlinky.

### Jellyfin memory leak
Issue #12838: idle RAM rastie ~100 MB/h. Cron:
```
0 4 * * 0 docker restart jellyfin
```

### Tailscale exit node + DNS bug (issue #15999)
Exit node na iOS/macOS môže obísť custom DNS. Preto AGH ako tailnet **global nameserver** (admin → DNS → Override local DNS), NIE cez exit node.

### Slovenské banking appky + Hagezi PRO++/Ultimate
Tatra banka, VÚB, SLSP, mBank používajú bot-detection SDK. Stop pri **Multi PRO + OISD Big**.

### Btrfs na pomalom Haswelli
CoW + checksumming žerie CPU. ext4 na médiá. Btrfs len pre Immich partíciu (bitrot ochrana).

### Watchtower auto-update
Default mode rozbije Postgres migrácie. Použi **monitor-only** + Telegram notify, alebo Diun/WUD.

## 8. Rozpočet RAM (8 GB host)

| Komponent | RAM (typ.) |
|---|---|
| Mint base | 1,2 GB |
| Jellyfin | 0,5–1 GB |
| Immich (s ML) | 1,5–3 GB |
| Postgres + Redis | 0,5 GB |
| AdGuard Home | 0,1 GB |
| Home Assistant | 0,4 GB |
| arr stack (5 + Gluetun) | 1–1,5 GB |
| Dockge + Beszel + Homepage | 0,15 GB |
| **Spolu pri záťaži** | **6–7 GB** |

Tuning: swap 4 GB, `vm.swappiness=10`, Docker log rotation v `/etc/docker/daemon.json`:
```json
{"log-driver": "json-file", "log-opts": {"max-size": "10m", "max-file": "3"}}
```

**4 GB host**: zakomentuj `immich-machine-learning`. Ak nestačí → N100 mini-PC ~150 €.

## 9. Tailscale Serve

**Aktuálny stav (aktívne):**

| Port | Služba | Localhost |
|---|---|---|
| 443 | Seafile | `http://localhost:8000` |
| 8443 | Immich | `http://localhost:2283` |

```bash
# Nakonfigurovať (ak sa stratí po reinstalácii TS):
sudo tailscale serve --https 443  http://localhost:8000   # Seafile
sudo tailscale serve --https 8443 http://localhost:2283   # Immich

# Plánované (po nasadení):
sudo tailscale serve --https 9443 http://localhost:8096   # Jellyfin
```

**URLs:**
- Seafile: `https://smart-home-pc.taile97bf5.ts.net`
- Immich: `https://smart-home-pc.taile97bf5.ts.net:8443`

**Gotcha:** `tailscale serve` konfigurácia sa stratí pri reinstalácii TS. Obnoviť cez príkazy vyššie.

Reverse proxy pridať iba ak: smart TV bez TS klienta, path-based routing, public sharing → Caddy.

## 9b. Seafile

- **Compose:** `/opt/stacks/seafile/compose.yaml`
- **Web UI:** `https://smart-home-pc.taile97bf5.ts.net` (cez Tailscale)
- **Data:** `/mnt/immich/seafile/` (ADATA SU800 256 GB SSD, btrfs)
- **DB:** MariaDB 11.4 (volume `seafile-db`)
- **Port (local):** `127.0.0.1:8000` (nginx v kontajneri)

### Klienti
| Klient | Konfigurácia |
|---|---|
| iPhone (Seafile app) | Server: `https://smart-home-pc.taile97bf5.ts.net` — camera auto-upload |
| iOS Files.app (WebDAV) | `https://smart-home-pc.taile97bf5.ts.net/seafdav` — App Password (nie hlavné heslo!) |
| Web | `https://smart-home-pc.taile97bf5.ts.net` |
| Windows (WebDAV) | Map network drive: `https://smart-home-pc.taile97bf5.ts.net/seafdav` — App Password |
| Windows (SeaDrive 3) | Alternatíva — ale mountuje do `S:\My Libraries\`, nie flat root |

### Prvé spustenie
```bash
cd /opt/stacks/seafile
cp .env.example .env  # vyplniť silné heslá (bez # znakov!)
docker compose up -d
# Počkať ~60s na init DB a migrácie
docker logs seafile -f
```

**Po prvom štarte overiť** `/shared/seafile/conf/seahub_settings.py`:
- `SERVICE_URL` a `FILE_SERVER_ROOT` musia byť `https://` — ak nie, opraviť a reštartovať.
- Default admin email je `me@example.com` ak `SEAFILE_ADMIN_EMAIL` nebolo správne načítané — opraviť v DB: `UPDATE EmailUser SET email='...' WHERE email='me@example.com';` v `ccnet_db`.

### Gotchas
- `SERVICE_URL` + `FILE_SERVER_ROOT` sa nastavujú cez `SEAFILE_SERVER_HOSTNAME` pri prvom štarte — **nezmeniť neskôr bez re-init**.
- Major upgrade (napr. 12 → 13): urobiť Restic backup pred upgadom mimo bežnej rotácie.
- Admin heslo z `.env` platí **iba pri prvom štarte**. Neskôr meniť vo web UI.
- iOS Files.app WebDAV: použiť **App Password** (vygenerovaný v Seafile web UI → Settings → App Password), nie hlavné heslo.
- `.env` heslá: **nepoužívať `#`** (Docker Compose to číta ako komentár). Citovať hodnoty s `"`.
- `JWT_PRIVATE_KEY` povinný v env od v12 — bez neho server nenaštartuje.
- `SERVICE_URL` + `FILE_SERVER_ROOT` musia byť `https://` (nie `http://`) — inak iOS app nevie sťahovať súbory. Súbor: `/shared/seafile/conf/seahub_settings.py`.
- Pri zabudnutom hesle: `printf "email\nnewpass\nnewpass\n" | ./reset-admin.sh` v kontajneri (`/opt/seafile/seafile-server-latest/`).
- Login attempt counter v Memcached — reset: `echo flush_all | nc localhost 11211` v `seafile-memcached` kontajneri.

### Operácie
```bash
# Logy
docker logs seafile -f --tail 50
docker logs seafile-db -f --tail 50

# Aktualizácia
cd /opt/stacks/seafile && docker compose pull && docker compose down && docker compose up -d

# Reštart
cd /opt/stacks/seafile && docker compose restart
```

### Backup (integrovaný do existujúceho Restic)
- MariaDB dump: automaticky v `/opt/stacks/backup/backup.sh` (ak `seafile-db` beží)
- Data: `/mnt/immich/seafile/` zahrnuté v restic backup cestách
- Dump súbory: `/opt/stacks/backup/dumps/seafile-YYYY-MM-DD.sql` (rotate 7 dní)

## 10. Home Assistant integrácia

- Vlastný compose file (`/opt/stacks/homeassistant/`) — nezlučovať.
- Zdieľaná sieť: `docker network create homelab`.
- Vstavané integrácie: **Jellyfin** (media player + remote), **Pi-hole** (funguje aj s AGH cez komunitnú integráciu).
- HACS: Uptime Kuma status sensor.
- Tailscale: NEintegruj.

## 11. Blocklisty (AdGuard Home)

```
https://big.oisd.nl/                                                                  # baseline
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.txt          # Threat Intel
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/popupads.txt     # pop-ups
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/fake.txt         # fake shops
```
Max: Hagezi Multi PRO. **NIKDY** PRO++/Ultimate.

## 12. Backup

**Provider:** Hetzner Storage Box BX11 — `u591423.your-storagebox.de:23` (SFTP, SSH key auth)
**Tool:** Restic 0.16.4, repozitár ID `30d967857f`
**Skript:** `/opt/stacks/backup/backup.sh`
**Cron:** `30 2 * * *` (root crontab) — lockfile `/var/lock/restic-backup.lock` (flock) bráni súbežným behom
**Log:** `/var/log/restic-backup.log`
**Credentials:** `/opt/stacks/backup/.env` + `/opt/stacks/backup/.restic-password`

### Čo sa zálohuje
- `/opt/stacks/` — compose súbory + appdata
- `/home/user/homeassistant/` — HA config, Z2M, Mosquitto, ESPHome (bez `.esphome` build cache)
- `/home/user/stacks/` — Beszel DB + Homepage config
- `/home/user/smart/` — Smart PWA zdrojový kód (bez `node_modules`)
- `/home/user/.ssh/` — SSH kľúče (potrebné po restore pre prístup na Storage Box)
- `/mnt/immich/library/` — Immich fotky/videá
- `/mnt/immich/seafile/` — Seafile data (block storage)
- `/var/lib/docker/volumes/smart_smart-data/_data` — Smart PWA dáta
- `/var/lib/docker/volumes/docker_smart-data/_data` — Smart PWA SQLite DB
- `pg_dump immich` → `/opt/stacks/backup/dumps/immich-YYYY-MM-DD.sql` (pred restic, rotate 7 dní)
- `mariadb-dump seafile-db` → `/opt/stacks/backup/dumps/seafile-YYYY-MM-DD.sql` (pred restic, rotate 7 dní)
- System config snapshot → `/opt/stacks/backup/dumps/sysconfig/` (DNS, Docker, crontab, UFW)

NEZÁLOHUJ médiá (cez *arr re-downloadable).

### Retention
7 denných / 4 týždenných / 6 mesačných snapshotov, automatický prune.

### Restore
Návod: `/opt/stacks/backup/RESTORE.md` (zálohovaný spolu so stackom)

Štvrťročný test: `restic restore latest --target /tmp/restore-test`

## 13. Monitoring a alerting

### Beszel
- UI: `http://smart-home-pc:8090` — grafy CPU, RAM, disk, Docker kontajnery
- Agent sleduje `/mnt/immich` cez `EXTRA_FILESYSTEMS=/mnt/immich`
- Beszel alerting **nepoužívame** — nahradený vlastným skriptom (viď nižšie)

### Homepage sekcie a služby
| Sekcia | Služba | URL |
|---|---|---|
| Home | Home Assistant | `http://smart-home-pc:8123` |
| Smart Home | Zigbee2MQTT | `http://smart-home-pc:8080` |
| Smart Home | ESPHome | `http://smart-home-pc:6052` |
| Monitoring | Beszel | `http://smart-home-pc:8090` |
| Monitoring | Dockge | `http://smart-home-pc:5001` |
| Network | AdGuard Home | `http://smart-home-pc:3002` |
| Media | Immich | `http://smart-home-pc:2283` |
| Files | Seafile | `https://smart-home-pc.taile97bf5.ts.net` |

### Homepage disk widgety
- `/` — hlavný disk
- `/mnt/immich` — Immich SSD

### Homepage info widgety (hlavička)
- **CPU + RAM + disk `/` + sieť** — `network: enxfce498e3f33d` (USB Ethernet adaptér)
- **Immich SSD** — disk `/mnt/immich`
- **Datum a čas** — `locale: sk-SK`, formát DD.MM.YYYY, 24h

Config: `/home/user/stacks/monitoring/homepage/config/widgets.yaml`

### Health check skript
**Skript:** `/opt/stacks/backup/check-health.sh`
**Cron:** `0 * * * *` (každú hodinu, root crontab)
**Log:** `/var/log/check-health.log`

Kontroluje:
| Check | Prah | Akcia |
|---|---|---|
| Disk `/` | > 85% | HA webhook → iPhone notifikácia |
| Disk `/mnt/immich` | > 85% | HA webhook → iPhone notifikácia |
| RAM | > 90% | HA webhook → iPhone notifikácia |
| SMART `/dev/sda` (Apple SSD, systém) | FAILED | HA webhook → iPhone notifikácia |
| SMART `/dev/sdc` (ADATA SU800, Immich) | FAILED | HA webhook → iPhone notifikácia |

**HA automation:** `beszel_alert_webhook` — webhook ID `beszel_alert`, posiela na `notify.mobile_app_iphone_uzivatela_david`

## 14. Pravidlá pre Claude Code

- Pred zmenami v compose: over `.env.example` existuje + `.env` v `.gitignore`.
- Pred `docker compose down` na live stacku: navrhni zálohu volumes.
- Nikdy `WATCHTOWER_CLEANUP=true` na databázových kontajneroch.
- Pri novom servise: zváž `tailscale serve` namiesto port forwardu na router.
- *arr download client config: hostname `gluetun`, port qBitu, NIE `qbittorrent`.
- Jellyfin profily: H.264 8-bit AAC primárny target, transcoding last resort.
- Immich upgrade: najprv `pg_dump`, potom `docker compose pull && up -d`. v2.x dodržuje semver.
- Slovenský VAT: Hetzner/Tailscale → reverse charge, VIES check OK.

## 15. Migrácia keď MacBook umrie

1. N100 mini-PC (~150 €) + Debian/Ubuntu Server.
2. `git clone` repo s `/opt/stacks`.
3. Restore Docker volumes z Restic.
4. Tailscale install + login (rovnaký hostname).
5. `docker compose up -d` per stack.
6. Refresh router DNS (nová IP).

Cieľ: < 1 hodina migrácia.

---

**Last updated**: 2026-05-10 (Seafile nasadený a funkčný)