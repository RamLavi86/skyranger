# 06 Secrets Management on the Edge Box

Contract: config never contains secrets; it references them as `secret://...` URIs and export never includes their values (tag-model.md section 9). This file defines the store behind those URIs.

## 1. Inventory

| Secret | URI / location | Consumer | Rotation |
|---|---|---|---|
| Broker credential for the sparkplug driver | `secret://mqtt/gateway` (contract example) | sparkplug driver via host | on demand |
| Device MQTT credentials | not in the secret store; only the hash lives in Mosquitto dynamic security; plaintext shown once at provisioning | Mosquitto | per device (04) |
| OPC UA client cert + key, user password | `secret://opcua/{connection}/client`, `secret://opcua/{connection}/user` | opcua driver | cert 90 d auto |
| Modbus: none | | | |
| OIDC client secret, LDAP bind | `secret://auth/oidc_client_secret`, `secret://auth/ldap_bind` | `skyrangerd` | on demand |
| TOTP seeds | `secret://users/{user_id}/totp` | `skyrangerd` | on re-enroll |
| PostgreSQL role passwords | files in `/run/secrets/` (not in DB: needed to reach the DB) | `skyrangerd`, migration job | yearly / on staff change |
| Site issuing CA key, service TLS keys | PKI store, `/var/lib/skyranger/pki/` encrypted with the master key | CA module, Mosquitto, Postgres | per 03 |
| Audit checkpoint signing key (Ed25519) | PKI store | audit module | yearly, old public keys kept |
| Backup encryption public key | config (public, not secret); private key off-box only | backup job | per 10 |
| Master key (KEK) | TPM-sealed blob or passphrase-derived (section 3) | `skyrangerd` at start | rare |

## 2. Secret store design

Storage: table `secrets` in PostgreSQL, so secrets are backed up and restored with config (encrypted, section 4 of 10).

```sql
CREATE TABLE secrets (
  name        text PRIMARY KEY,       -- path after secret://, e.g. 'opcua/plc-line1/user'
  version     integer NOT NULL,
  ciphertext  bytea NOT NULL,         -- AES-256-GCM(DEK, plaintext), nonce || ct || tag
  wrapped_dek bytea NOT NULL,         -- AES-256-GCM(KEK, DEK) (key wrap)
  kek_id      text NOT NULL,          -- which master key version wrapped it
  created_at  timestamptz NOT NULL,
  created_by  text NOT NULL,
  updated_at  timestamptz NOT NULL
);
```

- Envelope encryption: each secret has a random 256-bit data key (DEK); DEKs are wrapped by the master key (KEK). The secret `name` is passed as AES-GCM additional data so ciphertexts cannot be swapped between names.
- Only `skyrangerd` holds the KEK, in memory. The PostgreSQL role used by other components has no `SELECT` on `secrets`.
- API (additive, admin only): `PUT /api/v1/secrets/{name}` (write-only), `GET /api/v1/secrets` (names, versions, timestamps; never values), `DELETE /api/v1/secrets/{name}`. There is no read endpoint for values. Every call is audited (name only).
- Resolution: at config apply, `skyrangerd` resolves each `secret://` reference and passes the plaintext to the specific driver process inside `ConfigureRequest.settings` over the UDS (03 section 6). Drivers keep it in memory only and must not log it (`Error.Message` has "no secrets" per contract; the SDK redacts known secret fields in logs).
- Validation: a config revision that references a non-existent secret fails validation (`not_found` at the JSON Pointer of the reference).

Library: Go standard `crypto/aes`, `crypto/cipher` (GCM), `crypto/rand`. No custom cipher constructions.

## 3. Master key (KEK) protection

Options, in order of preference:

| Mode | How | Unattended reboot | Protects against disk theft | Notes |
|---|---|---|---|---|
| `tpm` (default when TPM 2.0 present) | KEK sealed to the TPM, policy bound to PCRs measuring firmware/bootloader (PCR selection UNVERIFIED per platform; start with PCR 7 Secure Boot state). Implemented with `go-tpm` or `systemd-creds --with-key=tpm2` | yes | yes (disk alone is useless) | Firmware/BIOS update may change PCRs; recovery passphrase needed (section 5) |
| `luks-keyfile` | KEK file on the LUKS-encrypted root FS, root-only `0400`, which itself unlocks via TPM (`systemd-cryptenroll --tpm2-device=auto`) | yes | yes, via LUKS | Simpler; KEK readable by root while running |
| `passphrase` | KEK derived from an operator passphrase with argon2id at startup | **no**: needs a human at every boot | yes | Only for sites that accept it; a SCADA box that does not come back after a power cut is usually unacceptable |
| `plain` | KEK file on unencrypted disk | yes | **no** | Development only; the release checklist (14) fails it |

