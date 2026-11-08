# Cross-Area Consistency Check

Reviewer: independent. Date: 2026-11-08. Companion to [contract-issues.md](contract-issues.md), which holds the issue IDs (B-, S-, M-) referenced here.

Nothing existing was edited. `docs/10-agent-team-and-roadmap/` was in progress and is cited only where it exposes a conflict.

## 1. Method and coverage (read this first)

| Material | How it was reviewed |
|---|---|
| `00-contracts/` (5 files) | Read in full. |
| `decisions.md`, `docs/README.md`, root `CLAUDE.md` | Read in full. |
| Area READMEs 01 to 09 | Read in full. |
| All "Open questions" sections of every file in areas 01 to 09 (about 92 KB extracted) | Read in full. |
| Bodies of the other roughly 110 area files | **Not read line by line.** Searched with targeted patterns (quality names, topic strings, ports, durations, `_system` names, event types, endpoints, kinds, DDL, retention numbers, licence wording) and read around every hit that mattered. Selected sections read directly: `03-storage/01-schema.md` sections 2 to 6, `03-storage/08-disk-power-corruption.md` ladder, `03-storage/09-capacity.md` sections 1 to 5 and 10, `01-tag-engine/derived.md` 6.1 and 6.4, `07-security/04-device-identity.md` ACL and provisioning, `05-hmi-api/08-api-extensions.md` sections 1, 7 to 10, `04-alarms/01-configuration.md` sections 4 and 6, `08-deployment/11-time-and-power.md` sections 3, `10-agent-team-and-roadmap/02-roadmap.md` M1. |

Consequence: a contradiction that lives only in the body of a file and uses wording my searches did not match could be missing. The areas that most need a second pass are 05 (HMI component bodies), 06 (plugin and SDK bodies) and 09 (individual test cases).

## 2. What was checked, with result

