# 08 API Extensions for the HMI

Depends on: [api.md](../00-contracts/api.md) (binding), [README.md](../00-contracts/README.md) conventions, [02-screen-format.md](02-screen-format.md), [01-architecture.md](01-architecture.md).

Everything in this file is an **additive contract proposal**. Nothing here renames or removes anything in api.md. Until the api.md owner accepts a proposal, the HMI must degrade as stated in each section ("Fallback"). All new endpoints follow api.md conventions: `snake_case`, RFC 3339 UTC ms timestamps, Go duration strings, cursor pagination, RFC 9457 problems with a stable `code`, `X-Request-Id`, and every mutation audited ([../07-security/08-audit-logging.md](../07-security/08-audit-logging.md)). Clients must ignore unknown fields (api.md section 8).

## 1. HMI document store (`/api/v1/hmi`)

Stores `Screen`, `Navigation`, `Symbol` documents (02) separately from tag config, with the same draft, diff, apply and revision pattern as `/config`.

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/hmi` | viewer | Current revision. `Accept: application/yaml`: multi-doc export with `tag_ids` filled; JSON: `{"revision", "documents": [...]}`. Documents the caller may not view (`access.view`) are omitted for roles below `engineer`; `engineer` and above get all (the export) |
| GET | `/hmi/screens` | viewer | `{"items": [{"name", "display_name", "labels", "access"}], "next_cursor"}`; filtered by role |
| GET | `/hmi/screens/{name}` | viewer | One document. `?resolve=true` adds `{"renamed": [{"old_path","new_path","tag_id"}], "dangling": [path...]}` (02 section 3.3). `ETag` = revision, `If-None-Match` supported (`304`) |
| GET | `/hmi/navigation/{name}` | viewer | Default `main`; `home` and `menu` already filtered to the caller's role |
| GET | `/hmi/symbols/{name}` | viewer | Sanitized SVG document |
| GET | `/hmi/schema` | viewer | JSON Schema 2020-12 (and `?draft=07` rendition) for editors |
| GET | `/hmi/usages` | viewer | `?tag_id=` returns `{"items": [{"kind", "name", "widget_id", "slot"}]}`: which screens bind this tag |
| GET | `/hmi/revisions`, `/hmi/revisions/{rev}` | viewer | Same shape as `/config/revisions` |
| POST | `/hmi/drafts` | engineer | Full document set or `{"base_revision","patch":[...]}` (RFC 6902, one document per patch item, as `/config/drafts`). Returns `{"draft_id","base_revision","valid","errors":[],"warnings":[],"diff":{}}`. Dangling bindings are `warnings`, not `errors` (02 section 3.3) |
| GET | `/hmi/drafts/{draft_id}` | engineer | |
| POST | `/hmi/drafts/{draft_id}/apply` | engineer | `{"comment"}` required. `409 conflict` when `base_revision` is stale |
| DELETE | `/hmi/drafts/{draft_id}` | engineer | |
| POST | `/hmi/revisions/{rev}/restore` | engineer | Creates a draft equal to that revision |
| POST | `/hmi/generate` | engineer | Scaffold a draft from a template recipe (04 section 4). Never applies |
| PUT | `/hmi/assets/{name}` | engineer | Static image for a screen background; PNG, JPEG, WebP or sanitized SVG, max 2 MiB. Content-Type sniffed server-side, not trusted. Part of the export (base64 documents in the YAML stream, `kind: Asset`) |

Rules:
- **Validation** (server, same JSON Schema as the HMI): schema, unique `metadata.name` per kind, unique widget `id` per screen, `${param}` resolution, symbol SVG allowlist (02 section 6), `control` widgets referencing non-writable tags (`not_writable` warning), referenced screens exist (`on_click`, `Navigation`), referenced symbols exist. Error pointers are RFC 6901 JSON Pointers into the submitted documents with the same shape as `/config/drafts` errors.
- **Apply never reloads the tag engine.** It bumps `hmi_revision` only.
- **Audit:** `hmi.draft_created`, `hmi.applied`, `hmi.restored` with before/after revision numbers and the diff summary. Document bodies are not stored in audit records (they are in the revision table).
- **Limits:** one document max 256 KiB; a revision max 8 MiB including assets; 200 revisions retained by default (configurable), older ones pruned except those named in a release.
- **Fallback if rejected:** HMI documents stored as a single `memory` tag of `string` datatype per document is not workable (4096 byte cap). The fallback is a read-only built-in screen set shipped with the HMI and engineer-edited files loaded from a gateway directory; this loses revisions and audit, so the proposal should be accepted.

### 1.1 Export bundle

`GET /hmi` YAML stream order: `Navigation`, `Symbol`s (sorted by name), `Screen`s (sorted by name), `Asset`s. Key order stable (02 section 7). Import is `POST /hmi/drafts` with the same stream; export then import yields an empty diff.

## 2. Bulk tag resolution

Problem: a screen with 400 bindings would otherwise need 400 `GET /tags/{id}` calls to learn units, ranges and writability, and api.md addresses definitions by `tag_id`, not path.

```
POST /api/v1/tags/resolve        role: viewer
{ "paths": ["site1/pumphouse/meter01/flow_rate", "..."], "ids": ["01933b7e-..."] }     // max 2000 combined
```

Response `200`:

```json
{
  "items": [
    { "path": "site1/pumphouse/meter01/flow_rate", "found": true,  "tag": { "tag_id": "...", "path": "...", "display_name": "Flow rate", "kind": "derived", "datatype": "float64", "unit": "L/min", "writable": false, "eng_range": {"low": 0, "high": 200}, "enum_labels": null, "alarms": [{"alarm_id": "...", "name": "...", "severity": "high"}], "history": {"enabled": true, "mode": "on_change", "max_interval": "15m"}, "config_revision": 43 } },
    { "path": "site1/pumphouse/meter01/gone",       "found": false }
  ],
  "config_revision": 43
}
```

- `tag` is the `GET /tags/{tag_id}` definition **without** `current` (live values come from the WebSocket snapshot).
- Unknown inputs return `found: false`, never `404`, so one stale binding does not fail the batch.
- `POST` rather than `GET` only because of the body size; it is read-only and needs no audit record.
- The HMI caches results per `config_revision` and invalidates on `config_changed`.
- **Fallback:** `GET /tags?prefix=...&limit=1000` per screen prefix, paging through definitions. Works but is chatty for scattered bindings.

## 3. Server time

Operators correlate events with plant time, and a PC or tablet clock can be minutes off (06 section 3).

- `welcome.server_time` (RFC 3339 UTC ms) and `pong.server_time`, additive fields on existing messages.
- HMI computes `offset = server_time - (local_now_at_receive - rtt/2)` on every `pong`, keeps the median of the last 5, and uses `Date.now() + offset` for the header clock, "age" displays, `last_updated`, and staleness arithmetic. It never mutates the browser clock.
- **Fallback:** the HTTP `Date` header of any response (1 s resolution) gives a coarser offset.

## 4. `hmi_changed` notification

```json
{"type": "hmi_changed", "revision": 13}
```

- `welcome` gains `hmi_revision`. After an HMI apply, the server sends `hmi_changed` to every connection.
- Behavior: viewers' open screens re-fetch their screen document (`ETag` makes it cheap) and re-render **only if the screen's content changed**; the swap happens at the next navigation or after 30 s of operator inactivity, whichever comes first, so a screen never rearranges under a hand mid-action. Kiosks apply immediately at the next rotation step. An open `control` confirm dialog is never replaced.
- A `hmi_revision` mismatch seen on reconnect has the same effect.
- **Fallback:** poll `GET /hmi` HEAD (`ETag`) every 60 s.

## 5. User preferences

Small per-user key/value store, so settings follow the person to any station (kiosks and service accounts have none).

| Method | Path | Role | Notes |
|---|---|---|---|
| GET | `/users/me/preferences` | any authenticated | `{"values": {"hmi.language": "he", ...}, "updated_at"}` |
| PUT | `/users/me/preferences` | any authenticated | Replace whole object. Max 64 KiB, max 100 keys. Unknown keys accepted; clients ignore what they do not know |

Defined keys (v1):

| Key | Type | Default | Used by |
|---|---|---|---|
| `hmi.language` | BCP 47 string | gateway default | 07 |
| `hmi.clock` | `"24h"` \| `"12h"` | `24h` | 07 |
| `hmi.idle_lock` | duration | `15m`, `0s` disables, max `8h` | 05 section 6 |
| `hmi.density` | `"comfortable"` \| `"compact"` | `comfortable` | 06 |
| `hmi.saved_trends` | array of `{name, range, pens: [{bind, y_axis, style}]}`, max 50 | `[]` | 03 section 5.2 |
| `hmi.alarm_sound` | bool | `false` | 06 |

Preferences are not security-relevant and carry no secrets; the server rejects values containing `sr_` token patterns. `idle_lock` is a convenience: the server's session idle timeout ([../07-security/02-authn-authz.md](../07-security/02-authn-authz.md)) is separate and authoritative. Changes are audited at a low level (`user.preferences_changed`, keys only).

**Fallback:** `localStorage` per browser (per-viewer convenience only).

## 6. Subscribing to all alarms

api.md section 6 allows `subscribe` with `target: "alarms"` and `paths` or `prefix`. The alarm banner needs "every alarm, always".

- **Proposal:** for `target: "alarms"` and `"events"`, an omitted `prefix` or `prefix: ""` means "everything the caller may see". For `target: "tags"` an empty prefix remains `bad_request` (it would be an unbounded tag subscription).
- The `snapshot` for alarms contains all alarm state objects not in `normal` plus shelved ones, not every configured alarm (an alarm in `normal` state is not interesting to a banner). Transitions to `normal` arrive as an `update` item with `state: "normal"` so clients can drop them. **Needs api.md owner to confirm**, because messages.md section 5 defines the state object and may want `normal` rows in the snapshot.
- **Fallback:** subscribe with `prefix` equal to the site root (`site1`), which works as long as all alarms live under one root; `_system` alarms need a second subscription.

## 7. Trend and history details

Additions that make dense trends cheap (03 section 5.2).

1. **`history` block in tag definitions** (`GET /tags/{id}`, `POST /tags/resolve`): `{"enabled", "mode", "interval"?, "max_interval"}`, copied from the resolved tag config. The trend uses `max_interval * 1.5` as the gap threshold.
2. **`agg=minmax`:** returns two values per bucket, `{"ts", "min", "max", "quality"}`, one request instead of two for the envelope. Optional; the HMI falls back to separate `min` and `max` requests. Aligns buckets identically for both so the envelope is consistent.
3. **`points=N`:** optional alternative to `interval`: the server picks `interval = ceil((end - start) / N)` rounded to a "nice" duration and echoes the chosen `interval` in the response. Removes the client-side density arithmetic and keeps all clients consistent. Max `N` 5000.
4. **Concurrency hints:** api.md leaves history concurrency to rate limits; security proposes 4 concurrent per principal ([../07-security/02-authn-authz.md](../07-security/02-authn-authz.md) section 8). A trend with 8 pens must therefore send **one** request with 8 `id`s (api.md allows 50), not 8 requests. The HMI enforces this.
5. **Interpolation** stays client-side; `agg=interpolated` is not proposed.

## 8. WebSocket usage details beyond api.md

These are implementation commitments for the HMI and constraints on the server, all inside the existing protocol.

| Topic | Detail |
|---|---|
| Subscription shape | One `paths` subscription per screen (max 5000 per api.md). A screen above that is a design error flagged by the validator; split with `tabs` (inactive tabs do not subscribe) |
| Rate limiting | Default `min_interval` `0s` for operator screens. Overview, kiosk and mobile screens use `1s`. A `trend` pen uses `500ms` while live. Server batching already caps at one `update` per 100 ms per subscription |
| Compression | `permessage-deflate` is negotiated by the browser and the server may enable it. Measure first: 100 ms batches of small JSON compress well, but CPU on a small edge box may matter more than LAN bandwidth. Default off; revisit with measurements (Open questions) |
| Payload size | A value object is about 200 to 250 bytes of JSON. 500 changes per second is about 125 KB/s per client. With 20 clients that is 2.5 MB/s, trivial on a LAN. The 1 MiB send queue holds roughly 4,000 pending items |
| `resync` storms | HMI raises `min_interval` to `1s` for that subscription after 3 resyncs per minute (06 section 5.4) |
| Backfill after `resync` for events | `GET /events?start={last_seen}`; de-duplicate by CloudEvent `id` |
| Close code 4429 | Too many connections: back off at least 30 s (01 section 5.1). Kiosks should use their own identities so a control room does not hit the 20-per-user cap |
| Origin | The server checks `Origin` on upgrade ([../07-security/02-authn-authz.md](../07-security/02-authn-authz.md) section 4). The HMI is same-origin, so this is transparent. Third-party browser apps on other origins need an allow-list entry (system setting) |
| Protocol version | `welcome.server_version` major mismatch: "reload required" banner (01 section 5.2) |

## 9. Problem codes added by these endpoints

All use the api.md problem format. New `code` values (additive): `screen_invalid` (422, in `/hmi/drafts`, with `errors`), `asset_too_large` (413), `asset_type_unsupported` (415), `too_many_items` (422, resolve over limit), `preferences_too_large` (413). Existing codes are reused where they fit (`conflict` for stale `base_revision`, `not_found`, `forbidden`).

## 10. Write step-up (proposal from 05 and security)

05 section 6 approximates "re-authenticate for critical writes" by a client-side fresh login. A server-enforced version:

- `POST /auth/reauth` with `{"password"}` (and `totp` when enrolled) returns `{"reauth_token", "expires_at"}`, valid `60s`, single use, bound to the session. Rate limited like login.
- `POST /tags/{tag_id}/write` accepts header `X-Reauth-Token`. A tag may declare `write_requires_reauth: true` (additive tag config field; owner: tag model). Without a valid token the server returns `401` with `code: "reauth_required"`.
- Not used for ordinary operator writes. Reserved for totalizer alignment and other tags the engineer marks.

This keeps the password check on the server and the audit trail whole (`auth.reauth` record). It is a proposal for security and api.md owners; the HMI ships the client-side approximation until accepted.

## Decisions made
1. HMI documents live in a separate revisioned store under `/api/v1/hmi` with the same draft/diff/apply/export mechanics as `/config`; applying never touches the tag engine.
2. Bulk definition lookup is one read-only `POST /tags/resolve` with `found: false` entries instead of per-tag `404`s.
3. Server time rides on `welcome` and `pong`; the HMI keeps a clock offset and never trusts the browser clock for plant-time displays.
4. A new `hmi_changed` message plus `hmi_revision` lets screens update without a reload, with swap rules that never rearrange a screen under an active operator.
5. Preferences are a small per-user JSON object with documented keys; they are conveniences, not security state.
6. Trend requests batch all pens into one `/history` call; `agg=minmax` and `points=N` are optional optimizations with clear fallbacks.
7. Every proposal here states its fallback so the HMI can ship before the contract changes.

## Alternatives rejected and why
- **Putting screens in `/config` as extra kinds**: see 02; couples label edits to tag-engine revisions.
- **GraphQL for batch lookups**: api.md rejected GraphQL; `POST /tags/resolve` solves the one real batching need.
- **Server-Sent Events for `hmi_changed`**: a second push channel; the WebSocket already exists.
- **Storing preferences in cookies**: contract has no cookies.
- **Making `hmi_changed` force an immediate re-render**: screens rearranging under an operator's hand is a safety-of-operation problem.
- **A service worker to cache screens offline**: rejected in 06 section 5.3; stale HMI after upgrade is worse.
- **Server-side interpolation (`agg=interpolated`)**: client-side step/linear drawing is enough and keeps the API small.

## Open questions
- All sections above need api.md owner sign-off. Highest priority for milestone 1: section 3 (server time), section 6 (all alarms), section 1 (document store), section 2 (resolve).
- Is `/hmi/assets` needed for milestone 1? Probably not; defer if background images are not used.
- Should `permessage-deflate` be on by default? Needs a measurement on the target edge hardware.
- Should the alarm snapshot include `normal` rows (section 6)? Depends on messages.md section 5 semantics.
- Does `write_requires_reauth` belong in tag-model.md (config field) or in a server-side policy list (so it applies without a tag config revision)?
- The `Deprecation` header and 8594 `Sunset` interplay for `/hmi` kinds is not specified (api.md marks the `Deprecation` RFC UNVERIFIED).
