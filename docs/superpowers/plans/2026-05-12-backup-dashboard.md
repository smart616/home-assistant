# Backup Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Zobraziť stav Restic zálohovania (stav poslednej zálohy, snapshoty, čo sa zálohuje) v Homepage dashboarde cez malý Flask API kontajner na serveri `smart-home-pc`.

**Architecture:** Nový Docker service `backup-api` (Python Flask, `python:3.12-alpine`) v monitoring compose `/home/user/stacks/monitoring/compose.yaml`. Mountuje log súbor, restic heslo a SSH kľúč (read-only), binárka restic z hostu. Homepage volá `http://localhost:7070` (host network). Tri endpointy: `/status` (parsuje log), `/snapshots` (live restic), `/sources` (statický zoznam).

**Tech Stack:** Python 3.12, Flask, restic CLI, Docker Compose, Homepage customApi widget

---

## File Map

| Súbor (server) | Akcia | Obsah |
|---|---|---|
| `/opt/stacks/backup/backup-api/app.py` | Create | Flask API server |
| `/opt/stacks/backup/backup-api/Dockerfile` | Create | Build image |
| `/opt/stacks/backup/backup-api/tests/test_app.py` | Create | pytest testy |
| `/home/user/stacks/monitoring/compose.yaml` | Modify | Pridať `backup-api` service |
| `/home/user/stacks/monitoring/homepage/config/services.yaml` | Modify | Pridať skupinu `Zálohy` |

---

## Task 1: Scaffold — adresár, Dockerfile, skeleton app.py

**Files:**
- Create: `/opt/stacks/backup/backup-api/app.py`
- Create: `/opt/stacks/backup/backup-api/Dockerfile`
- Create: `/opt/stacks/backup/backup-api/tests/__init__.py`

- [ ] **Step 1: Vytvor adresárovú štruktúru na serveri**

```bash
ssh smart-home-pc "mkdir -p /opt/stacks/backup/backup-api/tests"
```

Expected: žiadna chyba.

- [ ] **Step 2: Vytvor Dockerfile**

```bash
ssh smart-home-pc "cat > /opt/stacks/backup/backup-api/Dockerfile << 'EOF'
FROM python:3.12-alpine
RUN pip install flask --no-cache-dir
WORKDIR /app
COPY app.py .
CMD [\"python\", \"app.py\"]
EOF"
```

- [ ] **Step 3: Vytvor skeleton app.py so všetkými endpointmi (stub implementácie)**

