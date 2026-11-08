# 11 Go Storage Interfaces

Status: draft. Go package paths use the placeholder module name `skyranger/...` (the real module path is an open contract question). The goal is narrow: **a second database can be added without touching the tag engine, the alarm engine, the config service or the API handlers**, and the first database (PostgreSQL with TimescaleDB) does not leak its types into them. The goal is not to make every backend equal: a history-only backend is a first-class use.

## 1. Package layout

```
pkg/storage/                 interfaces, plain data types, errors, Capabilities, backend registry      (imported by everyone)
pkg/storage/storagetest/     conformance suite every backend must pass                                  (imported by backend tests)
internal/storage/pg/         shared pgx code for the "core" schema: config, catalog, events, alarms, audit, state
internal/storage/timescale/  history on hypertables, continuous aggregate, policies (embeds pg)
internal/storage/plainpg/    history on declarative partitions + Go-maintained rollup (embeds pg)  [built on demand]
internal/historian/          Sink, ring, writer, replayer, flood guard          (depends only on pkg/storage and internal/spool)
internal/spool/              disk spool (04)
internal/canonjson/          canonical JSON shared by config hashing and the audit chain
```

Dependency rule, enforced by an import-lint in CI: nothing outside `internal/storage/*` imports `pgx` or `database/sql`. The tag engine, alarm engine, config service and API see only `pkg/storage`.

## 2. The `Store`

```go
package storage

// Store is the whole persistence surface of skyrangerd. One Store per process.
type Store interface {
    History() HistoryStore
    Catalog() CatalogStore
    Config() ConfigStore
    Events() EventStore
    Alarms() AlarmStore
    State() StateStore
    Audit() AuditStore
    Admin() Admin

    // WithTx runs fn in a single database transaction spanning every repository EXCEPT History.
    // fn may be retried by the store on serialization failures, so it must be idempotent and free of side effects
    // other than calls on tx. A returned error rolls back; ErrConflict / ErrUnavailable are surfaced unchanged.
    WithTx(ctx context.Context, fn func(ctx context.Context, tx Tx) error) error

    Capabilities() Capabilities
    Ping(ctx context.Context) error
    Close() error
}

// Tx exposes the transactional repositories. History is deliberately absent (decision 5 in the README).
type Tx interface {
    Catalog() CatalogStore
    Config() ConfigStore
    Events() EventStore
    Alarms() AlarmStore
    State() StateStore
    Audit() AuditStore
}

type Capabilities struct {
    Transactions       bool   // WithTx is real (false would be a degraded backend and is not supported for core)
    NativeAggregates   bool   // server maintains the 15-minute rollup (Timescale: true; plainpg: false, maintained by Admin)
    Compression        bool
    TimezoneBuckets    bool   // server-side calendar buckets; if false the query layer does them in Go
    MaxBatchRows       int    // largest History.Write batch the backend accepts
    ExactUint64        bool   // can store the full uint64 range (all backends here use the text escape hatch, so true)
}
```

Why `History()` is outside `WithTx`: the historian writes thousands of rows per second in its own cadence; coupling it to alarm or config transactions would let a slow alarm commit stall history or the reverse. It also makes a history-only backend composable (section 7).

## 3. History

