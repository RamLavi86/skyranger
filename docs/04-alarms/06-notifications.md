# 06 Notifications

The alarm engine decides **that** someone must be told. The notifier decides **how** and guarantees that the attempt is made, recorded and retried. Channel and route configuration is in 01 section 4; escalation timing is in 05.

Honest guarantee, stated once: **the notifier guarantees at-least-once delivery to the next hop** (SMTP server accepted, HTTP endpoint returned 2xx, modem returned `+CMGS`, broker PUBACK). It does not and cannot guarantee that a human read it. The human-level guarantee is the ack plus escalation (05).

## 1. Channels

| Type (`channels[].type`) | Transport | "Delivered" means | Notes |
|---|---|---|---|
| `email` | SMTP submission, STARTTLS required by default, auth from `secret://` | SMTP `250` after `DATA` | Plain text plus a minimal HTML alternative. `Message-ID` = `<{event_id}.{contact}@gateway>`; `References` thread by alarm so clears reply to activations |
| `webhook` | HTTPS POST, `format: cloudevent` (structured JSON, messages.md s4), HMAC-SHA256 signature header | HTTP `2xx` | Header `X-SkyRanger-Signature: sha256=...` over the raw body. `Idempotency-Key` = event id. Redirects not followed. 5 s connect, 15 s total timeout |
| `mqtt` | Publish to an **external** broker (customer's or HQ's), not the internal bus | PUBACK (QoS 1) | The internal bus already carries every alarm (messages.md s3). This channel exists for a customer broker with its own topic scheme and credentials. Topic template, e.g. `plant/alarms/{severity}/{gateway_id}` |
| `push_ntfy` | HTTP POST to a self-hosted ntfy server (LAN or VPS), topic per contact | HTTP `200` | Priority mapped from severity (`critical` to max priority). Works on a LAN-only site only if phones can reach it; otherwise not a substitute for SMS |
| `sms_modem` | GSM/LTE modem on serial, AT commands (text mode or PDU) | `+CMGS: n` then `OK` | One modem, **single-flight** (one command at a time, per-modem mutex). Works with the internet unplugged: the reason it is first-class (README) |
| `sms_http` | HTTPS to an SMS gateway provider API (provider-specific adapter) | provider acceptance (`2xx` with message id) | For sites with internet but no modem; provider adapters are small plugins. Needs internet, so not a substitute for `sms_modem` as the offline path |

Choice of the first set for Sky Ranger v1: `sms_modem`, `email`, `push_ntfy`, `webhook`. `mqtt` and `sms_http` are specified here but built after the first project if not needed.

### Message content (what is on the phone)

An SMS is one segment if possible (GSM-7, 160 chars). Template (UNVERIFIED segmentation details of specific carriers):

```
[gw01] HIGH LEAK site1/pumphouse/meter01 Flow has not stopped for 2h: possible leak. Active 00:00 #7
```

Rules: gateway id first (an integrator has many sites), severity, alarm name, path, message, active time in `Gateway.spec.timezone`, `#seq`. Clear: `[gw01] CLEARED LEAK ... 03:25 (active 3h25m) #8`. Ack: `[gw01] ACKED LEAK by alice 00:07 #7`. Characters outside GSM-7 are transliterated to ASCII for SMS (messages are authored in config; the validator warns on non-GSM-7 characters in `message` for alarms routed to SMS). Emails and webhooks carry the full state object plus the rationale (cause, consequence, action).

## 2. Pipeline and the outbox

```
 engine transition ─► persister ONE transaction:
                        alarm_state upsert, events row, alarm_journal row,
                        notification_outbox rows (one per route x tier x contact x channel)
                      ─► commit ─► dispatcher per channel (worker goroutine) picks due rows
```

Because the outbox rows commit in the **same transaction** as the state change, there is no window where an alarm is recorded but not queued for notification, and none where a notification is queued for an alarm that was never recorded.

