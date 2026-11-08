# 02 Script API (Python)

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Depends on P1, P3, P4, P7 (see [README.md](README.md)). Transport is described in [01-runtime.md](01-runtime.md); permissions in [04-sandbox-and-permissions.md](04-sandbox-and-permissions.md).

## 1. Principles

- **One shape for values.** A tag value in Python is the canonical value object from the contracts README, as a frozen dataclass. No variants.
- **Synchronous and blocking.** Scripts are run-to-completion handlers. Calls block until done. No asyncio, no threads.
- **Same semantics as REST.** Every call is served by the same Go service function as the REST endpoint, so validation (`writable`, `write_limits`, type checks), error `code`s and audit are identical. A script can do nothing an `engineer`-scoped REST client could not, and usually much less (permissions, 04).
- **Python 3, typed, boring.** The `skyranger` package ships `py.typed` and is checked with `mypy --strict` in CI. Paths are plain `str` using the contract grammar. Durations are `datetime.timedelta` in Python and Go duration strings on the wire.
- **Names follow the contracts.** `path`, `tag_id`, `value`, `quality`, `ts`, `ts_source`, `ts_gateway`; error codes from api.md section 7; quality names from tag-model.md section 4.

## 2. Types

```python
from skyranger import TagValue, Quality
```

```python
@dataclass(frozen=True, slots=True)
class TagValue:
    tag_id: str                      # UUIDv7, lowercase hyphenated
    path: str
    value: bool | int | float | str | datetime | None
    quality: Quality
    ts: datetime                     # effective timestamp, tz-aware UTC
    ts_gateway: datetime
    ts_source: datetime | None = None

    @property
    def good(self) -> bool: ...      # quality.cls == 0

class Quality(str):                  # the contract name, e.g. "good", "uncertain_last_known"
    code: int                        # 0, 1, 64, ... (tag-model.md section 4)
    cls: int                         # code >> 6: 0 good, 1 uncertain, 2 bad
    is_good: bool
    is_uncertain: bool
    is_bad: bool
```

