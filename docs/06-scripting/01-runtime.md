# 01 Runtime: Python alongside the Go core

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Contract proposals P1 to P7 are defined in [README.md](README.md); this file depends on P1 and P2.

## 1. The question

`skyrangerd` is Go. User scripts are Python (decisions.md 2). How does Python code run next to a Go process that must keep controlling the plant when a script misbehaves?

Requirements, in priority order:
1. A script bug (infinite loop, memory blowup, uncaught exception, native crash) must not stop tags, alarms, history or the HMI.
2. A runaway script must be killable, with certainty.
3. Scripts get no filesystem, network or subprocess access unless granted (see [04-sandbox-and-permissions.md](04-sandbox-and-permissions.md)).
4. Real CPython 3 with its standard library, because engineers know it and decisions.md 2 says "Python".
5. Boring, debuggable, cross-compiles to the edge boxes (amd64 and arm64 containers).
6. The author knows C, C++ and Python and is learning Go: the Go part must stay small and plain.

## 2. Options compared

| Option | Isolation | Kill a runaway | Real CPython + stdlib | Build/ops cost | Verdict |
|---|---|---|---|---|---|
| A. Embed libpython in `skyrangerd` via cgo | None: shared address space, native crash kills the core | No (cannot interrupt C code or a GIL-holding loop safely) | Yes | cgo toolchain, per-arch libpython, no static binary, GIL | Rejected |
| B. Pure-Go Python interpreter (e.g. gpython) | In-process, but memory-safe | Possible via context checks | No: partial language, small stdlib (UNVERIFIED current state) | Low | Rejected |
| C. Starlark (`go.starlark.net`) | Hermetic by construction | Yes (step limit, cancel) | No: Python-like dialect, no stdlib, no third-party libs | Low | Rejected for scripts; candidate for a future expression tier |
| D. CPython compiled to WASI, run under a Go WASM runtime (e.g. wazero) | Strong (WASM sandbox) | Yes | Partial; extension modules and threads limited, performance and maturity UNVERIFIED | Medium, unfamiliar | Rejected for v1, revisit (open questions) |
| E. Separate CPython worker processes, in their own container (**chosen**) | OS process, container, seccomp, no network | Yes: `SIGKILL` | Yes | Two small Go binaries plus a Python bootstrap | **Chosen** |
| F. Python as a REST/WS client in its own container | Container | Yes | Yes | Low | Rejected as the primary path (section 7), kept as a fallback |
| G. Jython | JVM | Weak | Python 2.7 only | JVM | Rejected |

Reasoning for E: killing a process is the only fully reliable way to stop arbitrary code, and a process boundary is the only boundary that also survives a native crash. This is the same decision the driver contract made for drivers (driver-interface.md section 1: "a hung PLC library, panic, or memory leak in one driver cannot take down the tag engine"), applied to scripts. We reuse the same pattern: supervised child processes, gRPC over a Unix domain socket, latest-value semantics, `slog` JSON logs.

Cost of E, stated honestly: every script call that touches tags is an IPC round trip (worker pipe, then gRPC to the host). Budget: well under 1 ms per call on the same machine (UNVERIFIED, to be measured in phase v1). Scripts are for orchestration, reports and glue, not for per-sample processing; per-sample math belongs in derived tag built-ins (tag-model.md section 7).

## 3. Process layout

```
skyrangerd container                         skyranger-scripts container
 skyrangerd (Go)                              skyranger-scriptd (Go, PID 1 child of tini-like init)
  script host: triggers, scheduler,             pool "default"
  permissions, budgets, run records             ├─ shim ─exec─► python3 worker #0
  gRPC server on host.sock                      ├─ shim ─exec─► python3 worker #1
        ▲                                       └─ ...
        └──────── skyranger.script.v1 over UDS ────┘
        /run/skyranger/scripts/default/host.sock   (shared volume, mode 0600, owner scriptd)
```

