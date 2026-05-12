# HA Dashboards Design — Rekuperácia + Prehľad domov

**Dátum:** 2026-05-12  
**Stav:** Schválený

## Kontext

Existujúce dashboardy: Mapa, Test, Tepelné čerpadlo.  
Cieľ: Pridať dva nové — Rekuperácia (podobný štýl ako TČ) a Prehľad domov (pôdorys + kamery + sieť).

---

## Dashboard 1: Rekuperácia

**URL:** `/rekuperacia`  
**Lovelace storage:** `/config/.storage/lovelace.rekuperacia`  
**Layout:** `type: sections`, `max_columns: 4`

### Sekcia 1 — Prehľad

- Heading: "Rekuperácia Komfovent" + ikona `mdi:hvac`
- `tile` — `switch.rekuperacia_power` (ZAP/VYP)
- `tile` — `select.rekuperacia_operation_mode` (aktuálny mód)
- `tile` — `sensor.rekuperacia_active_alarms`
- `tile` — `binary_sensor.rekuperacia_status_alarm_fault`
- `tile` — `binary_sensor.rekuperacia_status_alarm_warning`
- `tile` — `sensor.rekuperacia_filter_clogging`
- `gauge` — `sensor.rekuperacia_heat_exchanger_efficiency` (0–100%, segmenty: <60% červená, 60–80% žltá, >80% zelená)

### Sekcia 2 — Vzduch a teploty

- `tile` — `sensor.rekuperacia_supply_temperature` (prívod vzduchu)
- `tile` — `sensor.rekuperacia_extract_temperature` (výfuk vzduchu)
- `tile` — `sensor.rekuperacia_outdoor_temperature` (vonkajší vzduch)
- `tile` — `sensor.rekuperacia_supply_flow` (prívod m³/h)
- `tile` — `sensor.rekuperacia_extract_flow` (výfuk m³/h)
- `tile` — `sensor.rekuperacia_heat_recovery`
- `tile` — `sensor.rekuperacia_supply_fan` + `sensor.rekuperacia_extract_fan`
- `history-graph` 24h: supply/extract/outdoor teploty + supply/extract flow

### Sekcia 3 — Energia

- `tile` — `sensor.rekuperacia_power_consumption`
- `tile` — `sensor.rekuperacia_specific_power_input`
- `tile` — `sensor.rekuperacia_heater_power`
- `tile` — `sensor.rekuperacia_electric_heater`
- `tile` — `sensor.rekuperacia_total_ahu_energy`
- `tile` — `sensor.rekuperacia_total_recovered_energy`
- `tile` — `sensor.rekuperacia_total_heater_energy`
- `history-graph` 7d: power_consumption + heater_power

### Sekcia 4 — Ovládanie a servis

- `entities` card "Hlavné nastavenia":
  - `switch.rekuperacia_power`
  - `select.rekuperacia_operation_mode`
  - `select.rekuperacia_scheduler_mode`
  - `switch.rekuperacia_eco_mode`
  - `switch.rekuperacia_auto_mode`
- `entities` card "Prevádzky":
  - `number.rekuperacia_boost_supply_flow` + `rekuperacia_boost_extract_flow`
  - `number.rekuperacia_away_supply_flow` + `rekuperacia_away_extract_flow`
  - `number.rekuperacia_kitchen_supply_flow` + `rekuperacia_kitchen_extract_flow`
  - `number.rekuperacia_kitchen_timer`
- `entities` card "Servis":
  - `button.rekuperacia_clean_filters_calibration`
  - `button.rekuperacia_clear_active_alarms`
  - `button.rekuperacia_set_system_time`
  - `sensor.rekuperacia_controller_firmware`
  - `datetime.rekuperacia_holidays_from` + `rekuperacia_holidays_until`

---

## Dashboard 2: Prehľad domov

**URL:** `/prehlad`  
**Lovelace storage:** `/config/.storage/lovelace.prehlad`  
**Layout:** `type: sections`, `max_columns: 4`

### SVG Pôdorys

- Súbor: `/home/user/homeassistant/config/www/podorys.svg`
- Dostupný na: `http://smart-home-pc:8123/local/podorys.svg`
- Zjednodušená kresba odvodená od `podorys.png` — hrubé steny, miestnosti pomenované, bez kót

**Rozloženie miestností (podľa archit. plánu):**
- Ľavý horný roh: Spálňa
- Ľavý stred: Chodba / Vchod (1.2), Kúpeľňa (1.3)
- Stred: Technická miestnosť (1.10)
- Ľavý dolný stred: Obývačka
- Pravý bok: ďalšie izby (2.8, 2.9)

### Sekcia 1 — Pôdorys domu (full-width)

Karta `picture-elements`:
- `image`: `/local/podorys.svg`
- Overlaye (pozície ako `top`/`left` % — laditeľné v HA code editore):

| Entity | Miesto na mape | Typ elementu |
|---|---|---|
| `sensor.teplota_teplota` | Obývačka | `state-badge` |
| `sensor.teplota_vlhkost` | Obývačka (vedľa) | `state-badge` |
| `sensor.teplota_2_teplota` | Obývačka | `state-badge` |
| `sensor.teplota_2_vlhkost` | Obývačka (vedľa) | `state-badge` |
| `binary_sensor.0xa4c1388ca7849bb9_contact` | Vchod | `state-icon` (dvere) |
| `binary_sensor.0xa4c138841a702c87_contact` | Terasa | `state-icon` (dvere) |
| `binary_sensor.0xa4c1385a3860ff48_water_leak` | Technická | `state-icon` (voda) |
| `binary_sensor.tepelne_cerpadlo_esp_heatpump_state` | Technická | `state-icon` (TČ) |
| `sensor.tepelne_cerpadlo_esp_dhw_temp` | Technická | `state-badge` (TÚV) |
| `switch.rekuperacia_power` | Technická | `state-icon` (ventilátor) |

### Sekcia 2 — Kamery (full-width, 2 stĺpce)

- `picture-entity` — `camera.jv_hd_stream`, názov "Juhovýchod"
- `picture-entity` — `camera.sz_hd_stream`, názov "Severozápad"
- `camera_view: live`, refresh každých 10s

### Sekcia 3 — Sieť (full-width)

- `tile` — `binary_sensor.archer_ax17_stav_wan` (WAN)
- `tile` — `sensor.archer_ax17_externa_ip`
- `tile` — `sensor.archer_ax17_rychlost_stahovania`
- `tile` — `sensor.archer_ax17_rychlost_odosielania`
- `history-graph` 1h: sťahovanie + odosielanie

---

## Implementačné poznámky

- SVG pôdorys vytvorí Claude na základe `podorys.png` — zjednodušené, laditeľné
- Pozície overlayov na mape sú `top`/`left` % hodnoty v YAML → ručná úprava v HA code editore
- Lovelace storage súbory uložené priamo na serveri (rovnaký prístup ako `lovelace.tepelne-cerpadlo`)
- Dashboardy zaregistrovať v `lovelace_dashboards` storage
