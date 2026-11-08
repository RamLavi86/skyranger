# 11 Time Sync and Power-Loss Behavior

Binding: [tag-model.md](../00-contracts/tag-model.md) section 5 (three timestamps, `auto` selection with `max_clock_skew` default `10s` and the 2020-01-01 floor; "gateway clock is assumed NTP-disciplined, run chrony and serve NTP to the plant LAN"), section 7 (`counter_total`, `derived_state`), [messages.md](../00-contracts/messages.md) (device heartbeat, rollover/reset representation), [driver-interface.md](../00-contracts/driver-interface.md) section 6 (historian spool, queues), [03-packaging-and-hardware.md](03-packaging-and-hardware.md) (RTC, PLP SSD, UPS signal), [02-containers.md](02-containers.md) (`time-sync.target`, PostgreSQL settings).

Plants lose power and lose internet. Both are normal. This document defines what the box does about each, and what we promise and do not promise afterwards.

# Part A: Time

## 1. Why time matters here (it is not decoration)

| Where time is used | What goes wrong when the clock is wrong |
|---|---|
| **History key `(tag_id, ts)`** and ordering | samples land in the wrong place; a clock that steps backwards produces out-of-order or overwriting rows (a same-`ts` sample conflicts with the key); trends show gaps and spikes |
| **`ts = ts_gateway` fallback** | tag-model.md `auto` falls back to the gateway clock whenever a device's `ts_source` is implausible. A wrong gateway clock then poisons *all* those samples |
| **Skew rule** `|ts_source - ts_gateway| <= max_clock_skew` (10 s) | if the gateway is wrong by more than 10 s, **every correct device** is rejected as skewed and counters `clock_skew_rejections` explode; if the gateway is right and the ESP32 is wrong, the ESP32 is (correctly) ignored |
| **Alarm times, acks, shelve expiry** (`24h` max) | operators see wrong event times; shelved alarms expire at the wrong moment |
| **Daily totals** (`agg=delta`, `interval=1d`, `tz`) | water consumption per day assigned to the wrong day: billing and compliance reports |
| **TLS certificate validity** (`notBefore`/`notAfter`) | a box booted at 1970 sees its own certificate as "not yet valid"; browsers and MQTT clients reject it. This is the classic embedded failure |
| **Session tokens** (`12h`), API key expiry, fleet certificate lifetimes | immediate expiry or never-expiry |
| **TimescaleDB chunks, compression and retention jobs** | chunk boundaries and "older than 400d" decisions use database time; a clock far in the future can drop data through retention early (UNVERIFIED mechanism per TimescaleDB version; treat as a risk) |
| **Logs and audit** | forensic value lost |
| **Sparkplug/MQTT** | timestamps on payloads from devices; broker `$SYS` uptimes (cosmetic) |
| **ESP32 meters** | no battery RTC on typical ESP32 boards: they get time via SNTP from the gateway at boot; the box is their time source |

Conclusion: the gateway must have a **trustworthy clock before it records**, and it is also the plant's time server for devices that have no other source.

## 2. Time architecture

- **chrony** on the host (not in a container, not hypervisor guest tools), as NTP client **and** LAN NTP server (chrony is boring and handles intermittent connectivity and RTC trimming well). Hosts keep UTC; `Gateway.spec.timezone` is display/calendar only (tag-model.md).
- **RTC**: hardware RTC (coin cell) is the boot-time reference. chrony's `rtcsync` keeps the RTC updated from the system clock (kernel 11-minute mode); `hwclock` must be set UTC.
- **No internet is normal.** If the site gives an upstream (customer NTP server, GPS-disciplined unit, domain controller), chrony uses it. If not, the box is the reference using its RTC and `local stratum 10` (orphan/fallback mode): all devices agree with each other and with the gateway even if the true time drifts slowly. The installer asks ([04](04-install-and-first-boot.md) section 3, "Upstream NTP servers; blank = this box is the reference, see warning") and prints a warning that without an upstream or GPS the absolute time depends on the RTC (typically seconds to a minute per month of drift for a cheap RTC, UNVERIFIED for specific hardware) and must be set at install and checked at service visits.
- **GPS/PPS or PTP**: optional hardware (a GPS module with PPS feeding chrony); later, only if a customer needs sub-millisecond or an air-gapped accurate reference. Not v1. Time quality needed for a water meter is seconds, not microseconds.

