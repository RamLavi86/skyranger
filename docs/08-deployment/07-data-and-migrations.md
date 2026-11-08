# 07 Data and Config Migration, Backup and Restore

Binding inputs: [tag-model.md](../00-contracts/tag-model.md) section 9 (config revisions, lossless import), [14-release-engineering.md](14-release-engineering.md) (`compat`, manifest), [06-updates-and-rollback.md](06-updates-and-rollback.md) (how migrations interact with rollback). The storage design ([../03-storage/](../03-storage/)) is not written yet; table names below are only those the contracts already name (`config_revisions`, `derived_state`, `events`, `audit_log`, `users`, history hypertable). This document owns the **migration policy**, the backup mechanism and the PostgreSQL/TimescaleDB upgrade procedure, not the schema.

## 1. Migration policy: expand, then contract

A schema change that must be rollback-safe is split across releases.

| Phase | Release | Change | `compat` |
|---|---|---|---|
| Expand | N | add nullable column / new table / new index (CONCURRENTLY where applicable); write both old and new shapes | `backward` |
| Migrate | N | backfill data (batched, resumable, never one giant transaction on a hypertable) | `backward` |
| Contract | N+1 or later | stop reading the old shape (code), then drop the old column/table in a later release | `breaking` for the drop, scheduled at least one minor after the code stopped using it |

`compat` values (the same words used in the manifest and [06](06-updates-and-rollback.md)):
- `backward`: the **previous release** runs correctly against the schema after this migration. Rollback is a binary swap.
- `breaking`: the previous release cannot run against the result. Rollback needs a restore from the pre-update dump.

Rules every migration PR must satisfy (a CI linter checks the mechanical ones):
1. File pair per migration: `NNNN_short_name.sql` and `NNNN_short_name.meta.yaml`.
2. `meta.yaml` declares `compat`, `est_duration` (measured on the golden DB), `needs_backup`, `locks` (what it locks and for how long), `resumable`.
3. Forward-only. There are no down migrations; rollback is the previous binary (backward) or a restore (breaking).
4. Idempotent where the engine allows (`IF NOT EXISTS`), and run inside a transaction unless the statement cannot (see TimescaleDB, section 3).
5. No migration may take a lock that blocks the historian writer for more than a few seconds. Long operations are batched. The historian's spool ([01-topologies.md](01-topologies.md) T2) is the safety net, not an excuse.
6. A migration that rewrites config (`config_revisions` content) must be reversible by a restore and never edits past revisions in place: it adds a new revision or writes a `schema_version`-tagged copy (section 5).

```yaml
# 0042_split_events_hypertable.meta.yaml
id: 0042_split_events_hypertable
compat: breaking
est_duration: 5m          # on the 10 GB golden database
needs_backup: true
locks: "ACCESS EXCLUSIVE on events for ~2s at the end"
resumable: true
notes: "previous release reads events from the old table"
```

### Who runs migrations and when

`skyrangerd` runs pending migrations at start, under a PostgreSQL advisory lock (`skyranger-migrate`, [04](04-install-and-first-boot.md) section 4), from `/usr/share/skyranger/migrations` baked into the image. This makes "start the new image" the only upgrade step and keeps one code path for install, update and restore. The lock means two `skyrangerd` (a T3 mistake) cannot migrate concurrently. A `schema_migrations` table records `id`, `checksum` of the `.sql`, `applied_at`, `release`, and the migration refuses to run if a recorded checksum differs from the file (someone edited history).

Migration tool: the **format** above is ours; the runner is a small Go function (a loop over embedded files plus a table), not a framework dependency. Alternative runners are listed below. A dry run is `skyrangerd migrate --plan` (prints pending migrations, `compat`, estimated durations) and is what `skyrangerctl update` shows the operator before confirmation.

Downgrade protection: if the database contains a migration the running binary does not know (a newer release already migrated it), `skyrangerd` **refuses to start** with a clear error instead of running against an unknown schema, unless the migration's `compat` is `backward` and recorded as compatible with this release's `release_id` list (the migration table stores `min_release_compatible`). That is how a rolled-back previous release runs after a `backward` migration: it declares it knows the migration is backward-compatible with it.

## 2. Compatibility matrix (what the update tooling enforces)

For release R with manifest `upgrade.min_from = M`:
- Direct update from any release `>= M`.
- Update from older than `M`: chain through intermediate releases; `skyrangerctl update` computes the chain from published manifests and applies each with its own gate.
- Rollback target `previous` must satisfy `rollback_to_min <= previous`. If not, the tool says "fast rollback not supported for this release; restore rollback only".

