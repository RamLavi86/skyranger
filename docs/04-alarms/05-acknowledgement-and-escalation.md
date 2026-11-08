# 05 Acknowledgement and Escalation

Acknowledgement is the one step that ties the alarm system to a human. It is the **end-to-end delivery guarantee**: a notification reaching a phone proves nothing; an ack proves a person has seen it. Escalation is what happens when no ack arrives.

## 1. Acknowledgement rules

| Rule | Detail |
|---|---|
| What is acked | The current activation (`active_unacked` to `active_acked`) or the return to normal (`cleared_unacked` to `normal`). Transitions in 02 section 3 rows 2 and 5 |
| Who | Role `operator` or above (api.md), and at least the alarm's `ack_role` (`operator` default, `engineer` optional). Roles are cumulative so an engineer may ack an operator alarm |
| Actor identity | The authenticated principal, recorded as `user:{name}` or `apikey:{name}`. System actors: `system:unshelve`, `system:unsuppress`, `system:shelve_expiry`, `system:config`. Nobody acks as anonymous |
| Comment | Optional, except severities listed in `Gateway.spec.alarms.ack_comment_required` (default `[critical]`), where an empty comment is rejected with `422 validation_failed` |
| Where | REST `POST /alarms/{alarm_id}/ack` and bulk `POST /alarms/ack` (api.md). No command topic on MQTT (messages.md s3). No SMS-reply acknowledgement (section 5) |
| Idempotent | Acking something not ackable returns `200` with `result: noop` per item; retries are safe |
| Audit | Every ack is in `audit_log` (api.md) and in the alarm journal (07), with the comment, actor, source IP/client, and the `seq` acked |
| Local vs remote | The HMI and an HQ system use the same endpoint. Remote acks use an API key with role `operator` and are labelled `apikey:hq-scada`, so reports can show who acks from where |

### 1.1 Acknowledge by seq (conditional ack)

An operator looking at a screen 30 seconds old must not ack an alarm that re-activated after the screen was drawn. Additive request field `expected_seq` (string, uint64):

```json
POST /api/v1/alarms/01933b81-0a1b-7c2d-8e3f-405162738495/ack
{ "comment": "Checked meter, hydrant flushing by crew", "expected_seq": "7" }
```

- Alarm `seq` equals `expected_seq`: acked.
- Differs: `409 conflict` with the current alarm state in the problem `errors` extension, and nothing is acked. The HMI refreshes and asks again.
- Absent: ack applies to whatever the state is now (scripts, curl). The HMI always sends it.

### 1.2 Bulk ack

```json
POST /api/v1/alarms/ack
{ "items": [ {"alarm_id": "...", "expected_seq": "7"}, {"alarm_id": "...", "expected_seq": "3"} ],
  "comment": "Acked after pump trip investigation" }
```

(`alarm_ids` from api.md remains valid: a list of ids without seq. Both forms are accepted; additive.)

- Maximum 100 items per call. Response per item: `acked`, `noop`, `conflict` (seq mismatch), `forbidden` (role), `not_found`.
- Comment applies to every item; required if any item's severity requires it.
- There is **no "ack all"**. The client must name the alarms. A button that acks everything is how a `critical` gets acked unseen; the HMI offers "ack visible/selected" which sends the ids and seqs on screen.
- Partial success returns `207`-style semantics inside a `200` body with per-item results (api.md uses RFC 9457 for errors only).

### 1.3 Ack and shelving, suppression, chatter

An alarm may be acked in any annunciation mode. Acking a shelved or suppressed alarm records the ack; it does not unshelve.

### 1.4 Acks and restart

Acks are journaled and persisted in the same transaction as the state change (09); an ack that returned `200` survives a power cut. An ack in flight when the gateway dies returned no response, and the client retries (idempotent).

## 2. Escalation

Escalation = "nobody has acked yet, tell someone else". It is configured in `AlarmRoute.tiers` (01 s4):

```yaml
tiers:
  - after: 0s
    to: [{ contact: oncall_phone, channels: [sms_modem, ntfy] }]
  - after: 30m
    to: [{ contact: manager, channels: [sms_modem, mail] }]
repeat: { every: 1h, severities: [critical] }
```

### 2.1 Semantics (normative)

1. A tier is a **timer anchored at the activation** (`active_at`), not at the previous notification. `after: 30m` means 30 minutes from activation.
2. At each tier time the engine checks: alarm still `active_unacked`, `annunciation == annunciated` (not shelved, suppressed, held-and-normal), the activation `seq` is the one that armed the timer. If all true, it enqueues notifications for that tier and journals `escalated` with the tier index.
3. **Ack cancels** pending tiers (and `repeat`). Condition clearing before ack also cancels pending tiers: an alarm that returned to normal is no longer urgent; the clear notification (`notify_on: clear`) tells the people already notified.
4. **Re-activation re-arms** from tier 0 with the new `active_at`.
5. **Shelve and suppress cancel** pending tiers. If the alarm is still `active_unacked` when shelve or suppression ends, tiers are re-armed from the moment it re-annunciates (`after` counted from that moment), because the operator has not yet had a fair chance.
6. `repeat` re-sends the **latest** tier's notification every `every` while the alarm stays `active_unacked`, for the listed severities only (default `critical`). A repeat is a distinct outbox row with `kind: repeat` and counts (`repeat #3`).
7. Tiers are evaluated per route. An alarm matching two routes escalates independently in each. Two routes sending to the same contact on the same channel produce **one** message (dedupe key: `(alarm_id, seq, contact, channel, kind, tier)`), not two.
8. Escalation must not depend on any external system: the timer is in the engine; the channel may be offline but the outbox row exists (06).
9. An **escalation that fails to deliver** (outbox row `dead`) raises the channel failing alarm (06), which is a different console's problem (`system`) and goes through its own route.

