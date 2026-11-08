# 06 Integration and End-to-End Tests (L3 harness, L4, L5)

This file covers the three layers that run the real product together: the in-process harness (L3), the Docker Compose integration stack (L4), and browser tests of the HMI (L5). Faults and scenarios are defined elsewhere (files 03, 04); this file defines the machinery that runs them, what runs per pull request versus nightly, and what is collected when something fails.

## 1. L3: the in-process harness (`test/harness`)

Already specified in the README at layer level. Concrete API surface, so agents and humans write tests the same way:

```go
h := harness.New(t,
    harness.Scenario("test/scenarios/wm-trace-contract.yaml"), // or harness.Config(yamlBytes)
    harness.Seed(1),
    harness.History(harness.HistoryMem),   // HistoryMem (default) | HistoryPG (needs SKYRANGER_TEST_PG)
)
h.RunFor(10 * time.Minute)                 // advances clock.Fake, pumping the sim and the engine
h.RunUntil(t0.Add(2 * time.Hour))
h.Eventually(2*time.Minute, func(c *harness.C) { c.TagQuality(path, tag.QualityUncertainLastKnown) })
h.AssertTagEqualsOracle(path, meter.Oracle().ExpectedTotal())
h.API()    // *apiclient.Client against the real REST/WS server on a loopback port, real auth
h.Events() // subscription to the internal event stream; assertions go through h.AssertEvent*
```

Rules:
1. **Assertions go through public surfaces**: REST, WS, events, internal bus (file 04 decision 2). The harness exposes no handle to engine internals; if a test needs one, the product is missing an observable (add a `_system` tag or API field, do not add a back door).
2. **One clock**: engine, drivers (in-process), simulator, API server timers and the alarm engine all take `h.Clock`. A component with its own `time.Now` is a harness bug, caught by the lint rule in file 01.
3. **Both history modes run the same scenarios**: `HistoryMem` per commit, `HistoryPG` nightly in the L4 job (README open question, proposal adopted here).
4. **Everything seeded and logged**: on failure the harness prints `seed`, scenario file, virtual time, and the last 50 events, so a failure reproduces from one command line: `go test ./test/harness -run TestX -seed=1234`.
5. **Hang guard**: `h.RunFor` fails the test if the virtual clock does not advance within 5 s of wall time (deadlock detector), and every harness test runs under `-timeout 60s` per package.
6. Speed budget: the whole L3 suite is 5 minutes on a 4-core runner (README table). A test that needs more than 5 s of wall time must justify itself in a comment, or move to L4.

## 2. L4: the Docker Compose integration stack

### 2.1 Services

Stack file `test/compose/compose.yaml` (an excerpt; the production compose file in 08-deployment is separate and is also tested, see section 2.5).

```yaml
name: sr-it-${RUN_ID:-local}
services:
  skyrangerd:
    image: ${SKYRANGER_IMAGE}              # always the CI-built image by digest, never a rebuild
    depends_on:
      timescaledb: { condition: service_healthy }
      mosquitto:   { condition: service_healthy }
    environment: { SKYRANGER_GATEWAY_ID: gw01 }
    ports: ["8443"]                         # random host port; the harness discovers it
    healthcheck: { test: ["CMD", "skyrangerd", "healthcheck"], interval: 2s, retries: 30 }   # command name UNVERIFIED (deployment doc owns)
    mem_limit: 1g
  mosquitto:
    image: eclipse-mosquitto:${MOSQUITTO_VERSION}   # pinned digest in the lockfile
    ulimits: { nofile: 65536 }
  timescaledb:
    image: timescale/timescaledb:${TSDB_VERSION}    # pinned; the version the product ships with
    tmpfs: ["/var/lib/postgresql/data:size=1g"]     # fast, disposable; fault tests use a real volume
  sim:
    image: ${SKYRANGER_SIM_IMAGE}
    command: ["--scenario", "/scenarios/${SCENARIO}", "--seed", "${SEED}", "--control", "0.0.0.0:9900", "--i-know-this-is-a-test-tool"]
    # control API reachable on the compose network only, never published (file 03 section 7)
  toxiproxy:
    image: ghcr.io/shopify/toxiproxy:${TOXIPROXY_VERSION}
  tap:
    image: ${SKYRANGER_CAPTURE_IMAGE}               # MQTT tap (file 05)
  # profiles
  netem-sidecar:
    profiles: ["fault-net"]
    network_mode: "service:skyrangerd"
    cap_add: [NET_ADMIN]                            # test-only; never in the product stack
  chrome-tests:
    profiles: ["e2e"]
    image: mcr.microsoft.com/playwright:${PLAYWRIGHT_VERSION}   # image name UNVERIFIED; check at build time
```