```bash
ssh smart-home-pc "cat > /opt/stacks/backup/backup-api/app.py << 'PYEOF'
import os
import re
import json
import subprocess
from datetime import datetime
from flask import Flask, jsonify

app = Flask(__name__)

LOG_PATH = \"/log/restic-backup.log\"
RESTIC_REPO = os.getenv(\"RESTIC_REPOSITORY\", \"sftp://u591423@u591423.your-storagebox.de:23/backup\")
RESTIC_PASSWORD_FILE = \"/etc/restic-password\"

SOURCES = [
    {\"name\": \"Immich photos\", \"path\": \"/mnt/immich/library/\"},
    {\"name\": \"Home Assistant\", \"path\": \"/home/user/homeassistant/\"},
    {\"name\": \"Seafile\", \"path\": \"/mnt/immich/seafile/\"},
    {\"name\": \"Smart PWA\", \"path\": \"/home/user/smart/\"},
    {\"name\": \"Stacks config\", \"path\": \"/opt/stacks/\"},
]

_TS_RE = re.compile(r\"\\[(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2})\\]\")


def parse_log(log_path=None):
    path = log_path if log_path is not None else LOG_PATH
    last_start = None
    last_end = None
    try:
        with open(path) as f:
            for line in f:
                m = _TS_RE.match(line)
                if not m:
                    continue
                ts = datetime.strptime(m.group(1), \"%Y-%m-%d %H:%M:%S\")
                if \"=== Backup start ===\" in line:
                    last_start = ts
                elif \"=== Backup done ===\" in line:
                    last_end = ts
    except FileNotFoundError:
        return {\"status\": \"unknown\", \"last_start\": None, \"last_end\": None,
                \"duration_seconds\": None, \"duration_formatted\": None}

    if last_start is None:
        return {\"status\": \"unknown\", \"last_start\": None, \"last_end\": None,
                \"duration_seconds\": None, \"duration_formatted\": None}

    now = datetime.now()
    duration_seconds = None
    duration_formatted = None

    if last_end and last_end >= last_start:
        status = \"ok\"
        duration_seconds = int((last_end - last_start).total_seconds())
        mins, secs = divmod(duration_seconds, 60)
        duration_formatted = f\"{mins}m {secs}s\"
    elif (now - last_start).total_seconds() < 7200:
        status = \"running\"
    else:
        status = \"failed\"

    return {
        \"status\": status,
        \"last_start\": last_start.isoformat(),
        \"last_end\": last_end.isoformat() if last_end else None,
        \"duration_seconds\": duration_seconds,
        \"duration_formatted\": duration_formatted,
    }


@app.route(\"/status\")
def status():
    return jsonify(parse_log())


@app.route(\"/snapshots\")
def snapshots():
    result = subprocess.run(
        [\"restic\", \"-r\", RESTIC_REPO, \"--password-file\", RESTIC_PASSWORD_FILE,
         \"--no-lock\", \"snapshots\", \"--json\"],
        capture_output=True, text=True, timeout=30,
    )
    if result.returncode != 0:
        return jsonify({\"error\": result.stderr.strip()}), 502
    snaps = json.loads(result.stdout)
    latest = max(snaps, key=lambda s: s[\"time\"]) if snaps else None
    return jsonify({
        \"total\": len(snaps),
        \"latest_time\": latest[\"time\"] if latest else None,
        \"snapshots\": [
            {\"id\": s[\"short_id\"], \"time\": s[\"time\"], \"tags\": s.get(\"tags\", [])}
            for s in snaps
        ],
    })


@app.route(\"/sources\")
def sources():
    return jsonify(SOURCES)


if __name__ == \"__main__\":
    app.run(host=\"0.0.0.0\", port=7070)
PYEOF"
```

- [ ] **Step 4: Vytvor prázdny `__init__.py`**

```bash
ssh smart-home-pc "touch /opt/stacks/backup/backup-api/tests/__init__.py"
```

- [ ] **Step 5: Overenie — súbory existujú**

```bash
ssh smart-home-pc "ls -la /opt/stacks/backup/backup-api/ && ls /opt/stacks/backup/backup-api/tests/"
```

Expected: `app.py`, `Dockerfile`, `tests/` adresár s `__init__.py`.

---

## Task 2: Testy a implementácia `/status` (log parsing)

**Files:**
- Create: `/opt/stacks/backup/backup-api/tests/test_app.py`

- [ ] **Step 1: Vytvor test súbor s testami pre parse_log()**

