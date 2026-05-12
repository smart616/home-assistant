# HA Dashboards — Rekuperácia + Prehľad domov Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Vytvoriť dva nové HA dashboardy — Rekuperácia (monitoring+ovládanie Komfovent) a Prehľad domov (SVG pôdorys so senzormi, kamery, sieť).

**Architecture:** Lovelace storage dashboardy zapísané priamo ako JSON do `/home/user/homeassistant/config/.storage/` na serveri `smart-home-pc` (SSH, key auth). SVG pôdorys sa uloží do `/home/user/homeassistant/config/www/` a slúži ako pozadie `picture-elements` karty. Dashboardy sa zaregistrujú v `lovelace_dashboards` storage. Jeden `docker restart homeassistant` na záver aktivuje všetky zmeny.

**Tech Stack:** SSH, Python (JSON editácia lovelace_dashboards), HA Lovelace storage JSON, SVG

---

## Súbory

| Súbor | Akcia |
|---|---|
| `/home/user/homeassistant/config/www/podorys.svg` | Vytvoriť — zjednodušený SVG pôdorys domu |
| `/home/user/homeassistant/config/.storage/lovelace.rekuperacia` | Vytvoriť — Rekuperácia dashboard JSON |
| `/home/user/homeassistant/config/.storage/lovelace.prehlad` | Vytvoriť — Prehľad domov dashboard JSON |
| `/home/user/homeassistant/config/.storage/lovelace_dashboards` | Upraviť — pridať záznamy pre oba dashboardy |
| `www/podorys.svg` | Vytvoriť v repo (kópia pre git) |

---

## Task 1: SVG pôdorys — vytvoriť a nahrať

**Files:**
- Create: `www/podorys.svg` (v repo aj na serveri)

- [ ] **Krok 1: Vytvoriť www/ adresár v repo**

```bash
mkdir -p /home/user/projects/home-assistant/www
```

- [ ] **Krok 2: Zapísať SVG pôdorys**

Uložiť do `/home/user/projects/home-assistant/www/podorys.svg`:

```svg
<svg viewBox="0 0 920 420" xmlns="http://www.w3.org/2000/svg">
  <defs>
    <style>
      .wall-ext { fill: none; stroke: #2c3e50; stroke-width: 8; stroke-linejoin: round; }
      .wall-int { fill: none; stroke: #2c3e50; stroke-width: 3; stroke-linejoin: round; }
      .room-fill { fill: #ecf0f1; }
      .ext-fill  { fill: #dfe6e9; }
      .label     { font-family: Arial, sans-serif; fill: #2c3e50; text-anchor: middle; dominant-baseline: middle; }
      .label-lg  { font-size: 15px; font-weight: bold; }
      .label-sm  { font-size: 11px; }
      .door-arc  { fill: none; stroke: #7f8c8d; stroke-width: 1.5; stroke-dasharray: 4,2; }
      .door-line { fill: none; stroke: #7f8c8d; stroke-width: 1.5; }
    </style>
  </defs>

  <rect width="920" height="420" fill="#f0f2f5"/>

  <!-- Prístavby (terasy) -->
  <rect x="0" y="120" width="84" height="180" class="ext-fill wall-ext"/>
  <rect x="836" y="120" width="84" height="180" class="ext-fill wall-ext"/>

  <!-- Hlavné telo domu -->
  <rect x="84" y="40" width="752" height="340" class="room-fill wall-ext"/>

  <!-- Spálňa: pravá stena -->
  <line x1="300" y1="40" x2="300" y2="185" class="wall-int"/>
  <!-- Spálňa: dolná stena -->
  <line x1="84" y1="185" x2="300" y2="185" class="wall-int"/>

  <!-- Chodba: dolná stena / kúpeľňa: horná stena -->
  <line x1="84" y1="250" x2="210" y2="250" class="wall-int"/>
  <!-- Kúpeľňa: pravá stena -->
  <line x1="210" y1="185" x2="210" y2="380" class="wall-int"/>

  <!-- Obývačka / technická zóna — vertikálna delenie -->
  <line x1="490" y1="40" x2="490" y2="380" class="wall-int"/>
  <!-- Obývačka / technická — horizontálne delenie -->
  <line x1="210" y1="205" x2="490" y2="205" class="wall-int"/>

  <!-- Technická / pravé izby — vertikálna stena -->
  <line x1="680" y1="40" x2="680" y2="380" class="wall-int"/>

  <!-- Pravé izby — horizontálne delenie -->
  <line x1="680" y1="210" x2="836" y2="210" class="wall-int"/>

  <!-- Priečky v technickej zóne -->
  <line x1="490" y1="180" x2="680" y2="180" class="wall-int"/>

  <!-- Dvere vchod (chodba, ľavá stena) -->
  <line x1="84" y1="215" x2="84" y2="245" class="door-line"/>
  <path d="M 84,215 A 30,30 0 0,1 114,215" class="door-arc"/>

  <!-- Dvere terasa (obývačka, dolná stena) -->
  <line x1="340" y1="380" x2="380" y2="380" class="door-line"/>
  <path d="M 340,380 A 40,40 0 0,0 340,340" class="door-arc"/>

  <!-- Popisky miestností -->
  <text x="192" y="112" class="label label-lg">Spálňa</text>
  <text x="147" y="212" class="label label-sm">Chodba</text>
  <text x="147" y="226" class="label label-sm">/ Vchod</text>
  <text x="147" y="320" class="label label-lg">Kúpeľňa</text>
  <text x="350" y="300" class="label label-lg">Obývačka</text>
  <text x="585" y="290" class="label label-lg">Technická</text>
  <text x="585" y="308" class="label label-sm">miestnosť</text>
  <text x="585" y="110" class="label label-sm">Chodba</text>
  <text x="758" y="120" class="label label-lg">Izba</text>
  <text x="758" y="310" class="label label-lg">Izba</text>
  <text x="42" y="210" class="label label-sm" transform="rotate(-90,42,210)">Terasa</text>
  <text x="878" y="210" class="label label-sm" transform="rotate(90,878,210)">Terasa</text>
</svg>
```

- [ ] **Krok 3: Nahrať SVG na server**

```bash
scp /home/user/projects/home-assistant/www/podorys.svg smart-home-pc:/home/user/homeassistant/config/www/podorys.svg
```

- [ ] **Krok 4: Overiť súbor na serveri**

```bash
ssh smart-home-pc "ls -la /home/user/homeassistant/config/www/podorys.svg"
```

Očakávaný výstup: súbor existuje, veľkosť > 1 kB.

- [ ] **Krok 5: Commitnúť**

```bash
cd /home/user/projects/home-assistant
git add www/podorys.svg
git commit -m "feat: pridaj zjednodušený SVG pôdorys domu"
```

---

## Task 2: Vytvoriť Rekuperácia dashboard

**Files:**
- Create: `/home/user/homeassistant/config/.storage/lovelace.rekuperacia`

- [ ] **Krok 1: Zapísať Python skript lokálne**

Uložiť nasledujúci kód do `/tmp/write_rekuperacia.py` (lokálne):