The gateway talks to the broker and the database **through Toxiproxy** in the fault profiles (`skyrangerd -> toxiproxy:1883 -> mosquitto`), and directly in the smoke profile, so happy-path tests do not carry proxy risk.

### 2.2 Go test driver (`test/integration`, build tag `integration`)

```go
//go:build integration

func TestMain(m *testing.M) { os.Exit(stack.Main(m)) }   // brings the stack up once per package, collects artifacts on exit

func TestSmoke_MeterEndToEnd(t *testing.T) {
    s := stack.Shared(t)                                  // shared, read-mostly stack (section 2.3)
    s.ImportConfig(t, "test/compose/config/smoke.yaml")    // via the public config API as role engineer
    sim := s.Sim(t)
    sim.Start(t, "wm-trace-contract-realtime.yaml")

    s.Eventually(t, 90*time.Second, func(c *stack.C) {
        c.TagQuality("site1/pumphouse/meter01/total_volume", tag.QualityGood)
    })
    sim.Wait(t)
    s.AssertTagEqualsOracle(t, "site1/pumphouse/meter01/total_volume", sim.Oracle(t).Device("meter01").ExpectedTotal)
    s.AssertHistoryNoGapsLongerThan(t, "site1/pumphouse/meter01/total_volume", 16*time.Minute)
}
```

Authentication: the harness uses a service API key (`sr_...`, role `engineer`, from the harness account `svc-test-harness` in 07-security/02-authn-authz.md). That account never exists in a release image (checked in the release gate, file 10).

### 2.3 Isolation and reset

| Test class | Stack | Reset between tests | Why |
|---|---|---|---|
| Smoke, read-mostly (scenario replays, API checks) | One shared stack per `go test` package | Import config (empty) + unique tag base path per test + sim reset via control API | Startup of timescaledb and migrations is the expensive part |
| Fault and restart tests (kill, stop, disk full, DB loss) | Fresh stack per test (`docker compose -p sr-it-<test-id> up`, `down -v` after) | Whole stack | A previous test's damage must not explain the next result |
| Perf, soak | Dedicated stack on reference hardware | n/a | file 07 |

Timing: stack start and healthy in at most 60 s (target to be measured, UNVERIFIED); if exceeded, the harness fails with "stack startup" rather than letting every test time out.

### 2.4 Failure artifacts (always collected, uploaded by CI on failure and nightly)

- `docker compose logs --timestamps` for every service, separated per service.
- The simulator oracle JSONL and its seed, the scenario file used.
- The MQTT tap capture (file 05), because it turns any L4 failure into an L3 replay candidate.
- A PostgreSQL dump of the `events`, `audit_log`, `config_revisions` tables and history for the test's tags (`pg_dump` with table filters).
- `GET /api/v1/...` snapshots: system status, connections, alarms, current values of the tags under test.
- For fault tests, the Toxiproxy toxic list and the timeline of injected faults with timestamps.
- Playwright trace and video (section 3) for L5.

Rule: a failing L4 test with no artifacts is a test bug.

### 2.5 What runs when

| Set | Contents | Trigger | Budget |
|---|---|---|---|
| **L4 smoke** | S1 stack boots, healthy, migrations applied. S2 `wm-trace-contract` in real-time compressed profile with exact oracle match. S3 reboot vs reconnect (F-DEV-01, F-DEV-02). S4 broker restart (F-BRK-01). S5 DB stop 60 s and restore (F-DB-01, short). S6 `skyrangerd` SIGKILL mid-flow, conservation exact (F-GW-04, F-CNT-05). S7 unapproved device creates no tag (F-DEV-08). S8 config export, wipe, import: empty diff | every PR | 5 min |
| **L4 full** | Every catalogue entry whose `tiers_required` contains L4, `fleet-500`, `mixed-plant`, replay corpus (file 05), PostgreSQL variant of all L3 scenarios | nightly | 60 min |
| **L4 production-stack check** | The *production* compose file and image set from 08-deployment started as a customer would, with the sim attached, a smoke subset run against it, `docker compose config` validated | nightly and release | 10 min |
| **L4 upgrade/rollback** | Start the previous release's images with the previous schema and data (generated by a fixture run), upgrade to the candidate, verify config revision, counters, alarms, history intact; then roll back per the secure-update procedure | release | 15 min |
| **L4 backup/restore** | Backup, destroy volumes, restore, compare history hash and config diff | release | 10 min |

Time compression: `test/compose/profiles/compressed-x10.yaml` (file 03 section 3) is allowed in smoke and nightly for logic. Evidence-pack timing claims come only from real-time runs (file 03 decision 4).

### 2.6 Determinism limits at L4

Real time, real broker, real DB mean L4 asserts with tolerances. Every tolerance is a named constant in `test/integration/tolerances.go` with a comment on why that number, and is a protected path (loosening a tolerance is test-weakening, file 12). Example: `WSLatencyP99 = 500 * time.Millisecond` for the smoke stack. Never assert on exact wall-clock instants; assert on ordering and on bounds.

