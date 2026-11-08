# 08 Disk Full, Power Loss, and Corruption

Status: draft. The edge box is a pumphouse PC on a wide-range DC supply, perhaps without a UPS, possibly with a consumer SSD. This file states what the storage layer does about the three things that actually kill field databases: the disk filling, the power failing at a bad moment, and bytes going bad. Hardware recommendations (industrial SSD with power-loss protection, hardware watchdog, RTC, power-fail signal) are owned by [03-packaging-and-hardware.md](../08-deployment/03-packaging-and-hardware.md); this file says what we do **when the hardware falls short**, and how to find out.

## 1. Disk-full: a ladder, not a cliff

PostgreSQL does not degrade gracefully at 100%: a failed WAL write is a PANIC and the server restarts into a recovery that may need more space than exists. Everything below exists so that the disk never reaches that point, and so that if it nearly does, the order in which things are sacrificed is deliberate.

**Measurement.** Every 10 s `skyrangerd` calls `statfs` on the data volume (`/var/lib/skyranger`, its own partition per [02-containers.md](../08-deployment/02-containers.md) section 3) and publishes `_system/gateway/disk_used_pct` (the tag 07-security/08 already proposes) computed as `1 - f_bavail / f_blocks`, i.e. from **blocks available to unprivileged processes**, because the PostgreSQL container does not run as root and hits "full" while root-reserved blocks remain. Inodes are checked the same way (`f_favail`).

### The ladder

| Used | State | Automatic actions | Alarm |
|---|---|---|---|
| < 70% | normal | none | none |
| 70% to 80% | watch | `doctor` shows projected days to full; run any overdue compression jobs now (transient extra space about the compressed size of one chunk, UNVERIFIED) | none |
| >= 80% | **warn** | as above; prune local backups to the newest two; prune old support bundles; delete expired dead letters beyond their cap | medium (`_system/gateway/disk_used_pct`, 07-security/08) |
| >= 85% | warn+ | raise to high; refuse non-essential growth (support bundle creation, new local backup) | high |
| >= 90% | **critical** | apply `storage.disk.full_policy` (below). Default `prune_oldest`: drop oldest raw history chunks one at a time until usage is below 85% or the raw floor `min_raw_retention` (default `30d`) is reached | critical |
| >= 95% | **shed** | writer enters `shedding` (`db_state = 3`): no history inserts, no history spool appends, `Offer` returns `Dropped` (counted, gap recorded). **Ballast file deleted.** Events, alarm transitions, audit, config and `derived_state` continue (all tiny) | critical, plus `_system/historian/db_state` alarm |
| >= 98% or any ENOSPC seen | **emergency** | step 3 of the archive valve (07, section 4: `archive_command = '/bin/true'` if archiving is on); stop compression/retention jobs from starting new work; keep serving live values; `GET /system/health` reports `degraded`; page the integrator | critical, repeated |

