# API Contract (REST + WebSocket)

Status: v1 contract. Depends on [tag-model.md](tag-model.md), [messages.md](messages.md), [driver-interface.md](driver-interface.md), and [README.md](README.md) conventions (value object, timestamps, durations, 64-bit ints as strings).

`skyrangerd` serves the HMI static files, the REST API, and the WebSocket on one HTTPS port (default `8443`). The HMI uses only this public API, with no private endpoints, so anything the HMI can do, an integrator's script can do.

## 1. Basics

- Base path: `/api/v1`. Content type `application/json; charset=utf-8`. YAML is accepted and produced only on config endpoints (`application/yaml`).
- Field names `snake_case`. Timestamps RFC 3339 UTC ms. Durations as Go duration strings. `int64`/`uint64` values as JSON strings.
- TLS always on. The gateway generates a self-signed certificate on first boot if none is provided (security doc owns certificate management).
- A machine-readable **OpenAPI 3.1** document is served at `GET /api/v1/openapi.json` and is the authoritative detailed spec once written. It must not contradict this doc.
- Tag paths appear in URLs as a single **percent-encoded** path parameter (`site1%2Fpumphouse%2Fmeter01%2Fflow_rate`) or, preferably, tags are addressed by `tag_id`. Endpoints that take paths accept them in query strings or bodies, where `/` needs no escaping.

## 2. Authentication and authorization

```
Authorization: Bearer <token>
```

Two token kinds, both opaque to clients:
- **Session tokens** for humans: `POST /api/v1/auth/login` with `{"username", "password"}` returns `{"access_token", "expires_at", "user": {...}}`. Lifetime `12h` (configurable), revocable at `POST /api/v1/auth/logout`. The HMI keeps the token in memory and in `sessionStorage`. No cookies, which avoids CSRF.
- **API keys** for machines: created at `POST /api/v1/api-keys` (shown once), prefixed `sr_` (e.g. `sr_k1_9f2c...`), stored hashed (argon2id), with a role and optional expiry.

Tokens are opaque random strings checked against the database, so they can be revoked immediately. They are not JWTs.

Roles (fixed set in v1, cumulative):

| Role | Can |
|---|---|
| `viewer` | read tags, history, alarms, events, config |
| `operator` | + write writable tags, ack/shelve alarms |
| `engineer` | + edit/apply config, approve discovery, manage connections |
| `admin` | + users, API keys, system settings |

WebSocket auth: browsers cannot set headers on WebSocket, so the token is sent in the **first message** (`hello`, section 6). Tokens are never put in the URL (they would end up in logs).

Every mutating request is written to an audit log (`audit_log` table, `GET /api/v1/audit`) with user, time, action, target, and before/after values.

## 3. Resources

### Tags
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/tags` | viewer | List tag definitions. Query: `prefix`, `kind`, `q` (substring of path/display_name), `limit`, `cursor` |
| GET | `/tags/{tag_id}` | viewer | Definition (resolved, after template expansion) + `current` value |
| GET | `/tags/values` | viewer | Current values. Query: `prefix` or repeated `path=` or `id=` (max 1000). Returns `{"items": [value objects]}` |
| POST | `/tags/{tag_id}/write` | operator | Body `{"value": <json value>, "comment": "..."}`. Returns `202` `{"write_id", "status": "pending"}`; outcome via event `skyranger.tag.write_completed.v1` or `GET /writes/{write_id}`. Pass `?wait=10s` to block up to that long and get the final result (`200`) |
| GET | `/writes/{write_id}` | viewer | `{"write_id", "tag_id", "status": "pending"\|"ok"\|"error", "error"?}` |

Tag definition response:

```json
{
  "tag_id": "01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a",
  "path": "site1/pumphouse/meter01/total_volume",
  "display_name": "Total volume",
  "kind": "derived",
  "datatype": "float64",
  "unit": "L",
  "writable": true,
  "eng_range": null,
  "enum_labels": null,
  "template": { "name": "pulse_water_meter", "instance": "site1/pumphouse/meter01" },
  "alarms": [ { "alarm_id": "...", "name": "...", "severity": "high" } ],
  "current": { "value": 1234630.0, "quality": "good", "ts": "2026-11-07T10:22:00.412Z", "ts_gateway": "2026-11-07T10:22:00.412Z" },
  "config_revision": 43
}
```

### History
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/history` | viewer | Query params below |

