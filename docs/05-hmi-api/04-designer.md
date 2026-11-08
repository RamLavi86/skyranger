# 04 Designer and Editor Approach

Depends on: [02-screen-format.md](02-screen-format.md), [08-api-extensions.md](08-api-extensions.md) (HMI drafts/apply).

## 1. Decision: code-first in v1, visual editor later

v1 authoring is **YAML in an editor with schema validation and a live preview**. A drag-and-drop visual editor comes later and edits the *same documents*.

Reasoning:
1. **The author is the first user.** An embedded/PLC engineer writing YAML is faster than one building a visual editor. A good visual editor (selection, snapping, undo, multi-select, property panels, alignment, zoom) is months of frontend work, the author's weakest area, and it delays the milestone that proves the product.
2. **The format must be right before an editor freezes it.** If the visual editor is the only way to make screens, the file format becomes an implementation detail and drifts towards whatever the editor finds convenient. Code-first forces a clean, hand-writable format, which is exactly the walk-away guarantee.
3. **Integrators work in text already.** Tag config is YAML in git; screens beside them in the same repo, reviewed in the same pull request, is a feature.
4. **Most integrator screens are generated, not drawn.** Overview screens for N meters are better produced from tag config (section 4) than placed by hand.
5. **Risk:** customers who expect a "drag a pump onto the canvas" experience (Ignition Perspective, WinCC) will find v1 spartan. Accepted for the integrator phase: the integrator, not the customer, builds screens.

