# 07 Backup and Restore

Status: draft. The host-level `skyrangerctl backup` command, the nightly timer, rotation and targets, and the whole-box restore (`skyrangerctl restore --from backup.tar`, site identity, secrets) are owned by [08-deployment](../08-deployment/README.md) (files 02, 04, and the not-yet-written 07-data-and-migrations). This file owns **the database part**: what it contains, how it is produced, how it is verified, how a restore behaves, and what is and is not recoverable. Commands marked proposed are not yet in the deployment doc.

All TimescaleDB restore procedures here are from memory of the TimescaleDB documentation and are **UNVERIFIED** until exercised against the pinned image. Restore is the single most important thing to test early; milestone 1 includes a restore drill (section 8).

## 1. Goals

| Data | Where else it exists | Target RPO (T1 default) | Notes |
|---|---|---|---|
| Configuration | snapshot file after every apply, git mirror (05) | **0** (last applied revision) | the database is not the only copy |
| Audit log | off-box syslog checkpoints, monthly paper anchor (07-security/08) | last backup, chain verifiable | gaps are detectable, not recoverable |
| Counter state (`derived_state`) | self-heals from the next device sample (below) | none needed while the device did not reboot | |
| History | nowhere else | **24 h** with nightly logical backup; minutes with physical + WAL archiving | the dominant exposure |
| Alarm journal and events | nowhere else | 24 h | small |
| Live values, alarm current state | rebuilt from devices within one heartbeat, re-evaluated | 0 | |

RTO is dominated by restore time, which scales with compressed database size: reinstall plus `pg_restore` of a few hundred MB (meter) is minutes; 74 GB (10,000 tags, 400 days, 09) is a restore of that size at the disk's sequential write speed plus index rebuild (UNVERIFIED; measure at milestone 1 and put the number in the handover document). The honest statement to customers: *the plant keeps running without the gateway; the gateway is restored from the last backup; history since that backup is lost; configuration and totals are not.*

## 2. What a database backup contains