A golden-database test (golden DBs per supported source version, realistic size, stored as dumps in the release pipeline) runs every supported path in CI ([14](14-release-engineering.md) upgrade matrix).

## 3. TimescaleDB specifics

All UNVERIFIED against current TimescaleDB documentation unless stated; check before the storage design is final:

- Extension upgrade is a separate step from the container image change: after starting a new image with a newer TimescaleDB, the extension in each database must be updated (`ALTER EXTENSION timescaledb UPDATE`, which must be the first command in a new session). `skyrangerd`'s first DB connection at start does this as migration `0000_extension_update` before any other statement. A manifest field `versions.timescaledb` tells the runner what it expects; mismatch refuses to start.
- Some DDL on hypertables with compressed chunks has limits (adding columns, changing types, dropping columns); depending on the version, the policy may need decompression first. Any such migration is marked `breaking` or `needs_backup`, and `est_duration` must be measured on a compressed golden DB, not a fresh one.
- `CREATE INDEX CONCURRENTLY` cannot run in a transaction; migrations that need it are marked `transaction: false` in `meta.yaml`.
- Compression and retention are background jobs. A migration must not rename or drop what a job references. After every migration run, `skyrangerd` verifies the jobs list (`timescaledb_information.jobs`, name UNVERIFIED) matches the expected set and logs an error otherwise.
- Continuous aggregates, if the historian uses them, are rebuilt rather than altered in place. Expected to be a source of long `est_duration`.
- Chunk interval changes affect only new chunks; document these as `backward`.

## 4. Backup

`skyrangerctl backup create` is what the nightly timer calls ([02](02-containers.md) section 6) and what updates call as `--tag pre-update`.

### What a backup is

A single directory (and optionally a `.tar`) under `/var/lib/skyranger/backups/<timestamp>-<tag>/`:

```
backup.json               format_version, gateway_id, release, created_at, config_revision, sizes, sha256 of every file
site/site.yaml            copy of /etc/skyranger/site.yaml
site/skyranger.env
config/                   normalized YAML export (05-config-as-code) of the current revision
db/skyranger.dump         pg_dump -Fc of the skyranger database (see below)
db/globals.sql            roles needed (pg_dumpall --globals-only, no password hashes unless asked)
secrets.tar.age           OPTIONAL: /etc/skyranger/secrets encrypted to the operator's age key or passphrase
release.lock              the signed manifest of the running release
```

- **Database**: `pg_dump --format=custom` run inside the postgres container (`docker compose exec`), writing to the bind-mounted backup dir. It takes a consistent snapshot without stopping the stack. Custom format allows selective, parallel restore.
- **Hypertable dumps need TimescaleDB-aware restore**: restoring a `pg_dump` into a database with the extension requires `SELECT timescaledb_pre_restore();` before and `SELECT timescaledb_post_restore();` after (documented by Timescale; UNVERIFIED which versions and whether `pg_restore --single-transaction` is allowed). `skyrangerctl restore` wraps this. Restore always uses the **same TimescaleDB version** as the dump unless the restore target is a newer minor with a matching update step (section 6).
- **Secrets are excluded by default.** A backup that can be copied to a USB stick or an S3 bucket must not hand out the MQTT credentials. `--include-secrets` encrypts to an `age` recipient (age is a small, boring, well-reviewed tool; UNVERIFIED packaging in Debian) or passphrase. Without it, a restore onto a new box regenerates secrets and the integrator re-issues device credentials. This trade-off is stated in the runbook.
- **History size policy** (`site.yaml`):

```yaml
backup:
  schedule: "02:17"          # see skyranger-backup.timer
  keep: { daily: 7, weekly: 4, monthly: 3 }
  history: full              # full | last_30d | none
  destinations:
    - { type: local }
    - { type: s3, endpoint: https://s3.example.net, bucket: sr-gw01, credentials: secret://backup/s3 }   # outbound
    - { type: path, path: /mnt/nas/skyranger }                       # SMB/NFS/USB mounted by the host
```

  `history: last_30d` dumps the config, derived state, alarms, events, audit and only the last 30 days of history chunks (per-chunk dump); older chunks are not in the backup, and the backup says so in `backup.json`. This keeps nightly backups small on large sites. `full` is the default because a backup that quietly omits history is a trap; sizing guidance in [03](03-packaging-and-hardware.md).
