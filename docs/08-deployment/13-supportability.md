# 13 Supportability

Goal: when a gateway misbehaves at a plant 300 km away, one person with one command output can tell what is wrong, without a site visit in most cases, without the customer having to trust us with more than they chose, and with nothing that needs us to be online. Builds on [10-observability.md](10-observability.md) (`doctor`, metrics ring, logs), [08-fleet.md](08-fleet.md) (support tunnel), [06-updates-and-rollback.md](06-updates-and-rollback.md), [11-time-and-power.md](11-time-and-power.md), [07-data-and-migrations.md](07-data-and-migrations.md).

## 1. Support ladder

| Tier | Who | Tools | Needs from customer |
|---|---|---|---|
| 0 Self-service | customer operator | HMI System page, `skyrangerctl status`, `doctor`, runbooks (section 6) | nothing |
| 1 Bundle | integrator, remote | **support bundle** (section 2), emailed, USB, or uploaded | one command or one HMI click (`engineer`+) |
| 2 Read-only remote | integrator | `skyrangerctl diag` over the tunnel (section 4) | customer enables support for N hours |
| 3 Remote shell | integrator | SSH over the tunnel, recorded | same, explicit |
| 4 On-site | integrator | hands, USB stick, cold spare | site visit |

Everything at tier 1 and 2 works with a bundle or read-only commands: **we should be able to solve most problems without a shell**.

## 2. The support bundle

`skyrangerctl support bundle [--since 72h] [--no-config] [--include-tag-values N] [--out FILE] [--upload]`

Output: `skyranger-support-<gateway_id>-<UTC timestamp>.tar.zst`, plus a printed SHA-256. Written under `/var/lib/skyranger/support/` (auto-pruned, keep last 5), or to a USB mount. The bundle is a plain tar: **the customer can unpack and read it before sending**, and `MANIFEST.txt` at the root lists every file with a one-line description and a "may contain" tag (`site-info`, `usernames`, `ip-addresses`, `tag-names`). Nothing hidden, nothing executable.

### Contents

```
MANIFEST.txt
summary.txt                       gateway_id, release, uptime, health, config revision, update state, doctor result: the one page a human reads first
doctor.json                       every DOC-xxx check with evidence
versions.json                     release manifest (release.lock), skyrangerctl, Docker, Compose, kernel, Debian, PostgreSQL, TimescaleDB, Mosquitto, chrony
update/                           update.json, boot-health.json, last 5 gate results, pre-update backup list (names, sizes, not content)
api/                              GET /system/health, /system/info, /connections, /drivers, /alarms (counts only), /config/revisions (list)
config/                           (unless --no-config) current config YAML export (no secrets by construction), drift state
metrics/                          current /metrics snapshot + the 24 h ring buffer (10)
logs/
  journal-host.json               last <since> of host units: skyranger*, docker, chrony, sshd (auth lines trimmed), systemd, kernel (dmesg)
  journal-containers.json         last <since> of containers (journald tags skyranger/<name>)
  postgres.log                    tail of PostgreSQL logs (via journald)
host/
  df.txt lsblk.txt mount.txt smartctl.txt(if installed) free.txt top.txt sensors.txt(if any)
  ip-addr.txt ip-route.txt nft-ruleset.txt listening-ports.txt
  chrony-tracking.txt chrony-sources.txt timedatectl.txt
  docker-info.txt docker-ps.txt docker-inspect-redacted.json compose-config-rendered.yaml(secrets are file paths, not values)
  systemd-status.txt (skyranger*.service, timers), systemctl-failed.txt
  power/ clean-shutdown marker state, last boots (`journalctl --list-boots`), `last -x reboot` lines
db/
  sizes.txt chunk-compression-summary.txt jobs.txt activity.txt replication.txt(T3) migrations.txt (schema_migrations list)
  (no table contents)
spool/                            historian spool size and oldest age only
sessions/                         support session recordings (section 4), if any exist
```

