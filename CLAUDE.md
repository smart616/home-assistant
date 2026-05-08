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

## Timezone

Server a HA kontajnery: `Europe/Berlin` (= Europe/Prague/Bratislava, UTC+1/+2)
