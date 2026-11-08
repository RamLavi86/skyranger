# 08 Audit Logging and Tamper Evidence

Contract: every mutating request is written to `audit_log` and readable at `GET /api/v1/audit` (admin) with user, time, action, target, and before/after values (api.md section 2). This file defines the schema, the event list, and how tampering is detected.

Audit log vs events vs application logs:

| Stream | Purpose | Store | Who reads |
|---|---|---|---|
| `audit_log` | Who did what, accountable, tamper-evident | PostgreSQL table, hash-chained | admin, auditors |
| `events` (CloudEvents) | What happened in the process/system (alarms, discontinuities, config applied) | TimescaleDB hypertable (messages.md section 4) | viewers, integrations |
| Application logs | Debugging (`slog` JSON to stderr) | container log driver, journald | integrator |

Some actions produce both an audit record and an event (config apply produces `skyranger.config.revision_applied.v1` and an audit record). The audit record links to the event via `event_id`.

## 1. What is audited

Everything that changes state or exposes sensitive state, plus security-relevant failures.

| Category | `action` values |
|---|---|
| Auth | `auth.login_succeeded`, `auth.login_failed`, `auth.logout`, `auth.mfa_failed`, `auth.setup_completed`, `auth.break_glass`, `auth.session_revoked`, `auth.oidc_login` |
| Users and keys | `user.created`, `user.updated` (role, enabled), `user.deleted`, `user.password_changed`, `user.totp_enrolled`, `apikey.created`, `apikey.revoked`, `apikey.used_from_disallowed_ip` |
| Tags | `tag.write_requested`, `tag.write_completed` (result), `tag.counter_adjusted` |
| Alarms | `alarm.acked`, `alarm.shelved`, `alarm.unshelved` |
| Config | `config.draft_created`, `config.applied`, `config.restored`, `config.draft_deleted` |
| Discovery and devices | `discovery.approved`, `discovery.ignored`, `discovery.unignored`, `device.provisioned`, `device.enrollment_token_created`, `device.enrolled`, `device.revoked`, `device.rotated`, `device.ota_started` |
| Connections | `connection.restarted`, `connection.browse`, `opcua.server_cert_trusted` |
| Secrets and PKI | `secret.written`, `secret.deleted`, `pki.cert_issued`, `pki.cert_revoked`, `pki.ca_rotated`, `tls.cert_uploaded` |
| System | `system.update_applied`, `system.update_rolled_back`, `system.backup_created`, `system.backup_restored`, `system.remote_access_enabled`, `system.remote_access_disabled`, `system.settings_changed`, `system.started`, `system.stopped`, `audit.checkpoint` |
| Access to sensitive reads | `audit.read`, `history.exported` (CSV export of large ranges) |

Not audited per request: normal reads of tags, history, alarms (too much volume, low value). WebSocket subscribe is not audited; WebSocket `hello` failures are, as `auth.login_failed` with `channel: ws`.

`auth.login_failed` stores the attempted username only if it matches an existing user (prevents storing passwords typed into the username field).

## 2. Record schema

```sql
CREATE TABLE audit_log (
  seq          bigint PRIMARY KEY,               -- gapless, assigned under lock; never reused
  audit_id     uuid NOT NULL UNIQUE,             -- UUIDv7
  ts           timestamptz NOT NULL,             -- gateway clock, ms precision
  actor        text NOT NULL,                    -- 'user:alice', 'service:svc-cloud-sync', 'system:skyrangerd', 'system:console'
  actor_role   text,                             -- role at time of action
  session_id   uuid,                             -- or api key_id in key_id
  key_id       text,
  source_ip    inet,
  channel      text NOT NULL,                    -- 'rest', 'ws', 'console', 'internal'
  request_id   uuid,                             -- = X-Request-Id (api.md section 7)
  action       text NOT NULL,
  target_kind  text,                             -- 'tag', 'alarm', 'config_revision', 'user', 'device', ...
  target_id    text,                             -- tag_id, alarm_id, revision number, user_id, edge_node_id
  target_path  text,                             -- tag path at time of action, for readability
  outcome      text NOT NULL CHECK (outcome IN ('success','failure','denied')),
  reason       text,                             -- user comment (required for shelve, counter adjust) or error code
  before       jsonb,                            -- redacted (no secrets)
  after        jsonb,
  event_id     uuid,                             -- linked CloudEvent if any
  prev_hash    bytea NOT NULL,                   -- hash of record seq-1 (32 zero bytes for seq 1)
  hash         bytea NOT NULL                    -- SHA-256(prev_hash || canonical_json(record without hash fields))
);
```

API response representation (JSON, follows contract conventions: `seq` is int64 so it is a JSON string):

```json
{
  "seq": "18234",
  "audit_id": "01933c02-...",
  "ts": "2026-11-07T14:03:22.123Z",
  "actor": "user:alice",
  "actor_role": "operator",
  "source_ip": "10.20.2.31",
  "channel": "rest",
  "request_id": "01933c02-...",
  "action": "tag.counter_adjusted",
  "target_kind": "tag",
  "target_id": "01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a",
  "target_path": "site1/pumphouse/meter01/total_volume",
  "outcome": "success",
  "reason": "Aligned to mechanical register after meter swap",
  "before": { "value": 1234630.0 },
  "after":  { "value": 1240000.0 },
  "event_id": "01933c02-...",
  "hash": "b64:Q2l..."
}
```

