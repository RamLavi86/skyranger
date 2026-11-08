# 09 Optional Cloud Sync

Binding: decision 1 (cloud is an add-on, never required), README principle 2 (outbound only), contracts: value object, `tag_id` (UUIDv7), `(tag_id, ts)` history key, CloudEvents `id` for de-duplication, northbound Sparkplug ([messages.md](../00-contracts/messages.md) section 8), `{gateway_id}:{path}` multi-gateway addressing reserved but not specified (contracts README open question; this document is where it gets specified, minimally).

"Cloud" here means **central**: a Linux VM running `skyranger-cloud` (the same images in `SKYRANGER_ROLE=central`, [01-topologies.md](01-topologies.md) T4). It can be ours, the customer's, or any hyperscaler VM. Nothing in the design uses a proprietary cloud service.

## 1. Principles

1. **The edge is the single writer for everything it produces.** Tag values, history, alarms, events, audit and config revisions of a gateway are created at that gateway. Central holds **copies**.
2. **Central never edits site config or writes to devices through sync.** Config changes arrive at a site only through the fleet desired state ([08-fleet.md](08-fleet.md) section 6) and are applied by the site's own apply path. Writes to tags from central are off by default and, when enabled, are REST calls to the site gateway through a tunnel-like authenticated channel (01), not part of sync.
3. **Sync is idempotent and resumable.** Every unit has a stable key, the receiver de-duplicates, the sender keeps a watermark. A month-long outage is just a long catch-up.
4. **Sync can lag, never block.** A full or slow central must not slow the gateway. Sync reads from the gateway's database, not from its hot path.
5. **Data minimization is configurable per site**, because uplinks can be cellular and data can be commercially sensitive.

## 2. What syncs

| Class | Direction | Granularity | Transport | Default |
|---|---|---|---|---|
| Heartbeat and health | edge to central | summary every 60 s | fleet channel ([08](08-fleet.md) section 4) | on when enrolled |
| Config revisions | edge to central | each `config_revisions` row as normalized YAML + metadata | HTTPS batch | on (read-only mirror, tiny) |
| Events and alarm state changes | edge to central | CloudEvents, keyed by `id` | HTTPS batch | on |
| Audit log | edge to central | records keyed by id | HTTPS batch | on, off by customer option |
| History (numeric and others) | edge to central | per tag, filtered and optionally aggregated | HTTPS batch | **off until `sync.yaml` selects tags** |
| Current values (live) | edge to central | latest-value, on change | Sparkplug B northbound (`spBv1.0`) over MQTT TLS 8883 outbound | off |
| Tag definitions (resolved) | edge to central | derived from config revision | with config | on |
| Desired state, releases | central to edge | signed documents | fleet channel | n/a |
| Commands / writes | central to edge | none in sync | n/a | not in sync |

Not synced, ever: secrets, TLS keys, the `users` password hashes, API key hashes, raw support session recordings (unless the customer attaches them to a support bundle).

### Selection: `/etc/skyranger/sync.yaml` (host file, in the repo's `host/`, [05](05-config-as-code.md) section 10)

```yaml
version: 1
enabled: true
target:
  url: https://central.example.net          # https only
  project: acme                             # see multi-tenancy
history:
  select:                                   # tag path prefixes (README prefix rule: "a/b" matches a/b and a/b/*)
    - prefix: site1/pumphouse
      exclude: [site1/pumphouse/raw]
  mode: raw                                 # raw | aggregated
  aggregate: { interval: 1m, functions: [avg, min, max, last] }   # only when mode: aggregated
  max_bandwidth: 256kbit/s                  # token bucket on the uploader
  window: "00:00-23:59"                     # when uploads may run (local tz)
live:
  enabled: false
  prefix: site1/pumphouse
events: { types: [skyranger.alarm., skyranger.connection., skyranger.config.] }
```

Why a host file and not a config `kind`: [tag-model.md](../00-contracts/tag-model.md) defines four kinds and the `Gateway` spec; adding sync to `Gateway.spec` is an additive contract change we do not need yet and that would make an operational transport choice part of the plant model. If sync gets richer (per-tag flags), a `Gateway.spec.sync` can be proposed. See Open questions.

