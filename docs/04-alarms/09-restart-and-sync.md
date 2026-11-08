# 09 State Recovery: Restart, DB Outage, Config Changes, Sync

The test of an alarm system is what it does the night the power fails. This file defines exactly what is persisted, what is rebuilt, what a restart must never do (clear an active alarm because it forgot, or re-announce everything as new), and how state moves to a cloud/HQ observer.

## 1. What is persisted and what is derived

| Item | Where | Persisted? | On restart |
|---|---|---|---|
| Alarm state, flags, `seq`, `active_at`, `acked_*`, `shelved_until`, `activation_count`, `chattering` flag | `alarm_state` | **yes**, same transaction as the journal row | loaded, authoritative |
| Journal, events, outbox | PostgreSQL | yes | outbox rows not yet `sent` are retried (06) |
| Raw condition, `rawActive`, deadband band state | engine memory | no | rebuilt from the first usable sample |
| On-delay / off-delay timers | engine heap | no | restart from zero (never shortened) |
| Rate-of-change buffers | engine memory | no | `unknown` until window refills (03) |
| Chatter activation window | engine memory | no | restarts; the `chattering` flag itself is persisted and cleared only after a quiet `hold` since restart |
| Suppression flag | `alarm_state.suppressed` | yes | re-evaluated from suppression tags; kept until verified |
| Escalation tiers | derived | no | recomputed from `active_at` and the outbox (05 s2.2) |
| Flood flag | derived | no | recomputed from the last `window` of journal activations |
| Engine heartbeat `last_alive_at` | `alarm_engine_state` (single row) | yes, checkpoint every 10 s and at clean shutdown | defines the downtime gap |

## 2. Startup sequence (normative)

```
1  open DB; read config revision; build alarm runtime set (01 validation already passed)
2  load alarm_state rows; for each alarm still in config:
      - state, flags, seq as stored;  held = true (nothing is trusted yet);  timers empty
3  write journal row engine_gap {from: last_alive_at, to: now} for each alarm that is
   active_*/cleared_unacked/shelved, and ONE engine event 'recovered' (cause: recovered)
4  expire shelves: shelved_until <= now  ->  shelve_expired (02 s3)
5  arm remaining shelve expiries against wall clock
6  start consuming tag mailbox (tag engine replays current values: every tag emits its
   current value, quality, on start; first values may be bad_waiting_for_initial_data)
7  startup_hold window begins (Gateway.spec.alarms.startup_hold_timeout, default 5m)
8  after the window, or as soon as an alarm's inputs have each received a post-start sample,
   the alarm is released from startup hold and evaluates normally
9  recompute escalation tiers for annunciated active_unacked alarms (05 s2.2)
10 notifier resumes outbox (06); engine_gap and 'recovered' notify per route if configured
```

### 2.1 The invariants

