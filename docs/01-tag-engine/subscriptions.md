# Subscriptions, Fan-out and Rate Limiting

Status: design draft v0.1. Binding inputs: [api.md](../00-contracts/api.md) section 6 (WebSocket), [messages.md](../00-contracts/messages.md) section 3 (internal bus), [tag-model.md](../00-contracts/tag-model.md) section 6 (deadband). Upstream: [pipeline.md](pipeline.md) stage 11.

The engine produces a stream of accepted changes. This file covers how consumers get them without ever slowing the engine loop. There are two consumer classes with different loss rules, and the split drives everything below.

| Class | Consumers | Needs | Mechanism |
|---|---|---|---|
| **Display** (latest value matters) | WebSocket hub, internal bus mirror, any future live-view client | Current state, bounded memory, no history of intermediate values | Per-subscription **coalescing mailbox**, latest value wins |
| **Lossless** (every change matters) | History sink, alarm engine, event writer | Every change, or an explicit signal that some were lost | Bounded queue with counted overflow, plus **resync from the store** where the consumer is level-triggered (section 6) |

## 1. Subscription model

```go
package subs // skyranger/internal/tagengine/subs

type Selector struct {
    Paths  []string // exact tag paths (api.md: max 5000), or
    Prefix string   // segment-wise prefix (README.md rule); "" = every tag
}

type Options struct {
    MinInterval time.Duration // per-tag rate limit, 0 = deadband only
    Name        string        // for metrics and logs: "ws:session-id:sub-id", "bus-mirror"
}

type Subscription struct {
    // Drain returns the changed tags whose rate limit allows delivery now, latest value each,
    // and the time at which the next held-back tag becomes due (zero if none).
    Drain(now time.Time) (items []*tag.Value, nextDue time.Time)
    // Ready is signalled (capacity 1, non-blocking send by the engine) when the mailbox went from empty to non-empty.
    Ready() <-chan struct{}
    // Resync returns a reason once if the subscription must be rebuilt (config_changed). The consumer then calls Hub.Snapshot again.
    Resync() <-chan ResyncReason
}

type Hub interface {
    // Subscribe registers atomically with a snapshot (section 3).
    Subscribe(sel Selector, opt Options) (*Subscription, []*tag.Value, error)
    Unsubscribe(*Subscription)
    // Snapshot re-takes a snapshot for an existing subscription (after Resync or client overflow).
    Snapshot(*Subscription) []*tag.Value
}
```

- The hub does not know about WebSocket frames, `sub_id`s or `seq`. The API layer owns the wire protocol (api.md section 6): it turns `Drain` output into `update` messages, assigns the contiguous `seq`, and enforces the 100 ms batch and send-queue bounds. The hub provides the engine-facing guarantees.
- Alarms and events WebSocket targets are not served by this hub. They have their own producers (alarm engine, event writer) but reuse `Mailbox` (section 2) so the semantics match.
- Permissions: v1 roles are global (api.md section 2); every authenticated user may subscribe to any tag.
- Unknown exact paths: the subscription is created, the unknown paths are absent from the snapshot, and they bind automatically if the path appears in a later revision. See Open questions for how the client learns of this.

## 2. Fan-out structure

All subscription state is **engine-private** and mutated only on the engine goroutine (via commands), so the fan-out lookup needs no lock on the hot path.

```go
type fanout struct {
    byTag [][]*mailboxRef      // indexed by TagIndex: who wants this tag. Rebuilt on registry swap.
}

type Mailbox struct {
    mu      sync.Mutex         // held only for pointer swaps; contention is the drainer vs the engine, microseconds
    pending map[*slot]*tag.Value // latest value per changed tag (bounded by subscribed tag count)
    due     map[*slot]time.Time  // earliest delivery time per tag when rate limited
    lastOut map[*slot]time.Time  // last delivery time per tag
    ready   chan struct{}
}
```

- **Prefix subscriptions are expanded into the per-tag index** at subscribe time and re-expanded on each registry swap. The per-change cost is one slice lookup plus one mailbox insert per interested subscription, independent of how many prefix subscriptions exist. The alternative, scanning all prefix subscriptions per change, is O(subscriptions) per change.
- **Engine side of a change** (pipeline stage 11, after the deadband passes): for each `mailboxRef` in `byTag[idx]`: lock, `pending[slot] = value`, unlock; non-blocking signal on `ready` if the mailbox was empty. The engine never waits on a consumer: the only wait is a mutex held by a drainer for the length of a map swap.
- **Drain**: lock, swap `pending` for a fresh map, unlock, then apply the rate limit outside the lock (held-back entries are merged back with latest-wins against anything newer that arrived meanwhile).
- **Memory bound**: a mailbox holds at most one pointer per subscribed tag, so a stalled consumer costs at most `tags_in_subscription * ~100 B` and is never an unbounded queue. A consumer that never drains is cut by its own layer (WebSocket send-queue overflow, api.md; bus client buffer, section 5).
- Values in mailboxes are the same immutable `*tag.Value` the store holds. Zero copies; the JSON encoding happens on the consumer's goroutine.

