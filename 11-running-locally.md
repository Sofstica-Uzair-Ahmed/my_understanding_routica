# 11 · Running it locally

[← 10 · Config and credentials](10-config-and-credentials.md) · [Index](00-START-HERE.md) · **Next →** [12 · Build, deploy and CI](12-build-deploy-ci.md)

---

## Pick a way to run

```mermaid
flowchart TB
    Q{"What do you need?"}
    Q -->|"Just read and run unit tests"| T["Option 0: venv + pytest<br/>(no DB, no Redis)"]
    Q -->|"Develop the API and chat;<br/>feed flow records"| H["Option 1 (recommended): HYBRID<br/>Postgres + Redis in Docker,<br/>app on your machine with uvicorn"]
    Q -->|"Production-like, with Zeek for pcaps"| D["Option 2: FULL DOCKER<br/>docker-compose.local.yml"]
```

Prerequisites: Python **3.12**, Docker + Compose, `git`. Zeek is only needed for pcap analysis, and it is already inside the Docker image.

---

## Option 0: unit tests only

```bash
cd core
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install pytest                      # not in requirements.txt

# config must import: dummy values are fine (engines are built at import, never connected)
export DATABASE_URL="postgresql+asyncpg://u:p@localhost:5432/x" \
       REDIS_HOST=localhost REDIS_PORT=6379 REDIS_PASSWORD=x \
       BACKEND_URL= BACKEND_API_KEY=
pytest app/tests -q
```

Some tests skip on purpose. They need a real Postgres (`ROUTICA_TEST_DATABASE_URL`), the production dependencies (`langgraph-checkpoint-redis` on 3.12), or busybox. See [13](13-testing-and-evaluation.md).

---

## Option 1: hybrid (recommended for day-to-day)

### 1. Create `core/.env` for the containers

```ini
# used by the timescale image
POSTGRES_USER=routica
POSTGRES_PASSWORD=routica
POSTGRES_DB=routica_ai
# used by the redis container
REDIS_PASSWORD=devpassword
```

### 2. Start only the database and Redis

```bash
cd core
docker compose -f docker-compose.local.yml up -d postgres redis
# Postgres → localhost:5434, Redis → localhost:6395
```

### 3. Create `core/dev.env` for the app

