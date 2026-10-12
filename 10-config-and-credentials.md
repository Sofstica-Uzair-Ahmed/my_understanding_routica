# 10 · Configuration and credentials

[← 09 · Security triage](09-security-triage.md) · [Index](00-START-HERE.md) · **Next →** [11 · Running locally](11-running-locally.md)

---

## Where config comes from

```mermaid
flowchart LR
    ENV["Process environment variables<br/>(highest priority)"] --> S["Settings()<br/>app/core/config.py"]
    DEV["dev.env file<br/>(hard-coded name, read from the working dir)"] --> S
    DEF["Defaults in the Settings class"] --> S
    OS["os.getenv() in some modules<br/>(GEOIP_*, SECURITY_EXTERNAL_LOOKUP_*,<br/>ANALYZER_TIMEOUT_SECONDS, ANALYSIS_MAX_ATTEMPTS)"]
```

| How you run it | Env source |
|---|---|
| `uvicorn` on your machine | `dev.env` in the current directory (copy from [`dev.env.example`](../core/dev.env.example)) |
| `docker-compose.yml` (staging) | `staging.env` → container env |
| `docker-compose.local.yml` (local/deployed) | `.env` → container env |

All three files are **git-ignored**. Never commit real secrets. Unknown keys are ignored (`extra="ignore"`).

---

## 🔑 Credentials you need

