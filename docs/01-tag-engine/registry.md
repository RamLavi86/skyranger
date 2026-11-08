# Tag Registry, Path Resolution and Templates

Status: design draft v0.1. Binding inputs: [tag-model.md](../00-contracts/tag-model.md) sections 1, 2, 9, 10; [README.md](../00-contracts/README.md) naming rules.

The **registry** is the engine's compiled, immutable view of one applied config revision: every tag, fully resolved (templates expanded, references turned into `tag_id`s, handles allocated, dependency ranks computed). It is built outside the engine loop by the **compiler** and swapped in atomically. The same compiler validates config drafts for the config service, so "the draft validated" and "the engine accepted it" can never disagree.

## 1. Compile pipeline

```
YAML/JSON documents of revision N+1 (Gateway, Connection[], Template[], TagGroup[])
  │ 1 schema validation (JSON Schema per kind, owned by config service)
  │ 2 template expansion  (section 4)            -> []ExpandedTag  (still has references)
  │ 3 identity            (section 5)            -> tag_id per tag; new ids minted
  │ 4 path checks         (grammar, collisions, _system reserved)
  │ 5 field validation    (section 6)
  │ 6 reference resolution: derived.inputs -> tag_id; dangling = error
  │ 7 dependency graph    (derived.md section 3): cycles = error; ranks
  │ 8 driver checks       ValidateAddress per source tag via driver host (no I/O)
  │ 9 handle allocation   (section 7), carrying over handles from revision N
  ▼
*Registry (immutable)  +  Diff against revision N  +  []ValidationError (JSON Pointer)
```

All errors are collected (not fail-fast) so an engineer sees every problem in one draft response, each as `{document, pointer, code, message}` (api.md section 3). The compiler is a pure function of `(documents, previous registry, driver address validators, id minting function)`, which makes it table-testable with golden files.

If step 8 cannot run because a driver process is down, the address is checked only syntactically by a copy of the driver's validator if it is built in (all v1 drivers are), else the draft is reported `valid: false` with code `driver_unavailable`. Apply is never allowed on an unvalidated address.

## 2. Key types

```go
package registry // skyranger/internal/tagengine/registry

type TagIndex uint32 // dense index into Registry.Tags and the value store; per revision

type Registry struct {
    Revision   int64
    Tags       []*TagDef                  // dense, index = TagIndex
    ByID       map[uuid.UUID]TagIndex
    ByPath     map[string]TagIndex
    sorted     []TagIndex                 // sorted by Path bytes, for prefix scans
    ByHandle   map[string][]TagIndex      // connection name -> slice indexed by handle (0 = unused)
    Dependents [][]TagIndex               // input -> derived tags that read it
    MaxRank    int
    Connections map[string]*ConnDef       // timestamp_source, enabled, driver
    Gateway    GatewayDef                 // max_clock_skew, timezone
}

type Kind uint8
const ( KindSource Kind = iota + 1; KindDerived; KindMemory; KindSystem ) // KindSystem: see section 8

type TagDef struct {
    Index       TagIndex
    ID          uuid.UUID
    Path        string
    DisplayName string
    Description string
    Kind        Kind
    DataType    tag.DataType
    Unit        string               // UCUM code, "" if none
    EnumLabels  map[int64]string
    Enabled     bool
    Writable    bool
    WriteLimits *Range

    // source only
    Conn        string
    Address     string
    Handle      uint32
    RawType     tag.DataType          // AddressInfo.DataType from ValidateAddress
    Interval    time.Duration
    Scale       *Scale                // nil = identity

    EngRange    *Range
    Deadband    *Deadband
    StaleAfter  time.Duration         // 0 = never
    History     HistoryCfg

    // derived only
    Derived     *DerivedDef           // fn, ordered inputs (name, TagIndex), params, eval_interval
    Rank        int                   // 0 for source/memory/system, 1+ for derived

    // memory only
    InitialValue *tag.Variant

    Alarms      []AlarmDef            // passed through to the alarm engine untouched
    Origin      Origin                // which document / template / instance produced it
    Hash        uint64                // hash of all behavior-affecting fields, for diffing
}

type Origin struct {
    Document string                   // "TagGroup/pumphouse"
    Pointer  string                   // JSON Pointer of the instance or tag entry
    Template string                   // "" for plain tags
    Instance string                   // instance root path, "" for plain tags
    TemplateTag string                // tag name inside the template
}
```