Use the minimal `dev.env` from [10 · Config](10-config-and-credentials.md#minimal-devenv-for-local-work): `DATABASE_URL=postgresql+asyncpg://routica:routica@localhost:5434/routica_ai`, `REDIS_PORT=6395`, `REDIS_CHAT_DB=0`, a generated `BACKEND_API_KEY`, your `GROQ_API_KEY`, `BACKEND_URL=` (empty).

### 4. Run the app

```bash
source .venv/bin/activate
uvicorn app.main:app --reload --port 5000
# NOT `python app/main.py`: it references "main:app" and is broken
```

On startup you should see: `Connected to database` → `Running database migrations...` → `migrations completed` → `Background analysis scheduler started`.

```bash
curl http://localhost:5000/health          # {"status":"healthy",...}
open http://localhost:5000/docs            # FastAPI Swagger UI
```

> Without Zeek on your machine, the **flow-record** path works (no Zeek needed). A `meta.pcap` upload makes the names-capture Zeek run fail, which is only a **warning**: names then come from the name cache or not at all.

---

## Option 2: full Docker stack

`docker-compose.local.yml` **pulls** a pre-built image (`hasanyousufi/sofstica:device-activity-monitor-local`). You may need `docker login` with the team's Docker Hub account.

`.env` needs everything: the `POSTGRES_*`/`REDIS_PASSWORD` above **plus** the app vars, with container hostnames:

```ini
DATABASE_URL=postgresql+asyncpg://routica:routica@postgres:5432/routica_ai
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_CHAT_DB=0
BACKEND_URL=
BACKEND_API_KEY=<generated>
GROQ_API_KEY=<key>
TIMEZONE=Asia/Karachi
```

```bash
docker compose -f docker-compose.local.yml up -d
docker compose -f docker-compose.local.yml logs -f app
curl http://localhost:8050/health          # app is mapped to 8050 here
```

To **build** the image yourself, use `docker-compose.yml` (it has a `build:` section and reads `staging.env`, app on port 5000): `docker compose up -d --build`. The build pulls `hasanyousufi/sofstica:zeek-base` and `:suricata-base`.

Inside the container the code lives in **`/tmp`** and logs in **`/tmp/logs`**:
```bash
docker exec -it DAM-local-backend sh -c 'cd /tmp && ls logs'
```

---

## First end-to-end test (no router needed)

```mermaid
sequenceDiagram
    autonumber
    participant You
    participant API
    participant Sch as Scheduler
    participant DB
    You->>API: 1 register router TEST-0001
    You->>API: 2 POST /dhcp (fake lease)
    You->>API: 3 POST /flows (fake records, ts = now)
    Sch->>DB: 4 within 5 min: analyse → facts
    You->>DB: 5 check rows
    You->>API: 6 mint JWT → POST /api/v1/agent/chat
```

> Don't name your test router `PCAP-*`, `SYN-*` or `EVAL-*`: the scheduler skips those.

```bash
BASE=http://localhost:5000
SERIAL=TEST-0001

# 1. Register the router
curl -s -X POST $BASE/api/v1/router-info/ -H 'Content-Type: application/json' \
  -d "{\"serial_number\":\"$SERIAL\",\"model\":\"dev\",\"firmware_version\":\"0\"}"

# 2. DHCP lease (dnsmasq format: expiry mac ip hostname client-id). Must land BEFORE flows.
echo "$(( $(date +%s) + 86400 )) aa:bb:cc:dd:ee:01 192.168.1.23 Samsung-TV 01:aa:bb:cc:dd:ee:01" > /tmp/leases
curl -s -X POST $BASE/dhcp -F file=@/tmp/leases -F hostname=$SERIAL

# 3. Flow records (Zeek conn.log-shaped NDJSON)
python3 - <<'EOF' > /tmp/flows.ndjson
import json, time
now = time.time()
for i in range(10):
    print(json.dumps({"ts": now - 60 + i*5, "uid": f"Fdemo{i}", "id.orig_h": "192.168.1.23", "id.orig_p": 50000,
        "id.resp_h": "142.250.1.1", "id.resp_p": 443, "proto": "tcp", "conn_state": "ESTABLISHED",
        "duration": 5.0 * (i+1), "flow_first_seen": now - 60, "orig_bytes": 2000, "resp_bytes": 500000,
        "orig_ip_bytes": 2500, "resp_ip_bytes": 520000, "orig_pkts": 10, "resp_pkts": 360,
        "local_orig": True, "local_resp": False, "missed_bytes": 0}))
EOF
curl -s -X POST $BASE/flows -H "X-Hostname: $SERIAL" -H 'Content-Type: application/x-ndjson' \
  --data-binary @/tmp/flows.ndjson
ls logs/$SERIAL/                      # conn.log, dhcp_leases.json ...
```

**4–5.** Wait up to 5 minutes for the scheduler, then:
```sql
SELECT status, accepted_records FROM ingestion_batch ORDER BY created_at DESC LIMIT 3;
SELECT ts, device_id, bytes_downloaded FROM device_traffic_summary ORDER BY ts DESC LIMIT 5;
```
(Without a `meta.pcap` there are no site names, so bytes show as unnamed destinations. That is expected.)

### Mint a JWT for testing

```bash
export BACKEND_API_KEY=<same value as in dev.env>
TOKEN=$(python3 - <<'EOF'
import base64, os, time, jwt   # PyJWT, from requirements.txt
key = base64.b64decode(os.environ["BACKEND_API_KEY"])
now = int(time.time())
print(jwt.encode({"sub": "routica-backend", "iat": now, "exp": now + 3600}, key, algorithm="HS256"))
EOF
)
```

**6.** Ask the agent (needs `GROQ_API_KEY`):
```bash
curl -s -X POST $BASE/api/v1/agent/chat -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d "{\"message\":\"How many devices are connected and how much data did they use today?\",\"hostname\":\"$SERIAL\",\"chatId\":1}"

# streaming version
curl -N -X POST $BASE/api/v1/agent/chat/stream -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' -d "{\"message\":\"Top devices today\",\"hostname\":\"$SERIAL\",\"chatId\":1}"
```

For a richer dataset, the eval tooling seeds a 30-day synthetic household: `python scripts/eval/chat_eval.py seed --days 30 --host SYN-HOME-1012` (see [13](13-testing-and-evaluation.md)).

---

## Turning on security locally

Add to `dev.env`: `SECURITY_TRIAGE_ENABLED=True` (and optionally `SECURITY_RAW_CAPTURE_ENABLED=True`). Restart, then watch for `security cycle:` log lines every 15 minutes and open `http://localhost:5000/security/dashboard`.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| App crashes at import: "field required" | Missing `REDIS_*` / `BACKEND_URL` / `BACKEND_API_KEY` (they must exist, even if empty) |
| `BACKEND_API_KEY must decode to at least 32 bytes` | Generate it with the one-liner in [10](10-config-and-credentials.md) |
| Every chat request 500 "Backend API key not configured" | `BACKEND_API_KEY` is empty |
| Every chat fails, Redis "Cannot create index on db != 0" | Set `REDIS_CHAT_DB=0` |
| Chat 404 "Router not registered." | Register the serial via `/api/v1/router-info/` |
| Analysis "succeeds" but all bytes unattributed | DHCP leases missing, or posted after the flows |
| Migration error at startup | Read the alembic stderr in the log. The app won't start until it is fixed |

---

[← 10 · Config and credentials](10-config-and-credentials.md) · [Index](00-START-HERE.md) · **Next →** [12 · Build, deploy and CI](12-build-deploy-ci.md)
