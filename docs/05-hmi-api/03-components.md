# 03 Component Library

Depends on: [02-screen-format.md](02-screen-format.md) (widget `type`, `bind` slots, `props`, state rules), [06-operator-ergonomics.md](06-operator-ergonomics.md) (styles, colors, quality display), [tag-model.md](../00-contracts/tag-model.md) (datatypes, quality, `enum_labels`, `eng_range`).

Package: `web/packages/components`. Every component is a React function component that receives a `ResolvedWidget` and reads live data only through `useTag`/`useAlarms` (01). Components never call `fetch` for live data.

## 1. Common contract

```ts
export interface WidgetProps<P = Record<string, unknown>> {
  widget: ResolvedWidget & { props: P };
  mode: 'run' | 'preview' | 'design';   // design: show slot names, no writes
}

/** Every component type registers itself. */
export interface ComponentSpec<P> {
  type: string;                          // 'value', 'gauge', ...
  slots: Record<string, SlotSpec>;       // binding slots
  propsSchema: JSONSchema;               // contributes to screen.v1.schema.json $defs
  minSize: { w: number; h: number };     // grid cells
  render: React.ComponentType<WidgetProps<P>>;
}

export interface SlotSpec {
  datatypes: Array<'bool' | 'int64' | 'uint64' | 'float64' | 'string' | 'datetime'>;
  required: boolean;
  writable?: boolean;                    // slot needs a writable tag (control)
  history?: boolean;                     // component will query /history for it
}
```

Every bound value goes through one shared formatter and one shared quality decorator so that all components show numbers, units, quality and staleness the same way:

```ts
export function formatValue(def: TagDefinition, v: TagValue, opts: FormatOpts, locale: string): string;
export function QualityFrame(props: { live: LiveTag; children: React.ReactNode }): JSX.Element;
```

`QualityFrame` rules (visuals in 06, section 4):

| Situation | Value text | Decoration | Tooltip/a11y text |
|---|---|---|---|
| quality class good, not stale | normal | none | "Good, updated 10:05:00" |
| `good_manual` | normal | small "M" badge | "Manually set" |
| class uncertain | normal text, muted | dashed outline + "?" badge | quality name, translated |
| class bad, value present | struck-through grey last value | solid outline + "✕" badge | quality name + last update time |
| class bad, value null | `--` | "✕" badge | quality name |
| browser `link_down` | last value muted | hatched overlay (screen-wide via banner) | "No connection to gateway since 10:07:12" |
| `uncertain_eng_range_exceeded` | normal | "?" badge + over/under arrow | |

Quality is never shown by color alone (shape badge + text), satisfying 07.

## 2. Component catalog (v1)

| `type` | Slots | Purpose |
|---|---|---|
| `value` | `value` (any) | Number/text with label and unit. The workhorse |
| `indicator` | `value` (bool, int64) | State text + semantic style from `states` or `enum_labels` (running/stopped, ok/stale/offline) |
| `bar` | `value` (numeric), optional `sp_low`, `sp_high` | Horizontal/vertical bar with range, alarm limit marks, setpoint marks. Preferred over gauges |
| `gauge` | `value` (numeric) | Analog dial, offered because customers ask; see 2.3 |
| `sparkline` | `value` (numeric, history) | Tiny trend next to a value, last 15 min to 24 h |
| `trend` | `pens[].bind` (numeric/bool, history) | Full trend chart (section 5) |
| `table` | `rows[].cells[].bind` | Grid of values; also "all tags under prefix" mode |
| `alarm_banner` | none (global) | Shell-level, most important unacked alarms (section 4) |
| `alarm_list` | none; `props.prefix` | Filterable, ackable list (section 4) |
| `alarm_summary` | none; `props.prefix` | Counts by severity for an area; click opens filtered list |
| `control` | `target` (writable), optional `feedback` | Setpoint entry, start/stop, open/close with confirmation (05) |
| `symbol` | per symbol | P&ID symbols, built-in or custom `Symbol` (section 7) |
| `pipe` | optional `flowing`, `quality_from` | Line with flow animation |
| `tank` | `level` (numeric) | Vessel with fill level |
| `text` | none | Static label, headings, notes (translatable) |
| `link` | none | Navigation button (screen/popup) |
| `group` | children | Frame with title, contains widgets |
| `tabs` | children per tab | Tabbed container; inactive tabs don't subscribe |
| `connection_status` | `state` (string, `_system/connections/{c}/state`) | Connection health chip |
| `last_updated` | `value` (any) | "Updated 12 s ago", ticks every second |