`TagDef` is never mutated after compile. The engine holds its mutable per-tag state separately (pipeline.md section 2), indexed by `TagIndex`.

## 3. Path resolution

- **Exact path**: `ByPath` map, O(1).
- **By id**: `ByID` map, O(1). The API addresses tags by `tag_id` in URLs (api.md), so this is the hot one.
- **Prefix** (segment-wise, README.md rule): binary search `sorted` for the first path `>= prefix`, then iterate while `strings.HasPrefix(path, prefix)` (byte-prefix matches are always a contiguous range in a sorted list), keeping only candidates where `path == prefix` or `path[len(prefix)] == '/'`. The filter is needed because the matching paths are interleaved: `-` (0x2D) sorts below `/` (0x2F), so `a/b-x` sits between `a/b` and `a/b/c` and must be skipped. Cost is O(log n + byte-prefix candidates), fine for 50k tags.
- **References in config** (`raw/pulse_count`, `/site1/other/thing`): relative to the instance root or `base_path`; leading `/` is absolute. Resolved once at compile; the runtime never sees references.
- Path lookups from the API go through the registry snapshot current at request time. After a rename, the old path returns `404`; there are no aliases (contract).

## 4. Template (UDT) instantiation

Expansion runs on the YAML node tree (`gopkg.in/yaml.v3` `yaml.Node`), before decoding into Go structs, exactly as tag-model.md section 10 requires.

Per instance:

1. **Bind params.** For each template param: take the instance value, else `default`, else error `required`. Unknown instance params are an error (typos must not pass silently). Type-check: `duration` must parse with `time.ParseDuration`; `uint64` must be a non-negative integer; `float64` accepts integers.
2. **Deep-copy** the template `spec.tags` node tree.
3. **Substitute** by walking every scalar node:
   - Scalar text exactly `${name}`: replace the node with the typed param value, re-tagged (`!!int`, `!!float`, `!!bool`, `!!str`; a `duration` becomes its canonical string, e.g. `150s` becomes `2m30s`).
   - `${name}` inside longer text: textual interpolation, result is a string.
   - `$${` becomes literal `${`. Any remaining `${...}` with an unknown name is an error with the JSON Pointer of the node.
   - Map keys are not substituted (tag names in templates are fixed, which keeps `tag_ids` maps stable).
4. **Apply overrides.** `overrides.{template tag name}` is a map of tag fields. Merge rule: **top-level field replace** (an override of `deadband` replaces the whole `deadband` object; an override of `derived` replaces the whole `derived` block; `alarms` replaces the whole list). An override naming a tag the template does not have is an error. Overrides are not substituted (they are written per instance, so params are pointless there).
5. **Prefix paths**: each tag's full path is `{base_path}/{instance name}/{tag name}`.

Example: instance `meter01` of `pulse_water_meter` with `edge_node_id: esp32-meter01` turns

```yaml
source: { connection: ${connection}, address: "spb://${group_id}/${edge_node_id}#pulse_count" }
stale_after: ${heartbeat_stale}
```

into

```yaml
source: { connection: local-sparkplug, address: "spb://skyranger/esp32-meter01#pulse_count" }
stale_after: 2m30s
```

Live linking: because instances are re-expanded from the current template on every compile, editing a template changes every instance in the next revision, and the diff (section 9) lists every affected resolved tag.

## 5. Tag identity

