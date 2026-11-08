# 06 Updates and Rollback

Binding inputs: [02-containers.md](02-containers.md) (releases directory, `current`/`previous` symlinks, units), [03-packaging-and-hardware.md](03-packaging-and-hardware.md) (RAUC A/B on the appliance), [14-release-engineering.md](14-release-engineering.md) (manifest, `compat`, `min_from`, channels), [07-data-and-migrations.md](07-data-and-migrations.md) (migration rules, backups).

Two update layers, deliberately separate:

| Layer | What changes | Boxes | Rollback unit |
|---|---|---|---|
| **App release** | `skyrangerd`, drivers, HMI, broker and DB images, compose file, `skyrangerctl`, host unit files | all (P1, P2, P3) | previous release directory plus, if needed, a pre-update database backup |
| **OS image** | kernel, rootfs, Docker engine | P2 appliance only | RAUC inactive slot (bootloader fallback) |

On P1, the OS is Debian: security patches via `unattended-upgrades` (security suite only, no automatic reboot), major upgrades are planned jobs with an LVM snapshot or disk image first ([03](03-packaging-and-hardware.md)). Kernel and Docker packages are held back by apt pinning so an unattended upgrade cannot change the container runtime under a running site (hold list written by `install`, reviewed per release).

## 1. Principles

1. **An update is a transaction with a gate.** Prepare, switch, verify, then commit or roll back. Every state is recorded on disk so power loss in the middle is recoverable.
2. **Nothing is downloaded at switch time.** The bundle is fetched (or copied from USB) and fully verified first. The switch itself needs no network.
3. **The previous release stays on disk and runnable** until the next update replaces it.
4. **Rollback without a data restore is guaranteed only across `compat: backward` migrations** (summary decision 5 in the README). Breaking migrations are gated, backed up, and announced.
5. **Config changes and software changes are separate** transactions with separate rollbacks (config: [05](05-config-as-code.md) section 6).
6. **The operator decides when**, unless a site opts into maintenance windows. No update interrupts a site mid-shift by surprise.

## 2. Release layout on disk (recap and additions)

```
/var/lib/skyranger/releases/
  0.4.0/                 unpacked bundle: manifest.json(.sig), compose.yaml, skyrangerctl, host/
  0.5.0/
  current  -> 0.5.0
  previous -> 0.4.0
/var/lib/skyranger/state/
  update.json            update state machine (section 4)
  boot-health.json       last N boot/gate results
  pre-update/<id>/       db dump + config export taken before a breaking update
```

Images are loaded into Docker from `images.oci.tar` (or pulled by digest with `--online`) **before** the switch. Old images are pruned only after the update is committed, keeping the images of `current` and `previous` (disk budget in section 9).

## 3. App update procedure

`skyrangerctl update [--bundle FILE | --source URL | --to VERSION]` (the agent runs the same code for fleet and windowed updates).

```
 0 preconditions     verify manifest + bundle signature (14), channel allowed, not on advisories "do_not_install",
                     current version >= manifest.upgrade.min_from (else compute intermediate steps and ask),
                     free disk >= images + dump + 20%, host skyrangerctl version == target (self-update step, below),
                     stack currently healthy (do not update a sick box; fix first, or --force)
 1 stage             docker load images, verify digests == manifest; unpack to releases/<new>/
 2 backup            skyrangerctl backup create --tag pre-update (always: config export + DB dump).
                     For any migration with compat: breaking this is the rollback point, verified by checksum.
 3 announce          write update.json {phase: switching}; log to journal and the fleet channel (events: see Open questions)
 4 switch            systemctl stop skyranger (compose down, 60 s grace so postgres checkpoints and
                     skyrangerd checkpoints derived_state), ln -sfn <new> releases/current, previous -> old,
                     systemctl start skyranger (compose up --wait). skyrangerd runs migrations under the
                     advisory lock at start ("skyranger-migrate", see 04 section 4).
 5 gate              health gate (section 5), default window 10 min
 6a commit           gate passed: update.json {phase: committed}, boot-health.json entry, prune old images,
                     keep releases/<old> as `previous`; mark RAUC slot good if an OS update is in the same transaction
 6b rollback         gate failed: section 6
```