`/etc/chrony/chrony.conf` produced by `skyrangerctl install/init` (sketch; options are chrony directives, check against the installed chrony version, UNVERIFIED per version):

```
# upstream (omitted when the box is its own reference)
server 10.0.0.1 iburst
# fallbacks the customer permits; pool entries are only emitted if the site has internet by choice
# pool 2.debian.pool.ntp.org iburst

driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3            # step (not slew) if off by more than 1 s, but only during the first 3 updates after start
rtcsync
hwclockfile /etc/adjtime  # RTC is UTC (hwclock --utc)

# serve the plant LAN (also devices: ESP32 SNTP)
allow 10.0.0.0/24
bindaddress 10.0.0.10     # LAN interface only (see 03/04: 123/udp on the OT LAN)

# when no upstream is reachable (or none exists) keep serving, so devices stay consistent
local stratum 10 orphan   # ("orphan" requires chrony 3.x+, UNVERIFIED exact semantics)
maxupdateskew 100.0
logdir /var/log/chrony
```

`makestep 1.0 3` is deliberate: large steps are only allowed immediately after boot, **before** `skyrangerd` starts recording (see next section). After that, corrections are slewed, so history never sees a time jump from NTP correction alone.

## 3. Boot ordering and the "is the clock sane yet" rule

Boot with a wrong clock is the dangerous moment: the RTC may be flat (battery dead, 1970) or a VM may have a stale snapshot clock.

```
 chronyd starts  ──►  time-sync.target reached when systemd-time-wait-sync (or chrony-wait) reports sync
                                │ (with a timeout)
 skyranger.service  After=time-sync.target   (02 unit)
   ExecStartPre=skyrangerctl preflight       clock plausibility check (04 preflight)
```

`skyrangerctl preflight` decides the **clock state** and writes `/run/skyranger/clock-state` (read by `skyrangerd` and the agent):

| State | Condition | Start behavior |
|---|---|---|
| `synced` | chrony `Leap status: Normal` and offset small | start normally |
| `rtc_only` | no sync source available, but clock is after the release's `built_at` and not more than 1 day ahead of the newest file timestamp on the data volume (heuristic) | start, health `degraded` with `clock: degraded`, log warning. This is the **normal state for an isolated site** with the box as the reference; it must not block operation |
| `suspect` | clock before the build date of the installed release (1970, a dead RTC), or earlier than the newest persisted timestamp (latest history row, latest `config_revisions.applied_at`, last backup) by more than 5 minutes | **wait** up to `skyranger-timewait` timeout (default 120 s) for NTP; if still suspect, start the **stack but hold the historian**: see below |

For `suspect`: the box has no idea what time it is. Options and the choice:
- *Block everything until time is fixed* means an operator must intervene at a remote pumphouse while no data is collected, but also no data is *corrupted*. Rejected as the sole behavior: availability of monitoring and alarming matters.
- *Start and record with the wrong timestamps*: corrupts history and the `(tag_id, ts)` key invariants. Rejected.
- **Chosen**: start the gateway, keep the live tag engine, alarms and HMI running (operators can still see values and alarms, with `ts` from the suspect clock flagged in the UI by the health banner), and **spool samples to the historian's on-disk spool with an ingest marker instead of writing them to the hypertable** until the clock is `synced` or an engineer sets it (`skyrangerctl time set "2026-11-08T10:00:00Z"`). When the clock becomes sane, the spool is replayed **with timestamps corrected by the measured offset** relative to the moment of the correction (a documented, approximate fix for the interval; it is flagged in an event). This needs historian cooperation (Open questions). Until that exists, the minimum viable behavior is: `suspect` blocks the historian writer and raises a critical alert, losing history for the interval rather than corrupting it, and counters recover through cumulative samples afterwards.

