# 14 · Traps and gotchas

[← 13 · Testing and evaluation](13-testing-and-evaluation.md) · [Index](00-START-HERE.md) · **Next →** [15 · Map of existing docs](15-existing-docs-map.md)

---

Each item here cost the team real debugging time. Most come from `core/CLAUDE.md`, where the full story is told. Skim them now and come back when something looks weird.

## 🔴 Data silently wrong

| Trap | Why it bites | What to do |
|---|---|---|
| **DHCP must land before analysis** | The analyzer keeps a connection only if its source IP has a DHCP lease, and devices are matched by `(router_id, MAC)`. With no leases, analysis "succeeds" and every byte is unattributed. `dhcp_monitor.sh` only re-posts when the file changes | Check `LOG_DIR/<serial>/dhcp_leases.json` exists; after a redeploy, make the router re-post (touch the lease file) |
| **Logs are in `/tmp/logs`, not `/app/logs`** | The image's `WORKDIR` is `/tmp`; compose mounts `/app/logs`, which the app never uses | Look in `/tmp/logs` in the container. Leases, name cache and unclaimed flows are lost on redeploy |
| **Never sum `duration` from flow records** | `flow_reporter` sends a record every 5 s with `duration` = lifetime so far. Summing turned 40 min into 9,620 min | Use `minute_bytes`, `connection_seconds()`, `get_streaming_sessions`, `active_minutes` |
| **A stalled scheduler makes one giant capture** | After an outage the backlog is analysed as ONE capture stamped at its first window (a real 45-hour row exists) | `active_minutes` caps a capture at 6 windows and flags it. Splitting by record time at ingest is not done yet |
| **`_daily` aggregates are UTC days** | A "day" would be wrong for non-UTC homes | Chat reads base tables with `_local_bucket` or rolls `_hourly` into local days |
| **`TIMEZONE` defaults to UTC** | "Today" and "yesterday" shift by the offset | Set the household's IANA zone |
| **`device_profile.lease_time` is the lease EXPIRY** | It was once shown as "lease timestamp", so a device whose lease ran out read as connected | Use `lease_expires`/`lease_active` from `get_connected_devices` |
| **`dns_activity.failed_queries` ≠ "sites that failed to load"** | It counts every non-NOERROR lookup, including `.arpa`, `.lan` and missing answers | The agent uses `failed_lookups`; security still reads `failed_queries` |
| **IPv6 is dropped entirely** | `_is_ipv4` checks `'.' in ip and ':' not in ip` | Known limitation |
| **A pcap AND flow records from one router** | The runner analyses the flow records, and the scheduler then deletes the pcap unanalysed | Don't run both collectors |

## 🟠 App won't start or requests fail

| Trap | Fix |
|---|---|
| Missing `REDIS_*`, `BACKEND_URL`, `BACKEND_API_KEY` → crash at import | Define them (empty is OK for the backend vars) |
| `BACKEND_API_KEY` empty → every chat 500 | Generate a base64 key of 32+ bytes |
| redis-stack: "Cannot create index on db != 0" → every chat fails | `REDIS_CHAT_DB=0` |
| `python app/main.py` doesn't work | Use `uvicorn app.main:app` |
| Migration order: `create_all` must run **before** `alembic upgrade head` | Don't reorder `main.py`. Guard model-less tables with `to_regclass` |
| A raw SQL insert fails NOT NULL | Models have Python-side defaults only, so supply every column |

## 🟡 Container and router quirks

| Trap | Fix |
|---|---|
| A helper script in the container's `/tmp` named like a stdlib module (`inspect.py`) **shadows Python's module** for the app and the analyzer | Never do that. Check with `python -c "import sys,pathlib;print([p.name for p in pathlib.Path('/tmp').glob('*.py') if p.stem in sys.stdlib_module_names])"` |
| `test` inside the app image is a Python script, not the shell builtin | `docker exec <c> sh -c 'test -f /path'` |
| The router has no `setsid`/`nohup`/`timeout`; SSH-started scripts die with the session | `start-stop-daemon -S -b -m -p /var/run/X.pid -x /bin/sh -- script` |
| `healthcheck.sh` can report a dead collector as running | Verify by pidfile |
| Restarting `flow_reporter.sh` loses its unsent bytes | Restart just after an upload |
| Every backend restart drops in-flight 10-s uploads | Expected; avoid restarting needlessly |

## 🔵 LLM behaviour (chat)

- **Every LLM client needs a timeout.** LangChain passes `timeout=None` (wait forever), and one hung call froze security for 2.6 days. Always build clients through `GroqProvider`/`OpenAIProvider`.
- **qwen3.8 can't reliably write the letters "nvr"** into a tool argument, so devices it can't name are offered by `device_id`.
- **qwen3.8 repeats identical tool calls** (once ~20 times). `guard_repeated_calls` and `repeated_call_brake` handle it.
- **Prompt rules are requests, not guarantees.** Anything that must hold (no serial leak, no prompt leak, the unattributed footnote, chart honesty) is enforced in code (`agent_guard`, `agent_charts`).
- **Hand the model numbers; don't make it derive them.** Tools return `device_count`, `period_totals` and so on. It once answered 5 and 3-vs-7 to the same question.
- **Wording in an instruction is copied into answers.** Never write a sample answer you'd not want sent.
- **One chat id on two routers:** history is keyed `serial:chatId`, so they don't mix.

## 🟣 Security-specific

- **Enrichment happens at triage *read* time.** Re-triaging an old finding reads its stored signals and does not re-enrich them.
- The 50-signal prompt cap must put **actionable signals before context** ones (`_signals_for_device` ORDER BY). Don't revert it to newest-first.
- `device_profile` is keyed by **MAC, never hostname**: two devices can share a name.
- Turning screening ON **changes the gate**: medium bands become context-only.
- `screening_active()` needs **both** `SECURITY_SCREENING_ENABLED` and `SECURITY_RAW_CAPTURE_ENABLED`.
- A new detector must be added to **exactly one** gate set, or `test_gate_covers_every_detector.py` fails.
- A new synthetic serial must use the `PCAP-`/`SYN-`/`EVAL-` prefixes, or production schedulers will process it and push its findings.
- Changing what a fact column means changes security detection too. See "Security consumers" in `docs/chat-analytics-data-design.md`.

## ⚪ Repo hygiene

- `*.md` is git-ignored, so use `git add -f`.
- No linter or formatter is configured. The convention is `black`/`ruff format` with sorted imports, on the honour system.
- No tests run in CI.

---

[← 13 · Testing and evaluation](13-testing-and-evaluation.md) · [Index](00-START-HERE.md) · **Next →** [15 · Map of existing docs](15-existing-docs-map.md)