```go
package storage

type HistoryStore interface {
    // Write inserts samples idempotently on (TagID, TS): a duplicate is ignored, never an error and never an overwrite.
    // The batch may be in any order. Returns how many rows were new.
    Write(ctx context.Context, batch []history.Sample) (WriteResult, error)

    ReadRaw(ctx context.Context, q RawQuery) (RawResult, error)
    ReadAgg(ctx context.Context, q AggQuery) ([]Series, error)
    Boundary(ctx context.Context, tagID uuid.UUID, before time.Time, lookback time.Duration) (*Point, error)
    Gaps(ctx context.Context, q GapQuery) ([]Gap, error)
    Export(ctx context.Context, q ExportQuery, w RowWriter) error
    Stats(ctx context.Context) (HistoryStats, error)

    // HistoryAdmin is the maintenance side: policies, rollup refresh, emergency pruning.
    HistoryAdmin
}

type HistoryAdmin interface {
    EnsurePolicies(ctx context.Context, p Policies) error                       // idempotent: retention, compression delay, rollup retention
    RefreshAggregates(ctx context.Context, from, to time.Time) error            // after replay or restore; no-op if the backend has none
    PruneOldest(ctx context.Context, r PruneRequest) (PruneResult, error)       // disk ladder (08): drop whole oldest raw ranges, report exactly what was dropped
    DiskUsage(ctx context.Context) (DiskUsage, error)
}

type Policies struct {
    RawRetention       time.Duration
    AggregateRetention time.Duration
    CompressAfter      time.Duration
    ChunkInterval      time.Duration // applies to new chunks/partitions only
}

type PruneRequest struct {
    KeepAtLeast time.Duration // the raw floor (min_raw_retention)
    TargetBytes int64         // stop when the history footprint is at most this
}
type PruneResult struct {
    DroppedFrom, DroppedTo time.Time
    Chunks                 int
    BytesFreed             int64
}

type WriteResult struct{ Submitted, Inserted int }

type RawQuery struct {
    TagIDs []uuid.UUID
    After  map[uuid.UUID]time.Time // per-series keyset position; zero = from Start
    Start, End time.Time          // [Start, End)
    Limit  int                    // per series
    Detail bool                   // include TSSource, TSGateway, Flags
}
type RawResult struct {
    Series    []Series
    Truncated map[uuid.UUID]time.Time // series that have more rows, with the last returned ts
}

type AggFunc uint8
const (AggAvg AggFunc = iota + 1; AggMin; AggMax; AggFirst; AggLast; AggCount; AggDelta; AggTimeWeightedAvg)

type AggQuery struct {
    TagIDs   []uuid.UUID
    Start, End time.Time
    Func     AggFunc
    Interval time.Duration     // multiple of 1 second; the planner decides raw vs rollup
    TZ       *time.Location    // calendar alignment for intervals of one day or more
    Carry    bool              // AggDelta only: carry-forward (10, section 7)
    Hold     map[uuid.UUID]time.Duration // per-tag hold limit (1.5 x max_interval), for TimeWeightedAvg and gap-aware queries
}

type Series struct {
    TagID  uuid.UUID
    Points []Point
}
type Point struct {
    TS        time.Time
    V         tag.Variant   // zero Variant = null
    Quality   tag.Quality   // for aggregates: worst usable class, or QualityBad when nothing was usable
    TSSource  time.Time     // Detail only
    TSGateway time.Time     // Detail only
    Flags     history.Flags // Detail only
}
```

`Variant` and `Quality` are the contract types from `skyranger/pkg/tag` (tag-model.md section 3 and 4). Interval durations are Go durations internally; the API layer converts the contract's `1d` style strings (README open question 1).

`history.Sample` is defined in `pkg/history` so that both the engine (producer) and storage (consumer) import it without importing each other:

```go
package history

type Flags uint16
const (
    FlagKeepAlive  Flags = 1 << 0
    FlagOutOfOrder Flags = 1 << 1
)

type Sample struct {
    TagID     uuid.UUID
    TS        time.Time   // effective timestamp, UTC, millisecond precision
    TSGateway time.Time
    TSSource  time.Time   // zero if absent
    Quality   tag.Quality
    Flags     Flags
    V         tag.Variant
}
```

The codec between `Sample` and the three value columns (01, section 4) lives in `internal/storage/pg` as `encodeValue`/`decodeValue`, tested with every datatype, `uint64` at 0, 2^63-1, 2^63 and 2^64-1, `float64` extremes and `bool`. A new backend reuses it or has its own and passes the same conformance cases.

## 4. The other repositories