| # | Dimension | Method | Result |
|---|---|---|---|
| 1 | Value object field names (`tag_id`, `path`, `value`, `quality`, `ts`, `ts_source`, `ts_gateway`) | Search for alternative names (`ts_device`, `ts_gw`, `received_at`, `source_ts` and others); read scripting API and storage views | **Consistent.** No area invents a variant. Storage splits the three timestamps into `ts` plus millisecond offsets internally but the public view `pub.history` and the API restore the contract names. |
| 2 | Quality code names and numbers | Extracted every backticked `good_*`, `bad_*`, `uncertain_*` token; compared `quality_codes` DDL with tag-model section 4 | **Consistent.** Only the 14 contract names appear (`bad_alias`, `bad_quality`, `bad_request` are fault-injection names or an HTTP code, not qualities). Numeric codes in storage match. Disagreement is only in *mapping* (S-25). |
| 3 | Internal bus topics `skyranger/v1/{gateway_id}/...` and `spBv1.0/...` | Search for any other topic root and for topic-like strings | **Consistent.** No other root exists. Security ACL tables enumerate Sparkplug message types correctly. Fleet and sync use HTTP paths (`/fleet/v1`, `/sync/v1`), not MQTT topics. |
| 4 | Who writes history | Search for any history-through-MQTT design | **Consistent.** Gateway in-process writer everywhere (messages.md alternatives, storage README, drivers alternatives). Scripting P1 reinforces it. Deployment cloud sync uses the optional northbound Sparkplug, not the history path. |
| 5 | How scripts reach tags | Cross-read contract, scripting, security, tag engine, deployment | **Contradiction.** See C-5 below. |
| 6 | Config revision model (`Gateway`, `Connection`, `Template`, `TagGroup`) | Search for `kind:` values, DDL, revision streams | **Partly consistent.** Storage `config_revisions` matches the contract wording (one row per revision, documents content-addressed). Contradiction: HMI separate stream, scripts, plugins, many new kinds (S-28, B-08). |
| 7 | Alarm state names | Compared messages.md section 5, tag-model section 8, alarms 02, storage `alarm_state` CHECK | **Consistent** on the four states; tag-model prose still says `shelved`/`disabled` states (S-12). |
| 8 | Durations and units | Search for `d`, `w`, `y` suffixes and `_ms` fields; UCUM unit strings | **Contradiction.** See C-1. `_ms` integers appear only as internal columns or third-party option names (`scan_ms`, `duration_ms`, `created_ms`, `auto_sync_read_ms`), none on a wire; `scan_ms` is reported via a `details` map the proto does not have. Units: only `Cel` and UCUM strings in configs; display symbols (`m3` to `m³`) are HMI only. |
| 9 | Ports | Collected every port number | **Contradiction.** See C-6. 8443 (HTTPS and WS) is consistent everywhere. |
| 10 | Default retention numbers | Collected every retention figure | **Contradiction.** See C-9. |
| 11 | Capacity estimates (storage vs deployment) | Recomputed deployment's example with storage constants | **Contradiction.** See C-8. |
| 12 | TimescaleDB licence assumptions | Searched every mention | **Gap.** Only one doc has any licence statement, flagged UNVERIFIED, while three other docs state or imply a licence position. See C-10. |
| 13 | Event types and CloudEvents `source` components | Collected all `skyranger.*.v1` types and `/gateways/.../` sources | **Drift.** About 25 types appear; the contract catalog lists 8. Components `script-host`, `storage`, `security`, agent are not in the contract list (S-20). |
| 14 | REST endpoints vs api.md | Collected every `METHOD /path` per area | **Drift.** About 60 endpoints appear in areas that are not in api.md (S-21, B-12). Path forms used for existing endpoints are consistent (`/tags/{tag_id}/write` etc.). |
| 15 | Roles | Search for roles beyond viewer/operator/engineer/admin | **Consistent.** Service accounts are principals with one of the four roles. |
| 16 | `counter_total` algorithm as implemented in 01 vs contract | Read derived 6.1 against tag-model section 7 | **Mostly consistent with additions** (`PendingEpoch`, `Hold`, `good_manual` emission on write). The additions are not in the contract (B-04). |
| 17 | Modbus address examples | Compared contract and modbus-tcp.md | Contract is ambiguous; driver doc resolves it correctly (S-24). |
| 18 | TLS and database access | Compared security, deployment, storage | **Contradiction.** See C-6, C-7. |
| 19 | Time and clock handling | Compared tag engine, storage, deployment 11, testing | **Contradiction.** See C-4. |
| 20 | Versioning and support windows | Compared api.md section 8, deployment 14, scripting 10 | **Consistent enough:** 12 months and two minors both sides; scripting's 24 months for `api_level` is separate. |

## 3. Contradictions found (exact references)

Each item names the conflicting statements. "Issue" gives the register ID.

### C-1 Duration grammar (B-01)
- `00-contracts/README.md`, "Field names and encodings": "Durations: Go duration strings everywhere in JSON/YAML".
- `00-contracts/tag-model.md` section 9 Gateway example: `default_retention: 400d`. Go's `time.ParseDuration` has no `d`.
- `00-contracts/api.md` section 3 History: `interval` "1d", `tz` "for `interval` >= `1d`".
- `03-storage/README.md` Open question 1 admits it and parses `Nd` privately.
- Not acknowledged anywhere: `3y` (`03-storage/02-timescale-and-policies.md`), `7y` and `1y` (`07-security/08-audit-logging.md`).

### C-2 `counter_total` and a lost `boot_count` (B-02, B-03, B-04)
- `00-contracts/messages.md` section 2: "`boot_count` ... incremented once per boot before connecting".
- `00-contracts/tag-model.md` section 7: branch `else if modulus > 0` classifies a decrease with unchanged epoch as rollover; `max_delta` is optional.
- `09-testing/08-hardware-and-power-cut.md` Open question 1 shows the consequence. `01-tag-engine/derived.md` 6.1.2 acknowledges "binding `epoch` is the only unambiguous configuration" but adds nothing that prevents the failure.
- `00-contracts/tag-model.md` section 7 sets `last_raw = raw` after `implausible`; `09-testing/04-fault-injection.md` F-CNT-03 shows the second wrong delta.

