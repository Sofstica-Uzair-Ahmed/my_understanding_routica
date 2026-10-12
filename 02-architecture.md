# 02 · Architecture

[← 01 · Big picture](01-big-picture.md) · [Index](00-START-HERE.md) · **Next →** [03 · Repo tour](03-repo-tour.md)

---

## 1. Containers and processes

In production everything runs as **three containers** (see `docker-compose.yml` / `docker-compose.local.yml`):

```mermaid
flowchart TB
    subgraph Docker["Docker host"]
        subgraph APP["app container (port 5000)"]
            UV["uvicorn → FastAPI<br/>app.main:app (1 worker)"]
            AS["Analysis scheduler<br/>(APScheduler thread, 5 min)"]
            SS["Security scheduler<br/>(APScheduler thread, 15 min)<br/>only if SECURITY_TRIAGE_ENABLED"]
            POOL["Thread pool: 5 workers<br/>(one job per router)"]
            ZK["/opt/zeek/bin/zeek<br/>(subprocess)"]
            AN["python zeek_analyzerv2.py<br/>(subprocess)"]
            AS --> POOL --> ZK
            POOL --> AN
        end
        PG[("postgres container<br/>timescale/timescaledb:latest-pg16<br/>host port 5434")]
        RD[("redis container<br/>redis/redis-stack:latest<br/>host port 6395")]
    end
    UV --> PG
    POOL --> PG
    SS --> PG
    UV --> RD
```

Important: the API, both schedulers and the analysis threads all live in **one Python process**. There is no Celery and no separate worker container.

---

## 2. Layers inside the app

```mermaid
flowchart TB
    subgraph Routes["app/routes (thin HTTP layer)"]
        RF["router_file_controller<br/>/upload /flows /dhcp /lan"]
        RI["router_info_controller<br/>/api/v1/router-info"]
        AC["agent_controller<br/>/api/v1/agent/chat ..."]
        SC["security_controller<br/>/api/v1/security/findings<br/>/security/alerts, /security/dashboard"]
        HC["health_controller /health"]
        AUTH["agent_auth.py (JWT check)"]
    end

    subgraph Services["app/services (logic)"]
        ZEEKS["zeek.py<br/>analysis_runnerV2"]
        SCHED["scheduler.py"]
        CAS["chat_analytics_service.py<br/>(writer)"]
        DPS["device_profile_service.py"]
        LOS["lan_observation_service.py"]
        AGT["agent_service / agent_runner /<br/>agent_tools / agent_charts /<br/>agent_guard / agent_stream / chat_context"]
        SECS["security/*<br/>(detectors, triage, alerts...)"]
    end

    subgraph Utils["app/utils"]
        ZA["zeek_analyzerv2.py<br/>(runs as a subprocess)"]
        SNI["handshake_sni.py"]
    end

    subgraph Core["app/core"]
        CFG["config.py (Settings)"]
        DBM["db.py (2 engines)"]
        PRV["providers/ (Groq, OpenAI)"]
        CONST["constant.py (LOG_DIR...)"]
    end

    RF --> DPS & LOS
    AC --> AUTH
    SC --> AUTH
    AC --> AGT
    SC --> SECS
    SCHED --> ZEEKS --> ZA --> SNI
    ZEEKS --> CAS
    AGT --> PRV
    SECS --> PRV
    Services --> DBM
```

Convention: **routes stay thin, logic lives in `app/services`.**

---

## 3. Two database engines

From [`app/core/db.py`](../core/app/core/db.py):

| Engine | Driver | Used by | How to get a session |
|---|---|---|---|
| **Async** | `asyncpg` (`postgresql+asyncpg://...`) | HTTP handlers, chat tools | `get_db()` (FastAPI dependency) or `async with get_session()` |
| **Sync** | `psycopg2` (same URL minus `+asyncpg`) | Scheduler threads (analysis, security), Alembic | `with get_sync_session():` |

Rule: **anything the scheduler calls must use `get_sync_session()`.** Scheduler threads have no event loop.

---

## 4. Two schedulers

