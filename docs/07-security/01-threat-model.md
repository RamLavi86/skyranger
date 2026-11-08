# 01 Threat Model

Method: list assets, actors, and trust boundaries, then walk each boundary with STRIDE (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege). Each threat gets a mitigation that points to the file where it is designed. Re-run this model whenever a new boundary is added (cloud sync, scripting, northbound OPC UA server).

## 1. Deployment scenarios

### A. Small plant (milestone target)
One edge box (industrial PC or small x86/ARM64 server) running containers: `skyrangerd` (with driver child processes), Mosquitto, PostgreSQL + TimescaleDB, the HMI static files. Field devices: ESP32 meters on Wi-Fi, possibly a Modbus TCP PLC and an OPC UA server. 2 to 10 users. One integrator (the author) with occasional remote access. Internet may be absent.

### B. Multi-site
Several small-plant gateways, each fully autonomous (decision 1), plus an optional central system: a cloud or HQ instance that receives synced history and alarms for fleet dashboards and reporting. Each site keeps its own users, CA, and audit. The central system is a consumer, not a controller.

## 2. Assets

| ID | Asset | Why it matters | Primary property |
|---|---|---|---|
| A1 | Ability to write to devices (setpoints, commands) | Physical effect on process | Integrity, authorization |
| A2 | Configuration (revisions, templates, alarm limits) | Wrong config = wrong alarms, wrong totals | Integrity |
| A3 | Live values and alarms | Operators act on them | Integrity, availability |
| A4 | History, totals (`total_volume`), events | Billing, compliance reports, leak evidence | Integrity, availability |
| A5 | Audit log | Accountability, incident investigation | Integrity (tamper evidence) |
| A6 | Credentials: user passwords, API keys, device MQTT credentials, CA private key, secret-store master key | Keys to everything else | Confidentiality |
| A7 | Edge box OS and container images | Root of all software trust | Integrity |
| A8 | Device firmware (ESP32) | Can lie about values | Integrity |
| A9 | Backups | Contain A2, A4, A5 and possibly A6 | Confidentiality, integrity |
| A10 | Plant topology and tag names | Reconnaissance value | Confidentiality (low) |

## 3. Actors

| ID | Actor | Capability assumed | In scope |
|---|---|---|---|
| T1 | Opportunistic internet attacker (scanners, botnets, ransomware crews) | Automated scans, known CVEs, default credentials | Yes, primary |
| T2 | Malicious or careless insider with LAN access (contractor laptop on plant network) | Network access to OT LAN, no credentials | Yes |
| T3 | Authenticated user exceeding their role (operator trying engineer actions) | Valid `viewer`/`operator` login | Yes |
| T4 | Physical attacker with access to a field device (ESP32 in a meter pit) | Can open device, read flash, reflash | Yes |
| T5 | Physical attacker with access to the edge box (theft, cabinet access) | Can remove disk, boot other media | Yes, partially (disk encryption, not hardened enclosure) |
| T6 | Compromised supply chain (dependency, base image, CI) | Injects code into a release | Yes |
| T7 | Compromised central/cloud system or integrator laptop (multi-site) | Holds sync credentials, remote access | Yes |
| T8 | Nation-state actor targeting critical infrastructure | Custom tooling, 0-days | Out of scope as a design target. We document that we do not defend against T8 and that critical infrastructure customers need defence in depth beyond Sky Ranger |

## 4. Trust boundaries (small plant)

```
                 Internet (untrusted)
                     │  outbound only (B7)
 ┌──────────── Site firewall ───────────────────────────────────────────┐
 │ Enterprise / office LAN (Purdue L4)                                   │
 │   office PCs, optional IdP (AD / OIDC)                                │
 └───────────────┬──────────────────────────────────────────────────────┘
                 │ B1: HTTPS 8443 only (to DMZ proxy or to gateway)
 ┌───────────────┴──── OT firewall ─────────────────────────────────────┐
 │ Supervisory LAN (L2): edge box, HMI clients                           │
 │  ┌────────────── edge box (B5: host boundary) ──────────────────┐    │
 │  │ skyrangerd ──UDS (B4)── driver processes                      │    │
 │  │     │ TLS (B6)                                                │    │
 │  │ PostgreSQL     Mosquitto (B3 for devices)   secret store      │    │
 │  └───────────────────────────────────────────────────────────────┘    │
 └───────────────┬──────────────────────────────────────────────────────┘
                 │ B2: field protocols (MQTT/TLS 8883, Modbus TCP 502, OPC UA 4840)
 ┌───────────────┴──────────────────────────────────────────────────────┐
 │ Field LAN / Wi-Fi (L1/L0): ESP32 meters, PLCs, OPC UA servers         │
 └───────────────────────────────────────────────────────────────────────┘
```

