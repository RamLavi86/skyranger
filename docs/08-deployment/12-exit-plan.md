# 12 Customer Exit Plan

Decision 3: "customers must be able to leave easily, which is a selling point against Siemens/Rockwell lock-in." This document turns that sentence into a command, a file format, a checklist and a test. If any item here is not true on release day, it is a release blocker ([14-release-engineering.md](14-release-engineering.md) exit-test gate).

Principle: **the customer owns the box, the data, the configuration and the right to keep running what they have.** Leaving must need no permission from us, no network access to us, no licence key and no proprietary tool.

## 1. What "leaving" means (three levels)

| Level | Customer wants | What they get |
|---|---|---|
| **A. Keep running without us** | stay on Sky Ranger, drop the integrator/support relationship | unenroll from fleet and tunnel, take over credentials, keep the repos and the signed releases they have; open source code and offline release bundles ([14](14-release-engineering.md) section 11) |
| **B. Move to another platform** | switch to Ignition, another SCADA, a data lake | a full export: config, history, alarms, events, audit, HMI screens, scripts, in plain formats, plus a guide for each kind of consumer |
| **C. Take the history only** | archive and decommission | the history export and the PostgreSQL dump |

All three start with the same command.

## 2. `skyrangerctl export`

```
$ sudo skyrangerctl export --to /mnt/usb/skyranger-export-gw01-20261108.tar.zst
        [--no-history | --history-since 2025-01-01] [--include-secrets] [--include-audit] [--format csv|ndjson]
```

Runs on the live box without stopping it, using only the public REST API and read-only database access (no hidden internal data: whatever the export holds, the customer could also get through the API). It requires an `admin` API key or local root. It is resumable and writes a manifest with checksums last, so a partial export is recognizable.

### Contents of the export

```
skyranger-export-gw01-20261108/
  EXIT-README.md                  generated, human guide to this export (section 4)
  MANIFEST.json                   file list with sha256, sizes, counts, schema versions, export tool version, time of export
  site/
    site.yaml                     identity, timezone, topology (no secrets)
    versions.json                 installed release, manifest digests, PostgreSQL/TimescaleDB/Mosquitto versions
  config/                         the normalized YAML (05-config-as-code layout), current revision, ids included
    gateway.yaml  connections/  templates/  tag-groups/
    revisions/                    every past revision as YAML + revisions.csv (revision, applied_at, applied_by, comment)
  config-git/                     the full local git repository (history of every revision), as a bare tar
  tags/
    tags.csv                      tag_id, path, display_name, datatype, unit, kind, source connection and address,
                                  scale, eng_range, deadband, stale_after, history policy, writable, alarms summary
    tags.json                     the resolved definitions (GET /tags with current config)
  history/
    schema.sql                    plain PostgreSQL DDL (no TimescaleDB needed) for the three tables below
    samples/                      one CSV file per month (zstd): ts,tag_id,path,value,quality,ts_source,ts_gateway
    quality_codes.csv             code, name, class (tag-model.md section 4), so quality is interpretable offline
    series.csv                    tag_id, path, unit, datatype, first_ts, last_ts, rows (exact counts per tag)
    load.sql                      \copy statements to load into the plain schema
  alarms/
    definitions.csv               from tags.json
    history.ndjson                state-change records (messages.md section 5 shape)
  events/events.ndjson            CloudEvents 1.0 as stored
  discovery/devices.ndjson        discovered/approved/ignored devices (messages.md section 7)
  audit/audit.ndjson              (with --include-audit; contains usernames)
  users/users.csv                 username, role, created_at, disabled. NO password hashes, NO API key hashes
  hmi/                            HMI screens/dashboards in their native JSON as defined by 05-hmi-api (UNVERIFIED storage path)
  scripts/                        user Python scripts (UNVERIFIED storage, owned by 06-scripting)
  derived_state/derived_state.json  current counter_total state (epoch, last raw, total) per derived tag (tag-model.md section 7)
  devices/                        per-device notes for the Sparkplug devices: node ids, group ids, topics
  db/
    skyranger.dump                pg_dump -Fc (needs TimescaleDB to restore; see section 5)  [optional, --with-dump]
  secrets.tar.age                 only with --include-secrets, encrypted (age recipient or passphrase)
```