### C-3 Ordering and clock steps (B-05)
- `00-contracts/tag-model.md` section 5: older `ts` does not replace the current value.
- `01-tag-engine/README.md` key decision 5: "Gateway-timestamped samples are always newer in arrival order". This silently differs from the contract.

### C-4 What to do with a wrong gateway clock (S-16, B-06)
- `03-storage/03-write-path.md` section 8: sample with `ts < 2020-01-01` or `> now + 5 min` is dead-lettered, never inserted; clock steps are a hazard "this layer cannot repair".
- `08-deployment/11-time-and-power.md` section 3: in `suspect` state, samples are spooled and replayed "with timestamps corrected by the measured offset"; the historian refuses samples older than the last written row on the wrong side of a step.
- `00-contracts/tag-model.md` section 5, last bullet: backfilled data is out of scope for v1.
- `01-tag-engine/README.md` decision 5 again.

### C-5 Scripts and tags (S-08)
- `00-contracts/messages.md` section 3: "The Python scripting runtime will use the REST API for writes".
- `01-tag-engine/writes.md`: "Scripting writes go through the same REST endpoint".
- `07-security/02-authn-authz.md` principal table: `svc-python-runtime`, role `operator`, "writes go through REST"; `07-security/04-device-identity.md` ACL row gives it MQTT subscribe on listener 8884.
- `06-scripting/README.md` P1: scripts use a script host over `skyranger.script.v1`; workers have no sockets. The scripting area states this disagreement itself.
- `08-deployment/02-containers.md`: no scripts container in the compose file or the container table.

### C-6 Broker listeners and TLS (B-09)
- `00-contracts/tag-model.md` section 9: `broker_url: mqtt://mosquitto:1883`.
- `07-security/03-tls-and-pki.md`: "No plaintext 1883 listener at all"; 8883 field listener; 8884 internal-bus listener with mandatory client certificates; `skyrangerd` authenticates by client certificate.
- `08-deployment/02-containers.md`: 1883 plaintext inside the compose network (also used by the broker health check and `SIM_MQTT_URL`), only 8883 on the LAN; no 8884.
- `02-drivers/sparkplug.md` section 2: username/password credential, optional `tls.ca`; no client-certificate setting.
- `10-agent-team-and-roadmap/02-roadmap.md` M1: plain MQTT on a trusted bench.
- Related port drift: metrics `:9100` (`08-deployment/02-containers.md`, `10-observability.md`) collides with node_exporter; simulator control `:8090` in compose vs `:9900` in `09-testing/03-simulator.md`.

### C-7 Database access and secrets (S-17, S-18)
- `08-deployment/02-containers.md`: `SKYRANGER_DB_URL: postgres://skyranger@postgres:5432/skyranger?sslmode=disable`; secrets in `/etc/skyranger/secrets` bind-mounted; secret name `db_password`.
- `03-storage/01-schema.md` section 2: roles `sr_owner`, `sr_app`, `sr_audit_writer`, `sr_readonly`.
- `07-security/03-tls-and-pki.md`: `sslmode=verify-full`.
- `07-security/06-secrets.md`: values encrypted in PostgreSQL with a KEK at `/run/secrets/kek`; DB password in `/run/secrets/pg_*` (file `pg_skyrangerd_password` in the compose excerpt).
- Postgres image: `timescale/timescaledb@sha256` (deployment) vs `registry.example/skyranger/postgres-timescaledb` (`07-security/09-hardening.md`).

