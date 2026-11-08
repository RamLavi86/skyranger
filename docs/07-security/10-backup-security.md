# 10 Backup Security

Ownership: the deployment area owns *how* backups are scheduled and restored (nightly `skyranger-backup.timer` running `skyrangerctl backup create --rotate`, restore with `skyrangerctl restore --from backup.tar`, data under `/var/lib/skyranger/backups`; see [../08-deployment/](../08-deployment/README.md)). This file owns the *security* requirements: what is in a backup, how it is encrypted, who can read it, and how we know a restore works. Related: [06-secrets.md](06-secrets.md) (KEK, recovery kit), [08-audit-logging.md](08-audit-logging.md) (audit in backups), [07-secure-update.md](07-secure-update.md) (pre-update backup).

A backup is the single most valuable file on a site: it holds configuration (A2), history and totals (A4), the audit log (A5), and, in encrypted form, secrets and CA keys (A6, A9). A stolen unencrypted backup is a complete compromise of the site. Backups are therefore **always encrypted, with a key that is not on the box**.

## 1. What a backup contains

| Content | Included | Notes |
|---|---|---|
| PostgreSQL dump: config revisions, HMI revisions, users and sessions metadata, API key hashes, `secrets` table (already ciphertext), `audit_log`, alarms, events, `derived_state` | yes | `pg_dump` custom format; history tables (TimescaleDB hypertables) are included by default, optionally excluded for config-only backups |
| PKI store (`/var/lib/skyranger/pki/`): issuing CA, service keys, CRL, audit signing key | yes, encrypted by the backup key **in addition to** being wrapped by the KEK | Root CA key is not on the box by default (03 section 1), so it is not in the backup |
| Site identity: `gateway_id`, host settings, `/etc/skyranger` (without `secrets/`) | yes | |
| KEK (master key) | **no** | The KEK is in the recovery kit (06 section 5), never in the same artifact as the data it protects |
| Session tokens | no (hashes only exist; restored sessions are revoked on restore) | |
| Mosquitto dynamic security database | yes | Device credential hashes and ACLs |
| Host OS, container images | no | Reproducible from signed releases (07) |
| `/run/secrets`, TPM state | no | |

A manifest (`manifest.json`, signed by the audit signing key) lists contents, sizes, SHA-256 of each part, `gateway_id`, Sky Ranger version, schema version, `created_at`, and the last audit `seq` and hash included. The manifest is outside the encryption only if it contains no sensitive data; otherwise it is inside. Default: inside, with a second plaintext `backup.meta` containing only `gateway_id`, version, `created_at`, size, and the recipient key fingerprint, so an operator can identify a file without decrypting it.

## 2. Encryption

| Item | Decision |
|---|---|
| Tool and format | **age** (X25519 recipients, ChaCha20-Poly1305 chunked streaming), library `filippo.io/age` in Go. Chosen for being small, audited in design, and having no configuration to get wrong |
| Recipients | At least two public keys configured per site: (1) the **recovery key** (offline, in the recovery kit, 06 section 5), (2) an optional **customer key** if the customer wants to hold their own copy. The box holds public keys only. Compromise of the box therefore does not expose old backups |
| Where the private key lives | Recovery kit: age identity printed on paper (QR) in the customer's safe and stored on a hardware token or encrypted USB held by the integrator. Never on the edge box, never in the support bundle |
| Streaming | `pg_dump | tar | age -r ... > backup.tar.age`. Nothing unencrypted is written to disk; the staging area is a tmpfs or pipe |
| Integrity | age authenticates ciphertext. The signed manifest additionally binds the parts together and proves they came from this site |
| Cipher agility | Backup header records the age version. A format change ships with a restore test for old files |
| Passphrase mode | age's scrypt passphrase recipient is allowed for sites that refuse a key kit, with a policy of at least 6 random words, stored by the customer. It is weaker operationally (humans forget); the release checklist warns |

## 3. Storage locations and access