### 2.2 Restart behavior

Tier timers are derived, not stored: on startup, for every alarm in `active_unacked` that is annunciated, the engine computes which tiers' times have already passed relative to `active_at` and enqueues those not yet present in the outbox (`(alarm_id, seq, tier)` uniqueness constraint makes this idempotent), then arms the remaining ones. After a long outage the manager gets the tier-1 message promptly, not 30 minutes after restart. Message text includes "gateway restarted, alarm active since 00:00" (09, 11).

### 2.3 Escalation vs standing alarms

Escalation applies to **unacked** alarms only. An alarm that stays `active_acked` for hours is a *standing alarm*; it appears in KPIs (08) and optionally in a daily digest to the console owner (`notify_on: [digest]` route, 06), not as repeating pages.

## 3. Roles and who may do what

| Action | Minimum role | Extra rule |
|---|---|---|
| View alarms, journal | `viewer` | |
| Ack | `operator` (or alarm `ack_role`) | comment per severity policy |
| Shelve / unshelve | `operator` | `shelvable`, comment required, `<= max_shelve` |
| Change alarm config (limits, delays, routes, out-of-service) | `engineer` | config revision, audit |
| Change contacts, channels, `max_shelve` | `engineer` (or `admin`, see Open questions) | config revision |
| Force-clear / reset an alarm | not provided | (section 5) |

Area-based permissions ("this operator may only ack alarms of console water") are not in api.md; see Open questions.

## 4. Notification of acknowledgement

Acks produce a `state_changed` event (07) and route according to `notify_on: ack`. By default only the HQ route receives acks. An optional `notify_on: ack` on the on-call route sends "ACKED by alice 00:07" to the contacts that received the activation, so the second person does not drive to site for an alarm someone already owns. Delivery is best-effort relative to ordering: if the ack message overtakes the activation message in a slow SMS path, the activation text states the seq and the ack text states the seq, so the reader can tell.

## 5. Things deliberately not provided

- **Ack by replying to the SMS.** Anyone who can send an SMS to the modem number would be able to ack; phone numbers are not authentication. If wanted later, require a per-alarm one-time code in the SMS and the contact's registered number, and log it as `contact:oncall_phone`.
- **Force-clear / reset of an active alarm.** An active alarm is active because the condition is true. If the condition is wrong, fix the signal, shelve, or take it out of service.
- **Auto-ack on timeout.** The alarm would look handled when nobody handled it.
- **Ack from a view-only API key.** Role enforced in the API, not in the HMI.

## Decisions made
1. Ack is the end-to-end guarantee; notifications are best-effort hops, escalation covers the gap.
2. Ack is idempotent, audited, role-checked, and optionally conditional on `seq` (`expected_seq`, additive).
3. No "ack all"; bulk ack names alarms, max 100, per-item results.
4. Tiers are anchored at activation, cancelled by ack, clear, shelve, suppress; re-armed on re-activation or re-annunciation; derived (not stored) across restarts with idempotent outbox keys.
5. Critical alarms require an ack comment (configurable); system actors for automatic closes.
6. No SMS-reply ack, no force-clear, no auto-ack timeout in v1.

## Alternatives rejected and why
- **Per-notification delivery receipts as the guarantee** (SMS delivery reports, email read receipts): unreliable, not available on every path, and prove delivery to a phone, not a person.
- **Tier delay measured from the previous tier's send**: drifts when a channel retries; "30 minutes after activation" is what the rationalization states and the operator remembers.
- **Escalating acked alarms**: re-pages people about alarms that someone already owns; digest and KPI cover the legitimate need.
- **Group ack ("ack all in console")**: see above.
- **Acking via MQTT command topic**: the contract has no command topics (messages.md s3); audit and auth live in the API.
- **Server-side "last write wins" ack with no seq**: allows acking an activation the operator never saw.

## Open questions
- `expected_seq`, the `items` form of bulk ack, and per-item results are additive API changes to api.md.
- Area-based permissions: should an operator be limited to alarms of their console? api.md has global roles only. Needed for multi-customer integrator accounts.
- Who edits contacts: `engineer` (proposed) or `admin` (a wrong phone number silently breaks the alarm system, which argues for admin plus a test-send function)?
- Do we need a "second person ack" (four-eyes) for `safety` class? Not proposed for v1.
- ISA-18.2 describes concepts such as repeated annunciation and operator response timelines; specifics are UNVERIFIED and not copied.