### 2.1 `value`

```yaml
- id: total
  type: value
  bind: { value: ${meter}/total_volume }
  props:
    label: "Total volume"        # default: tag display_name
    unit_display: m3             # optional display conversion among UCUM-compatible units (section 3)
    decimals: 2                  # default: auto from eng_range span, max 3
    size: large                  # small | normal | large | xl (xl only for 1-3 key KPIs per screen)
    show_quality_text: false     # true: show "Stale" next to the value instead of badge only
    alarm_from_tag: true         # show alarm-state border if the bound tag has an active alarm
```

```
┌─────────────────────────┐
│ Total volume            │   label: secondary text color
│ 1 234.63  m³         ?  │   value: tabular figures, right-aligned; unit smaller
└─────────────────────────┘   "?" uncertain badge, end-aligned (flips in RTL)
```

Datatype handling: `int64`/`uint64` arrive as strings and are formatted with `BigInt`-safe grouping (never `Number()` above 2^53). `bool` shows `props.true_label`/`false_label` (default translated "On"/"Off"). `datetime` shows in the gateway timezone. `string` is plain text, escaped (no markup ever).

### 2.2 `bar`

```
 Flow rate                         12.0 L/min
 0 ├████████░░░░░░░░░░░░░░░░░░░░░░░│ 200
            ▲SP          ▼H 150
```

Range from `eng_range` (or `props.min/max`). Alarm limits are drawn from the tag's alarm definitions (`type: high/low` setpoints) when `props.show_alarm_limits: true`. Normal fill is neutral grey; fill turns to the alarm color only when that alarm is active (06). Out-of-range values pin to the end with an arrow.

### 2.3 `gauge`

Analog dials waste space and are slower to read than bars for exact values, so the designer marks them "prefer `bar`". Provided as a 240-degree arc, neutral, with a numeric readout in the center (the number is primary, the arc secondary). Same range and alarm-limit rules as `bar`.

### 2.4 `table`

Two modes:

```yaml
- id: meters
  type: table
  props:
    columns: [ { key: name, label: "Meter" }, { key: flow, label: "Flow" }, { key: total, label: "Total" }, { key: health, label: "Health" } ]
    rows:
      - { name: "Meter 01", flow: { bind: site1/pumphouse/meter01/flow_rate }, total: { bind: site1/pumphouse/meter01/total_volume }, health: { bind: site1/pumphouse/meter01/health } }
      - { name: "Sim 02",   flow: { bind: site1/sim/meter02/flow_rate },       total: { bind: site1/sim/meter02/total_volume },       health: { bind: site1/sim/meter02/health } }
```

```yaml
- id: all_tags
  type: table
  props:
    mode: prefix                 # engineer diagnostic: every tag under a prefix
    prefix: site1/pumphouse
    columns: [path, value, unit, quality, ts]
```

Prefix mode is the one place a screen subscribes by `prefix` instead of `paths` (dynamic membership, follows `resync` on config change). Virtualized; sortable by any column; quality column filterable ("show only non-good"), which makes it the integrator's commissioning tool.

### 2.5 `connection_status` and `last_updated`

Small, but they carry trust: an operator should never have to guess whether data is live. Every generated overview screen gets one `connection_status` per relevant connection and `last_updated` next to the key KPI.

## 3. Units and number formatting