## 3. L5: HMI browser tests (Playwright)

### 3.1 Principles

1. The HMI uses only the public API (api.md), so L5 tests the same surface customers' scripts use, from a user's point of view. They check what an operator sees and can do, not pixel details.
2. **Few and valuable**: 2% of tests (README). Each test maps to an operator-visible requirement or a safety-relevant display rule (quality shown honestly). Anything that can be tested in Vitest or against the API goes there instead.
3. **No retries in gating runs** (`retries: 0`), traces on first failure.
4. **Selectors**: `getByRole`, `getByLabel`, and `data-testid` for dense widgets. No CSS-class or XPath selectors. Agents adding components must add the test ids (HMI component docs, 05-hmi-api).
5. **Contract tap**: a Playwright fixture records every API response the page receives and validates it against `GET /api/v1/openapi.json` and the golden shapes. A UI that works only because the server misbehaves in a way the contract forbids fails here.
6. **Accessibility** is part of the test: `@axe-core/playwright` runs on every screen visited by the smoke tests (a real Playwright companion library; exact API UNVERIFIED), failing on serious or critical violations. Detail rules belong to 05-hmi-api/07-i18n-a11y.
7. **No pixel-diff gating.** Visual snapshots are brittle and agents "fix" them by re-snapshotting. Screenshots are kept as artifacts for humans. The sole exception is the quality-colour legend page (rendered per quality class), compared to a reviewed baseline, because colour semantics are a safety display rule.

### 3.2 Test list