Canonical JSON for hashing: RFC 8785 JSON Canonicalization Scheme (JCS) (UNVERIFIED that we need full JCS; a simpler rule "Go `encoding/json` of a fixed struct with sorted map keys" is acceptable if documented and frozen, because only our code computes it).

Redaction: `before`/`after` never include secret values, password hashes, or tokens. For secrets only the name and version are recorded.

## 3. Tamper evidence

Threat: someone with DB access (T5/T7, or an insider with the PostgreSQL superuser password) edits or deletes audit rows to hide an action.

Layers:

1. **Append-only at the DB permission level.** `skyrangerd` writes with role `sr_audit_writer`, which has `INSERT` and `SELECT` only on `audit_log`; no `UPDATE`, `DELETE`, `TRUNCATE`. A trigger rejects `UPDATE`/`DELETE` even from roles that have table privileges (superuser can still disable triggers, hence layers 2 to 4).
2. **Hash chain.** Each record includes `prev_hash`; any edit or deletion in the middle breaks every later hash. Insertions are serialised (`pg_advisory_xact_lock`) so `seq` is gapless.
3. **Signed checkpoints.** Every hour (and at shutdown) `skyrangerd` writes an `audit.checkpoint` record whose `after` contains `{"seq": "N", "hash": "...", "signature": "ed25519:..."}`, signed with the audit signing key (06). Truncating the tail and rebuilding a consistent chain would require the signing key.
4. **Off-box copies.** Checkpoints (small) and optionally all records are forwarded:
   - syslog over TLS (RFC 5424 format, RFC 5425 transport, TCP 6514) to the customer's SIEM or log server, structured data containing `seq`, `action`, `actor`, `hash`;
   - included in every backup (10);
   - printed in the monthly report the integrator delivers (checkpoint hash of the last day of the month), which gives a paper anchor.
5. **Verification tool.** `skyrangerctl audit verify [--from SEQ]` recomputes the chain, checks signatures on checkpoints, compares against forwarded checkpoints if a syslog export is provided, and reports the first broken `seq`. Also exposed as `GET /api/v1/audit/verify` (admin, additive) and run daily; a failure raises a `critical` `_system` alarm and event `skyranger.security.audit_chain_broken.v1` (proposed).

What this does not do: prevent a root attacker from adding false records going forward after stealing the signing key, or from stopping the logger. It makes after-the-fact detection likely if off-box copies exist. This is the honest claim we make (principle 7).

## 4. Availability and retention

- **Fail closed for changes.** If the audit insert fails, the mutating request fails with `500 internal` and the change is not made (audit and change are in the same DB transaction where possible; for writes to devices, the `tag.write_requested` record is committed before the write is sent to the driver).
- **Fail open for monitoring.** Reads, live values, alarms, and history recording continue when audit storage has a problem.
- Retention: default `7y` for audit (it is small: roughly 1 KB per record); configurable minimum `1y`. Events follow history retention (tag-model.md `Gateway.spec.history.default_retention`). Audit retention cleanup removes whole chain prefixes only and records `audit.pruned` with the first remaining `seq` and its `prev_hash`, so the chain stays verifiable from that point.
- Disk alarms at 80% and 90% for the data volume (`_system/gateway/disk_used_pct`, proposed `_system` tag).

## 5. Security alarms

Generated by `skyrangerd` as `_system/security/...` tags with alarms (system tags are gateway-generated, read-only per tag-model.md):

| Tag (proposed) | Alarm when |
|---|---|
| `_system/security/login_failures_1h` | > 20 for one account or > 100 total in 1 h |
| `_system/security/audit_chain_ok` | false |
| `_system/security/device_flapping` | any device > 10 NBIRTH per hour (04) |
| `_system/security/remote_access_active` | true (informational, `low`) |
| `_system/security/cert_expiry_days` | < 30 for any service certificate |
| `_system/security/insecure_connections` | count of Connections with `allow_insecure: true` > 0 |

## Decisions made
1. Audit lives in PostgreSQL `audit_log`, append-only by DB privileges and trigger, with a SHA-256 hash chain and gapless `seq`.
2. Hourly Ed25519-signed checkpoints; off-box copies via syslog over TLS and backups.
3. Audit write failure blocks the change; monitoring never blocks on audit.
4. Default audit retention 7 years; pruning keeps the chain verifiable.
5. Security conditions surface as `_system/security/...` tags with alarms, so operators see them in the same alarm list.

## Alternatives rejected and why
- **Blockchain / external ledger**: adds dependency and cost; signed checkpoints plus off-box copies provide the same practical evidence.
- **Audit only in application log files**: easy to rotate or delete, hard to query.
- **WORM storage hardware**: good for regulated sites; can be the syslog target. Not required by the product.
- **Separate audit database**: more moving parts; privileges and hash chain give separation within one DB.
- **Auditing every read**: volume without value; audit sensitive reads only.

## Open questions
- Contract additions: `GET /api/v1/audit/verify`, `GET /api/v1/audit` filters (`action`, `actor`, `target_id`, `start`, `end`, `cursor`), event `skyranger.security.audit_chain_broken.v1`, `_system/security/...` and `_system/gateway/disk_used_pct` tags.
- JCS vs frozen Go struct encoding for the hash input (RFC 8785 details UNVERIFIED).
- Should `GET /audit` be available to `engineer` for their own actions?
- Does any target market (water utilities) have a regulatory audit retention requirement? UNVERIFIED, varies by jurisdiction.
