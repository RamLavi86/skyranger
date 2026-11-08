# 02 Screen and View Definition Format

Depends on: [tag-model.md](../00-contracts/tag-model.md) sections 1, 9, 10 (envelope, references, `${param}` substitution), [README.md](../00-contracts/README.md) (identifier grammar, encodings).

Screens are **data, not code**. A screen is a YAML (or JSON) document that names components, positions them, and binds them to tags. A customer can export all screens as one file, read it in a text editor, put it in git, and rebuild it in another product if they leave. No screen ever contains JavaScript.

## 1. Document kinds

Same Kubernetes-style envelope as gateway config (tag-model.md section 9), same `apiVersion: skyranger/v1`.

| `kind` | Purpose |
|---|---|
| `Screen` | One page: layout, widgets, bindings, optional parameters (a parameterized screen is a faceplate or popup) |
| `Navigation` | Menu tree, home screen per role, kiosk rotations |
| `Symbol` | A reusable SVG graphic with named parts and state rules (custom P&ID symbols) |

These kinds are **not** part of the gateway tag configuration (`/api/v1/config`), which stays `Gateway`, `Connection`, `Template`, `TagGroup`. They live in a separate, separately revisioned HMI store (`/api/v1/hmi`, see [08-api-extensions.md](08-api-extensions.md)). Reason: editing a screen must never cause a tag-engine reload, and screen edits are made by different people at a different rhythm. **Contract proposal:** register `Screen`, `Navigation`, `Symbol` as HMI kinds under `apiVersion: skyranger/v1`.

## 2. `Screen`

```yaml
apiVersion: skyranger/v1
kind: Screen
metadata:
  name: meter_overview                   # segment grammar; URL: /s/meter_overview
  display_name: "Water meter"
  description: "Overview of one pulse water meter"
  labels: { area: pumphouse }
spec:
  params:                                # optional; makes the screen reusable (faceplate)
    meter:
      type: tag_prefix                   # tag_prefix | string | int64 | float64 | bool
      default: site1/pumphouse/meter01
      display_name: "Meter"
  layout:
    type: grid                           # grid | canvas
    columns: 12
    row_height: 56                       # CSS px at 100% zoom
    gap: 8
    breakpoints: { narrow: 600 }         # below 600 px: widgets stack in `order`
  refresh:
    min_interval: 0s                     # passed to WS subscribe (api.md section 6)
  access:
    view: viewer                         # minimum role to open (05-roles-and-control.md)
  widgets:
    - id: flow                           # unique within screen, segment grammar
      type: value                        # component type (03-components.md)
      grid: { x: 0, y: 0, w: 4, h: 2 }
      order: 1
      bind:
        value: ${meter}/flow_rate        # tag path after substitution
      props:
        label: "Flow rate"               # default: tag display_name
        decimals: 1
    - id: flow_trend
      type: trend
      grid: { x: 4, y: 0, w: 8, h: 5 }
      order: 3
      props:
        range: 1h
        pens:
          - bind: ${meter}/flow_rate
            y_axis: left
          - bind: ${meter}/total_volume
            y_axis: right
            style: step
  tag_ids:                               # written on export; optional on import
    site1/pumphouse/meter01/flow_rate:    01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3b
    site1/pumphouse/meter01/total_volume: 01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a
```

### 2.1 Envelope rules

- `metadata.name`: segment grammar, unique per kind. Screens are addressed by name in URLs.
- Unknown fields: rejected by the editor's schema validation inside `spec` (catches typos), ignored inside `metadata.annotations` and `props` of unknown component types (forward compatibility, api.md section 8). An unknown widget `type` renders a visible "unsupported widget: {type}" placeholder, never an empty space.
- Durations: Go duration strings (`1h`, `500ms`). Timestamps never appear in screen documents.

### 2.2 Layouts

**`grid`** for dashboards and lists: 12-column grid, `grid: {x, y, w, h}` in cells. Below `breakpoints.narrow` the grid collapses to one column ordered by `order` (mobile, 06). This is what most integrator screens should use.

**`canvas`** for P&ID-style process graphics: a fixed coordinate space scaled to fit (SVG `viewBox` semantics, aspect preserved, letterboxed).

```yaml
  layout:
    type: canvas
    width: 1920
    height: 1080
    background: none                     # or a Symbol name for static piping drawings
  widgets:
    - id: p101
      type: symbol
      at: { x: 420, y: 300, w: 80, h: 80, rotate: 0 }   # canvas units; rotate in degrees, 0/90/180/270
      props: { symbol: pump_centrifugal, label: "P-101" }
      bind:
        running: site1/pumphouse/p101/running
        fault:   site1/pumphouse/p101/fault
      on_click: { open: pump_faceplate, params: { pump: site1/pumphouse/p101 } }
    - id: line1
      type: pipe
      props: { points: [[500, 340], [800, 340], [800, 500]], width: 6 }
      bind:
        flowing: site1/pumphouse/p101/running
```

