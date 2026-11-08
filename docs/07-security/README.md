# 07 Security

Status: design v1 (draft). Owner: security area. Conforms to [../00-contracts/](../00-contracts/README.md). Where this area needs something the contracts do not yet say, it is listed as a proposed **additive** change in the relevant file's Open questions; nothing here renames or removes a contract field, topic, role, or quality code.

Sky Ranger will be asked hard questions before a customer replaces a Siemens or Rockwell system: "Who can change a setpoint?", "What happens if your box is stolen?", "How do I know the update is yours?", "Where is the audit trail?". This folder is the written answer. Each answer has to be implementable by one integrator with ordinary tools and has to survive an audit.

## Files

| # | File | Answers |
|---|---|---|
| 01 | [01-threat-model.md](01-threat-model.md) | What we protect, from whom, trust boundaries, STRIDE table for a small plant and for multi-site |
| 02 | [02-authn-authz.md](02-authn-authz.md) | Local users, roles to permissions, API keys, service accounts, sessions, MFA, OIDC/LDAP |
| 03 | [03-tls-and-pki.md](03-tls-and-pki.md) | Site CA, certificates, TLS on HTTPS, MQTT, PostgreSQL, OPC UA security policies, gRPC driver boundary |
| 04 | [04-device-identity.md](04-device-identity.md) | ESP32 identity, per-device credentials, certificate vs password, provisioning, MQTT ACLs, revocation, firmware security |
| 05 | [05-network-segmentation.md](05-network-segmentation.md) | Purdue mapping, reference small-site network, firewall rules, outbound-only cloud, remote access |
| 06 | [06-secrets.md](06-secrets.md) | `secret://` store, master key, TPM, what lives where |
| 07 | [07-secure-update.md](07-secure-update.md) | Signed images, rollback, offline update bundles, supply chain, SBOM, ESP32 OTA |
| 08 | [08-audit-logging.md](08-audit-logging.md) | What is audited, record schema, hash chain, signed checkpoints, forwarding |
| 09 | [09-hardening.md](09-hardening.md) | Edge box OS, containers, driver processes, PostgreSQL, Mosquitto, HMI headers |
| 10 | [10-backup-security.md](10-backup-security.md) | Encrypted backups, recovery kit, restore testing |
| 11 | [11-safety.md](11-safety.md) | SCADA is not a safety system; what the product must never claim |
| 12 | [12-iec62443.md](12-iec62443.md) | Foundational requirements mapping and a realistic certification path |
| 13 | [13-vulnerability-disclosure.md](13-vulnerability-disclosure.md) | How to report, timelines, advisories, CVEs |
| 14 | [14-release-checklist.md](14-release-checklist.md) | The gate every customer deployment must pass |

## Security principles (normative)

1. **Edge-first applies to security too.** Every security control works with the internet unplugged: local login, local CA, local audit, offline update bundles. Cloud and enterprise IdP are optional add-ons, and the site never depends on them to log in, alarm, or record.
2. **Availability is a security property.** Controls must not lock an operator out during an incident. Example: login throttling slows attackers down but never hard-locks accounts (02). Security failures degrade to "read-only and loud", not "dark".
3. **One door for change.** All writes, acks, and config changes go through the REST API (api.md, messages.md section 3). That gives one place for authentication, authorization, validation, and audit. No side channels: no command topics, no writes over WebSocket, no direct DB edits in normal operation.
4. **TLS on every TCP socket Sky Ranger controls both ends of.** Unencrypted channels exist only as Unix domain sockets (kernel-local, peer-credential checked) or as legacy southbound protocols that have no security (Modbus TCP), which are confined to the field zone by the network design (03, 05).
5. **Every device and service has its own identity.** No shared MQTT passwords, no shared admin account, no "default" credential that ships in an image (04, 02).
6. **Signed or it does not run.** Container images, update bundles, and ESP32 firmware are signed; the edge box verifies before installing (07).
7. **Tamper-evident, not tamper-proof.** We do not claim an attacker with root cannot alter records. We claim that alteration is detectable afterwards (08).
8. **Honest scope.** Sky Ranger is a monitoring and supervisory system. It is not a safety system and the documentation, UI, and sales material never say otherwise (11).
9. **Boring tools.** Go `crypto/tls`, Mosquitto, PostgreSQL roles, nftables, systemd, LUKS, cosign, age, ESP-IDF secure boot and flash encryption. No home-grown cryptography, no home-grown protocols.

## Summary of key decisions (details in each file)

| Area | Decision |
|---|---|
| Identity of humans | Local users with argon2id passwords always available; OIDC optional for enterprise; LDAP later; break-glass local admin always works |
| Authorization | Contract's four cumulative roles, implemented internally as permissions so per-area scoping can be added later without breaking v1 |
| Tokens | Opaque DB-backed tokens as per api.md; session tokens hashed with SHA-256, API keys with argon2id; no JWTs, no cookies |
| MFA | TOTP optional for all users in v1.1, required for `admin` and for any remote access path; not forced on local operators |
| PKI | One CA per site, generated at install, used for broker, HTTPS, PostgreSQL, OPC UA client certs; customer PKI can replace it |
| Device identity (ESP32) | Per-device random MQTT credential over TLS with CA pinning in v1; client certificates (mTLS) as the v2 option; credentials and ACL per device managed by `skyrangerd` in Mosquitto's dynamic security plugin |
| Network | Gateway lives at Purdue level 2; no inbound connections from the internet, ever; cloud sync and remote access are outbound-initiated only |
| Cloud | v1 cloud sync is one-way (site to cloud). No write path from cloud to site |
| Secrets | `secret://` values encrypted in PostgreSQL with a master key sealed by TPM 2.0 when present |
| Updates | cosign-signed images pinned by digest; signed offline bundles; A/B rollback; SBOM per release |
| Audit | Append-only `audit_log` with a SHA-256 hash chain and hourly Ed25519-signed checkpoints, optionally forwarded to syslog |
| Standards | Align the development process with IEC 62443-4-1 now, self-assess components against 62443-4-2 at SL 2, certify only when a customer pays for it |

## Decisions made
1. Security documentation is split by control family, with one checklist (14) that gates every customer delivery.
2. The nine principles above are normative for all other design areas.
3. Contract gaps are raised as proposed additive changes, not implemented silently.

## Alternatives rejected and why
- **One large security document**: hard to review, hard to hand a single section to a customer's security team.
- **Defer security until after milestone 1**: device identity, token storage, and audit schema are expensive to retrofit once devices are deployed in the field.

## Open questions
- Who owns `SECURITY.md` at repo root (13 proposes its content; creating the file is outside this directory).
- A `decisions.md` entry summarising the security decisions should be added by the decisions owner.
