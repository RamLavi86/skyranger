# 07 Testing scripts

Status: design draft. Conforms to [../00-contracts/](../00-contracts/README.md). Builds on the simulator and layers of [../09-testing/](../09-testing/README.md) (notably [03-simulator.md](../09-testing/03-simulator.md): scenarios, virtual time, control API, oracle log; layers named there as L3 in-process harness, L4 compose integration). API in [02-script-api.md](02-script-api.md).

An integrator who ships scripts to a water utility needs three kinds of evidence: the logic is right (unit), the script behaves on a gateway without hurting anything (dry run), and it behaves under the faults real plants produce (integration with the simulator).

## 1. Three levels

| Level | Where it runs | Needs | Speed | Proves |
|---|---|---|---|---|
| T1 Offline unit tests | developer laptop or CI, plain `pytest` | `pip install skyranger-testing` (wheel built from this repo; no gateway) | milliseconds | logic, quality handling, budgets, error paths |
| T2 Dry run | on a live gateway (or the L4 stack) | engineer role, scripts container | seconds | the script loads, its imports and permissions are valid, its reads work against real data, its intended writes (recorded, not executed) are what you expect |
| T3 Integration with the simulator | compose stack with `skyranger-sim` | docker | seconds to minutes (virtual time) | end-to-end triggers, real writes, faults: stale device, reboot, counter reset, driver loss |

## 2. T1: offline unit tests (`skyranger.testing`)

A pure-Python package that implements the **same `skyranger` module surface** against an in-memory fake gateway, so a script file imports unchanged. It is the Python half of the API contract and is versioned with it (10).

```python
# tests/test_daily_report.py
from datetime import datetime, timedelta, timezone
import pytest
from skyranger.testing import FakeGateway, load_script

METER = "site1/pumphouse/meter01"

@pytest.fixture
def gw():
    g = FakeGateway(clock=datetime(2026, 11, 8, 0, 5, tzinfo=timezone.utc))
    g.add_tag(f"{METER}/total_volume", datatype="float64", unit="L")
    g.add_tag(f"{METER}/yesterday_l", datatype="float64", unit="L", kind="memory", writable=True)
    return g

def test_writes_yesterday_total(gw):
    gw.set_history(f"{METER}/total_volume", [
        ("2026-11-07T00:00:00Z", 1_000_000.0, "good"),
        ("2026-11-07T23:59:00Z", 1_004_821.0, "good"),
    ])
    script = load_script("scripts/daily_report.py", gw)      # imports with the fake `skyranger`
    script.run(trigger=gw.timer(scheduled_for="2026-11-08T00:05:00Z"))

    assert gw.last_write(f"{METER}/yesterday_l").value == 4821.0
    assert gw.events("skyranger.user.daily_report_done.v1")

def test_uncertain_data_is_not_written(gw):
    gw.set_history(f"{METER}/total_volume", [("2026-11-07T00:00:00Z", 1.0, "uncertain_last_known"),
                                              ("2026-11-07T23:59:00Z", 2.0, "good")])
    load_script("scripts/daily_report.py", gw).run(trigger=gw.timer(scheduled_for="2026-11-08T00:05:00Z"))
    assert gw.writes == []
    assert gw.events("skyranger.user.daily_report_incomplete.v1")
```

`FakeGateway` provides:
- `add_tag`, `set_value(path, value, quality="good", ts=...)`, `set_history`, `set_alarm`, `advance(timedelta)` (the fake clock behind `skyranger.now()`).
- Trigger builders: `gw.tag_change(path, new, old)`, `gw.timer(...)`, `gw.event(type, data, subject)`, `gw.startup()`, `gw.manual(params)`.
- Recorders: `gw.writes`, `gw.events(...)`, `gw.http_calls`, `gw.logs`, `gw.state`.
- `gw.http.stub("POST", "https://hooks.example.com/...", status=200, json={})`, unstubbed calls fail the test.
- **Fidelity checks**, because a fake that is more lenient than the host is worse than none: the fake enforces the datatype rules (`TypeMismatch`), `writable`, `write_limits`, permissions from the script's YAML (pass `permissions=` or load the `Script` document), the import allowlist (importing `os` fails in the test), run budgets (`BudgetExceeded`) and `ScriptTimeout` (simulated with a fake-clock deadline). `skyranger.testing.conformance` is a shared table of API behaviors run against **both** the fake and a real gateway in CI (T3), so drift is caught.
- `load_script` takes a `.py` path (split layout, 06) or a `Script` YAML document; with a YAML document it also checks triggers and permissions against what the fake saw (`assert_permissions_sufficient()` reports permissions the test never used and calls denied by missing ones).