Also at runtime: the agent watches chrony (`chronyc tracking`) every 30 s. If the system clock **steps** by more than 1 s while `skyrangerd` runs (an operator ran `date -s`, a VM resumed from a snapshot, or a GPS jump), the agent emits an alert and `skyrangerd` is told to treat the step as a discontinuity: it logs a `warn`, notes the step window in an event, and the historian refuses samples that would carry a `ts` older than the last written sample for that tag **only** if they are on the wrong side of the step (the `(tag_id, ts)` key makes an exact duplicate a conflict; the contract accepts older samples into history, so this is a data-quality annotation, not rejection). Detailed rule is the historian's (Open questions).

## 4. Time for devices (ESP32 and others)

- The gateway serves NTP on the OT LAN (123/udp); the ESP32 firmware's SNTP server is `mqtt_host` (04 section 5 item 3 sets "NTP server 10.0.0.10").
- Devices that boot without time publish `ts_source` near 1970; the `auto` rule rejects it and counts `clock_skew_rejections` ([tag-model.md](../00-contracts/tag-model.md)). This is working as designed: the sample still arrives with `ts_gateway`. A rising `skyranger_clock_skew_rejections_total` for one connection means "that device's clock is wrong", for all connections means "the gateway's clock is wrong" ([10](10-observability.md)). The first thing a runbook checks.
- Pulse-meter volume does not depend on device time at all: `pulse_count` is cumulative and the gateway computes the rate on its own timeline. Time errors move *when* liters are attributed, not *how many*.

## 5. Time in VMs, containers and the appliance

- VMs (P1 on a hypervisor): disable hypervisor time-sync *stepping* of the guest (host-guest tools) and use chrony; after a VM snapshot resume or live migration the clock may jump: covered by the step detection above.
- Containers share the host kernel clock; do not run an NTP daemon in a container. `skyrangerd` never sets the system clock.
- Appliance (P2): read-only root; `/var/lib/chrony` and `/etc/adjtime` live on the data partition; the factory image has a baked-in minimum plausible date (the image build date) so a dead RTC cannot go earlier than the build.
- Time zone: UTC on the OS. The site timezone is `Gateway.spec.timezone` (IANA), changed through config, not `timedatectl`.

## 6. Leap seconds, DST
- DST is a display concept only (UTC storage; calendar aggregation via `tz` parameter, api.md). A daily total across a DST change is 23 or 25 hours long: documented in report notes, handled by the `tz`-aware bucket alignment.
- Leap seconds: chrony default step/slew handling; UNVERIFIED if we enable `leapsectz`. A leap second causes at most a one-second duplicate/insertion; below the 10 s skew limit. Not a design driver.

# Part B: Power loss

## 7. What happens when power is cut (and what we promise)

Failure modes in order of how common they are at a pumphouse: brief brownout or blip (seconds), a cut of minutes to hours, a cut at the worst moment (during a checkpoint, a backup, an update), repeated cycling during a storm.

**Promises** (to be proven by the power-cut soak gate, [14](14-release-engineering.md)):

1. **The database survives**: PostgreSQL recovers from any number of unclean shutdowns without manual repair, provided the storage honors fsync (see hardware) and the settings in 02 are unchanged. We never run PostgreSQL with `fsync=off`, `synchronous_commit=off` for config and state tables, or `full_page_writes=off`.
2. **Configuration is never lost or half-applied**: config is applied as one atomic transaction (tag-model.md); an interrupted apply is either revision N+1 or still N.
3. **History loses at most a bounded recent window**: the samples that were in memory (tag engine to historian queue, historian batch not yet committed) at the instant of the cut. The bound is the historian's batch/flush interval (target at most 1 s, 5 s worst case; the historian owns the number, Open questions). Samples are not "half written"; a batch is a transaction.
4. **Cumulative counters lose no volume** across the gap: pulse counts are cumulative on the device (decision 6); the first sample after power returns carries the total, so volume accounting is correct even though the gap's trend shape is unknown (`counter_total`, `derived_state`). This holds as long as the **device itself** did not lose power (the device has its own supply; a device that reboots uses `boot_count` epoch, messages.md).
5. **Gaps are visible, never filled with lies**: `history.max_interval` (tag-model.md default `15m`) forces a row even for flat values so a missing row means "not recorded", not "unchanged"; the restart is recorded as an event (below).
6. **The box comes back by itself**: no keypress, no password prompt (disk encryption at rest, if ever used, must use a TPM or network unlock: security doc).

