# 06 Operator Ergonomics, Degraded Behavior, Mobile and Kiosk

Depends on: [tag-model.md](../00-contracts/tag-model.md) section 4 (quality), [messages.md](../00-contracts/messages.md) section 5 (alarm states), [01-architecture.md](01-architecture.md) (link state, stale flags), [03-components.md](03-components.md).

## 1. High-performance HMI principles we adopt

Sources of the ideas: the "High Performance HMI" practice (Hollifield et al., *The High Performance HMI Handbook*) and ISA-101 (HMI for process automation). We follow their spirit; we have not checked our design clause-by-clause against ISA-101 (UNVERIFIED).

1. **Grey is normal.** Normal operation looks calm: grey background, dark grey equipment, black/dark text. Color appears only when something needs attention.
2. **Color has one meaning each.** Alarm colors are used only for alarms. Never for equipment state (no "green = running, red = stopped").
3. **Equipment state by shape and fill, not hue.** Running pump: filled dark grey. Stopped: outline. Fault: alarm shape overlay.
4. **Analog over digital where it helps.** Bars with normal range marks and sparklines show "where is it relative to normal and which way is it going", which a bare number does not.
5. **Hierarchy of displays.** Level 1 overview (whole site, KPIs, alarm summary) -> Level 2 area/unit (P&ID-ish) -> Level 3 detail/faceplate -> Level 4 diagnostics (prefix table, raw tags, connection health). Navigation is shaped accordingly (02, `Navigation`).
6. **Data quality always visible.** No value is ever shown without its quality and freshness being discoverable at a glance.
7. **Minimal motion.** Movement only for unacknowledged alarms and actual flow (03, section 7.2).

## 2. Color tokens

Defined once in `web/packages/tokens` as CSS custom properties; screens refer to semantic style names (02, section 4.1), never to hex values. Customers can override the token file; the HMI validates contrast (section 2.3) on load in engineer mode.

### 2.1 Neutral palette (light theme, default for control rooms with daylight)

| Token | Use | Value (starting point) |
|---|---|---|
| `--sr-bg` | screen background | `#D9D9D9` (light grey, not white: less glare) |
| `--sr-surface` | widget background | `#E6E6E6` |
| `--sr-text` | primary text | `#1A1A1A` |
| `--sr-text-muted` | labels, units | `#4D4D4D` |
| `--sr-equip` | equipment line/fill (running = fill) | `#333333` |
| `--sr-equip-off` | stopped/inactive outline | `#808080` |
| `--sr-pipe` | process lines | `#666666` |
| `--sr-normal-band` | normal range band on bars | `#BFBFBF` |
| `--sr-pen1..8` | trend pens | 8 distinguishable non-alarm hues (blues, purples, teals, browns), checked for color-vision deficiency |

Dark theme (night shift, dark control rooms) uses the same token names with dark-grey background (`#262626`) and light text. Theme follows a user preference, not the OS, because control-room PCs are often shared.

### 2.2 Alarm and state palette

| Style token | Meaning | Color | Shape | Text |
|---|---|---|---|---|
| `alarm_critical` | severity `critical` | magenta/purple `#B000B0` | octagon | "CRITICAL" |
| `alarm_high` | severity `high` | red `#D00000` | triangle | "HIGH" |
| `alarm_medium` | severity `medium` | amber/orange `#E07000` | diamond | "MEDIUM" |
| `alarm_low` | severity `low` | yellow `#E0C000` with dark outline | circle | "LOW" |
| `warning` | non-alarm attention (stale device, feedback mismatch) | yellow-tan outline only, no fill | none or "!" | descriptive |
| `abnormal` | off-normal state that is not an alarm | dark text on `--sr-surface`, bold outline | none | descriptive |
| `normal` | default | neutrals | none | |
| `open`/`closed`/`running`/`stopped` | equipment state for symbols | fill vs outline in neutrals | per symbol | in tooltip/a11y |
| `quality_uncertain` | uncertain quality | neutral, dashed outline | "?" | quality name |
| `quality_bad` | bad quality | neutral grey, solid outline, value struck | "✕" | quality name |

Severity names are those of tag-model.md section 8 (`low | medium | high | critical`). Mapping magenta to critical and red to high follows common high-performance HMI practice; customers with existing site standards (e.g. red = critical) can remap via tokens.

Alarm state presentation (messages.md states):

| `state` | Presentation |
|---|---|
| `active_unacked` | severity color fill on shape, shape blinks 1 Hz |
| `active_acked` | severity color fill, steady |
| `cleared_unacked` | severity color outline only, steady, "cleared" text, still in banner |
| `normal` | not shown in banner/list (list filter can show) |
| shelved (`shelved_until` in future) | hidden from banner; list shows with "shelved until" and a muted style; a "N shelved" counter is always visible |