## 3. Snapshot-then-update ordering

Contract (api.md): `ack`, then `snapshot`, then `update`s, with no gap and no stale update after the snapshot.

`Hub.Subscribe` executes **inside the engine loop** as a command:
1. Resolve the selector against the current registry.
2. Copy the current `*tag.Value` pointers of the selected tags (pointer loads only; for 50,000 tags this is about 50,000 atomic loads, a fraction of a millisecond; the cap of 5,000 exact paths is not a concern, and a whole-gateway prefix is the worst case).
3. Create the mailbox and insert it into `byTag`.

Because the loop is the only writer, every change before the command is in the snapshot and every change after it goes to the mailbox: no gap, no duplicate. The consumer encodes the snapshot (JSON, off-loop) while updates accumulate in the mailbox, then sends `snapshot` followed by `update`s. Nothing in the mailbox can be older than the snapshot.

A client that needs a fresh snapshot (WS send-queue overflow or `resync`) calls `Hub.Snapshot`, which does steps 2 and "clear the mailbox" in one loop command, preserving the same property.

Registry swap (go-design.md section 4): the loop re-expands every subscription. A subscription whose set of resolved tags changed (tag added, removed, renamed under a prefix, or an exact path now resolving to a different tag) gets `Resync(config_changed)`; the API layer sends the WebSocket `resync` and a new `snapshot`. A swap that only changes metadata (units, display names, alarms) changes no resolved set and sends nothing from the hub; the API layer still sends `config_changed` to every client (api.md) so they refetch definitions.

## 4. Rate limiting at each layer

There is no single rate limiter. Each layer bounds the thing it can afford, and each is independently configured.

| Layer | Mechanism | Default | Where defined | What it protects |
|---|---|---|---|---|
| 1. Driver / device | Device publishes at most 1 `NDATA`/s per the profile; polled drivers use `Item.Interval`; SDK coalesces latest-wins per handle when the sink blocks | profile / `1s` | driver-interface.md, messages.md | Network, driver memory |
| 2. Engine intake | Per-connection credit of 64 batches (go-design.md section 3) | 64 | here | One chatty connection starving the others |
| 3. Propagation deadband | `deadband` on the tag, against last **propagated** value; quality changes always pass | none | tag-model.md section 6 | Everything downstream of the engine, for analog noise |
| 4. History gate | `history.mode`, `history.deadband`, `max_interval` | `on_change`, `15m` | pipeline.md section 7 | Database growth |
| 5. Subscription `min_interval` | Per-tag throttle in `Drain`: leading edge delivers immediately, then at most one delivery per `min_interval`, trailing edge delivers the latest value | `0s` | api.md section 6 | One slow dashboard asking for everything |
| 6. WebSocket batch | At most one `update` per subscription per 100 ms, latest per tag | `100ms` | api.md section 6 | Per-client frame rate |
| 7. Client send queue | 1 MiB bound; overflow drops queued updates and sends `resync` | `1MiB` | api.md section 6 | A client that is not reading |
| 8. Bus mirror | One internal subscription with its own `min_interval`, MQTT client buffer cap | `250ms`, 10,000 msgs | section 5 | The broker and external subscribers |

Details that are easy to get wrong:
- **Trailing edge is mandatory.** With throttling and no trailing edge, a value that changes at `t` and then stays constant is never delivered if `t` falls inside the throttle window. `Drain` keeps the held entry in the mailbox and reports `nextDue`, so the consumer's timer wakes it.
- **Quality changes do not bypass `min_interval`**; they are delayed by at most `min_interval`. Latest-wins means a flip and flip-back inside the window is never shown, which is acceptable for display. Alarms do not use this path (section 6), so no alarm is delayed.
- **`min_interval` has a floor** of `100ms` at the API layer (the batch interval); smaller values behave as the batch interval.
- **Deadband is not a time limiter.** A noisy tag within its deadband emits nothing; a tag moving by more than the deadband every sample emits every sample and relies on layers 5 and 6. A per-tag change-rate guard (a tag changing more than N times per second for M seconds is probably a misconfigured source) is listed as an open question. It would need a tag-level setting the contract does not have.
- The engine never rate-limits **ingest into the store**: derived tags must see every accepted change (pipeline.md decision 3).

## 5. Internal bus mirror

A small component `busmirror` is a subscriber like any other: `Subscribe(Selector{Prefix: ""}, Options{MinInterval: 250ms})`, then a loop that drains on `Ready()` or the `nextDue` timer and publishes.