```bash
ssh smart-home-pc "cat > /opt/stacks/backup/backup-api/tests/test_app.py << 'TESTEOF'
import json
import os
import sys
import pytest
from unittest.mock import patch

os.environ.setdefault(\"RESTIC_REPOSITORY\", \"test-repo\")
sys.path.insert(0, \"/app\")


def write_log(tmp_path, content):
    log = tmp_path / \"backup.log\"
    log.write_text(content)
    return str(log)


def test_parse_log_ok(tmp_path):
    from app import parse_log
    log = write_log(tmp_path,
        \"[2026-05-12 02:30:00] === Backup start ===\\n\"
        \"[2026-05-12 02:47:23] === Backup done ===\\n\"
    )
    result = parse_log(log)
    assert result[\"status\"] == \"ok\"
    assert result[\"last_start\"] == \"2026-05-12T02:30:00\"
    assert result[\"last_end\"] == \"2026-05-12T02:47:23\"
    assert result[\"duration_seconds\"] == 1043
    assert result[\"duration_formatted\"] == \"17m 23s\"


def test_parse_log_running(tmp_path):
    from datetime import datetime
    from app import parse_log
    now = datetime.now().strftime(\"%Y-%m-%d %H:%M:%S\")
    log = write_log(tmp_path, f\"[{now}] === Backup start ===\\n\")
    result = parse_log(log)
    assert result[\"status\"] == \"running\"
    assert result[\"duration_seconds\"] is None


def test_parse_log_failed(tmp_path):
    from app import parse_log
    log = write_log(tmp_path, \"[2020-01-01 02:30:00] === Backup start ===\\n\")
    result = parse_log(log)
    assert result[\"status\"] == \"failed\"


def test_parse_log_duplicate_lines(tmp_path):
    from app import parse_log
    log = write_log(tmp_path,
        \"[2026-05-12 02:30:00] === Backup start ===\\n\"
        \"[2026-05-12 02:30:00] === Backup start ===\\n\"
        \"[2026-05-12 02:47:23] === Backup done ===\\n\"
        \"[2026-05-12 02:47:23] === Backup done ===\\n\"
    )
    result = parse_log(log)
    assert result[\"status\"] == \"ok\"
    assert result[\"duration_seconds\"] == 1043


def test_parse_log_missing_file():
    from app import parse_log
    result = parse_log(\"/nonexistent/path/backup.log\")
    assert result[\"status\"] == \"unknown\"


def test_status_endpoint(tmp_path):
    import app as app_module
    log = write_log(tmp_path,
        \"[2026-05-12 02:30:00] === Backup start ===\\n\"
        \"[2026-05-12 02:47:23] === Backup done ===\\n\"
    )
    with patch(\"app.LOG_PATH\", log):
        app_module.app.testing = True
        client = app_module.app.test_client()
        resp = client.get(\"/status\")
    assert resp.status_code == 200
    data = resp.get_json()
    assert data[\"status\"] == \"ok\"
    assert data[\"duration_formatted\"] == \"17m 23s\"


def test_sources_endpoint():
    import app as app_module
    app_module.app.testing = True
    client = app_module.app.test_client()
    resp = client.get(\"/sources\")
    assert resp.status_code == 200
    data = resp.get_json()
    assert len(data) == 5
    names = [s[\"name\"] for s in data]
    assert \"Immich photos\" in names
    assert \"Home Assistant\" in names
    assert \"Seafile\" in names


def test_snapshots_endpoint():
    import app as app_module
    mock_snaps = [
        {\"short_id\": \"abc123\", \"time\": \"2026-05-12T02:47:23.000+02:00\",
         \"tags\": [\"auto\"], \"paths\": [\"/opt/stacks\"]}
    ]
    app_module.app.testing = True
    client = app_module.app.test_client()
    with patch(\"subprocess.run\") as mock_run:
        mock_run.return_value.returncode = 0
        mock_run.return_value.stdout = json.dumps(mock_snaps)
        resp = client.get(\"/snapshots\")
    assert resp.status_code == 200
    data = resp.get_json()
    assert data[\"total\"] == 1
    assert data[\"latest_time\"] == \"2026-05-12T02:47:23.000+02:00\"
    assert data[\"snapshots\"][0][\"id\"] == \"abc123\"


def test_snapshots_endpoint_restic_error():
    import app as app_module
    app_module.app.testing = True
    client = app_module.app.test_client()
    with patch(\"subprocess.run\") as mock_run:
        mock_run.return_value.returncode = 1
        mock_run.return_value.stderr = \"unable to open repo\"
        resp = client.get(\"/snapshots\")
    assert resp.status_code == 502
    data = resp.get_json()
    assert \"error\" in data
TESTEOF"
```

- [ ] **Step 2: Spusti všetky testy — overenie nastavenia**

```bash
ssh smart-home-pc "docker run --rm \
  -v /opt/stacks/backup/backup-api:/app \
  -w /app \
  python:3.12-alpine \
  sh -c 'pip install flask pytest -q && python -m pytest tests/ -v 2>&1' | tail -30"
```

Expected: všetky testy PASS, žiadne import errory.

Ak vidíš `ModuleNotFoundError: No module named 'app'` — over, že `-w /app` je v príkaze.

- [ ] **Step 3: Spusti len parse_log testy**

```bash
ssh smart-home-pc "docker run --rm \
  -v /opt/stacks/backup/backup-api:/app \
  -w /app \
  python:3.12-alpine \
  sh -c 'pip install flask pytest -q && python -m pytest tests/test_app.py -k parse_log -v 2>&1'"
```

Expected: všetky `test_parse_log_*` PASS (implementácia parse_log je kompletná).

- [ ] **Step 4: Spusti všetky testy**