### 2.3 Contrast and color-vision rules

- Text contrast at least 4.5:1 against its background (WCAG 2.x AA), alarm shapes 3:1 against background (non-text contrast).
- No information by color alone: every alarm has shape and text, every quality state has a badge and text (07).
- The pen palette and alarm palette are checked with a color-vision-deficiency simulator in CI (Playwright screenshot + simulated deuteranopia/protanopia filter, manual review in v1).

## 3. Layout conventions

```
┌────────────────────────────────────────────────────────────────────────────┐
│ [≡] Sky Ranger · Example Water Co.   Overview › Pumphouse › Meter 01    alice·operator  10:31:05 (Europe/London) │  header
├────────────────────────────────────────────────────────────────────────────┤
│ ▲ HIGH 12:00:30 Meter 01 / Continuous flow  Flow has not stopped… [Ack] +3 │  alarm banner (always)
├────────────────────────────────────────────────────────────────────────────┤
│ (link banner appears here when degraded, section 5)                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         screen content                                      │
│                                                                             │
├────────────────────────────────────────────────────────────────────────────┤
│ gw01 · rev 43 · hmi rev 12 · connections: local-sparkplug ● sim ●          │  status bar
└────────────────────────────────────────────────────────────────────────────┘
```

- Header clock shows **gateway time** in the gateway timezone (computed from server-time offset, 08), because operators correlate with plant events, not browser time.
- Alarm banner and status bar are on every screen, including kiosks; they cannot be hidden by screen config.
- Minimum touch target 44x44 CSS px for any control on touch devices; 32 px on desktop.
- Typography: one sans-serif family bundled (an open-licensed font with Latin and Hebrew coverage, candidate Noto Sans + Noto Sans Hebrew, licenses OFL), tabular figures for numbers, minimum value text 14 px desktop, 16 px touch.

## 4. Data quality and freshness display

The operator must be able to distinguish, at a glance, four situations from tag-model.md:

| Situation | Server says | HMI shows |
|---|---|---|
| Live, flowing | `good`, value > 0 | normal value |
| Live, no flow | `good`, value 0.0, health 2 | normal "0.0 L/min" |
| Device quiet (dead device, broker up) | `uncertain_last_known`, health 0/1 | muted value + "?" badge, health indicator "Stale"/"Offline" in `warning`/`abnormal` style |
| Link down (gateway lost broker/PLC) | `bad_not_connected` | struck grey last value + "✕", connection chip in status bar shows `disconnected` |
| Browser lost gateway | (nothing arrives) | link banner + all values hatched, header clock shows "last update 10:07:12" |

Hover/tap on any value opens a small details popover: quality name (translated), `ts`, `ts_source`, `ts_gateway`, age, path, unit, a "Trend" link. This is the operator's tool for "do I trust this number?".

