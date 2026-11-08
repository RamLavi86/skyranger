# 01 Alarm Configuration

Alarm definitions are part of the tag definition (tag-model.md section 8). This file keeps every contract field unchanged, adds optional fields, and defines the gateway-wide alarm settings. Every addition is **additive and optional**: a definition written against the contract alone is valid and behaves exactly as the contract says.

## 1. Contract fields (unchanged)

```yaml
alarms:
  - name: leak                 # segment grammar, unique per tag
    type: equal                # high | low | equal | not_equal | quality
    setpoint: true             # high: v > sp, low: v < sp, equal/not_equal: v ==/!= sp
    deadband: 0                # numeric types, for high/low clear
    on_delay: 0s
    off_delay: 0s
    severity: high             # low | medium | high | critical
    message: "Continuous flow detected: possible leak"
    enabled: true
```

The alarm's identity is `(tag_id, name)`, and its immutable key is `alarm_id` (UUIDv7), assigned on first apply exactly as `tag_id` is.

## 2. Full field reference (contract + proposed)

`C` = contract field. `P` = proposed additive field (needs contract amendment, see Open questions).

| Field | | Type / values | Default | Meaning |
|---|---|---|---|---|
| `name` | C | segment | required | Unique per tag |
| `id` | P | UUIDv7 | assigned | `alarm_id`. Written by export; on import matched by `(tag path, name)` if absent |
| `type` | C/P | `high` `low` `equal` `not_equal` `quality` (C); `deviation` `rate_of_change` (P) | required | See 03-alarm-types.md |
| `setpoint` | C | JSON value matching tag datatype | required unless `setpoint_ref` | Constant limit |
| `setpoint_ref` | P | tag reference | none | Limit taken from another tag (usually a `memory` tag the operator may adjust). Mutually exclusive with `setpoint` |
| `reference` | P | tag reference | required for `deviation` | The value to deviate from |
| `window`, `per` | P | duration | `1m`, `1m` | `rate_of_change` only |
| `quality_match` | P | `not_good` \| `bad` | `not_good` | `quality` type only. `not_good` is the contract behavior |
| `deadband` | C | number, engineering units, `>= 0` | `0` | Clear hysteresis, high/low/deviation/rate_of_change |
| `on_delay` | C | duration, `0s`..`1h` | `0s` | Condition must hold continuously this long before activation |
| `off_delay` | C | duration, `0s`..`1h` | `0s` | Clear condition must hold continuously this long before return to normal |
| `severity` | C | `low` `medium` `high` `critical` | required | The alarm **priority** (02, 04). Name kept from contract |
| `class` | P | `safety` `environmental` `process` `equipment` `system` `maintenance` | `process` | Classification for reporting, routing and policy |
| `message` | C | string, max 256, `${param}` interpolated in templates | required | Shown to operator and in notifications |
| `enabled` | C | bool | `true` | `false` = out of service (04 section 8) |
| `disabled_reason` | P | string | none | Required by validation when `enabled: false` |
| `ack_required` | P | bool | `true` | `false`: return to normal goes straight to `normal` (no `cleared_unacked`) |
| `ack_role` | P | `operator` \| `engineer` | `operator` | Minimum role to acknowledge |
| `shelvable` | P | bool | `true` (`false` default for `class: safety`) | Whether operators may shelve |
| `max_shelve` | P | duration | `Gateway.spec.alarms.max_shelve` | Per-alarm cap, never above the gateway cap |
| `suppress_when` | P | list of conditions (04 section 7) | none | Suppression by condition (suppressed-by-design) |
| `suppress_release_delay` | P | duration | `30s` | Hold suppression after the condition ends |
| `chatter` | P | `{count, window, hold}` | gateway default | Chatter detection override |
| `console` | P | console name | from `Gateway.spec.alarms.consoles` prefix match | Who is responsible; used for KPIs and routing |
| `routes` | P | list of route names | routes matching by selector | Notification routing override |
| `rationale` | P | `{cause, consequence, action, response_time}` | none | Rationalization record. Shown on the HMI alarm detail page. Strongly recommended for `high` and `critical` |

Validation rules (apply time, error = revision rejected):
- `setpoint` and `setpoint_ref` are mutually exclusive; one is required for `high`, `low`, `equal`, `not_equal`, `deviation`, `rate_of_change`.
- `setpoint` must be representable in the tag datatype. `high`, `low`, `deviation`, `rate_of_change` require a numeric tag (`int64`, `uint64`, `float64`). `equal`/`not_equal` accept any datatype.
- `deadband` is rejected on non-numeric types and must be smaller than `|setpoint|` span to any opposite alarm on the same tag (warning, not error).
- References in `setpoint_ref`, `reference`, `suppress_when` resolve to `tag_id` at apply time; dangling references reject the revision (same rule as derived inputs).
- `on_delay`/`off_delay` above `1h` are rejected. Longer conditions belong in derived tags (05 Decision 5, see 03 section 6).
- `enabled: false` without `disabled_reason` is rejected. The reason lands in the revision diff and audit.
- Warnings (shown in the draft diff, not blocking): tag `deadband` larger than alarm `deadband` on the same tag; `high`/`critical` without `rationale`; more than 4 alarms on one tag; severity distribution far from guidance (08-kpis.md).

