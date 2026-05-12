# Backup Dashboard — Design Spec

**Dátum:** 2026-05-12  
**Téma:** Prehľad zálohovania v Homepage

## Cieľ

Zobraziť stav Restic zálohovania (Hetzner Storage Box) priamo v Homepage dashboarde — bez otvárania ďalších nástrojov. Zahrnúť: posledný beh, výsledok, trvanie, počet snapshotov, čo sa zálohuje.

## Architektúra

Nový Docker service `backup-api` (Python Flask, `python:3.12-alpine`) v monitoring compose `/home/user/stacks/monitoring/compose.yaml`.

### Mounty (read-only)

| Host path | Container path | Účel |
|---|---|---|
| `/var/log/restic-backup.log` | `/log/restic-backup.log` | Log zálohovania |
| `/opt/stacks/backup/.restic-password` | `/etc/restic-password` | Heslo k Restic repo |
| `/root/.ssh` | `/root/.ssh` | SSH kľúč pre SFTP na Hetzner |

Restic binárka inštalovaná v Docker image (`apk add restic`).

**Env premenná:** `RESTIC_REPOSITORY=sftp://u591423@u591423.your-storagebox.de:23/backup`

**Port:** `7070` (LAN only, bez SSL).

**Sieť:** Homepage container a `backup-api` v rovnakej Docker bridge sieti — Homepage volá `http://backup-api:7070`.

## API endpointy

### `GET /status`

Parsuje `/log/restic-backup.log`. Vracia:

```json
{
  "status": "ok",
  "last_start": "2026-05-12T02:30:00",
  "last_end": "2026-05-12T02:47:23",
  "duration_seconds": 1043
}
```

**Logika statusu:**
- `"ok"` — posledný `=== Backup done ===` nasleduje po poslednom `=== Backup start ===`
- `"running"` — posledný start nemá zodpovedajúci done
- `"failed"` — posledný start je starší ako 26 hodín a nemá done

### `GET /snapshots`

Spustí `restic -r sftp://... --password-file /etc/restic-password snapshots --json`. Live volanie (~2–5 s). Vracia:

```json
{
  "total": 17,
  "latest_time": "2026-05-12T02:47:23",
  "snapshots": [
    {"id": "abc123", "time": "2026-05-12T02:47:23", "tags": ["auto"]}
  ]
}
```

### `GET /sources`

Statický zoznam zálohovaných adresárov definovaný priamo v `app.py` ako Python list. Vracia:

```json
[
  {"name": "Immich photos", "path": "/mnt/immich/library/"},
  {"name": "Home Assistant", "path": "/home/user/homeassistant/"},
  {"name": "Seafile", "path": "/mnt/immich/seafile/"},
  {"name": "Smart PWA", "path": "/home/user/smart/"},
  {"name": "Stacks config", "path": "/opt/stacks/"}
]
```

## Homepage widgety

Nová skupina `Zálohy` v `services.yaml` s 3 kartami.

### Karta 1 — Stav zálohy

`customApi` widget → `GET /status`

Zobrazí: `status`, `last_start` (formát dátumu), `duration_seconds` (prepočítané na min:s).

### Karta 2 — Snapshoty

`customApi` widget → `GET /snapshots`

Zobrazí: `total` (počet snapshotov), `latest_time` (formát dátumu).

### Karta 3 — Čo sa zálohuje

Statická service karta (bez widgetu) — zoznam zálohovaných položiek v `description` poli. Žiadny HTTP request.

## Čo sa nezahrňuje (v tejto verzii)

- Manuálny trigger zálohy
- Live veľkosť jednotlivých snapshotov (restic `stats` je pomalé)
- Smart PWA backup (`~/smart/scripts/backup.sh`) — samostatný cron, iný log

## Súbory na zmenu

| Súbor | Zmena |
|---|---|
| `/home/user/stacks/monitoring/compose.yaml` | Pridať `backup-api` service |
| `/home/user/stacks/monitoring/homepage/config/services.yaml` | Pridať skupinu `Zálohy` |
| `/opt/stacks/backup/backup-api/app.py` | Nový Flask API server |
| `/opt/stacks/backup/backup-api/Dockerfile` | Build image |
