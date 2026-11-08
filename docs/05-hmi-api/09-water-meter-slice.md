# 09 Milestone 1 Dashboard: Water Meter Vertical Slice

Depends on: [tag-model.md](../00-contracts/tag-model.md) section 11 (the `pulse_water_meter` template and its tags), [messages.md](../00-contracts/messages.md), [02-screen-format.md](02-screen-format.md), [03-components.md](03-components.md), [05-roles-and-control.md](05-roles-and-control.md), [06-operator-ergonomics.md](06-operator-ergonomics.md), [08-api-extensions.md](08-api-extensions.md).

Goal: the smallest HMI that proves the whole chain: ESP32 (or simulated) meter, tag engine, history, live browser. It uses only components from 03 and documents from 02. It is also the first **screen recipe** (04 section 4): the faceplate is parameterized by `meter` so every `pulse_water_meter` instance gets one.

## 1. Scope

In scope: login, navigation shell (header, alarm banner, link banner, status bar), one overview screen listing meters, one faceplate screen per meter, built-in alarm list and trends pages, discovery approval page (admin form, thin), Hebrew and English, kiosk rotation.

Out of scope for the slice: P&ID canvas screens, the designer UI (screens are written as YAML and pushed with the API or CLI), mobile-specific layouts beyond the one-column reflow, user management UI (use the API).

Data used (all from the template, nothing new):

| Tag (under `site1/pumphouse/meter01/`) | Datatype, unit | Shown as |
|---|---|---|
| `total_volume` | float64, `L` | KPI value, `unit_display: m3` |
| `flow_rate` | float64, `L/min`, `eng_range` 0..200 | KPI value, bar, sparkline, trend |
| `health` | int64 enum 0 offline, 1 stale, 2 ok | indicator |
| `leak` | bool, alarm `leak` (high) | indicator and alarm banner |
| `raw/online` | bool | visibility of the "device offline" hint |
| `raw/rssi_dbm` | int64 | small value, engineer diagnostics |
| `raw/uptime_s` | uint64, `s` | `last_updated` source |
| `raw/boot_count` | uint64 | diagnostics table only |
| `_system/connections/local-sparkplug/state` | string | `connection_status` |

A second, simulated meter `site1/sim/meter02/...` (same template, `sim` driver) makes the overview table meaningful and lets fault injection run without touching hardware (decision 7).

## 2. Information hierarchy (what the operator sees first)

Per 06: grey and quiet when normal, color only for abnormal. Reading order on the faceplate, top to bottom:

1. **Is this device alive?** `health` indicator and `connection_status`, with `last_updated`.
2. **Is water flowing, how much?** `flow_rate` KPI and bar.
3. **How much has passed?** `total_volume` KPI.
4. **What happened lately?** trend of flow and total, 1 h default.
5. **Is anything wrong?** alarms for this meter (list filtered by prefix).
6. **Change something** (engineers and operators): "Align total with register" control, collapsed under a group.

## 3. Navigation

```yaml
apiVersion: skyranger/v1
kind: Navigation
metadata: { name: main }
spec:
  home: { viewer: plant_overview, operator: plant_overview, engineer: plant_overview }
  menu:
    - label: "Overview"
      screen: plant_overview
    - label: "Meter 01"
      screen: meter_faceplate
      params: { meter: site1/pumphouse/meter01 }
    - label: "Meter 02 (sim)"
      screen: meter_faceplate
      params: { meter: site1/sim/meter02 }
    - { label: "Alarms",  builtin: alarms }
    - { label: "Trends",  builtin: trends }
    - { label: "Devices", builtin: discovery }
  kiosks:
    lobby_wall:
      screens:
        - { screen: plant_overview, dwell: 30s }
        - { screen: meter_faceplate, params: { meter: site1/pumphouse/meter01 }, dwell: 30s }
      pause_on_alarm: true
```

The `Devices` entry is hidden below `engineer` (05 section 1).

## 4. Screen: `plant_overview` (grid)