```python
import json

data = {
  "version": 1,
  "minor_version": 1,
  "key": "lovelace.rekuperacia",
  "data": {
    "config": {
      "views": [
        {
          "title": "Rekuperácia",
          "icon": "mdi:hvac",
          "type": "sections",
          "max_columns": 4,
          "sections": [
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Rekuperácia Komfovent", "icon": "mdi:hvac"},
                {"type": "tile", "entity": "switch.rekuperacia_power", "name": "Stav rekuperácie", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "select.rekuperacia_operation_mode", "name": "Prevádzkový mód", "icon": "mdi:air-filter", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_active_alarms", "name": "Aktívne alarmy", "icon": "mdi:alert-circle-outline", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "binary_sensor.rekuperacia_status_alarm_fault", "name": "Porucha", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "binary_sensor.rekuperacia_status_alarm_warning", "name": "Varovanie", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_filter_clogging", "name": "Upchatie filtra", "icon": "mdi:air-filter", "grid_options": {"columns": 12, "rows": 1}},
                {
                  "type": "gauge",
                  "entity": "sensor.rekuperacia_heat_exchanger_efficiency",
                  "name": "Efektivita výmenníka",
                  "min": 0, "max": 100, "needle": True,
                  "segments": [
                    {"from": 0, "color": "#e53935"},
                    {"from": 60, "color": "#f9a825"},
                    {"from": 80, "color": "#43a047"}
                  ],
                  "grid_options": {"rows": "auto", "columns": "full"}
                }
              ]
            },
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Vzduch a teploty", "icon": "mdi:thermometer-lines"},
                {"type": "tile", "entity": "sensor.rekuperacia_supply_temperature", "name": "Prívod vzduchu", "icon": "mdi:thermometer-chevron-up", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_extract_temperature", "name": "Výfuk vzduchu", "icon": "mdi:thermometer-chevron-down", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_outdoor_temperature", "name": "Vonkajší vzduch", "icon": "mdi:thermometer", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_supply_flow", "name": "Prívod (m³/h)", "icon": "mdi:arrow-down-circle-outline", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_extract_flow", "name": "Výfuk (m³/h)", "icon": "mdi:arrow-up-circle-outline", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_heat_recovery", "name": "Tepelná regenerácia", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_supply_fan", "name": "Ventilátor prívod (%)", "icon": "mdi:fan", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_extract_fan", "name": "Ventilátor výfuk (%)", "icon": "mdi:fan", "grid_options": {"columns": 9, "rows": 1}},
                {
                  "type": "history-graph", "title": "Teploty vzduchu – 24h", "hours_to_show": 24,
                  "entities": [
                    {"entity": "sensor.rekuperacia_supply_temperature", "name": "Prívod"},
                    {"entity": "sensor.rekuperacia_extract_temperature", "name": "Výfuk"},
                    {"entity": "sensor.rekuperacia_outdoor_temperature", "name": "Vonku"}
                  ],
                  "grid_options": {"columns": 48, "rows": "auto"}
                },
                {
                  "type": "history-graph", "title": "Prietoky – 24h", "hours_to_show": 24,
                  "entities": [
                    {"entity": "sensor.rekuperacia_supply_flow", "name": "Prívod m³/h"},
                    {"entity": "sensor.rekuperacia_extract_flow", "name": "Výfuk m³/h"}
                  ],
                  "grid_options": {"columns": 48, "rows": "auto"}
                }
              ]
            },
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Energia", "icon": "mdi:lightning-bolt"},
                {"type": "tile", "entity": "sensor.rekuperacia_power_consumption", "name": "Aktuálny príkon", "icon": "mdi:flash", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_specific_power_input", "name": "Merný príkon", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_heater_power", "name": "Elektrický ohrev (W)", "icon": "mdi:radiator", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_total_ahu_energy", "name": "Celková spotreba VZT", "icon": "mdi:counter", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_total_recovered_energy", "name": "Zregenerovaná energia", "icon": "mdi:recycle", "grid_options": {"columns": 9, "rows": 1}},
                {"type": "tile", "entity": "sensor.rekuperacia_total_heater_energy", "name": "Energia el. ohrevu", "grid_options": {"columns": 9, "rows": 1}},
                {
                  "type": "history-graph", "title": "Príkon – 7 dní", "hours_to_show": 168,
                  "entities": [
                    {"entity": "sensor.rekuperacia_power_consumption", "name": "VZT príkon"},
                    {"entity": "sensor.rekuperacia_heater_power", "name": "El. ohrev"}
                  ],
                  "grid_options": {"columns": 48, "rows": 6}
                }
              ]
            },
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Ovládanie a servis", "icon": "mdi:tune", "heading_style": "title"},
                {
                  "type": "entities", "title": "Hlavné nastavenia",
                  "entities": [
                    {"entity": "switch.rekuperacia_power", "name": "Rekuperácia – ZAP/VYP"},
                    {"entity": "select.rekuperacia_operation_mode", "name": "Prevádzkový mód"},
                    {"entity": "select.rekuperacia_scheduler_mode", "name": "Plánovač"},
                    {"entity": "switch.rekuperacia_eco_mode", "name": "ECO mód"},
                    {"entity": "switch.rekuperacia_auto_mode", "name": "AUTO mód"},
                    {"entity": "select.rekuperacia_eco_heat_recovery", "name": "ECO rekuperácia tepla"}
                  ]
                },
                {
                  "type": "entities", "title": "Manuálny override",
                  "entities": [
                    {"entity": "select.rekuperacia_override_activation", "name": "Override aktivácia"},
                    {"entity": "number.rekuperacia_override_supply_flow", "name": "Override prívod"},
                    {"entity": "number.rekuperacia_override_extract_flow", "name": "Override výfuk"},
                    {"entity": "number.rekuperacia_override_timer", "name": "Override časovač"}
                  ]
                },
                {
                  "type": "entities", "title": "Boost / Away / Kitchen",
                  "entities": [
                    {"entity": "number.rekuperacia_boost_supply_flow", "name": "Boost prívod"},
                    {"entity": "number.rekuperacia_boost_extract_flow", "name": "Boost výfuk"},
                    {"entity": "number.rekuperacia_away_supply_flow", "name": "Away prívod"},
                    {"entity": "number.rekuperacia_away_extract_flow", "name": "Away výfuk"},
                    {"entity": "number.rekuperacia_kitchen_supply_flow", "name": "Kuchyňa prívod"},
                    {"entity": "number.rekuperacia_kitchen_extract_flow", "name": "Kuchyňa výfuk"},
                    {"entity": "number.rekuperacia_kitchen_timer", "name": "Kuchyňa časovač"}
                  ]
                },
                {
                  "type": "entities", "title": "Dovolenka",
                  "entities": [
                    {"entity": "datetime.rekuperacia_holidays_from", "name": "Od"},
                    {"entity": "datetime.rekuperacia_holidays_until", "name": "Do"},
                    {"entity": "number.rekuperacia_holidays_temperature", "name": "Teplota"},
                    {"entity": "select.rekuperacia_holidays_micro_ventilation", "name": "Micro ventilácia"}
                  ]
                },
                {
                  "type": "entities", "title": "Servis",
                  "entities": [
                    {"entity": "button.rekuperacia_clean_filters_calibration", "name": "Kalibrácia – čisté filtre"},
                    {"entity": "button.rekuperacia_clear_active_alarms", "name": "Vymazať alarmy"},
                    {"entity": "button.rekuperacia_set_system_time", "name": "Nastaviť čas"},
                    {"entity": "sensor.rekuperacia_controller_firmware", "name": "Firmware"}
                  ]
                }
              ]
            }
          ]
        }
      ]
    }
  }
}

with open('/home/user/homeassistant/config/.storage/lovelace.rekuperacia', 'w') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
print('OK')
```