```bash
ssh smart-home-pc "docker run --rm \
  -v /opt/stacks/backup/backup-api:/app \
  -w /app \
  python:3.12-alpine \
  sh -c 'pip install flask pytest -q && python -m pytest tests/ -v 2>&1'"
```

Expected: všetky testy PASS (`test_status_endpoint`, `test_sources_endpoint`, `test_snapshots_endpoint`, `test_snapshots_endpoint_restic_error`).

Ak niektorý FAIL — skontroluj chybovú hlášku a oprav `app.py` podľa nej.

---

## Task 3: Build Docker image + pridaj do monitoring compose

**Files:**
- Modify: `/home/user/stacks/monitoring/compose.yaml`

- [ ] **Step 1: Prečítaj aktuálny monitoring compose**

```bash
ssh smart-home-pc "cat /home/user/stacks/monitoring/compose.yaml"
```

- [ ] **Step 2: Pridaj `backup-api` service do compose.yaml**

Pridaj nasledovný blok na koniec sekcie `services:` v `/home/user/stacks/monitoring/compose.yaml`:

```yaml
  backup-api:
    build:
      context: /opt/stacks/backup/backup-api
    container_name: backup-api
    restart: unless-stopped
    network_mode: host
    volumes:
      - /var/log/restic-backup.log:/log/restic-backup.log:ro
      - /opt/stacks/backup/.restic-password:/etc/restic-password:ro
      - /root/.ssh:/root/.ssh:ro
      - /usr/bin/restic:/usr/bin/restic:ro
    environment:
      RESTIC_REPOSITORY: sftp://u591423@u591423.your-storagebox.de:23/backup
```

Upozornenie: `/var/log/restic-backup.log` je čitateľný iba rootom — kontajner beží ako root štandardne, takže mount funguje.

- [ ] **Step 3: Build image**

```bash
ssh smart-home-pc "cd /home/user/stacks/monitoring && docker compose build backup-api 2>&1"
```

Expected: `Successfully built ...` alebo `=> exporting to image`.

Ak `COPY app.py .` zlyhá — over, že súbor existuje na `/opt/stacks/backup/backup-api/app.py`.

- [ ] **Step 4: Spusti kontajner**

```bash
ssh smart-home-pc "cd /home/user/stacks/monitoring && docker compose up -d backup-api 2>&1"
```

Expected: `Container backup-api  Started`.

- [ ] **Step 5: Skontroluj logy kontajnera**

```bash
ssh smart-home-pc "docker logs backup-api 2>&1"
```

Expected: `* Running on all addresses (0.0.0.0)` a `* Running on http://0.0.0.0:7070`.

- [ ] **Step 6: Otestuj `/status` endpoint**

```bash
ssh smart-home-pc "curl -s http://localhost:7070/status | python3 -m json.tool"
```

Expected: JSON s `"status": "ok"` (alebo `"failed"` ak záloha dávno bežala), `last_start`, `duration_formatted`.

- [ ] **Step 7: Otestuj `/sources` endpoint**

```bash
ssh smart-home-pc "curl -s http://localhost:7070/sources | python3 -m json.tool"
```

Expected: JSON pole s 5 položkami (Immich photos, Home Assistant, Seafile, Smart PWA, Stacks config).

- [ ] **Step 8: Otestuj `/snapshots` endpoint (bude trvať ~5s)**

```bash
ssh smart-home-pc "curl -s http://localhost:7070/snapshots | python3 -m json.tool"
```

Expected: JSON s `"total"` > 0, `"latest_time"`, `"snapshots"` pole.

Ak vidíš `"error": "..."` — pravdepodobne SSH problém. Skontroluj:
```bash
ssh smart-home-pc "docker exec backup-api ssh -T u591423@u591423.your-storagebox.de -p 23 -o BatchMode=yes 2>&1 || true"
```
Ak `Host key verification failed` — known_hosts v `/root/.ssh/` neobsahuje Hetzner. Fix:
```bash
ssh smart-home-pc "ssh-keyscan -p 23 u591423.your-storagebox.de >> /root/.ssh/known_hosts"
```
Potom reštartuj kontajner: `docker restart backup-api`

- [ ] **Step 9: Commit compose zmeny v lokálnom repo**

V lokálnom repo `/home/user/projects/home-assistant/`:
```bash
git add docs/superpowers/plans/2026-05-12-backup-dashboard.md
git commit -m "docs: implementačný plán pre Backup Dashboard — Flask API + Homepage widgety"
```