Query: `id` (repeated, max 50) or `path` (repeated), `start`, `end` (RFC 3339; `end` default now), `agg` (`raw` default, `avg`, `min`, `max`, `first`, `last`, `count`, `delta`), `interval` (required when `agg != raw`, e.g. `1m`, `1h`), `limit` (raw only, default 10000, max 100000), `cursor`, `tz` (IANA, for `interval` >= `1d` bucket alignment; default `Gateway.spec.timezone`).

Response:

```json
{
  "series": [
    {
      "tag_id": "01933b7e-...",
      "path": "site1/pumphouse/meter01/total_volume",
      "unit": "L",
      "agg": "delta",
      "interval": "1h",
      "points": [
        { "ts": "2026-11-07T10:00:00.000Z", "value": 720.0, "quality": "good" },
        { "ts": "2026-11-07T11:00:00.000Z", "value": 0.0,   "quality": "good" }
      ]
    }
  ],
  "next_cursor": null
}
```

- Aggregates use only `good` and `uncertain` samples. A bucket's quality is the worst included quality; a bucket with no usable samples has `value: null, quality: "bad"`.
- `delta` = last minus first within the bucket, for monotonic totals (daily consumption). It is computed on `total_volume`, not on raw counts, so resets are already handled.
- Raw points include `ts_source` and `ts_gateway` when `?detail=true`.
- Export: `GET /history/export?...` same params, `Accept: text/csv` streams CSV with columns `ts,path,value,quality`.

### Alarms
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/alarms` | viewer | Current alarm states (messages.md section 5). Query: `state` (repeated), `prefix`, `severity`, `shelved` |
| GET | `/alarms/history` | viewer | State-change events. `start`, `end`, `prefix`, `cursor` |
| POST | `/alarms/{alarm_id}/ack` | operator | `{"comment": "..."}` |
| POST | `/alarms/ack` | operator | Bulk: `{"alarm_ids": [...], "comment": "..."}` |
| POST | `/alarms/{alarm_id}/shelve` | operator | `{"duration": "8h", "comment": "required"}`. Max duration `24h` (configurable) |
| POST | `/alarms/{alarm_id}/unshelve` | operator | |

### Events and audit
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/events` | viewer | CloudEvents (messages.md section 4). Query `type` (prefix match, repeated), `subject_prefix`, `start`, `end`, `cursor` |
| GET | `/audit` | admin | Audit records |

### Discovery
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/discovery/devices` | viewer | Discovered device objects (messages.md section 7). Query `status`, `connection` |
| GET | `/discovery/devices/{discovery_id}` | viewer | |
| POST | `/discovery/devices/{discovery_id}/approve` | engineer | Body below. Returns a draft (or a revision if `apply: true`) |
| POST | `/discovery/devices/{discovery_id}/ignore` | engineer | |
| POST | `/discovery/devices/{discovery_id}/unignore` | engineer | |

```json
{
  "tag_group": "pumphouse",
  "base_path": "site1/pumphouse",
  "instance_name": "meter01",
  "template": "pulse_water_meter",
  "params": { "initial_total_l": 1234567 },
  "apply": false,
  "comment": "Commission meter01"
}
```

Params that come from the discovered device (`connection`, `group_id`, `edge_node_id`) are filled in by the server when the template declares them and the request omits them. `tag_group` is created if it does not exist, with `base_path`.

### Configuration
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/config` | viewer | Current revision as YAML multi-doc (`Accept: application/yaml`) or JSON `{"revision", "documents": [...]}`. `?include_ids=true` default. This is the export |
| GET | `/config/revisions` | viewer | `{"items": [{"revision", "applied_at", "applied_by", "comment", "summary"}]}` |
| GET | `/config/revisions/{revision}` | viewer | Full content of a past revision |
| POST | `/config/drafts` | engineer | Body: full document set (YAML or JSON), or `{"base_revision", "patch": [...]}`. Validates and returns `{"draft_id", "base_revision", "valid", "errors": [...], "diff": {...}}` |
| GET | `/config/drafts/{draft_id}` | engineer | |
| POST | `/config/drafts/{draft_id}/apply` | engineer | `{"comment"}`. `409` if `base_revision` is no longer current (optimistic concurrency) |
| DELETE | `/config/drafts/{draft_id}` | engineer | |
| POST | `/config/revisions/{revision}/restore` | engineer | Creates a draft whose content equals that revision |

