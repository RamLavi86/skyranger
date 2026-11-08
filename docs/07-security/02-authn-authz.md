# 02 Authentication and Authorization

Binding contract: [api.md section 2](../00-contracts/api.md). This file implements it and does not change it: `Authorization: Bearer <token>`, opaque DB-backed tokens (not JWTs), no cookies, session lifetime `12h` configurable, `sr_` API keys stored with argon2id, four cumulative roles `viewer` < `operator` < `engineer` < `admin`, WebSocket auth by first `hello` message, every mutation audited. Additions proposed here are additive endpoints or fields and are listed in Open questions for contract review.

## 1. Principals

| Principal kind | `kind` value | Authenticates with | Example |
|---|---|---|---|
| Human, local | `user` (source `local`) | username + password (+ TOTP if enrolled) | `alice` |
| Human, federated | `user` (source `oidc` / `ldap`) | external IdP, then a local session token | `bob@example.com` |
| Service account | `service` | API key only, cannot log in interactively | `svc-report-export`, `svc-cloud-sync`, `svc-python-runtime` |
| Internal component | not a DB principal | OS-level identity: UID, Unix socket peer credentials, TLS client cert | `skyrangerd`, driver processes, Mosquitto |

In audit records and events, principals are written `user:alice`, `service:svc-report-export`, `system:skyrangerd` (the `requested_by: "user:alice"` form in messages.md).

API keys always belong to a principal (a user or a service account) and carry a role no higher than that principal's role. Recommended practice: machines use service accounts, not a human's API key, so that a person leaving does not break an integration and vice versa.

## 2. Roles and permissions

The contract fixes four cumulative roles. Internally, each route requires a **permission**, and each role is a fixed set of permissions. This costs nothing now and lets per-area scoping (Open questions) be added without touching route code.

| Permission | viewer | operator | engineer | admin | Routes (api.md section 3) |
|---|---|---|---|---|---|
| `tags.read` | x | x | x | x | `GET /tags*`, `GET /writes/*`, `GET /history*` |
| `alarms.read` | x | x | x | x | `GET /alarms*` |
| `events.read` | x | x | x | x | `GET /events` |
| `config.read` | x | x | x | x | `GET /config*`, `GET /connections*`, `GET /drivers`, `GET /discovery/*`, `GET /system/info` |
| `tags.write` | | x | x | x | `POST /tags/{id}/write` |
| `alarms.ack` | | x | x | x | `POST /alarms/*/ack`, `POST /alarms/ack` |
| `alarms.shelve` | | x | x | x | `POST /alarms/*/shelve`, `unshelve` |
| `config.edit` | | | x | x | `POST/GET/DELETE /config/drafts*`, `restore` |
| `config.apply` | | | x | x | `POST /config/drafts/*/apply` |
| `discovery.manage` | | | x | x | `approve`, `ignore`, `unignore` |
| `connections.manage` | | | x | x | `browse`, `restart` |
| `audit.read` | | | | x | `GET /audit` |
| `users.manage` | | | | x | `/users*` |
| `apikeys.manage` | | | | x | `/api-keys*` (create, list, revoke any) |
| `system.manage` | | | | x | security settings, TLS certificate upload, backup, remote-access toggle |

Notes:
- `discovery.approve` with `"apply": true` requires both `discovery.manage` and `config.apply`.
- Writing a `counter_total` tag (meter register alignment) is `tags.write` per contract. Because it changes billing totals, the HMI requires a non-empty `comment` and shows a confirmation; the audit record captures before/after. Whether it should require `engineer` is an open question (also raised in tag-model.md).
- `GET /system/health` and `GET /openapi.json` are unauthenticated per contract. Health returns no tag data. OpenAPI discloses the API shape only, which is public anyway.
- Users cannot change their own role. An admin cannot delete or demote the last enabled local admin (prevents lockout).

### Enforcement rules (normative)
1. **Deny by default.** The router refuses to start if any route has no permission declared (unit test plus startup check).
2. One middleware resolves token to principal to role to permission set, then checks the route permission. Handlers never inspect roles directly.
3. A CI test enumerates every route in the OpenAPI document and calls it with a token of each role, asserting `403 forbidden` exactly where the table says. This test is a ship gate (14).
4. Permission changes take effect on the next request; WebSocket subscriptions are re-checked and dropped with `error forbidden` as api.md section 6 requires.

## 3. Local users and passwords

Data model (PostgreSQL, schema owned by the core; security fields shown):