- **Retention** is by the `keep` counts, applied after a successful verify. A failed backup never deletes older ones.
- **Verification**: every backup is checksummed at creation. Weekly the agent runs `backup verify --restore-test`: restore the newest backup into a throwaway PostgreSQL container on the same box (limited CPU/IO, `nice`/`ionice`), run the migration check and a row-count comparison, and report pass/fail as a metric and alert ([10](10-observability.md)). An untested backup is a hope, not a backup. The test is skipped when free disk is under the dump size times 2.
- **Off-box copy**: destinations other than `local` are required for the site handover checklist; the HMI System page warns if the last off-box copy is older than 48 h.
- **Not covered**: point-in-time recovery. Nightly backups mean up to ~24 h of history lost on total disk loss (plus whatever the historian spool and devices hold). WAL archiving to object storage (pgBackRest or WAL-G) is a known upgrade path, see Alternatives; the nightly dump is the supported baseline in v1.

### Config-only backup
`skyrangerctl backup create --config-only` is the config export plus `site/` in seconds. The config git mirror ([05](05-config-as-code.md)) gives the same safety continuously if a remote is configured.

## 5. Restore

`skyrangerctl restore --from <backup dir|tar> [--new-box] [--to-release X]`:

1. Preflight as in install: supported host, enough disk, release bundle for `backup.json.release` (or newer) present.
2. Stop the stack. If PGDATA is non-empty, move it to `/var/lib/skyranger/postgres.pre-restore-<ts>` (never delete automatically).
3. Write `site/` files (`--new-box`: regenerates TLS certificates and secrets unless `secrets.tar.age` is provided with its key).
4. Start only `postgres` on an empty PGDATA, create roles, `timescaledb_pre_restore()`, `pg_restore`, `timescaledb_post_restore()`.
5. Start the stack on the **backup's release** first, health gate, then offer the normal update to a newer release if the operator wants it (restoring into a newer release directly works only if the manifest's `upgrade.min_from` allows it; migrations then run on start).
6. Emit an event "restored from backup X" and record the data window that is missing (from `backup.created_at` to now).
7. T2/T3: the standby is rebuilt from the restored primary (`ha rebuild-standby`).

Cold spare target (restate of 01): pre-imaged spare box, restore in under 30 minutes for a 10 GB database (target, UNVERIFIED until measured on the first site).

## 6. PostgreSQL major and TimescaleDB upgrades

PostgreSQL major versions (for example 17 to 18) cannot be swapped by changing the image: PGDATA is major-specific. Policy:

- A PostgreSQL major change is **only** introduced in a product MINOR with `db_major.upgrade_from` in the manifest ([14](14-release-engineering.md)), never in a PATCH, and never together with a breaking schema migration (one risky thing per update).
- Method: **dump and restore** into a new PGDATA with the new major, not `pg_upgrade`. Reason: it is the same code path as backup/restore (tested weekly), it compacts and verifies, and it works across the TimescaleDB image pairing without needing both major binaries in one image. Cost: downtime proportional to DB size (for a 15 GB database expect tens of minutes, UNVERIFIED). Larger sites need a maintenance window and the update refuses to start without confirmation of the estimated time.
- Procedure (`skyrangerctl update` does this automatically when the manifest says `db_major.required != installed`):
  1. Take the pre-update backup with `history: full` and verify it.
  2. Stop the stack; keep old PGDATA in place as `postgres.<oldmajor>`.
  3. Start the new postgres image on an empty `postgres/`; restore.
  4. Start the stack on the new release; gate. Roll back by moving PGDATA directories back and switching `current` (**data written since the switch is lost**; same rule as a breaking migration).
  5. Old PGDATA is removed by the operator after the commit grace period, never automatically.
- TimescaleDB minor upgrades within a PostgreSQL major use the extension-update path in section 3 and are an ordinary release.

## 7. Config schema migration (`apiVersion` changes)

Within `skyranger/v1`, changes are additive only (README conventions); old documents keep working.

For a future `skyranger/v2`:
1. **Pure function**: `migrate(v1 documents) -> v2 documents`, written in Go, in the `skyranger/config` package ([05](05-config-as-code.md) open question), with golden tests (every shipped example and every customer-sanitized config we have as a test fixture).
2. `config_revisions` rows carry `schema_version` (`skyranger/v1`). The migration **adds** a new revision in v2 and keeps the old v1 revisions readable through the same upgrader, so `GET /config/revisions/{n}` of an old revision still returns something meaningful (served as stored, with the `apiVersion` it had).
3. The gateway of the release that introduces v2 serves **both** `apiVersion`s on import (accepts v1, migrates in memory, stores v2) for at least the deprecation period (14 section 3), so old repos and scripts still apply. Export is in v2 by default, `?api_version=skyranger/v1` where a down-conversion is lossless, otherwise an error naming the unrepresentable fields.
4. `skyrangerctl config migrate --to skyranger/v2 DIR` migrates a git repo in place (one commit), so the integrator sees the change as a normal diff.