Canvas screens do not reflow on mobile; they pan/zoom (06, section 6). Integrators should provide a grid "summary" screen for phones.

### 2.3 Widget fields

| Field | Required | Notes |
|---|---|---|
| `id` | yes | Segment grammar, unique per screen. Used in tests, i18n keys, and the future visual editor |
| `type` | yes | Component type (03) |
| `grid` / `at` | per layout | Position |
| `order` | no | Reading/stacking order; default document order. Also the keyboard tab order (07) |
| `bind` | no | Map of **binding slot** to tag path. Slots are defined per component type |
| `props` | no | Static properties, schema per component type |
| `visible` | no | Visibility rule (section 4) |
| `on_click` | no | Navigation action: `open` (screen, as page) or `popup` (screen as dialog), with `params`. Writes are never `on_click` actions; they use the `control` component type (05) |
| `access` | no | `{view: <role>}`; hidden below this role |
| `children` | no | For `group` and `tabs` containers: nested widgets, positions relative to container |

## 3. Binding

### 3.1 Syntax

A binding value is a **tag path** string, after `${param}` substitution. Syntax and substitution rules are those of tag-model.md section 10 (whole-scalar typed substitution, `${...}` interpolation inside strings, `$${` for literal, unknown `${...}` is a validation error). No expressions, conditionals, or loops, deliberately matching the tag model.

```yaml
bind:
  value: site1/pumphouse/meter01/flow_rate      # absolute path, no leading slash
  value: ${meter}/flow_rate                     # param of type tag_prefix + relative tail
```

Unlike config references, screen bindings have no relative form without a param: screens are not inside an instance tree. Paths are the readable choice; `tag_id`s are the safety net (section 3.3).

Binding to `_system/...` tags is allowed (connection state, gateway health) and is how the link/health indicators are built.

### 3.2 What a binding gives the component

A binding slot resolves to two things:
1. The live `LiveTag` (value, quality, stale) from `TagStore` (01).
2. The tag definition (`display_name`, `unit`, `datatype`, `eng_range`, `enum_labels`, `writable`, `alarms`) from `GET /tags/...` (cached via TanStack Query). Components take defaults from the definition: a `value` widget bound to `flow_rate` shows "Flow rate", unit `L/min`, and a gauge uses `eng_range` as its scale, unless `props` override them.

So the integrator configures units, ranges and names **once**, in the tag config, and screens inherit them.

### 3.3 Rename safety: `tag_ids`

Paths are mutable (tag-model.md section 1); renaming breaks path bindings. Mechanism, mirroring the TagGroup `tag_ids` map:

- On **save/apply**, the server resolves every binding path and writes `spec.tag_ids: {path: tag_id}` into the stored document.
- On **load**, the resolver checks each path. If a path no longer exists but its `tag_id` does (renamed), the HMI binds to the new path and shows a "screen has renamed tags, re-save to update" notice to engineers. The server's `GET /hmi/screens/{name}?resolve=true` performs the same check and reports `renamed: [{old_path, new_path, tag_id}]`.
- A path with neither a live path nor a known id is a **dangling binding**: the widget renders "--" with quality-style "config error" marking (`bad_config_error` visual) and a tooltip naming the missing path. Unlike tag config, a dangling screen binding does **not** block saving (screens are often drafted before tags exist), but it is reported as a validation warning.
- Import of a file without `tag_ids` matches by path only.

### 3.4 Static binding extraction

Because bindings are plain strings in known slots, the resolver (`packages/screen-model/src/resolve.ts`) can compute the complete path list of a screen **before rendering**. This is what makes one-subscription-per-screen possible (01, section 4.2) and lets the server answer "which screens use this tag?" (`GET /hmi/usages?tag_id=`), shown in the tag editor before a rename or delete.

```ts
export interface ResolvedScreen {
  name: string;
  params: Record<string, ParamValue>;
  widgets: ResolvedWidget[];
  paths: string[];                 // unique, sorted; feeds one WS subscribe
  problems: BindingProblem[];      // dangling, renamed, type mismatch
}

export interface ResolvedWidget {
  id: string;
  type: string;
  position: GridPos | CanvasPos;
  bind: Record<string, string>;    // slot -> resolved path
  props: Record<string, unknown>;
  visible?: VisibilityRule;
  onClick?: NavAction;
  children?: ResolvedWidget[];
}

export interface BindingProblem {
  widget_id: string;
  slot: string;
  path: string;
  code: 'dangling' | 'renamed' | 'datatype_unsupported' | 'not_writable';
  new_path?: string;
}
```