Pytest, `ruff`, `mypy --strict` work on the split layout directly. The fake does **not** run the sandbox (no seccomp, no process isolation): T1 proves logic, not containment.

## 3. T2: dry runs on a live gateway

```
POST /api/v1/scripts/dry-run        (engineer)
{
  "document": { "apiVersion": "skyranger/v1", "kind": "Script", "metadata": {"name": "low_flow_flag"}, "spec": { ... } },
  "trigger": { "kind": "tag_change", "path": "site1/pumphouse/meter01/flow_rate",
               "new": { "value": 0.4, "quality": "good", "ts": "2026-11-08T10:00:00.000Z" } },
  "params": {}
}
```

Also `POST /api/v1/scripts/{name}/run` with `"dry_run": true` for a saved script. Semantics (`ctx.dry_run == True`):

| Call | Behavior in dry run |
|---|---|
| `tags.read`, `tags.read_many`, `tags.browse`, `history.read`, `events.query`, `alarms.active`, `state.get` | real, with the script's permissions |
| `tags.write`, `tags.write_many` | **not executed**; validated (datatype, writable, `write_limits`, permission) and recorded; returns a `WriteResult` with `status="ok"` and `dry_run=true` |
| `events.emit`, `alarms.ack`, `state.set/delete` | not executed; validated and recorded |
| `http.*` | not executed unless the allowlist entry has `dry_run_allow: true` (opt-in, GET only); otherwise returns a synthetic `HttpResponse(status=204)` and records the would-be request (host, method, path, body size) |
| `log` | captured and returned |