- Unit display uses a small table mapping UCUM codes to display symbols (`m3` -> `m³`, `Cel` -> `°C`, `L/min` -> `L/min`, `kW.h` -> `kWh`, `%` -> `%`, `1` -> no unit). Unknown codes show the code verbatim.
- Display conversion (`unit_display`) is limited to a fixed table of linear conversions within a dimension (L <-> m3, L/min <-> m3/h, Cel <-> [degF] (UCUM code UNVERIFIED), kPa <-> bar). No general UCUM algebra in the browser in v1.
- Numbers use `Intl.NumberFormat` with the user's locale for grouping/decimal separator, `useGrouping: true`, fixed `minimumFractionDigits = maximumFractionDigits = decimals` so digits do not jump. Tabular (monospace) figures via `font-variant-numeric: tabular-nums`.

## 4. Alarm banner and alarm list

Data: `AlarmStore` (01, section 4.4) from the WS `alarms` subscription; acks and shelving via REST (api.md section 3, Alarms). Alarm state names and fields are those of messages.md section 5.

### 4.1 Banner (always visible in the shell, all screens)

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│ ▲ HIGH  12:00:30  site1/pumphouse/meter01/leak  Flow has not stopped for 2h...  [Ack]│ +3 ▸│
└──────────────────────────────────────────────────────────────────────────────────┘
```

- Shows the single most important alarm: unacked before acked, then severity (`critical` > `high` > `medium` > `low`), then newest `active_at`. Plus a counter of other unacked alarms; click opens the list.
- Unacked alarms: the severity shape blinks at 1 Hz (one blink, not the whole banner; respects `prefers-reduced-motion` by using a static double border instead). Acked active alarms: steady. `cleared_unacked`: steady, outline-only style with "cleared" text, still needs ack.
- Severity is shown by color **and** shape **and** text: critical = octagon, high = triangle, medium = diamond, low = circle (shapes from 06).
- Optional audible annunciation for `high`/`critical` new alarms; browsers require a user gesture before audio plays (autoplay policy), so the HMI shows "click to enable alarm sound" once per session; kiosks can be configured to allow autoplay at browser level (06, section 7).
- Path is shown with the tag's `display_name` when available, path in tooltip.

### 4.2 Alarm list

```
 Filter: [All severities ▾] [Active+unacked ▾] [prefix: site1/pumphouse    ] [Shelved: hide ▾]
 ┌──┬────────┬──────────┬──────────────────────────┬────────────────────────────┬──────────┬──────┐
 │☐ │Sev     │Active at │Source                    │Message                     │State     │      │
 ├──┼────────┼──────────┼──────────────────────────┼────────────────────────────┼──────────┼──────┤
 │☐ │▲ high  │12:00:30  │Meter 01 / Continuous flow│Flow has not stopped for 2h │unacked   │[Ack] │
 │☐ │◆ medium│10:21:00  │Meter 01 / Device health  │Water meter esp32-meter01...│cleared,  │[Ack] │
 │  │        │          │                          │                            │unacked   │      │
 └──┴────────┴──────────┴──────────────────────────┴────────────────────────────┴──────────┴──────┘
 [Ack selected (2)]  [Shelve…]                                       History ▸  (GET /alarms/history)