Recommended hardware for customer boxes: a platform with TPM 2.0 (most current industrial PCs; Raspberry Pi has none without an add-on). The deployment doc owns the hardware list.

Using `systemd-creds` (systemd 250+, UNVERIFIED exact version) is attractive because it is boring and well documented: `systemd-creds encrypt --with-key=tpm2 kek.bin kek.cred` and `LoadCredentialEncrypted=kek:/etc/skyranger/kek.cred` in the unit; the container receives it through a bind mount of `$CREDENTIALS_DIRECTORY`. This keeps TPM handling out of our Go code.

## 4. Container secret delivery

- Files, not environment variables. Env vars leak through `/proc/*/environ`, crash dumps, `docker inspect`, and support bundles.
- Compose example:

```yaml
services:
  skyrangerd:
    image: registry.example/skyranger/skyrangerd@sha256:...   # digest pinned (07)
    secrets:
      - source: pg_skyrangerd_password
        target: /run/secrets/pg_password
        mode: 0400
      - source: kek
        target: /run/secrets/kek
        mode: 0400
secrets:
  pg_skyrangerd_password:
    file: /etc/skyranger/secrets/pg_skyrangerd_password   # root:root 0400 on the host, on LUKS volume
  kek:
    file: /run/credentials/skyranger.service/kek          # decrypted by systemd-creds at boot, tmpfs
```

- The KEK file lives on tmpfs (`/run/credentials/...`) and is never written to persistent storage in clear.
- Support bundles (`skyrangerctl support-bundle`) exclude `/run/secrets`, `/etc/skyranger/secrets`, the PKI store, and redact any string matching `sr_[a-z0-9]+_` and PEM private key blocks.

## 5. Recovery

If the TPM is replaced (motherboard swap) or PCRs change unexpectedly, the KEK is unrecoverable from the box. Therefore at install:
1. `skyrangerctl secrets export-recovery` writes the KEK encrypted to the **recovery public key** (age X25519) on the integrator's recovery kit (10).
2. The recovery kit (age private key on paper/QR and on a hardware token, in the customer's safe and with the integrator) can re-seal the KEK on new hardware.
3. Rotation of the KEK (`skyrangerctl secrets rotate-kek`) re-wraps all DEKs; old KEK version retained until all rows use the new `kek_id`.

## Decisions made
1. Secrets live in PostgreSQL with envelope encryption (AES-256-GCM, per-secret DEK, name as AAD), only `skyrangerd` can decrypt.
2. The master key is TPM-sealed by default (via `systemd-creds` or `go-tpm`), with LUKS full-disk encryption underneath; `passphrase` mode is available but not default.
3. Secrets are write-only through the API; values are never returned, exported, or logged.
4. Containers receive secrets as files on tmpfs, never as environment variables.
5. A recovery copy of the KEK is encrypted to an offline age key held in the recovery kit.

## Alternatives rejected and why
- **HashiCorp Vault / OpenBao on the edge**: proven, but a stateful service that must be unsealed, monitored, and backed up; disproportionate for a dozen secrets. May be supported as an external `secret://` backend for enterprises later.
- **SOPS-encrypted YAML files**: good for GitOps, but the decryption key still needs a home on the box and the API would need to edit files.
- **Kubernetes Secrets**: no Kubernetes on the edge.
- **Environment variables**: leak too easily.
- **Plain DB columns protected by DB permissions only**: a DB dump or backup leak would expose everything.

## Open questions
- Contract addition: `/api/v1/secrets` endpoints (admin, write-only).
- TPM PCR policy selection and `systemd-creds` version requirements are UNVERIFIED and platform specific.
- Should drivers receive secrets via a separate RPC (e.g. `GetSecret` from host) rather than embedded in `settings`, so `settings` can be logged safely? Embedding is simpler and matches `Configure(settings)`; the SDK redacts.