## 3. Wire protocol (`sync/v1`)

HTTPS, outbound, JSON (NDJSON for batches), `Content-Encoding: zstd` or gzip, client certificate (same enrollment as fleet, [08](08-fleet.md) section 3). Follows the contracts' conventions (snake_case, RFC 3339 UTC ms, int64 as strings, UUIDv7, RFC 9457 errors).

```
POST /sync/v1/{project}/{gateway_id}/history     NDJSON of samples (value objects, tag_id + ts + value + quality + ts_gateway)
POST /sync/v1/{project}/{gateway_id}/events      NDJSON of CloudEvents
POST /sync/v1/{project}/{gateway_id}/config      one revision per call
POST /sync/v1/{project}/{gateway_id}/audit       NDJSON
GET  /sync/v1/{project}/{gateway_id}/state       receiver watermarks (what central has)
```

- **Keys**: history `(project, gateway_id, tag_id, ts)`, events `(project, gateway_id, id)`, config `(project, gateway_id, revision)`, audit `(project, gateway_id, audit id)`. The receiver uses `INSERT ... ON CONFLICT DO NOTHING`; a retried batch is harmless.
- **Watermarks**: the sender keeps per-class progress in the gateway DB (`sync_state`). For history the watermark is **ingest order**, not sample time: late or out-of-order samples ([tag-model.md](../00-contracts/tag-model.md) section 5 accepts them into history) must still be uploaded. This requires an ingest sequence or insertion timestamp on history rows (Open questions, storage doc). Without it, a time-based watermark would miss late samples; the fallback is a periodic re-scan of the last 24 h by `(tag_id, ts)` with the receiver's de-duplication absorbing the overlap.
- **Batching**: up to 10,000 samples or 1 MiB compressed per request, bounded by `max_bandwidth`. Oldest-first within a class so a recovery after outage is chronological.
- **Backpressure and failure**: HTTP `429`/`503` with `Retry-After`; exponential backoff capped at 5 minutes; no data is dropped by the sender on failure (it is still in the gateway DB). The only loss mechanism is the gateway's own retention: if central is unreachable longer than retention, old history is gone locally and never arrives. `skyranger_sync_lag_seconds` ([10](10-observability.md)) alerts well before that (default alert at 24 h, critical at 50% of retention).
- **Integrity**: each batch carries a count and a SHA-256 of the canonical body; the receiver verifies and acknowledges with the highest accepted key. The gateway advances its watermark only on acknowledgement.
- **Versioning**: `sync/v1` additive within a version; receiver tolerates unknown fields, sender tolerates unknown response fields; the skew policy is in [08](08-fleet.md) section 8.