```sql
CREATE TABLE users (
  user_id         uuid PRIMARY KEY,              -- UUIDv7
  kind            text NOT NULL CHECK (kind IN ('user','service')),
  username        text NOT NULL UNIQUE,          -- segment grammar for local/service; IdP subject mapped separately
  display_name    text,
  role            text NOT NULL CHECK (role IN ('viewer','operator','engineer','admin')),
  source          text NOT NULL DEFAULT 'local' CHECK (source IN ('local','oidc','ldap')),
  password_hash   text,                          -- PHC string, argon2id; NULL for service and federated
  password_set_at timestamptz,
  must_change_password boolean NOT NULL DEFAULT false,
  totp_secret_ref text,                          -- secret://users/{user_id}/totp, never the secret itself
  enabled         boolean NOT NULL DEFAULT true,
  failed_logins   integer NOT NULL DEFAULT 0,
  last_failed_at  timestamptz,
  last_login_at   timestamptz,
  created_at      timestamptz NOT NULL,
  created_by      text NOT NULL
);
```

Password hashing: argon2id via `golang.org/x/crypto/argon2`, stored in PHC string format. Parameters sized for a small ARM64 box (target 250 to 500 ms per hash):

```yaml
# skyrangerd security settings (system settings, not tag config)
auth:
  password:
    argon2id: { memory_kib: 65536, iterations: 3, parallelism: 2, salt_bytes: 16, key_bytes: 32 }
    min_length: 12
    max_length: 128
    check_common_list: true      # reject top-N breached passwords (offline list shipped in image)
    expire_after: 0s             # no forced periodic rotation; change on suspicion of compromise
  login_throttle:
    per_account: { free_attempts: 5, delay_start: 1s, delay_max: 5m }   # exponential, never permanent lockout
    per_source_ip: { burst: 20, per: 1m }
  session:
    lifetime: 12h                # contract default
    idle_timeout: 30m            # 0s disables; kiosks use a dedicated low-privilege account instead
    max_per_user: 20             # matches WS connection limit
```

Password rules follow the current NIST SP 800-63B direction as we understand it (length over complexity rules, block known-breached passwords, no periodic forced rotation). The exact SP 800-63B requirement text is UNVERIFIED and must be checked before we claim alignment to a customer.

Why no permanent lockout: in a plant, locking the only operator account during an alarm flood is a safety and availability failure. An exponential delay makes online guessing impractical (5 free attempts, then 1 s, 2 s, ... capped at 5 min per attempt) while a legitimate user can always get in eventually. Every failed attempt is audited; more than 20 failures per account per hour raises a `_system/security/...` alarm (see 08).

Login responses never reveal whether the username exists: same `401 unauthenticated` and comparable timing (hash a dummy password when the user is unknown).

### First boot: no default credentials
1. On first start with an empty `users` table, `skyrangerd` generates a 12-character one-time setup code, prints it to the container log and to the console (`journalctl -u skyranger`), and writes it to `/var/lib/skyranger/setup-code` (mode `0600`, root).
2. The HMI shows a setup page only while no admin exists. It requires the setup code, a new admin username, and password.
3. On success the code is deleted, the setup endpoint returns `404` forever, and an audit record `auth.setup_completed` is written.
4. The setup endpoint listens only for 24 h after first boot; after that the code must be regenerated from the console (`skyrangerctl setup-code --regenerate`), which proves local access.

### Break-glass recovery
`skyrangerctl user reset-password <username>` run on the edge box console (requires root on the host) sets a temporary password with `must_change_password = true` and writes an audit record `auth.break_glass` with `system:console` as actor. This is the only recovery path; there is no remote "forgot password" flow (no email dependency at the edge).

## 4. Tokens

### Session tokens (humans)
- Format: `sr_s1_` + 32 random bytes base64url (no padding). The prefix lets secret scanners and log scrubbers recognise them. 256 bits of entropy.
- Storage: `sessions` table keyed by `SHA-256(token)`. A fast hash is correct here because the token is high-entropy random; argon2id would add login-path latency for no security gain.
- Fields: `session_id`, `user_id`, `token_hash`, `created_at`, `expires_at` (absolute, `12h`), `last_seen_at` (idle check, updated at most once per minute to limit writes), `source_ip`, `user_agent`, `mfa` (bool), `revoked_at`.
- Revocation: `POST /auth/logout` (own session), and admin actions: disabling a user, changing a user's role or password revokes all their sessions immediately. Open WebSockets on a revoked session close with `4401` (contract).
- Client storage: in memory and `sessionStorage` (contract). With a strict CSP (09) and no third-party scripts, the XSS exposure of `sessionStorage` is acceptable; cookies were rejected by the contract to avoid CSRF.

