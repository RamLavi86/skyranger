# 04 Alarms and Events

Status: design draft v0.1. Conforms to [../00-contracts/](../00-contracts/README.md), which is binding. Where this area needs something the contracts do not yet allow, it is proposed as an **additive extension** and listed under Open questions. The alarm design does not silently deviate from any contract.

An alarm in Sky Ranger means one thing: **a condition an operator must respond to, within a time that matters, with a known action.** Anything else (a status change, a record, a hint) is an *event*. The alarm engine exists to make that distinction hold up on a real plant: few alarms, each one trusted, none lost, and none repeated until people mute the phone.

The author has lived with nuisance alarms. The design is therefore biased toward:
1. **Never losing an alarm.** Transitions are durable before they are announced, and they survive gateway restarts, DB outages and notification failures.
2. **Never crying wolf.** Deadband, delays, quality-hold, suppression by condition, chatter and flood handling are first-class, and the KPIs show where the noise is coming from.
3. **Boring mechanics.** One goroutine, a pure state-machine function, PostgreSQL transactions, an outbox table, plain SMTP, HTTP and a GSM modem.

## Files

| File | Covers |
|---|---|
| [01-configuration.md](01-configuration.md) | Alarm definition YAML (extends tag-model.md section 8), new fields, `Gateway.spec.alarms`, the proposed `NotificationChannel` and `AlarmRoute` kinds, validation, ids |
| [02-state-machine.md](02-state-machine.md) | The four contract states, orthogonal flags (shelved, suppressed, disabled, chattering), text state diagram, full transition table, quality-hold rule |
| [03-alarm-types.md](03-alarm-types.md) | `high`, `low`, `equal`, `not_equal`, `quality` (contract) plus proposed `deviation`, `rate_of_change`; comms loss, stale data, duration alarms; evaluation formulas |
| [04-nuisance-management.md](04-nuisance-management.md) | Deadband, on-delay, off-delay, chattering, flood, shelving, suppression by condition, out-of-service; what each one is for and what it must never do |
| [05-acknowledgement-and-escalation.md](05-acknowledgement-and-escalation.md) | Ack rules, who can ack, comments, bulk ack, conditional ack, escalation tiers |
| [06-notifications.md](06-notifications.md) | Channels (email, webhook, MQTT, push, SMS modem and SMS HTTP), outbox, delivery guarantees, retries, failure alarms, dead-man heartbeat |
| [07-journal-and-events.md](07-journal-and-events.md) | Event model, `skyranger.alarm.state_changed.v1` usage, journal tables, retention, queries |
| [08-kpis.md](08-kpis.md) | Alarm rate per console, floods, chattering, stale and standing alarms, top offenders; SQL and `_system` KPI tags |
| [09-restart-and-sync.md](09-restart-and-sync.md) | State persistence, gateway restart recovery, DB outage spool, config revision changes, cloud sync rules |
| [10-go-design-and-performance.md](10-go-design-and-performance.md) | Go packages and types, engine loop, timers, persistence batching, performance targets and budget |
| [11-water-meter-e2e.md](11-water-meter-e2e.md) | The ESP32 leak alarm and device-offline alarm end to end, with gateway restart, broker loss and suppression variants |

Read in order. The water meter file doubles as an acceptance-test script for the L3 harness (docs/09-testing).

## The alarm engine in one picture

```
 tag engine (single writer)                       REST /api/v1/alarms (ack, shelve)
   │ lossless mailbox: Value changes                 │ commands (with actor, comment)
   │ (post-deadband, in ts order per tag)            │
   ▼                                                 ▼
 ┌──────────────────── ALARM ENGINE LOOP (one goroutine) ─────────────────────┐
 │ 1 route value -> alarms on that tag_id (+ alarms that reference it)        │
 │ 2 quality gate (usable / hold)        5 state.Next() pure function         │
 │ 3 condition with deadband             6 flags: shelve, suppress, chatter   │
 │ 4 on/off-delay timers (heap)          7 emit Transition{seq, ...}          │
 └────────────────────────────┬────────────────────────────────────────────────┘
                              ▼
          persister: ONE PostgreSQL transaction per batch (<= 50 ms group commit)
            alarm_state upsert + events row + alarm_journal row + notification_outbox rows
            (DB down: fsync'd JSONL spool, replayed later, idempotent by event id)
                              │ after commit
          ┌───────────────────┼─────────────────────┬──────────────────────┐
          ▼                   ▼                     ▼                      ▼
   WS hub (alarms subs)  internal bus retained   notifier (outbox       KPI aggregator
                         skyranger/v1/{gw}/      dispatcher per channel: -> _system/alarms/...
                         alarms/{path}/{name}    smtp, http, mqtt, ntfy,   tags
                         + events/...            gsm modem)
```