| Destination | Security requirement |
|---|---|
| `/var/lib/skyranger/backups` on the box | Directory `0700` root, on the LUKS volume. Protects against accidental deletion, **not** against box loss: it must not be the only copy |
| Second disk or USB | Mount `nodev,nosuid,noexec`. The file is ciphertext, so theft exposes nothing but its existence and size; still labelled and kept in a locked cabinet |
| SMB/NFS share | Credentials in the secret store; write-only share account (create, no delete, no list) where the server supports it, so a compromised edge box cannot erase its own backups. SMB signing/encryption or NFS over Kerberos where available (UNVERIFIED per customer environment) |
| S3-compatible bucket (outbound) | Dedicated access key with `PutObject` only; bucket versioning and object lock (retention) on, so a compromised box cannot delete or overwrite history. Egress allow-list entry per 05 section 6. TLS to the endpoint, verified |
| Integrator copy | Allowed only if the customer agrees in the handover document; encrypted file only, never the recovery key together with it |

Rules:
- At least **one copy off the box and off the network segment**, documented at handover. The 3-2-1 rule (three copies, two media, one off-site) is the target; the minimum for a customer who will not pay for more is the box plus one off-box copy.
- Backup retention: daily for 14 days, weekly for 8 weeks, monthly for 12 months (defaults; deployment owns the rotation tool). Deleting a backup is an audited action (`system.backup_deleted`, proposed addition to the list in 08).
- Reading or downloading a backup through the API/HMI is an `admin` action (`system.manage`), audited. Backup files are never served over the unauthenticated or `viewer` surface.
- Backups of the audit log protect against deletion of the live log. The off-box copy is one of the "off-box copies" in 08 section 3.

## 4. Backups and secrets

- The `secrets` table in the dump is already AES-256-GCM ciphertext wrapped by the KEK. The backup layer encrypts the whole dump again with age. Two layers, two different keys held in two different places: restoring the data requires the **age identity**; reading the secrets inside additionally requires the **KEK**, which on a new box is re-sealed from the recovery kit (06 section 5).
- The recovery kit therefore holds two things: the age identity (to open backups) and the encrypted KEK export (to re-seal secrets). They are stored together in the customer's safe because they are only useful together with the backup files, which are elsewhere. Losing the kit means losing the secrets: the site can be restored, and every secret (device credentials, OPC UA passwords, OIDC secret) is re-issued. This is the designed degraded outcome.
- Device MQTT credentials are not in the secret store (06 section 1): only hashes live in the Mosquitto database. After a restore from backup the credentials still work because the hashes are restored. After a restore *without* the Mosquitto database, devices must be re-provisioned (04).

## 5. Restore safety

Restoring is a high-risk operation: it can resurrect revoked users, old keys, and old config.

1. **Authorization:** restore is a console action (`skyrangerctl restore`, requires root on the host) or `admin` via API with a fresh re-authentication. It is audited as `system.backup_restored` with the backup's `created_at`, manifest hash and last audit `seq`.
2. **Verify before apply:** decrypt, check the manifest signature against the audit signing key public part (carried in `backup.meta`/recovery kit) and every part hash; refuse on mismatch. A restore from a backup with a different `gateway_id` requires `--as-new-site` or `--replace-site` and an explicit confirmation, because it would clone a site identity (two sites with one CA).
3. **Revocations are not undone silently:** after restore, `skyrangerd` replays the **revocation list** it holds in memory from before the restore (when restoring onto the same running box) and prints a table of users, API keys and devices that exist in the backup but were disabled or revoked since. Sessions are all revoked on restore. API keys and users created after the backup are listed as "lost, recreate".
4. **Audit continuity:** the restored audit log ends at the backup's last `seq`. The tool writes `audit.restore_gap` with the previous known head hash (from the off-box checkpoint copies, 08) so a reviewer can see that the chain was rolled back, when, and by whom. Restoring and then pretending the later records never existed is exactly what tamper evidence is meant to expose.
5. **Post-restore rotation:** if the backup file or restore media left the customer's control, or the reason for restore is a suspected compromise, rotate: all passwords (forced change), all API keys, issuing CA (03 section 1), device credentials, KEK (`skyrangerctl secrets rotate-kek`), audit signing key. The restore wizard asks which of the two situations applies and prints the rotation checklist.
6. **Version rule:** restore into the same or a newer Sky Ranger version; migrations run forward (deployment area). Restoring onto an older version is refused.