```go
type CatalogStore interface {
    Resolve(ctx context.Context, paths []string) (map[string]CatalogRow, error)   // live tag wins, else most recently removed
    Get(ctx context.Context, ids []uuid.UUID) (map[uuid.UUID]CatalogRow, error)
    ListPrefix(ctx context.Context, prefix string, includeRemoved bool, limit int, after string) ([]CatalogRow, error)
    Upsert(ctx context.Context, rows []CatalogRow, revision int64) error          // used by ConfigStore.Apply, exposed for the migrator and tests
    MarkRemoved(ctx context.Context, liveIDs []uuid.UUID, revision int64) (int, error)
}

type ConfigStore interface {
    Head(ctx context.Context) (RevisionMeta, error)                                // zero revision if none
    Revision(ctx context.Context, n int64) (RevisionContent, error)
    ListRevisions(ctx context.Context, before int64, limit int) ([]RevisionMeta, error)
    // Apply must be called inside WithTx together with the audit append and the event (05, section 2),
    // or alone (the store opens its own transaction and does the audit append via ApplyRequest.Audit).
    Apply(ctx context.Context, req ApplyRequest) (RevisionMeta, error)             // ErrConflict if req.BaseRevision != head
    CreateDraft(ctx context.Context, d Draft) error
    Draft(ctx context.Context, id uuid.UUID) (Draft, error)
    DeleteDraft(ctx context.Context, id uuid.UUID) error
    ExpireDrafts(ctx context.Context, now time.Time) (int, error)
}

type ApplyRequest struct {
    BaseRevision int64
    AppliedBy    string
    Comment      string
    RestoredFrom *int64
    GitCommit    string
    Documents    []ConfigDocument       // changed or new documents: Kind, Name, canonical JSON, sha256
    Manifest     []ManifestItem         // all documents of the new revision: Kind, Name, sha256
    Catalog      []CatalogRow           // resolved live tags of the new revision
    Summary      json.RawMessage
    Audit        AuditBuilder           // builds the config.applied record given the chain head
    Event        events.Event           // skyranger.config.revision_applied.v1
}

type EventStore interface {
    Append(ctx context.Context, evs []events.Event) (inserted int, err error)      // idempotent on (Time, ID)
    Query(ctx context.Context, q EventQuery) (EventPage, error)                    // keyset on (time, id)
}

type AlarmStore interface {
    LoadStates(ctx context.Context) ([]AlarmState, error)
    ApplyTransitions(ctx context.Context, ts []AlarmTransition) error              // state upsert guarded by last_seq + journal insert; idempotent
    Journal(ctx context.Context, q JournalQuery) (JournalPage, error)
}

type StateStore interface {
    LoadDerived(ctx context.Context) ([]DerivedState, error)
    SaveDerived(ctx context.Context, rows []DerivedState) error                    // multi-row upsert, one statement
    LoadMemory(ctx context.Context) ([]MemoryValue, error)
    SaveMemory(ctx context.Context, v MemoryValue) error
    SystemTagID(ctx context.Context, path string) (uuid.UUID, error)              // allocates on first use
    PurgeOrphans(ctx context.Context, olderThan time.Time) (int, error)
}

type AuditBuilder func(prev AuditHead) (AuditRecord, error)

type AuditStore interface {
    // AppendChained takes the audit lock, reads the chain head, calls build, inserts the record. Inside WithTx it
    // participates in the caller's transaction (06, section 4).
    AppendChained(ctx context.Context, build AuditBuilder) (AuditRecord, error)
    Head(ctx context.Context) (AuditHead, error)
    Query(ctx context.Context, q AuditQuery) (AuditPage, error)
    Iterate(ctx context.Context, fromSeq int64, fn func(AuditRecord) error) error  // for verify
    Prune(ctx context.Context, beforeSeq int64) (int, error)                       // pruner role only
}

type Admin interface {
    Migrate(ctx context.Context) error
    Verify(ctx context.Context, level VerifyLevel) (VerifyReport, error)           // amcheck-class checks (08)
    JobStatus(ctx context.Context) ([]JobStatus, error)                            // compression, retention, refresh jobs
    Health(ctx context.Context) (Health, error)
}
```