## 4. State-driven behavior without expressions

Every dynamic behavior is a declarative rule against one bound value. This covers the large majority of real HMI needs and stays diffable and portable.

### 4.1 State map (used by symbols, indicators, text)

```yaml
- id: meter_health
  type: indicator
  bind: { value: ${meter}/health }
  props:
    states:                              # first match wins; evaluated in order
      - when: { eq: 2 }   # ok
        style: normal
        label: "Reporting"
      - when: { eq: 1 }
        style: warning                   # semantic style names, see 06
        label: "Stale"
      - when: { eq: 0 }
        style: abnormal
        label: "Offline"
    default: { style: abnormal, label: "Unknown" }
```

`when` operators: `eq`, `ne`, `lt`, `le`, `gt`, `ge`, `in` (list), `between` (`[low, high]`, inclusive), `quality` (`good` | `uncertain` | `bad`, class match). Semantic `style` names come from the design tokens (06); screens never set raw colors for states. This makes customer-wide color changes one token file, and prevents rainbow screens.

If the bound tag has `enum_labels` and no `states` are given, labels come from the tag and style is `normal`.

### 4.2 Visibility

```yaml
visible: { bind: ${meter}/raw/online, when: { eq: true } }
```

Hidden widgets still subscribe (cheap) so they appear instantly. A widget hidden by `access` does not subscribe.

### 4.3 What is deliberately not possible in v1

- Arbitrary expressions (`value * 3.6`, `a && b`). Compute it as a derived tag instead, so it is historized, alarmable and visible to every client. When scripting lands, a `kind: script` tag is the answer, not screen JavaScript.
- Custom JavaScript widgets. A future plugin mechanism for components would need sandboxing (Open questions).
- Writes from `on_click`.

## 5. `Navigation`

```yaml
apiVersion: skyranger/v1
kind: Navigation
metadata: { name: main }
spec:
  home:                                  # per role, falls back up the role ladder
    viewer: plant_overview
    operator: plant_overview
    engineer: plant_overview
  menu:
    - label: "Overview"
      screen: plant_overview
    - label: "Pumphouse"
      children:
        - label: "Meter 01"
          screen: meter_overview
          params: { meter: site1/pumphouse/meter01 }
        - label: "Pumps"
          screen: pumps
    - label: "Alarms"
      builtin: alarms                    # built-in pages: alarms, trends, events, discovery, connections
  kiosks:
    lobby_wall:
      screens:
        - { screen: plant_overview, dwell: 30s }
        - { screen: meter_overview, params: { meter: site1/pumphouse/meter01 }, dwell: 30s }
      pause_on_alarm: true               # stop rotating while unacked high/critical alarms exist
```

One `Navigation` named `main` is used by default. Menu labels are translatable (07). Menu entries for screens the user cannot view are hidden.

## 6. `Symbol` (custom SVG)

Built-in symbols (03, section 7) cover common P&ID shapes. Customers and integrators add their own:

```yaml
apiVersion: skyranger/v1
kind: Symbol
metadata: { name: valve_gate }
spec:
  view_box: [0, 0, 64, 64]
  slots:                                 # binding slots this symbol accepts
    position: { datatype: [bool, int64], required: true }
    fault:    { datatype: [bool] }
  svg: |
    <g>
      <path data-part="body" d="M8 16 L32 32 L8 48 Z M56 16 L32 32 L56 48 Z"/>
      <rect data-part="fault_marker" x="26" y="2" width="12" height="12"/>
    </g>
  rules:
    - part: body
      slot: position
      states:
        - { when: { eq: true },  style: open }
        - { when: { eq: false }, style: closed }
    - part: fault_marker
      slot: fault
      states:
        - { when: { eq: true },  style: alarm_high, show: true }
        - { when: { eq: false }, show: false }
  quality_part: body                     # which part gets the bad/uncertain quality overlay
```

SVG rules (enforced by the server on save and by the client on render, using a strict allowlist):
- Allowed elements: `g`, `path`, `rect`, `circle`, `ellipse`, `line`, `polyline`, `polygon`, `text`, `tspan`. No `script`, `foreignObject`, `image`, `use` with external href, `style` elements, `on*` attributes, or `href`/`xlink:href` attributes.
- Allowed attributes: geometry, `transform`, `data-part`, `fill`/`stroke` only as `currentColor` or `none` (color always comes from the style token). Anything else is stripped with a warning.
- Rationale: the security threat model (07-security, threat 8) requires strict CSP and no injected markup; symbols are customer-editable content.