## Summary of key decisions

1. **Contract states are the state machine.** `normal`, `active_unacked`, `active_acked`, `cleared_unacked`. Shelved, suppressed, disabled and chattering are orthogonal flags, as messages.md section 5 says. The transition function is pure and table-tested.
2. **Bad or stale input holds the alarm.** A limit alarm neither activates nor clears on a value we do not trust (`bad_*`, `uncertain_last_known`, `uncertain_initial_value`). Loss of data is alarmed separately and explicitly (quality or health alarms), never inferred by a limit alarm going quiet.
3. **Persist, then announce.** Every transition and its notification rows commit in one PostgreSQL transaction before the WS, MQTT, or any notification sees it. Notifications use a transactional outbox with at-least-once delivery and an idempotency key equal to the CloudEvent `id`.
4. **Nuisance handling never drops the journal.** Flood and chatter handling throttle *annunciation* (notifications, sounds), never recording. Everything is in the journal, and KPIs expose it.
5. **Long-running conditions belong in derived tags.** The 2 h leak is detected by `continuous_activity` (tag engine), and the alarm on it is a plain `equal true`. Alarm `on_delay` is for seconds-to-minutes debounce, not for hours of plant state.
6. **Edge decides, cloud observes.** Alarm state is owned by the gateway. A cloud or HQ system receives events and may *request* acks through the API, conditionally on the alarm's `seq`.
7. **SMS via a local GSM/LTE modem is a first-class channel**, because it is the one path that works with the internet unplugged (decision 1).

## Terminology and standards

ISA-18.2 (*Management of Alarm Systems for the Process Industries*) and IEC 62682 (understood to be its international counterpart) define the alarm-management lifecycle. Concepts used here with confidence: alarm philosophy, rationalization (every alarm has a cause, consequence, operator action and time to respond), alarm states with separate acknowledgement and return-to-normal, shelving as an operator-controlled temporary removal with a time limit, suppression by design (logic-based, e.g. equipment not running), out-of-service (removal under management of change), chattering, stale and flood alarms, and KPIs such as alarm rate per operator.

Specific state letters, numeric KPI thresholds, and exact definitions are **UNVERIFIED** and must be checked against the standard text before being quoted to a customer. Where this doc uses a number from that family of guidance (for example "10 alarms in 10 minutes" for a flood), it is a configurable default and marked UNVERIFIED. EEMUA 191 is a related guide; anything attributed to it here is also UNVERIFIED.

## Decisions made
- Split the area into eleven files plus this index; the water meter file is a runnable acceptance scenario.
- The seven summary decisions above, detailed in the linked files.
- Alarm-engine events use CloudEvents `source` `/gateways/{gateway_id}/alarm-engine` (a component name the contract already lists), including events from the notifier, which lives inside the alarm area.

## Alternatives rejected and why
- **One large document**: hard to review and reference in parallel.
- **Alarm engine as a separate service**: the contract's single-process `skyrangerd` holds; a separate service adds a network hop between the tag engine and alarms, which is exactly where alarms get lost.
- **Using an off-the-shelf alerting stack (Prometheus Alertmanager, Grafana alerting)**: built for IT metrics, with no acknowledgement-with-RTN state model, no shelving semantics, no quality, and an internet-oriented notification design. Good ideas borrowed (grouping, inhibition), product not used.
- **Implementing ISA-18.2 letter-for-letter from memory**: we would be quoting a standard we have not read in full. Concepts yes, specifics after verification.

## Open questions
- The proposed contract extensions (new alarm types and fields, two new config kinds, additive state-object fields, two new event types, two new endpoints) need a contract amendment and a decisions.md entry before implementation. Each file lists its own.
- An operator-console concept (which alarm areas a person is responsible for) is needed for "alarm rate per operator". Proposed here as `Gateway.spec.alarms.consoles`. Does the HMI or security area want to own it?
- Should the ISA-18.2 / IEC 62682 texts be purchased early so the UNVERIFIED items can be closed before the first customer project?