1. **A restart never changes an alarm's state by itself.** `active_unacked` stays `active_unacked`, `active_acked` stays `active_acked`, `cleared_unacked` stays `cleared_unacked`. State changes only by evaluating a *usable* post-restart value or by an operator action.
2. **No activation or clear from an untrusted value**, including the first values after boot (`bad_waiting_for_initial_data`): the quality hold (02 s4) already freezes the alarm.
3. **A condition that cleared during downtime** is detected by the first usable sample: the alarm goes through off-delay and then clears normally, journaled with the actual (post-restart) timestamp. The journal's `engine_gap` row tells the reader the clear may have happened any time during the gap.
4. **A condition that appeared and disappeared during downtime** is invisible. This is a physical limit; the gap row makes it explicit. (If the tag engine's history has samples, an offline reconciliation tool can list excursions; not part of v1.)
5. **A condition that appeared during downtime and persists** activates after the on-delay counted from the first post-restart sample. Activation time is therefore late by downtime plus on-delay, and the journal records `condition_since = first post-restart sample`.
6. **No notification storm.** Alarms already `annunciated` before the crash, whose activation notifications were already `sent`, are not renotified. A single `recovered` summary is sent ("gateway restarted at 03:12, 2 active alarms, 1 unacked") when the downtime exceeded `recovery_notify_after` (default `2m`).
7. **Startup hold is per alarm and bounded.** An alarm whose inputs have not reported after `startup_hold_timeout` is released from startup hold: if its input is still unusable it stays `held` by the normal quality rule, and the `quality`/health alarms (never quality-held, but subject to the same startup hold, 2.2) decide whether to raise comms-loss.

### 2.2 Comms-loss alarms at startup

Liveness tags (`health`) and system `not_connected` alarms report based on current knowledge and do not wait for samples, so after a restart every device is momentarily `online = false`. The alarm `on_delay` (60 s for `health/not_ok`) and the startup hold exist for exactly this: the `not_ok` alarm is **not evaluated** until the device's inputs report or `startup_hold_timeout` elapses. Result: a device that comes back within the window never alarms; a device that does not return alarms at most `startup_hold_timeout + on_delay` after boot.

## 3. Database outage

The engine does not stop when PostgreSQL does. Alarms are safety-relevant; the plant keeps running.

```
engine loop ─► transition ─► persister
   DB up:   batch txn (<= 50 ms group commit)
   DB down: append JSONL records to spool file(s), fsync before announcing; engine continues in memory
            spool record = { "type": "transition", "event_id", "journal_row", "state_row", "outbox_rows" }
                          | { "type": "delivery_result", "outbox_id", "status", "attempts", "sent_at" }
   DB back: replayer applies spool in order, idempotent (ON CONFLICT on event_id and outbox unique key),
            deletes the spool file after the commit succeeds
```

Rules:
- "Durable before announced" holds for the spool: the fsync completes before the WS push, bus publish or SMS.
- The spool has a size limit (default 256 MB). At 80 % the `_system/historian/spool_bytes` alarm (01 s6) fires. At 100 % the engine switches to **degraded**: it keeps evaluating and notifying from memory, increments `_system/alarms/engine/spool_full`, and logs `engine_gap`-style loss markers; ack commands still succeed in memory and are kept in the ring buffer. This state raises a critical system alarm. The spool must be sized so it does not happen in a real outage (alarm records are tiny, a few hundred bytes each).
- During a DB outage the REST API for ack/shelve depends on the API's auth (also DB-backed; api.md). Operator commands cannot be authenticated without the DB. Open question below.
- If the gateway **also** restarts during a DB outage, startup loads state from the spool replayed on top of the last DB snapshot (spool is applied to an in-memory `alarm_state` before the DB is available). The engine starts, and the DB sync catches up later.

## 4. Config revision changes while running

Applying revision N+1 (config area) re-initializes the alarm runtime set **incrementally**, keyed by `alarm_id`:

| Change | Effect on running alarm |
|---|---|
| New alarm | state `normal`, evaluated after startup rules for its inputs (hold until usable). Journal `reconfigured` (created) |
| Removed alarm (or tag removed) | If active: final transition `removed` (state archived), journal row, notification `clear` suppressed (a removal is not a plant return to normal). Retained bus topic cleared |
| `setpoint`, `deadband`, `on_delay`, `off_delay`, `window` changed | Takes effect immediately: timers restart, condition re-evaluated against the current value. Active alarm stays active until the new condition clears. Journal `reconfigured` with the diff |
| `severity`, `class`, `message`, `routes`, `ack_role`, `shelvable` changed | Updates the state object in place; `seq++`; pending tiers recomputed from `active_at` under the new route. No renotify unless the new route has a tier whose time already passed (then it sends, as in 05 s2.2) |
| `type` changed | Treated as a redefinition: state forced to `normal` with cause `reconfigured`, journal records the previous state. Same `alarm_id` (the alarm's identity is `(tag, name)`) |
| `enabled: true -> false` | Out of service (02 row 13) |
| `enabled: false -> true` | Returned to service (02 row 15) |
| Tag renamed (path changed, `tag_id` same) | `alarm_id` unchanged; state object gets the new path; journal keeps the old path in old rows |
| Tag's `datatype` changed | Alarm re-validated; if no longer valid, the whole revision is rejected (01) |

Rolling back to a previous revision is just another revision; the alarm engine has no special rollback logic.

## 5. Sync to cloud or HQ ("edge decides, cloud observes")

The gateway owns alarm state. A cloud or HQ system:

1. **Receives** the event stream (`state_changed` and the two new types) through the webhook channel (06) or a future cloud sync component, and/or reads `GET /alarms`, `GET /alarms/history`, `GET /events` over a secured link.
2. **May request** an ack or shelve through the API (05), conditionally on `expected_seq`. The gateway's decision stands; a conflict returns `409` and the remote refreshes. Remote acks are labelled with the key's identity.
3. **Never writes** alarm state directly. There is no merge, no vector clock, no CRDT.

Catch-up after a link outage:
- Journal `id` (UUIDv7) is the sync cursor. The receiver keeps the last `id` it applied and asks `GET /alarms/history?cursor=<id>` (or the sync component streams). Rows are ordered by `(ts, id)`; the cursor is opaque to clients.
- The receiver de-duplicates by `event_id` (CloudEvents `id`).
- First sync or lost cursor: receiver loads `GET /alarms` (current state, snapshot) then reads the journal from its retention horizon. The snapshot has `seq` per alarm so the receiver can tell which journal rows are older than the snapshot.
- Webhook channel rows (06) that expired during a long link outage are not retried forever; catch-up through the cursor is the completeness mechanism, the webhook is the low-latency path.

During sync the gateway is not blocked: queries use their own connection pool and a read-only transaction; alarm commits are never delayed by a sync client (the persister has priority).

## 6. Clocks

- All durations in the engine use the monotonic clock. A wall-clock step (NTP correction) does not shorten or lengthen delays, escalation or shelve remaining time.
- Persisted `shelved_until` is wall time; after an NTP step it is re-armed against the new clock at restart or when the clock-step detector (monotonic vs wall drift > 5 s) fires. A large backward step cannot extend a shelve beyond `max_shelve`; the engine clamps `shelved_until` to `shelved_at + max_shelve`.
- `ts` ordering of transitions follows sample timestamps (tag-model s5). The journal also records `ts_gateway`; a device clock that is wrong has already been rejected by the skew rule.

## Decisions made
1. A restart never changes alarm state by itself; state changes only through usable post-restart samples or operator commands.
2. Delay timers and condition memory are not persisted; they restart conservatively. Alarm state, flags and `seq` are persisted in the same transaction as the journal.
3. Downtime is explicit: `engine_gap` journal rows per active alarm and a `recovered` event/summary notification; no notification storm on restart.
4. Startup hold bounds comms-loss evaluation at boot to `startup_hold_timeout + on_delay` and prevents false offline alarms.
5. The engine keeps running through a DB outage using a fsync'd spool that carries transitions and delivery results, replayed idempotently.
6. Config changes are applied incrementally by `alarm_id`; `type` changes reset state with the old state recorded; removals journal `removed`.
7. The cloud observes and requests; it never writes state. The journal `id` is the sync cursor.

## Alternatives rejected and why
- **Clearing all alarms on restart and re-evaluating**: loses the unacked flag and the history, and re-notifies everything. The 3 a.m. storm after a power blip.
- **Persisting timers**: wall-clock arithmetic across crashes for tiny benefit.
- **Re-notifying active alarms after restart**: duplicate SMS to the same people; the single `recovered` summary is enough.
- **Bidirectional alarm state merge with cloud**: ack conflicts, hard to reason about, and violates decision 1 (cloud optional).
- **Stopping alarm processing when the DB is down**: removes the alarm system exactly when infrastructure is failing.
- **Event sourcing replay at startup**: see 07.

## Open questions
- Commands during a DB outage: ack/shelve go through the API, whose auth is DB-backed (api.md: opaque tokens in the database). Options: cache validated tokens in memory for N minutes; or a local break-glass role. Needs the security area.
- `continuous_activity` (tag-model s7) state: the contract persists `counter_total` in `derived_state` but says nothing for `continuous_activity`. If its `active_since` is lost at restart, a real leak would read false after restart, the alarm would clear (after the held period), and re-fire after another 2 h. Request: checkpoint `active_since` and last increase time exactly as `counter_total` does. (11 shows both outcomes.)
- Spool-replayed state on startup while the DB is down (section 3) is a complicated path; consider failing fast (refuse to start the alarm engine until the DB is reachable, but allow it to run from the last spool) after testing.
- Sync component design (cloud) belongs to a later area; section 5 sets only the rules.
- `recovery_notify_after`, `startup_hold_timeout`, spool limit defaults are ours.