## 2. v1 designer (route `/designer`, role `engineer`)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Designer   [meter_overview ▾] [+ New]   draft d-0193… (base rev 12)   [Validate] [Apply…]│
├───────────────────────────────────┬──────────────────────────────────────────────┤
│ YAML (Monaco + monaco-yaml)       │ Live preview (real data, writes disabled)     │
│  1 apiVersion: skyranger/v1       │ ┌──────────────┐ ┌─────────────────────────┐ │
│  2 kind: Screen                   │ │ Flow rate    │ │ trend …                 │ │
│  3 metadata:                      │ │ 12.0 L/min   │ │                         │ │
│  4   name: meter_overview         │ └──────────────┘ └─────────────────────────┘ │
│  …  bind:                         │ params: meter = [site1/pumphouse/meter01 ▾]   │
│      value: ${meter}/flow_ra|     │ [ ] show slot names  [ ] show grid            │
│          ┌───────────────────┐    │ viewport: [Desktop ▾] [LTR|RTL] [lang: en ▾]  │
│          │ flow_rate  L/min  │◄── tag path completion from GET /tags?prefix=      │
│          │ flow_rate_avg ... │    │                                              │
├───────────────────────────────────┴──────────────────────────────────────────────┤
│ Problems: ⚠ widget meter_health: binding value: path site1/.../helth not found     │
│ Tags used: 7   Diff vs rev 12: +1 widget, ~2 props                                │
└──────────────────────────────────────────────────────────────────────────────────┘
```

Features, all thin layers over the format and the public API:
- **Schema completion and validation:** `monaco-yaml` loaded with `GET /api/v1/hmi/schema` gives property completion, enums, hover docs from schema `description`s. (monaco-yaml's JSON Schema 2020-12 support level is UNVERIFIED; if only draft-07 is supported, the server also serves a draft-07 rendition.)
- **Tag path completion:** custom completion provider inside `bind` values, querying `GET /tags?prefix=...&limit=50`; shows display name, unit, datatype; warns on datatype not accepted by the slot.
- **Live preview:** debounced 300 ms parse -> resolve -> render in the right pane with real live data. Preview mode disables `control` writes (renders them with a "preview" stripe) so an engineer cannot actuate equipment while typing.
- **Viewport, direction and language toggles:** preview at phone width, in RTL, in each configured language, which catches layout bugs before operators see them.
- **Click-to-source:** clicking a widget in the preview moves the editor cursor to its `id:` line. This single feature gives much of the benefit of a visual editor for little cost.
- **Drafts and apply:** saving creates/updates an HMI draft (`POST /api/v1/hmi/drafts`); `Apply…` shows the diff and requires a comment, then `POST /api/v1/hmi/drafts/{id}/apply`. 409 on stale `base_revision` shows a three-way choice: reload theirs, overwrite (re-base and apply), or copy mine to a new screen.
- **Import/export:** upload/download multi-doc YAML from the designer.
- **No lock-in to the designer:** everything it does is available via the API and the CLI. An engineer may use VS Code with the same JSON Schema (served by URL) and `skyranger-cli hmi apply`.

## 3. Grid helpers without a visual editor

To make code-first layouts tolerable:
- Preview overlay shows grid cell coordinates on hover ("x 4, y 2").
- Overlapping widgets are a validation warning with both ids.
- `grid` may omit `y` (auto-flow: next free row) so most screens only state `w`/`h` and order.

## 4. Generators (scaffolding from tag config)

The fastest way to a decent screen is to generate it from template instances:

- `POST /api/v1/hmi/generate` with `{"template": "pulse_water_meter", "kind": "faceplate" | "overview", "instances_prefix": "site1"}` returns a draft `Screen` document (not applied) built from a **screen recipe** per template.
- Recipes are themselves YAML documents stored with the template in the HMI store (`kind: Screen` with `params` of type `tag_prefix`, annotated `skyranger.io/template: pulse_water_meter`). The water meter faceplate in [09-water-meter-slice.md](09-water-meter-slice.md) is the first recipe.
- `overview` generates a `table` with one row per instance plus an `alarm_summary`.
- Generated output is ordinary YAML the engineer then edits. No hidden linkage: regenerating overwrites only when asked.

## 5. Visual editor (later, v2 of the designer, not of the format)

Scope when built:
- Canvas and grid editing of existing documents: drag, resize, align, distribute, z-order, multi-select, copy/paste between screens, undo/redo.
- Property panel generated from component `propsSchema` (JSON Schema forms; a library like `@rjsf/core` is a candidate, UNVERIFIED fit).
- Tag browser drag-and-drop onto slots.
- Symbol editor: import SVG, sanitize, name parts, define rules.
- Round-trip guarantee: open YAML -> edit visually -> save produces a minimal diff; comments in YAML are preserved where the `yaml` library can (it keeps comments attached to nodes; complex edits may move them, UNVERIFIED in practice).

Criteria to start it: the format has survived two real customer projects without breaking changes, and a customer (not the integrator) needs to make their own screens.

## Decisions made
1. v1 authoring is code-first: YAML in Monaco with JSON Schema completion, tag path completion, live preview with writes disabled, click-to-source, drafts with diff and apply.
2. Generators produce draft screens from templates; the output is plain, editable YAML.
3. The future visual editor edits the same documents and must round-trip with minimal diffs; it is not started until the format has been proven on two projects.
4. Every designer function is available through the public API and CLI.

## Alternatives rejected and why
- **Visual editor first**: months of frontend work in the author's weakest area before the first dashboard; freezes an unproven format.
- **Adopting an existing open-source HMI designer** (e.g. the editors of FUXA or other open SCADA projects): each carries its own storage format and visual style; integrating or forking would make their format ours. Worth studying for ideas (UNVERIFIED which ones are actively maintained).
- **Grafana as the HMI/designer**: excellent dashboards and trends, but it is not built for operator control, P&ID graphics, alarm acknowledgment workflows, or tight write confirmation, and it would be a second product to secure and update. Customers can still point Grafana at TimescaleDB for analysis (plain SQL, decision 3).
- **Low-code JSON forms only (no YAML editor)**: worse for diffing and bulk edits.

## Open questions
- Is Monaco's size and complexity worth it versus CodeMirror 6 (smaller, good YAML support via language packages, schema completion needs more integration work; UNVERIFIED)? Decide in the first spike: whichever gives schema completion with less code.
- Should screen recipes be shipped inside `Template` annotations in tag config instead of the HMI store? Keeping them in the HMI store avoids tag-engine revisions for UI tweaks.
- Do we need per-user draft isolation, or is one shared draft list (like `/config/drafts`) enough for 1-3 engineers?
