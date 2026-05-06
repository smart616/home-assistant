# Home Assistant Project

Dokumentácia, konfigurácia a údržba Home Assistant na serveri `smart-home-pc`.

## Server

- **Host:** `smart-home-pc` (SSH, key auth, user: `user`)
- **OS:** Ubuntu 24.04, x86_64
- **Stack dir:** `/home/user/homeassistant/`

## Stack (Docker Compose)

Súbor: `/home/user/homeassistant/docker-compose.yml`

| Kontajner | Image | Sieť | Port |
|---|---|---|---|
| `homeassistant` | `ghcr.io/home-assistant/home-assistant:stable` | host | 8123 |
| `mosquitto` | `eclipse-mosquitto:latest` | host | 1883 |
| `zigbee2mqtt` | `koenkk/zigbee2mqtt:latest` | host | 8080 |

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

### Mosquitto (MQTT broker)
- Config: `/home/user/homeassistant/mosquitto/config/mosquitto.conf`
- Anonymous auth povolený
- Persistence: `/mosquitto/data/`

## Zigbee zariadenia

| IEEE | Friendly name | Typ |
|---|---|---|
| `0xa4c138841a702c87` | Dvere terasa | Kontaktný senzor (dvere/okno) |
| `0xa4c1388ca7849bb9` | Dvere vchod | Kontaktný senzor (dvere/okno) |
| `0xa4c1385a3860ff48` | Senzor vody | Senzor vody |

Kontaktné senzory: `binary_sensor.<entity>_contact`, `on` = otvorené.

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

## Timezone

Server a HA kontajnery: `Europe/Berlin` (= Europe/Prague/Bratislava, UTC+1/+2)