```yaml
apiVersion: skyranger/v1
kind: Screen
metadata: { name: plant_overview, display_name: "Overview" }
spec:
  default_language: en
  layout: { type: grid, columns: 12, row_height: 56, gap: 8, breakpoints: { narrow: 600 } }
  refresh: { min_interval: 1s }
  access: { view: viewer }
  widgets:
    - id: summary
      type: alarm_summary
      grid: { x: 0, y: 0, w: 12, h: 1 }
      order: 1
      props: { prefix: site1 }
    - id: meters
      type: table
      grid: { x: 0, y: 1, w: 12, h: 4 }
      order: 2
      props:
        columns:
          - { key: name,   label: "Meter" }
          - { key: flow,   label: "Flow" }
          - { key: total,  label: "Total" }
          - { key: health, label: "Health" }
        rows:
          - name: "Meter 01"
            flow:   { bind: site1/pumphouse/meter01/flow_rate }
            total:  { bind: site1/pumphouse/meter01/total_volume }
            health: { bind: site1/pumphouse/meter01/health }
            on_click: { open: meter_faceplate, params: { meter: site1/pumphouse/meter01 } }
          - name: "Meter 02 (sim)"
            flow:   { bind: site1/sim/meter02/flow_rate }
            total:  { bind: site1/sim/meter02/total_volume }
            health: { bind: site1/sim/meter02/health }
            on_click: { open: meter_faceplate, params: { meter: site1/sim/meter02 } }
    - id: conn_spb
      type: connection_status
      grid: { x: 0, y: 5, w: 3, h: 1 }
      order: 3
      bind: { state: _system/connections/local-sparkplug/state }
    - id: conn_sim
      type: connection_status
      grid: { x: 3, y: 5, w: 3, h: 1 }
      order: 4
      bind: { state: _system/connections/sim/state }
```

Row-level `on_click` inside `table` rows is a small extension of the `on_click` navigation action in 02 section 2.3 (still navigation only, never a write). If the table component does not support it at implementation time, the first column becomes a `link` widget list instead. (Open questions.)

## 5. Screen: `meter_faceplate` (grid, parameterized)