## 3. Example: a well-rationalized limit alarm

```yaml
- name: pump_room_temp
  kind: source
  datatype: float64
  unit: Cel
  source: { connection: plc1, address: "modbus://1/ir/30001?type=int16" }
  scale: { type: linear, multiplier: 0.1, offset: 0 }
  deadband: { type: absolute, value: 0.1 }     # tag propagation deadband (tag-model 6)
  alarms:
    - name: high
      type: high
      setpoint: 35
      deadband: 1.0            # clears below 34.0
      on_delay: 30s
      severity: medium
      class: equipment
      message: "Pump room temperature high"
      rationale:
        cause: "Ventilation fan failed or door closed in summer"
        consequence: "Motor and VFD derate, trip above 45 Cel"
        action: "Check fan F-101, open louvres"
        response_time: 30m
    - name: high_high
      type: high
      setpoint_ref: pump_room_temp_trip_sp       # memory tag, engineer-writable
      deadband: 1.0
      on_delay: 10s
      severity: high
      class: equipment
      message: "Pump room temperature very high: drives will trip"
```

## 4. Gateway-wide settings: `Gateway.spec.alarms`

The contract shows `Gateway.spec` with a few fields and lets area docs own subtrees (the integration doc owns `sparkplug.northbound`). The alarm area owns `Gateway.spec.alarms`:

```yaml
apiVersion: skyranger/v1
kind: Gateway
metadata: { name: gw01 }
spec:
  # ... contract fields ...
  alarms:
    max_shelve: 24h                    # api.md default, hard cap for all alarms
    journal_retention: 1095d           # alarm events kept 3 years (UNVERIFIED for any regulation)
    ack_comment_required: [critical]   # severities that need a comment on ack
    chatter: { count: 3, window: 60s, hold: 10m }      # defaults, UNVERIFIED vs any standard
    flood:   { count: 10, window: 10m, digest_interval: 5m, bypass_severities: [critical] }  # UNVERIFIED
    startup_hold_timeout: 5m           # 09-restart-and-sync.md
    consoles:                          # areas of responsibility (KPIs, routing)
      - name: water
        display_name: "Water network"
        prefixes: [site1/pumphouse, site1/network]
      - name: system
        prefixes: [_system]
    contacts:                          # people who are notified; need not be login users
      - name: oncall_phone
        display_name: "On-call phone"
        sms: "+447700900123"           # E.164
        ntfy_topic: sr-gw01-oncall
      - name: alice
        email: alice@example-water.test
        sms: "+447700900456"
      - name: manager
        email: ops-manager@example-water.test
        sms: "+447700900789"
    channels:                          # how messages leave the gateway (06-notifications.md)
      - name: sms_modem
        type: sms_modem
        device: /dev/ttyUSB2
        sim_pin: secret://alarms/sim_pin
        rate_limit: { per_contact: 20, per: 1h }
      - name: mail
        type: email
        smtp_url: smtp://mail.example-water.test:587
        starttls: required
        from: "Sky Ranger gw01 <skyranger-gw01@example-water.test>"
        credentials: secret://alarms/smtp
      - name: ntfy
        type: push_ntfy
        url: https://ntfy.example-water.test   # self-hosted ntfy, or LAN instance
        credentials: secret://alarms/ntfy
      - name: scada_hq
        type: webhook
        url: https://hq.example-water.test/hooks/skyranger
        format: cloudevent
        hmac_secret: secret://alarms/hq_hmac
    routes:                            # who gets what, when (05, 06)
      - name: water_oncall
        match: { consoles: [water], severities: [medium, high, critical] }
        notify_on: [activate, clear, escalate]
        tiers:
          - after: 0s
            to: [{ contact: oncall_phone, channels: [sms_modem, ntfy] }]
          - after: 30m                 # still active_unacked after 30 min
            to: [{ contact: manager, channels: [sms_modem, mail] }]
        repeat: { every: 1h, severities: [critical] }
      - name: everything_to_hq
        match: {}                      # all alarms
        notify_on: [activate, clear, ack, shelve, unshelve]
        tiers: [{ after: 0s, to: [{ channel: scada_hq }] }]
      - name: system_health
        match: { consoles: [system] }
        notify_on: [activate, clear]
        tiers: [{ after: 0s, to: [{ contact: alice, channels: [mail, sms_modem] }] }]
    heartbeat:                         # dead-man: tells the outside world we are alive
      - channel: scada_hq
        every: 5m
      - contact: alice
        channel: sms_modem
        daily_at: "08:00"              # local time, Gateway.spec.timezone
```

Why contacts, channels and routes are config, not database rows:
- They change rarely, they must go through review (a wrong phone number silently breaks the alarm system), and they must export with the rest of the config (decision 3).
- Phone numbers and email addresses are personal data but not secrets, so they may appear in exports. SMTP passwords, SIM PINs, HMAC secrets and tokens are `secret://` references, never values.

Login users (api.md) and contacts are separate on purpose: the plumber or the duty manager often needs an SMS but has no HMI account. A contact may set `user: alice` to link to a login user, which lets the HMI show "you are on call".

