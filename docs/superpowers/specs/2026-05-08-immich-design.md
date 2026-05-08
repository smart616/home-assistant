# Immich v2.x — Design Spec

**Date:** 2026-05-08  
**Server:** smart-home-pc (MacBook Air 2013, 3.8 GB RAM, i5-4260U)

## Context

Google Photos náhrada. Záloha fotiek z iPhone (Companion app) + bulk import existujúcej knižnice. RAM 3.8 GB → ML vypnuté. Storage začína na internom SSD, neskôr presun na USB HDD.

## Architektúra

3 kontajnery na default bridge sieti:

| Kontajner | Image | Účel |
|---|---|---|
| `immich-server` | `ghcr.io/immich-app/immich-server:release` | API + web UI |
| `immich-postgres` | `ghcr.io/immich-app/postgres:14-vectorchord0.3.0-pgvecto.rs0.3.0` | DB s pgvector |
| `immich-redis` | `docker.io/valkey/valkey:8-bookworm` | cache / queue |
| ~~`immich-machine-learning`~~ | zakomentovaný | 4 GB RAM nedostatok |

Port `2283` mapovaný na host → LAN prístup `http://smart-home-pc:2283`.

## Súborová štruktúra

```
/opt/stacks/immich/
├── compose.yaml
├── .env                  # gitignored — obsahuje DB heslo, secret
├── .env.example          # template bez hodnôt
└── library/              # UPLOAD_LOCATION — fotky z Companion app
```

External library (bulk import):
- Bind mount `/home/user/Photos` (alebo iná cesta) → `/usr/src/app/external` (read-only)
- Cesta nastavená v `.env` ako `EXTERNAL_PATH`

## Volumes

| Volume | Typ | Obsah |
|---|---|---|
| `immich_library` | bind mount (`./library/`) | fotky z Companion app |
| `immich_external` | bind mount (`$EXTERNAL_PATH`) | existujúce fotky, read-only |
| `immich_postgres` | named Docker volume | PostgreSQL data |
| `immich_model-cache` | named Docker volume | zakomentovaný, pre ML |

## .env premenné

```env
# DB
DB_PASSWORD=<silné heslo>
DB_USERNAME=immich
DB_DATABASE_NAME=immich

# Storage
UPLOAD_LOCATION=./library
EXTERNAL_PATH=/home/user/Photos    # cesta k existujúcim fotkám

# Immich
IMMICH_VERSION=release
```

## Operácie

### Companion app (iPhone)
1. Nainštaluj Immich iOS app
2. Server URL: `http://192.168.0.219:2283`
3. Zapni auto-backup v nastaveniach app

### Bulk import existujúcich fotiek
```bash
# Spusti po prvom nasadení
docker exec -it immich-server immich-admin upload --recursive /usr/src/app/external
# alebo cez web UI: Administration → Jobs → Library scan
```

### Presun na USB HDD (keď príde)
1. Zastav stack: `docker compose stop immich-server`
2. `rsync -av /opt/stacks/immich/library/ /mnt/media/immich/library/`
3. Zmeň `.env`: `UPLOAD_LOCATION=/mnt/media/immich/library`
4. `docker compose start immich-server`

## Backup

```bash
# pg_dump pred zálohou (one-shot)
docker exec immich-postgres pg_dump -U immich immich > /opt/stacks/immich/backup-$(date +%F).sql

# Restic targets
# - /opt/stacks/immich/library/
# - /opt/stacks/immich/backup-*.sql
# - Docker named volume: immich_postgres (cez tar kontajner)
```

### Upgrade postup
1. `pg_dump` snapshot (vyššie)
2. `docker compose pull && docker compose up -d`
3. Immich dodržuje semver v2.x — bezpečné

## ML zapnutie (budúcnosť)

Keď príde väčší stroj alebo sa uvoľní RAM:
1. Odkomentuj `immich-machine-learning` v `compose.yaml`
2. Odkomentuj `MACHINE_LEARNING_WORKERS=1` v `.env`
3. `docker compose up -d`

## Tailscale Serve (budúcnosť)

```bash
sudo tailscale serve --bg --https 8443 http://localhost:2283
```

## Hotové / Nehotové

| | |
|---|---|
| ✅ | compose.yaml, .env.example, library/ |
| ✅ | LAN prístup (port 2283) |
| ✅ | Companion app setup |
| ✅ | Bulk import (external mount) |
| ⬜ | Tailscale Serve (neskôr) |
| ⬜ | USB HDD presun (neskôr) |
| ⬜ | ML (neskôr, väčší stroj) |
| ⬜ | Restic backup (krok 8) |