### API keys (machines)
- Format (contract prefix `sr_`): `sr_k1_{key_id}_{secret}` where `key_id` is 8 base32 chars and `secret` is 32 random bytes base64url. Example: `sr_k1_7q2m9xkd_Jf3...`.
- Storage: `api_keys(key_id, principal_id, role, name, argon2id_hash, created_at, created_by, expires_at, last_used_at, last_used_ip, revoked_at, allowed_cidrs)`.
- Verification: look up by `key_id`, verify `secret` with argon2id (contract). To keep per-request cost bounded, a verified key is cached in memory for `60s` keyed by `SHA-256(full key)`; revocation clears the cache entry immediately.
- Shown once at creation. Default expiry `365d`; `expires_at: null` allowed only for service accounts and flagged in the security report (14).
- Optional `allowed_cidrs` (e.g. `["10.20.2.15/32"]`) restricts where a key may be used. Recommended for `svc-cloud-sync` and for report servers.
- Role is fixed at creation. To change it, create a new key and revoke the old.

### WebSocket
Per contract: token in the first `hello` message within `10s`, never in the URL. The server additionally checks the `Origin` header on the upgrade request against the gateway's configured HMI origins and rejects others with HTTP `403` before the upgrade, which blocks cross-site WebSocket hijacking from a browser on the plant LAN.

## 5. MFA position

| Phase | Position |
|---|---|
| v1 (milestone) | No MFA. Local accounts on an isolated LAN; remote access disabled. Documented as a known gap. |
| v1.1 | TOTP (RFC 6238) optional for every local user, **required** for `admin` and for any session originating through the remote-access path (05). Recovery codes: 10 single-use codes, argon2id-hashed. |
| Enterprise | When OIDC is used, MFA is the IdP's job; the gateway records the `amr`/`acr` claim if present (UNVERIFIED claim semantics per IdP) and can require it for `admin`. |
| Later | WebAuthn/passkeys for engineer and admin. Requires HTTPS with a stable origin; feasible once the site CA is trusted by browsers (03). |

Not forced on operators at a local HMI: a shared control-room screen with gloves and a phone ban makes TOTP impractical, and the control is better placed in physical access and network segmentation. The customer can turn it on.

Proposed additive login flow when TOTP is enrolled: `POST /auth/login` returns `401` with `code: "mfa_required"` and a short-lived `mfa_ticket` (5 min, single use); the client then calls `POST /auth/mfa` with `{"mfa_ticket", "totp"}` and receives the normal login response. Existing clients without MFA users are unaffected.

## 6. Enterprise identity: OIDC and LDAP

Design rule: **the IdP authenticates, the gateway authorizes and issues its own opaque session token.** All downstream behaviour (revocation, WS hello, audit) stays exactly as the contract defines. If the IdP is unreachable, local accounts still work (principle 1).

### OIDC (preferred, v1.2)
Authorization code flow with PKCE, gateway as a confidential client. Tested targets: Microsoft Entra ID, Keycloak, Authentik.

Sequence (no tokens in URLs, no cookies):
1. HMI calls `POST /api/v1/auth/oidc/start` → `{ "authorize_url" }`. Server stores `state`, `nonce`, PKCE verifier server-side for `10m`.
2. Browser navigates to the IdP, user signs in (and does MFA at the IdP).
3. IdP redirects to `https://{gateway}/api/v1/auth/oidc/callback?code=...&state=...`. Server validates `state`, exchanges the code, validates the ID token (issuer, audience, signature against cached JWKS, `nonce`, expiry).
4. Server maps claims to a user (create on first login if the group mapping grants a role), then redirects to `/hmi/#login_code={one-time code}`. The one-time code is random, valid for `60s`, single use, and is **not** a session token. A URL fragment is not sent to servers or written to access logs.
5. HMI calls `POST /api/v1/auth/oidc/exchange {"login_code"}` and receives the standard `{"access_token", "expires_at", "user"}`.

Role mapping config:

```yaml
auth:
  oidc:
    enabled: true
    issuer: https://login.example.com/realms/plant
    client_id: skyranger-gw01
    client_secret: secret://auth/oidc_client_secret
    redirect_url: https://gw01.plant.example.local:8443/api/v1/auth/oidc/callback
    scopes: [openid, profile, email, groups]
    username_claim: preferred_username
    groups_claim: groups
    role_mapping:                    # highest matching role wins; no match = login refused
      - { group: "SCADA-Admins",    role: admin }
      - { group: "SCADA-Engineers", role: engineer }
      - { group: "SCADA-Operators", role: operator }
      - { group: "SCADA-Viewers",   role: viewer }
    resync_role_every: 1h            # re-evaluate on session refresh; role loss revokes sessions
```