```sql
CREATE TABLE notification_outbox (
  id            uuid PRIMARY KEY,                 -- UUIDv7, also the delivery id
  event_id      uuid NOT NULL,                    -- CloudEvent id of the state change
  alarm_id      uuid NOT NULL,
  alarm_seq     bigint NOT NULL,
  route         text NOT NULL,
  tier          int  NOT NULL,
  contact       text,                              -- null for pure channel targets (webhook)
  channel       text NOT NULL,
  kind          text NOT NULL,                     -- activate|clear|ack|escalate|repeat|unshelve|shelve|digest|chatter_start|chatter_end|heartbeat|test
  payload       jsonb NOT NULL,                    -- rendered per channel at enqueue time
  not_before    timestamptz NOT NULL,
  expires_at    timestamptz NOT NULL,
  status        text NOT NULL,                     -- pending|in_flight|sent|retry|dead|expired|digested|cancelled
  attempts      int NOT NULL DEFAULT 0,
  lease_until   timestamptz,
  last_error    text,
  created_at    timestamptz NOT NULL,
  sent_at       timestamptz,
  UNIQUE (alarm_id, alarm_seq, contact, channel, kind, tier)   -- dedupe (05 s2.1 rule 7)
);
CREATE INDEX ON notification_outbox (channel, status, not_before) WHERE status IN ('pending','retry');
```

For `webhook`/`mqtt` rows with no contact, `contact` is stored as `''` (empty string) so the unique constraint applies.

### Dispatcher behavior

1. Per channel, one goroutine (the modem: exactly one; email/webhook: small pool, default 2) selects due rows ordered by `(alarm_id, alarm_seq, created_at)` with `FOR UPDATE SKIP LOCKED`, sets `in_flight` and `lease_until = now + lease` (default 60 s), sends.
2. Success: `status = sent`, `sent_at`, a journal row `notified` (compact: channel, contact, latency from `active_at`).
3. Failure: `attempts++`, `status = retry`, `not_before = now + backoff(attempts)`, `last_error`.
4. Lease expiry (process died mid-send): row returns to `pending`; this is the **duplicate window** (at-least-once). Duplicates are tolerated; webhook/MQTT receivers dedupe by event id; humans see two SMS, and the `#seq` makes it clear.
5. **Ordering**: rows for the same alarm and contact go out in `alarm_seq` order per channel; a pending older row blocks the newer one only while it is `in_flight` or within its first retry (do not let a clear overtake its activation by more than the retry delay; if still failing the clear goes ahead, annotated with the seq).

### Retry and expiry

| Attempt | Backoff |
|---|---|
| 1 to 3 | 5 s, 15 s, 60 s |
| 4 onward | 5 min, then 15 min cap, ± 20 % jitter |

`expires_at` per kind (config `notify_ttl`, defaults): `activate`/`escalate`/`repeat` of `critical`: 24 h; other severities: 6 h; `clear`/`ack`/`digest`: 2 h; `heartbeat`: 2 × interval. After expiry: `status = expired`, **not silently**: journal `notification_expired`, and the channel failing alarm counts it. A stale "leak started 7 h ago" text arriving after the modem recovers is replaced by the current state: when a row is sent after a long delay, the template prefixes `(delayed 7h)` and, if the alarm has since cleared, the dispatcher sends one combined message `ACTIVATED 00:00, CLEARED 03:25` and marks the superseded activation row `cancelled` with a reason.

### Failure alarms and channel health

Per channel the notifier maintains `failing`: true when no success for `failing_after` (default: 3 consecutive failures **and** at least 5 min), false after 2 consecutive successes. It drives `_system/alarms/notifier/{channel}/failing` (bool tag, gateway-defined alarm in 01 s6, severity `high`). That alarm routes through the `system_health` route (01), which should use a **different** channel than the one that failed (the example uses email and SMS; if the modem is the failing channel the email route reaches Alice).

Also exposed: `_system/alarms/notifier/{channel}/queue_depth`, `oldest_pending_age`, `modem/signal_dbm` and `modem/registered` (for `sms_modem`; a modem that has lost network registration is `failing` before the first send fails).

### Rate limits

`rate_limit: {per_contact: 20, per: 1h}` (01). When a contact exceeds it, further rows for non-bypass severities become one rolled-up `digest` row; `critical` is never rate-limited. A runaway alarm cannot empty the SIM's credit or get the number blocked.

## 3. Heartbeat (dead-man's switch)

The scariest failure is silence: gateway dead, modem dead, power off, and nothing says so. Heartbeats tell someone **outside** the gateway that it is alive. Config (01): `heartbeat: [{channel: scada_hq, every: 5m}, {contact: alice, channel: sms_modem, daily_at: "08:00"}]`.