```yaml
apiVersion: skyranger/v1
kind: Screen
metadata:
  name: meter_faceplate
  display_name: "Water meter"
  annotations: { skyranger.io/template: pulse_water_meter }
spec:
  default_language: en
  params:
    meter: { type: tag_prefix, default: site1/pumphouse/meter01, display_name: "Meter" }
  layout: { type: grid, columns: 12, row_height: 56, gap: 8, breakpoints: { narrow: 600 } }
  refresh: { min_interval: 0s }
  access: { view: viewer }
  widgets:
    - id: health
      type: indicator
      grid: { x: 0, y: 0, w: 3, h: 1 }
      order: 1
      bind: { value: ${meter}/health }
      props:
        states:
          - { when: { eq: 2 }, style: normal,   label: "Reporting" }
          - { when: { eq: 1 }, style: warning,  label: "Stale" }
          - { when: { eq: 0 }, style: abnormal, label: "Offline" }
        default: { style: abnormal, label: "Unknown" }
    - id: updated
      type: last_updated
      grid: { x: 3, y: 0, w: 3, h: 1 }
      order: 2
      bind: { value: ${meter}/raw/uptime_s }
    - id: leak
      type: indicator
      grid: { x: 6, y: 0, w: 3, h: 1 }
      order: 3
      bind: { value: ${meter}/leak }
      props:
        states:
          - { when: { eq: true },  style: alarm_high, label: "Continuous flow" }
          - { when: { eq: false }, style: normal,     label: "No continuous flow" }
        default: { style: abnormal, label: "Unknown" }
    - id: flow
      type: value
      grid: { x: 0, y: 1, w: 4, h: 2 }
      order: 4
      bind: { value: ${meter}/flow_rate }
      props: { size: large, decimals: 1, alarm_from_tag: true }
    - id: flow_bar
      type: bar
      grid: { x: 4, y: 1, w: 4, h: 2 }
      order: 5
      bind: { value: ${meter}/flow_rate }
      props: { show_alarm_limits: true }
    - id: total
      type: value
      grid: { x: 8, y: 1, w: 4, h: 2 }
      order: 6
      bind: { value: ${meter}/total_volume }
      props: { size: large, unit_display: m3, decimals: 3 }
    - id: flow_trend
      type: trend
      grid: { x: 0, y: 3, w: 12, h: 5 }
      order: 7
      props:
        range: 1h
        pens:
          - { bind: ${meter}/flow_rate,    y_axis: left,  style: line }
          - { bind: ${meter}/total_volume, y_axis: right, style: step }
    - id: meter_alarms
      type: alarm_list
      grid: { x: 0, y: 8, w: 12, h: 3 }
      order: 8
      props: { prefix: ${meter} }
    - id: adjust
      type: group
      grid: { x: 0, y: 11, w: 12, h: 3 }
      order: 9
      access: { view: operator }
      props: { title: "Maintenance", collapsed: true }
      children:
        - id: align_total
          type: control
          grid: { x: 0, y: 0, w: 6, h: 2 }
          bind: { target: ${meter}/total_volume }
          props:
            kind: setpoint
            label: "Align total with register"
            min_role: operator
            confirm: always
            require_comment: true
            reauth: true
            warning: "This adjusts the totalizer. It is recorded as a manual adjustment."
            input: { decimals: 0, unit_display: L }
    - id: diag
      type: table
      grid: { x: 0, y: 14, w: 12, h: 3 }
      order: 10
      access: { view: engineer }
      props:
        mode: prefix
        prefix: ${meter}/raw
        columns: [path, value, unit, quality, ts]
```

Notes:
- `leak` uses the semantic style `alarm_high` (06) so color comes from tokens; the label is always shown, so color is never the only signal.
- Units display as `m3` through UCUM-compatible conversion (03 section 3); `total_volume` itself stays in `L` in the tag model and history.
- `total_volume` is writable (`writable: true` in the template). A `control` bound to it is valid; the validator would warn `not_writable` on any other tag.
- The built-in `trends` page serves ad-hoc comparisons; the faceplate trend is fixed.

### 5.1 Hebrew translation block (same document)

```yaml
  i18n:
    he:
      metadata.display_name: "מד מים"
      health.props.states.0.label: "מדווח"
      health.props.states.1.label: "לא עדכני"
      health.props.states.2.label: "לא מקוון"
      leak.props.states.0.label: "זרימה רצופה"
      leak.props.states.1.label: "אין זרימה רצופה"
      align_total.props.label: "יישור סך המונה לקריאת המונה"
```

(State label indexes follow the document order shown above; the validator warns if a key no longer matches, 07 section 2.) Machine-written Hebrew is a placeholder; a Hebrew speaker reviews before any customer sees it.

## 6. Behavior by scenario

Mirrors the trace in tag-model.md section 11. Expected HMI output for the faceplate of `meter01` (browser link stays up throughout unless stated):