| ID | Test | Key assertions | Set |
|---|---|---|---|
| HMI-01 | Login, logout, wrong password | Bearer token never appears in URL or localStorage dump containing the password; wrong password shows RFC 9457 `detail` text without leaking whether the user exists (07-security decides the exact wording) | smoke |
| HMI-02 | Role gating: `viewer` sees no write controls, `operator` can write a memory tag and ack an alarm, `engineer` can open the config editor | controls absent in DOM (not only disabled) for forbidden roles; direct URL to forbidden page shows `403` screen; API returns `403` if the test forces the call | smoke |
| HMI-03 | **No flow versus dead device** (`wm-no-flow-vs-dead`) | Heartbeats with zero flow: flow reads 0 and health shows ok. Device silent: value greyed, health shows stale, then offline. Network down: shows "no connection" distinct from device offline | smoke |
| HMI-04 | Bad and uncertain quality rendering | For each quality class (`good`, `uncertain_*`, `bad_*`), the value cell exposes the quality in text or accessible name, not colour only; last known value shown for bad quality; `null` value shows a placeholder, not "0" | smoke |
| HMI-05 | Live update path | Change a source value via sim; the displayed value changes within `WSLatencyP99` plus batching; the displayed timestamp equals the sample `ts` | smoke |
| HMI-06 | Leak alarm lifecycle | Alarm appears in banner/list at `active_unacked`; operator acks: `active_acked`; flow stops: `cleared_unacked`; a further ack returns it to `normal`. States match the contract names in text; audit entry visible to `engineer` | nightly (smoke: activation only) |
| HMI-07 | **Gateway restart / WS drop** (F-API-02) | "Connection lost" banner appears, all values greyed, no stale value presented as live; after reconnect, fresh snapshot and banner cleared; no duplicate alarm rows | nightly |
| HMI-08 | Slow link behavior | Playwright throttles network (CDP, Chromium only); UI stays usable, no blank screen, "last update" indicator advances | nightly |
| HMI-09 | Discovery and approval flow | New sim node appears under Discovered as `pending`; approve with template; diff shown; apply; tags appear; revision visible in config history | nightly |
| HMI-10 | Config edit safety | Invalid config (dangling reference) is rejected with a visible error and nothing applied; a valid change shows diff before apply | nightly |
| HMI-11 | Manual total write (F-CNT-07) | Operator-with-engineer-role writes `total_volume`, enters a comment, result shown, `manual_adjust` event listed | nightly |
| HMI-12 | History trend | Select tag and range; chart shows data points matching `/api/v1/history` for the same range (compare plotted series from the page's own data layer exposed under a test hook, not from pixels); gap rendered where history has a gap | nightly |
| HMI-13 | Session expiry and revocation (F-API-03) | After logout from another session, page returns to login and discards cached values | nightly |
| HMI-14 | Large screen / many tags | 2,000-tag list renders and scrolls without freezing (main-thread long task under a bound, measured via Performance API); live updates continue | weekly |
| HMI-15 | Cross-browser | HMI-01..05 on Chromium (every PR), Firefox (nightly) | nightly |

WebKit is not run in v1 (UNVERIFIED whether the target customers use Safari on operator stations; plant HMIs are typically Chrome/Edge). Revisit on the first customer that says otherwise.

### 3.3 Sketch

```ts
// hmi/e2e/tests/no-flow-vs-dead.spec.ts
import { test, expect } from '../fixtures';   // fixtures: loggedInAs(role), sim client, contract tap

test('HMI-03: zero flow is not a dead meter', async ({ page, sim, loggedInAs }) => {
  await loggedInAs('viewer');
  await sim.start('wm-no-flow-vs-dead.yaml');                 // seeded, compressed profile
  await page.goto('/screens/pumphouse');

  const flow   = page.getByTestId('tag-site1/pumphouse/meter01/flow_rate');
  const health = page.getByTestId('tag-site1/pumphouse/meter01/health');

  await expect(health).toHaveAttribute('data-quality', 'good');
  await expect(health).toContainText(/ok/i);
  await expect(flow).toContainText('0');                      // zero flow, heartbeats alive

  await sim.action('meter01', 'stop_publishing');             // link up, device silent
  await expect(health).toContainText(/stale/i, { timeout: 60_000 });
  await expect(flow).toHaveAttribute('data-quality', 'uncertain_last_known');

  await sim.action('meter01', 'network_down');
  await expect(health).toContainText(/offline/i, { timeout: 90_000 });
  await expect(page.getByRole('status', { name: /connection/i })).toHaveCount(0); // the gateway link is fine; only the device is gone
});
```

(The `data-quality` attribute is a proposed test hook: HMI components expose the quality name as an attribute. It is a contract between the HMI docs and this file; see Open questions.)

### 3.4 Running L5

- Playwright runs in the `e2e` compose profile against the same stack as L4 smoke, so one stack start serves both.
- Per PR: Chromium, tests tagged smoke, 5 minutes. Nightly: Chromium and Firefox, everything except weekly tests.
- Artifacts: trace zip on any failure, video for nightly, HAR, the contract tap's violation list, accessibility violation JSON.
- Console errors fail the test (`page.on('console')` for `error`, `pageerror`), unless allow-listed in `hmi/e2e/allowed-console.txt` (protected, with a reason per line).

## 4. Cross-layer rules

- A scenario file is written once and reused: L3 (virtual), L4 (compressed or real), L5 (HMI assertions). `faults_covered` metadata makes the fault coverage table (file 04) honest across layers.
- Every layer runs against **the same build artifact it will ship as**: L4 and L5 use the CI-built image by digest. Rebuilding for tests is forbidden (file 10).
- If an L4 or L5 test fails and the same scenario passes at L3, the finding is about real time, real broker or real DB behavior. Triage starts from the tap capture, and the fix includes a replay test (file 05).

## Decisions made
1. L3 harness exposes only public surfaces, shares one fake clock, and runs both in-memory and PostgreSQL history variants.
2. L4 is Docker Compose with Toxiproxy, a netem sidecar profile, an MQTT tap, and images pinned by digest. Fault tests get a fresh stack each; smoke tests share one.
3. Every L4 and L5 failure collects logs, oracle, tap capture, DB dump and API snapshots, because without them failures are not reproducible.
4. L5 is small, selector-disciplined, uses `retries: 0`, validates every API response against the OpenAPI contract, runs axe accessibility checks, and does not gate on pixel diffs.
5. Tolerances are named, documented, protected constants.
6. The production compose files and the upgrade/rollback and backup/restore paths are tested in L4, not only the test stack.

## Alternatives rejected and why
- **Kubernetes (kind/k3d) for integration**: we do not run Kubernetes on the edge (file 04 alternatives); compose matches what is deployed.
- **Testcontainers per test**: good for isolated services, but the slow part is the full stack; the shared-stack plus fresh-stack split gives the same isolation where it matters.
- **Cypress or Selenium**: workable. Playwright chosen for multi-browser, built-in tracing, auto-waiting and its CDP access in one boring tool. Not a strong objection.
- **Visual regression as a gate**: brittle, trains agents to re-snapshot.
- **Retrying flaky browser tests**: hides real intermittent bugs (principle 7 in the README).
- **Mocking the API in HMI e2e**: mocks test our assumptions. HMI component tests with mocks belong in Vitest; L5 uses the real gateway.

## Open questions
1. `data-quality` and `data-testid` conventions in the HMI need to be adopted by 05-hmi-api component docs (a small additive requirement, not a contract change).
2. The healthcheck command, image names and versions in the compose excerpt are placeholders owned by 08-deployment and the build pipeline; marked UNVERIFIED where they matter.
3. Chromium throttling through CDP is Chromium-only; Firefox slow-link behavior is untested in v1.
4. Whether the HMI's chart exposes its data series to test hooks without shipping a debug surface in production (alternative: assert through axe-readable table fallback, which also helps accessibility).
5. Stack start time budgets (60 s) are guesses until measured on the CI runner.