Self-update of the host tool: `skyrangerctl` is part of the bundle. Step 0 runs the **new** bundle's `skyrangerctl` for steps 1 to 6 (`releases/<new>/skyrangerctl update --continue`), so a fix to the updater ships with the release that needs it. The installed `/usr/local/bin/skyrangerctl` is replaced (atomically, rename) only in 6a. `previous` keeps its own copy so rollback uses the old tool with the old release.

Downtime: step 4 stops the stack. Expected: tens of seconds for a patch, minutes if migrations run. During downtime, devices keep their own counters (water meter publishes cumulative counts; the next sample recovers volume, tag-model.md `counter_total`) and the historian spool is not used (the gateway is down). Data sources that cannot buffer have a gap; this is stated in the release notes for any update with a long migration (`est_duration` in the manifest, shown to the operator before they confirm).

## 4. State machine and crash safety

`update.json` is written (fsync, atomic rename) at every phase boundary:

```json
{
  "id": "01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a",
  "from": "0.4.0", "to": "0.5.0",
  "phase": "switching",
  "started_at": "2026-11-07T14:03:22.123Z",
  "pre_update_backup": "pre-update/01933b7e",
  "breaking_migrations": false,
  "gate": { "window": "10m", "started_at": null }
}
```

Phases: `staged`, `backed_up`, `switching`, `gating`, `committed`, `rolling_back`, `rolled_back`, `failed_needs_human`.

On agent start, if `phase` is not terminal:
- `staged` or `backed_up`: nothing switched; abandon cleanly (delete staged release on request).
- `switching` or `gating`: the box lost power or the updater died mid-update. The agent **resumes the gate**: if the new release is running and passes the gate for its window (the window restarts, since a reboot invalidates observations), commit; otherwise roll back.
- `rolling_back`: continue the rollback; it is idempotent.
- `failed_needs_human`: leave the box running on whatever is healthy, raise a critical alert ([10](10-observability.md)).

## 5. The health gate

A gate pass is a conjunction. The agent evaluates every 10 s for the whole window (default `10m`, per-release override in the manifest, minimum `3m`); **any hard failure fails the gate immediately**, soft conditions must hold at the end.

| Check | Type | Definition |
|---|---|---|
| Containers | hard | every compose service `running`; none restarted more than twice in the window; `healthy` where a healthcheck exists |
| Gateway health | hard | `GET /api/v1/system/health` reports `status: ok` for the last 3 consecutive minutes of the window (a transient `degraded` while drivers connect is allowed earlier) |
| Migrations | hard | migration table at the target; no migration error logged |
| Connections | hard | every `Connection` that was `connected` before the update (snapshot taken in step 2 from `GET /connections`) is `connected` again by the end. Connections that were already down are ignored (we do not blame the update for a broken PLC) |
| Data flow | hard | `good`-quality tag update rate at the end is at least 80% of the pre-update rate measured over the 5 min before step 4 (from `/metrics`, see [10](10-observability.md)); historian spool is not growing |
| Config | hard | `GET /system/info` `config_revision` equals the pre-update revision (an update must never change config) |
| Clock | soft | chrony reports synchronized ([11](11-time-and-power.md)) |
| Logs | soft | error-level log rate under 10x the pre-update baseline |
| Disk | soft | `/var/lib/skyranger` below 90% |

On the appliance, passing the gate also **marks the RAUC slot good**; a failed gate or a crash before the mark means the bootloader falls back on the next reboot ([03](03-packaging-and-hardware.md)).

Operators can extend the window (`--gate-window 1h` for critical sites) or add a manual confirmation step (`--require-confirm`: gate passes technically, then waits up to 24 h for `skyrangerctl update confirm`, rolling back on timeout). That fits sites where only a human can verify a pump station is controlled correctly.

## 6. Rollback

Triggered by a failed gate (automatic), by `skyrangerctl rollback` (manual, within the commit-grace period, section 7), or by the fleet service. Decision tree:

```
 any migration applied by this update with compat: breaking ?
   │
   ├─ no  (all backward) ──► "fast rollback":
   │        compose down; ln -sfn previous current; compose up --wait.
   │        Database untouched. Data collected since the switch is KEPT.
   │        Old release must tolerate the new schema: guaranteed by the compat contract
   │        and verified by the rollback matrix gate (14).
   │
   └─ yes ───────────────────► "restore rollback":
            compose down; restore pre-update DB dump (07); previous -> current; compose up --wait.
            Data collected since the switch is LOST (bounded by the gate window, usually minutes;
            reported in the rollback event with the exact interval). Counters recover: pulse totals are
            cumulative and re-derived from the next device sample (tag-model.md counter_total);
            alarm states re-evaluate from live values; config is unchanged (config gate check).
```