- **`skyrangerd` script host** (package `internal/scripthost`): owns configuration (`Script`, `ScriptLibrary`), triggers, queues, permission checks, budgets, run records, `_system` tags, audit. It contains no Python.
- **`skyranger-scriptd`** (Go, `cmd/skyranger-scriptd`): supervisor inside the scripts container. Keeps N workers alive, assigns runs, enforces wall-clock and memory limits, kills and respawns. It holds the per-run tokens; workers never see them.
- **Worker**: `python3 -I -S -X ...` (isolated mode) running the bundled `skyranger_runtime` bootstrap (Python standard library only, no pip install at runtime). One run at a time per worker.
- **Shim**: `skyranger-scriptd --sandbox-exec`, a re-exec of the same Go binary that applies rlimits, `no_new_privs` and a seccomp filter in the child, then `execve`s Python. This is how the filter exists before any user code can run. Details in 04.
- The scripts container is **optional**. Without it the gateway is fully functional with scripting unavailable (`_system/script_pools/default/state = "unavailable"`).
- Who dials whom: `skyrangerd` listens on `host.sock`; `skyranger-scriptd` dials in. (Drivers are the reverse, because a driver is a server the host launches; here scriptd is a client that must survive host restarts and reconnect.)

Worker lifecycle:
- Pool size `workers` (default 2, max 8). Idle workers are pre-started and have already imported the allowed stdlib modules and the current libraries.
- A worker is **recycled** after `max_runs` (default 1000), on any exceeded memory limit, on timeout (killed), on revision change of any library it loaded, and on any protocol error. Recycling is cheap; correctness never depends on a worker staying alive.
- Module-level state in a script is **not** a persistence mechanism. It may survive between runs on one worker and vanish at any time. Persistence is `state` (see 02).
- Restart backoff mirrors drivers: 1s, 2s, 4s ... max 60s, reset after 5 min stable.

## 4. Protocols

### 4.1 Host protocol `skyranger.script.v1` (scriptd to skyrangerd)

New gRPC package, same conventions as `skyranger.driver.v1` (proto3, `buf lint`/`buf breaking`, never renumber). Pull model for runs, unary for calls:

```protobuf
syntax = "proto3";
package skyranger.script.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/duration.proto";

service ScriptHost {
  // scriptd opens once; host streams config snapshots and run assignments.
  rpc Attach(AttachRequest) returns (stream HostMessage);
  // Worker calls, relayed by scriptd. run_token binds the call to one active run.
  rpc Call(CallRequest) returns (CallResponse);
  // End of run (also sent for killed runs, synthesized by scriptd).
  rpc Complete(CompleteRequest) returns (CompleteResponse);
  rpc Log(LogBatch) returns (LogResponse);
}

message AttachRequest { string pool = 1; string scriptd_version = 2; uint32 workers = 3; }

message HostMessage {
  oneof msg {
    ScriptSet script_set = 1;   // full set of enabled scripts + libraries for this pool, with revision
    RunAssignment run = 2;
    CancelRun cancel = 3;
  }
}

message RunAssignment {
  string run_id = 1;                 // UUIDv7
  string run_token = 2;              // opaque, unguessable, valid until Complete or deadline
  string script = 3;
  uint64 config_revision = 4;
  string source_sha256 = 5;
  google.protobuf.Duration timeout = 6;
  bytes trigger_json = 7;            // trigger object, see 03; int64/uint64 as strings
  bytes params_json = 8;             // manual-trigger params, may be empty
  bool dry_run = 9;
}

message CallRequest { string run_id = 1; string run_token = 2; string method = 3; bytes args_json = 4; }
message CallResponse { bytes result_json = 1; Error error = 2; }
message Error { string code = 1; string message = 2; map<string,string> details = 3; }   // codes = api.md section 7
```

(Draft-time validation and dry runs need the host to call *into* scriptd; see 06 and 07. Open question 3 asks whether that is a second service served by scriptd or a request/response pair on the `Attach` stream. The sketch above is illustrative and the `.proto` is authoritative once written.)

Rules:
- `Call.method` is a closed list (`tags.read`, `tags.write`, `history.read`, `events.emit`, `alarms.list`, `alarms.ack`, `http.request`, `state.get`, `state.set`, `log` etc., see 02). Unknown method: `bad_request`.
- Every call: host looks up `run_id`, compares `run_token` in constant time, checks the run is active and within its deadline, checks the permission and budget, then calls the **same Go service functions the REST handlers call**. The principal is `script:{pool}/{script}` (P4).
- Errors use the API error `code` values (api.md section 7), so a script sees the same codes as a REST client.

### 4.2 Worker pipe protocol (scriptd to worker)

Two pipes inherited at fd 3 (to worker) and fd 4 (from worker). Newline-delimited JSON, one object per line, max line 8 MiB. stdout and stderr are **not** protocol: scriptd captures them as log lines (`stream: "stdout"|"stderr"`).