## 5. Templates and alarm ids

Alarms inside a `Template` follow the same substitution rules as tags. Alarm ids of instance alarms are stable per `(instance, template tag name, alarm name)`. Export writes them next to `tag_ids` (proposed additive field):

```yaml
instances:
  - name: meter01
    template: pulse_water_meter
    params: { connection: local-sparkplug, edge_node_id: esp32-meter01 }
    tag_ids:   { total_volume: 01933b7e-8f1a-7c3d-9a2b-4f6e8d1c2b3a }
    alarm_ids: { "leak#leak": 01933b81-0a1b-7c2d-8e3f-405162738495,
                 "health#not_ok": 01933b81-0a1b-7c2d-8e3f-405162738496 }
```

The key is `{tag name}#{alarm name}`, because tag names may contain `/` and `#` is not allowed in either.

Instance `overrides` may override alarm fields by alarm name:

```yaml
overrides:
  leak:
    alarms:
      leak: { severity: critical, suppress_when: [ { tag: irrigation_active, equal: true } ] }
```

Overriding an alarm that the template does not define is a validation error (no adding alarms through overrides; add a plain tag alarm instead).

## 6. System alarms (gateway-generated)

`_system/...` tags are gateway-created and never in user config (tag-model section 1). Their alarms are also gateway-created, with fixed names and default severities that `Gateway.spec.alarms.system` may adjust (severity, routes, enabled) but not redefine:

| Tag | Alarm | Condition | Default severity |
|---|---|---|---|
| `_system/connections/{name}/state` | `not_connected` | `not_equal connected` for `30s` (degraded counts as connected) | high |
| `_system/connections/{name}/state` | `faulted` | `equal faulted` | high |
| `_system/database/state` | `unavailable` | DB not reachable for `30s` | critical |
| `_system/historian/spool_bytes` | `high` | > 80 % of spool limit | high |
| `_system/disk/used_pct` | `high` / `high_high` | > 80 / > 90 % | medium / high |
| `_system/alarms/notifier/{channel}/failing` | `failing` | `equal true` | high |
| `_system/alarms/engine/input_overflow` | `overflow` | `equal true` | critical |
| `_system/alarms/flood/{console}` | none | flood is a banner and an event, not an alarm (04 section 5) | |

```yaml
spec:
  alarms:
    system:
      - alarm: _system/connections/local-sparkplug/state#not_connected
        severity: critical
        routes: [system_health, water_oncall]
```

## Decisions made
1. Alarm definitions stay inside tag definitions (contract). All new fields are optional and additive; a contract-only definition is valid.
2. `severity` is the priority field. It is not renamed to `priority`, so we do not fork a contract name.
3. `on_delay`/`off_delay` capped at `1h`; long-running conditions are derived tags.
4. Contacts, channels, routes and heartbeats are config under `Gateway.spec.alarms`, versioned and exported with everything else; credentials are `secret://` refs.
5. Out-of-service is `enabled: false` with a mandatory `disabled_reason`, so taking an alarm out of service is always a reviewed, audited config revision (management of change).
6. Rationalization fields (`cause`, `consequence`, `action`, `response_time`) live next to the alarm so the HMI can show the operator what to do.
7. System alarms on `_system` tags are gateway-defined, adjustable but not redefinable.

## Alternatives rejected and why
- **Separate `Alarm` config kind referencing tags by path**: decouples alarms from templates and would let an alarm outlive its tag. The contract puts alarms on tags, which is also how integrators think ("the tag has a high alarm").
- **New top-level kinds `NotificationChannel`/`AlarmRoute`**: cleaner separation, but the contract lists exactly four kinds; a `Gateway.spec` subtree has precedent (northbound) and needs no new kind. Revisit if routing grows per-site in multi-gateway deployments.
- **Contacts as login users only**: excludes the people who most often need the SMS.
- **Notification config in the database, edited through a settings UI only**: not exportable, not diffable, not reviewable.
- **Expression strings for conditions (`"v > 35 && pump_running"`)**: second language, untested semantics; the tag-model contract rejected expressions for the same reason.
- **Percent deadband for alarms**: tag deadbands have it, but alarm limits are reviewed in engineering units; one form avoids confusion. Can be added later.

## Open questions
- Contract amendment needed for: `id`, `setpoint_ref`, `reference`, `window`, `per`, `quality_match`, `class`, `disabled_reason`, `ack_required`, `ack_role`, `shelvable`, `max_shelve`, `suppress_when`, `suppress_release_delay`, `chatter`, `console`, `routes`, `rationale`; types `deviation`, `rate_of_change`; instance `alarm_ids`; overrides of alarms by name.
- May user config **reference** (read-only) `_system/...` tags, e.g. in `suppress_when`? tag-model says `_system` tags are "never in user config", which we read as "never defined in user config". The water meter suppression (11) depends on referencing them. Needs a one-line contract clarification.
- Should `ack_role` become a per-area permission once api.md decides on area-based permissions?
- Regulatory retention for alarm journals in water utilities (country-specific) is UNVERIFIED; `1095d` is a placeholder.