## 6. Restore testing

An untested backup is a hope. Requirements:

| Test | Frequency | How |
|---|---|---|
| Automatic verify | every backup | The box cannot decrypt its own backups (it has no private key), so it verifies the age header, the file size and the SHA-256 of the ciphertext against values computed while writing; `system.backup_created` records them |
| Decrypt and list | monthly, by the integrator | With the recovery kit on the integrator's workstation: decrypt the latest file, verify the manifest signature, list contents |
| Full restore drill | at commissioning, then every 6 months, and before each major upgrade | Restore onto a scratch VM or the cold spare (08-deployment cold spare procedure); start the stack; run the water meter acceptance test (05-hmi-api/09 section 7) against it; confirm `skyrangerctl audit verify` passes and `GET /system/health` is `ok` |
| Kit audit | yearly | Confirm the customer can find the paper recovery key and that the QR still scans; confirm the token works |

A failed drill blocks the release checklist item "backup restore tested" (14). The date of the last successful drill is shown on the admin page and in the monthly integrator report.

## 7. Threats covered

| Threat | Control |
|---|---|
| Backup file stolen (USB, share, cloud) | age encryption to an offline key; no key on box or beside the file |
| Ransomware on the edge box deletes backups | Write-only or object-locked off-box copy; second disk is unmounted outside the backup window |
| Attacker restores an old backup to resurrect a deleted admin | Restore needs root console or fresh admin re-auth; revocation diff and mandatory rotation question; audit gap record |
| Backup tampered with to inject config | Signed manifest plus age authentication; restore refuses unsigned or mismatched parts |
| Backup contains plaintext secrets | Secrets table is ciphertext under the KEK; KEK is excluded |
| Lost recovery kit | Documented outcome (secrets re-issued); yearly kit audit |

## Decisions made
1. Every backup is encrypted with age to at least one offline recovery key; the box holds only public keys.
2. The KEK is excluded from backups and travels separately in the recovery kit; backups contain secrets only as KEK-wrapped ciphertext.
3. A signed manifest binds backup contents to the site and to a point in the audit chain.
4. At least one copy lives off the box; remote destinations use write-only or object-locked credentials.
5. Restore is a privileged, audited, verify-first operation that reports revoked principals, writes an audit gap record, and offers rotation after suspected exposure.
6. Restore drills are scheduled and gate release.

## Alternatives rejected and why
- **GPG for encryption**: ubiquitous, but key management and options are error-prone; age is simpler and sufficient.
- **Backup encrypted with a key stored on the box**: protects only against casual theft of the file, not against theft of the box or its disk.
- **Backing up the KEK with the data**: collapses two protections into one.
- **A passphrase as the only option**: humans lose passphrases; keys in a kit are recoverable.
- **Cloud backup by default**: violates edge-first (decision 1); it is an optional destination only.
- **Block-level disk snapshots as the only backup**: restores everything including malware and old keys, cannot be moved to new hardware cleanly, and is hard to verify.

## Open questions
- Whether `pgBackRest` or WAL archiving (point-in-time recovery) is worth the complexity for customers with billing-grade totals; `pg_dump` nightly loses up to a day of history on disk loss. Deployment area to decide; the encryption requirements here apply either way.
- Exact object-lock and write-only share semantics per storage vendor are UNVERIFIED.
- `system.backup_deleted` and `audit.restore_gap` are proposed additions to the audit action list in 08.
- Should the customer-held second recipient key be mandatory so the integrator alone cannot restore a site (separation of duties)?
- Retention defaults (14 daily, 8 weekly, 12 monthly) are placeholders until storage sizing from the deployment doc is final.
