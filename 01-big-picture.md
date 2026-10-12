# 01 · The big picture

[← Index](00-START-HERE.md) · **Next →** [02 · Architecture](02-architecture.md)

---

## What is Routica AI?

Routica sells home routers (OpenWrt firmware). **Routica AI** is the cloud service, in this repo, that makes those routers smart:

1. The router quietly records **who in the house talks to what on the internet**: device, destination, bytes and time.
2. Every 5 minutes this service crunches that into tidy facts per device, such as "the Samsung TV downloaded 1.2 GB from netflix.com between 20:00 and 20:05".
3. Two AI features read those facts:

| Product | What the customer gets | Code |
|---|---|---|
| 💬 **Talk to Data** (chat) | Ask in plain English: *"Which device used the most data yesterday?"*, *"How long was the kids' iPad on YouTube?"*, *"Show me a pie chart of websites this week"* | `app/services/agent_*.py`, `agent_tools.py` |
| 🛡️ **Threat Detection** (security triage) | An alert on the phone: *"Your laptop keeps contacting an unfamiliar server in a pattern typical of malware"* | `app/services/security/` |

The service has **no UI of its own** (except two small security HTML pages for ops). The customer uses a **mobile app**, which talks to a **separate backend** owned by another team. That backend calls this service and receives answers and alerts back.

---

## The four actors

```mermaid
flowchart TB
    Router["🛜 Router<br/>(OpenWrt, shell scripts)<br/>sends raw data"]
    Core["⚙️ Routica AI core<br/>(this repo: FastAPI + Zeek + TimescaleDB + Redis)"]
    Backend["🏢 External backend<br/>(another team; owns users, phones, push)"]
    LLM["🧠 Groq LLM API<br/>(qwen/qwen3.8-27b)"]

    Router -->|"no auth; identifies itself by serial number"| Core
    Backend -->|"chat questions, findings API<br/>(JWT signed with BACKEND_API_KEY)"| Core
    Core -->|"chat answers + security alerts<br/>(JWT signed with the same key)"| Backend
    Core <-->|"GROQ_API_KEY"| LLM
```

---

## A story: the life of one byte

Follow one Netflix stream from the TV to an answer in the app.

```mermaid
sequenceDiagram
    autonumber
    participant TV as Samsung TV
    participant R as Router scripts
    participant API as FastAPI (/flows, /upload, /dhcp)
    participant Disk as LOG_DIR files
    participant S as Scheduler (5 min)
    participant Z as Zeek + analyzer
    participant DB as TimescaleDB
    participant U as User (mobile app)
    participant A as Chat agent

    TV->>R: streams video from netflix.com
    R->>API: POST /dhcp (lease: TV = 192.168.1.23, MAC aa:bb..)
    R->>API: POST /flows every 30 s (byte counters per connection)
    R->>API: POST /upload every 10 s (meta.pcap: DNS + TLS handshakes)
    API->>Disk: append to logs/<serial>/conn.log, meta.pcap
    S->>Disk: every 5 min: anything new?
    S->>Z: run Zeek on meta.pcap, analyzer on conn.log
    Z->>Z: "192.168.1.23 → 52.x.x.x is netflix.com (from TLS SNI)"
    Z->>DB: write per-device facts for this 5-min window
    U->>A: "How long did the TV stream Netflix tonight?"
    A->>DB: get_streaming_sessions(device="TV", ...)
    A->>U: "About 1 h 40 min, from 20:05 to 21:45"
```

Key ideas in that story:

- **The router serial number is the identity everywhere.** In the code it is called `hostname` (confusing, but consistent). It maps to `router_info.serial_number` → `router_id`.
- **DHCP leases matter.** The analyzer only counts traffic from IPs that have a known DHCP lease. No leases means traffic is counted as "unattributed" (see [14 · Traps](14-traps-and-gotchas.md)).
- **Names come from TLS handshakes (SNI) first, DNS second.** Most traffic is encrypted, but the first packet of a TLS or QUIC connection says which site it is for.
- **Everything is chopped into 5-minute windows.** Each database row has a `ts` floored to a 5-minute boundary.

---

## Talk to Data in one picture

```mermaid
flowchart LR
    Q["'Which device used the most<br/>data yesterday? pie chart'"] --> AG["LangChain agent<br/>+ system prompt"]
    AG -->|"picks tools"| T1["get_device_usage"]
    AG --> T2["get_chart"]
    T1 --> DB[("TimescaleDB")]
    T2 --> T1
    AG --> G["Reply guards<br/>(redact serial, block prompt leaks,<br/>footnote unattributed bytes)"]
    G --> OUT["Answer text + chart spec<br/>(app draws the chart)"]
```

## Threat Detection in one picture

```mermaid
flowchart LR
    F[("Facts in DB")] --> DET["~40 deterministic detectors<br/>(beaconing, scans, exfil, DNS tricks...)"]
    DET -->|"signals"| GATE{"Serious enough?<br/>(gate)"}
    GATE -->|"yes"| LLM["LLM triage:<br/>what does it mean?<br/>write a plain sentence"]
    GATE -->|"no"| CTX["kept as context only"]
    LLM --> FIND["Finding<br/>(severity, title, message, steps)"]
    FIND --> PUSH["POST to backend → phone"]
```

Design rule from the team: **detectors decide WHO gets looked at, the LLM decides WHAT IT MEANS.** The LLM cannot lower a severity below the floor a detector sets.

---

## Tech stack at a glance

| Layer | Tech |
|---|---|
| Language | Python 3.12 |
| Web | FastAPI 0.109, uvicorn |
| Traffic parsing | Zeek 7.0.0 (built from source), custom analyzer `zeek_analyzerv2.py`, custom SNI/QUIC reader |
| Database | PostgreSQL 16 + TimescaleDB (hypertables, continuous aggregates); SQLAlchemy 2 + Alembic |
| Chat memory | Redis Stack (RediSearch) via `langgraph-checkpoint-redis` |
| AI | LangChain 1.0 / LangGraph 1.0, Groq API (`qwen/qwen3.8-27b`), OpenAI optional |
| Scheduling | APScheduler (thread-based) |
| Observability | Langfuse (optional) |
| Packaging | Docker (zeek-base → app image), GitHub Actions, SSH deploy to a Windows host |

---

[← Index](00-START-HERE.md) · **Next →** [02 · Architecture](02-architecture.md)