## 8. Tags, IDs and history across migrations

- `tag_id` (UUIDv7) is immutable and is the history key; no migration may regenerate or reuse one.
- A migration that touches the history hypertable must preserve `(tag_id, ts)` uniqueness semantics (README: history keyed on `(tag_id, ts)`), and quality codes as `smallint` per tag-model.md.
- Quality code additions are additive (codes 192+ are reserved); a release never renumbers an existing code.

## 9. Data retention and disk pressure

Retention is `Gateway.spec.history.default_retention` (tag-model.md). When `/var/lib/skyranger` passes thresholds the agent alerts at 80% and 90%; at 95% `skyrangerd` is asked to run its **emergency retention** policy (the historian owns it; proposal: shorten retention for `sampled` tags before `on_change` tags, never touch config, alarms or events tables) rather than letting PostgreSQL hit a full disk (a full PGDATA volume is the worst failure the box can have). Backups never write to the data volume if less than the dump size times 1.5 is free.

## Decisions made
1. Expand/contract migrations with a mandatory `compat: backward|breaking` and `meta.yaml` per migration; forward-only, no down migrations.
2. `skyrangerd` runs migrations at start under a PostgreSQL advisory lock; a checksum table protects against edited history; it refuses to start on an unknown schema unless the migration declares compatibility.
3. Backup is `pg_dump -Fc` plus config export plus site files, one directory with checksums; secrets excluded by default and optionally age-encrypted; verified weekly by an actual restore test.
4. Restore is the same path as install plus data; a restore writes down the data window that is missing.
5. PostgreSQL major upgrades use dump and restore, are never combined with a breaking migration, and need operator confirmation of the downtime.
6. Config schema migration is a pure, tested function with both old and new apiVersions accepted for the deprecation period; old revisions are never rewritten.
7. History size in backups is a visible setting (`full` by default), never silently truncated.

## Alternatives rejected and why
- **A migration framework dependency as the policy** (golang-migrate, goose, Atlas): they are fine runners (any of them can replace our small loop), but none enforces `compat`, `est_duration` or our rollback story. We keep our format and may adopt a library as the loop later.
- **Down migrations**: rarely tested, wrong in practice for data-bearing changes. Restore plus backward-compat is more honest.
- **`pg_upgrade` for major jumps**: faster for huge databases, but couples our image to two PostgreSQL major binaries and has its own TimescaleDB constraints (UNVERIFIED); dump/restore is exercised weekly anyway.
- **Physical backups (`pg_basebackup`, file copy of PGDATA)**: fast restore, but major-version and architecture bound (amd64 vs arm64 boxes), not readable by a customer. Logical dumps are portable and fit the exit plan.
- **WAL archiving (pgBackRest, WAL-G) as the default**: gives point-in-time recovery, but needs an object store, config, and monitoring on every site. Kept as an opt-in later for sites that need an RPO under a day.
- **Backing up secrets by default**: convenient, but a lost USB stick becomes a credentials leak.
- **Snapshotting the whole box (LVM or VM snapshot) as the backup**: useful before OS upgrades (03), but crash-consistent only and opaque; not a portable backup.
- **Editing old config revisions in place on schema change**: destroys the audit trail the revision table exists for.

## Open questions
- **Storage owner**: where do `schema_migrations` and `min_release_compatible` live and does the historian accept a `last_30d` chunk-level dump? Must align with [../03-storage/](../03-storage/) once it exists.
- **Hypertable restore specifics** (`timescaledb_pre_restore`, compression of restored chunks, chunk ordering) UNVERIFIED for the chosen TimescaleDB version. A restore test in CI against the golden DB settles it.
- Whether `uint64` tags (tag-model.md open question) change the dump/exit format (`numeric(20,0)`).
- RPO target for the first customer: is 24 h of lost history acceptable, or do we need WAL archiving from day one for a regulated utility?
- Emergency retention behavior at 95% disk belongs to the historian; the deployment side only asks for it.
- Whether the backup should be encrypted at rest by default with a key derived from a site secret, given the history itself may be commercially sensitive (security doc).
- `age` availability and licensing in Debian (UNVERIFIED) versus `gpg --symmetric`.