```json
{"type":"run","run_id":"01933b90-...","script":"daily_report","timeout":"10s","trigger":{...},"params":{}}
{"type":"call_result","id":7,"result":{...}}
{"type":"call_result","id":8,"error":{"code":"forbidden","message":"..."}}
```
```json
{"type":"call","id":7,"method":"tags.read","args":{"paths":["site1/pumphouse/meter01/flow_rate"]}}
{"type":"log","level":"info","msg":"computed","fields":{"liters":1234.5}}
{"type":"done","ok":true,"result":null}
{"type":"done","ok":false,"error":{"type":"ValueError","message":"...","traceback":"..."}}
```

`run_token` is absent from the pipe by design: scriptd attaches it when relaying `call`. Values follow the contract encodings (int64/uint64 as strings on the wire; the Python runtime converts to `int`, see 02 section 2).

## 5. Go sketches

Supervisor core: one goroutine per worker, one run at a time, kill on deadline.

```go
package pool

type Worker struct {
    cmd   *exec.Cmd
    toW   *json.Encoder  // fd 3
    fromW *bufio.Reader  // fd 4
    runs  int
}

// Run executes one assignment. It always returns a Result; it never panics the supervisor.
func (p *Pool) Run(ctx context.Context, w *Worker, a *scriptv1.RunAssignment) Result {
    deadline := a.Timeout.AsDuration()
    hard := time.AfterFunc(deadline+p.cfg.KillGrace, func() { // grace 2s: soft timeout fires inside Python first
        w.cmd.Process.Kill() // SIGKILL: the only reliable stop
    })
    defer hard.Stop()

    if err := w.toW.Encode(runMsg(a)); err != nil {
        return Result{Status: "error", Err: "worker_io"}
    }
    for {
        line, err := w.fromW.ReadBytes('\n')
        if err != nil { // EOF: worker died or was killed
            return p.classifyDeath(w, a, deadline)
        }
        switch msgType(line) {
        case "call":
            // relay to host with the run token; reply on fd 3
            resp := p.host.Call(ctx, a.RunId, a.RunToken, line)
            w.toW.Encode(resp)
        case "log":
            p.logs.Add(a.RunId, line)
        case "done":
            return parseDone(line)
        }
    }
}
```

Memory limit: a goroutine samples `/proc/{pid}/status` `VmRSS` every 250 ms and kills at `memory_limit` (default 256 MiB per worker); the container's cgroup memory limit is the outer net. `RLIMIT_AS` is not used as the primary control because virtual-address limits interact badly with CPython and its allocators (UNVERIFIED for the pinned version; measure).

Host side, trigger to run:

```go
package scripthost

type Run struct {
    ID       uuid.UUID
    Script   *ScriptRev  // immutable snapshot: name, revision, source hash, permissions, limits
    Trigger  Trigger
    Params   json.RawMessage
}

func (h *Host) enqueue(s *ScriptRev, t Trigger) {
    q := h.queues[s.Name]
    switch q.Offer(t) { // coalesce / drop policy from 03
    case Dropped:
        h.metrics.Dropped(s.Name)
    }
}

func (h *Host) dispatch(ctx context.Context, a *scriptv1.RunAssignment) { /* push on Attach stream */ }
```

Python bootstrap (stdlib only), the call primitive every API function uses:

```python
# skyranger_runtime/_ipc.py
import json, os, threading
_in = os.fdopen(3, "r", encoding="utf-8", buffering=1)
_out = os.fdopen(4, "w", encoding="utf-8", buffering=1)
_lock = threading.Lock()
_next = 0

def call(method: str, **args):
    global _next
    with _lock:
        _next += 1
        _out.write(json.dumps({"type": "call", "id": _next, "method": method, "args": args}) + "\n")
        reply = json.loads(_in.readline())
    if "error" in reply:
        raise _errors.from_code(reply["error"]["code"], reply["error"]["message"])
    return reply["result"]
```

## 6. Failure behavior (what the plant sees)

