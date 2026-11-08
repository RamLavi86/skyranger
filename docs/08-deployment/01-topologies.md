# 01 Reference Deployment Topologies

Four topologies, in the order we will support them. Every topology is built from the same images and the same compose file; only the profiles, the environment file and the host count change.

| ID | Name | Supported from | Typical customer |
|---|---|---|---|
| T1 | Single box | milestone 1 | one pumphouse, small water utility, demo |
| T2 | Box plus separate DB | after first real project | site with an existing PostgreSQL server or IT-run VM host, long retention |
| T3 | Small HA pair | after T1 has run 6+ months in the field | site where a gateway outage means a truck roll at night |
| T4 | Multi-site with central | when we have 3+ sites for one customer | utility with several pumping stations, integrator fleet |

## T1: Single box (default)

```
  ┌──────────────── edge box ────────────────┐
  │ mosquitto  skyrangerd(+drivers)  postgres │
  │ chrony     skyrangerctl agent             │
  └───────────────────────────────────────────┘
        ▲ OT LAN: MQTT 8883, HTTPS 8443, NTP 123
```

- All containers on one host, compose profiles `core` (default).
- PostgreSQL listens only on the compose network. No port is published.
- Backups go to `/var/lib/skyranger/backups` and, if configured, to a second disk, a USB stick, an SMB/NFS share, or an S3-compatible bucket (outbound).
- Failure domain: the box. Mitigations: the hardware watchdog, the systemd restart chain, the power-loss design ([11-time-and-power.md](11-time-and-power.md)), and a documented **cold spare procedure**: a pre-imaged spare box restored from the latest backup in under 30 minutes (target, to be measured on the first site).

Why this is the default: it is the smallest thing that is fully correct, and edge-first means the box must already be complete on its own.

## T2: Box plus separate database

```
  ┌──── edge box ──────────────┐        ┌──── DB host (customer IT or our 2nd box) ─┐
  │ mosquitto  skyrangerd       │──TLS──►│ PostgreSQL 17 + TimescaleDB               │
  │ chrony     skyrangerctl     │ 5432   │ owned backups, larger disk                 │
  └─────────────────────────────┘        └────────────────────────────────────────────┘
```

- Compose: `postgres` service disabled (do not enable profile `db`), `SKYRANGER_DB_URL` points to the external server. TLS required (`sslmode=verify-full`).
- The historian already spills to a local on-disk WAL if PostgreSQL is unavailable ([driver-interface.md](../00-contracts/driver-interface.md) section 6, historian doc owns the format). In T2 this spool is what keeps history gap-free during a DB host reboot. Size it for at least 24 h of the site's sample rate (see sizing in [03-packaging-and-hardware.md](03-packaging-and-hardware.md)).
- **Config is also in that database.** If the DB host is down when `skyrangerd` starts, the gateway cannot load its config. Mitigation: `skyrangerd` keeps a **last-applied config snapshot** at `/var/lib/skyranger/state/config-snapshot.yaml` (written after every successful apply) and starts from it in read-only-config mode, with `/system/health` reporting `degraded`. This is a requirement on the tag engine / config service; see Open questions.
- Supported DB versions: exactly those listed in the release manifest ([14-release-engineering.md](14-release-engineering.md)). A customer DB with a TimescaleDB version we have not tested is a support exclusion.
- Who owns DB backups is written into the site handover document. Our backup command still runs `pg_dump` remotely by default.

## T3: Small HA pair (active / warm standby)

Industrial SCADA HA is usually primary/standby with automatic or manual switchover. We do the boring version.

```
                    virtual IP 10.0.0.10 (keepalived, VRRP)
                         │
   ┌──── box A (primary) ┴───────┐   streaming replication   ┌──── box B (standby) ──────────┐
   │ mosquitto  skyrangerd ACTIVE │──────────────────────────►│ postgres (hot standby, RO)    │
   │ postgres (primary)           │                            │ skyrangerd STOPPED            │
   │ keepalived MASTER            │◄──── keepalived VRRP ─────►│ mosquitto running (idle)      │
   └──────────────────────────────┘                            │ keepalived BACKUP             │
                                                               └───────────────────────────────┘
```

Rules:
- **Exactly one `skyrangerd` is active.** Two active gateways would both act as Sparkplug host application with the same `host_id`, both write history, and could both send writes to PLCs. The standby's `skyrangerd` container is stopped, not idle.
- Devices, HMIs and NTP clients use the virtual IP only. The ESP32 `mqtt_host` is the VIP.
- PostgreSQL streaming replication (asynchronous) from A to B. Async means a failover can lose the last few seconds of history; that is acceptable and documented. Synchronous replication would stall the primary when B is down, which is worse for an edge gateway.
- Failover sequence (keepalived `notify_master` script on B, calls `skyrangerctl ha promote`):
  1. Fence: confirm A does not answer on the VIP and on its direct address (ping plus `GET /api/v1/system/health`). If A answers on its direct address but has lost the VIP, do not promote (split-brain guard), alert instead.
  2. `pg_ctl promote` on B's PostgreSQL (via `docker compose exec`).
  3. Start `skyrangerd` on B. It loads config from the now-primary DB, publishes Sparkplug `STATE` online, and requests rebirths. Devices reconnect to the VIP, now on B.
  4. Emit an event and an alert: "failover to B". Failback is **manual** (`skyrangerctl ha rebuild-standby` on A, then optional planned switchover), because automatic failback causes flapping.
