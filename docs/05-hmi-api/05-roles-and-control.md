# 05 Role-Based Views, Write Permissions and Control Actions

Depends on: [api.md](../00-contracts/api.md) sections 2 (roles), 3 (writes, alarms), 5 (write encoding), 7 (errors); [driver-interface.md](../00-contracts/driver-interface.md) section 4 (writes never retried; timeout = unknown outcome); [tag-model.md](../00-contracts/tag-model.md) (`writable`, `write_limits`).

Principle: **the server is the authority; the HMI only hides what will be refused anyway and adds friction where mistakes are costly.** Hiding a button is a convenience, not a security control.

## 1. Roles in the HMI

The fixed, cumulative v1 roles from api.md: `viewer` < `operator` < `engineer` < `admin`. The HMI learns the role from `welcome.user.role` (WS) and the login response.

| Area | viewer | operator | engineer | admin |
|---|---|---|---|---|
| View screens (per `access.view`) | yes | yes | yes | yes |
| Trends, history export | yes | yes | yes | yes |
| Alarm list | read | + ack, shelve | same | same |
| `control` widgets | shown read-only | enabled | enabled | enabled |
| Designer, discovery, connections, config | hidden | hidden | yes | yes |
| Users, API keys, audit | hidden | hidden | hidden | yes |

Screen- and widget-level `access.view` (02) hides content below a role. That is UX, not confidentiality: a viewer can still read all tags via the API (api.md: viewer can read tags). If a customer needs data hidden from a role, that requires per-area permissions in the API (open question in api.md), not screen tricks.

Unknown roles (future additive enum): treat as `viewer` and show a notice.

## 2. The `control` component

All writes from screens go through one component type so that confirmation, feedback and audit behave identically everywhere.

```yaml
- id: meter_align
  type: control
  bind:
    target: ${meter}/total_volume          # must be writable (tag definition `writable: true`)
    feedback: ${meter}/total_volume        # optional; defaults to target
  props:
    kind: setpoint                         # setpoint | toggle | momentary | choice
    label: "Align total with register"
    min_role: operator                     # default operator; may be raised to engineer, never lowered
    confirm: always                        # always | never (only allowed for kind: setpoint on memory tags) | value_change
    require_comment: true                  # comment sent in write body, stored in audit
    input:
      decimals: 0
      unit_display: L
    # limits come from tag write_limits; props cannot widen them
```

| `kind` | Use | UI |
|---|---|---|
| `setpoint` | numeric/string value | current value + "Change…" button opening an entry dialog |
| `toggle` | bool target such as enable/disable | two explicit buttons with state labels ("Start" / "Stop"), never a switch that flips on tap |
| `momentary` | write a fixed value (e.g. `true` to a reset command) | one button; `props.value` |
| `choice` | `int64` with `enum_labels` | radio list in dialog |

A `control` bound to a non-writable tag renders read-only with a "not writable" note in design/preview mode (validation problem `not_writable`), never a dead button.

## 3. Confirmation flow (two-step, select-then-confirm)

Industrial HMI practice: an action needs selection, then confirmation, showing *what*, *from*, *to*, and *where*.

```
 ┌───────────────────────────────── Confirm change ────────────────────────────────┐
 │  Meter 01 · Total volume                                                          │
 │  site1/pumphouse/meter01/total_volume                                             │
 │                                                                                   │
 │     Current      1 234 630 L      (good, 10:22:00)                                │
 │     New          1 240 000 L      [ 1240000        ]   limits: 0 … 1e12           │
 │     Change       +5 370 L                                                         │
 │                                                                                   │
 │  Comment (required)  [Aligned to mechanical register reading 1240.000 m³    ]     │
 │                                                                                   │
 │  ⚠ This adjusts the totalizer. It is recorded as a manual adjustment.             │
 │                                                                                   │
 │                         [ Cancel ]                     [ Apply change ]           │
 └───────────────────────────────────────────────────────────────────────────────────┘
```

Rules:
- Focus starts on **Cancel**; Enter does not confirm. Confirm button is placed at the inline-end and requires an explicit click/tap or Tab+Space.
- The dialog re-reads the live value while open; if the current value or quality changes, it is highlighted ("value changed while you were editing").
- Writes are blocked in the dialog while `link_down` (the HMI cannot show current state, so it must not encourage action). The REST call might still work, but the operator would be acting blind.
- Client-side validation mirrors the server (datatype, `write_limits` from the tag definition); the server still validates (`out_of_range`, `type_mismatch`).
- Custom warning text: `props.warning` (translatable).
- `confirm: never` exists only for low-risk memory-tag setpoints (e.g. a display preference). The validator rejects `confirm: never` on `source` tags.
- Double-submit guard: the Apply button is disabled after the first click until the outcome arrives.

## 4. Sending the write and showing the outcome