The shapes are intentionally plain: structs with exported fields, no ORM types, no `pgtype` in signatures. Anything the API returns (a revision, an alarm state, an event) is the contract JSON shape through the `json` tags on the type, so a handler never maps between three representations.

Composition by transaction, example (alarm persister, 06 section 3), showing the style that replaces cross-repository methods:

```go
err := st.WithTx(ctx, func(ctx context.Context, tx storage.Tx) error {
    if err := tx.Alarms().ApplyTransitions(ctx, batch.Transitions); err != nil { return err }
    if _, err := tx.Events().Append(ctx, batch.Events); err != nil { return err }
    return outbox.Insert(ctx, tx, batch.Notifications)   // the alarm area's outbox repository is also Tx-aware
})
```

## 5. Errors

```go
package storage

type Kind uint8

const (
    KindUnavailable Kind = iota + 1 // connection, server starting/stopping, timeouts, too many connections: retry later, spool
    KindConflict                    // stale base revision, unique violation the caller must handle
    KindNotFound
    KindInvalid                     // bad data or request; retrying will not help (poison)
    KindCorrupt                     // XX001/XX002-class: stop and alarm
    KindDiskFull
    KindUnsupported                 // backend lacks a capability (for example AggTimeWeightedAvg)
)

type Error struct {
    Kind Kind
    Op   string // "history.write"
    Err  error  // underlying driver error, never exposed over the API
}

func (e *Error) Error() string { return e.Op + ": " + e.Err.Error() }
func (e *Error) Unwrap() error { return e.Err }
func (e *Error) Is(target error) bool { t, ok := target.(*Error); return ok && t.Kind == e.Kind }

var (
    ErrUnavailable = &Error{Kind: KindUnavailable}
    ErrConflict    = &Error{Kind: KindConflict}
    ErrNotFound    = &Error{Kind: KindNotFound}
    ErrInvalid     = &Error{Kind: KindInvalid}
    ErrCorrupt     = &Error{Kind: KindCorrupt}
    ErrDiskFull    = &Error{Kind: KindDiskFull}
    ErrUnsupported = &Error{Kind: KindUnsupported}
)

func IsRetryable(err error) bool { return errors.Is(err, ErrUnavailable) || errors.Is(err, ErrDiskFull) }
```

Callers branch with `errors.Is(err, storage.ErrUnavailable)`. The PostgreSQL SQLSTATE to `Kind` mapping (03, section 6) lives in one function in `internal/storage/pg` with an exhaustive table test. API mapping to RFC 9457 `code`s: `ErrConflict` to `conflict`, `ErrNotFound` to `not_found`, `ErrUnavailable` to the proposed `unavailable` (503), everything else to `internal` with the underlying error logged, never returned.

## 6. Backend registry

```go
package storage

type Config struct {
    Backend string            // "timescale" (default) | "plainpg" | others
    URL     string            // postgres://... ; secret resolved by the caller from /etc/skyranger/secrets
    Profile string            // meter | small | medium | large (02, section 3)
    Pools   PoolConfig        // writer, state, query, export, admin sizes (01, section 10)
    History *HistoryBackendConfig // optional: a different backend for History only (section 7)
}

type Factory func(ctx context.Context, cfg Config) (Store, error)

func Register(name string, f Factory)         // called from init() of each backend package, linked in by build tags
func Open(ctx context.Context, cfg Config) (Store, error)
```

Backends are **compile-time** Go packages selected by build tag and linked into `skyrangerd`. There is no dynamic plugin loading (Go plugins are fragile, and an out-of-process storage plugin over gRPC, in the style of drivers, is a possible future that the interfaces above do not preclude, but nothing needs it now). The default release binary contains `timescale` and `plainpg`.

Backend choice is a **host setting** (`SKYRANGER_STORAGE_BACKEND`, `site.yaml storage.backend`), not `Gateway` config, because the configuration lives in the database the backend opens (README, decision 3).

## 7. Adding another database

Three levels, from cheap to expensive, so a contributor knows what is being asked:

| Level | What you implement | What you keep |
|---|---|---|
| History only (for example ClickHouse, QuestDB, InfluxDB, a cloud time-series service) | `HistoryStore` + `HistoryAdmin` (about 10 methods, section 3) behind `storage.Config.History` | PostgreSQL (plain, no extension required) for everything else. `Compose(core, history)` returns a `Store` whose `History()` delegates. This works precisely because history is never in a transaction with anything else |
| A different PostgreSQL flavor (plain PostgreSQL, a managed service where extensions cannot be installed) | the `plainpg` migrations and a `HistoryAdmin` that creates partitions and maintains the rollup in Go | the whole `internal/storage/pg` core |
| A different relational database for everything (SQLite, MySQL, SQL Server) | all repositories. Needs `WithTx` with serializable-or-better semantics, row-value comparisons or an equivalent for keyset pagination, `ON CONFLICT DO NOTHING` equivalents, `jsonb`-like storage, and advisory locks or an equivalent serialization primitive for the config and audit locks | nothing |

Requirements any backend must meet, enforced by the conformance suite rather than by prose:
1. **Idempotent writes** keyed as in 01 (history `(tag_id, ts)`, events `(time, id)`, journal `(alarm_id, seq, ts)`). This is what makes the spool correct on any backend.
2. **Exact int64 and the uint64 escape hatch.** No silent float conversion of integers.
3. **Ordered, gapless audit chain** under serialization.
4. **Atomic config apply** with `ErrConflict` on stale `base_revision`.
5. **Prefix queries are byte-wise** (the half-open range, never locale collation).
6. **Error kinds** map as in section 5 so the writer's spool-and-probe logic works unchanged.

The honest limit: a backend that cannot give transactions across config, audit and events cannot back the core. The interface does not hide that; `Capabilities.Transactions` is required true for the core.

## 8. How the historian uses it

```go
// internal/historian/writer.go (sketch)
func (w *Writer) flush(ctx context.Context, batch []history.Sample) {
    switch w.state {
    case stateDirect, stateDraining:
        res, err := w.hist.Write(ctx, batch)       // storage.HistoryStore
        switch {
        case err == nil:
            w.metrics.rows.Add(int64(res.Inserted)); w.metrics.dedup.Add(int64(res.Submitted - res.Inserted))
        case errors.Is(err, storage.ErrInvalid):
            w.bisectAndDeadLetter(ctx, batch)
        case errors.Is(err, storage.ErrCorrupt):
            w.enterCorrupt(err); w.spool(batch)
        case storage.IsRetryable(err):
            w.enterSpooling(err); w.spool(batch)   // unknown outcome is safe: idempotent replay
        default:
            w.spool(batch)
        }
    case stateSpooling, stateShedding: ...
    }
}
```

Nothing in the writer knows PostgreSQL exists.

## 9. Conformance suite and tests

`storagetest.Run(t, factory)` takes a function returning a fresh `Store` and runs the same tests against every backend. Groups:

| Group | Asserts |
|---|---|
| History idempotency | double `Write` of the same batch inserts once; conflicting values for one key keep the first; any batch order |
| Value codec | all six datatypes round-trip exactly, including `uint64` boundaries, null values, `bool` |
| Read paths | raw window, per-series keyset pagination (no duplicates, no gaps across pages), boundary lookback, half-open ranges |
| Aggregates | every `AggFunc` against a Go reference implementation on random data; **rollup path equals raw path**; quality semantics (a bucket of [bad, good, uncertain] has `worst = uncertain`, usable count 2; all-bad bucket is null/bad) |
| Delta | carry-forward deltas sum to `end - start` exactly; literal delta differs by the documented amount |
| Time zones | local-day consumption across a DST change (23 and 25 hour days) for `Europe/London`, a half-hour zone and a quarter-hour zone |
| Prefix queries | `a/b` matches `a/b` and `a/b/c`, never `a/bc` or `a/b-x`, with `_` in names |
| Config | atomic apply, stale base conflict, lossless round trip with manifest hash equality, restore creates new revision, rename swap does not violate path uniqueness |
| Audit | gapless seq under concurrent appends, chain verifies, `sr_app` cannot update or delete |
| Alarms | replay of transitions is idempotent; stale `last_seq` does not move state backwards |
| Events | idempotent append, keyset paging, type and subject prefix |
| Errors | induced faults (connection killed mid-batch, statement timeout, disk-full simulation, constraint violation) produce the right `Kind` |
| Maintenance | `EnsurePolicies` idempotent; retention drops whole ranges; `PruneOldest` reports exactly what it dropped; `RefreshAggregates` after back-dated insert |

