# Home Assistant Project

Dokumentácia, konfigurácia a údržba Home Assistant na serveri `smart-home-pc`.

## Server

- **Host:** `smart-home-pc` (SSH, key auth, user: `user`)
- **Hardvér:** MacBook Air 2013 (Haswell, Intel HD 5000) — detaily v `HOMELAB.md`
- **OS:** Linux Mint 22.3, x86_64
- **LAN IP:** `192.168.0.219`
- **Stack dir:** `/home/user/homeassistant/`

## Stack (Docker Compose)

Súbor: `/home/user/homeassistant/docker-compose.yml`

| Kontajner | Image | Sieť | Port |
|---|---|---|---|
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:stable` | host | 8123 |
| `mosquitto` | `eclipse-mosquitto:latest` | host | 1883 |
| `zigbee2mqtt` | `koenkk/zigbee2mqtt:latest` | host | 8080 |
| `esphome` | `ghcr.io/esphome/esphome:stable` | host | 6052 |

Všetky kontajnery bežia s `network_mode: host`.

## Základné operácie

```bash
# Pripojenie na server
ssh smart-home-pc

# Restart všetkých služieb
cd /home/user/homeassistant && docker compose restart

# Restart len HA
docker restart homeassistant

# Logy
docker logs homeassistant -f --tail 50
docker logs zigbee2mqtt -f --tail 50
docker logs mosquitto -f --tail 50

# Aktualizácia na novú verziu HA
cd /home/user/homeassistant && docker compose pull && docker compose up -d
```

## Konfigurácia

### Home Assistant
- Config dir: `/home/user/homeassistant/config/`
- `configuration.yaml` – hlavný config (default_config, includes)
- `automations.yaml` – automatizácie
- `scripts.yaml` – skripty
- `scenes.yaml` – scény
- `input_numbers.yaml` – vstupné čísla (nastavenia)
- `secrets.yaml` – tajné hodnoty (API kľúče, tokeny)

### Zigbee2MQTT
- Config: `/home/user/homeassistant/zigbee2mqtt/data/configuration.yaml`
- Adapter: `/dev/ttyACM0` (zstack)
- MQTT broker: `mqtt://localhost:1883`
- Web UI: `http://smart-home-pc:8080`
- Base topic: `zigbee2mqtt`

### ESPHome
- Config dir: `/home/user/homeassistant/esphome/`
- Web UI: `http://smart-home-pc:6052`

### Mosquitto (MQTT broker)
- Config: `/home/user/homeassistant/mosquitto/config/mosquitto.conf`
- Listener: `0.0.0.0:1883`
- Anonymous auth povolený (`allow_anonymous true`)
- Persistence: `./mosquitto/data/`
- Logy: `./mosquitto/log/mosquitto.log`
- Používa: Zigbee2MQTT (depends_on), prípadné MQTT zariadenia

### ESPHome
- Config dir: `/home/user/homeassistant/esphome/`
- Web UI: `http://smart-home-pc:6052`
- Spúšťa sa s `privileged: true`
- Zariadenia:
  - `esphome-web-45da1c.yaml` → Panasonic Aquarea (ESP32, IP: 192.168.0.158)

## Zigbee zariadenia

| IEEE | Friendly name | Typ |
|---|---|---|
| `0xa4c138841a702c87` | Dvere terasa | Kontaktný senzor (dvere/okno) |
| `0xa4c1388ca7849bb9` | Dvere vchod | Kontaktný senzor (dvere/okno) |
| `0xa4c1385a3860ff48` | Senzor vody | Senzor vody |

Kontaktné senzory: `binary_sensor.<entity>_contact`, `on` = otvorené.

## Tepelné čerpadlo – Panasonic Aquarea

- **Model:** WH-SDC09H3E8 (IDU) / WH-UD09HE8 (ODU), 9kW
- **Integrácia:** ESP32 (ESPHome) → HA Native API (nie MQTT)
  - ESPHome komponent: `panasonic_heatpump` od ElVit (`github://ElVit/esphome_components`)
  - UART pripojenie na CN-CNT port TČ: GPIO16 (RX) / GPIO17 (TX), 9600 baud, EVEN parity
  - ESP32 IP: `192.168.0.158`, device name: `panasonic-heatpump`
  - ESPHome config: `/home/user/homeassistant/esphome/esphome-web-45da1c.yaml`
- **Entity prefix:** `tepelne_cerpadlo_esp_`
- **Dashboard:** `http://smart-home-pc:8123/tepelne-cerpadlo` (Lovelace storage: `/config/.storage/lovelace.tepelne-cerpadlo`)
- **Konfig zóny:** Len Zóna 1 aktívna, Zóna 2 nie je zapojená, `cool_mode: false`
- **Nainštalované:** DHW (TÚV), bez buffera, bez bazéna, bez solárneho kolektora
- **Bez pulzného merača:** `_extra` energy senzory sú `unknown`

### Kľúčové entity

| Entity | Popis |
|---|---|
| `switch.tepelne_cerpadlo_esp_set_heatpump` | Hlavný ZAP/VYP |
| `select.tepelne_cerpadlo_esp_set_operation_mode` | Režim (HEAT, TANK, HEAT+TANK, COOL, AUTO...) |
| `sensor.tepelne_cerpadlo_esp_outside_temp` | Vonkajšia teplota |
| `sensor.tepelne_cerpadlo_esp_main_inlet_temp` | Prívod vody |
| `sensor.tepelne_cerpadlo_esp_main_outlet_temp` | Spiatočka vody |
| `sensor.tepelne_cerpadlo_esp_dhw_temp` | TÚV aktuálna teplota |
| `number.tepelne_cerpadlo_esp_set_dhw_temp` | TÚV nastavená teplota |
| `sensor.tepelne_cerpadlo_esp_compressor_freq` | Frekvencia kompresora (Hz) |
| `sensor.tepelne_cerpadlo_esp_pump_flow` | Prietok čerpadla (l/min) |
| `sensor.tepelne_cerpadlo_esp_heat_power_consumption` | Energia vykurovanie – spotreba |
| `sensor.tepelne_cerpadlo_esp_heat_power_production` | Energia vykurovanie – výroba |