After either path: health check against the *old* release, record `rolled_back` in `update.json`, emit an event and alert, keep the failed release directory for the support bundle ([13](13-supportability.md)), and mark the version as `failed` in the local state so the fleet/agent does not retry it unattended.

Breaking migrations are only run with an explicit operator action (`--allow-breaking`, or a fleet rollout whose policy says so). The tool prints the estimated duration, the data-loss window of a restore rollback, and requires a verified pre-update dump.

## 7. Commit grace and manual rollback

After a committed update the box keeps `previous` and its images. `skyrangerctl rollback` works until the **next update**, with the same decision tree. If the update had breaking migrations and meaningful time has passed (more than the gate window), the tool warns that a restore rollback would discard everything since the pre-update dump and requires `--accept-data-loss-since <ts>`. Otherwise (all backward), it is a fast rollback with no data loss.

## 8. OS A/B updates (appliance, P2)

RAUC bundle contains the rootfs image for the inactive slot; the app release is not inside the OS bundle because app images live on the data partition and survive slot switches ([03](03-packaging-and-hardware.md)). A combined update (OS + app) is two transactions run in order by `skyrangerctl update --with-os`:

```
1 rauc install os-<ver>.raucb       (writes the inactive slot, sets it as the next boot target with try counter)
2 skyrangerctl update (app)         stage and back up as in section 3, but do not switch yet
3 reboot into the new slot          systemd starts skyranger.service with whatever release is `current`
                                    (the new app release was already made `current` before the reboot, so the
                                    reboot is also the app switch; compatibility OS<->app is declared in the manifest)
4 gate runs after boot              pass: rauc status mark-good + commit;  fail or crash: no mark,
                                    bootloader try counter expires, next boot is the OLD slot with the OLD
                                    app release restored by the agent from update.json
```

Details to verify on first implementation (all UNVERIFIED until built and tested on hardware): RAUC's try-counter mechanics with GRUB on EFI, the exact `rauc` CLI sequence, and RAUC `system.conf` slot definitions. Sketch of the intent:

```ini
# /etc/rauc/system.conf (sketch, UNVERIFIED keys per RAUC version)
[system]
compatible=skyranger-x86-64
bootloader=grub
data-directory=/var/lib/skyranger/rauc

[keyring]
path=/etc/rauc/ca.cert.pem

[slot.rootfs.0]
device=/dev/disk/by-partlabel/rootfs-a
type=ext4
bootname=A

[slot.rootfs.1]
device=/dev/disk/by-partlabel/rootfs-b
type=ext4
bootname=B
```

The agent's unit file ties the mark-good to the gate rather than to "boot reached multi-user":

```ini
# /etc/systemd/system/skyranger-markgood.service (appliance only)
[Unit]
Description=Mark RAUC slot good after Sky Ranger update gate
After=skyranger-agent.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/skyrangerctl update mark-good   # no-op unless update.json says the gate passed
```

An OS-only security update with no app change uses the same gate (app health after reboot).

## 9. Disk budget