Rendering: the sanitized SVG is parsed once into a React element tree (never `dangerouslySetInnerHTML`), and rules toggle CSS classes on parts (`sr-style-open`, `sr-hidden`).

## 7. Versioning, storage and export

- **Schema version** is `apiVersion: skyranger/v1`. Within v1, changes are additive (new widget types, new optional props). A breaking change ships as `skyranger/v2` with a converter in the gateway (`POST /api/v1/hmi/convert`, future) and the old version stays readable for one release cycle (same policy as api.md section 8).
- **Revisions:** the HMI store is revisioned like `/config`: drafts, validation, diff, apply with `base_revision` optimistic concurrency, history of revisions with author and comment, restore. Every apply is audited. Detailed endpoints in [08-api-extensions.md](08-api-extensions.md).
- **Export:** `GET /api/v1/hmi` with `Accept: application/yaml` returns every `Screen`, `Navigation` and `Symbol` as one multi-document YAML stream with `tag_ids` filled in. Export then import of the same content produces an empty diff (same lossless rule as tag config).
- **Full walk-away bundle:** `GET /api/v1/config` (tags) + `GET /api/v1/hmi` (screens) + `GET /api/v1/history/export` (CSV) is everything a customer needs. The HMI's admin page offers a "Download everything" button that fetches the first two and lets the user pick a history range for the third.
- **Git-friendly:** the integrator workflow is to keep `screens/*.yaml` in a repo and push them with `skyranger-cli hmi apply screens/` (CLI owned by the deployment area; it calls the same API). Key order in exported YAML is stable (envelope, then `spec` keys in schema order, `widgets` in document order, `tag_ids` sorted) so diffs are small.
- **JSON Schema:** `packages/screen-model/schema/screen.v1.schema.json` (2020-12) is the normative per-field spec, served at `GET /api/v1/hmi/schema` for editors. Component `props` schemas are included via `$defs` per widget `type`.

## Decisions made
1. Screens, navigation and symbols are YAML/JSON documents in the same `apiVersion`/`kind`/`metadata`/`spec` envelope as config, stored in a separate revisioned HMI store, never in tag config.
2. Bindings are tag paths with the tag model's `${param}` substitution; the server writes a `tag_ids` map so renames are detected and recovered.
3. Two layouts: `grid` (responsive, default) and `canvas` (fixed coordinates, P&ID).
4. No expressions or JavaScript in screens: dynamic behavior is ordered `when` rules against one bound value, mapped to semantic styles. Computation belongs in derived tags.
5. Custom symbols are sanitized SVG with `data-part` names and rules; color comes only from style tokens.
6. Dangling bindings are warnings, not errors, and render visibly as config errors.
7. Writes are a dedicated `control` component, never a generic click action.

## Alternatives rejected and why
- **Storing screens in `/config` as more kinds**: one revision stream for tags and pictures; a label change would create a tag-engine revision and an apply diff full of noise. Contract says v1 config kinds are four.
- **Screens as React/TSX code**: maximal flexibility, but not exportable as data, unsafe to accept from customers, requires a build to change a label, and impossible to render in another product.
- **An expression language in bindings** (JS subset, CEL, JSONata): every HMI that has one ends up with logic hidden in pictures that nobody can audit or historize. Matches the tag-model decision to keep logic in Go built-ins now and Python later.
- **Binding by `tag_id` only**: unreadable by humans and in diffs; integrators write paths.
- **Absolute pixel layout everywhere**: unusable on phones and different monitor sizes.
- **Arbitrary SVG upload**: XSS vector (scripts, event handlers, external references).

## Open questions
- Contract proposal: register HMI kinds `Screen`, `Navigation`, `Symbol` under `apiVersion: skyranger/v1` and the `/api/v1/hmi` store (needs api.md owner sign-off).
- Is "no expressions" too strict for common integrator needs like "show A minus B"? Mitigation is a derived tag; is a `difference`/`sum` built-in needed in the tag model? (tag-model owner).
- Should screens be able to reference a **template instance** rather than a prefix (`params.meter: {type: instance, template: pulse_water_meter}`), so the designer can offer a picker filtered by template? Likely yes; needs instance listing in the API.
- Plugin components (customer-written JS widgets): needs a sandbox (iframe with `sandbox` attribute and postMessage?) and is deferred.
- Should `access.view` support per-area permissions? Depends on api.md open question on areas.