A backup is a directory (inside the deployment's backup tar), produced by one consistent run:

```
db/
  MANIFEST.json            { created_at, gateway_id, config_revision, audit_head: {seq, hash},
                             postgres_version, timescaledb_version, schema_migrations: [...],
                             mode: "logical" | "physical", sizes, tool versions }
  skyranger.dump           pg_dump -Fc output          (logical)    -- or --
  base.tar.zst + wal/      pg_basebackup + archived WAL (physical)
  config-head.yaml         GET /config export of the head revision (05), independent of the database format
  audit-head.json          the latest signed checkpoint record (07-security/08 section 3)
  tags.csv                 pub.tags export: tag_id, path, datatype, unit (so history can be interpreted without the database)
  SHA256SUMS               hashes of all of the above
```

Why `config-head.yaml`, `audit-head.json`, `tags.csv` next to the dump: if the database part of a backup is damaged or the Timescale version cannot be reproduced, the customer still gets configuration, an audit anchor and a mapping from `tag_id` to names in plain text.

Not in the database backup: the spool (transient), Mosquitto retained messages (rebuilt), secrets and site identity (host backup, security doc), logs.

## 3. Logical mode (default while the database is under 20 GB)

```
pg_dump --format=custom --compress=zstd:3 --no-owner --dbname=skyranger -f skyranger.dump
```

(`zstd` compression in `pg_dump` is believed to need PostgreSQL 16 or later, UNVERIFIED; gzip is the fallback.)

- `pg_dump` takes one MVCC snapshot, so it is consistent without stopping the gateway. It runs as a dedicated role with `SELECT` on everything (`sr_backup`, proposed).
- A dump of a TimescaleDB database must be restored with the documented procedure: create an empty database with the same extension version, `SELECT timescaledb_pre_restore();`, `pg_restore`, `SELECT timescaledb_post_restore();` (UNVERIFIED names and exact steps; check the TimescaleDB "backup and restore" documentation for the pinned version). **The restore container is always the image digest recorded in `MANIFEST.json`** (`release.lock` records it per release), never "latest".
- Compressed chunks are believed to be dumped in their compressed form, so dump size tracks compressed size (UNVERIFIED).
- Long dumps hold back vacuum on the small hot tables (`derived_state`, `alarm_state`) for the duration. At typical counts this is bounded bloat that the next autovacuum clears; it is another reason the logical mode has a size ceiling.
- **Why 20 GB:** beyond it the nightly dump costs tens of minutes of sustained disk I/O on a small box, competing with the writer and replay. The threshold is a planning default (UNVERIFIED), configurable; `skyrangerctl doctor` recommends the switch when the last dump exceeded 15 minutes.

## 4. Physical mode (large databases, or sites that need minutes of RPO)

- `pg_basebackup` weekly plus continuous WAL archiving (`archive_mode=on`, `archive_command` copying into `/var/lib/skyranger/backups/wal/` and to the second target). Point-in-time recovery is then possible to any moment since the base backup: for example "just before someone shortened retention", and the recovery target is chosen with `recovery_target_time`.
- Restore requires the **same PostgreSQL major version and the same TimescaleDB version** as the backup (physical copies are binary). `MANIFEST.json` records both.
- **Archiving can kill the box.** If the archive target fills or disappears, `archive_command` fails and PostgreSQL keeps every WAL segment, filling `pg_wal` until the disk is full, at which point PostgreSQL panics. Therefore:
  1. The WAL archive directory has a quota well below the data volume (default `20%` of the volume) and prefers a separate disk.
  2. `pg_stat_archiver` is read every minute; `last_failed_time` newer than `last_archived_time` for 10 minutes raises a high alarm (`_system/storage/archive_failing`, proposed).
  3. The emergency valve: `ALTER SYSTEM SET archive_command = '/bin/true'; SELECT pg_reload_conf();` (changing `archive_command` needs only a reload, believed, UNVERIFIED), documented in 08 as step 3 of the disk ladder. Archiving gaps are then recorded and the next base backup is forced.
- The replication slot used by a T3 standby (01-topologies) has the same hazard and is bounded by `max_slot_wal_keep_size` (01, section 9).
- A **standby is not a backup**: it replicates `DROP`, `DELETE`, and a mistaken retention change instantly.

## 5. Schedule, space, and where backups go

- Nightly at a time chosen at install (default 02:30 local) via the systemd timer ([02-containers.md](../08-deployment/02-containers.md)). Rotation proposal (deployment owns it): 7 daily, 4 weekly, 12 monthly.
- **A backup must never harm the box.** Pre-checks: free space on the target at least `1.2 x` the size of the last backup, and local backup space capped at 20% of the data volume. If a check fails: skip, raise `_system/storage/backup_failed` (medium), keep the older backups. Backups never delete the last good one before the new one has verified (section 6).
- Local backups on the same disk as the database protect against corruption and mistakes, not against disk or box loss. `skyrangerctl doctor` warns when no off-volume target is configured (second disk, SMB/NFS share, S3-compatible bucket, as in 01-topologies T1).
- Backups hold audit and configuration; they are confidential. Encryption before leaving the box (public-key, so the box cannot decrypt its own offsite copies) is a security-area decision ([06-secrets](../07-security/06-secrets.md)).
- `system.backup_created` and `system.backup_restored` are audit actions already (07-security/08), with `MANIFEST.json`'s hash in `after`.

## 6. Verification (a backup that was never restored is a hope)

| Level | When | What | Cost |
|---|---|---|---|
| L1 | every backup | `SHA256SUMS` check; `pg_restore --list skyranger.dump` (reads the table of contents); `MANIFEST.json` parse; size within 50% to 200% of the previous | seconds |
| L2 | weekly | restore **only the small tables** (`config_*`, `audit_log`, `derived_state`, `memory_values`, `tag_catalog`, `alarm_state`) into a scratch database in a scratch container using `pg_restore --table=...`, run `skyrangerctl audit verify` (hash chain) and compare `config-head.yaml` with an export of the restored config (empty diff) | minutes, small disk |
| L3 | monthly if free space >= 2 x database size, else quarterly on a spare machine | full restore into a scratch container, `ANALYZE`, row counts per hypertable within 1% of the manifest, one sampled range query compared with the original (`SELECT count(*), min(ts), max(ts), sum(v_f)` over a closed day) | long |
| Drill | before go-live and every year | full runbook of section 7 on a spare box, timed | hours |

Results are recorded in `_system/storage/last_backup_ok_age_s` and `last_restore_test_age_s` with alarms at 36 h and 45 days.

## 7. Restore runbook (database part of a T1 restore)

1. **Stop** `skyrangerd` (`skyrangerctl stop`); leave Mosquitto running. Devices continue publishing; their cumulative counters are not lost (messages.md section 6), but nothing is stored until the gateway is back.
2. **Preserve evidence.** Move the existing data directory to `/var/lib/skyranger/postgres.damaged-<date>`. Never restore over it. (If the disk is too full to keep it, an operator decision is required, and the runbook says so explicitly.)
3. Start an empty PostgreSQL from the digest in `MANIFEST.json`. Logical: `timescaledb_pre_restore()`, `pg_restore --exit-on-error --no-owner --dbname=skyranger skyranger.dump`, `timescaledb_post_restore()`. Physical: extract the base backup, configure `restore_command` and the recovery target, start, wait for promotion.
4. **Migrate forward** if the backup is older than the running release (`skyrangerd migrate`, expand/contract policy of deployment 07).
5. **Verify before serving:** `skyrangerctl audit verify` (chain intact, newest checkpoint signature valid); config drift check (05, section 6) against `config-snapshot.yaml`; `SELECT count(*)` sanity on hypertables; `ANALYZE`.
6. **Start `skyrangerd`.** It loads the head revision, replays any spool (older rows dedupe, newer rows fill the gap), rebuilds live state from devices (rebirth requests, messages.md), and the writer refreshes the rollup range from the backup time to now (`RefreshAggregates`, 02).
7. **Operator checks:** compare each billing meter's `total_volume` with its mechanical register and, if they differ, apply the audited `manual_adjust` write (tag-model section 7). Review the drift decision from step 5. Record the incident in the site log.

### What is lost, what self-heals

- **History** between the backup and the failure is lost, except what the spool still holds (rows spooled during the outage and not yet replayed are in `spool/`, which survives a database loss on the same volume). Devices do not buffer, so there is no other source.
- **Counter totals self-heal** while the device did not reboot since the backup: `derived_state` is older, the next sample's `raw - last_raw_old` includes every pulse since (tag-model section 7, and the reason the water meter design suits recovery). If the device **did** reboot in the gap, the pulses between the last pre-reboot sample the backup knew about and the reboot are unrecoverable: the total is understated by at most those pulses, and a `counter_discontinuity{kind: reset}` is emitted. Step 7's register comparison is exactly the fix.
- **Alarm current state:** the restored `alarm_state` may be older. The engine re-evaluates every condition from live values; acknowledgements made since the backup are lost and the corresponding journal rows are missing (the gap is visible as a hole between `alarm_journal.ts` and now, and the audit log shows the gap if audit also lost rows).
- **Audit:** the restored chain is verifiable up to the backup's head. Records after it are lost; off-box checkpoints (syslog, monthly paper anchor) prove how far the chain legitimately extended.

## 8. Exit export (the customer leaves, or wants a copy)

Owned end to end by [08-deployment/12-exit-plan.md](../08-deployment/README.md) (not yet written); the storage side guarantees the following is possible with stock PostgreSQL tools and nothing of ours:

```
# configuration and identity
skyrangerctl config export > config.yaml                                   (05)
\copy (SELECT * FROM pub.tags) TO 'tags.csv' CSV HEADER

# history, one file per month, plain CSV, any spreadsheet or database can load it
\copy (SELECT path, ts, value_float, value_int, value_text, quality, ts_source, ts_gateway
       FROM pub.history WHERE ts >= '2026-01-01' AND ts < '2026-02-01' ORDER BY path, ts)
  TO 'history-2026-01.csv' CSV HEADER

# rollups: a fraction of the size, and what most analyses want
\copy (SELECT ... FROM history_15m ...) TO 'history_15m.csv' CSV HEADER
\copy (SELECT * FROM pub.events ...) / pub.alarm_journal ...
```

- A CSV row is about 90 to 100 bytes. 10,000 tags at the typical rate is about 1.1 GB of CSV per day (compresses roughly 8x with `zstd`). Full raw export of 400 days is therefore hundreds of GB for a large site: the tool supports `--since`, `--rollups-only`, and resumable per-month files. For the water meter the entire raw history is under 100 MB of CSV.
- A schema README (`export/README.md`) ships inside the export describing each file and column, quality codes (`quality_codes`), the encoding of `uint64` and `datetime` (01, section 4), and the meaning of `keepalive`.
- A full `pg_dump` is **not** the exit format (it requires the same TimescaleDB to restore). It is a backup format. CSV (and optionally Parquet later) are exit formats.
- The customer's own PostgreSQL (T2) means their DBA owns backups of the database (01-topologies); our backup command still dumps remotely by default, and the handover document states who is responsible.

## 9. Tests

- **Restore drill in CI:** build a database with a week of simulator history (compressed and uncompressed chunks, rollups, config revisions, audit chain), take logical and physical backups, restore each into a fresh container of the recorded image digest, run L2 and L3 checks. Fails the build if restore fails.
- **Backup under load:** dump while the simulator writes at the `typical` rate; assert write latency p95 stays within 2x.
- **Disk-pressure backup:** fill the target to leave less than `1.2 x`; assert skip plus alarm and older backups kept.
- **Version drift:** restore a backup from release N into release N+1 (migrates forward) and refuse to restore into N-1.
- **Counter recovery:** restore an older database during a simulated pulse stream and assert the total equals the true count (no reboot case) and flags a `reset` (reboot case).

## Decisions made
1. The database backup is a self-describing directory (manifest, dump or base backup, plus plain-text config, audit head and tag map), so partial recovery is possible even when the dump is not restorable.
2. Logical `pg_dump -Fc` is the default below about 20 GB; physical `pg_basebackup` plus WAL archiving above that or when RPO must be minutes.
3. Restore always runs in the container image digest recorded in the backup, always preserves the damaged data directory first, and always verifies the audit chain and config drift before serving.
4. Verification is tiered: every backup, weekly small-table restore, monthly or quarterly full restore, yearly drill.
5. A backup must never endanger the box: space pre-checks, quotas, and an archive-failure alarm with a documented emergency valve.
6. Exit format is CSV (plus rollups), not a database dump.

## Alternatives rejected and why
- **Standby replica as the backup**: replicates mistakes instantly.
- **Filesystem snapshots (LVM, ZFS, btrfs) of a running PostgreSQL**: excellent when crash-consistent snapshots are verified, but they tie the product to a filesystem choice on customer hardware and restore tests are harder. LVM snapshots are used for OS rollback by deployment (03-packaging); for the database, logical and physical backups are portable. Could be added as a third mode.
- **`pg_dump` per hypertable or per chunk for incremental backups**: complex to keep consistent with the TimescaleDB catalog.
- **Backing up only configuration and treating history as disposable**: acceptable for some sites, wrong as a default for a billing meter. Customers who want it can set backup scope in `site.yaml` (open question).
- **Trusting backups without restoring them**: see section 6.
- **Parquet export in v1**: attractive for analytics, adds a Go dependency and a format to maintain; CSV is universally loadable and enough to leave.

## Open questions
- All Timescale restore procedure details (UNVERIFIED), and whether compressed chunks dump in compressed form.
- Real dump and restore throughput on the target hardware; the 20 GB threshold and the 15 minute `doctor` trigger are planning numbers.
- `sr_backup` role and `skyrangerctl backup verify`, `doctor` recommendations are proposed commands for the deployment doc.
- Backup scope option (`full` vs `config-only`) and its warning text.
- Whether to add a filesystem-snapshot mode for customers with ZFS or LVM-thin and verified procedures.
- Backup encryption design (security area): who holds the key, and how a restore on a new box obtains it.
- PostgreSQL 17 `pg_dump --compress=zstd` availability is UNVERIFIED; also whether `pg_basebackup` compression options on the pinned image match the flags above.