- [ ] **Krok 2: Nahrať a spustiť skript**

```bash
scp /tmp/write_rekuperacia.py smart-home-pc:/tmp/write_rekuperacia.py
ssh smart-home-pc "python3 /tmp/write_rekuperacia.py"
```

Očakávaný výstup: `OK`

- [ ] **Krok 2: Overiť JSON**

```bash
ssh smart-home-pc "python3 -c \"import json; json.load(open('/home/user/homeassistant/config/.storage/lovelace.rekuperacia')); print('JSON OK')\""
```

Očakávaný výstup: `JSON OK`

---

## Task 3: Vytvoriť Prehľad domov dashboard

**Files:**
- Create: `/home/user/homeassistant/config/.storage/lovelace.prehlad`

Overlay pozície (`top`/`left` v %) sú odvodené od SVG viewBox 920×420. Po nasadení je možné upravovať priamo v HA code editore — každý element má komentár s názvom entity.

- [ ] **Krok 1: Zapísať Python skript lokálne**

Uložiť nasledujúci kód do `/tmp/write_prehlad.py` (lokálne):

```python
import json

# Pozície overlayov (top%, left%) odvodené od SVG viewBox 920×420
# Upraviteľné neskôr cez HA code editor

data = {
  "version": 1,
  "minor_version": 1,
  "key": "lovelace.prehlad",
  "data": {
    "config": {
      "views": [
        {
          "title": "Prehľad domov",
          "icon": "mdi:home",
          "type": "sections",
          "max_columns": 4,
          "sections": [
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Prehľad domov", "icon": "mdi:home-outline"},
                {
                  "type": "picture-elements",
                  "image": "/local/podorys.svg",
                  "grid_options": {"columns": 48, "rows": "auto"},
                  "elements": [
                    # --- Obývačka: teploty a vlhkosť ---
                    # sensor.teplota_teplota
                    {"type": "state-badge", "entity": "sensor.teplota_teplota", "style": {"top": "72%", "left": "36%", "transform": "translate(-50%,-50%)"}},
                    # sensor.teplota_vlhkost
                    {"type": "state-badge", "entity": "sensor.teplota_vlhkost", "style": {"top": "72%", "left": "44%", "transform": "translate(-50%,-50%)"}},
                    # sensor.teplota_2_teplota
                    {"type": "state-badge", "entity": "sensor.teplota_2_teplota", "style": {"top": "82%", "left": "36%", "transform": "translate(-50%,-50%)"}},
                    # sensor.teplota_2_vlhkost
                    {"type": "state-badge", "entity": "sensor.teplota_2_vlhkost", "style": {"top": "82%", "left": "44%", "transform": "translate(-50%,-50%)"}},
                    # --- Vchod: dvere ---
                    # binary_sensor.0xa4c1388ca7849bb9_contact (dvere vchod)
                    {"type": "state-icon", "entity": "binary_sensor.0xa4c1388ca7849bb9_contact", "state_color": True, "style": {"top": "52%", "left": "9%", "transform": "translate(-50%,-50%) scale(1.5)"}},
                    # --- Terasa: dvere ---
                    # binary_sensor.0xa4c138841a702c87_contact (dvere terasa)
                    {"type": "state-icon", "entity": "binary_sensor.0xa4c138841a702c87_contact", "state_color": True, "style": {"top": "91%", "left": "40%", "transform": "translate(-50%,-50%) scale(1.5)"}},
                    # --- Technická miestnosť ---
                    # binary_sensor.0xa4c1385a3860ff48_water_leak (senzor vody)
                    {"type": "state-icon", "entity": "binary_sensor.0xa4c1385a3860ff48_water_leak", "state_color": True, "style": {"top": "48%", "left": "61%", "transform": "translate(-50%,-50%) scale(1.4)"}},
                    # binary_sensor.tepelne_cerpadlo_esp_heatpump_state (TČ stav)
                    {"type": "state-icon", "entity": "binary_sensor.tepelne_cerpadlo_esp_heatpump_state", "state_color": True, "style": {"top": "62%", "left": "61%", "transform": "translate(-50%,-50%) scale(1.4)"}},
                    # sensor.tepelne_cerpadlo_esp_dhw_temp (TÚV teplota)
                    {"type": "state-badge", "entity": "sensor.tepelne_cerpadlo_esp_dhw_temp", "style": {"top": "62%", "left": "70%", "transform": "translate(-50%,-50%)"}},
                    # switch.rekuperacia_power (rekuperácia stav)
                    {"type": "state-icon", "entity": "switch.rekuperacia_power", "state_color": True, "style": {"top": "35%", "left": "61%", "transform": "translate(-50%,-50%) scale(1.4)"}}
                  ]
                }
              ]
            },
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Kamery", "icon": "mdi:cctv"},
                {
                  "type": "picture-entity",
                  "entity": "camera.jv_hd_stream",
                  "name": "Juhovýchod",
                  "camera_view": "live",
                  "show_state": False,
                  "grid_options": {"columns": 24, "rows": "auto"}
                },
                {
                  "type": "picture-entity",
                  "entity": "camera.sz_hd_stream",
                  "name": "Severozápad",
                  "camera_view": "live",
                  "show_state": False,
                  "grid_options": {"columns": 24, "rows": "auto"}
                }
              ]
            },
            {
              "type": "grid",
              "column_span": 4,
              "cards": [
                {"type": "heading", "heading": "Sieť", "icon": "mdi:router-wireless"},
                {"type": "tile", "entity": "binary_sensor.archer_ax17_stav_wan", "name": "WAN stav", "icon": "mdi:web", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "sensor.archer_ax17_externa_ip", "name": "Externá IP", "icon": "mdi:ip-network", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "sensor.archer_ax17_rychlost_stahovania", "name": "↓ Sťahovanie", "icon": "mdi:download", "grid_options": {"columns": 12, "rows": 1}},
                {"type": "tile", "entity": "sensor.archer_ax17_rychlost_odosielania", "name": "↑ Odosielanie", "icon": "mdi:upload", "grid_options": {"columns": 12, "rows": 1}},
                {
                  "type": "history-graph",
                  "title": "Sieť – 1h",
                  "hours_to_show": 1,
                  "entities": [
                    {"entity": "sensor.archer_ax17_rychlost_stahovania", "name": "↓"},
                    {"entity": "sensor.archer_ax17_rychlost_odosielania", "name": "↑"}
                  ],
                  "grid_options": {"columns": 48, "rows": "auto"}
                }
              ]
            }
          ]
        }
      ]
    }
  }
}

with open('/home/user/homeassistant/config/.storage/lovelace.prehlad', 'w') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
print('OK')
```