Diff shape: `{"added": [{"kind", "name", "path"?}], "changed": [{"kind", "name", "path"?, "fields": ["deadband", ...]}], "removed": [...]}`, at both document and resolved-tag level.

Validation errors use JSON Pointer into the submitted documents:

```json
{ "document": "TagGroup/pumphouse", "pointer": "/spec/instances/0/params/edge_node_id", "code": "required", "message": "param edge_node_id is required by template pulse_water_meter" }
```

Draft `patch` items use RFC 6902 JSON Patch against a single document: `{"document": "TagGroup/pumphouse", "ops": [{"op": "add", "path": "/spec/instances/-", "value": {...}}]}`.

### Connections and drivers
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/connections` | viewer | `{"items": [{"name", "driver", "enabled", "state", "health": {...}}]}` (health = driver HealthResponse in JSON) |
| GET | `/connections/{name}` | viewer | |
| POST | `/connections/{name}/browse` | engineer | `{"start": "", "page_size": 500, "page_token": ""}` → driver Browse; results also upserted into discovery |
| POST | `/connections/{name}/restart` | engineer | Restarts the driver process |
| GET | `/drivers` | viewer | Installed drivers: `Info` (name, version, capabilities, `config_schema`, `address_syntax`) |

### Users and keys (admin)
`GET/POST /users`, `GET/PATCH/DELETE /users/{user_id}`, `GET/POST /api-keys`, `DELETE /api-keys/{key_id}`.

### System
| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/system/health` | none | `{"status": "ok"\|"degraded"\|"down", "checks": {"database": "ok", "broker": "ok", "drivers": "degraded"}}`. Unauthenticated, for container health checks; returns no tag data |
| GET | `/system/info` | viewer | `{"gateway_id", "version", "config_revision", "started_at", "timezone"}` |
| GET | `/openapi.json` | none | |

## 4. Pagination

Cursor-based for all list endpoints: request `limit` (default 100, max 1000) and `cursor`; response `{"items": [...], "next_cursor": "opaque" | null}`. History uses `series` instead of `items` but the same `next_cursor`. Offsets are not supported.

## 5. Writes: value encoding

`value` in a write uses the tag's JSON wire encoding: `true`, `12.5`, `"42"` for int64/uint64 (a JSON number is also accepted for `int64`/`uint64` when it is an exact integer within ±2^53), `"text"`, `"2026-11-07T10:00:00.000Z"`. The engine applies inverse scaling, then `write_limits`, then sends the raw value to the driver.

## 6. WebSocket protocol

Endpoint: `wss://{host}:8443/api/v1/ws`, subprotocol **`skyranger.v1`** (`Sec-WebSocket-Protocol`). JSON text frames, one message per frame. Every message has `type`; client requests carry a client-chosen `id` that the server echoes in the reply.

### Client to server
| `type` | Fields | Reply |
|---|---|---|
| `hello` | `token`, `client` (`"hmi/0.1.0"`) | `welcome` or `error` and close `4401`. Must be the first message, within `10s` |
| `subscribe` | `id`, `sub_id` (client-chosen string), `target`: `"tags"`\|`"alarms"`\|`"events"`, `paths` (exact tag paths, max 5000) or `prefix`, `min_interval` (optional duration, per-tag rate limit, default `0s` meaning deadband only); for `events`, `types` (prefix list) | `ack` then a `snapshot` and then `update`s |
| `unsubscribe` | `id`, `sub_id` | `ack` |
| `ping` | `id` | `pong` |

