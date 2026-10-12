# 06 · Analysis pipeline (raw data → facts, every 5 minutes)

[← 05 · API endpoints](05-api-endpoints.md) · [Index](00-START-HERE.md) · **Next →** [07 · Database](07-database.md)

---

This is the heart of the system: how files on disk become rows in TimescaleDB.

## The files on disk (`LOG_DIR`)

`LOG_DIR` is **not** an env var. It is `BASE_DIR/logs` in [`app/core/constant.py`](../core/app/core/constant.py):
- running from a checkout: `core/logs/`
- **in Docker: `/tmp/logs`** (the image's working dir is `/tmp`; see [14 · Traps](14-traps-and-gotchas.md))

```
LOG_DIR/
  <serial>.pcap                    accounting pcap (old pcap mode, usually absent)
  <serial>/meta.pcap               names capture: DNS + TLS/QUIC handshakes
  <serial>/lan.pcap                device-to-device capture
  <serial>/conn.log                flow_reporter records (POST /flows)
  <serial>/conn.<epoch>.log        a "claimed" batch being analysed / left by a failed run
  <serial>/dhcp_leases.txt|.json   POST /dhcp
  <serial>/name_cache.json         IP → name memory between windows (24 h TTL)
  analysis/<serial>/               (= ANALYSIS_DIR)
     zeek_logs/ zeek_meta_logs/ zeek_lan_logs/ intel/
     data_extraction_<epoch>.json  analyzer output per window (never cleaned up)
     failed_inputs/<UTC>-<name>    inputs that failed 3 times
```

---

## One scheduler tick

```mermaid
flowchart TB
    T["⏰ APScheduler 'periodic_analysis'<br/>every ANALYSIS_INTERVAL_MINUTES (5)"] --> P["pending_routers():<br/>glob LOG_DIR/*.pcap, */conn.log, */conn.*.log<br/>skip PCAP-/SYN-/EVAL- serials"]
    P --> POOL["scheduler_executor (5 threads)<br/>one job per router"]
    POOL --> RUN["analysis_runnerV2(serial)"]

    subgraph RUNNER["analysis_runnerV2 (app/services/zeek.py)"]
        S1["1. lan.pcap? → Zeek → append to conn.log, delete lan.pcap"]
        S2["2. Prepare Zeek intel file from threat_intel table (optional)"]
        S3["3. Leftover conn.<t>.log from failed runs → analyse each, oldest first"]
        S4["4. CLAIM conn.log: rename → conn.<now>.log (atomic)"]
        S5["5. _analyze_batch()"]
        S1 --> S2 --> S3 --> S4 --> S5
    end
    RUN --> S1

    subgraph BATCH["_analyze_batch"]
        B1{"flow records?"}
        B1 -- "no" --> B2["zeek -C -r serial.pcap (5-min timeout)"]
        B1 -- "yes" --> B3["skip Zeek: use reporter's records"]
        B4["meta.pcap? → zeek on it (for dns.log)<br/>failure = warning only"]
        B5["python zeek_analyzerv2.py ... (120 s timeout)<br/>→ data_extraction_<t>.json"]
        B6["write_chat_analytics_batch(): ONE DB transaction"]
        B7["save_raw_capture() if SECURITY_RAW_CAPTURE_ENABLED"]
        B8["commit OK → delete meta.pcap + claimed conn file"]
        B2 --> B4
        B3 --> B4
        B4 --> B5 --> B6 --> B7 --> B8
    end
    S5 --> B1
```

### Why "claim by rename"?
The router keeps appending to `conn.log` while the analysis runs. Renaming first means new records go into a fresh file. Nothing is lost and nothing is counted twice.

### Failure handling

```mermaid
flowchart LR
    F1["Zeek or analyzer fails"] --> C{"attempts for this batch<br/>≥ ANALYSIS_MAX_ATTEMPTS (3)?"}
    C -- "no" --> K["keep inputs, retry next tick"]
    C -- "yes" --> Q["move to analysis/<serial>/failed_inputs/<br/>log ERROR, router's new data flows again"]
    F2["DB write fails"] --> K2["keep inputs, retry FOREVER<br/>(not counted as an attempt)"]
```

The attempt counter lives in memory, so it resets when the app restarts.

---

## The exact commands

**Zeek** ([`zeek.py::run_zeek`](../core/app/services/zeek.py)), with `cwd` set to the logs subdir:
```bash
/opt/zeek/bin/zeek -C -r <abs pcap> LogAscii::use_json=T local [<intel loader>]
```
(Zeek's path is hard-coded, so running outside Docker needs Zeek at `/opt/zeek`.)

**Analyzer** (`run_data_extractionV2`):
```bash
python app/utils/zeek_analyzerv2.py \
  --dhcp LOG_DIR/<serial>/dhcp_leases.json -o ANALYSIS_DIR/<serial>/data_extraction_<t>.json \
  [-d zeek_logs] [--conn <claimed conn file>] [--meta-logs zeek_meta_logs] \
  [--meta-pcap LOG_DIR/<serial>/meta.pcap] --name-cache LOG_DIR/<serial>/name_cache.json
```
Its timeout is `ANALYZER_TIMEOUT_SECONDS` (default 120).

---

## Inside the analyzer (`zeek_analyzerv2.py`)

```mermaid
flowchart TB
    A["Load conn records<br/>(Zeek conn.log or flow_reporter conn.log)"] --> B{"Originator IP has a<br/>DHCP lease?"}
    B -- "yes, IPv4" --> L["local_connections ✅<br/>(counted for a device)"]
    B -- "no, private IP" --> UA["unattributed_ips / bytes<br/>(no device to blame)"]
    B -- "no, public IP and the far end spoke first" --> MID["midstream_connections<br/>(security only)"]
    B -- "IPv6" --> X["dropped"]
    L --> N["Name each destination"]
    subgraph Names["Name sources, best first"]
        N1["TLS SNI / QUIC SNI from meta.pcap<br/>(confidence 1.0)"]
        N2["DNS answer seen before the connection, within TTL<br/>(0.8)"]
        N3["name_cache.json from earlier windows<br/>(0.55, skipped if ambiguous)"]
    end
    N --> N1 --> N2 --> N3
    N3 --> P["Build per-device profile:<br/>traffic_summary, service_summary,<br/>activity_timeline, dns_activity,<br/>website_consumption (with minute_bytes)"]
    P --> O["JSON: window_metadata, overall_traffic_summary,<br/>local_device_profiles, local_responder_profiles,<br/>midstream_raw_connections, inbound_raw_connections"]
```

Notes:
- **Why SNI comes from our own reader (`handshake_sni.py`) and not Zeek:** the router's filter keeps only handshake segments, so Zeek never sees a SYN and won't parse TLS. Modern browsers also split the QUIC ClientHello over several packets. Our reader parses TLS ClientHellos and **decrypts QUIC Initial packets** (public keys, RFC 9001; needs `cryptography`).
- **Streaming is not detected here.** The analyzer only records `minute_bytes` (downloaded bytes per minute per domain). The chat tool `get_streaming_sessions` works out sessions at query time.
- The analyzer runs its analysis twice (`generate_report` and `save_json`). It is wasteful but harmless.

---

## The writer (`chat_analytics_service.write_chat_analytics_batch`)

```mermaid
sequenceDiagram
    autonumber
    participant W as Writer
    participant DB as TimescaleDB
    W->>W: window_start = capture_start floored to 5 min (UTC)
    W->>W: batch_id = uuid5(serial + capture_start + capture_end)
    W->>DB: BEGIN
    W->>DB: router_info by serial
    W->>DB: ingestion_batch[batch_id] committed already?
    alt already committed
        W-->>W: return 0 rows (idempotent retry)
    else new
        W->>DB: insert ingestion_batch (status=processing)
        W->>DB: match each device by (router_id, MAC) → device_profile
        W->>DB: merge overall_traffic_summary
        W->>DB: merge device_traffic_summary
        W->>DB: merge device_domain_activity
        W->>DB: merge dns_activity
        W->>DB: merge device_dns_domain_activity
        W->>DB: merge device_destination_activity
        W->>DB: ingestion_batch.status = committed, router.last_seen
        W->>DB: COMMIT
    end
```

- **Idempotent:** the same capture always gets the same `batch_id`, so a retry cannot double-count.
- **Two captures in one 5-minute window are merged:** counts are summed and ratios recomputed. A row from the same batch is a no-op.
- **Devices are matched by MAC, never by hostname.** A device with no `device_profile` row is rejected (counted in `rejected_records`).

---

[← 05 · API endpoints](05-api-endpoints.md) · [Index](00-START-HERE.md) · **Next →** [07 · Database](07-database.md)
