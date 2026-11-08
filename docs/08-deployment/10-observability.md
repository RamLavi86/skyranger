# 10 Monitoring the Gateway Itself

Binding: [api.md](../00-contracts/api.md) `GET /system/health` (unauthenticated, `status: ok|degraded|down`, `checks` map, no tag data), `GET /system/info`; [messages.md](../00-contracts/messages.md) `status` topic with Will; reserved `_system` tag root ([README.md](../00-contracts/README.md), `_system/connections/{name}/state` in tag-model.md); [02-containers.md](02-containers.md) (agent, journald, port 9100 placeholder).

A SCADA gateway that silently stops is worse than one that fails loudly. This document defines what the gateway reports about itself, who looks at it, and what happens when nobody hears anything.

## 1. Four signals, one rule

| Signal | Mechanism | Consumer |
|---|---|---|
| **Health** (is it working right now) | `GET /api/v1/system/health`, container health checks | agent, load balancers, fleet, HMI banner |
| **Metrics** (how is it trending) | Prometheus text format on a metrics endpoint | agent, optional customer Prometheus, fleet summary |
| **Logs** (what happened) | JSON to stdout, journald | `skyrangerctl logs`, support bundle, optional forwarding |
| **Heartbeat** (is anybody home) | agent to fleet every 60 s, plus optional external ping | central, third-party dead man's switch |

**Rule: every important failure must be visible in at least two independent places**, one of which does not depend on the thing that failed. For example, the gateway's HMI is a poor place to learn that the gateway is dead.

## 2. Health