- Topic `skyranger/v1/{gateway_id}/tags/{path}`, retained, QoS 0, canonical value object JSON (messages.md section 3). Per-tag publication order is `ts` order because a mailbox holds one value per tag.
- **Broker down**: the MQTT client buffers at most 10,000 messages and then drops the oldest. Because retained topics hold only the latest value, dropping old messages is harmless. On reconnect `busmirror` takes a fresh `Snapshot` and republishes every tag, since a restarted broker without persistence has lost its retained values. Mosquitto persistence settings are a deployment topic.
- **Rename and removal**: on a registry swap the engine hands `busmirror` the list of old paths to clear (`TopicsToClear`); it publishes empty retained payloads (contract). This is the only engine-to-mirror message other than the subscription itself.
- **Gateway startup**: before the first sample arrives, tags are `bad_waiting_for_initial_data`; the mirror publishes them like any other value (value `null`, quality `bad_waiting_for_initial_data`). A client reading a retained topic therefore never sees a stale `good` left from before a crash that the new process has not yet confirmed. On clean shutdown the mirror does **not** clear retained values; the `status` Will carries `online: false`.
- The mirror can be disabled by configuration without touching the engine (messages.md Open questions asks whether it should be optional; it can be, cheaply).

## 6. Lossless consumers

The engine hands accepted changes to three sinks (history at the history gate, alarms after the propagation deadband, events as they occur) through interfaces defined in go-design.md section 2. None of them blocks the loop.

**History sink** (`HistorySink.Offer(v *tag.Value, tagID, reason) bool`): enqueues into a bounded buffer (default 100,000 entries) drained by the historian writer, which has its own on-disk spill (historian doc). If `Offer` returns false the engine counts `_system/engine/history_dropped` and sets `lastHistory = nil` for that tag so its next stored value is written regardless of deadband. Dropping rows from a cumulative series loses resolution, not totals: `delta` aggregates use first and last in a bucket.

**Alarm engine** (level-triggered by design): alarms are conditions on the current value and quality, so the alarm engine needs the latest state reliably, not every intermediate. The engine feeds a bounded ring (default 65,536 entries) of `(TagIndex, *tag.Value)`. Alarm evaluation uses `on_delay` and `off_delay` against the entry's timestamps. **If the ring overflows** the engine sets an `overflow` flag and drops; the alarm engine on seeing the flag discards its queue and **re-reads the current value of every tag that has an alarm directly from the store** (lock-free loads), then continues. State is therefore never lost; only the exact instant of intermediate transitions inside the overflow window is. Counted in `_system/engine/alarm_resyncs`. The alarm engine receives values **after the propagation deadband**: alarm deadband is the alarm's own `deadband` field, and propagation deadband only removes sub-deadband noise. Alarms on a tag with a coarse propagation deadband therefore see values at that coarse resolution; a documented consequence (see Open questions).

**Event writer** (`EventSink.Emit(Event)`): non-blocking enqueue (cap 10,000). Discontinuity events do not use this path; they go through the checkpoint outbox (derived.md section 7). `write_completed` events do use it; if the queue is full the engine counts `_system/engine/events_dropped` and logs the event body, since a dropped `write_completed` is recoverable from `tag_writes` (writes.md section 6).

## Decisions made
1. Two consumer classes: display consumers get coalescing mailboxes; history, alarms and events get bounded queues with counted overflow.
2. Subscribe and snapshot run inside the engine loop, which gives gap-free, duplicate-free snapshot-then-update ordering from the single-writer property alone.
3. Prefix subscriptions are expanded into a per-tag fan-out index; the per-change cost does not depend on the number of subscriptions.
4. Throttling is leading-edge plus trailing-edge latest-wins; mailbox memory is bounded by tag count.
5. The alarm engine is level-triggered: overflow becomes a resync from the store, never a silent loss.
6. Rate limiting is layered (device, intake credit, deadband, history gate, subscription interval, WS batch, send queue, bus mirror), with no global limiter.
7. The bus mirror is an ordinary subscriber, republishes everything after a broker reconnect, and clears retained topics on rename or delete.

## Alternatives rejected and why
- **A Go channel per subscription carrying every change**: unbounded or lossy; a slow client either stalls the engine or silently misses changes.
- **Engine writes directly to WebSocket connections**: couples the single writer to network speed.
- **Take the snapshot outside the loop and reconcile with version numbers**: more machinery than copying pointers in the loop.
- **Scanning prefix subscriptions per change**: O(subscriptions) per change.
- **Lossless queue for alarms with an unbounded buffer**: memory risk; level-triggered resync is bounded and loses no state.
- **Rate limiting before derived inputs**: would make totals depend on display settings (pipeline.md).
- **Leading-edge-only throttling**: final values can be dropped.

## Open questions
- CONTRACT (api.md): `subscribe` with exact `paths` containing unknown paths: should `ack` carry `unknown_paths: [...]` (additive field)?
- CONTRACT (api.md): is the `min_interval` floor of 100 ms acceptable to document, or should lower values be rejected with `bad_request`?
- Should the alarm engine see values before the propagation deadband, so a coarse display deadband cannot hide a limit crossing? Proposal: yes for tags with alarms (feed the alarm ring at stage 9, not stage 11). Costs a few more ring entries; avoids a subtle misconfiguration.
- A per-tag change-rate guard needs a tag-level field (for example `max_rate`); not in the contract.
- Bus mirror interval (250 ms) and client buffer (10,000) are guesses to be tuned against a real broker.
- Should WebSocket subscriptions with `prefix: ""` require the `engineer` role, because they are the most expensive request? Not in the contract.