```
 click Apply
   │ POST /api/v1/tags/{tag_id}/write?wait=10s   {"value": 1240000.0, "comment": "..."}
   ├─ 200 {"write_id","status":"ok"}             → "Applied" toast; wait for feedback tag to show new value
   ├─ 200 {"status":"error","error":{...}}       → error dialog with code text
   ├─ 202 {"status":"pending"} (wait elapsed)    → "Pending" state; poll GET /writes/{write_id} every 2 s up to 60 s;
   │                                               also listen to events subscription skyranger.tag.write_completed.v1
   ├─ 4xx problem: forbidden | not_writable | out_of_range | type_mismatch → explain, no retry offered for 403/409
   ├─ 503 not_connected                          → "Device not connected. Nothing was sent."
   ├─ 504 timeout                                → "OUTCOME UNKNOWN" state (below)
   └─ network error (no response)                → "OUTCOME UNKNOWN" state
```

The response shape of `?wait` returning `200` with an error status versus a problem response is inferred from api.md: it lists `502 device_error`, `503 not_connected`, `504 timeout` "with `?wait`", so the HMI handles both the problem responses and a `200` body with `status: "error"` defensively.

**Outcome unknown** (504, network failure, or pending beyond 60 s): this is the dangerous case. The HMI:
- Shows a persistent, non-dismissable-by-timeout notice on the control: "Outcome unknown for write at 10:31:02. Check the actual value before trying again."
- **Never retries automatically** (driver-interface.md section 4).
- Offers "Check status" (`GET /writes/{write_id}`) rather than "Retry".
- Keeps the notice until the operator dismisses it or the `write_completed` event arrives.

Feedback vs command: after `ok`, the control shows "Applied, waiting for feedback" until the `feedback` tag reflects the new value (with a tolerance for floats, `props.feedback_tolerance`) or `props.feedback_timeout` (default `30s`) passes, then shows "Feedback mismatch" in the warning style. Command acknowledgment from a driver is not the same as the process doing it.

## 5. Audit visibility

Every write is audited server-side (api.md section 2). The faceplate for a writable tag has an "Changes" tab listing recent writes for that tag (`GET /events?type=skyranger.tag.write_completed.v1&subject_prefix={path}`), so operators see who changed what without admin rights to `/audit`. (Viewer can read `/events`; `/audit` is admin only.)

## 6. Session and identity on shared stations

- Login shows username prominently in the header ("alice · operator"). On shared control-room PCs, operators must log out at shift change; a `Lock` button returns to login without losing the screen.
- **Idle step-down:** after `props`-configurable inactivity (default `15m`, setting `hmi.idle_lock` in user preferences, 08) the HMI hides write controls and requires re-entering the password for the next write (`POST /auth/login` again). Viewing continues. Kiosks run as `viewer` (06) and never write.
- Re-authentication for critical writes (`props.reauth: true`): prompts for the password in the confirm dialog and sends a fresh login before the write. This is a v1 approximation; proper "electronic signature" (21 CFR Part 11 style two-person rules) is out of scope.

## Decisions made
1. The server enforces roles; the HMI hides and disables by role as convenience only.
2. All screen writes go through a single `control` component with kinds `setpoint`, `toggle`, `momentary`, `choice`.
3. Two-step select-then-confirm dialog showing current, new, delta, path; focus on Cancel; writes blocked while the link is down.
4. Writes use `?wait=10s`; pending writes are polled and matched to `write_completed` events; timeouts and network failures are shown as "outcome unknown" and never auto-retried.
5. Command success and process feedback are tracked separately with a feedback tag and timeout.
6. Comment required by default on control writes and always for shelving; idle step-down hides writes after 15 min.

## Alternatives rejected and why
- **Writes without confirmation for operators** (one-tap): fast, but tablets and touchscreens produce accidental taps; confirmation is standard practice for anything that moves equipment.
- **Writes over WebSocket for lower latency**: forbidden by api.md, and the latency gain is irrelevant at human speeds.
- **Auto-retry on timeout**: may repeat an action on the plant.
- **Switch-style toggles**: state and action are ambiguous ("is it on, or will tapping turn it on?"). Explicit verb buttons are clearer.
- **Screen-level access as a security feature**: false sense of security; data is readable via the API.

## Open questions
- Per-tag minimum write role (tag-model open question: setpoints writable only by `engineer`). The HMI's `min_role` prop is display-only until the API enforces it; the API should own it.
- Should the confirm dialog's `reauth` become a server feature (`POST /tags/{id}/write` with a `reauth` password field or a short-lived step-up token)? Security area to decide.
- Two-person rule (second operator approval) for some writes: not in v1; would need a server-side pending-approval write state.
- Exact shape of a `?wait` write that finishes with a device error (200 with `status: error` vs `502` problem) should be pinned in the OpenAPI document.