Writes and acks are deliberately **not** WebSocket messages; use REST. This keeps one auth, validation, and audit path.

### Server to client
| `type` | Fields |
|---|---|
| `welcome` | `session_id`, `server_version`, `gateway_id`, `config_revision`, `user`: `{username, role}` |
| `ack` | `id`, `ok: true` |
| `error` | `id` (if in reply to a request), `error`: problem object (section 7) |
| `snapshot` | `sub_id`, `items`: full current state (value objects for `tags`, alarm state objects for `alarms`). Always sent first after subscribe and after any `resync` |
| `update` | `sub_id`, `seq` (uint, per subscription, starts at 1 after each snapshot), `items`: changed value objects / alarm states / CloudEvents |
| `resync` | `sub_id`, `reason`: `"overflow"`\|`"config_changed"`. Client must discard state for this sub; a fresh `snapshot` follows |
| `config_changed` | `revision`: tag set may have changed; clients re-resolve definitions. Subscriptions by `prefix` pick up new tags automatically (via `resync`) |
| `pong` | `id` |

Example:

```json
→ {"type":"hello","token":"sr_k1_9f2c...","client":"hmi/0.1.0"}
← {"type":"welcome","session_id":"01933b90-...","server_version":"0.1.0","gateway_id":"gw01","config_revision":43,"user":{"username":"alice","role":"operator"}}
→ {"type":"subscribe","id":"1","sub_id":"meter01","target":"tags","prefix":"site1/pumphouse/meter01","min_interval":"500ms"}
← {"type":"ack","id":"1","ok":true}
← {"type":"snapshot","sub_id":"meter01","items":[{"tag_id":"...","path":"site1/pumphouse/meter01/flow_rate","value":12.0,"quality":"good","ts":"2026-11-07T10:04:50.000Z","ts_gateway":"2026-11-07T10:04:50.000Z"}]}
← {"type":"update","sub_id":"meter01","seq":1,"items":[{"tag_id":"...","path":"site1/pumphouse/meter01/flow_rate","value":11.8,"quality":"good","ts":"2026-11-07T10:05:00.000Z","ts_gateway":"2026-11-07T10:05:00.000Z"}]}
```

Server behavior:
- Updates are batched: at most one `update` per subscription every `100ms`, containing the latest value per tag (latest-value-wins, same as drivers).
- **Backpressure:** each connection has a bounded send queue (default 1 MiB). On overflow, the server drops queued `update`s for the affected subscription and sends `resync` with `overflow`, then a snapshot. Clients never see a silent gap: `seq` is contiguous between snapshots, and a gap is a client bug or a reason to resubscribe. `events` subscriptions on overflow get `resync` and must backfill via `GET /events`.
- Server sends WebSocket ping frames every `30s` and closes after `60s` without pong.
- Token expiry or revocation closes the socket with `4401`. Permission loss sends `error` with `forbidden` and drops the affected subscriptions.
- Max subscriptions per connection: 100. Max connections per user: 20.

Close codes: `1000` normal, `1008` policy, `4400` protocol error (bad message), `4401` unauthenticated, `4403` forbidden, `4429` too many connections, `4500` server error.

## 7. Error format

All non-2xx responses use **RFC 9457 Problem Details** (`application/problem+json`), extended with a stable `code`:

```json
{
  "type": "/api/v1/problems/validation_failed",
  "title": "Validation failed",
  "status": 422,
  "detail": "2 errors in submitted configuration",
  "instance": "/api/v1/config/drafts",
  "code": "validation_failed",
  "request_id": "01933b91-...",
  "errors": [
    { "document": "TagGroup/pumphouse", "pointer": "/spec/instances/0/template", "code": "not_found", "message": "template pulse_meter does not exist" }
  ]
}
```