| ID | Boundary | Crossing | Controls |
|---|---|---|---|
| B1 | User to gateway | HTTPS REST, WSS | TLS, bearer tokens, roles, audit (02, 03, 08) |
| B2 | Gateway to field devices | MQTT, Modbus TCP, OPC UA | TLS where the protocol allows, per-device identity, segmentation (03, 04, 05) |
| B3 | Device to broker | MQTT CONNECT, PUBLISH | Per-device credentials, ACL restricted to own Sparkplug topics (04) |
| B4 | `skyrangerd` to driver process | gRPC over UDS | Socket permissions, peer UID check, separate UID per driver (03, 09) |
| B5 | Container/host | Kernel, mounts | Rootless/non-root containers, read-only FS, seccomp, dropped capabilities (09) |
| B6 | `skyrangerd` to PostgreSQL and Mosquitto | TCP on internal container network | TLS, per-service DB roles, Mosquitto ACL for internal bus (03, 09) |
| B7 | Site to cloud / remote access | Outbound HTTPS / MQTT-TLS / WireGuard | Outbound-only, mTLS, one-way data, MFA for remote humans (05) |
| B8 | Update and supply chain | Image pull or USB bundle | Signatures, digest pinning, SBOM (07) |

Multi-site adds B9: **site to central**. Each site authenticates to central with its own client certificate. Central never holds credentials that can write to a site. Compromise of central (T7) exposes synced data (A3, A4, A10) for all sites but does not give control of any site.

## 5. STRIDE: small plant

| # | Boundary | STRIDE | Threat | Mitigation | Where |
|---|---|---|---|---|---|
| 1 | B1 | S | Credential stuffing / brute force on `/auth/login` | argon2id, per-account and per-IP throttling with exponential delay, no username enumeration, optional TOTP | 02 |
| 2 | B1 | S | Stolen session token replayed from another machine | Short idle timeout, absolute lifetime 12 h, logout revokes, tokens never in URLs or logs, HTTPS only | 02 |
| 3 | B1 | T | Operator changes alarm limits or config | Role check: config apply requires `engineer`; every mutation audited with before/after | 02, 08 |
| 4 | B1 | R | User denies acknowledging an alarm or writing a setpoint | Audit record with user, time, source IP, request id, before/after; hash chained | 08 |
| 5 | B1 | I | Viewer reads secrets via config export | Secrets referenced as `secret://`, export never includes values (tag-model.md section 9) | 06 |
| 6 | B1 | I | Passive sniffing of HMI traffic on the LAN | TLS 1.2+ (prefer 1.3), HSTS | 03 |
| 7 | B1 | D | HMI flood (many WS connections, huge history queries) | Contract limits (20 WS per user, 100 subs, history caps) plus rate limits per key and per IP | 02, 09 |
| 8 | B1 | E | XSS in HMI via tag `display_name` or alarm `message` | React escaping, strict CSP with no `unsafe-inline` scripts, no `dangerouslySetInnerHTML` on config strings | 09 |
| 9 | B1 | E | Authorization bug lets `viewer` call engineer endpoint | Permission table enforced in one middleware, deny by default, automated test that iterates every route against every role | 02, 14 |
| 10 | B2/B3 | S | Rogue device publishes as `esp32-meter01` | Per-device MQTT credential bound by ACL to its own `edge_node_id` topics; unknown credentials rejected | 04 |
| 11 | B2/B3 | S | Rogue device appears as a new, plausible meter | Discovery never creates tags (decision 4); approval is an `engineer` action; operator verifies serial on device label | 04 |
| 12 | B2/B3 | T | Device sends implausible counts (tampered firmware or bug) | `counter_total` `max_delta` marks `implausible`, raw history retained for recomputation | tag-model.md 7 |
| 13 | B2/B3 | T | Attacker spoofs Sparkplug `STATE` or `NCMD` toward devices | Only the gateway's credential may publish `spBv1.0/STATE/{host_id}` and `NCMD`/`DCMD`; devices cannot | 04 |
| 14 | B2 | T | Modbus TCP write injection from field LAN | Modbus has no authentication. Field LAN isolation, PLC-side write protection where available; documented residual risk | 05, 11 |
| 15 | B2 | I | Wi-Fi sniffing of meter data | MQTT over TLS (8883); WPA2/WPA3 on the field SSID as a second layer | 03, 04 |
| 16 | B2 | D | Wi-Fi jamming or deauth kills meters | Not preventable in software. Detected: `health` goes 0/1, `not_ok` alarm fires; counts are cumulative so no data lost on recovery except across reboot | tag-model.md 11 |
| 17 | B3 | E | Device subscribes to `#` and reads everything | Device ACL: publish only own `spBv1.0/{group}/+/{edge_node_id}[/#]`, subscribe only own `NCMD`/`DCMD` and `STATE` | 04 |
| 18 | B3 | E | Device publishes on `skyranger/v1/...` internal bus | Only `skyrangerd` principal may publish there (messages.md section 3) | 04 |
| 19 | B4 | S/E | Another local process connects to a driver socket and issues `Write` | Socket dir `0750` owned by `skyrangerd`; driver checks peer UID with `SO_PEERCRED`; only the host may dial | 03, 09 |
| 20 | B4 | E | Compromised driver (parser bug in Modbus library) attacks host or other drivers | Separate UID per driver, no network except its target, seccomp, no access to DB or secret store; the host passes only that connection's resolved settings | 09 |
| 21 | B5 | E | Container escape | Non-root, no `--privileged`, no host network, read-only root FS, dropped caps, default seccomp, AppArmor/SELinux where available | 09 |
| 22 | B5 | T | Stolen disk reveals DB and keys | LUKS full-disk encryption with TPM-sealed unlock; master key sealed to TPM | 06, 09 |
| 23 | B6 | I/T | Attacker on internal network reads or edits DB | PostgreSQL listens only on internal container network with TLS and SCRAM-SHA-256 per role; Mosquitto internal listener requires credentials | 03, 09 |
| 24 | B6 | T | DB admin edits history or audit rows | Audit hash chain and signed checkpoints detect it; history immutability is a non-goal (see 08 for scope) | 08 |
| 25 | B7 | S | Fake cloud endpoint receives site data | Site pins central CA; mTLS with per-site client cert | 03, 05 |
| 26 | B7 | E | Compromised cloud pushes commands to site | v1 cloud sync is one-way; no API or topic exists for central-to-site writes | 05 |
| 27 | B7 | E | Remote access tool (TeamViewer etc.) left open | Remote access only over outbound WireGuard to a jump host, enabled per session by a local admin, MFA, auto-expiry, audited | 05 |
| 28 | B8 | T | Malicious image or update bundle | cosign signature verification against pinned public key before install; digest pinning in compose | 07 |
| 29 | B8 | T | Compromised dependency | Go module checksum DB, `govulncheck`, `npm ci` from lockfile, SBOM per release, minimal dependencies | 07 |
| 30 | B8 | T | Malicious ESP32 OTA | Firmware signed (ESP-IDF secure boot v2 on production devices); gateway refuses unsigned images | 04, 07 |
| 31 | all | R | Logs rotated or deleted to hide activity | Audit is in DB with hash chain; optional syslog forwarding to customer SIEM | 08 |
| 32 | all | D | Disk full stops history and audit | Retention policies, disk alarms at 80/90%, audit write failure blocks mutating requests (fail closed for changes, open for monitoring) | 08, 09 |
| 33 | B1 | S | Default credentials on first boot | No default password exists. First-boot setup creates the admin from a one-time setup code printed on the console/label | 02 |