| Secret | Required? | What for | Where to get it |
|---|---|---|---|
| `DATABASE_URL` | ✅ (default is `""`, but `app/core/db.py` builds the engine at import, so an empty or invalid URL fails there; a dummy URL is fine for unit tests) | Postgres + TimescaleDB, `postgresql+asyncpg://user:pw@host:port/db` | Your local compose DB, or ask the team for staging |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` | ✅ **crash at import** if missing | Chat history (LangGraph checkpointer) | Local compose Redis, or the team |
| `BACKEND_URL` | ✅ must be present (may be **empty**) | Where callbacks and alerts go. Empty = callbacks skipped | The backend team |
| `BACKEND_API_KEY` | ✅ must be present (may be **empty**) | Shared HS256 secret: verifies incoming JWTs and signs callbacks. Base64, decoding to **≥ 32 bytes** | The backend team (must match theirs). Generate a local one, see below |
| `GROQ_API_KEY` | needed for chat and security | LLM calls | console.groq.com / the team |
| `OPENAI_API_KEY` | optional | Only if `DEFAULT_PROVIDER=openai` | The team |
| `LANGFUSE_PUBLIC_KEY` / `_SECRET_KEY` / `_HOST` | optional | Tracing | The team's Langfuse |
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | for the compose DB container | Used by the timescale image (in the same `.env`) | You choose locally |

> ⚠️ If `BACKEND_API_KEY` is empty, **every chat request returns 500** "Backend API key not configured", because the JWT can't be checked. For local chat testing, generate a key.

Generate a valid key:
```bash
python -c "import base64,os;print(base64.b64encode(os.urandom(32)).decode())"
```

**CI/CD secrets** (GitHub repo settings, names only): `ANTHROPIC_API_KEY` (AI PR review), `LOCAL_DOCKER_USERNAME`, `LOCAL_DOCKER_PASSWORD`, `LOCAL_SSH_IP`, `LOCAL_SSH_USERNAME`, `LOCAL_SSH_PASSWORD`. See [12](12-build-deploy-ci.md).

---

## Minimal `dev.env` for local work

```ini
DATABASE_URL=postgresql+asyncpg://routica:routica@localhost:5434/routica_ai
REDIS_HOST=localhost
REDIS_PORT=6395
REDIS_PASSWORD=devpassword
REDIS_CHAT_DB=0              # redis-stack images refuse search indexes on db != 0
BACKEND_URL=                 # empty = skip callbacks
BACKEND_API_KEY=<output of the python one-liner above>
GROQ_API_KEY=<your key>
TIMEZONE=Asia/Karachi        # the household's IANA zone; decides what "today" means
ANALYSIS_INTERVAL_MINUTES=5  # keep at 5
```

---

## All settings, grouped

### Service basics
| Var | Default | Notes |
|---|---|---|
| `DEBUG` | True | Dev default. Also turns on SQL echo |
| `PORT` | 5000 | |
| `ALLOWED_ORIGINS` / `ALLOWED_HOSTS` | `["*"]` | **Not actually used.** CORS is hard-coded to `*` |
| `MAX_WORKER_THREADS` | 8 | Request thread pool |
| `SCHEDULER_MAX_WORKERS` | 5 | Routers analysed in parallel |
| `TIMEZONE` | UTC | **Set it.** Leaving UTC shifts "yesterday" for non-UTC homes |

### Analysis
| Var | Default | Notes |
|---|---|---|
| `ANALYSIS_INTERVAL_MINUTES` | 5 | Also the stored window size. Don't change in production |
| `ANALYZER_TIMEOUT_SECONDS` | 120 | `os.getenv` in `zeek.py` |
| `ANALYSIS_MAX_ATTEMPTS` | 3 | Then inputs move to `failed_inputs/` |

### LLM and chat
| Var | Default | Notes |
|---|---|---|
| `DEFAULT_PROVIDER` | groq | or `openai` |
| `DEFAULT_MODEL` | `qwen/qwen3.8-27b` | An unknown model silently falls back to the provider's first |
| `LLM_REQUEST_TIMEOUT_SECONDS` | 120 | Must be > 0. Without it, a call can hang forever |
| `MAX_HISTORY_MESSAGES` | 20 | Conversation window |
| `AGENT_RECURSION_LIMIT` | 60 | Graph steps per turn |
| `CHAT_MESSAGE_MAX_CHARS` | 2000 | Longer → 422 |
| `CHAT_CONTEXT_BUDGET_TOKENS` | 64000 | Size sent to the model |
| `REDIS_CHAT_DB` | 2 | **Set 0 on redis-stack images** |
| `STREAMING_MIN_KBPS` / `_GAP_MINUTES` / `_MIN_MINUTES` | 250 / 2 / 2 | What counts as watching video |
| `CHAT_TRACE_DB_QUERIES` / `_MAX_ROWS` | False / 50 | Trace SQL into Langfuse (eval only) |

### Security (all optional)
| Var | Default |
|---|---|
| `SECURITY_TRIAGE_ENABLED` | False |
| `SECURITY_TRIAGE_INTERVAL_MINUTES` | 15 |
| `SECURITY_TRIAGE_MODEL` | `qwen/qwen3.8-27b` (must be in Groq's list; set it empty to use `DEFAULT_MODEL` / OpenAI) |
| `SECURITY_TRIAGE_TEMPERATURE` | 0.1 |
| `SECURITY_ALERT_PATH` | `/api/v1/notifications/security-alert` |
| `SECURITY_ALERT_MIN_SEVERITY` | low |
| `SECURITY_ALERT_COOLDOWN_HOURS` / `SECURITY_SIGNAL_COOLDOWN_HOURS` | 24 / 6 |
| `SECURITY_BASELINE_DAYS` / `SECURITY_NEWDOMAIN_LOOKBACK_DAYS` | 14 / 30 |
| `SECURITY_RESOLVE_AFTER_HOURS` | 8 (must stay > the 6 h cooldowns) |
| `THREAT_FEED_REFRESH_HOURS` | 12 |
| `SECURITY_LLM_DETECTION_ENABLED` | True |
| `SECURITY_RAW_CAPTURE_ENABLED` / `SECURITY_SCREENING_ENABLED` / `SECURITY_AGENTIC_TRIAGE_ENABLED` | False / False / False |
| `SECURITY_SCREEN_LOOKBACK_HOURS` / `_MIN_INTERVAL_MINUTES` / `_MAX_DEVICES_PER_CYCLE` | 6 / 60 / 40 |
| `GEOIP_ASN_URL`, `GEOIP_ASN_CACHE`, `GEOIP_ASN_TTL_DAYS`, `GEOIP_PTR_ENABLED`, `GEOIP_PTR_TIMEOUT` | see `dev.env.example` (`os.getenv`) |
| `SECURITY_EXTERNAL_LOOKUP_ENABLED`, `SECURITY_WHOIS_/RDAP_/CRTSH_TIMEOUT`, `SECURITY_EXTERNAL_LOOKUP_CACHE_TTL` | see `dev.env.example` (`os.getenv`) |

### Router-side config
This lives on the router in `scripts/router_scripts/config` (`ROUTICA_SERVER`, `LAN_CAPTURE`…). See [04](04-router-side.md).

---

## Rule when adding a variable
Add it to `Settings` in `config.py` (or `os.getenv` in its module) **and** to `dev.env.example` with a placeholder and comment.

---

[← 09 · Security triage](09-security-triage.md) · [Index](00-START-HERE.md) · **Next →** [11 · Running locally](11-running-locally.md)
