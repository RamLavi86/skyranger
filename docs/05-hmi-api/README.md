# 05 Web HMI and API

Status: design draft for v1. Owner: HMI/API area.

This area covers the browser HMI (TypeScript + React) and the parts of the REST/WebSocket API that the HMI needs beyond the binding contract in [../00-contracts/api.md](../00-contracts/api.md). Nothing here overrides the contracts. Where this area needs a contract change, it is listed as a **contract proposal** and repeated in the Open questions of the relevant file.

## Reading order

| # | File | Covers |
|---|---|---|
| 1 | [01-architecture.md](01-architecture.md) | Monorepo layout, stack, state management, WebSocket client (reconnect, resync, seq checks), performance with thousands of live tags |
| 2 | [02-screen-format.md](02-screen-format.md) | `Screen`, `Navigation`, `Symbol` documents (YAML/JSON), tag binding, parameters (faceplates), versioning, storage, export |
| 3 | [03-components.md](03-components.md) | Component library: value, bar, gauge, trend, table, alarm banner/list, P&ID symbols with SVG animation; charting library choice |
| 4 | [04-designer.md](04-designer.md) | Code-first authoring in v1, preview workflow, path to a visual editor |
| 5 | [05-roles-and-control.md](05-roles-and-control.md) | Role-based views, write permissions, confirmation flow for control actions, write outcomes |
| 6 | [06-operator-ergonomics.md](06-operator-ergonomics.md) | High-performance HMI principles, color and alarm palette, quality display, offline and degraded behavior, mobile and kiosk |
| 7 | [07-i18n-a11y.md](07-i18n-a11y.md) | Localization, right-to-left (Hebrew), number/time/unit formatting, accessibility |
| 8 | [08-api-extensions.md](08-api-extensions.md) | Additive REST and WebSocket details: HMI document endpoints, tag resolve, preferences, server time, trend queries |
| 9 | [09-water-meter-slice.md](09-water-meter-slice.md) | Milestone 1 dashboard spec: layout, bindings, scenario behavior, acceptance tests |

## One-paragraph summary

The HMI is a static single-page app served by `skyrangerd`, built from an npm-workspaces monorepo under `web/`. It talks only to the public API: REST for reads and every mutation, one WebSocket per browser tab for subscriptions. Live values do not live in React state. They live in a framework-free `TagStore` that applies `snapshot`/`update` batches, checks `seq`, and notifies only the components bound to changed tags, at most once per animation frame. Screens are versioned YAML/JSON documents (`kind: Screen`) with the same Kubernetes-style envelope as gateway config, bound to tags by path with a `tag_ids` safety map, parameterizable like templates, and exportable as one multi-document YAML file a customer can take away. v1 authoring is code-first (YAML + JSON Schema in an editor, live preview from a draft); a visual editor comes later and writes the same files. The visual language follows high-performance HMI practice: grey and muted by default, color reserved for abnormal states and alarms, quality and link state always visible, never conveyed by color alone.

## Area-wide decisions (details in each file)

1. Live data path: WebSocket into a custom `TagStore` read by `useSyncExternalStore`; server data (definitions, history, alarm lists) via TanStack Query; no Redux.
2. Screens are documents with `apiVersion: skyranger/v1`, kinds `Screen`, `Navigation`, `Symbol`, stored in a separate revisioned HMI store with the same draft/apply/export API pattern as `/config`.
3. Bindings are by tag path (readable, hand-editable), with an exported `tag_ids` map to survive renames.
4. Trend charts use uPlot (canvas); gauges, bars and P&ID symbols are own SVG components.
5. Code-first authoring for v1; visual editor deferred.
6. Control actions: REST write with a confirm dialog, `?wait=10s`, explicit "outcome unknown" handling, never auto-retried.
7. Displayed times use the gateway timezone (`/system/info` `timezone`), not the browser's.

## Decisions made
1. Split into nine files by concern, with the water meter slice last so it can reference everything else.
2. Contract proposals are additive only and are listed explicitly rather than assumed.

## Alternatives rejected and why
- **One large document**: hard for parallel review, and the screen format deserves its own stable reference.
- **Putting HMI-specific endpoints in a private API**: forbidden by api.md ("the HMI uses only this public API") and bad for walk-away.

## Open questions
- Contract proposals collected from all files (owners must accept or reject):
  1. `welcome.server_time` and `pong.server_time` WebSocket fields (08).
  2. New server message `hmi_changed` and `welcome.hmi_revision` (08).
  3. `/api/v1/hmi/...` endpoints and the `Screen`/`Navigation`/`Symbol` kinds sharing `apiVersion: skyranger/v1` (02, 08).
  4. `POST /api/v1/tags/resolve` for bulk definition lookup (08).
  5. `history` (mode, `max_interval`) in the tag definition response, needed to draw gaps correctly (03, 08).
  6. Subscribing to all alarms (`prefix: ""` or omitted) on WebSocket (08).
  7. Localized tag `display_name` (07).
  8. `GET/PUT /users/me/preferences` (08 section 5).
  9. Trend helpers: `agg=minmax`, `points=N` on `/history` (08 section 7).
  10. Server-enforced write step-up: `POST /auth/reauth`, `X-Reauth-Token`, `write_requires_reauth` (08 section 10; security area co-owns).