## 6. STRIDE: additions for multi-site

| # | Boundary | STRIDE | Threat | Mitigation |
|---|---|---|---|---|
| M1 | B9 | S | One site impersonates another to central | Per-site client cert, subject CN = `gateway_id`; central maps cert to site and rejects data for other `gateway_id`s |
| M2 | B9 | T | Replay or duplicate synced data | Sync messages carry CloudEvents `id` (UUIDv7) and are de-duplicated (messages.md section 4) |
| M3 | B9 | I | Central breach exposes all sites' data | Data minimisation: sync only configured tag prefixes; encryption at rest on central; separate tenant per customer |
| M4 | B9 | E | Shared credentials across sites let one breach spread | No shared secrets across sites: each site has its own CA, admin accounts, device credentials, backup key |
| M5 | B9 | E | Central admin uses fleet tooling to change every site at once | Not in v1. Fleet config push, if added, requires a site-local approval (draft created remotely, applied locally by a site `engineer`) |
| M6 | B9 | R | Cannot tell which central user saw which site's data | Central has its own audit; site audit records sync sessions |
| M7 | B9 | D | Central outage | No effect on sites (decision 1); sync queues and resumes |
| M8 | Integrator laptop | E | Integrator laptop holds keys for every customer | Per-customer credentials in a password manager, hardware-backed SSH/WireGuard keys, no customer CA keys on the laptop after install (CA key stays on the box or offline) |

## 7. Residual risks we accept and state to customers

- Modbus TCP and other unauthenticated protocols: anyone on the field LAN can write to the PLC directly, bypassing Sky Ranger. Mitigation is network design, not Sky Ranger.
- An attacker with root on the edge box can do anything the box can do, including writes. We provide detection (audit chain, signed checkpoints off-box), not prevention.
- ESP32 devices without secure boot and flash encryption (development units) can be cloned by anyone with physical access. Production devices must have both enabled (04).
- Wi-Fi field devices can be jammed.
- Sky Ranger does not protect against T8.

## Decisions made
1. Primary design adversaries are T1 to T7; T8 is explicitly out of scope and documented as such.
2. Multi-site central is a read-only consumer of site data in v1; no control path from central to site.
3. No shared secrets across sites, ever.
4. Audit write failure blocks mutating requests but never blocks monitoring.

## Alternatives rejected and why
- **Attack trees instead of STRIDE**: more detail per threat but much slower to maintain for a one-person team; STRIDE per boundary gives adequate coverage.
- **Treating the OT LAN as trusted**: the most common real-world ICS failure; contractor laptops and flat networks are normal.
- **Central-to-site writes in v1**: turns a cloud breach into a plant incident; value does not justify the risk at this stage.

## Open questions
- Should the threat model be re-reviewed by an external ICS security consultant before the first paid deployment? (Recommended; budget dependent.)
- When Python scripting lands, scripts become a new boundary (user code with API access). Needs its own STRIDE pass.
- When northbound Sparkplug or an OPC UA server is enabled, third-party hosts become a boundary; not modelled here yet.