**Not promised**: samples after the last committed batch; alarm *transitions* that were in flight (alarm states are recomputed from live values at start, so a still-true condition re-alarms; an acknowledged alarm's ack is durable once committed); device values during the outage; the derived `rate` value exactly (it restarts with `uncertain_initial_value`, tag-model.md quality 67, until the window refills).

## 8. Storage layer rules

| Layer | Rule |
|---|---|
| SSD | industrial, with power-loss protection (PLP) capacitors ([03](03-packaging-and-hardware.md)); volatile write cache without PLP is disabled or the drive is not supported (`hdparm -W0`, UNVERIFIED per drive; a consumer SSD that lies about flush is the common real-world failure) |
| Filesystem | ext4 `data=ordered` (default), `errors=remount-ro`, `noatime`, **never** `nobarrier`/`barrier=0`; journal on the data volume; periodic `fsck` is not forced at boot (`tune2fs -c 0`) but the journal replay always runs |
| PostgreSQL | `fsync=on`, `full_page_writes=on`, `synchronous_commit=on` (02); `wal_sync_method` default; checkpoint tuned (`max_wal_size=1GB`, `checkpoint_timeout=10min` default, so crash recovery replays at most about that much WAL; recovery time is seconds to a couple of minutes, UNVERIFIED until measured) |
| Docker | `/var/lib/docker` on the same filesystem class; images are immutable so corruption is detectable (`docker load` digest check in repair, [13](13-supportability.md)) |
| journald | persistent journal; a torn journal file is rotated away on boot (journald self-heals), no impact on data |
| SD cards, USB-attached DB, consumer SSDs | unsupported for production (03) |
| Backups | written to a temp name and renamed after fsync and checksum, so a power cut never leaves a "valid-looking" partial backup ([07](07-data-and-migrations.md)); `backup.json` is written last |
| `update.json` | atomic rename with fsync ([06](06-updates-and-rollback.md) section 4) |

`synchronous_commit=on` has a cost: one fsync per commit. The historian uses batched `COPY`/multi-row inserts so the fsync rate is per batch, not per sample; the sizing in 03 assumes this. If a site's disk is slow, the answer is a better disk, not turning durability off.

## 9. Graceful shutdown on power-fail signal

Hardware recommendation ([03](03-packaging-and-hardware.md)): UPS or supercap module that signals the box. Software: **NUT (Network UPS Tools)** is the boring, standard answer for USB/serial/SNMP UPSes (UNVERIFIED for any specific UPS model); for DIN supercap modules with a digital input, a small `power-fail` GPIO/serial watcher in the agent.

```
 UPS on battery ──► upsmon (NUT) or agent GPIO watcher
                      │  after `grace` seconds on battery (default 30 s; ignore blips)
                      ▼
          skyrangerctl power-fail          (called by upsmon SHUTDOWNCMD or the agent)
            1. emit alert "gateway on battery / shutting down"        (outbound alert, 10)
            2. systemctl stop skyranger     (graceful: skyrangerd checkpoints derived_state,
                                             drivers get 5 s, postgres stops with 60 s grace, 02)
            3. sync; systemctl poweroff
```

- **Runtime budget**: shutdown must complete within the UPS runtime minus the grace. Required: graceful stop of the stack in under 90 s (02 `stop_grace_period`s add up to <=90 s for skyrangerd + postgres in the worst case; the unit has `TimeoutStopSec=120`). A UPS that cannot hold the box for `grace + 120 s` is under-sized; `preflight` and the handover checklist record the UPS runtime.
- Do not shut down for a blip: short outages should be ridden through. A site may prefer to keep running until battery low (`LOWBATT`) rather than after a fixed grace; both are NUT configurations.
- **Restart policy after power returns**: the BIOS/firmware option "power on after AC loss" must be **on** (the box must come back by itself; a hardware qualification item). If the box shut down because of low UPS, it boots when mains returns; if the UPS is still charging and power drops again, the cycle repeats safely because shutdown was graceful.
- For boxes without a UPS (many pumphouses): the design is **crash-safe**, not graceful. Every promise in section 7 holds without a UPS; the UPS buys a clean checkpoint and a short recovery.

## 10. Recovery after an unclean shutdown

Sequence after power returns:

```
 firmware POST ──► bootloader ──► kernel (ext4 journal replay) ──► systemd
   chrony  (RTC time, then NTP)  ──► time-sync.target / clock-state (section 3)
   docker  ──► skyranger.service: preflight, compose up --wait
       postgres: crash recovery from WAL (log: "database system was not properly shut down; automatic recovery in progress")
       mosquitto: loads persistence
       skyrangerd: migrations check, load config, drivers connect, Sparkplug STATE online, rebirth requests
   agent: detects UNCLEAN SHUTDOWN, records it, evaluates health, gate if update was in progress
```

- **Unclean shutdown marker**: the agent (and `skyranger.service` ExecStop) writes `/var/lib/skyranger/state/clean-shutdown` (containing the time) only at the end of a graceful stop, and **removes** it at the end of a successful start. At boot, if the file is missing, the previous run ended uncleanly. The agent then: emits an alert (warning: "unexpected restart"), records `boot_reason: unclean` in `boot-health.json`, adds the interval (last health-ok time from the metrics ring, to now) as the **known data gap** to the HMI System page and to the support bundle, and runs `skyrangerctl doctor` once the stack is up.
- **Mid-update power loss**: `update.json` is not terminal, so the agent resumes the gate or rolls back ([06](06-updates-and-rollback.md) section 4).
- **Mid-backup power loss**: the temporary directory is detected and removed; the previous good backups are untouched; the next scheduled run (Persistent=true on the timer, 02) runs promptly.
- **Mid-restore or mid-migration**: migrations run in a transaction where possible and are resumable ([07](07-data-and-migrations.md)); a restore that did not finish leaves the old PGDATA moved aside, never deleted.
- **Corruption**: if PostgreSQL cannot start (`pg_controldata`/WAL damage), the agent does **not** try repairs. It marks the box `failed_needs_human`, collects a bundle, and the runbook says: restore from the latest verified backup onto a cold spare or the same box ([07](07-data-and-migrations.md), target recovery under 30 minutes for a 10 GB DB).
- **Devices**: ESP32 devices reconnect; Sparkplug `NBIRTH` rebirth requests are rate-limited by the driver (messages.md, once per node per 30 s) so a site-wide power return does not storm the broker. With many sites restarting at once after a regional outage, the fleet/sync uploads add jitter to their start (`sync` and `heartbeat` use random start delays up to 5 min) so central is not stampeded.
- **Brownout loops**: if the box boots, loses power within 2 minutes, boots again repeatedly, the agent counts boots per hour; above 4 it stays at preflight (starts only broker and database, not drivers) and raises an alert, to avoid damaging the database with perpetual crash recoveries. Detail UNVERIFIED as to whether this is useful versus risky; see Open questions.

## 11. Testing power-loss behavior (so the promises are measured)

- **Hardware qualification (before a box model is approved)**: relay-controlled power cycling rig; 500 cycles at random points during simulator load (including at checkpoints, during a nightly backup, during an update to a test release). Pass criteria: PostgreSQL starts every time, `pg_dump` and a row-count/checksum comparison against an independent reference log shows no corruption, only the bounded recent-window loss, config unchanged, the gate/rollback logic resumes correctly.
- **CI approximation**: a VM hard-reset loop (QEMU `quit` without shutdown, or `echo b > /proc/sysrq-trigger`) during the simulator soak: validates software promises but not SSD behavior.
- **Reported measurements**: crash recovery time, data-loss window, time to first sample after power return: into the release notes and the hardware list.

## Decisions made
1. chrony on the host is NTP client and LAN server; without upstream the box is the time reference via its RTC (`local stratum 10`), documented with its drift limits.
2. A boot-time clock-sanity check produces `synced`, `rtc_only` or `suspect`; `suspect` never records timestamps that corrupt history (hold the historian, alert).
3. `makestep` is only allowed right after boot; later corrections slew; clock steps at runtime are detected and flagged.
4. The gateway is crash-safe by design (no durability knobs turned off) and graceful when a UPS signals; promises and non-promises are written down (section 7).
5. Storage requirements: PLP SSD, ext4 with barriers, `errors=remount-ro`; consumer SSDs and SD cards unsupported for production.
6. UPS integration via NUT or a GPIO watcher calling `skyrangerctl power-fail` with a grace period, within a stated shutdown time budget.
7. Unclean shutdown is detected by a marker file, recorded as an event/alert and a known data gap; the agent never attempts database repair.
8. Power-cut soak (hardware rig and VM loop) is a release/qualification gate.

## Alternatives rejected and why
- **Trusting hypervisor or guest tools for time**: VM snapshots and migrations make them step the clock under a running recorder.
- **Starting only after NTP sync, always**: correct for a connected site, a deadlock for an isolated one. The `rtc_only` and `suspect` states separate "no sync source by design" from "I do not know what time it is".
- **Using device timestamps as primary**: ESP32 clocks start wrong on every boot ([tag-model.md](../00-contracts/tag-model.md) `auto` rule).
- **Disabling `synchronous_commit` to gain throughput**: trades silent data loss for speed; batching achieves the speed.
- **Relying on a UPS and calling the box "protected"**: UPSes die and batteries age; crash-safety is the base layer.
- **Auto-repair of PostgreSQL (`pg_resetwal` and similar) by the agent**: can silently lose or corrupt data. A human or a restore decides.
- **Journaling filesystems with write barriers disabled** or **tmpfs for the database**: speed hacks that cost the guarantee.
- **GPS/PTP in v1**: precision beyond what a water meter needs.

## Open questions
- **Historian owner**: (a) flush interval and the data-loss bound in section 7; (b) the `suspect`-clock spool behavior (hold, replay with offset correction, flag) and the runtime-step rule in section 3; (c) whether history rows carry a `clock_suspect` flag ([09](09-cloud-sync.md) conflict table uses the same idea).
- **Tag-engine owner**: how often `derived_state` is checkpointed (every update vs interval) determines how much `counter_total` re-derives after a crash; cumulative counts make this safe, but the interval should be stated.
- Whether the engine needs a `time_sane` input from the host, or reads `/run/skyranger/clock-state` (a file in a read-only mount) itself; the container mounts `/run/skyranger` as tmpfs today (02), so the file must be bind-mounted from the host.
- chrony directives `local stratum 10 orphan`, `makestep`, `rtcsync` and `bindaddress` semantics need checking against the Debian chrony version at release (UNVERIFIED).
- RTC drift and battery life of the chosen hardware: measure; a dead RTC battery is a field failure to expect, and `suspect` handling must be tested with the RTC removed.
- Brownout-loop guard (section 10): useful or risky? Needs field experience or an experiment with the power-cut rig.
- Whether encrypted data partitions will ever be required by a customer (needs unattended unlock, security doc).
- TimescaleDB retention behavior when the clock jumps forward: UNVERIFIED; test in the `suspect` clock scenarios.
- NUT model support for the UPS hardware we recommend: evaluate with the first hardware qualification.