Needed for an app update: images of `current` + `previous` + new (about 3 sets; the `skyrangerd` image is expected to be on the order of 100 MB and the DB image several hundred MB, UNVERIFIED until built), the pre-update dump (up to the size of the DB; `backup create` can use `--history last-30d` for a breaking update's restore point, see [07](07-data-and-migrations.md)) and 20% slack. `preflight` computes this and refuses early rather than failing at step 4.

## 10. Maintenance windows and unattended updates

`site.yaml`:

```yaml
updates:
  mode: manual            # manual | windowed | fleet
  channel: stable
  window: "Sun 02:00-04:00"   # local timezone, only for windowed/fleet
  allow_breaking: false
  gate_window: 10m
  require_confirm: false
```

- `manual` (default): the box downloads nothing by itself. Integrator runs `update`, or the fleet shows "update available" ([08](08-fleet.md)).
- `windowed`: the agent checks the configured source daily and applies `stable` PATCH releases inside the window; MINOR releases need an explicit approval.
- Never two boxes of a T3 pair at once: the HA pair is updated standby first, health-gated, then switched over, then the old primary ([01](01-topologies.md)). `skyrangerctl ha update` performs that order; it is out of scope for the first milestone (T3).

## 11. Failure modes covered

| Failure | Result |
|---|---|
| Bundle corrupt or unsigned | refused at step 0, nothing changes |
| Disk full during staging | refused before step 4 by preflight |
| Power cut before step 4 | old release runs; staged files abandoned |
| Power cut during or after step 4 | boot starts `current`; agent reads `update.json`, resumes the gate or rolls back |
| New release crash-loops | gate fails (container restarts), automatic rollback |
| Migration fails | `skyrangerd` exits at start; gate fails; fast rollback if the failing migration did not commit (migrations are transactional where PostgreSQL allows, UNVERIFIED for Timescale-specific DDL), else restore rollback |
| Rollback itself fails | `failed_needs_human`, critical alert, support bundle; the DB dump and the old release remain untouched on disk |
| New OS slot does not boot | bootloader falls back (appliance) |
| New release works but is semantically wrong (wrong flow rates) | not detectable by a gate; mitigated by `--require-confirm`, the pilot ring ([08](08-fleet.md)), and the simulator release gates |

## Decisions made
1. Two layers (app release, OS image), two rollback mechanisms, never entangled except through the compatibility fields in the manifest.
2. Update is a recorded state machine (`update.json`) that survives power loss and resumes or rolls back at agent start.
3. Health gate is a conjunction of hard and soft checks with a baseline taken before the switch; connections already down before the update are excluded; the config revision must be unchanged.
4. Rollback chooses between fast (backward migrations, no data loss) and restore (breaking migrations, bounded data loss reported explicitly) by the manifest's `compat` flags.
5. Breaking migrations require an explicit operator or fleet-policy opt-in plus a verified pre-update backup.
6. The new bundle's `skyrangerctl` drives its own update; `previous` keeps the old tool.
7. Updates are operator-initiated by default; windowed automatic updates are opt-in and limited to PATCH.
8. Debian docker/kernel packages are held on P1; OS patching never changes the container runtime unattended.

## Alternatives rejected and why
- **Watchtower or tag-following auto-update**: unsigned, unpinned, unordered with migrations, no gate.
- **Blue/green containers side by side on one box**: two `skyrangerd` would double the Sparkplug host application and the PLC writes (the same reason T3 forbids active/active). The switch is stop-then-start.
- **Full-disk A/B for P1**: requires owning the OS image. P1 stays "Debian plus app rollback plus LVM snapshot for OS jumps".
- **Always restore the DB on rollback**: simple but throws away good data for the common (backward) case.
- **Never allow breaking migrations**: forces ugly permanent schema compromises. Instead, allow them with a gate, a dump and an explicit flag.
- **Downgrade by running down-migrations**: tested rarely, wrong when it matters. Backward-compatible schemas plus restore is easier to test (rollback matrix gate).
- **Fixed gate window for all sites**: a pump station and a demo bench differ; window and confirmation are per site.

## Open questions
- **Agent event emission**: `update.json` transitions, gate failures and rollbacks should appear as events (`skyranger.system.update_*`). The event catalog in messages.md has no such type and only `skyrangerd` publishes on the internal bus. Options: the agent writes to a host-side event log that `skyrangerd` ingests, or `POST /api/v1/events` for the agent (not in api.md). Needs a contract decision. Until then, agent events go to the journal and the fleet channel.
- Whether `/metrics` provides a stable "good-quality update rate" counter for the data-flow check: see [10-observability.md](10-observability.md); needs the metrics endpoint agreed.
- Whether a gate that discovers a **device-affecting** regression (writes to PLCs) needs a dry-run mode in `skyrangerd` (start with writes disabled for the gate window, then enable). Attractive for safety; needs tag-engine support.
- Timescale-specific DDL transactionality for migration rollback, and what happens to compression policies during a restore rollback: see [07](07-data-and-migrations.md).
- Minimum Docker and Compose versions for `up --wait`, `required: false` and `down --timeout` semantics (UNVERIFIED in 02).
- T3 update orchestration (`ha update`) is described, not designed; needs the HA design pass after T1 is in the field.
- Whether Debian `unattended-upgrades` should be disabled entirely on regulated customer sites in favour of customer patch management (customer policy decision).