```

- Columns: severity, `active_at`, source (display name + alarm name), `message`, state, value at activation, `acked_by`/`acked_at`, `shelved_until`.
- Ack: single (`POST /alarms/{alarm_id}/ack`) or bulk (`POST /alarms/ack`) with optional comment; role `operator` (05). Bulk ack requires a confirm step listing the count ("Acknowledge 14 alarms?") to discourage ack-all without reading.
- Shelve: requires comment and duration picker capped at the server max (`24h` default); the server enforces the cap and the comment requirement, the UI mirrors it.
- Sort default: unacked first, then severity, then time. Virtualized for thousands of rows.
- Unknown `state` or `severity` values (api.md: tolerate unknown enums) render as "unknown" with neutral style, never crash.

## 5. Trend chart and charting library choice

### 5.1 Choice: uPlot

Requirements: dense data (hundreds of thousands of points), several pens, multiple Y axes, live append, zoom/pan, small bundle, works offline, MIT-like license.

| Library | Rendering | Fit |
|---|---|---|
| **uPlot** | Canvas 2D | Built for dense time series; very small (tens of KB, UNVERIFIED exact size); fast; MIT. Lower-level API, fewer built-in interactions; we write a thin React wrapper and our own legend/cursor UI. **Chosen.** |
| Apache ECharts | Canvas/SVG | Capable, large-data modes exist, Apache-2.0, but big bundle and a large option surface; reasonable fallback |
| Chart.js | Canvas | Fine for small charts; dense time series needs decimation plugin and is slower in our experience (UNVERIFIED benchmark) |
| Recharts / Victory / Nivo | SVG | One DOM node per point; unsuitable beyond a few thousand points |
| Plotly.js | SVG + WebGL | Heavy bundle; strong for analysis, overkill for HMI |
| Highcharts | SVG/canvas boost | Commercial license; conflicts with open/no-lock-in |

Risk: uPlot is largely maintained by one person (UNVERIFIED current maintainer count). Mitigation: it is small enough to fork, and our wrapper isolates it (`TrendEngine` interface) so ECharts could replace it.

### 5.2 Trend data flow

```
 range = [now-1h, now]
   │
   ├─ initial: GET /history?id=..&start=&end=&agg=?&interval=?    (TanStack Query)
   │     choose agg by density: points_wanted = plot width in px * 2
   │       if raw count estimate <= points_wanted*4  -> agg=raw
   │       else -> agg=min AND agg=max (two requests) with interval = range / points_wanted
   │            drawn as min/max band + avg line (keeps spikes visible)
   │
   └─ live: append from TagStore updates for the same tags (only when range end = now)
          trailing window slides; history re-fetched on zoom/pan with debounce 250 ms