Timescale-specific behavior tests (every UNVERIFIED item in 02) live in `internal/storage/timescale/behavior_test.go` and run against the **pinned image digest**. A TimescaleDB upgrade is only merged if they pass.

Test infrastructure: containers started from the pinned digest by the test harness ([09-testing](../09-testing/README.md) owns the harness); one database per test package, schema reset per test via `TRUNCATE`; fault injection with a TCP proxy in front of PostgreSQL that can sever or stall connections. Backends other than PostgreSQL bring their own container.

## Decisions made
1. Persistence is behind `pkg/storage`; only `internal/storage/*` imports a database driver (CI import-lint).
2. The `Store` is a set of small repositories plus `WithTx` for composition. History is excluded from transactions on purpose, which also makes history-only backends possible.
3. Plain data structs with `json` tags matching the contracts; no ORM, no driver types in signatures.
4. Errors are a small `Kind` taxonomy with `errors.Is` helpers; retryability is derived, never string-matched.
5. Backends are compile-time packages registered by name; selection is a host setting.
6. A shared conformance suite is the definition of "a supported backend".
7. `pgx` v5 with hand-written SQL, no query generator and no ORM.

## Alternatives rejected and why
- **`database/sql` plus a portable SQL dialect everywhere**: lowest common denominator would give up `unnest` batch inserts, `ON CONFLICT`, `jsonb`, hypertables, and advisory locks, the things this design relies on. The abstraction is at the repository level, not the SQL level.
- **An ORM (GORM, ent)**: hides the batch insert, locking and keyset patterns this design depends on, adds reflection on a hot path, and makes the SQL harder to review. Hand-written SQL in a few files per repository is reviewable.
- **`sqlc`-generated code**: attractive for type safety and plausible later; the `unnest` batch shapes and dynamic keyset queries are awkward for it, and it adds a build step. Revisit if the query count grows.
- **One giant `Storage` interface**: untestable and forces a history-only backend to stub dozens of methods.
- **Cross-repository convenience methods** (`CommitAlarmTransition(state, journal, event, outbox)`): bakes today's composition into the interface; `WithTx` composes any combination.
- **Dynamic plugins** (Go `plugin`, or out-of-process gRPC like drivers) for storage now: no consumer exists; compile-time registration is simpler and keeps the single static binary.
- **Treating history as part of `WithTx`**: couples the hottest write path to the slowest transactions.

## Open questions
- Go module path (`skyranger/...` placeholder) and where `pkg/history` and `pkg/events` types live relative to the tag engine's `pkg/tag` (the tag engine Go design is not written yet).
- Should `Interval`, `Hold` and `TZ` be resolved by the API layer or by the planner inside `HistoryStore`? This sketch puts the planner (raw vs rollup, tail union) inside the Timescale backend, so another backend can do it differently; if every backend needs the same planner it should move up into a shared layer over a smaller `HistoryStore`.
- Whether a gRPC out-of-process storage plugin (reusing the driver host pattern) is worth designing now for customer-chosen history databases.
- `Capabilities.TimezoneBuckets`: is a Go fallback for local-day buckets needed for any realistic backend, or can it be dropped?
- Uniform handling of the alarm area's `notification_outbox` in `Tx` (it is outside this interface list because its DDL belongs to the alarm area).
- `google/uuid` UUIDv7 generation availability in the pinned library version (UNVERIFIED).