- [ ] **Krok 2: Nahrať a spustiť skript**

```bash
scp /tmp/write_prehlad.py smart-home-pc:/tmp/write_prehlad.py
ssh smart-home-pc "python3 /tmp/write_prehlad.py"
```

Očakávaný výstup: `OK`

- [ ] **Krok 3: Overiť JSON**

```bash
ssh smart-home-pc "python3 -c \"import json; json.load(open('/home/user/homeassistant/config/.storage/lovelace.prehlad')); print('JSON OK')\""
```

Očakávaný výstup: `JSON OK`

---

## Task 4: Zaregistrovať oba dashboardy

**Files:**
- Modify: `/home/user/homeassistant/config/.storage/lovelace_dashboards`

- [ ] **Krok 1: Pridať záznamy pre Rekuperácia a Prehľad domov**

```bash
ssh smart-home-pc "python3 - << 'EOF'
import json

path = '/home/user/homeassistant/config/.storage/lovelace_dashboards'
with open(path) as f:
    data = json.load(f)

existing_ids = {item['id'] for item in data['data']['items']}

new_items = [
    {
        'id': 'rekuperacia',
        'show_in_sidebar': True,
        'title': 'Rekuperácia',
        'icon': 'mdi:hvac',
        'require_admin': False,
        'mode': 'storage',
        'url_path': 'rekuperacia'
    },
    {
        'id': 'prehlad',
        'show_in_sidebar': True,
        'title': 'Prehľad domov',
        'icon': 'mdi:home',
        'require_admin': False,
        'mode': 'storage',
        'url_path': 'prehlad'
    }
]

for item in new_items:
    if item['id'] not in existing_ids:
        data['data']['items'].append(item)

with open(path, 'w') as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
print('OK —', len(data['data']['items']), 'dashboardov celkom')
EOF"
```