```mermaid
flowchart LR
    subgraph A["Analysis scheduler (always on)"]
        A1["every ANALYSIS_INTERVAL_MINUTES = 5"] --> A2["pending_routers():<br/>scan LOG_DIR"] --> A3["analysis_runnerV2(serial)<br/>per router, 5 in parallel"] --> A4["facts written to DB"]
    end
    subgraph S["Security scheduler (opt-in)"]
        S1["every SECURITY_TRIAGE_INTERVAL_MINUTES = 15"] --> S2["run_detection → run_screening →<br/>run_triage → run_coverage →<br/>sweep_resolved → push_pending_findings"]
        S3["every 12 h: refresh threat feeds<br/>(abuse.ch)"]
    end
    A4 -. "security reads the same facts" .-> S2
```

Both skip **evaluation routers**, whose serials start with `PCAP-`, `SYN-` or `EVAL-`. Test harnesses drive those themselves.

---

## 5. Startup sequence (order matters!)

From [`app/main.py`](../core/app/main.py) `lifespan()`:

```mermaid
sequenceDiagram
    autonumber
    participant M as main.py lifespan
    participant DB as Postgres
    participant AL as alembic (subprocess)
    participant SCH as Schedulers
    M->>DB: init_db(): import all models, Base.metadata.create_all
    M->>DB: check_db_health(): SELECT 1 (only logs on failure)
    M->>AL: python -m alembic upgrade head (failure = app does not start)
    M->>SCH: start analysis scheduler (5 min)
    alt SECURITY_TRIAGE_ENABLED
        M->>SCH: start security scheduler (15 min + feed refresh)
    end
    M->>M: init Langfuse client (warning only on error)
    Note over M: app serves requests
    M->>SCH: shutdown(wait=True) on exit, then thread pool, then DB
```

**Why `create_all` runs before migrations:** the migrations (hypertables, aggregates, extra columns) assume the tables already exist.

---

## 6. Data flows at a glance

```mermaid
flowchart LR
    subgraph IN["Ingest (router → API → disk/DB)"]
        U1["/upload → LOG_DIR/serial.pcap,<br/>serial/meta.pcap, serial/lan.pcap"]
        U2["/flows → LOG_DIR/serial/conn.log"]
        U3["/dhcp → dhcp_leases.json + device_profile"]
        U4["/lan → lan_neighbor, lan_server_sighting,<br/>lan_port_forward, router_dns_rebind_event"]
    end
    subgraph PROC["Process (every 5 min)"]
        P1["Zeek"] --> P2["Analyzer"] --> P3["Writer: canonical facts"]
    end
    subgraph OUT["Use"]
        O1["Chat tools (read facts)"]
        O2["Security detectors (read facts + raw rows + LAN tables)"]
    end
    U1 & U2 & U3 --> P1
    P3 --> O1 & O2
    U4 --> O2
```

---

## 7. Outbound calls the service makes

| To | When | Auth | Code |
|---|---|---|---|
| `{BACKEND_URL}/agent/chat` | Once per chat turn (answer or "AI unavailable") | Bearer JWT (HS256, key = base64-decoded `BACKEND_API_KEY`, 5-min expiry) | `agent_chat_callback_service.py` |
| `{BACKEND_URL}{SECURITY_ALERT_PATH}` (default `/api/v1/notifications/security-alert`) | Each new or escalated finding | Same JWT | `security/alert_callback.py` |
| Groq API | Chat turns, screening, triage | `GROQ_API_KEY` | `core/providers/groq.py` |
| Langfuse | Traces (optional) | `LANGFUSE_*` keys | `chat_tracing.py`, `security/tracing.py` |
| iptoasn.com, abuse.ch, RDAP/WHOIS, crt.sh, Spamhaus DNS | Security enrichment (no keys) | none | `security/asn_lookup.py`, `threat_feeds.py`, `external_lookup.py` |

---

[← 01 · Big picture](01-big-picture.md) · [Index](00-START-HERE.md) · **Next →** [03 · Repo tour](03-repo-tour.md)