### 2.1 The existing endpoint
`GET /api/v1/system/health` is as in api.md: `{"status", "checks": {"database": "ok", "broker": "ok", "drivers": "degraded"}}`. Semantics we require (these are definitions of the contract's existing values, not changes):

- `ok`: all checks ok.
- `degraded`: the gateway is doing its job but something is wrong (a driver connection down, spool growing, clock unsynced). It is up and serving.
- `down`: the process cannot do its job (no config loaded, DB unavailable and no snapshot). In practice the endpoint is unreachable, which the agent treats the same.

### 2.2 Checks we need (additive keys in `checks`)
api.md fixes the shape of `checks` but not the key set beyond examples. We ask for these additional keys (additive, tolerated by clients that ignore unknown fields):

| Key | `ok` | `degraded` |
|---|---|---|
| `database` | queries succeed under 1 s | slow (over 1 s) or running from the config snapshot (T2) |
| `broker` | connected, subscribed | reconnecting |
| `drivers` | all enabled connections `connected` | at least one enabled connection not `connected`; names are in `/connections`, not here (no detail leakage on an unauthenticated endpoint) |
| `historian` | writer queue below 50%, spool empty | spool non-empty or queue above 50% |
| `clock` | chrony synchronized, offset under 100 ms | unsynchronized or offset over 1 s ([11](11-time-and-power.md)) |
| `disk` | data volume under 85% | over 85% |
| `config` | revision loaded | running from snapshot |

The unauthenticated endpoint returns no tag data and no names, only a status per check (as api.md requires). Details live behind auth in `GET /connections` and in metrics.

### 2.3 Gateway health as tags (alarmable, visible in the HMI)
Proposal (additive, needs tag-engine agreement): `skyrangerd` and the agent publish gateway vitals as ordinary `_system` tags with `kind: source`-like, read-only values and the usual quality codes:

```
_system/gateway/health            enum/string  ok|degraded|down
_system/gateway/uptime            float64 s
_system/gateway/cpu_temp          float64 Cel
_system/gateway/load1             float64
_system/gateway/mem_avail         float64 %
_system/gateway/disk_data_free    float64 %
_system/gateway/clock_offset      float64 s
_system/gateway/spool_bytes       int64
_system/gateway/last_backup_age   float64 h
```

The point: the **ordinary alarm engine** then alarms on "disk free below 15%" with the same acknowledgement, shelving, history and routing as a plant alarm, and the customer sees gateway health in the same HMI as the process. The agent is the writer for host-level values (it already reads `/proc`, `/sys`, `chronyc`); the interface for it is an open question (a driver, `skyranger-driver-host`, over the existing gRPC contract would reuse everything: a first-party driver with connection `host`, addresses like `host://disk/data/free_pct`. That is the cleanest answer and is the recommended proposal).

## 3. Metrics

Endpoint: `skyrangerd` serves Prometheus text exposition on `SKYRANGER_METRICS_ADDR` (02: `:9100`; **port is an open question** because 9100 is node_exporter's default). Bound to localhost by default; LAN bind is a per-site option. The agent exports its own host and supervision metrics on a second localhost port. No authentication on localhost; if bound to the LAN, a bearer API key is required (security doc). Prometheus is a format here, not a dependency: nothing on the box scrapes it except the agent.

Naming: `skyranger_` prefix, snake_case, base units (seconds, bytes), `_total` for counters (Prometheus conventions). Label values follow the identifier grammar (connection names, driver names); **never tag paths as labels** (cardinality).

| Metric | Type | Labels | Meaning |
|---|---|---|---|
| `skyranger_build_info` | gauge=1 | `version`, `commit`, `gateway_id` | |
| `skyranger_uptime_seconds` | gauge | | |
| `skyranger_config_revision` | gauge | | |
| `skyranger_tags` | gauge | `kind` | tag counts |
| `skyranger_tag_updates_total` | counter | `quality_class` (`good`,`uncertain`,`bad`) | engine output; the update gate's data-flow check ([06](06-updates-and-rollback.md)) uses the `good` rate |
| `skyranger_tags_by_quality` | gauge | `quality_class` | current counts |
| `skyranger_connection_state` | gauge | `connection`, `driver`, `state` | 1 for the current state (states from driver-interface.md section 3) |
| `skyranger_driver_dropped_samples_total` | counter | `connection` | the driver contract's `dropped_samples` |
| `skyranger_driver_restarts_total` | counter | `connection` | |
| `skyranger_driver_overruns_total` | counter | `connection` | polled-driver cycle overruns (driver-interface.md section 6) |
| `skyranger_clock_skew_rejections_total` | counter | `connection` | tag-model.md `clock_skew_rejections` |
| `skyranger_historian_queue_depth` | gauge | | |
| `skyranger_historian_spool_bytes` | gauge | | |
| `skyranger_historian_write_seconds` | histogram | | batch insert latency |
| `skyranger_alarms_active` | gauge | `severity`, `state` | |
| `skyranger_api_requests_total` | counter | `route`, `code` | route template, not raw path |
| `skyranger_api_request_seconds` | histogram | `route` | |
| `skyranger_ws_connections` | gauge | | |
| `skyranger_db_up` | gauge | | |
| `skyranger_db_size_bytes` | gauge | | |
| `skyranger_sync_lag_seconds` | gauge | `class` | ([09](09-cloud-sync.md)) |
| `skyranger_config_drift` | gauge | `state` | ([05](05-config-as-code.md)) |
| `skyranger_backup_last_success_timestamp_seconds` | gauge | | |
| `skyranger_backup_restore_test_ok` | gauge | | |
| `skyranger_agent_*` | | | restarts triggered, watchdog notifications, gate results |
| `skyranger_host_*` | | | disk free by mount, mem, load, cpu temperature, chrony offset and state, SMART health where available |

Retention on the box: **none required**. Metrics are for scraping and for the agent's decisions (a ring buffer of the last 24 h at 1 min in `/var/lib/skyranger/state/metrics-ring`, used by support bundles so "what did the last day look like" is answerable on a box that had no Prometheus). A customer Prometheus/Grafana is supported and welcome; the platform does not ship or require a TSDB for itself. If the agent's own ring buffer is not enough, the gateway's `_system` tags are in the historian anyway (section 2.3).

## 4. Logs

- **Format**: `skyrangerd` and the agent log JSON lines to stdout/stderr (`SKYRANGER_LOG_FORMAT=json`, 02). Fields: `time` (RFC 3339 UTC ms), `level` (`debug|info|warn|error`), `component` (the CloudEvents `source` components: `tag-engine`, `alarm-engine`, `config`, `discovery`, `driver-host`, `api`, plus `historian`, `agent`), `msg`, `request_id` (matches the `X-Request-Id` in api.md), `connection`, `error`. No secrets and no tag values at `info` (tag values only at `debug`, opt-in).
- **Transport**: Docker `journald` driver (02) so host and containers share one journal with identifiers `skyranger/<container>`. `Storage=persistent`, `SystemMaxUse=2G` (04), `MaxRetentionSec=` default 30 days; journal stays under 2 GB so it cannot eat the data disk. On a box with the appliance's read-only root, the journal lives on the data partition.
- **Access**: `skyrangerctl logs [--since 1h] [--component api] [--level warn]` wraps `journalctl -o json` with these filters. The support bundle includes the last 72 h, redacted ([13](13-supportability.md)).
- **Forwarding (optional)**: `systemd-journal-upload`/`rsyslog` over TLS to a customer syslog or log collector, configured in `host/` (outbound only). Not part of the core.
- **Rate limiting**: repeated identical errors (a dead PLC) are collapsed by the component (log once per state change, then every 5 minutes with a count) so the journal and the support bundle stay readable. A log flood from one noisy connection must not rotate out the evidence of the original failure.
- **Audit** is not a log: it is the `audit_log` table (api.md). Fleet actions append to it ([08](08-fleet.md)).

## 5. The watchdog chain

Each layer supervises the layer below it and is supervised by the layer above. Layers escalate in order; nothing escalates to a reboot without trying the cheaper action first.

```
 hardware watchdog (/dev/watchdog; reboot if PID 1 stops petting it)       resets: kernel hang, systemd hang
   ▲ pet every few s by systemd (RuntimeWatchdogSec=30s, RebootWatchdogSec=10min)
 systemd PID 1
   ▲ WatchdogSec=30s on skyranger-agent.service: agent pings sd_notify only if its own checks pass
 skyrangerctl agent (host)                                               supervises:
   │   - docker daemon responsive (docker info under 5 s)
   │   - each compose service: running, healthcheck status
   │   - GET /system/health every 10 s
   │   - disk, memory, clock sanity
   ▲ container healthchecks (02) and Docker restart: unless-stopped
 skyrangerd (container)                                                  supervises:
   │   - driver child processes (restart with backoff, per driver-interface.md)
   │   - internal self-checks (event-loop lag, goroutine stall detector)
   ▲ the driver processes, MQTT broker connection, DB pool
```

### Escalation ladder (agent, per failing component; thresholds are defaults in `site.yaml: supervision:`)

| Step | Trigger | Action | Cooldown |
|---|---|---|---|
| 1 | a container unhealthy for 90 s (3 failed checks at 30 s) or `/system/health` not `ok` and not recovering for 5 min | `docker restart` that service (Docker does not restart on unhealthy by itself, see 02) | 5 min |
| 2 | 3 step-1 restarts of the same service in 30 min | restart the whole compose project (`systemctl restart skyranger`, full graceful stop) | 30 min |
| 3 | step 2 failed to produce `ok` within 10 min, or Docker daemon unresponsive | restart `docker.service` and the project | 1 h |
| 4 | still not healthy 15 min after step 3, **and** the box was healthy at some time in the last 24 h (so this is not a bad install) | **reboot the host**, max once per 6 h | 6 h |
| 5 | rebooted twice within 24 h without reaching healthy | **stop escalating**: leave the box in its current state, raise a critical alert, write a support bundle to disk. A boot loop destroys evidence and can corrupt the disk |

Also: an update gate failure triggers rollback ([06](06-updates-and-rollback.md)), which is the ladder's higher-level sibling. A PostgreSQL that is up but unhealthy is never `kill -9`'d by the ladder (step 1 restarts gracefully with the 60 s stop timeout, 02); data loss is worse than waiting.

The agent is itself supervised: if the agent dies, systemd restarts it (`Restart=always`) and after repeated failures the hardware watchdog does not help (it only watches PID 1). Therefore the agent also exposes its liveness through the heartbeat (section 6). **Agent silence is detected from outside.**

Hardware watchdog caveats (restating 03): many mini PCs have no usable watchdog; VMs use a virtual device; `preflight` reports "no hardware watchdog" as a warning and the supervision still works, minus protection from kernel hangs.

## 6. Alert on silence (dead man's switch)

Silence is the failure no internal check can report. Three independent detectors, from "most likely to exist" to "most independent":

1. **Fleet heartbeat** ([08](08-fleet.md) section 4). Central marks a gateway `silent` after 3 missed 60 s heartbeats (so ~3 to 4 min) and alerts by email/webhook. Distinguishes: heartbeat missing (box or its uplink gone) vs heartbeat present but `health: down` (box up, gateway sick). Requires fleet enrollment and uplink.
2. **External ping** for sites without fleet: the agent can `GET`/`POST` a configured URL every minute (`site.yaml: heartbeat.url`), designed for third-party dead-man's-switch services (Healthchecks-style: the service alerts when pings stop; self-hostable open-source Healthchecks is an option, UNVERIFIED exact feature set). It carries no tag data: `{gateway_id, version, health}`. Outbound HTTPS only.
3. **Local-network detector, for no-uplink sites**: the MQTT `status` topic already has a Will (`{"online": false}`, messages.md section 3). Any other device on the plant LAN (a PLC with MQTT, a second SCADA, a small watcher script or a Raspberry Pi) can subscribe to `skyranger/v1/{gateway_id}/status` and alarm when it flips to `online: false` or stops being refreshed. Retained `status` also shows the last-known version and config revision. This also needs the broker to be alive; if the whole box is dead the Will fires only on a broker *ungraceful client disconnect*, so a dead box produces no Will at all (the broker died with it). A watcher must therefore also treat "broker unreachable" as down. We document this rather than hide it.
Related but different: **silence of the data, not of the box**. The customer's real question is "is the pump station data current?" Per-tag staleness is handled in the tag engine (`stale_after`, quality `uncertain_last_known`, tag-model.md) and the leak/health tags of the water meter. A plant alarm "device dead" is separate from "gateway dead".

Also for the water meter: the ESP32's 60 s heartbeat (decision 6) distinguishes "no flow" from "dead device"; if the *gateway* is dead, nobody is reading that heartbeat, which is exactly what detectors 1 to 3 cover.

### Alert delivery from the box (no central needed)
Alerts that originate at the gateway (as opposed to plant alarms) are routed by the **agent**, not by `skyrangerd`, so they survive a dead `skyrangerd`:

| Channel | Config | Notes |
|---|---|---|
| Email via SMTP (customer relay or local) | `site.yaml: alerts.smtp` | secrets via `secret://` |
| Webhook (HTTPS POST) | `alerts.webhook` | integrates with Teams/Slack/PagerDuty-style receivers (UNVERIFIED payload formats; ours is generic JSON) |
| Journal + HMI banner | always on | last resort; the HMI banner reads `/system/health` |
| Local relay/GPIO | later | a dry contact to a PLC input for the plant's own annunciator (a classic and robust pattern; hardware dependent) |

Alert content: severity, `gateway_id`, what, since when, suggested runbook id ([13](13-supportability.md)). De-duplicated by a key, re-sent every 6 h while active, an explicit "resolved" message, and a rate limit so a flapping condition cannot spam.

### Severity table (defaults)

| Condition | Severity |
|---|---|
| Gateway silent, no heartbeat 3 min | critical |
| `/system/health` `down` > 2 min | critical |
| Escalation step 5 reached | critical |
| Data disk > 90% | critical; > 80% warning |
| Backup older than 48 h or restore test failed | warning |
| Clock unsynchronized > 15 min or offset > 1 s | warning |
| Historian spool non-empty > 10 min | warning |
| Update gate failed or rolled back | critical |
| Config drift `diverged` > 24 h | warning |
| Sync lag > 24 h | warning |
| Fleet certificate expiring < 30 days | warning |
| Host reboot detected (unexpected, not planned) | warning |
| Drive SMART health failing, if available | critical |

## 7. PostgreSQL and TimescaleDB specifics

The agent (through the `postgres` container, read-only role) collects: database size, WAL size and `max_wal_size` pressure, longest transaction age, `pg_stat_activity` counts, compression job last success and chunk counts per compression state, retention job last success, replication lag (T3), and dead-tuple/bloat on the small config tables. Expressed as `skyranger_db_*` metrics and `doctor` checks. A failed compression or retention job is a **warning with a runbook**, since it silently turns into a full disk later.

## 8. `skyrangerctl doctor`

One command that evaluates the above on demand (also run by the agent every 6 h and by `preflight`): each check has an id (`DOC-xxx`), a result (`pass|warn|fail`), evidence and a runbook link. It is the first thing a support engineer asks for ([13](13-supportability.md)) and is included in every support bundle.

## Decisions made
1. Four signals (health, metrics, logs, heartbeat); every important failure must be visible in two independent places.
2. Health keeps api.md's shape; we define `ok|degraded|down` semantics and request additive check keys (`historian`, `clock`, `disk`, `config`).
3. Prometheus text format with `skyranger_` metrics; no TSDB on the box, a 24 h ring buffer for support; labels never contain tag paths.
4. JSON logs to journald with a fixed field set; log flood control by state-change logging.
5. Watchdog chain: hardware watchdog, systemd, agent (`WatchdogSec`), containers; a five-step escalation ladder with cooldowns and a hard stop before a boot loop.
6. Silence is detected from outside the box (fleet heartbeat, external dead-man's-switch ping, LAN watcher of the MQTT `status` topic); the agent, not `skyrangerd`, sends gateway alerts.
7. Gateway vitals become `_system/gateway/...` tags through a first-party `host` driver proposal so the ordinary alarm engine and historian cover them.

## Alternatives rejected and why
- **A full monitoring stack on every box (Prometheus + Grafana + Alertmanager + Loki)**: four more processes that can fail on a gateway whose job is to be small and boring. The format is standard so a customer can add them.
- **Only internal health checks, no outside heartbeat**: a dead box reports nothing.
- **Letting Docker restart on unhealthy (autoheal sidecars)**: needs the Docker socket in a container (rejected in 02). The agent does it from the host with a ladder.
- **Rebooting at the first sign of trouble**: reboots hide root causes and can corrupt a stressed disk; the ladder tries cheaper actions first and refuses to loop.
- **Pushing metrics to central by default**: bandwidth and privacy cost; the fleet heartbeat carries a small summary, and full metrics are pull-on-demand or local.
- **Logging to files with logrotate in containers**: second rotation system (02 already rejected json-file).
- **SNMP traps**: still common in OT; a later gateway-side adapter can map alerts to SNMP for customers who require it. Not core.

## Open questions
- **Contract (api.md)**: confirm the extra `checks` keys (section 2.2); add `GET /metrics` or fix the metrics port (9100 collides with node_exporter; candidate 9452, UNVERIFIED against registered ports); and a health listener on a Unix socket (02) so the HTTPS port stays TLS-only.
- **Contract (tag-model/driver)**: a first-party `host` driver and the `_system/gateway/*` tag set (section 2.3). The agent running as root should not become a driver host; the likely shape is a small read-only provider in `skyrangerd` plus the agent pushing values through a local API key, or the host driver running as an unprivileged process reading files the agent writes under `/run/skyranger`.
- Metric names and cardinality are provisional until the tag-engine and driver-host owners implement them.
- Whether to expose `/debug/pprof` (Go) behind admin auth for support, or only in support bundles via a signal (security doc).
- External dead-man's-switch services (Healthchecks and similar) are a customer choice; confirm acceptable data content (`{gateway_id, version, health}`).
- Alert routing for no-uplink, no-SMTP sites: GPIO/relay output hardware is not selected ([03](03-packaging-and-hardware.md) hardware list).
- Escalation ladder thresholds are first guesses; tune from soak tests ([14](14-release-engineering.md) gates).