### Live values
Live values use the optional Sparkplug northbound already specified in messages.md section 8 (gateway as edge node `{gateway_id}` in a configured `group_id`, metric name = tag path), toward a central MQTT broker over TLS 8883 (outbound), or MQTT over WebSocket on 443 where 8883 is blocked (UNVERIFIED client support). Reasons: it is the open standard the decision log chose; any Sparkplug host (including a customer's Ignition) can consume it; and it keeps the customer's choice of consumers open. Live values are not authoritative and are **not** used to rebuild history (central history comes only from the batch path).

## 4. Conflicts

By construction there are few conflicts, because every object has one writer.

| Object | Writer | Conflict possible? | Rule |
|---|---|---|---|
| History, events, alarms, audit | edge | no (append-only, keyed) | duplicates ignored; **first write wins** (`ON CONFLICT DO NOTHING`); an edge-side correction (rare, e.g. a deleted bad sample) is a new `events` entry, not an update |
| Config revisions | edge | no (numbered at the edge) | mirrored; central shows them read-only |
| Tag definitions | edge (via config) | no | derived from mirrored config |
| Desired state | central | edge disagrees with local policy | **local policy wins** ([08](08-fleet.md) section 5), refusal reported |
| Config **intent** from central (git commit to apply) | central names, edge applies | commit vs gateway edit (`diverged`) | never merged automatically ([05](05-config-as-code.md) section 5); reported; human picks |
| Users/roles on central vs edge | each own | none: separate identity domains | central users cannot log into a gateway; central never provisions gateway users. Federated login is a security-doc topic |
| Clock disagreement | edge `ts_gateway` | possible if an edge clock was wrong | history is stored as the edge recorded it. Central never rewrites timestamps. Samples whose `ts_gateway` is outside plausibility at ingest (older than 2020-01-01, or more than 1 day in the future) are stored with a flag column `clock_suspect` in the central schema and shown as such (see [11](11-time-and-power.md)) |
| Tag rename | edge | `path` mutable, `tag_id` immutable | central keys on `tag_id`; path history is kept per config revision, so a query by an old path resolves through revisions (a central feature to build) |
| Gateway replacement (cold spare restored from backup) | the restored box continues the same `gateway_id` | duplicate rows for the overlap interval | idempotent keys absorb them. Watermarks restored with the DB |
| **Two gateways claiming one `gateway_id`** (misconfigured clone, or a T3 split brain) | n/a | yes | receiver binds `(project, gateway_id)` to the enrollment certificate fingerprint; a second certificate for the same id is rejected and alerts. A restored spare needs `fleet re-enroll --same-id`, an explicit human act |

Edge wins everything it produced; central wins only fleet intent, and even that is a request the edge validates.

## 5. Multi-tenancy: project and site

Hierarchy:

```
 organization (the integrator's account on central)
   └─ project            a customer engagement; the hard isolation boundary
        └─ site          a physical location (pumphouse1); a label group, not a security boundary by itself
             └─ gateway  gateway_id (gw01); T1/T2 one box, T3 one logical gateway
```

Rules:
- **`project` is the isolation boundary.** One PostgreSQL **database per project** on central (separate roles, separate dump). Why: exit is `pg_dump` of one database ([12-exit-plan.md](12-exit-plan.md)); a customer can be moved to their own central by restoring that dump onto their VM; a bug in a query cannot cross customers because no query has two projects' tables in reach. Row-level security inside one shared database is a possible later optimization for many tiny projects, rejected for v1.
- `gateway_id` is unique **within a project**. The contracts say it is unique per deployment (README); at central a deployment is a project. The sync URL path carries `{project}/{gateway_id}`.
- **`site`** is metadata (`Gateway` display name, labels) used for grouping and dashboards. Permissions are granted per project, and per site as a filter on the central read-only HMI. Tag paths already encode site (`site1/pumphouse/...`), so the central views group by path prefix.
- Tag paths remain unique per gateway only. The central address of a tag is `{gateway_id}:{path}` (the syntax reserved in the contracts README), which this document now defines as: `gateway_id` + `:` + tag path, no spaces, used only in central UIs, central APIs and Sparkplug-northbound group/edge-node mapping, **never** in edge config or the edge REST API. Contract owners confirm (Open questions).
- Roles on central (`viewer`, `analyst`, `admin`) are central-only and per project; edge roles (viewer/operator/engineer/admin, api.md) are unchanged.
- The integrator organization sees all its projects; a customer's login sees only its project. A customer who runs their own central is the project's only org.

## 6. Central, concretely

- Same Go binary in `SKYRANGER_ROLE=central`: sync receiver, central database (PostgreSQL + TimescaleDB, project DBs), read-only HMI and API (the same `/api/v1` read endpoints with `gateway_id` addressing added in a v1-additive way, Open questions), report generation later.
- Compose: `skyranger-cloud` is the central image set plus the fleet service ([08](08-fleet.md)) and the tunnel endpoint; one `docker compose` project on a VM. Backups follow the same mechanism ([07](07-data-and-migrations.md)).
- Central being read-only toward sites is the main safety property: even a fully compromised central cannot change a gateway's tag values or config (it can only offer signed releases and ask the site to apply a commit, which the site validates, [08](08-fleet.md) section 5).

## 7. Cost and bandwidth notes

Rough numbers, UNVERIFIED until measured: 1,000 tags on-change at ~1 change/10 s is 100 samples/s, about 8.6 M samples/day (the same example as the sizing in [03](03-packaging-and-hardware.md)). At roughly 50 bytes per sample as NDJSON before compression and 5x to 10x compression (UNVERIFIED), that is on the order of 40 to 90 MB/day, which is fine on broadband and expensive on cellular. Hence `mode: aggregated` (1-minute avg/min/max/last cuts samples by ~10x or more at that rate, tag dependent) and `max_bandwidth`. Counters (water meter) lose nothing in `last`/`delta` aggregates.

## Decisions made
1. Cloud sync is optional, outbound-only, one-directional for data (edge to central), and idempotent with receiver-side de-duplication on contract keys.
2. Edge is the single writer for everything it produces; central never edits site config or writes to devices through sync.
3. Sync selection is a host file (`sync.yaml`), default off for history and live values, with prefix selection, aggregation and a bandwidth cap.
4. Transport: HTTPS batches for config, events, audit and history; Sparkplug B northbound for optional live values.
5. Conflicts are resolved by ownership; the only human-resolved conflict is config `diverged`; a duplicate `gateway_id` is rejected by certificate binding.
6. Multi-tenancy: organization, project (one database per project, the isolation boundary), site (metadata), gateway. The central address of a tag is `{gateway_id}:{path}`.
7. Central is a Linux VM with Docker running the same images; no cloud-vendor services.

## Alternatives rejected and why
- **Bidirectional config sync (multi-master)**: needs conflict resolution on plant configuration. Wrong risk profile. Intent flows down as a commit, state flows up as copies.
- **MQTT bridge (Mosquitto `bridge`) of the whole internal bus to central**: simple, but it carries QoS 0 values with no durable watermark, so an outage creates silent history gaps, and it exposes the internal namespace. Fine for live values (Sparkplug northbound), not for history.
- **Central reading the edge DB directly (logical replication of the history hypertable, or a PostgreSQL FDW)**: needs inbound reachability to the edge DB or a long-lived outbound replication connection, ties central to the edge schema and PostgreSQL version, and exposes a database. Logical replication is attractive; rejected for coupling to schema and the `compat` rules, but worth a spike for large fleets (Open questions).
- **A shared multi-tenant database with row-level security for v1**: smaller footprint, but weaker isolation, and a customer exit becomes a filtered export rather than a dump.
- **Proprietary IoT clouds (AWS IoT Core, Azure IoT Hub) as the transport**: lock-in; violates the exit principle.
- **Store-and-forward via MQTT QoS 1 persistent sessions**: the contracts already say store-and-forward is out of scope for v1 at the device level; for edge-to-central we use database watermarks, which survive broker restarts and make recovery auditable.
- **Syncing every tag by default**: surprises customers on data egress and cost. Default off, explicit selection.

## Open questions
- **Storage owner**: history needs a monotonic ingest sequence (or a reliable `ts_gateway` ordering with a late-sample rule) for upload watermarks, plus a `sync_state` table. Without it the 24 h re-scan fallback is used.
- **Contract owners**: (a) define `{gateway_id}:{path}` as in section 5; (b) additive `gateway_id`-qualified read endpoints for central (`/api/v1/gateways/{gateway_id}/...` or a header) versus a separate central API; (c) whether a `Gateway.spec.sync` section is wanted later.
- Whether central history for `mode: aggregated` can coexist with later `raw` upload for the same tags (backfill of higher resolution); allowed by keys only if aggregates are stored as a distinct series. Needs the central schema.
- `clock_suspect` flag semantics at ingest: needs the same definition as the edge's own time-sanity rule ([11](11-time-and-power.md) open question).
- Logical replication spike for large fleets: when does it beat the batch protocol? Decide after measuring the batch uploader at 10,000 tags.
- Live values over MQTT-over-WebSocket on 443 for networks that only allow 443: client support in the Go Sparkplug northbound implementation UNVERIFIED.
- Rate and cost numbers in section 7 are estimates only.
- Whether the audit log should sync at all by default (usernames and IPs), a privacy and customer-contract question.
