# 16 · Glossary and cheat sheet

[← 15 · Map of existing docs](15-existing-docs-map.md) · [Index](00-START-HERE.md)

---

## Glossary

| Term | Meaning here |
|---|---|
| **hostname** | In code, almost always the **router serial number** (e.g. `RQ01000039`), not a device name. Maps to `router_info.serial_number` |
| **serial / router_id** | Serial = external id; `router_id` = internal integer PK |
| **device** | A row in `device_profile`, identified by **MAC** per router |
| **pcap** | A packet-capture file (tcpdump output) |
| **meta.pcap / names capture** | DNS + TLS/QUIC handshakes only, used to learn **which site** each connection is for |
| **flow records / flow_reporter** | Byte counters per connection from the router's kernel, shaped like Zeek's `conn.log` |
| **Zeek** | Open-source network analyzer that turns pcaps into logs (`conn.log`, `dns.log`) |
| **SNI** | Server Name Indication: the site name sent in the clear at the start of a TLS/QUIC connection |
| **QUIC Initial** | The first QUIC packets. Their encryption keys are public, so we can decrypt them to read the SNI |
| **DHCP lease** | The router's record "MAC X has IP Y until time Z". Needed to attribute traffic |
| **unattributed** | Traffic from an IP with no lease, so we can't say which device |
| **window** | A 5-minute slot. Every fact row's `ts` is a window start |
| **analysis window vs conversation window** | Two unrelated things: the 5-min data slot, vs the ~20 chat messages kept as history |
| **canonical facts** | The 7 analytics tables (`overall_traffic_summary`, `device_traffic_summary`, `dns_activity`, `device_domain_activity`, `device_dns_domain_activity`, `device_destination_activity`, `ingestion_batch`) |
| **ingestion_batch** | One analysed capture. Its deterministic UUID makes retries idempotent |
| **hypertable / continuous aggregate** | TimescaleDB time-partitioned table / auto-maintained rollup (`_hourly`, `_daily`) |
| **tool (chat)** | A Python function the LLM may call (9 of them in `agent_tools.py`) |
| **checkpointer** | LangGraph's saved chat state, in Redis |
| **guard** | Code that fixes or blocks an LLM reply (`agent_guard.finalize_reply`) |
| **signal** | One detector hit (`security_signals`) |
| **finding** | A triaged conclusion the customer may see (`security_findings`) |
| **gate** | Rules deciding which signals may send a device to the LLM |
| **floor** | A minimum severity or verdict the LLM can't go below |
| **screening** | Optional LLM pre-review of each device's destinations |
| **agentic investigator** | Optional LLM with 13 tools that investigates a device step by step |
| **coverage findings** | Router-level findings with no LLM (rogue DHCP, exposed ports, inbound scans) |
| **evaluation routers** | Serials `PCAP-*`, `SYN-*`, `EVAL-*`, ignored by the production schedulers |
| **held-out captures** | Three malware captures nothing may be tuned on |
| **C2 / beaconing** | Malware "command and control": regular check-ins to an attacker's server |
| **IOC** | Indicator of compromise: a known-bad domain, IP or hash |
| **ATT&CK** | MITRE's catalogue of attacker tactics and techniques |
| **Groq** | The LLM API provider; model `qwen/qwen3.8-27b` |
| **Langfuse** | LLM tracing and observability service (optional) |
| **backend** | The other team's service: users, the mobile app, push notifications |

---

## Cheat sheet

### Setup and run
```bash
cd core && python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt pytest
cp dev.env.example dev.env      # then edit (see page 10)
docker compose -f docker-compose.local.yml up -d postgres redis   # DB :5434, Redis :6395
uvicorn app.main:app --reload --port 5000
curl localhost:5000/health ; open http://localhost:5000/docs
```

### Tests
```bash
export DATABASE_URL="postgresql+asyncpg://u:p@localhost/x" REDIS_HOST=x REDIS_PORT=1 REDIS_PASSWORD=x BACKEND_URL= BACKEND_API_KEY=
pytest app/tests -q
sh app/tests/test_flow_reporter.sh
```

### Secrets
```bash
python -c "import base64,os;print(base64.b64encode(os.urandom(32)).decode())"   # BACKEND_API_KEY
```

### Database
```bash
psql -h localhost -p 5434 -U <user> <db>
alembic upgrade head            # also runs automatically at startup
alembic revision -m "slug"      # then write upgrade/downgrade by hand
pytest app/tests/test_migration_chain.py
```

### Docker
```bash
docker compose -f docker-compose.local.yml up -d && docker compose -f docker-compose.local.yml logs -f app
docker exec -it DAM-local-backend sh -c 'cd /tmp && ls logs'              # code + logs live in /tmp
docker exec -it DAM-local-backend sh -c 'cd /tmp && python scripts/eval/evaluate_chat.py run --suite household'
```

### Data flow in one line
```
router → /upload /flows /dhcp /lan → LOG_DIR → scheduler (5 min) → zeek + zeek_analyzerv2 → write_chat_analytics_batch → TimescaleDB → chat tools / security detectors
```

### Key files
| What | File |
|---|---|
| Startup | `app/main.py` |
| Settings | `app/core/config.py`, `dev.env.example` |
| Paths | `app/core/constant.py` |
| Ingest | `app/routes/router_file_controller.py` |
| 5-min job | `app/services/scheduler.py` → `app/services/zeek.py` |
| Analyzer | `app/utils/zeek_analyzerv2.py`, `app/utils/handshake_sni.py` |
| DB writer | `app/services/chat_analytics_service.py` |
| Chat | `app/routes/agent_controller.py`, `app/services/agent_service.py`, `agent_runner.py`, `agent_tools.py` |
| Security | `app/services/security/scheduler.py`, `signal_service.py`, `detectors.py`, `triage_service.py` |

### Log lines worth knowing
| Line | Means |
|---|---|
| `Background analysis scheduler started` | Startup OK |
| `Reply guard ...` | A chat reply was changed by the guard |
| `security cycle: N signals, ...` | A security cycle finished |
| ERROR about a skipped security cycle | The previous cycle is hung, so investigate |
| ERROR moving inputs to `failed_inputs` | A batch failed 3 analyses |
| `watcher alive` | The router's DHCP/RA watcher heartbeat |

---

🎉 **You've reached the end.** Next steps: run the [first end-to-end test](11-running-locally.md#first-end-to-end-test-no-router-needed), then read `core/CLAUDE.md` once more. It will read very differently now.

[← 15 · Map of existing docs](15-existing-docs-map.md) · [Index](00-START-HERE.md)