### Redaction and exclusion rules (enforced in code, tested)
- **Never included**: `/etc/skyranger/secrets/**`, private keys, `.env` values whose names match `PASSWORD|TOKEN|SECRET|KEY|CREDENTIAL`, API keys and bearer tokens, password hashes, session tokens, `secrets.tar.age`, backup contents, the PostgreSQL data files, **tag values and history** (default).
- `docker inspect` output and the rendered compose file go through a redactor that replaces values of environment entries on the deny-list; `SKYRANGER_DB_URL` has any password component removed.
- Log lines are scanned for known secret patterns (bearer tokens `sr_`, `Authorization:` headers, passwords in URLs) and masked. Components must not log secrets in the first place (10); the scan is the second layer, not the first.
- `--include-tag-values N` adds the **last N samples of selected tags** (CSV) when a support case is about wrong values; opt-in per tag prefix and visible in the manifest tag "contains tag values".
- Usernames and client IPs appear in audit/auth log lines: tagged in the manifest; `--redact-users` hashes usernames consistently so correlation still works.
- The config export can reveal plant structure (tag names, connection addresses). `--no-config` omits it; without it, support can still read `summary.txt` and logs but not reproduce.

### Transport
1. **File**: the integrator or customer copies it (USB, email, customer's file share). Default and always available.
2. **Upload**: `--upload` HTTPS POST to a configured support endpoint (`site.yaml: support.upload_url`), outbound, with the fleet client cert if enrolled, else a one-time upload token the integrator issues for the case. Size capped (default 100 MB); bundle size is dominated by logs.
3. **HMI**: `POST` an admin action (needs an API addition, see Open questions); until then, console/SSH only.

### Reproduce on a bench
The config in the bundle plus the release bundle gives a **reproduction path**: `skyrangerctl support repro --bundle X.tar.zst` starts a throwaway compose project with profile `sim`, imports the config with connections swapped for simulator connections of the same names (a mapping rule: first-party drivers get `sim` equivalents; unknown drivers are marked `bad_config_error` and ignored), and replays a scenario file derived from the metrics/log timeline if present. It is a diagnostic aid, not a guarantee of fidelity (UNVERIFIED how far the replay can go).

## 3. `skyrangerctl doctor`

Checks (each has `DOC-nnn`, severity, evidence, runbook id). Initial set; grows with incidents:

| Id | Check |
|---|---|
| DOC-001 | host supported, Docker/Compose versions in range |
| DOC-010 | all compose services running and healthy |
| DOC-011 | no crash loops (restart counts) |
| DOC-020 | `/system/health` status and per-check detail |
| DOC-021 | each enabled connection `connected`; last error per failed connection |
| DOC-030 | data volume free percent; projected days to full (bytes per sample from actual data, 03) |
| DOC-031 | `/` and `/var/lib/docker` free |
| DOC-040 | chrony synced, offset, clock-state ([11](11-time-and-power.md)); RTC present and plausible |
| DOC-041 | clock skew rejections rising for all connections vs one |
| DOC-050 | last backup age, last off-box copy age, last restore test |
| DOC-060 | PostgreSQL: size, WAL, compression/retention jobs last success, long transactions |
| DOC-070 | historian spool empty; queue depth |
| DOC-080 | config drift state, config snapshot present (T2) |
| DOC-090 | update state terminal; previous release present; pending advisory for the installed version |
| DOC-100 | hardware watchdog present; SMART health; CPU temperature |
| DOC-110 | unclean shutdowns in the last 30 days |
| DOC-120 | TLS certificate validity (days left) and clock plausibility vs `notBefore` |
| DOC-130 | MQTT: broker reachable on 8883 from the LAN interface, `status` topic retained, number of connected clients |
| DOC-140 | NTP served to LAN (UDP 123 listening on the LAN IP) |
| DOC-150 | fleet enrollment and heartbeat last success; certificate days left; tunnel state |

`doctor` is read-only, safe to run on a sick box, finishes in under 30 s, and prints a human table or `--json`.

## 4. Remote diagnostics

### `skyrangerctl diag` (read-only subcommands, work locally and over the tunnel)

```
diag status                  one-page state (same as summary.txt)
diag connections [--watch]   connection states, last errors, sample rates per connection
diag device <connection> <id>  last values, ages, quality codes, rebirth/NBIRTH history for a Sparkplug device
diag mqtt tail [--topic ...] [--seconds 30]   bounded subscribe on the internal bus and Sparkplug topics (read-only user)
diag net <host:port>          TCP connect/TLS handshake test from the box (to a PLC, an NTP server, central)
diag time                     chrony tracking + comparison against a given reference host
diag db                       PostgreSQL stats, job status, slow queries (read-only role)
diag tag <path>               definition, current value, last N history rows, derived inputs, alarm state, recent discontinuity events
diag disk                     usage by directory, projected full date
```

Properties: no writes, no config changes, no restarts (those are separate `skyrangerctl` verbs that need `sudo` and are audited), bounded output and runtime, no secrets printed. A support engineer on the tunnel uses `diag` first and a shell only to fix something.

### The tunnel session ([08](08-fleet.md) section 7)
- Opened only after the customer enables it; time-boxed; per-person SSH keys.
- The session is recorded on the box under `/var/lib/skyranger/support/sessions/` and **the customer can read it**; it is included in the next bundle. The recordings never leave the box on their own.
- Elevated actions (restart, restore, config apply) are done through the same `skyrangerctl` verbs, so they land in the audit log and the update/rollback logs.
- Banner on console and SSH login while a support session is active.

### Without a tunnel
The bundle plus a phone call is the fallback; the on-site person runs `doctor` and, if asked, the diag verbs, and reads the output. All of this is deliberately text.

## 5. Support-oriented design rules (apply to all components)

1. Every error the operator can hit has a **stable code** and a runbook id (API errors have `code`; `doctor` has `DOC-nnn`).
2. Every state change worth knowing is a log line with `component`, `connection`, `request_id`; no free-text-only errors.
3. Components log **why they did something** (restart decisions in the escalation ladder include the measurements that triggered them).
4. Versions are always findable: `GET /system/info`, `skyranger_build_info`, `release.lock`, in every bundle.
5. A support engineer should never need to ask "what version are you on" or "what is your config revision".

## 6. Runbooks (index, written alongside the code; first four sketched)

Stored in the repository under `docs/runbooks/` (path proposed; not created here). Each runbook is: symptoms, quick checks (the exact `diag`/`doctor` commands), probable causes ordered by likelihood, fix, how to confirm, when to escalate.

| Id | Symptom |
|---|---|
| RB-01 | HMI unreachable / gateway silent |
| RB-02 | Device or connection down (Sparkplug, Modbus, OPC UA) |
| RB-03 | Data present but wrong (time, scaling, counter totals) |
| RB-04 | Disk filling / disk full |
| RB-05 | Clock wrong / NTP not syncing |
| RB-06 | Update failed or rolled back |
| RB-07 | PostgreSQL will not start or unhealthy |
| RB-08 | After power loss: gaps, restart loop |
| RB-09 | Backup failing / restore test failing |
| RB-10 | Certificate expired / browser or MQTT TLS errors |
| RB-11 | Fleet silent / tunnel will not open |
| RB-12 | Cold spare: restore onto a new box |

Sketches:

**RB-01 HMI unreachable.** (1) Is the box up? `ping`, console/BMC. No: power/hardware; check UPS; the unclean-shutdown marker after boot ([11](11-time-and-power.md)). (2) `https://<ip>:8443/api/v1/system/health`: no answer but box reachable -> `skyrangerctl status`; `docker compose ps`; `doctor` DOC-010/011. (3) Firewall: nft rules, LAN address (`SKYRANGER_LAN_ADDR`), certificate error vs refused. (4) If `skyranger.service` failed: `journalctl -u skyranger -n 200`; the `preflight` reason is printed (disk, clock, secrets, signature). (5) Still nothing: the escalation ladder state in `boot-health.json`; step 5 reached means a human is required.

**RB-02 Device or connection down.** `diag connections`: distinguish `bad_not_connected` (we cannot hear the network: broker, cable, credentials, driver) from `uncertain_last_known` (the link is up, the device went quiet: device power/Wi-Fi/firmware). Sparkplug: `diag device`, check NBIRTH/NDEATH history and whether the device's `boot_count` rose (reboots); `diag mqtt tail` on the device topic; broker auth failures in the Mosquitto log. Modbus/OPC UA: `diag net <plc:port>`, the driver's last error code (driver-interface error model), recent `overruns`.

**RB-03 Data wrong.** Time first: DOC-040/041. Then `diag tag <path>` (scale, units, last `counter_discontinuity` events, raw inputs). For the water meter compare the mechanical register at the site with `total_volume`; check `boot_count` epochs and discontinuity events; verify that `initial_total_l` was set correctly at commissioning (a wrong initial is the usual cause of a constant offset).

**RB-04 Disk.** `diag disk`; `doctor` DOC-030/060; compression/retention jobs ok? spool growing? backups local dir large? Docker images (`docker image prune` only of images not in `current`/`previous`, via `skyrangerctl cleanup`, never manual `docker system prune` which could delete images of `previous`). Last resort: reduce retention (config) rather than deleting files under PGDATA; never delete WAL by hand.

Others follow the same format when written.

## 7. Customer-facing diagnostics

- The HMI System page shows: version, uptime, health with the check list, connection states, config revision and drift, last backup (local and off-box), clock state, disk free, support session status, last unexpected restart, last update result. No console needed to answer "is the gateway OK".
- A one-page **site handover pack** (generated, versioned in the config repo `docs/`): gateway id, versions, network addresses, certificate fingerprint, backup destination and last test, UPS runtime, who has which credentials, the exit checklist ([12](12-exit-plan.md)).

## 8. Support policy knobs (site.yaml)

```yaml
support:
  tunnel_allowed: true          # we may request a tunnel (still needs local enable)
  always_allowed: false         # customer choice for unattended sites (08)
  upload_url: https://support.example.net/upload
  bundle:
    redact_users: true
    include_config: true
  contact: "ops@customer.example"   # shown in HMI, in alerts
```

## Decisions made
1. Support ladder from self-service to on-site; bundle and read-only diagnostics first, shell last.
2. The support bundle is a readable tar with a manifest describing every file and what sensitive class it may contain; the customer can inspect it before sending.
3. Redaction is enforced in code on a deny-list; secrets, tag values and history are never in a default bundle; tag values are opt-in per prefix.
4. `skyrangerctl doctor` with stable `DOC-nnn` ids is the first thing run and is in every bundle.
5. `skyrangerctl diag` provides read-only remote diagnostics, so most cases never need a shell; shell sessions are recorded on the box and readable by the customer.
6. Runbooks are written with the code and indexed by stable ids (`RB-nn`); errors carry codes that map to them.
7. A config bundle can be replayed against the simulator to reproduce a problem on the bench.

## Alternatives rejected and why
- **Upload all logs and metrics continuously to central**: bandwidth and privacy cost, and it makes support depend on central. The bundle is on demand.
- **A remote-shell agent running permanently**: breaks outbound-only/customer-consent principles ([08](08-fleet.md)).
- **Bundle contents chosen by the support engineer remotely**: invites over-collection; the bundle is a fixed, documented set plus opt-in flags.
- **Collecting tag history by default "to help debugging"**: customer data; opt-in and bounded.
- **A proprietary diagnostics format**: tar, JSON, text. Anyone can read it.
- **Free-text error messages as the only error surface**: cannot be matched to runbooks or searched reliably.
- **Fixing sick boxes with `docker system prune`-style cleanup**: risk of deleting the rollback target; `cleanup` knows what must stay.

## Open questions
- Contract/API: an admin endpoint to trigger `support bundle` from the HMI and to show/toggle support-session state ([08](08-fleet.md) open question). Not in api.md.
- Location of runbooks in the repo (`docs/runbooks/`, proposed) and whether the HMI links directly to the runbook for a `DOC-nnn` id (needs a public docs site; contracts README open question about a docs domain).
- How far `support repro` can go for non-simulated drivers (PLC specifics) is unknown.
- Legal/privacy review of bundle contents (usernames, IPs, tag names, device addresses) for customers with strict data policies; a `--minimal` profile (summary, doctor, versions only) may be needed.
- Go profiling endpoints in bundles (see [10](10-observability.md)).
- Bundle upload endpoint hosting and retention policy (how long we keep customer bundles; proposal 30 days after case closure).
- `diag mqtt tail` needs a read-only MQTT user and ACL (security doc owns the broker ACLs).