---

## Task 4: Pridaj Homepage widgety — skupina Zálohy

**Files:**
- Modify: `/home/user/stacks/monitoring/homepage/config/services.yaml`

- [ ] **Step 1: Prečítaj aktuálny services.yaml**

```bash
ssh smart-home-pc "cat /home/user/stacks/monitoring/homepage/config/services.yaml"
```

- [ ] **Step 2: Pridaj skupinu `Zálohy` na koniec súboru**

Pridaj na koniec `/home/user/stacks/monitoring/homepage/config/services.yaml`:

```yaml
- Zálohy:
    - Záloha - stav:
        icon: mdi-backup-restore
        description: Restic → Hetzner Storage Box
        widget:
          type: customapi
          url: http://localhost:7070/status
          refreshInterval: 300000
          mappings:
            - field: status
              label: Status
              format: text
            - field: last_start
              label: Posledná záloha
              format: relativeDate
            - field: duration_formatted
              label: Trvanie
              format: text

    - Záloha - snapshoty:
        icon: mdi-database
        description: Hetzner Storage Box
        widget:
          type: customapi
          url: http://localhost:7070/snapshots
          refreshInterval: 600000
          mappings:
            - field: total
              label: Snapshotov
              format: number
            - field: latest_time
              label: Posledný snapshot
              format: relativeDate

    - Čo sa zálohuje:
        icon: mdi-folder-multiple
        description: "Immich · Home Assistant · Seafile · Smart PWA · Stacks"
```

Použi editor alebo heredoc cez SSH:
```bash
ssh smart-home-pc "cat >> /home/user/stacks/monitoring/homepage/config/services.yaml << 'YAMLEOF'

- Zálohy:
    - Záloha - stav:
        icon: mdi-backup-restore
        description: Restic → Hetzner Storage Box
        widget:
          type: customapi
          url: http://localhost:7070/status
          refreshInterval: 300000
          mappings:
            - field: status
              label: Status
              format: text
            - field: last_start
              label: Posledná záloha
              format: relativeDate
            - field: duration_formatted
              label: Trvanie
              format: text

    - Záloha - snapshoty:
        icon: mdi-database
        description: Hetzner Storage Box
        widget:
          type: customapi
          url: http://localhost:7070/snapshots
          refreshInterval: 600000
          mappings:
            - field: total
              label: Snapshotov
              format: number
            - field: latest_time
              label: Posledný snapshot
              format: relativeDate

    - Čo sa zálohuje:
        icon: mdi-folder-multiple
        description: "Immich · Home Assistant · Seafile · Smart PWA · Stacks"
YAMLEOF"
```

- [ ] **Step 3: Overenie YAML syntaxe**

```bash
ssh smart-home-pc "python3 -c \"import yaml; yaml.safe_load(open('/home/user/stacks/monitoring/homepage/config/services.yaml'))\" && echo 'YAML OK'"
```

Expected: `YAML OK`. Ak syntax error — oprav súbor.

- [ ] **Step 4: Reštartuj Homepage**

```bash
ssh smart-home-pc "docker restart homepage"
```

- [ ] **Step 5: Overenie v prehliadači**

Otvor `http://smart-home-pc:3001` v prehliadači.

Skontroluj:
- Skupina `Zálohy` sa zobrazuje
- Karta `Záloha - stav` ukazuje `Status: ok` (alebo `failed`/`running`), relatívny čas poslednej zálohy, trvanie
- Karta `Záloha - snapshoty` ukazuje počet snapshotov a čas posledného (táto karta načítava ~5s pri prvom zobrazení)
- Karta `Čo sa zálohuje` zobrazuje popis

Ak widgety ukazujú chybu — skontroluj DevTools → Network pre URL `http://localhost:7070/status`.

Ak `customapi` widget nefunguje — over verziu Homepage: `docker exec homepage sh -c 'cat /app/package.json | grep version'`. Customapi widget je dostupný od verzie `v0.6.0`.

- [ ] **Step 6: Commit finálneho stavu**

```bash
git add docs/superpowers/plans/2026-05-12-backup-dashboard.md
git commit -m "docs: update plán — backup dashboard implementácia"
```