## Custom integrations (HACS)

Uložené v `/home/user/homeassistant/config/custom_components/`:

| Integration | Účel |
|---|---|
| `hacs` | Home Assistant Community Store |
| `komfovent` | Riadenie vetráceho/klimatizačného systému Komfovent |
| `localtuya` | Lokálne ovládanie Tuya zariadení (bez cloudu) |
| `tapo_control` | TP-Link Tapo smart zariadenia |

## Automatizácie

| ID | Alias | Popis |
|---|---|---|
| `terasa_dvere_otvorene_dlho` | Terasa - dvere otvorené dlho | Notifikácia na iPhone (David) keď sú dvere terasy otvorené dlhšie ako `input_number.dvere_terasa_notify_delay` minút |

## Input Numbers (nastavenia)

| Entity | Názov | Rozsah | Default |
|---|---|---|---|
| `input_number.dvere_terasa_notify_delay` | Terasa - oneskorenie notifikácie | 1-120 min | 1 min |

## Mobile notifications

Cieľ: `notify.mobile_app_iphone_uzivatela_david` (David's iPhone cez HA Companion app)

## Súvisiaci projekt

`/home/user/smart/` – samostatná PWA (Node.js/Express, port 3000) – osobný systém (Planner, kanban, atď.). Nie je súčasťou HA, ale beží na tom istom serveri.

## Monitoring

### Beszel
- Config dir: `/home/user/stacks/monitoring/`
- Compose: `/home/user/stacks/monitoring/compose.yaml`
- Hub: `http://smart-home-pc:8090` (`network_mode: host`)
- Agent: `localhost:45876` (`network_mode: host`, sleduje Docker + systém)
- Key: v `/home/user/stacks/monitoring/.env` ako `BESZEL_KEY`

```bash
# Reštart
cd /home/user/stacks/monitoring && docker compose restart

# Logy
docker logs beszel -f --tail 50
docker logs beszel-agent -f --tail 50

# Aktualizácia
cd /home/user/stacks/monitoring && docker compose pull && docker compose up -d
```

**Gotcha:** Oba kontajnery musia byť `network_mode: host` — inak hub nedosiahne agenta cez `localhost:45876`.

## AdGuard Home

- **Compose:** `/opt/stacks/adguardhome/compose.yaml`
- **Web UI:** `http://smart-home-pc:3002`
- **DNS port:** 53 (host network)
- **Data:** `/opt/stacks/adguardhome/workdir` + `confdir`
- **Router DNS:** `192.168.0.219` (primárny, žiadny sekundárny — inak zariadenia obídu AGH)
- **Upstream DNS:** `https://family.cloudflare-dns.com/dns-query` (blokuje malware + dospelý obsah)

### UFW porty (potrebné pre DNS)
```bash
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
```

### Blocklisty
```
https://big.oisd.nl/
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/popupads.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/fake.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/nsfw.txt
https://nsfw.oisd.nl/domainswild2
```

**Max list:** Hagezi Multi PRO. **NIKDY** PRO++/Ultimate — rozbije slovenské bankové appky.

### Operácie
```bash
cd /opt/stacks/adguardhome && docker compose restart
docker logs adguardhome -f --tail 50
cd /opt/stacks/adguardhome && docker compose pull && docker compose up -d
```

### Gotcha: systemd-resolved konflikt
`systemd-resolved` drží `127.0.0.53:53` štandardne. Fix v `/etc/systemd/resolved.conf.d/adguardhome.conf`:
```ini
[Resolve]
DNSStubListener=no
DNS=1.1.1.1 9.9.9.9
```
+ `sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf` + restart resolved.

## Immich

- **Compose:** `/opt/stacks/immich/compose.yaml`
- **Web UI:** `http://smart-home-pc:2283`
- **Upload dir:** `/opt/stacks/immich/library/` (presun na `/mnt/media/immich/library/` keď príde USB HDD)
- **External library:** `/home/user/Photos` → `/usr/src/app/external` (read-only mount, import manuálne)
- **ML:** vypnuté (3.8 GB RAM) — odkomentovať `immich-machine-learning` v compose.yaml keď príde väčší stroj
- **Verzia:** Immich v2.7.5

```bash
# Logy
docker logs immich-server -f --tail 50

# Aktualizácia (pg_dump PRED pulom!)
docker exec immich-postgres pg_dump -U immich immich > /opt/stacks/immich/backup-$(date +%F).sql
cd /opt/stacks/immich && docker compose pull && docker compose up -d
```

### Gotcha: DB_HOSTNAME
Immich defaultne hľadá postgres na hostnamu `database`. Keďže naša služba sa volá `immich-postgres`, treba `DB_HOSTNAME=immich-postgres` v `.env`.

### Presun na USB HDD (keď príde)
1. `docker compose stop immich-server`
2. `rsync -av /opt/stacks/immich/library/ /mnt/media/immich/library/`
3. Zmeň `.env`: `UPLOAD_LOCATION=/mnt/media/immich/library`
4. `docker compose up -d`

## Timezone

Server a HA kontajnery: `Europe/Berlin` (= Europe/Prague/Bratislava, UTC+1/+2)
