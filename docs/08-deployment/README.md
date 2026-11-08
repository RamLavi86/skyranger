# 08 Deployment and Operations

Status: draft v1. Owner: deployment. Binding inputs: [../00-contracts/](../00-contracts/README.md) and [../decisions.md](../decisions.md).

This area answers one question: **how does a Sky Ranger gateway get onto a box at a plant, stay up for years, get updated without drama, and get handed back to the customer when they leave?**

## Operating principles

1. **The box runs alone.** Nothing in this area may make the gateway depend on the internet, a central server, or us (decision 1). Fleet, cloud sync and remote access are optional add-ons that can be removed with one command.
2. **Outbound only.** No inbound port is ever opened on a customer firewall for us. Everything that talks to the outside world initiates the connection from the gateway.
3. **Boring parts, few of them.** Debian, systemd, Docker Engine plus Compose, PostgreSQL plus TimescaleDB, Mosquitto, chrony, OpenSSH. Our own code on the host is one Go binary: `skyrangerctl`.
4. **Every change is reversible or explicitly marked as not.** Config changes are numbered revisions; app updates keep the previous release; database migrations declare whether they are backward compatible.
5. **Everything is exportable in plain formats.** YAML, CSV, SQL dumps, journald exports. The customer exit plan is a tested command, not a promise.
6. **Measure the gateway like a plant asset.** It has a health endpoint, metrics, a watchdog chain, and someone gets told when it goes silent.

## Documents

| # | File | Covers |
|---|---|---|
| 01 | [01-topologies.md](01-topologies.md) | Single box, box plus separate DB, small HA pair, multi-site with central |
| 02 | [02-containers.md](02-containers.md) | Runtime choice (Docker vs Podman), container layout, compose file for the vertical slice, host directory layout, systemd units |
| 03 | [03-packaging-and-hardware.md](03-packaging-and-hardware.md) | Generic Linux PC, signed appliance image (Yocto + RAUC), Raspberry Pi demo kit, hardware recommendations and sizing |
| 04 | [04-install-and-first-boot.md](04-install-and-first-boot.md) | Install sequence, `skyrangerctl init`, first boot, first login, commissioning the ESP32 meter |
| 05 | [05-config-as-code.md](05-config-as-code.md) | Git-managed YAML, repo layout, apply, drift detection, rollback |
| 06 | [06-updates-and-rollback.md](06-updates-and-rollback.md) | Release bundles, health-gated app updates, OS A/B updates, automatic rollback |
| 07 | [07-data-and-migrations.md](07-data-and-migrations.md) | Schema migration policy (expand/contract), config migration across versions, backup and restore, PostgreSQL and TimescaleDB upgrades |
| 08 | [08-fleet.md](08-fleet.md) | Inventory, config push, version skew policy, outbound-only remote access |
| 09 | [09-cloud-sync.md](09-cloud-sync.md) | What syncs, direction, conflicts, multi-tenancy by project and site |
| 10 | [10-observability.md](10-observability.md) | Health endpoint, metrics, logs, watchdog chain, alert on silence |
| 11 | [11-time-and-power.md](11-time-and-power.md) | Time sync and why it matters, power-loss behavior and recovery |
| 12 | [12-exit-plan.md](12-exit-plan.md) | Export everything and walk away |
| 13 | [13-supportability.md](13-supportability.md) | Support bundle, remote diagnostics, runbooks |
| 14 | [14-release-engineering.md](14-release-engineering.md) | Versioning, release channels, artifacts, signing, release gates, support windows |

## The whole picture on one page