```

- Min/max envelope is essential: plain `avg` downsampling hides the spike the operator is looking for. The API offers `min`, `max`, `avg` separately (api.md), so this costs 2-3 requests per pen; acceptable on LAN. **Contract proposal (minor):** a combined `agg=minmax` returning both per bucket would halve requests; not required.
- Gaps: a `null` bucket (`quality: bad`) breaks the line. For raw data, a gap is drawn when consecutive points are further apart than the tag's `history.max_interval` (default `15m`) times 1.5, because on-change history means a flat line has no points between changes. That needs `history` settings in the tag definition response, which api.md's definition example does not show. **Contract proposal:** include `history: {enabled, mode, interval, max_interval}` in `GET /tags/{tag_id}`.
- Quality on trends: segments from `uncertain` samples are drawn dashed; `bad` points are not drawn (gap) and a thin grey band marks the bad period under the axis.
- Step vs linear: `bool`, `int64` with `enum_labels`, and totals default to step (`style: step`); float measurements to linear. On-change data uses "hold last value" (step-after) by default, which is the honest interpretation of deadband history.
- Interpolation is client-side only; api.md leaves `agg=interpolated` open.
- Time axis in the gateway timezone (07); DST transitions shown correctly because points are UTC and only labels are zoned.
- Max 8 pens per trend, 2 Y axes (left/right); more pens make trends unreadable.
- Ad-hoc trends: the built-in `trends` page lets any user pick tags and save a personal trend (stored as user preference, 08, section 5). Export to CSV calls `GET /history/export` with the same params.
- Cursor readout shows value, quality and timestamp for each pen at the cursor; keyboard: arrow keys move the cursor by one point, `+`/`-` zoom (07).

```ts
export interface TrendPen {
  bind: string;                    // tag path
  y_axis?: 'left' | 'right';
  style?: 'line' | 'step' | 'points';
  label?: string;
  style_token?: 'pen1' | 'pen2' | 'pen3' | 'pen4' | 'pen5' | 'pen6' | 'pen7' | 'pen8'; // never alarm colors
}
export interface TrendProps {
  range: string;                   // duration, e.g. '1h'
  live?: boolean;                  // default true
  pens: TrendPen[];
  y_left?: { min?: number; max?: number; auto?: boolean };
  y_right?: { min?: number; max?: number; auto?: boolean };
}
```

## 6. Sparkline

uPlot with axes hidden, 1 pen, `agg=avg` + min/max band at fixed 120 buckets. Used next to `value` on overview screens to show direction-of-travel (a core high-performance HMI practice: trends beat bare numbers).

## 7. P&ID symbols and SVG animation

### 7.1 Built-in symbol set (v1)

Small, ISA-5.1-inspired, deliberately simple line art (exact conformance to ISA-5.1 is UNVERIFIED; we do not claim it):

| Symbol | Slots | States |
|---|---|---|
| `pump_centrifugal` | `running` (bool), `fault` (bool), `mode` (int64 enum, optional) | stopped (outline), running (filled neutral dark), fault (alarm shape overlay) |
| `valve_gate`, `valve_ball` | `position` (bool or 0-100 float), `fault` | open/closed/travel/in-between |
| `tank_vertical` | `level` (float, % or eng range) | fill level, high/low marks |
| `meter_flow` | `flow` (float), `health` (int64) | flowing indicator, health |
| `motor` | `running`, `fault` | |
| `sensor_bubble` | `value` | ISA-style tag bubble with value readout |
| `pipe` (component) | `flowing` (bool) | static, flowing (dash animation) |

### 7.2 Animation rules

- Animation is **state, not decoration**. Only these animate: pipe flow (slow marching dashes, 1 cycle per 2 s), alarm blink on unacked alarms (1 Hz), and travel state of valves (slow pulse). No spinning impellers: motion draws the eye, and eyes belong on abnormal conditions.
- Implementation: CSS classes toggled on `data-part` elements; CSS `@keyframes` on `stroke-dashoffset` for pipes. No JS animation loops, no SMIL.
- `prefers-reduced-motion: reduce` replaces pipe animation with static arrow markers and blink with a static double outline (07).
- Fill level: `transform: scaleY()` on a clip rect, updated per value (no transition).
- Quality overlay: the symbol's `quality_part` gets the same dashed (uncertain) or solid-with-badge (bad) treatment as `QualityFrame`.

## Decisions made
1. One component contract (`ComponentSpec` with typed slots and a props JSON Schema), one shared formatter and one `QualityFrame` used by every component.
2. Bars preferred over gauges; gauges exist but are marked as such in the designer.
3. uPlot (canvas) for trends and sparklines, wrapped behind a `TrendEngine` interface; ECharts is the documented fallback.
4. Trends use min/max envelopes from separate `agg=min`/`agg=max` queries when data is dense; gap detection from `max_interval`; step-after rendering for on-change data.
5. Alarm severity always shown by color, shape and text; only unacked alarm symbols blink.
6. P&ID animation limited to flow, alarm blink and valve travel; implemented with CSS classes on SVG parts.
7. `table` prefix mode doubles as the commissioning/diagnostic view.

## Alternatives rejected and why
- **SVG chart libraries (Recharts etc.) for trends**: DOM node per point.
- **Highcharts**: commercial license contradicts the open, walk-away promise.
- **A third-party "SCADA component kit"**: none we know of is open, maintained and React-based enough to trust (UNVERIFIED survey); we would inherit its visual language, which is usually the opposite of high-performance HMI.
- **Smooth value transitions (tweening numbers and bars)**: makes values look live when they are not and delays the true reading.
- **3D/gradient tank and pump graphics**: harder to read, distract from abnormal states.

## Open questions
- Contract proposal: `history` block (`enabled`, `mode`, `interval`, `max_interval`) in the tag definition response, needed for correct gap drawing.
- Optional contract proposal: `agg=minmax` for trend envelopes.
- Is a server-side `points=N` (LTTB or min/max downsampling to N points) worth adding to `/history`? It would make trend requests one call per pen; historian owner to decide.
- UCUM display symbol table and the Fahrenheit code need checking against the UCUM spec (UNVERIFIED).
- uPlot bundle size, RTL behavior of its axis labels, and maintenance status: UNVERIFIED, check before committing.
- Built-in symbol set: does a customer need ISA-5.1 exactness, or is "simple and consistent" enough? Ask the first two customers.