The response is the run record plus `effects: [{"call": "tags.write", "path": ..., "value": ..., "validated": true, "error": null}, ...]`. A dry run uses the real worker pool (so the sandbox, imports and limits are real), takes the unsaved document inline (so the draft's permissions are tested), and is audited (`script.dry_run`) without producing writes. Dry-run records are kept with `dry_run: true` for 7 days.

Honest limits: a dry run cannot predict behavior that depends on the effect of its own writes (read-after-write). Scripts that do that need T3.

## 4. T3: integration tests against the simulator

Uses the compose stack of the integration layer (L4) with `skyranger-sim` publishing real Sparkplug B, and the scripts container enabled. Tests drive the gateway only through the public REST API (api.md), like any client.

Scenario (`skyranger-sim` YAML, schema owned by 09-testing/03-simulator.md; shape abbreviated here):

```yaml
scenario:
  name: leak-script
  seed: 42
  time: { mode: virtual }          # or scaled with a compressed config profile (03-simulator.md)
devices:
  - id: esp32-meter01
    model: pulse_meter
    params: { lpm: 12 }
timeline:
  - at: 10m   action: reboot       # reset event; the reset_audit script must emit meter_reboot_noted
  - at: 30m   action: network_down # device keeps counting; liveness goes 1 then 0
```

Test (pytest + `httpx`, using fixtures from the integration harness; fixture names illustrative):

```python
def test_reset_audit_script_fires_on_reboot(stack):          # stack: compose + sim + gateway with config applied
    stack.apply_config("fixtures/meter_with_reset_audit.yaml")   # includes the Script document
    stack.sim.start("scenarios/leak-script.yaml")
    stack.sim.advance("11m")                                  # control API, virtual time
    ev = stack.wait_event("skyranger.user.meter_reboot_noted.v1", timeout="10s")
    assert ev["data"]["path"].endswith("meter01/total_volume")
    run = stack.api.get("/api/v1/scripts/reset_audit/runs?status=ok").json()["items"][0]
    assert run["config_revision"] == stack.revision
```

Fault matrix scripts should be exercised under (choose from the fault catalogue in 09-testing/04-fault-injection.md; names there are authoritative):
- **stale/dead device**: scripts reading `flow_rate` must not treat `uncertain_last_known` as zero flow.
- **connection loss** (`bad_not_connected` everywhere): scripts see non-good quality, do not write garbage.
- **reboot and rollover**: scripts reacting to `counter_discontinuity` events.
- **burst**: tag-change trigger flood; the script stays within queue/limits, drops are counted, the core is unaffected.
- **scripts container killed mid-run** (kill the container): core unaffected, pool `unavailable`, triggers drop and count, recovery re-attaches (the 01 failure table as a test).
- **script misbehavior fixtures** (a library of intentionally bad scripts, below).

### Sandbox and limits regression suite

A fixed set of hostile/buggy scripts that must fail in the documented way (run against the real container on every release; 09-testing owns CI wiring):

| Fixture | Expected |
|---|---|
| `while True: pass` | `timeout`, worker killed, core latency unaffected (measured) |
| allocates 2 GiB | `killed (memory)` |
| `import os`, `import socket`, `open("/etc/passwd")` | `Forbidden`/import denied, no effect |
| raw `socket.socket()` via `ctypes` attempt | seccomp denies (`EPERM`), test asserts at syscall level |
| tight `tags.write` loop | `BudgetExceeded` after the per-run cap |
| write to a tag it triggers on | no self-trigger (`ignore_own_writes`), depth limit for A/B loops |
| `http.post` to non-allowlisted host, to `169.254.169.254`, to a private IP without `allow_private` | `HttpDenied` |
| forged `Call` with a wrong or stale run token (test client against `host.sock`) | rejected, audited |
| `except BaseException: pass` around a busy loop | hard kill at deadline + 2s |

## 5. What the author runs, in order

1. `pytest` on the laptop (T1) while writing.
2. Dry run against the commissioning gateway (T2) after `config draft` validation.
3. T3 before tagging a customer release; the evidence pack (09-testing/11) includes the script fixtures' results.

## Decisions made
1. Three levels: offline `skyranger.testing` (T1), dry run (T2), simulator integration (T3). Each answers a different question and none replaces another.
2. The fake gateway implements the real API surface and enforces the real rules (types, permissions, budgets, import allowlist); a shared conformance table runs against both fake and real gateway to prevent drift.
3. Dry run uses the real worker pool and real reads; writes, events, state and HTTP are validated and recorded but not executed.
4. Integration tests go through the public API only, with the simulator's virtual time and control API (09-testing/03-simulator.md).
5. A maintained suite of hostile scripts proves the sandbox and limits on every release.
6. Scripts live as real `.py` files in the split layout so ordinary Python tooling applies.

## Alternatives rejected and why
- **Mock-by-patching in user tests** (`unittest.mock.patch("skyranger.tags.read")`): couples tests to internals and skips type and permission rules.
- **Only integration tests**: slow feedback and hard to cover error paths.
- **Dry run that executes writes against a shadow copy of tags**: needs a second tag engine instance; the recorded-effects model is simpler and honest about its limit.
- **Letting dry-run `http` execute by default**: webhooks would fire (pages someone at 3 a.m.).
- **A separate test DSL for scripts**: pytest already exists and every Python engineer knows it.
- **Real-time simulator runs in CI**: slow; virtual time exists for this reason.

## Open questions
1. Simulator fidelity for time: scripts using `skyranger.now()` and cron triggers need the gateway's clock to follow the simulator's virtual time, which the simulator doc treats as device-side time only. How does a gateway under test advance its scheduler (a test-mode clock, or only scaled-time scenarios)? Needs agreement with 09-testing. UNVERIFIED that it is feasible without invasive hooks.
2. Packaging the testing wheel: published on PyPI, or vendored from the repo (offline-friendly integrators)?
3. `FakeGateway.history` semantics for aggregates: reuse the Go aggregation code via a shared test vector file instead of reimplementing in Python (golden files per 09-testing/01).
4. Should dry run of `startup`/`timer` triggers be allowed with simulated clock offsets (`scheduled_for` supplied by the caller)? Currently yes: the caller supplies the trigger object.
5. Coverage reporting for scripts (coverage.py inside the worker): useful, but out of v1.