| Failure | Effect on core | Effect on scripts |
|---|---|---|
| Script raises | None | Run recorded `error`, event `skyranger.script.run_failed.v1`, counts toward quarantine |
| Script loops forever | None | Soft timeout in Python at `timeout`; `SIGKILL` at `timeout + 2s`; worker respawned; run `timeout` |
| Script allocates without bound | None | scriptd kills worker at limit; run `killed` (`reason: memory`) |
| Worker segfaults | None | Run `killed` (`reason: crash`) |
| scriptd or scripts container down | None | Pool state `unavailable`; triggers queue up to depth then drop (counted); on return, `Attach` replays current script set |
| `skyrangerd` restarts | Normal restart | scriptd reconnects; in-flight runs are cancelled (worker killed); runs are **not** resumed |
| Postgres down | Per core behavior | Run records buffered in memory (bounded), then dropped with a counter |

## Decisions made
1. CPython 3 in separate worker processes inside a separate network-less container; no embedded interpreter in `skyrangerd`.
2. A small Go supervisor (`skyranger-scriptd`) per pool; workers are pooled, recycled, and killed with `SIGKILL` on timeout or memory excess. A soft in-Python timeout fires first so well-behaved scripts unwind with a clear error.
3. Triggers and permissions live in Go (host); Python only executes assigned runs and cannot reach anything except through `Call`.
4. New gRPC package `skyranger.script.v1` over a Unix domain socket: scriptd dials the host; runs are pushed over a server stream, calls are unary and carry an unforgeable per-run token that workers never see.
5. Worker protocol is JSON lines on fd 3/4, stdlib-only bootstrap. stdout and stderr are log capture, never protocol.
6. Module-level Python state is explicitly not persistent; persistence is the `state` API.
7. The scripts container is optional; the core is unaffected when it is absent or failing.

## Alternatives rejected and why
- **cgo-embedded CPython**: cannot stop a runaway, a native crash takes down the core, cgo complicates arm64 cross-builds and static binaries.
- **Pure-Go interpreters (gpython)**: not the Python engineers know; library support is thin (UNVERIFIED current state).
- **Starlark**: excellent for hermetic config logic and a real candidate for a later "safe expression" feature, but no standard library and not Python as users expect it.
- **CPython on WASM**: attractive sandbox, but maturity of stdlib, threading and performance for this use is UNVERIFIED and the toolchain is unfamiliar. Revisit once measured.
- **Fresh process per run**: strongest isolation and simplest, but startup cost on every tag-change run (tens of ms, UNVERIFIED) and no warm imports. Kept as a possible per-script option `isolation: run` in v2.
- **Python as a REST/WebSocket client (original contract wording)**: see section 7 and P1; loses tag-change events under load and cannot give per-run identity without extra machinery.
- **gRPC bidirectional stream for everything**: harder to debug than unary calls plus one server stream; correlation logic for no gain.
- **Threads or asyncio in user scripts**: complicates timeouts and the single-pipe protocol. Scripts are synchronous run-to-completion handlers. `threading` is not importable by user code (see 04).

## 7. Relationship to contract wording (P1)

messages.md section 3 says "The Python scripting runtime will use the REST API for writes", and 07-security/02-authn-authz.md (`svc-python-runtime`, role `operator`) assumes that. P1 replaces this with the script host. Reasons: (1) tag-change triggers must not be lost to QoS 0 retained topics or 100 ms WebSocket batching (api.md section 6), because "run this when the total changes" needs the same in-process delivery the tag engine gives alarms; (2) per-run identity and per-script permissions are enforced where the data is, rather than trusting a long-lived `operator` API key held in a process that runs user code; (3) the worker needs no sockets at all. The REST API is unchanged; external Python clients use it. **Fallback if P1 is rejected:** scriptd becomes a REST/WS client with one API key per pool and enforces permissions itself; triggers consume `skyranger/v1/...` MQTT or WS. Weaker, and documented as a degraded mode only.

## Open questions
1. P1 needs an owner decision and a decisions.md entry; 07-security/02 must then drop the `svc-python-runtime` REST key.
2. Measure: worker cold start, per-call latency over pipe plus UDS, idle memory per worker on the target edge box (UNVERIFIED all). Decide pool defaults from numbers.
3. Host-to-scriptd requests (validate, dry run): a second service on `scriptd.sock`, or request/response messages on the `Attach` stream.
4. Which CPython minor to pin (newest minor shipped by the base image at release, pinned by digest; UNVERIFIED which).
5. WASM-hosted CPython as the v3 isolation upgrade: measure when the toolchain is stable.
6. Should unprivileged user namespaces be used so that scriptd and workers run as different UIDs without `CAP_SETUID`? (Related to 04 section 3.)
