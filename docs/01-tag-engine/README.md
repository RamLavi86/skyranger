# 01 Tag Engine and Data Model

Status: design draft v0.1. Conforms to [../00-contracts/](../00-contracts/README.md), which is binding. Where this area found a gap or tension in a contract, it is listed under Open questions in the relevant file. Nothing here silently deviates from it.

The tag engine is the part of `skyrangerd` that turns raw driver samples into trusted, quality-stamped engineering values, computes derived tags, and hands every change to history, alarms, the internal bus, the WebSocket hub and (later) northbound Sparkplug. It is the heart of the product: if it is wrong, every chart, alarm and report is wrong.

## Files

| File | Covers |
|---|---|
| [registry.md](registry.md) | Tag registry, path resolution and prefix lookup, compiling config into a registry, template (UDT) instantiation, validation, hot reload and state carry-over |
| [pipeline.md](pipeline.md) | In-memory value store, the source-sample update pipeline stage by stage, quality propagation, timestamp selection, units and scaling, deadband, `stale_after`, connection-state effects, history gate |
| [derived.md](derived.md) | Expression language decision, derived-function interface, dependency graph, cycle detection, glitch-free evaluation order, timers, the six built-ins, `counter_total` implementation and checkpointing |
| [subscriptions.md](subscriptions.md) | Subscription model, fan-out, mailboxes, snapshot-then-update ordering, rate limiting at each layer, internal bus mirror |
| [writes.md](writes.md) | Write path from REST to device and back: validation, inverse scaling, one-in-flight rule, memory and `counter_total` writes, results and events |
| [go-design.md](go-design.md) | Go package layout, key types, the single-writer concurrency model, startup and shutdown sequences, testing hooks |
| [performance-and-failures.md](performance-and-failures.md) | Performance targets with reasoning, memory budget, failure modes and the engine's response to each |
| [water-meter-e2e.md](water-meter-e2e.md) | The ESP32 pulse water meter from NBIRTH to dashboard, including reboot, rollover, NDEATH, broker loss, gateway restart, and a manual register alignment |

Read in order: README, registry, pipeline, derived, subscriptions, writes, go-design, performance-and-failures, water-meter-e2e. The water meter file is the best single place to check that the pieces fit.

## The engine in one picture

```
 driver host (per connection, gRPC stream)          REST/API, config service, timers
      │ SampleBatch + ts_gateway                        │ commands (write, subscribe, apply revision)
      ▼                                                 ▼
 ┌──────────────── per-connection bounded queues (64 batches) ──┐   command channel (priority)
 │ fair round-robin                                              │        │
 └──────────────────────────────┬────────────────────────────────┘        │
                                ▼                                         ▼
                ┌─────────────────── ENGINE LOOP (one goroutine, single writer) ───────────────────┐
                │ 1 resolve handle -> tag     5 out-of-order check     9  derived: dirty set by rank │
                │ 2 type check / convert      6 change detect          10 history gate               │
                │ 3 scale, eng_range          7 store (atomic swap)    11 deadband -> fan-out        │
                │ 4 select ts                 8 mark dependents dirty  12 timers (stale, eval, hist) │
                └───────┬─────────────────┬──────────────────┬─────────────────┬─────────────────────┘
                        ▼                 ▼                  ▼                 ▼
                 HistorySink        alarm engine        WS hub / bus        checkpoint writer
                 (non-blocking,     (lossless mailbox)  (coalescing         (derived_state,
                  historian WAL)                         mailboxes)          memory values)
 Readers (REST GET /tags/values, snapshots) read the store lock-free via atomic pointers.
```

## Summary of key decisions

1. **One engine goroutine owns all mutable tag state** (single writer). Readers use atomic pointers to immutable values and an immutable registry snapshot. Deterministic, easy to test with a fake clock, and fast enough by a wide margin (performance-and-failures.md).
2. **Derived evaluation is batch-level and glitch-free.** All samples in a batch are applied first, then dirty derived tags are evaluated once each in topological rank order. This is required for correctness of `counter_total` when `boot_count` and `pulse_count` arrive in the same NBIRTH.
3. **No expression language in v1** (as the contract says). Derived tags are Go built-ins behind a small `derived.Func` interface, so a later `kind: script` or expression function slots in without touching the pipeline.
4. **The engine never blocks on a consumer.** Display consumers get latest-value-wins mailboxes bounded by tag count; history and alarms get lossless queues with explicit, counted overflow behavior.
5. **Gateway-timestamped samples are always "newer"** in arrival order. The out-of-order rule only applies when `ts` came from the device clock, so a gateway clock step cannot freeze current values.
6. **Configuration is compiled outside the loop** into an immutable `Registry`, then swapped atomically inside it. Runtime state is keyed by `tag_id` and survives revisions unless a defined incompatible change occurs.
7. **Writes are never queued or retried**; one in-flight write per tag; outcome reported as `skyranger.tag.write_completed.v1`. The displayed value only changes when the device reports it back.

## Decisions made
- Split the area into eight topic files plus this index; the water meter walkthrough is a separate file so it can serve as an acceptance test script.
- The seven summary decisions above, detailed in the linked files.

## Alternatives rejected and why
- **One large document**: hard for parallel reviewers and agents to reference precisely.
- **Tag engine as a separate process or service**: contradicts contract decision 4 (`skyrangerd` is one process) and adds a network hop on the hottest path.

## Open questions
- Collected per file. The ones that need a contract change are flagged "CONTRACT" in each file's Open questions, so the contract owner can find them with one search.