Format decisions:
- **CSV (zstd-compressed, monthly files) for history** is the lowest common denominator: every database, spreadsheet, data lake and BI tool loads it. Columns follow the canonical value object (README) with `ts` in RFC 3339 UTC ms, `quality` as the **name** string, and the `quality_codes.csv` mapping. `uint64`/`int64` values are written as exact decimal strings in CSV (no float rounding).
- The same rows are available as a **plain PostgreSQL** dump (`schema.sql` + `load.sql`) so a customer without TimescaleDB loses nothing but compression. Parquet is a plausible extra (columnar, smaller) but adds a library dependency to the export tool; offered later (Open questions).
- Config is the YAML the platform itself uses (decision 3: configuration in exportable JSON/YAML), so the export is also a **backup of the product's own input**: it re-imports into a fresh Sky Ranger without conversion.
- Everything is reachable with `tar`, `zstd`, `psql`, `jq` and a text editor. No Sky Ranger binary is needed to *read* the export.

### What the export deliberately leaves out
Secrets and private keys (unless `--include-secrets`, encrypted), password and API-key hashes (they are not portable and are sensitive), the fleet client certificate, support session recordings (they are the customer's, copied if they ask, [13](13-supportability.md)), and Docker images (they are in the release bundle the customer already has).

### Size and time
Exports stream from the database in per-month chunks; CPU and IO are throttled (`nice`/`ionice`, `--max-rate`) so a running plant is not disturbed. For the sizing example in 03 (about 10 to 16 GB of compressed history at 400 days) the CSV export is expected to be larger than the compressed database (compression ratios lost, UNVERIFIED, estimate several times larger); `export` prints the estimated size first and checks free space on the destination. Target (UNVERIFIED, measure on the golden DB): under an hour for 10 GB of database on recommended hardware.

## 3. API-only path (customer has no console access)

The same data is available over the public API with an `admin` key; the export command is a convenience around these:
- `GET /api/v1/config` (YAML, `include_ids=true`) and `GET /api/v1/config/revisions/{n}`
- `GET /api/v1/history/export?...` (CSV `ts,path,value,quality`, api.md)
- `GET /api/v1/tags`, `/alarms/history`, `/events`, `/audit`, `/discovery/devices`

A customer's engineer can script these without us. The CI exit test ([14](14-release-engineering.md)) runs both paths and compares them.

## 4. `EXIT-README.md` (generated into every export)

Contents (template maintained in the repo, filled with real counts and versions):
1. What this export contains, with the tag/row counts and the time range.
2. How to verify it: `sha256sum -c`, `MANIFEST.json` counts vs `series.csv`.
3. **Load the history into plain PostgreSQL** (copy-paste `psql` commands), into TimescaleDB, into SQLite/DuckDB (one-liners with `read_csv`, UNVERIFIED versions), and into a spreadsheet for small sets.
4. **Point Grafana at it**: a ready dashboard JSON and datasource example for PostgreSQL (UNVERIFIED against current Grafana).
5. **Take your devices with you**: the ESP32 meters speak standard Sparkplug B and the `skyranger.pulse_meter.v1` profile (documented). Any Sparkplug host (Ignition with the MQTT modules, UNVERIFIED specifics) or any MQTT client can consume them: change the broker address in the device config. The topic layout and payload description is in `devices/`. Modbus/OPC UA devices are just connection settings in `config/connections/`: copy host/unit-id/addresses into any other master.
6. **Map our quality codes to others**: table of the 14 codes and their OPC UA-like names with the **UNVERIFIED** mapping caveat from tag-model.md.
7. **Recreate alarms elsewhere**: `alarms/definitions.csv` and the tag config show limits, deadbands and delays in readable fields.
8. **Counters**: how `counter_total` derives volume from `pulse_count`/`boot_count` (the algorithm in tag-model.md section 7 reproduced), so a new system can reproduce totals from raw history if `raw/*` tags were historized, and verify totals at the cut-over.
9. **Cut-over checklist** (below).
10. **Who to contact** (customer-supplied), and a statement that nothing in the export expires.

## 5. Moving to another Sky Ranger (self-hosting, or another integrator)

This is exit level A and it must be easy, otherwise the open-source promise is hollow.

- `skyrangerctl backup create` ([07](07-data-and-migrations.md)) plus the release bundle restores onto any compatible Linux host. `site.yaml` and the config repo travel with it.
- A new integrator needs: the config repo (customer-owned remote or the export), a backup or export, the release bundle (public, signed), and secrets re-issued (`--new-box`), or `secrets.tar.age`.
- **Database restore requires TimescaleDB** for the `pg_dump` custom-format dump (hypertables). The plain-CSV and plain-PostgreSQL schema route does not. Both are in the export so the customer chooses.
- The signing public keys and fleet CA are not needed to run a box; the customer can replace the release key set with their own (`keys.d`, [14](14-release-engineering.md)) if they rebuild from source and re-sign. This is documented because "you can rebuild it" must include "and your box will accept your build".

## 6. Walk-away checklist (the customer-facing document, one page)

This table is the source for the one-page checklist printed in the site handover pack:

| # | Step | Command / evidence | Who |
|---|---|---|---|
| 1 | Agree the cut-over date; decide level A, B or C | | customer |
| 2 | Full export to customer-owned media | `skyrangerctl export --to ...` ; verify checksums | integrator + customer |
| 3 | Confirm customer holds the config git remote (or the `config-git/` archive) | repo URL / archive | customer |
| 4 | Confirm customer has **admin** credentials and the root/SSH key on the box; remove ours | users list, `/etc/skyranger/support_authorized_keys` | customer |
| 5 | Customer receives all secrets they need (MQTT, DB, TLS key, PLC credentials) or re-issues them | `--include-secrets` or re-issue | customer |
| 6 | Disable remote access | `skyrangerctl support disable --permanent` removes the tunnel key and support authorized keys | customer / integrator |
| 7 | **Unenroll from fleet** | `skyrangerctl fleet unenroll`; central deletes the record | integrator |
| 8 | Disable cloud sync; request central data deletion or hand over the project DB | `skyranger-cloud` project export (below), then deletion with a written confirmation | integrator |
| 9 | Hand over the release bundle currently installed (and the signing public key fingerprint) | `/var/lib/skyranger/releases/*` | integrator |
| 10 | Run parallel for an agreed period (new system receives the same devices or a copy of the broker stream), compare totals (counters) | `skyranger/v1/{gateway_id}/tags/#` is a read-only internal bus anyone can subscribe to | customer |
| 11 | Decommission: stop the stack, wipe or keep the disk | `skyrangerctl decommission --wipe-data` (confirmation phrase, three steps) | customer |

Steps 6, 7 and 11 require no contact with us. A customer can do 1 to 11 alone using this document.

## 7. Central side exit

For T4/sync customers: central keeps **one database per project** ([09](09-cloud-sync.md) section 5), so:
- `fleetctl project export acme` produces the same export format (history, events, config revisions of every gateway in the project, inventory, no secrets).
- `pg_dump` of the project's database is a complete, restorable copy onto the customer's own central VM (the same images).
- `fleetctl project delete acme --confirm` deletes the database, the certificates, the enrollment records and the project's backups, and prints a deletion statement with timestamps and what was deleted. Backups of central are retained for at most the documented window, stated in the contract (Open questions).

## 8. Testing the exit (so it does not rot)

- **Exit test, CI gate**: on every release candidate, from the simulator site with golden data: run `export`; load `history/` into `postgres:17` **without** the TimescaleDB extension and compare row counts and checksums per tag against the source; re-import `config/` into a fresh gateway and assert an empty diff (tag-model.md lossless rule); restore `backup` onto a clean VM; check that `EXIT-README.md` commands work verbatim (the test extracts the code blocks and runs them).
- **Format stability**: export formats are a **stable surface** under the deprecation policy ([14](14-release-engineering.md) section 3): fields are added, never renamed; `MANIFEST.json` has `export_format_version`.
- **Annual exit drill** (once we have customers): do a full export and a cold-spare restore on a spare box for one real site, with the customer's agreement.

## 9. Things that are NOT lock-in, and things to watch

Not lock-in by design: PostgreSQL/Timescale open schema (documented), YAML config, Sparkplug B and MQTT, Modbus/OPC UA, CSV exports, Docker images in an open bundle, open-source code, no activation or phone-home, config git repo owned by the customer.

Watch list (things that could quietly become lock-in, reviewed at each release):
- HMI screen format (`05-hmi-api`): needs a documented schema and an export; if it is only meaningful to our HMI, the export of screens is a courtesy, not a migration path.
- Python scripts that call Sky Ranger's REST API are portable only in logic.
- Template (UDT) semantics: another platform has its own UDT model; the export includes both the template and the resolved tags, so the instances are not trapped.
- Tag `tag_id`s are ours; the export keeps them, and `path` is the human key elsewhere.
- Dependence on our fleet/support servers: none by design; verified by a CI test that runs a box with all outbound traffic blocked.

## Decisions made
1. One command, `skyrangerctl export`, produces everything in plain formats (YAML, CSV zstd, NDJSON, SQL), runs on a live box, and is verifiable with checksums. No Sky Ranger tool is needed to read it.
2. History is exported as CSV and as a plain-PostgreSQL schema so TimescaleDB is not required to take the data away; the `pg_dump` is an optional extra.
3. Secrets and hashes are excluded by default; `--include-secrets` is encrypted.
4. A generated `EXIT-README.md` tells the customer how to load, verify, and migrate devices and alarms.
5. Walk-away needs no contact with us: unenrollment, tunnel removal and decommission are local commands.
6. Central keeps one database per project so a project export and deletion are clean.
7. The exit test is a CI release gate; export formats are stable surfaces.

## Alternatives rejected and why
- **"Contact us for your data"**: contradicts the selling point and a support burden at the worst moment.
- **TimescaleDB dump as the only export**: needs the same extension and version to read; fine for restore, bad for leaving.
- **Parquet or a proprietary archive as the primary format**: efficient but needs libraries; CSV is universally readable. Parquet may be added as an option.
- **Export through a web page only**: hard to automate and to test; the CLI and the API are the contract, the HMI can call them later.
- **Including password hashes**: not portable between systems, and leaks risk. New system, new passwords.
- **Time-limited or licence-gated exports**: antithetical to the platform.
- **A "migration tool" to a specific competitor's format as part of v1**: unbounded work and unverifiable fidelity (UNVERIFIED competitor formats). We give clean data and documentation; converters can be community or paid work.

## Open questions
- **Licence** (decision 5 defers it): the open licence and the right to run and rebuild the installed release are prerequisites for the promise in section 1, level A. Must be settled before 1.0.
- HMI screens and scripts: where do they live (DB, files, config documents)? The exit layout assumes exportable files; confirm with [../05-hmi-api/](../05-hmi-api/) and [../06-scripting/](../06-scripting/).
- Parquet as an additional history format: worth the dependency? Probably yes for large sites; decide with the first customer exporting more than 50 GB.
- Retention of central backups after project deletion (contract term) and a deletion-certificate format.
- Should the export include `derived_state` history (not just current state) to let another system verify counter totals at cut-over? The `counter_discontinuity` events are in the events export, which covers reset/rollover audit.
- How far to go for Ignition/other-vendor import helpers (UNVERIFIED formats): documentation only for v1.
- Export of very large histories in one tarball is awkward (single-file limits on some media); `--split 4GiB` is a probable addition.