Hysteresis: leaving a state requires usage 3 percentage points below its entry level (warn, entered at 80%, ends below 77%; critical, entered at 90%, ends below 87%). `shed`, entered at 95%, ends only below 88%. State changes are events (`skyranger.storage.state_changed.v1`, proposed) and audit records when an automatic prune deleted data (`system.emergency_prune`, proposed action; the audit action list is the security area's).

### `storage.disk.full_policy` (host-level, `site.yaml`)

| Value | Meaning | When to choose |
|---|---|---|
| `prune_oldest` (default) | at 90%, delete oldest raw chunks down to the floor `min_raw_retention`; rollups are untouched. If the floor is reached and usage is still >= 90%, go straight to shed | most sites: today's operation matters more than year-old raw rows, and rollups preserve the long-range picture |
| `stop_history` | at 90%, go directly to shed; never delete history automatically | sites with a retention obligation: they accept losing *new* history over losing *old* history, and they will respond to the alarm |

Both choices are communicated at commissioning (the handover document states which one is configured and what the days-to-full projection was). `prune_oldest` is a **data-loss decision made automatically**, so it is loud: audit record, event with the dropped time range, an alarm that stays active until acknowledged, and a line in the monthly report.

### Ballast

At `skyrangerctl init`, `fallocate` a file `/var/lib/skyranger/ballast.bin` of 1 GiB (256 MiB for the `meter` profile) on the data volume (ext4 or XFS; `fallocate` semantics on btrfs/ZFS differ, and those are not supported data filesystems here). It is deleted at 95% (shed) and recreated when usage falls below 80%. Reason: the operations that rescue a full disk themselves need space (`DROP` of a chunk writes WAL, a checkpoint needs room, vacuum and compression need scratch). Without ballast the rescue fails at the moment it is needed. This is a well-known operational trick for databases on small volumes. `doctor` verifies the file exists with real blocks (`st_blocks`).

### Never shed first

Configuration, audit log, alarm state and journal, events, `derived_state`, `memory_values`. They are kilobytes per day. The ladder sacrifices raw history first (oldest, then newest), rollups only if everything else failed.

### Commissioning check (prevents the ladder from ever being used)

At install and at every config apply that changes retention, `skyrangerctl doctor` computes the **steady-state size** with the formula of 09 (section 9: daily compressed bytes times retention days, plus the uncompressed window, plus rollups, plus 20% working space and the spool cap) from the *measured* rows/day and bytes/row of the last 7 days, and compares it with the volume size. If it exceeds 70% it raises a `high` alarm "configuration does not fit the disk" and states which of retention, deadbands, or disk to change. The ladder is the safety net, not the plan.

## 2. Power loss on a cheap edge box

A clean shutdown is the exception on a pumphouse. Layer by layer:

| Layer | What a power cut does | Protection in this design | Residual risk |
|---|---|---|---|
| SSD volatile write cache | a drive without power-loss-protection capacitors acknowledges `fsync` from RAM and loses it, sometimes reordering writes | hardware guidance (PLP SSD), plus the pull-plug test (section 5) to find out empirically | consumer SSDs can still corrupt PostgreSQL despite `fsync=on`. Nothing in software fully fixes a drive that lies. **This is why the hardware document requires PLP and the test exists** |
| Filesystem | torn metadata or a lost last few seconds of data | ext4 (or XFS) on the data partition with default `data=ordered`, barriers on, `noatime`; never `nobarrier`, `data=writeback` or `fsync=off`. `errors=remount-ro` (section 4) | the filesystem journal protects metadata, not application consistency; PostgreSQL and the spool provide that themselves |
| PostgreSQL | crash recovery replays WAL from the last checkpoint | `fsync=on`, `full_page_writes=on`, `synchronous_commit=on` (01, section 9), data checksums, default `checkpoint_timeout=5min`, `max_wal_size=1GB` | recovery time: WAL since the last checkpoint is replayed at start. At typical rates that is a few MB (09); worst case up to `max_wal_size`. The deployment health check `start_period: 60s` may be too tight on a slow disk: **request `120s`** (cross-area request below) |
| History rows in flight | the ring and unflushed batch vanish | bounded to `flush_max_wait` (500 ms) plus the batch in flight; counters self-heal (03, section 9) | up to about half a second of gauge samples. Accepted, documented |
| Spool (DB down) | torn last frame | CRC frames, torn-tail truncation, sync markers, fsync discipline (04) | at most the 100 ms unsynced tail |
| Small state files (`config-snapshot.yaml`, cursors) | partial write | write temp, `fsync`, `rename`, `fsync` directory | none beyond the previous version |
| `derived_state` | last second of checkpoints lost | the next sample recomputes delta from the checkpointed `last_raw` (tag-model section 7) | counts between a device reboot and the lost checkpoint if both happen together (reported as `reset`) |
| Clock | no RTC: time wrong until NTP | start-up gating on a plausible clock (deployment), storage timestamp sanity window (03, section 8) | rows stamped before a clock step keep their old time; step is evented |
| Docker layers and overlay | irrelevant to data | all data on bind mounts under `/var/lib/skyranger`, never in overlay or named volumes (deployment decision 2) | none |

### Unclean shutdown detection

`skyrangerd` removes `state/clean-shutdown` at start and writes it (fsync'd) at the end of a graceful stop. If the marker is **absent at start**, the previous run did not end cleanly: audit action `system.started` is recorded with `unclean: true`, an event is emitted, and a **quick self-check** runs before the writer begins: spool open and verify (04, section 5), `pg_amcheck` on the small tables, `audit verify`, config drift check. The result is visible in `GET /system/health` for 24 hours. Repeated unclean starts (more than three in 24 hours) raise a medium alarm "power instability" because the cause is physical and the integrator should look at the supply.

### Power-fail signal and hold-up

If the box has a UPS or supercap module with a power-fail input (hardware doc), `skyrangerctl` triggers the **fast stop**, not the full 30 to 60 s graceful stop: flush the writer ring, append the remainder to the spool, `fdatasync`, write the clean-shutdown marker. Budget: **under 2 seconds**, so hold-up time of 5 s is ample. PostgreSQL is deliberately not shut down cleanly in this path: its WAL makes crash recovery safe, and waiting for a checkpoint is the part that does not fit in a supercap. A full graceful stop (checkpoint included) is used when the signal says the supply is gone for good and time remains (UPS runtime).

## 3. Corruption: detect early, scope the damage

### Detection

| Mechanism | Finds | Cost |
|---|---|---|
| PostgreSQL data checksums (`initdb --data-checksums`) | silent page corruption on read (SQLSTATE `XX001`), including by vacuum, dumps and queries | small CPU, enabled at init |
| Logical backup (nightly `pg_dump`) | reads every heap page of every table, so checksum failures in data surface within 24 hours even for cold chunks. A nightly dump is therefore also a full verification scan of table data (not of indexes) | already paid for |
| `pg_amcheck` (verifies heap and B-tree structure; PostgreSQL 14 and later; needs the `amcheck` extension, believed to be in contrib in the image, UNVERIFIED) | corrupt indexes and heap structure | weekly on the small tables; monthly on the primary-key indexes of the newest 7 days of `history` chunks; on demand by `doctor` |
| Spool frame CRCs | spool damage | on read |
| Config and audit hashes | altered or damaged documents, a broken chain | `audit verify` daily (07-security/08), manifest hash check on every config read |
| Physical backup (`pg_basebackup`) | believed to verify checksums while copying (UNVERIFIED) | weekly |
| Read-only filesystem probe | ext4 `errors=remount-ro` events | write `state/.rwprobe` every 30 s, alarm if it fails |

### Response by class

| Finding | Writer behavior | Scope of damage | Recovery |
|---|---|---|---|
| Spool frame bad | skip to next marker, count, continue (04) | that frame's samples | none needed; gap event |
| Index corrupt | writes to that table fail with `XX002`; writer spools (bounded), does not retry, raises `_system/storage/corruption` (critical, with SQLSTATE and relation) | none lost, just not stored yet | `REINDEX INDEX CONCURRENTLY` on the named index (Admin function, audited), then drain the spool |
| Heap page corrupt in a **history chunk** | queries touching that chunk fail; inserts to old chunks unaffected | at most that chunk's raw rows (one chunk interval). **Rollups are stored separately and survive** | try to salvage readable rows into a new table, otherwise drop the chunk (`drop_chunks` for that range) and record the gap (event with range); the raw gap is covered by the 15-minute rollup if the rollup is intact |
| Heap page corrupt in the **current** (hot) chunk | inserts into that chunk fail (index or page) | recent raw rows | stop the writer's history inserts (spool), restore from backup per 07, replay the spool. The spool bound (04) is the time the integrator has |
| Corruption in `config_*`, `audit_log`, `derived_state`, `alarm_*` or the Timescale catalog | affected operations fail closed | that table | restore from backup (07); configuration is recoverable from the snapshot file and git regardless |
| `fsync` failure on the spool | segment abandoned, frames re-appended, no retry on the same file (04, section 4) | none if the re-append succeeds | `_system/storage/spool_io_errors`; two consecutive failures put the stream in shed |
| Filesystem remounted read-only | everything that writes fails | writes since the event | reboot (fsck) or manual repair; shed state meanwhile; live values keep working |

PostgreSQL's own behavior on `fsync` failure (panic and recover from WAL) is relied on, not worked around.

Salvage tools (`ignore_checksum_failure`, `zero_damaged_pages`) are destructive last resorts that lose the unreadable page. They are named here so a support engineer knows they exist, and are **not** automated. Their exact behavior on the pinned version is UNVERIFIED.

## 4. What the engine keeps doing

When any of the above takes the database away: drivers, the tag engine, live values, WebSocket, MQTT internal bus, alarm evaluation and notifications (with the alarm engine's own in-memory state and the fsync'd alarm spool) continue. History goes to the spool (until its cap or until shed). The HMI banner shows `database: degraded` from `/system/health`. This is the practical meaning of "edge-first, trust and reliability are the product": a storage failure must be a degraded-history incident, not a loss of visibility into the plant.

## 5. The pull-plug test (acceptance criterion for any hardware we ship)

Equipment: a relay-switched supply (smart plug or programmable bench PSU) controlled by a test host; the unit under test running the full stack with the simulator at the `typical` rate (rows/s scaled to the profile) and a **pulse-meter simulator that logs ground truth on the test host** (every emitted pulse count and boot count, with timestamps).

Cycle: power on, wait until `GET /system/health` is `ok`, run for a random 20 to 120 s, cut power without warning, wait 5 s, repeat. **200 cycles for a candidate box, 1,000 for a qualification.** Include cuts during: a compression job, a retention drop, a config apply, a spool replay (arranged by stopping PostgreSQL first).

After every cycle (cheap checks) and every 20th cycle (deep checks):

| Check | Pass condition |
|---|---|
| PostgreSQL starts and accepts connections | every cycle, within the health `start_period` |
| Spool opens | no mid-file corruption reported; torn tail truncated silently |
| `audit verify` | chain intact (deep: every 20th; cheap: head only) |
| Config head | `manifest_sha256` equals the one before the cut |
| `pg_amcheck` small tables and newest chunk PKs (deep) | clean |
| Counter total | exactly equal to ground truth after the device is observed again (no device reboot), and flagged `reset` with bounded loss when the simulator also rebooted |
| History vs ground truth | for periods when the gateway was up: no rows missing except in the final `flush_max_wait` + one batch before each cut; **zero duplicate rows; zero rows outside the sanity window; zero rows with values not in the simulator's log** |
| Time to ready | median and p95 recorded (the number for the handover document) |

Run the full test twice on candidate hardware: with the drive's volatile write cache as shipped, and with it disabled (vendor tool; the effect of `hdparm -W 0` on NVMe or SATA industrial parts is UNVERIFIED). If the first run fails and the second passes, the box ships only with the cache disabled and the performance cost measured. If both fail, the box is rejected. The same test on a deliberately cheap consumer SSD, kept in the project as a **negative control**, shows what failure looks like (the aim is that the team recognizes it).

## Decisions made
1. A disk-full ladder (80/85/90/95/98%), computed from `f_bavail`, with a default of `prune_oldest` to a `30d` raw floor and an alternative `stop_history`; both loud, audited and alarmed.
2. A ballast file (1 GiB, 256 MiB on `meter`), deleted at 95% and recreated below 80%, so rescue operations have room.
3. Configuration, audit, alarms, events and counter state are never shed before history.
4. A commissioning-time capacity check prevents the ladder from being the normal mode.
5. Unclean shutdown is detected by a marker file and triggers a self-check; repeated unclean starts raise an alarm.
6. A power-fail signal triggers a fast stop (flush, fsync, marker) in under 2 seconds; PostgreSQL relies on WAL, not a clean shutdown.
7. Corruption response is scoped by table: index (reindex), history chunk (salvage or drop, rollup survives), critical tables (restore).
8. The hardware acceptance criterion is a pull-plug test with ground truth, not a datasheet.

## Alternatives rejected and why
- **Letting PostgreSQL fill the disk and recover by hand**: crash loops on a remote box.
- **Silently deleting old history at a fixed threshold with no policy choice**: a data-loss decision made behind the customer's back. The default is automatic but visible and switchable.
- **Quota via a dedicated loopback filesystem or LVM thin volumes for each component** (so the spool cannot starve PostgreSQL): attractive isolation, but one more storage layer to fail on customer hardware. Caps in software plus the ballast are simpler. Could revisit with the appliance image (P2).
- **`fsync=off` or `synchronous_commit=off` for write speed**: trades the one property we sell for throughput we do not need.
- **RAID1 inside the box as the answer to disk failure**: valid for T2 servers; on a single-board edge box the answer is a second target for backups and the cold spare in 01-topologies.
- **Relying on a UPS alone**: a UPS that runs flat without telling anyone only moves the cut (deployment hardware doc). Both the signal and the crash-safe design are needed.
- **Auto-salvage with `zero_damaged_pages`**: silent data loss with a clean-looking result.

## Open questions
- **Cross-area requests:** health-check `start_period: 120s` for the postgres service; `POSTGRES_INITDB_ARGS=--data-checksums`; ballast creation and `doctor` checks in `skyrangerctl init`; the `/var/lib/skyranger` partition and `f_bavail` semantics; `11-time-and-power` (referenced by deployment, not yet written) should adopt the fast-stop budget and the clean-shutdown marker.
- New contract surface: `_system/storage/*` tags (`corruption`, `spool_io_errors`, `backup_failed`, `archive_failing`, `days_to_full`), events `skyranger.storage.state_changed.v1`, audit action `system.emergency_prune`, `system.started{unclean}`. All additive, all need owners to accept.
- Is a default of `prune_oldest` right for a water utility with an unknown retention obligation (UNVERIFIED)? An argument can be made for `stop_history` as the safer default for regulated customers and `prune_oldest` for integrators' own demo boxes.
- Effect of disabling a drive's volatile cache on an NVMe or SATA industrial SSD (UNVERIFIED), and whether fsync latency can be a usable heuristic for "the drive is lying" (probably not).
- `pg_amcheck` availability and runtime on the pinned image (UNVERIFIED); whether weekly is too frequent on a 4 GB box.
- Hysteresis numbers are first guesses; tune with the fault-injection tests.