### C-8 Capacity (S-13)
- `08-deployment/03-packaging-and-hardware.md` "Sizing history storage": 40 to 60 B/row, 10x to 20x compression, 1,000 tags at 1 change/10 s = about 430 MB/day uncompressed, 25 to 45 MB/day compressed, 10 to 16 GB at 400 d.
- `03-storage/09-capacity.md` sections 1 and 10: `B_unc` 140 B, `B_cmp` 16 B; the same example gives 8.64 M rows/day, 1.2 GB/day, 138 MB/day compressed, 55 GB at 400 d (3.5x to 5.5x more). Storage says its own figure is "more conservative" and unmeasured.
- Tag engine `01-tag-engine/performance-and-failures.md` T1 50,000 configured tags, T2 5,000 accepted changes/s; testing `09-testing/07-soak-and-performance.md` "10,000 supported, 100,000 target"; storage 09 "100,000 is not an edge target"; deployment "10,000 tags at 1 s" (10x busier than storage's busiest profile, which already needs about 1 TB).

### C-9 Retention and backups (S-14, S-15, M-07)
- Raw history 400 d (tag-model Gateway example, storage 09, deployment 03, testing 07); rollup `3y` (proposed); alarm journal `1095d`; events follow raw history (`07-security/08-audit-logging.md`, `03-storage/06-events-alarms-audit.md`); audit `7y`.
- Disk ladder: `03-storage/08-disk-power-corruption.md` prunes whole raw chunks at 90% and sheds at 95%; `08-deployment/07-data-and-migrations.md` runs emergency retention at 95% and proposes per-tag shortening ("`sampled` before `on_change`"), which `03-storage/README.md` decision 5 and 02 rule out.
- Backup rotation: 7/4/12 at 02:30 (`03-storage/07-backup-restore.md`), 7/4/3 at 02:17 (`08-deployment/07-data-and-migrations.md`), 14/8/12 (`07-security/10-backup-security.md`).

### C-10 TimescaleDB licence (B-11)
- `03-storage/02-timescale-and-policies.md` licence note: from memory, unverified.
- `08-deployment/12-exit-plan.md` line "Not lock-in by design: PostgreSQL/Timescale open schema ... open-source code" and its CI exit gate that loads history into plain `postgres:17`.
- `CLAUDE.md` rule 10 and `decisions.md` 3 ("open standards, no lock-in").
- `06-scripting/10-sdk-compat-licensing.md` Open question 4 repeats the concern. No document states a verified licence.

### C-11 Duplicate and missing DDL (B-08)
- `alarm_journal`: `03-storage/01-schema.md` section 7 (columns `transition`, `from_state NOT NULL`, key `(alarm_id, seq, ts)`) vs `04-alarms/07-journal-and-events.md` (adds `id`, `event_id NOT NULL`, `kind`, `class`, `console`, `condition_since`, `actor NOT NULL`, flags).
- `derived_state`: `01-tag-engine/derived.md` section 7 (`state_version`, `orphaned_at`) vs `03-storage/01-schema.md` section 6 (`schema_version`, `orphaned_since`, fillfactor settings).
- `tag_writes`: defined in `01-tag-engine/writes.md` section 6; absent from the storage ownership matrix that claims "storage owns all DDL".
- HMI revisions, assets and preferences, script documents and plugin state: no table anywhere.

### C-12 Alarms and `_system` tags (B-10, S-07)
- `00-contracts/tag-model.md` section 1: `_system` tags "read-only, never in user config".
- `04-alarms/01-configuration.md` section 6 defines alarms on `_system/database/state`, `_system/disk/used_pct`, `_system/historian/spool_bytes`; storage and security name the same quantities `_system/gateway/disk_used_pct`, `_system/storage/*`, `_system/historian/db_state`.
- Severity ladders also differ: alarms `high`/`high_high` at 80/90% (medium/high) vs storage warn 80 (medium), 85 (high), 90 (critical).

### C-13 `continuous_activity` (S-03)
- `01-tag-engine/derived.md` 6.4 and section 7: state persisted in `derived_state`; frozen while input bad; outage longer than `max_gap` restarts the streak.
- `04-alarms/09-restart-and-sync.md` Open questions and `04-alarms/11-water-meter-e2e.md`: say the contract is silent on persistence and ask for it; recommend freezing time across outages, which 01 does not do after recovery.

### C-14 Driver privileges (S-19)
- `07-security/09-hardening.md` section 3: per-driver UID `20000 + n`, `cap_add: [SETUID, SETGID]`, helper `skyranger-netd`.
- `08-deployment/02-containers.md`: `skyrangerd` has `cap_drop: ALL`, `user: 10001:10001`, no netd (only the postgres and mosquitto services add capabilities).
- `02-drivers/framework.md`: "drivers must work with read-only root filesystem and no capabilities".

### C-15 Other items
- `04-alarms/README.md` files table describes proposed kinds `NotificationChannel`/`AlarmRoute`; `04-alarms/01-configuration.md` Alternatives rejects them; `06-scripting/09-plugins.md` assumes a `NotificationChannel` kind (S-11).
- `05-hmi-api/06-operator-ergonomics.md` kiosk key in `localStorage` vs `00-contracts/api.md` section 2 (S-31).
- `08-deployment/README.md` states a file-backed secret store is assumed "until that doc says otherwise"; the security doc does say otherwise (C-7).
- `10-agent-team-and-roadmap/02-roadmap.md` M1 requires per-device MQTT credentials "from day one" but lists `skyrangerctl` as out of M1, while `07-security/04-device-identity.md` provisioning is built on `skyrangerctl` and `POST /devices` (B-12).

## 4. Points where areas agree and the agreement is sound

Recorded so that nobody reopens them without cause:

- Writes are never retried, outcome unknown on timeout (contract, drivers, tag engine, HMI, security).
- One door for change: REST only, no command topics (contract, security principle 3, HMI, tag engine).
- TLS 8443 for HMI, REST and WS on one port; no separate HMI container (contract, deployment README).
- `liveness` health design (0/1/2) and the "no flow vs dead device" distinction are carried identically through tag engine, alarms, HMI and the testing oracle.
- Alarm state machine: four contract states plus flags (alarms, storage CHECK constraint).
- Sparkplug rebirth rate limit 30 s and the `bdSeq`-is-not-an-epoch rule (contract, drivers, testing).
- Config export then import gives an empty diff (contract, storage 05, deployment 05, testing).

## 5. What I could not verify

| Area | Not verified, and why |
|---|---|
| External standards | Sparkplug B 3.0 `STATE` payload and wait rule, timestamp optionality, seq rules; OPC UA status code names; UCUM codes (`us`, `By`, `dBm`); ISA-18.2 and IEC 62682 details; IEC 62443 terms; the `Deprecation` header RFC number. No spec text was available; all remain UNVERIFIED as the docs say. |
| Product facts | TimescaleDB licence and feature set per edition; `ON CONFLICT DO NOTHING` on compressed chunks; Mosquitto dynamic security behaviour; libplctag, gopcua, gos7 maturity and licences; Debian 13 and Docker iptables interaction. Network access to confirm these was not used. |
| Numbers | Every capacity, throughput and latency figure is an estimate in its source doc. I recomputed the deployment example against storage constants (C-8) but did not measure anything; nothing can be measured because no code exists. |
| Bodies of ~110 area files | Searched, not read (section 1). In particular unchecked: individual HMI component specs, the 12 scripting files beyond README and Open questions, most test-case tables in 09-testing, and 10-agent-team-and-roadmap (in progress). |
| Goldens and schemas | `testdata/contracts/`, `proto/` and `schemas/` do not exist yet, so "docs vs machine-readable contract" could not be checked. |
| Arithmetic inside docs | Spot checks only (the water-meter row counts were not recomputed; the deployment example was). |

## Decisions needed

1. Rule on the twelve blocking items in [contract-issues.md](contract-issues.md) (top 10 table at the end) so contracts and goldens can be updated before any agent writes code.
2. Decide which of C-5 to C-9 and C-14 are settled by contract amendment (needs a decisions.md entry) and which are settled by area owners reconciling their own docs (no decision needed): C-7, C-8 (except the supported-tag claim), C-9 backup rows and C-15 are owner-level fixes.
3. Authorise a second review pass over the unread bodies (section 1) before the first implementation work package is cut, or accept the residual risk.

## Open questions

- Should every area README gain a one-line "Contract deviations" table in a fixed format so the next review is mechanical? Today the flagged issues are in five different shapes (README tables, "CONTRACT:" prefixes, numbered lists, P1 to P9, free text).
- Who owns the single `_system` registry and the single "API additions" table once the contracts accept them (README of 00-contracts, or a new contract file)?
- Does the roadmap (area 10, in progress) intend `skyrangerctl` to exist at M1? Three areas assume it does or does not, and the answer changes where first-admin creation and device provisioning live.