- `scada_hq` (webhook) receives `skyranger.alarm.heartbeat.v1`-shaped payload (a webhook-only body, **not** a catalog event; see Open questions) every 5 min. HQ or a cheap external monitor treats **absence** of N consecutive beats as the alarm. The gateway cannot raise this alarm for itself, so the receiving side must; this is documented for integrators as a mandatory part of commissioning.
- Daily SMS at 08:00: "[gw01] alive, 0 active alarms, 2 unacked, modem -71 dBm". If it does not arrive, the person knows.
- Heartbeats go through the outbox (`kind: heartbeat`) with the short expiry, so a failing channel is visible in `failing`.
- The heartbeat message contains the active/unacked counts, config revision, uptime and journal queue depth: the same message doubles as a daily summary.
- Heartbeats never retry past their expiry and are never batched.

## 4. DB outage and the notifier

If PostgreSQL is unreachable, the persister writes the transition to the fsync'd JSONL spool (09). The notifier must still send critical SMS during a DB outage, so the dispatcher can run from the **spool-fed in-memory queue**: outbox rows destined for the DB are also pushed to an in-memory queue, delivery results are appended to the spool as `delivery_result` lines, and replay applies them. Duplicates after a crash in this mode are possible; all receivers see the same dedupe keys.

## 5. Testing a channel

`POST /api/v1/alarms/notifications/test` (proposed, `engineer`): enqueues a `kind: test` row to a contact and channel and returns the delivery id. The HMI shows the result. A phone number is never trusted until a test has been received and confirmed (commissioning checklist). Failure to deliver a test is shown immediately, not at 3 a.m.

## 6. Security

- Secrets only as `secret://` references; never in the outbox payload or logs. Log lines show `contact`, `channel`, `delivery_id`, not phone numbers (masked: `+4477*****123`).
- Webhook signing with HMAC; receivers verify and check timestamp header to limit replay (header `X-SkyRanger-Timestamp`, receivers should reject older than 5 minutes; the exact signing scheme is ours, not a standard).
- The modem accepts no inbound commands: incoming SMS are ignored in v1 (05 s5).
- Alarm text may contain operational information; SMS is not encrypted. Do not put credentials or customer personal data in `message`.

## Decisions made
1. Transactional outbox in the same transaction as the state change; at-least-once to the next hop; idempotent keys on every receiver-visible message.
2. The real end-to-end guarantee is the ack plus escalation, stated explicitly.
3. v1 channels: `sms_modem`, `email`, `push_ntfy`, `webhook`; `mqtt` and `sms_http` specified, built on demand.
4. Modem is single-flight with modem-level health tags; a lost network registration counts as failing.
5. Failed and expired notifications are never silent: statuses, journal rows, channel failing alarm routed on another channel.
6. Dead-man heartbeat is part of commissioning; the receiving side raises the alarm.
7. Stale activations are combined with the clear when delivered late; critical is never rate-limited, digested or dropped.
8. During a DB outage the dispatcher runs from the spool-fed queue.

## Alternatives rejected and why
- **Fire-and-forget notification from the engine goroutine**: lost on crash, blocks the engine on a slow SMTP server.
- **Exactly-once delivery**: not achievable across SMTP, a modem or HTTP without receiver cooperation; at-least-once with dedupe keys is honest.
- **Message broker (NATS, Kafka) between engine and notifier**: another process to keep alive; PostgreSQL is already the durable store and the outbox is a table.
- **Rely on an external alerting product (Alertmanager)**: noted in README.
- **MQTT as the only notification path** (the internal bus): requires a subscriber; an on-call phone is not one, and the bus stops if the broker stops.
- **Email-to-SMS carrier gateways**: need internet and a mail path, which are exactly what may be down.
- **Heartbeat raised by the gateway itself**: a dead gateway cannot alarm.

## Open questions
- `skyranger.alarm.heartbeat.v1` and `skyranger.alarm.delivery_failed.v1`: delivery_failed is one of the two proposed new event types (07); heartbeat as a webhook-only body would be best promoted to the catalog if HQ wants it on the bus.
- Pick the first modem (and test its AT behavior with SIM PIN, network registration and long-message splitting). Details of SMS PDU encoding and concatenation are UNVERIFIED here.
- Should email/webhook channels be failover targets for SMS in a tier (`failover:` field), or is the system_health route enough? Currently parallel channels per contact are the redundancy mechanism.
- Time zone and daylight saving for `daily_at` are `Gateway.spec.timezone` (tag-model); behavior on the DST-skipped hour needs a test.
- Voice call channel: out of scope; many on-call regimes want it.
- Webhook signature scheme (our own, above) should be documented in the integration area (docs/05?) before an HQ consumer is built.