- Plain tags: `id` field if present. Absent: match by full path against revision N; if no match, mint a new UUIDv7.
- Template instance tags: `instances[].tag_ids[{template tag name}]` if present. Absent: match by `(instance root, template tag name)` against revision N, then by full path; else mint.
- **Minted ids are written back** into the stored revision documents (`id` on plain tags, `tag_ids` on instances) by the config service at apply time. That makes export carry every id, so export then import is an empty diff (contract requirement), and a template rename is visible as delete plus create.
- Duplicate `tag_id` in one revision is an error.
- An id that existed in an older revision but not in N (e.g. after `restore`) is accepted and keeps its history and its persisted derived state (derived.md section 7).

## 6. Field validation (beyond JSON Schema)

| Check | Error code |
|---|---|
| Path grammar, 16 segments, 255 bytes, `_system` reserved | `invalid_path` |
| Two tags resolve to the same path | `path_collision` |
| A path is both a tag and a prefix of another tag (`a/b` and `a/b/c`) | allowed (MQTT topics allow it; the HMI tree shows a node that is also a folder) |
| `unit` is not a recognized UCUM code | `invalid_unit` (see note) |
| `scale` on a tag whose `datatype` is not `float64` | `scale_requires_float64` |
| `scale.type: range` with `raw_low == raw_high` | `invalid_scale` |
| `deadband.type: percent` without `eng_range` | `deadband_requires_eng_range` |
| `eng_range.low >= high` | `invalid_range` |
| `enum_labels` on a non-`int64` tag | `invalid_field` |
| `writable: true` on a derived tag whose function has no write semantics (only `counter_total` has them) | `not_writable` |
| `writable: true` on a source tag whose address `ValidateAddress` says is not writable, or whose driver lacks `Capabilities.Write` | `not_writable` |
| `write_limits` on a non-numeric or non-writable tag | `invalid_field` |
| Source `connection` does not exist | `not_found` |
| Derived `fn` unknown, input names wrong, params wrong type, declared `datatype` differs from function output type | `invalid_derived` |
| `eval_interval` missing for `rate`, `liveness`, `continuous_activity` | `eval_interval_required` (derived.md section 5) |
| Dangling reference | `not_found` |
| Cycle | `cycle` (message lists the cycle path) |
| Dependency depth > 32 | `too_deep` |

UCUM note: the full UCUM grammar is large. v1 validates against a checked-in table of accepted codes (`L`, `L/min`, `L/s`, `m3`, `m3/h`, `Cel`, `K`, `Pa`, `kPa`, `bar`, `%`, `1`, `s`, `min`, `h`, `W`, `kW`, `kW.h`, `V`, `A`, `Hz`, `m`, `mm`, `kg`, ...) built from the UCUM tables. Adding a code is a one-line change. We are not aware of a maintained Go UCUM parser (UNVERIFIED); a full parser can replace the table later without changing configs.

## 7. Handles

- One handle per source tag, `uint32`, allocated per connection, starting at 1. Two tags may use the same address (e.g. the same register scaled two ways); each gets its own handle and the driver may share the underlying read.
- **Handles are carried over** for tags whose `(connection, address, RawType)` is unchanged, so a revision that does not touch a tag causes no `Unsubscribe`/`Subscribe` for it (contract: "unchanged tags keep their state and subscriptions").
- Freed handles are not reused for the life of the driver process, so a late sample for a removed tag can never be attributed to a new tag. `ByHandle` slices are rebuilt per revision; a sample with an unknown handle is dropped and counted (`_system/engine/unknown_handle_samples`).
- After a driver restart the host re-subscribes the same handles (driver-interface.md section 3). After a `skyrangerd` restart, handles are assigned fresh; that is safe because the driver processes are children and restart too.

## 8. `_system` tags

- Created by the engine and driver host, never by config. Internally `KindSystem`; in the API they report `kind: source` with no `source` block (see Open questions).
- Their `tag_id`s must be stable across restarts so their history continues. They are stored in a small table `system_tags(path text primary key, tag_id uuid)`; created on first use.
- They are compiled into every registry by the compiler from the list of connections (`_system/connections/{name}/...`) and the fixed engine list (`_system/engine/...`, performance-and-failures.md section 4).

## 9. Hot reload: diff and state carry-over

The compiler diffs registry N+1 against N by `tag_id` and classifies each tag. The engine applies the classification inside its loop (go-design.md section 4).

