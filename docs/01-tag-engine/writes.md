# Write Path and Write-back to Devices

Status: design draft v0.1. Binding inputs: [api.md](../00-contracts/api.md) sections 3, 5, 7; [driver-interface.md](../00-contracts/driver-interface.md) sections 2 and 4 (writes are never retried); [tag-model.md](../00-contracts/tag-model.md) sections 2, 6, 7; [messages.md](../00-contracts/messages.md) section 4 (`write_completed`, `counter_discontinuity`).

Three kinds of tag accept writes: `source` tags with `writable: true`, `memory` tags, and `derived` tags whose function is writable (`counter_total` only). One entry point serves all three. The engine does not write to devices from the loop, it never retries, and the displayed value changes only when the new state is observed.

## 1. Sequence

```
REST POST /tags/{id}/write {value, comment}            (role operator, checked by API)
  │ API: decode JSON per the tag's datatype (api.md section 5), mint write_id (UUIDv7),
  │      record audit entry (who, tag, requested value, comment)
  ▼
Engine.Write(cmd) ── command channel ──► ENGINE LOOP
  │ 2 validate (section 2)           ── fails ──► synchronous 4xx to the API (no 202)
  │ 3 claim the one-in-flight slot (section 3) ── busy ──► 409 conflict
  │ 4 insert tag_writes row status=pending         (section 6)
  │ 5 branch by kind:
  │     source  ─► dispatch goroutine ─► driverhost.Write ─► device           (section 4)
  │     memory  ─► persist goroutine  ─► PostgreSQL memory_values             (section 5)
  │     derived ─► in loop: fn.Write, sync checkpoint                         (section 5)
  ▼
API replies 202 {write_id, status: pending}      (or waits ?wait= on the completion, section 6)
  ... outcome arrives as a command: WriteDone{write_id, result} ► loop
  │ release slot, update tag_writes, emit skyranger.tag.write_completed.v1
  ▼
Source tag value changes later, by the NORMAL sample pipeline, when the device reports it.
```

The engine keeps no optimistic copy of a written value. After a successful write to a source tag the tag still shows the old value until the driver delivers a sample with the new one. That is the point: the HMI shows what the plant reports, not what an operator hoped for.

## 2. Validation (synchronous, in the loop, before any side effect)

Order matters because the first failure is what the user sees.

| # | Check | Failure (api.md section 7) |
|---|---|---|
| 1 | Tag exists in current registry | `404 not_found` |
| 2 | `tag.Writable` and kind allows writes (`source` with driver `Capabilities.Write`, `memory`, derived with `WritableFunc`) | `409 not_writable` |
| 3 | Tag `Enabled`, connection not `faulted` or disabled | `409 not_writable` (detail says out of service) |
| 4 | Value decodes to the tag datatype exactly (no lossy conversion; int64/uint64 from a JSON number only if exactly integral within +/-2^53, api.md) | `422 type_mismatch` |
| 5 | NaN or Inf rejected for float64 | `422 type_mismatch` |
| 6 | `write_limits` low/high (on the **engineering** value the user supplied) | `422 out_of_range` |
| 7 | Source tags with `scale`: inverse scale (section 4); the raw value must be representable in the driver's raw type | `422 out_of_range` |
| 8 | Source tag, connection `disconnected`: reject immediately | `503 not_connected` |
| 9 | One-in-flight slot free | `409 conflict` |

Checks 1 to 7 and 9 are pure registry and slot lookups. Check 8 reads the connection overlay already held by the engine; the driver host would report `not_connected` too, but failing in the loop avoids a pointless gRPC call and a 10 s wait.

Write limits are validated against the engineering value; `eng_range` is **not** a write limit (it flags readings, it does not constrain setpoints).

## 3. One in flight per tag

`slot.writeInFlight uuid.UUID` (pipeline.md section 2) is non-zero from step 3 until `WriteDone` for that `write_id`.

- A second write to the same tag while one is in flight is rejected with `409 conflict`, not queued. Queued writes to a PLC can execute long after the operator's intent expired; the operator sees the rejection and decides.
- The slot is held until the driver call returns, including after the API's `?wait` has elapsed (the client got `504 timeout`, unknown outcome, but the slot stays busy until the driver's context ends, at most the write deadline, default 10 s, max 60 s). This stops a retrying client from overlapping an earlier write whose outcome is still unknown.
- Writes to different tags are independent. The driver host serializes per connection if the protocol requires it; the engine does not.
- On registry swap, a write in flight to a tag whose `(connection, address)` changed completes against the old address (the dispatch captured it). The result is reported normally.
- On shutdown, in-flight writes are cancelled; their rows are completed as `error` with code `timeout` and message `gateway shutdown, outcome unknown`. Rows still `pending` at startup (crash) are completed the same way before the engine loop starts.