Očakávaný výstup: `OK — 5 dashboardov celkom`

- [ ] **Krok 2: Overiť obsah**

```bash
ssh smart-home-pc "python3 -c \"
import json
d = json.load(open('/home/user/homeassistant/config/.storage/lovelace_dashboards'))
for item in d['data']['items']:
    print(item['title'], '->', item['url_path'])
\""
```

Očakávaný výstup:
```
Mapa -> map
Test -> dashboard-test
Tepelné čerpadlo -> tepelne-cerpadlo
Rekuperácia -> rekuperacia
Prehľad domov -> prehlad
```

---

## Task 5: Reštartovať HA a overiť dashboardy

- [ ] **Krok 1: Reštartovať HA**

```bash
ssh smart-home-pc "docker restart homeassistant"
```

- [ ] **Krok 2: Počkať na štart (cca 30s)**

```bash
ssh smart-home-pc "sleep 35 && docker logs homeassistant --tail 5"
```

Overiť, že logy neobsahujú `ERROR` súvisiaci s lovelace.

- [ ] **Krok 3: Overiť Rekuperácia dashboard**

Otvoriť v browseri: `http://smart-home-pc:8123/rekuperacia`

Skontrolovať:
- Zobrazuje sa sidebar položka "Rekuperácia"
- Viditeľné 4 sekcie: Prehľad, Vzduch a teploty, Energia, Ovládanie
- Gauge efektivita výmenníka zobrazuje hodnotu
- Tiles majú dáta (nie `unavailable`)