| Time | Event | `flow` | `total` | `health` indicator | Alarm banner | Notes |
|---|---|---|---|---|---|---|
| 10:00:00 | Device boots, NBIRTH | `--`, then `?` badge (`uncertain_initial_value`) | 1 234.567 m3 | Reporting | none | Dashed outline on flow until the 5 min window fills |
| 10:00 to 10:05 | 12 L/min | 12.0 L/min, bar ~6% | rising | Reporting | none | Trend shows pen 1 climbing, pen 2 stepping |
| 10:05 to 10:20 | Heartbeats only | decays to 0.0 L/min (good) | steady | Reporting | none | "0.0" is **normal**, not an error: health proves the device is alive |
| 10:20:00 | Power cut, NDEATH | 0.0 muted with `?` (`uncertain_last_known`) | muted with `?` | **Offline** (`abnormal`) | none yet | `last_updated` ages; hover shows quality name and `ts` |
| 10:21:00 | `health/not_ok` alarm after 60 s | same | same | Offline | MEDIUM "Water meter esp32-meter01 not reporting", unacked | Banner flashes 1 Hz until ack |
| 10:22:00 | Operator acks | same | same | Offline | steady, then clears when health returns | Ack recorded with user and comment |
| 10:25:00 | Device back (NBIRTH, new `boot_count`) | `?` initial, then good | continues from held total; reboot adds counts since boot | Reporting | alarm clears (`cleared_unacked` if not acked) | Trend shows a gap while offline (null bucket / gap rule, 03 section 5.2) |
| any | Flow never stops for `leak_duration` (2 h) | non-zero | rising | Reporting | HIGH "Flow has not stopped for 2h0m0s: possible leak" | `leak` indicator `alarm_high` |
| any | Gateway loses the broker | muted, struck (`bad_not_connected`) | same | Offline or Stale | connection chip `disconnected` | Distinct from device-quiet: struck grey vs muted `?` (06 section 4) |
| any | Browser loses the gateway | hatched, frozen | hatched | hatched | link banner after 3 s | Writes disabled; clock shows last update time (06 section 5.1) |

## 7. Acceptance tests

Run as Playwright tests against `skyrangerd` with the `sim` driver (and the real ESP32 for the manual column). The simulator's fault injection (decision 7) drives each case. "Pass" means the visible text and attributes below, asserted via accessible names and `data-testid` hooks derived from widget `id`.

| # | Test | Pass criterion |
|---|---|---|
| 1 | Login, then land on `plant_overview` | Header shows `alice · operator` and gateway-time clock; status bar shows `gw01`, config revision, HMI revision |
| 2 | Live update latency | Sim changes `pulse_count`; `flow` or `total` updates on screen within 500 ms of the gateway's `ts_gateway` plus the 10 s rate evaluation where applicable (measured on LAN) |
| 3 | Quality: device quiet | Sim sends NDEATH. `health` shows "Offline", values show the `?` badge text "Last known" (accessible name), no struck-through values |
| 4 | Quality: link down | Sim breaks the gateway's broker connection. Values struck with `✕` badge, connection chip `disconnected` |
| 5 | Browser link loss | Kill the WS (proxy). After 3 s the amber banner shows duration and retry countdown; `control` buttons disabled; on recovery all values show `resyncing` then live; no page reload |
| 6 | Leak alarm | Sim emits continuous pulses for `leak_duration` (set to `2m` in test). Banner shows HIGH; ack from the banner works; an audit record for the acknowledgement exists with the comment (action name per [../07-security/08-audit-logging.md](../07-security/08-audit-logging.md) section 1) |
| 7 | Health alarm | Sim goes silent; MEDIUM alarm after `on_delay` 60 s; clears when NBIRTH returns |
| 8 | Rollover and reboot | Sim wraps `pulse_count` at 2^32 and then reboots (new `boot_count`). `total` stays monotonic; a `skyranger.tag.counter_discontinuity.v1` event (kinds `rollover`, then `reset`) is available through `GET /events` |
| 9 | Control: success | As operator: Change... opens dialog, Cancel is focused, Apply requires a comment, `?wait=10s` returns `ok`, feedback shows new total, audit record exists |
| 10 | Control: unknown outcome | Sim delays the write beyond `10s` and drops the response: HMI shows "Outcome unknown", offers "Check status", never retries; request count to `/write` is exactly 1 |
| 11 | Role gating | As viewer: Maintenance group absent; direct `POST /tags/{id}/write` with the viewer token returns `403 forbidden` |
| 12 | Gap rendering | Sim stops history for 40 min; trend shows a gap, not a flat line (needs `history.max_interval` in the definition, 08 section 7) |
| 13 | Dense trend | 8 pens x 24 h of raw 1 s sim data render in under 1 s on the reference PC; spike present in the min/max envelope that `avg` alone would hide |
| 14 | Performance | Generated 2,000-binding screen against 5,000 `random_walk` tags: under 30% CPU, no dropped `seq`, frame time p95 under 33 ms (nightly) |
| 15 | Hebrew RTL | Language `he`: `dir="rtl"`, header and menu mirrored, numbers and units readable, trend time axis still left to right, no clipped text; screenshot diff reviewed by a Hebrew speaker |
| 16 | Accessibility | axe-core: no serious or critical violations on overview, faceplate, alarm list, confirm dialog; every control reachable by keyboard; alarm banner announces new alarms in an `aria-live` region |
| 17 | Kiosk | `/kiosk/lobby_wall` rotates every 30 s; stops rotating while a HIGH alarm is unacked; no write controls rendered; recovers after a gateway restart without manual action |
| 18 | Config change | Rename `flow_rate` in a new revision. Open screen re-resolves via `tag_ids`; an engineer sees the "renamed tags" notice; operator sees no break |
| 19 | Export | `GET /hmi` YAML, wipe, import, diff is empty |
| 20 | Offline gateway at load | Stop `skyrangerd` after the page loaded: values hatched, banner correct. Navigate to a screen not yet cached: "not available offline" |