Age display: values show "age" only when older than a threshold (default: 2x the tag's `stale_after` if set, else none), to avoid noisy "3 s ago" everywhere.

## 5. Offline and degraded behavior

### 5.1 Link-down banner

Driven by `LinkStatus` (01, section 5.1):

| Link state | Banner | Values | Controls |
|---|---|---|---|
| `live` | none | live | enabled per role |
| `backoff`/`connecting` < 3 s | none (avoid flicker on brief blips) | marked `link_down` immediately (hatched) | disabled immediately |
| `backoff`/`connecting` >= 3 s | **amber full-width banner**: "No connection to gateway since 10:07:12 (2 min). Values are frozen. Retrying in 8 s. [Retry now]" | hatched, last values kept | disabled |
| `authenticating` after reconnect | thin "Reconnecting…" strip | `resyncing` | disabled |
| `needs_login` | login dialog over screen | hatched | disabled |
| gateway reports `degraded` (`/system/health`) | grey strip: "Gateway degraded: database unavailable, history not being saved" | live | enabled |

The banner color is `warning`, not an alarm color: losing the HMI link is not a process alarm, and operators must not confuse it with one. The banner states the duration and the age of the data, because "frozen" values that look live are the worst failure an HMI can have.

### 5.2 Gateway health polling

`GET /system/health` every 30 s (unauthenticated, cheap). `checks.database != ok` shows "history not being recorded"; `checks.broker != ok` or `checks.drivers != ok` shows connection problems, with details from `GET /connections` in the status bar popover.

### 5.3 What still works offline from the gateway

- The SPA is loaded; the current screen keeps rendering with frozen, clearly marked values.
- Navigation to screens already cached (TanStack Query cache) works; others show "not available offline".
- No writes, acks or shelving (they need the gateway).
- A **service worker** is not used in v1: the HMI is always served by the gateway, and a stale cached HMI version after a gateway upgrade is a worse problem than not loading when the gateway is down (when the gateway is down there is no data to show anyway).

### 5.4 Server-side degraded cases the HMI must render correctly

- History query fails (database down): trend shows the live portion only with "History unavailable" overlay.
- `resync` storms (repeated overflow): if more than 3 `resync`s per minute on one subscription, the HMI raises its `min_interval` for that sub to `1s` and shows a small "reduced update rate" note to engineers.
- Config changed while an operator is on a screen: the screen re-resolves silently; if bindings became dangling, affected widgets show config-error style, the rest keeps running.

## 6. Mobile and tablet

- Grid screens reflow to one column below 600 px (02). The alarm banner collapses to a counter chip with severity shape; tap opens the alarm list.
- Canvas (P&ID) screens: pinch-zoom and pan; double-tap a symbol to open its faceplate popup. Recommend integrators provide grid summaries for phones.
- Writes on phones: allowed for operators, but the confirm dialog is full-screen and the Apply button is separated from Cancel by at least one button height.
- Connectivity: phones on plant Wi-Fi drop often; the 15 s app ping (01) and 3 s banner threshold make this visible quickly.
- Remote access over the internet is not an HMI feature; it is a VPN/security-area decision. The HMI assumes it is reached on the LAN or via VPN.
- Native apps are not planned; the SPA is responsive. "Add to home screen" requires a web app manifest (provided) but not a service worker for basic install on Android (UNVERIFIED; some browsers require a service worker for installability).

## 7. Kiosk mode

URL: `/kiosk/{kiosk_name}` from `Navigation.spec.kiosks` (02) or `/s/{screen}?kiosk=1`.

- Hides navigation, menus, designer; shows header (clock, site), alarm banner, status bar.
- Runs as a dedicated `viewer` API identity. Login with an API key is supported for kiosks: the kiosk page accepts an API key once via a setup screen and keeps it in `localStorage` **only in kiosk mode** (an exception to api.md's `sessionStorage` guidance, needed so a kiosk survives reboots; see Open questions). The key must have role `viewer`.
- Rotation per `kiosks.{name}.screens` with `dwell`; `pause_on_alarm` stops rotation on unacked high/critical alarms.
- Auto-recovery: full page reload if the link has been down > 10 min and comes back (picks up a new HMI version after gateway upgrade); reload daily at 03:00 gateway time to clear any memory growth.
- Burn-in: on OLED/plasma walls, a 1 px content shift every hour (`props.pixel_shift`), off by default.
- Browser setup (Chromium `--kiosk`, autoplay flags for alarm audio, auto-login OS user) is documented by the deployment area; flag names UNVERIFIED.

## Decisions made
1. High-performance HMI visual language: grey normal, color only for alarms and attention, state by shape/fill, analog indicators and sparklines, four-level display hierarchy.
2. Semantic color tokens with a fixed alarm palette (critical magenta, high red, medium orange, low yellow) plus shape and text per severity; customer-remappable.
3. Link-down banner after 3 s, in `warning` style (not an alarm color), stating time since last data; values hatched and controls disabled immediately.
4. Quality/freshness popover on every value; age shown only beyond a threshold.
5. No service worker in v1.
6. Kiosks are `viewer` identities with optional API-key login persisted in `localStorage` in kiosk mode only; rotation pauses on serious alarms; periodic self-reload.
7. Header clock and all times in gateway time and timezone.

## Alternatives rejected and why
- **Traditional "green running, red stopped" coloring**: makes red mean two things (stopped and alarm), and floods normal screens with saturated colors that hide real alarms.
- **Red link-down banner**: competes with process alarms for the same attention channel.
- **Hiding values on link loss**: operators lose context; frozen-but-marked values with a timestamp are more useful.
- **Service-worker offline app**: risk of serving an old HMI against a newer API; little benefit because data needs the gateway.
- **Dark theme only**: many control rooms are bright; offer both, light default.

## Open questions
- Kiosk API key in `localStorage` deviates from api.md's guidance (token in memory and `sessionStorage`). Security area must accept this, or provide a device-bound kiosk enrollment flow instead.
- Should alarm audio be part of v1 (needs autoplay handling) or deferred?
- Exact palette hex values must be validated for contrast and color-vision deficiency on real displays; values above are starting points.
- ISA-101 conformance claims: none made; review against the standard text before marketing anything as "ISA-101 aligned" (UNVERIFIED).
- Font choice: confirm Noto Sans Hebrew coverage and OFL license terms for bundling (UNVERIFIED).