- [ ] **Krok 4: Overiť Prehľad domov dashboard**

Otvoriť: `http://smart-home-pc:8123/prehlad`

Skontrolovať:
- Viditeľný SVG pôdorys s overlaymi
- Overlaye zobrazujú hodnoty/ikony senzorv
- Kamery JV a SZ zobrazujú stream
- Sieť tiles majú dáta

- [ ] **Krok 5: Commit finálneho stavu**

```bash
cd /home/user/projects/home-assistant
git add docs/superpowers/plans/2026-05-12-dashboards.md
git commit -m "feat: implementácia dashboardov Rekuperácia + Prehľad domov"
```

---

## Task 6: Fine-tuning SVG overlay pozícií (voliteľné)

Ak overlaye na mape nesedia presne, upraviť `top`/`left` hodnoty priamo v HA:

1. Otvoriť HA → Settings → Dashboards → Prehľad domov → Edit → Raw configuration editor
2. Nájsť element podľa `entity` ID
3. Zmeniť `top`/`left` percentá (+/- 2-5% = výrazný posun)
4. Uložiť → okamžitý efekt bez reštartu

**Referenčná mapa SVG koordinát → % pozície:**

| Miesto | SVG x,y (z 920×420) | top% | left% |
|---|---|---|---|
| Obývačka (stred) | 350, 300 | 71% | 38% |
| Technická (stred) | 585, 290 | 69% | 64% |
| Chodba/Vchod | 84, 230 | 55% | 9% |
| Terasa (dvere) | 360, 380 | 90% | 39% |

---

## Poznámky k implementácii

- Python skripty pre Task 2 a 3 zapisovať lokálne ako `/tmp/write_rekuperacia.py` a `/tmp/write_prehlad.py`, potom `scp` + `ssh python3`
- Komentáre `#` v Python dict-och plánu nie sú súčasťou výsledného JSON — json.dump ich ignoruje
- Ak `picture-elements` karta nezobrazuje SVG: skontrolovať `http://smart-home-pc:8123/local/podorys.svg` priamo v browseri
- Ak entity sú `unavailable`: je to normálne pre novú inštaláciu, dáta prídu po chvíli