```
                       plant LAN (OT)                                     uplink (IT / internet, optional)
  ESP32 meter ──MQTT 8883──┐                                                        ▲  outbound only
  PLCs ◄──Modbus/OPC UA────┤                                                        │  HTTPS 443 (fleet, sync)
  HMI browsers ──HTTPS 8443┤                                                        │  SSH-over-443 (remote support, on demand)
  NTP clients ──UDP 123────┤                                                        │
                           ▼                                                        │
  ┌──────────────── edge box (Debian 13 or Sky Ranger appliance image) ─────────────┴──────┐
  │ host: systemd, chrony (NTP server), journald, hardware watchdog, sshd (LAN only)      │
  │       skyrangerctl agent  (supervision, updates, backups, fleet, support bundle)       │
  │                                                                                        │
  │  Docker Engine, compose project "skyranger"                                            │
  │   ┌──────────────┐   ┌────────────────────────────────────────┐   ┌─────────────────┐  │
  │   │ mosquitto    │◄─►│ skyrangerd (one Go process)            │──►│ postgres         │  │
  │   │ 8883 (LAN)   │   │  + driver child processes (sparkplug,  │   │ +TimescaleDB     │  │
  │   │ 1883 (int.)  │   │    modbus, opcua, sim) over UDS gRPC   │   │ internal only    │  │
  │   └──────────────┘   │  + HMI static files, REST/WS on 8443   │   └─────────────────┘  │
  │                      └────────────────────────────────────────┘                        │
  │   [profile sim] simulator (fake Sparkplug nodes, Modbus TCP, OPC UA)                   │
  │                                                                                        │
  │  /var/lib/skyranger  (postgres, spool, backups, config-git, releases)                  │
  └────────────────────────────────────────────────────────────────────────────────────────┘
```

Note on the HMI: [api.md](../00-contracts/api.md) states that `skyrangerd` serves the HMI static files on the same HTTPS port as the API. There is therefore **no separate HMI container** in production. The HMI build output is baked into the `skyrangerd` image. A Vite dev server container exists only under the compose `dev` profile.

## Summary of decisions in this area

1. Docker Engine + Compose v2 is the only supported runtime for v1, on every packaging option. OCI images and a plain compose file are the deployable unit; no Docker-only features beyond Compose are used, so Podman stays possible later.
2. Bind mounts under `/var/lib/skyranger` and `/etc/skyranger`, never anonymous or named volumes, so backup, support and exit are `tar` and `pg_dump` of known paths.
3. One host binary, `skyrangerctl`, provides install, the supervising agent, updates, backups, config-as-code, fleet enrollment, support bundles and export.
4. Two update layers: the **app release** (signed manifest of image digests plus migrations, health-gated, auto-rollback) on all boxes, and the **OS image** (RAUC A/B, bootloader fallback) on the appliance only.
5. Database migrations follow expand/contract and declare `compat: backward | breaking`. Rollback to the previous release without a data restore is guaranteed only across `backward` migrations.
6. PostgreSQL `config_revisions` stays the authoritative config store ([tag-model.md](../00-contracts/tag-model.md) section 9). Git is the engineering workflow and an off-box mirror, and every apply records the git commit in the revision comment.
7. Fleet and remote access are outbound only: HTTPS polling for desired state, and on-demand, customer-enabled, time-boxed OpenSSH reverse tunnels for remote support.
8. Generic Debian on an industrial x86-64 PC is the first production target. The signed appliance image (Yocto + RAUC) comes after two or three real deployments. Raspberry Pi 5 with an SSD is for demos only.

## Decisions made
See the summary above. Each file records its own decisions in detail.

## Alternatives rejected and why
- **Kubernetes / k3s at the edge**: an extra control plane, etcd or SQLite state, and a networking layer, all of which can fail. A single box with a dozen processes does not need a scheduler. Revisit only if a customer standardizes on it.
- **Bare-metal install without containers**: decision 2 says containerized, and containers give us exact, digest-pinned dependency sets (PostgreSQL + TimescaleDB versions in particular) and clean rollback.
- **A separate HMI container (nginx)**: contradicts api.md (one port, served by `skyrangerd`) and adds a hop and a TLS config for no benefit.

## Open questions
- Several items here need small, additive contract changes (a metrics endpoint, extra health check keys, a first-boot setup endpoint, a fleet topic or API family). They are collected in the files that need them and summarized in [10-observability.md](10-observability.md), [04-install-and-first-boot.md](04-install-and-first-boot.md) and [08-fleet.md](08-fleet.md). None is used until the contract owner agrees.
- The security doc ([../07-security/](../07-security/)) owns TLS, MQTT ACLs, secrets storage and sandboxing. This area assumes a file-backed secret store under `/etc/skyranger/secrets` until that doc says otherwise.