Type mapping (the engine's six datatypes, tag-model.md section 3):

| `datatype` | Python | Notes |
|---|---|---|
| `bool` | `bool` | |
| `int64`, `uint64` | `int` | JSON strings on the wire are converted both ways. Python ints beyond the 64-bit range raise `TypeMismatch` on write |
| `float64` | `float` | `NaN` and `Inf` never arrive (the engine turns them into `None` plus `bad_type_mismatch`). Writing them raises `ValueError` locally |
| `string` | `str` | max 4096 bytes UTF-8 (engine rule) |
| `datetime` | `datetime` | always tz-aware UTC. Naive datetimes are rejected on write (`ValueError`) |
| null | `None` | only with a `bad` quality and no held value |

A write value is checked against the tag's `datatype` by the host: an `int` to a `float64` tag is widened; a non-integral `float` to an `int64` tag raises `TypeMismatch` (`type_mismatch`, 422).

**Always check quality.** A value with `quality="uncertain_last_known"` is the last value from a device that went quiet. Examples below show the habit.

## 3. Modules

```python
from skyranger import tags, history, events, alarms, http, state, log, now
```

### 3.1 `tags`

```python
tags.read(path: str) -> TagValue
tags.read_many(paths: Sequence[str]) -> dict[str, TagValue]    # one call, max 1000 (api.md /tags/values)
tags.read_prefix(prefix: str, *, kind: str | None = None) -> dict[str, TagValue]  # segment-wise prefix
tags.browse(prefix: str = "", *, limit: int = 1000) -> list[TagInfo]   # definitions: path, tag_id, datatype, unit, writable, kind
tags.write(path: str, value, *, comment: str | None = None, wait: timedelta = timedelta(seconds=10)) -> WriteResult
tags.write_many(items: Mapping[str, Any], *, comment=None, wait=...) -> dict[str, WriteResult]  # independent writes, per-item result
```

- `tags.read` of a path the script may not read raises `Forbidden`; of a path that does not exist raises `NotFound`. A read never returns "bad because forbidden".
- `tags.write` is equivalent to `POST /tags/{tag_id}/write?wait=...`. It returns when the outcome is known. `WriteResult` has `write_id`, `status` (`"ok"`), `value`. On failure it raises the mapped exception (`NotWritable`, `OutOfRange`, `TypeMismatch`, `NotConnected`, `DeviceError`, `Timeout`). `Timeout` means **outcome unknown** (driver-interface.md section 4: writes are never retried; the script decides). Scripts must not blindly retry writes to devices.
- `comment` is stored in the audit record and in `skyranger.tag.write_completed.v1`. `requested_by` is `script:{pool}/{script}` (P4).
- Writing a `memory` tag stores quality `good` (P7), not `good_manual`.
- Writing a `source` tag requires `writable: true` on the tag **and** `tags.write` permission covering it. Writing a `derived` tag works only where the built-in defines write semantics (`counter_total`).
- No script API to write quality or timestamps in v1. Computation belongs in derived tags or (later) `kind: script` tags.

### 3.2 `history`

```python
history.read(paths: str | Sequence[str], start: datetime, end: datetime | None = None, *,
             agg: Literal["raw","avg","min","max","first","last","count","delta"] = "raw",
             interval: timedelta | None = None, limit: int = 10_000) -> dict[str, Series]

@dataclass(frozen=True)
class Series:  path: str; unit: str | None; agg: str; interval: timedelta | None; points: list[Point]
@dataclass(frozen=True)
class Point:   ts: datetime; value: ...; quality: Quality
```

Same parameters, defaults, limits (raw max 100000) and aggregate rules as `GET /history` (api.md): aggregates use only good and uncertain samples; a bucket with none has `value=None`, `quality="bad"`. `interval` is required when `agg != "raw"`. Daily buckets align to `Gateway.spec.timezone` unless `tz=` is passed (IANA string). The call pages internally up to `limit`; exceeding raises `BudgetExceeded`.

### 3.3 `events`

```python
events.emit(name: str, data: Mapping[str, Any] | None = None, *, subject: str | None = None) -> str   # returns event id
events.query(types: Sequence[str] = (), *, subject_prefix: str | None = None, start: datetime | None = None, end: datetime | None = None, limit: int = 100) -> list[Event]
```

- `emit("daily_report_done", {...})` publishes `skyranger.user.daily_report_done.v1` (P3). `name` must satisfy the segment grammar. `source` is `/gateways/{gateway_id}/script-host`. `data` is a JSON object, at most 16 KiB. The event is written to the `events` table and the internal bus like any other.
- Scripts cannot emit event types outside `skyranger.user.*` (no forging `skyranger.alarm.*`). `permissions.events.emit` lists the allowed names.
- `Event` mirrors the CloudEvents envelope fields (`id`, `source`, `type`, `time`, `subject`, `data`).

### 3.4 `alarms`

```python
alarms.active(prefix: str = "", *, state: Sequence[str] = ()) -> list[AlarmState]   # messages.md section 5 shape
alarms.ack(alarm_id: str, comment: str) -> None            # permission alarms.ack; audit as the script principal
```

**Scripts do not create alarms.** Alarms are tag configuration (tag-model.md section 8). The idiom for a script-detected condition is: define a `memory` bool tag with an alarm, and have the script write it. That keeps one alarm engine, one state machine, one HMI path. Shelving is not available to scripts in v1.

### 3.5 `http`

```python
http.request(method: str, url: str, *, params=None, json=None, data: bytes | str | None = None,
             headers: Mapping[str, str] | None = None, timeout: timedelta = timedelta(seconds=10)) -> HttpResponse
http.get(url, **kw); http.post(url, **kw); http.put(url, **kw); http.delete(url, **kw)

@dataclass(frozen=True)
class HttpResponse:  status: int; headers: dict[str, str]; body: bytes
                     def text(self) -> str; def json(self) -> Any; def raise_for_status(self) -> None
```

Executed by the **host** in Go (workers have no network), against the script's allowlist. Secrets are injected host-side from the allowlist entry; the script never sees them. Details, SSRF rules, size caps in 04 section 5. `HttpDenied` (code `forbidden`) when the URL is not allowlisted.

### 3.6 `state`

```python
state.get(key: str, default=None) -> Any          # JSON-serializable values
state.set(key: str, value: Any) -> None
state.delete(key: str) -> None
```

A small per-script key-value store persisted by the host in PostgreSQL (limits: 64 KiB total per script, 64 keys), cleared when the script is deleted. Meant for "last report sent" style bookkeeping, not data. Writes are atomic per call, not transactional across calls. Included in backups, **not** in config revisions or export (it is runtime state, not config).

### 3.7 `log`, `now`

```python
log.debug|info|warning|error(msg: str, **fields)   # structured; fields must be JSON-serializable
now() -> datetime                                  # tz-aware UTC; the testing harness can replace the clock
```

`print()` is captured as `info` with `stream="stdout"`. Use `log`. Details in [05-errors-and-logging.md](05-errors-and-logging.md). Do not call `datetime.now()` in code you want to unit-test; use `skyranger.now()`.

### 3.8 `ctx` (handler argument)

```python
def handler(ctx):
    ctx.run_id: str
    ctx.script: str                    # name
    ctx.trigger: Trigger               # kind + details, see below
    ctx.params: dict                   # manual trigger only, validated against params_schema
    ctx.deadline: datetime             # soft timeout instant
    ctx.dry_run: bool
    ctx.remaining() -> timedelta
```

`ctx.trigger` shapes (03 defines when they fire):

| `ctx.trigger.kind` | Extra attributes |
|---|---|
| `tag_change` | `path`, `new: TagValue`, `old: TagValue | None` |
| `timer` | `scheduled_for: datetime`, `late: timedelta` |
| `event` | `event: Event` |
| `startup` | `reason: "gateway_start" | "config_applied"` |
| `manual` | `requested_by: str` |

### 3.9 Libraries

`ScriptLibrary` documents (03) are importable as `libs.<name>`: `from libs import units`. Library code runs with the **caller's** permissions and budgets, in the caller's run. Libraries are pure Python from the allowed standard library; they cannot declare permissions of their own in v1.

## 4. Exceptions

All derive from `skyranger.SkyrangerError`, which carries `code` (api.md section 7 string), `message`, `details`.

| Exception | `code` | Raised when |
|---|---|---|
| `BadRequest` | `bad_request` | bad argument (also plain `ValueError` for local checks) |
| `Forbidden` | `forbidden` | not permitted by the script's permissions or pool ceiling |
| `NotFound` | `not_found` | tag, alarm or key missing |
| `NotWritable` | `not_writable` | tag not writable |
| `OutOfRange` | `out_of_range` | outside `write_limits` |
| `TypeMismatch` | `type_mismatch` | value does not fit datatype |
| `NotConnected` | `not_connected` | write with the connection down |
| `DeviceError` | `device_error` | driver reported a device exception |
| `Timeout` | `timeout` | write outcome unknown, or a host call exceeded its deadline |
| `BudgetExceeded` | `rate_limited` | per-run or per-second budget exceeded (03) |
| `HttpDenied` | `forbidden` | URL not allowlisted |
| `ScriptTimeout` | (none) | run wall-clock exceeded; raised inside the worker at the soft deadline. Deliberately derives from `BaseException`, not `Exception`, so a broad `except Exception` does not swallow it; a bare `except:` still can, hence the hard kill |

Unknown codes from a newer host map to `SkyrangerError` itself (forward compatible: "tolerate unknown enum values", api.md section 8).

## 5. Worked examples (water meter)

These use the tags from tag-model.md section 11. They assume a `memory` tag `site1/pumphouse/meter01/yesterday_l` (float64, `L`) has been added to the TagGroup.

### 5.1 Daily consumption report (timer trigger)

```python
# script: daily_report   trigger: timer cron "5 0 * * *" (00:05, Gateway timezone)
from datetime import timedelta
from skyranger import tags, history, events, log, now, state

METER = "site1/pumphouse/meter01"

def handler(ctx):
    end = ctx.trigger.scheduled_for                 # deterministic: not now()
    start = end - timedelta(days=1)
    s = history.read(f"{METER}/total_volume", start, end, agg="delta", interval=timedelta(days=1))
    point = s[f"{METER}/total_volume"].points[0]

    if point.value is None or not point.quality.is_good:
        # no usable data, or a bucket that includes uncertain samples: flag it, do not write a wrong number
        log.warning("daily total unreliable", quality=str(point.quality))
        events.emit("daily_report_incomplete", {"path": METER, "day": start.date().isoformat()})
        return

    tags.write(f"{METER}/yesterday_l", point.value, comment=f"daily_report {start.date()}")
    events.emit("daily_report_done", {"path": METER, "day": start.date().isoformat(), "liters": point.value})
    state.set("last_report_day", start.date().isoformat())
```

Note this uses `delta` on `total_volume`, so reboots and rollovers are already handled by `counter_total` (api.md history rules). A `counter_discontinuity` of kind `reset` in the period is reported by the next example.

### 5.2 Notify on leak alarm (event trigger plus outbound call)

```python
# script: leak_webhook   trigger: event type "skyranger.alarm.state_changed.v1", subject_prefix "site1/pumphouse"
from skyranger import http, log

def handler(ctx):
    a = ctx.trigger.event.data                      # alarm state object plus previous_state (messages.md)
    if a["name"] != "leak" or a["state"] != "active_unacked":
        return
    r = http.post("https://hooks.example.com/sky-ranger",     # must be in permissions.http
                  json={"text": f'{a["path"]}: {a["message"]}', "severity": a["severity"]})
    r.raise_for_status()                            # status >= 400 raises; the run fails and is recorded
    log.info("leak webhook sent", path=a["path"], status=r.status)
```

The webhook token is added by the host from the allowlist entry (`secret://webhooks/leak`), so it is absent from the source and from exports.

### 5.3 React to a counter reset (event trigger)

```python
# script: reset_audit   trigger: event "skyranger.tag.counter_discontinuity.v1"
from skyranger import events

def handler(ctx):
    d = ctx.trigger.event.data
    if d["kind"] == "reset":
        # prev_raw / new_raw are strings on the wire (uint64); convert deliberately
        # (counts between the last sample and the reboot are unrecoverable, tag-model.md section 7)
        new_raw = int(d["new_raw"])
        events.emit("meter_reboot_noted", {"path": d["path"], "new_epoch": d["new_epoch"], "new_raw": new_raw},
                    subject=d["path"])
```

### 5.4 Tag-change interlock-style logic (tag change trigger)

```python
# script: low_flow_flag   trigger: tag_change on meter01/flow_rate, debounce 30s
from skyranger import tags

SRC = "site1/pumphouse/meter01"

def handler(ctx):
    v = ctx.trigger.new
    if not v.good:                       # dead device is not "low flow"
        return
    tags.write(f"{SRC}/low_flow", v.value < 1.0, comment="low_flow_flag")
```

Scripts are **not** safety interlocks. Control logic with safety consequences belongs in the PLC, not in a gateway script (scripts can be delayed, queued, dropped or quarantined by design, see 03).

## Decisions made
1. Values, quality, timestamps, error codes and event envelopes in Python are exactly the contract shapes; `int64`/`uint64` become Python `int`, `datetime` is always tz-aware UTC.
2. The API is synchronous and blocking; `tags.write` waits for the outcome by default (`?wait` semantics) and `Timeout` means unknown outcome, never silently retried.
3. Scripts raise alarms by writing memory tags that carry alarm config; there is no `alarms.raise`. One alarm engine, one state machine.
4. User events live under `skyranger.user.{name}.v1` (P3); scripts cannot forge system event types.
5. Outbound HTTP is a host-executed call with allowlist and host-side secret injection; there is no `requests` or `socket` in the worker.
6. `state` is a small, per-script, host-persisted KV; module globals are explicitly not persistent.
7. Libraries (`libs.*`) execute with the caller's permissions.
8. `ScriptTimeout` derives from `BaseException` so ordinary `except Exception` cannot absorb a timeout.
9. `skyranger.now()` is the only clock scripts should use if they want tests to control time.

## Alternatives rejected and why
- **Mirroring Ignition's `system.*` namespace** (`system.tag.readBlocking`): familiar to Ignition engineers but names a Java/Jython heritage (blocking vs async variants) we do not have. Module-per-concern (`tags`, `history`) is plainer Python. A migration cheat sheet is in [08-ignition-comparison.md](08-ignition-comparison.md).
- **Async API**: handlers are short, run-to-completion; async adds a second execution model and complicates the pipe protocol and timeouts.
- **Returning quality-aware wrappers that behave as plain numbers** (operator overloading): hides bad quality in arithmetic. Explicit `.value` and `.good` is deliberately a little verbose.
- **A script-created alarm API**: duplicates alarm state, shelving, ack, audit and HMI wiring.
- **A SQL API to the historian**: bypasses history rules (quality, aggregation, tz alignment) and widens the attack surface; `history.read` is enough and stable across storage changes.
- **Direct tag write of quality/timestamp**: lets scripts forge provenance; reports could not trust history.
- **Passing secrets into scripts**: any secret a script can read can end up in logs, events or tags.

## Open questions
1. Is `http` enough for notification in v1, or does v1 need `notify.send(channel, ...)` (a plugin extension point, see [09-plugins.md](09-plugins.md))? Current answer: webhook only until v2.
2. `history.read` of `derived` tags that were recomputed after a config change: nothing special in v1, but the historian doc owns the semantics.
3. Should `tags.write` accept an optional `quality` for scripts that mirror external data into memory tags (e.g. a weather feed that is stale)? Deferred; likely `kind: script` tags are the better answer (tag-model.md section 2).
4. `alarms.ack` by a script is an audited automated acknowledgement: is it acceptable under the alarm philosophy (04-alarms)? Default is off (no permission).
5. Type stubs generation: hand-written or generated from the `.proto` plus a small schema; undecided.
6. `tags.subscribe`-style long-lived callbacks are intentionally absent (declarative triggers instead). Revisit if integrators ask for state machines (v2 `long_running` scripts, 03 open questions).