Federated users get a local `users` row (`source: oidc`, no password). Their role is recomputed at each login; an admin cannot raise a federated user's role locally (prevents drift from the IdP).

### LDAP / Active Directory (later, only if a customer requires it)
LDAPS only (port 636) or StartTLS with certificate validation; bind with a service account (`secret://auth/ldap_bind`); search user by `sAMAccountName`; re-bind as the user to verify the password; map `memberOf` groups to roles as above. LDAP is lower priority than OIDC because modern AD deployments can expose OIDC via Entra ID or ADFS, and LDAP puts user passwords through the gateway.

## 7. Service accounts

| Service account | Role | Used by | Notes |
|---|---|---|---|
| `svc-python-runtime` | `operator` (default) | Python scripting runtime (writes go through REST, messages.md section 3) | Scoped down per script once area permissions exist |
| `svc-cloud-sync` | `viewer` | Outbound cloud sync agent | Read-only by design (01, 05) |
| `svc-report-*` | `viewer` | Report generators, BI tools | `allowed_cidrs` recommended |
| `svc-test-harness` | as needed | Simulator and integration tests | Never present on customer systems (14) |

Service accounts cannot log in with a password, cannot hold TOTP, and cannot be `admin`.

## 8. Rate limits (proposed values for api.md Open question)

| Scope | Limit | Response |
|---|---|---|
| `POST /auth/login` per source IP | 20 per minute | `429 rate_limited` with `Retry-After` |
| Authenticated requests per token | 50 per second, burst 200 | `429` |
| `POST /tags/{id}/write` per principal | 20 per second | `429` |
| `GET /history*` concurrent per principal | 4 | `429` |
| Unauthenticated, per source IP (all routes) | 10 per second | `429` |

Implemented as in-memory token buckets in `skyrangerd` (single process, so no shared store needed).

## Decisions made
1. Contract roles are implemented as fixed permission sets; routes declare permissions; deny by default; a CI test checks every route against every role.
2. Session tokens `sr_s1_...` hashed with SHA-256; API keys `sr_k1_{key_id}_{secret}` verified with argon2id and cached `60s`.
3. No default credentials: first-boot setup code proves local access; break-glass is console-only.
4. Throttle logins with exponential delay; never permanently lock accounts.
5. Idle timeout `30m` in addition to the contract's `12h` absolute lifetime.
6. MFA: none in v1, TOTP in v1.1 (required for admin and remote sessions), IdP MFA for OIDC.
7. OIDC authenticates, the gateway issues its own opaque token; local login always remains available.
8. WebSocket upgrades check `Origin`.

## Alternatives rejected and why
- **JWT sessions**: rejected by the contract; would also need a denylist for instant revocation.
- **Cookies with CSRF tokens**: rejected by the contract; complicates non-browser clients.
- **Permanent account lockout after N failures**: an attacker could lock out operators during an incident.
- **Fully custom per-tag ACLs in v1**: high complexity, high bug risk, no customer asking yet.
- **SAML**: heavier and less pleasant to implement than OIDC; most IdPs that support SAML also support OIDC.
- **Argon2id for session tokens**: no benefit for 256-bit random tokens; adds latency.
- **Email-based password reset**: edge boxes often have no mail relay; adds an internet dependency.

## Open questions
- **Contract additions (additive, need api.md update):** `POST /auth/mfa`, `code: "mfa_required"`, `POST /auth/oidc/start`, `GET /auth/oidc/callback`, `POST /auth/oidc/exchange`, `POST /auth/setup`, `allowed_cidrs` on API keys, `Origin` checking on WS.
- The contract mandates argon2id for API keys. For 256-bit random secrets SHA-256 would be sufficient and cheaper; the 60 s cache makes argon2id workable. Keep or relax?
- Area-scoped permissions (e.g. operator only for `site1/pumphouse/...`): add a `scopes: [prefix]` field on users and keys as an additive change when the first multi-area customer needs it.
- Should writing a `counter_total` (billing total) require `engineer` rather than `operator`?
- Should `idle_timeout` be per role (operators on a control-room screen may want longer)?
- NIST SP 800-63B specifics and OIDC `amr`/`acr` claim values are UNVERIFIED.