| Change | Examples | Effect on runtime state |
|---|---|---|
| none | | keep everything, same handle, no events |
| metadata | `display_name`, `description`, `unit`, `enum_labels`, `alarms`, `history`, `write_limits` | keep value and state; alarm engine gets the new definitions |
| rename | path changed | keep value and state; old retained bus topic cleared; WS `config_changed` and prefix `resync` |
| processing | `scale`, `eng_range`, `deadband`, `stale_after` | keep state; current value **re-derived from the retained last raw sample** with the new scale, so the HMI reflects the change immediately |
| source rebind | `connection`, `address`, raw type | unsubscribe old handle, subscribe new; value resets to `bad_waiting_for_initial_data` (a value from a different point is not this tag's last value) |
| datatype | `datatype` changed | reset to `bad_waiting_for_initial_data`; derived state discarded |
| derived params | same `fn`, same input `tag_id`s, different `params` | keep function state (a `counter_total` total survives a `multiplier` correction; the new multiplier applies to future deltas only; `initial_total` is ignored once state exists) |
| derived inputs | same `fn`, an input now points to a different `tag_id` | function decides: `counter_total` keeps `total` and **rebaselines** (forgets `last_raw`/`last_epoch`; the next sample sets the baseline with zero delta); others reset |
| derived fn | `fn` changed | state discarded, warm-up starts again |
| enabled -> disabled | | unsubscribe; quality `bad_out_of_service`, last value held; derived state kept |
| removed | | unsubscribe; retained topic cleared; in-memory state dropped. Persisted derived state rows are **kept** (marked orphaned), so restoring a revision that brings the same `tag_id` back resumes the total instead of restarting it |
| added | | `bad_waiting_for_initial_data` (or restored memory value / derived state if the id was seen before) |

The diff response from the draft endpoint shows these classes, so an engineer sees "total_volume: params changed, state kept" before applying.

## Decisions made
1. One compiler, used for both draft validation and engine load; pure function, golden-file tested.
2. Registry is immutable per revision, dense-indexed, swapped atomically.
3. Template expansion on `yaml.Node` trees; typed whole-scalar substitution; overrides are top-level field replacement.
4. Minted ids are written back into stored revision documents so exports always carry ids.
5. One handle per source tag, carried over when unchanged, never reused within a driver process lifetime.
6. Change classification table drives state carry-over; `counter_total` state survives everything except a function change, datatype change, or removal, and even removal keeps the persisted row.
7. UCUM validation via a checked-in code table in v1.

## Alternatives rejected and why
- **Mutable registry with per-tag locks**: concurrent edits during evaluation make derived ordering and diffs hard to reason about.
- **Deep merge for overrides**: surprising with lists (alarms) and hard to show in a diff; top-level replace is predictable.
- **Expanding templates after decoding into Go structs**: the contract requires node-tree substitution, and typed replacement is cleaner on nodes.
- **Path-keyed state**: renames would lose `counter_total` state. Everything is keyed by `tag_id`.
- **Radix tree for path index**: unnecessary at 50k tags; sorted slice plus map is simpler and fast enough.
- **Deleting derived state on tag removal**: an accidental removal then restore would reset a utility meter total to `initial_total`; keeping orphaned rows costs a few bytes.

## Open questions
- CONTRACT: `_system` tags need a `kind` in the API. Options: report `source`, or add `kind: system`. Adding a value is additive under api.md versioning rules; recommend `system`.
- CONTRACT: the override merge rule (top-level replace) is not stated in tag-model.md section 10. Should be added there.
- CONTRACT: rebaseline after a `counter_total` input change emits no `counter_discontinuity` event because none of the four `kind`s fits. Propose adding `kind: rebaseline` (additive).
- Retention of orphaned derived state rows: keep forever, or purge after the tag's history retention? Proposal: purge together with history retention.
- Should the compiler warn (not error) when a template change affects more than N instances, as a guard against accidental mass edits?
- Driver address validation when the driver binary is third-party and its process is down: currently blocks apply. Acceptable?