## 4. Source tags: inverse scaling and the driver call

```go
type writeCmd struct {
    WriteID     uuid.UUID
    TagID       uuid.UUID
    Value       tag.Variant        // engineering value, already type-checked
    RequestedBy string             // "user:alice", "apikey:k1"
    Comment     string
    Reply       chan<- writeAccepted // synchronous validation outcome, then closed
}

func (e *Engine) startSourceWrite(s *slot, c writeCmd) {
    raw, err := s.def.Scale.Inverse(c.Value.Float())   // identity if no scale
    // convert to RawType: float raw to int types by math.Round (half away from zero), range-checked
    req := driver.WriteRequest{WriteID: c.WriteID, Address: s.def.Address, Value: rawVariant}
    deadline := e.cfg.WriteTimeout                      // default 10s, per-request override up to 60s
    go func() {
        res, err := e.drivers.Write(ctx, s.def.Conn, req)   // never retries
        e.post(writeDone{WriteID: c.WriteID, Result: toResult(res, err)})
    }()
}
```

- **Inverse scaling**: `Scale.Inverse` (pipeline.md section 5). `linear` inverts as `(eng - offset) / multiplier`; `range` inverts the interpolation. Writable tags with a non-invertible scale are rejected at compile time. Integer raw types are rounded to nearest; writing `12.34` to a point scaled by 0.1 sends raw `123`, and the tag will read back `12.3`. The API response does not claim the exact value written; the read-back shows it. A tag-level `write_resolution` is not defined (Open questions).
- **Driver outcome mapping** (driver-interface.md section 4 to api.md section 7): `ok` to `ok`; `device_error` to `502 device_error`; `not_connected` to `503`; `timeout` to `504 timeout` (**unknown outcome**: the device may or may not have acted); `not_writable`, `type_mismatch`, `not_found` to their same-name codes; anything else to `500 internal`.
- **No retries, anywhere.** Not in the engine, not in the driver, not in the API. A retried write to a PLC can repeat an action.
- **Back to the device (Sparkplug)**: the `sparkplug` driver turns a write into `NCMD`/`DCMD` for the metric. A successful return means the command was published (the contract's `Write` must not return before acknowledgement; for Sparkplug the only acknowledgement available is the device republishing the metric, and the driver contract leaves how the driver decides to the driver doc; see Open questions). The tag changes only when the device reports the new value.
- **Polled drivers** (Modbus): write acknowledged by the device, then the next poll (within `Item.Interval`) shows the value. The engine does not trigger an immediate re-read in v1; an immediate `Driver.Read` after a successful write is a cheap later optimization (Open questions).

## 5. Memory tags and derived (`counter_total`) writes

Both are state the gateway owns, so durability is part of the write.

**Memory tag** (pipeline.md section 9):
1. Validation as in section 2 (no connection checks).
2. The loop starts a persist goroutine: `INSERT ... ON CONFLICT UPDATE` into `memory_values(tag_id, value jsonb, ts)` with a 2 s timeout.
3. On success the goroutine posts `memoryPersisted`; the **loop** then stores the value with quality `good_manual`, `ts = ts_gateway = now`, propagates, and completes the write `ok`.
4. If persistence fails (database down), the write fails with `503`-class `internal` ("could not persist") and the in-memory value does **not** change. A setpoint that would silently revert after a restart is worse than a rejected write.

**`counter_total` write** (meter register alignment):
1. Validation as in section 2; `write_limits` applies (typically none).
2. In the loop: `WritableFunc.Write(ctx, v)` (derived.md section 6.1.4) sets the total, returns `Bump` and a `manual_adjust` discontinuity.
3. The write is applied **to the live state in the loop immediately**, so counts arriving after it build on the new total. The engine then requests an **immediate, synchronous checkpoint** (`FlushNow`) containing the new state and the discontinuity event (outbox, derived.md section 7) and waits for the commit off-loop, holding the one-in-flight claim. The write completes `ok` only after the commit.
4. If the commit fails, the engine applies the **inverse adjustment** (`total -= v_new - v_old`, no event) so any counts accumulated meanwhile are kept, removes the outbox event, publishes the corrected value and fails the write. Restoring the old state wholesale would lose those counts.
5. On `ok`, the new value was published with `good_manual` once; the next input-driven run returns quality to `good`. Dependents (`rate`, `continuous_activity`) see `Generation` change and reset their buffers.

Between step 3 and the commit the value is visible to readers although not yet durable. That window is bounded by the checkpoint timeout (2 s) and the failure path above repairs it. It is the one place the engine shows a value ahead of durability; memory tags avoid it by persisting first because they have no accumulating state.

## 6. Results, events, audit, status

```sql
CREATE TABLE tag_writes (
    write_id     uuid PRIMARY KEY,
    tag_id       uuid        NOT NULL,
    requested_by text        NOT NULL,
    value        jsonb       NOT NULL,       -- engineering value as requested
    raw_value    jsonb,                      -- what was sent to the device, source tags
    comment      text,
    status       text        NOT NULL,       -- pending | ok | error
    error        jsonb,                      -- {"code","message"}
    created_at   timestamptz NOT NULL,
    completed_at timestamptz
);
```

- `GET /writes/{write_id}` reads this table (the engine keeps the last 1,000 in memory to answer `?wait` without a query).
- `?wait=10s` (api.md): the API handler registers a waiter keyed by `write_id`; the loop signals it on `WriteDone`. The waiter is the handler's only link; if the client disconnects, the write is unaffected.
- Event `skyranger.tag.write_completed.v1` is emitted on every completion (`ok` and `error`, including shutdown/crash recovery completions). `value` in the event is the requested engineering value, matching the contract.
- **Audit**: the API layer writes the audit entry at request time (api.md section 2: user, time, target, before/after). The "before" is the current value at validation; "after" is the requested value. The completion event closes the loop: audit says what was asked, `tag_writes` and the event say what happened.
- Write access is role `operator` (api.md); per-tag permission is an open contract question.
- Scripting writes go through the same REST endpoint (messages.md: no command topics), so the Python runtime gets the same validation and audit for free.

## 7. Failure cases

| Case | Behavior |
|---|---|
| Device offline when write is requested | `503 not_connected` immediately (check 8); no `tag_writes` row |
| Link drops during the write | Driver returns `not_connected` or `timeout`; mapped as above; slot released at return |
| Driver process crashes mid-write | Host reports transport error; mapped to `503 not_connected`; outcome unknown is reported as `timeout` only if the request had been sent (host knows) |
| Gateway crashes mid-write | Row is `pending` at restart; completed as `error`/`timeout` ("outcome unknown") at startup, event emitted |
| PostgreSQL down | Memory writes and `counter_total` writes fail (durability required). Source writes proceed: the `tag_writes` row is written asynchronously and retried by the writer; the event is the record if the row is lost |
| Client retries after `504` | If the earlier write is still in flight: `409 conflict`. Otherwise it is a new write. There is no idempotency key in v1 (Open questions) |
| Write races a config revision that removes the tag | Write completes against the old definition; the result is reported; the tag then vanishes. Harmless |

## Decisions made
1. One write entry point and one `writeCmd` for source, memory and derived tags; the branch happens after validation.
2. Validation is synchronous and complete before any side effect, so 4xx never leaves residue.
3. Exactly one write in flight per tag, held until the driver call really ends; extra writes get `409`, never a queue.
4. No retries at any layer; `timeout` means unknown outcome and is reported that way.
5. No optimistic update: a source tag changes only when the device reports the value back.
6. Memory tag and `counter_total` writes are durable before they are acknowledged; failure to persist means the write fails and the value does not change.
7. `tag_writes` records every write, and startup completes orphaned `pending` rows as unknown-outcome errors.
8. Rejecting writes to a disconnected device immediately, rather than queueing them.

## Alternatives rejected and why
- **Queue writes per tag and send in order**: stale commands executing after the operator moved on; the plant impact is unpredictable.
- **Optimistic local update then reconcile**: shows the operator a state the device may never have reached.
- **Write-and-verify by default** (read back, compare): protocol-specific and slow; left as a driver capability (driver-interface.md open question).
- **Writes over MQTT command topics**: second auth/audit path (messages.md).
- **Acknowledge memory writes before the database commit**: loses operator setpoints on a crash.
- **Engine retries on `timeout`**: repeated actions on the device.
- **Engine writes to the driver from the loop goroutine**: a 10 s device timeout would stall every tag.

## Open questions
- The inverse-adjustment rollback for a failed `counter_total` commit (section 5) needs a property test against interleaved samples before it is final.
- CONTRACT: Sparkplug `Write` "must not return before the device acknowledged": what counts as acknowledgement for `NCMD`/`DCMD`? (The sparkplug driver doc must decide; likely "command published at QoS 1 and puback received", with read-back by the device's next `NDATA`.)
- CONTRACT: idempotency key on `POST .../write` (additive header). Useful for flaky mobile clients; not in v1.
- Should a successful write to a polled source trigger an immediate `Driver.Read` and ingest the result as a sample, for a faster HMI echo?
- Tag-level `write_resolution` or rounding statement for scaled integer registers.
- Per-tag or per-area write permission (already open in tag-model.md and api.md).
- Two-person confirmation for critical setpoints: out of scope for v1.