Manual column (real ESP32): pour water through the meter and confirm liters match a measuring jug within one pulse; pull Wi-Fi and see "Offline" within `heartbeat_stale` plus UI latency; confirm a Hebrew-speaking person finds the faceplate readable.

## 8. Build order for the slice

1. `api-client` and `live` packages with the WS state machine and `TagStore` tested in Node against a recorded WS script.
2. Shell (login, header, banner, link banner, status bar) with a fake gateway.
3. `value`, `indicator`, `bar`, `last_updated`, `connection_status`, `alarm_banner`, `alarm_list`, `alarm_summary`, `table`.
4. `trend` with uPlot (raw only first; envelope after).
5. `control` with the confirm dialog and outcome handling.
6. Screen loader, validator and the two screens above; YAML pushed via `POST /hmi/drafts` (or, before the server store exists, a gateway-side directory, see 08 section 1 fallback).
7. Playwright suite, then i18n and RTL pass, then kiosk.

## Decisions made
1. The slice uses only template tags and documented components; it adds no new tag kinds, widgets or endpoints beyond the 08 proposals.
2. The faceplate is parameterized by `meter` and is the first generator recipe, so every future `pulse_water_meter` instance gets a screen for free.
3. "No flow" (0.0, health Reporting) is displayed as ordinary data; "dead device" and "link down" have distinct visuals (06 section 4).
4. Maintenance writes sit in a collapsed, role-gated group, with required comment and re-authentication.
5. A simulated second meter is part of the slice so every acceptance test can run without hardware.
6. Acceptance criteria are executable Playwright tests driven by the simulator's fault injection.

## Alternatives rejected and why
- **A P&ID-style canvas for the first screen**: more impressive, but proves nothing the grid does not, and costs the symbol library.
- **Showing leak state only as an alarm**: the `leak` indicator on the faceplate lets an operator see the condition after the alarm is acknowledged.
- **Raw pulse count on the main screen**: engineering values only; raw tags are in the engineer diagnostics table.
- **Writing the slice as TSX**: violates 02; the slice must exercise the screen format.

## Open questions
- Does `table` support row-level `on_click` navigation (section 4), or is a `link` column needed? Decide when implementing 03.
- The faceplate "Changes" tab (05 section 5) lists `write_completed` events; should it also list `counter_discontinuity` events (test 8 only checks the API)?
- Hebrew strings need review by a native speaker; the placeholders above are not approved.
- `last_updated` bound to `raw/uptime_s` shows device heartbeat time; should it use the heartbeat tag's `ts` instead (same thing here, but state which in 03)?
- Is `reauth: true` on `align_total` too heavy for the integrator's first customers? Engineers can set it to false in the document.