- Retained MQTT state is not replicated. Devices re-publish NBIRTH on reconnect, and `skyrangerd` republishes retained internal-bus values at startup, so broker state rebuilds itself within one heartbeat interval.
- `counter_total` state lives in `derived_state` in PostgreSQL ([tag-model.md](../00-contracts/tag-model.md) section 7), so it is replicated. Because pulse counts are cumulative, counts that arrive during the failover gap are recovered by the next sample. This is a big reason the water meter design suits HA well.
- Expected switchover time: under 60 s (target, UNVERIFIED until measured; dominated by VRRP advert timeout, container start and device reconnect backoff).

What T3 is not: it is not load sharing, not zero-loss, and not a fix for a bad config (both boxes run the same config). It covers box hardware failure, disk failure and OS hang.

Alternative considered for T3: Patroni + etcd. Rejected for a two-node pair: etcd needs three members for quorum, and a third box (or a witness) at a pumphouse is rarely available. keepalived + a fencing check + manual failback is understandable by the person on call at 2 a.m.

## T4: Multi-site with central

```
  site A edge box (T1/T2/T3) ──┐
  site B edge box ─────────────┼── outbound HTTPS 443 / MQTT-over-TLS 8883 ──► central
  site C edge box ─────────────┘                                              ┌───────────────────────────┐
                                                                              │ fleet service (inventory, │
                                                                              │   desired state, releases)│
                                                                              │ sync broker (Mosquitto)    │
                                                                              │ central skyranger-cloud    │
                                                                              │   (PostgreSQL+Timescale,   │
                                                                              │    read-only HMI, reports) │
                                                                              │ support tunnel endpoint    │
                                                                              └───────────────────────────┘
```

- Each site is a complete T1, T2 or T3 deployment. It keeps running, alarming and storing history if central disappears for a month.
- Central runs the **same images** with a different role (`SKYRANGER_ROLE=central`), plus the fleet service and a tunnel endpoint. Central can be hosted by us, by the customer on their own VM, or in any cloud. It is a Linux VM with Docker, nothing cloud-proprietary, so the customer can take it with them.
- Data flows and conflict rules: [09-cloud-sync.md](09-cloud-sync.md). Fleet: [08-fleet.md](08-fleet.md).
- Control (writes to PLCs) from central is **off by default** and, when enabled, still goes through the site gateway's REST API with site-level auth and audit. Central never talks to field devices directly.

## Choosing a topology

| Question | If yes |
|---|---|
| Does a 4-hour gateway outage cost more than a second box? | T3 |
| Does customer IT insist on running the database, or want > 2 years of 1 s history? | T2 |
| More than one site for the same customer, or we support > 5 sites? | T4 (fleet at least, sync optional) |
| Otherwise | T1 |

## Decisions made
1. T1 single box is the default and the only topology for milestone 1.
2. All topologies use the same images and compose file; differences are profiles and environment.
3. HA is active/warm-standby with keepalived VIP, async PostgreSQL streaming replication, a fencing check before promotion, and manual failback. Exactly one `skyrangerd` runs at a time.
4. Central is the same software in a different role, hostable anywhere, never required by a site.
5. The configuration snapshot file on the edge box lets the gateway start when an external DB (T2) is unreachable.

## Alternatives rejected and why
- **Active/active gateways**: two Sparkplug host applications with the same `host_id`, duplicated history and duplicated PLC writes. The coordination needed is not worth it at this scale.
- **Patroni / etcd / Pacemaker for the pair**: correct but heavy; quorum needs a third node; Pacemaker is hard to operate for integrators.
- **Synchronous replication**: primary stalls when the standby is down, which turns a redundancy feature into an outage cause.
- **Central-first ("thin edge")**: violates decision 1.
- **Shared storage (DRBD/SAN) HA**: more moving parts and a single storage failure domain.

## Open questions
- The config snapshot fallback (T2) needs agreement from the tag-engine / config owners ([../01-tag-engine/](../01-tag-engine/)). Is starting from a snapshot in read-only-config mode acceptable, or should `skyrangerd` refuse to start without a database?
- Sparkplug 3.0 host application failover rules (does the new host need a different `host_id`, or is reusing the same `host_id` after the old STATE Will fires correct?) are UNVERIFIED and affect T3.
- Should T3 also support a Mosquitto bridge so that buffered QoS 1 device messages survive failover, or is device reconnect enough? (Pulse counts are cumulative, so for the water meter it is enough.)
- Whether keepalived is acceptable on customer OT networks that block VRRP multicast (unicast VRRP is supported by keepalived; confirm per site).