Clients branch on `code`, never on `title` or `detail`. Every response (success or error) has an `X-Request-Id` header equal to `request_id`, which is also in logs.

| HTTP | `code` | When |
|---|---|---|
| 400 | `bad_request` | malformed JSON/YAML, bad query param |
| 401 | `unauthenticated` | missing/invalid/expired token |
| 403 | `forbidden` | role insufficient |
| 404 | `not_found` | resource missing |
| 409 | `conflict` | stale `base_revision`, duplicate name |
| 409 | `not_writable` | tag not writable |
| 422 | `validation_failed` | config invalid (with `errors`) |
| 422 | `out_of_range` | write outside `write_limits` |
| 422 | `type_mismatch` | write value doesn't fit datatype |
| 429 | `rate_limited` | with `Retry-After` |
| 502 | `device_error` | driver reported device exception on write (`?wait`) |
| 503 | `not_connected` | connection down (write with `?wait`) |
| 504 | `timeout` | write outcome unknown (`?wait` elapsed or driver timeout) |
| 500 | `internal` | bug |

Driver `Code`s (driver-interface.md section 4) map to API codes by the same name where one exists, and to `internal` otherwise.

## 8. Versioning

- The major version is in the path (`/api/v1`) and the WS subprotocol (`skyranger.v1`). Within v1, changes are additive only: new endpoints, new optional request fields, new response fields, new enum values in `type`/`code`/`kind`-like fields. **Clients must ignore unknown fields and tolerate unknown enum values** (render as "unknown").
- Breaking changes ship as `/api/v2` alongside `/api/v1` for at least one minor release cycle (lifecycle policy owned by the deployment doc). Deprecated endpoints return `Deprecation` and `Sunset` headers (RFC 8594 for Sunset; the `Deprecation` header RFC number is UNVERIFIED).
- `GET /system/info` `version` is the gateway semver; the HMI checks compatibility on `welcome`.

## Decisions made
1. One public API for the HMI and third parties. REST for request/response and all mutations; WebSocket only for subscriptions.
2. Opaque, database-backed bearer tokens (sessions and `sr_` API keys), four cumulative roles, every mutation audited. WS auth via the first message, never the URL.
3. RFC 9457 problem details with a stable `code` field and JSON Pointer validation errors.
4. WebSocket: `snapshot` then contiguous-`seq` `update`s, 100 ms batching, latest-value-wins, explicit `resync` on overflow, no silent gaps.
5. Config is edited as drafts with diffs and applied with optimistic concurrency on `base_revision`. `GET /config` is the lossless export.
6. Writes are asynchronous by default (`202` + `write_id`), with an optional `?wait`.
7. Cursor pagination everywhere; tags addressed by `tag_id` in URL paths.

## Alternatives rejected and why
- **GraphQL**: flexible, but more machinery, harder caching and auth reasoning, and less familiar to SCADA integrators than REST + OpenAPI.
- **JWTs**: cannot be revoked instantly without a denylist, which removes their benefit for a single gateway with a local DB.
- **Cookie sessions**: need CSRF protection and complicate non-browser clients.
- **Writes over WebSocket**: two audit/validation paths.
- **Server-Sent Events instead of WebSocket**: one-way; subscription changes would need extra REST calls bound to the stream. Workable, but WS is the norm for HMIs.
- **OPC UA as the HMI API**: heavyweight in browsers. OPC UA *server* northbound for third-party clients is a separate, later feature.
- **Offset pagination**: unstable under inserts (events, history).

## Open questions
- SSO (OIDC/LDAP) for customers with an AD: likely needed, deferred to the security doc.
- Is a per-tag permission model (areas) needed before multi-customer use, or are the four global roles enough for the integrator phase?
- Should `/history` support server-side interpolation (`agg=interpolated`) for trend charts, or is it left to the client?
- Rate limits values (per key, per IP) are not set.
- The `Deprecation` header standard reference is UNVERIFIED.
