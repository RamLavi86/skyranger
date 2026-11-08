# 08 Hardware-in-the-Loop, Power-Cut and Chaos Testing (L7)

Everything above L7 runs on virtual or containerized stand-ins. Real SCADA failures are physical: the supply drops mid-write, a switch reboots, an ESP32 browns out, an SSD lies about a flush. This file defines the **HIL rig**: real edge box, real ESP32, ground-truth pulse generator, relay-controlled power and network. It is the source of the most persuasive evidence in the pack ("we cut power 1,000 times and lost no configuration and no counted liters beyond what the contract bounds").

## 1. Rig design

```
                   rig controller (small Linux SBC or mini-PC, Python, runs test/hil)
                   - CI runner agent (label lab-hil), results upload, rig safety supervisor
     ┌───────────────┬─────────────────┬──────────────────┬───────────────────┬──────────────┐
     │ relay board A │ relay board B   │ pulse generator  │ managed switch    │ impairment   │
     │ DUT-box DC    │ ESP32 DC supply │ (own MCU, own    │ (port on/off via  │ box (2 NICs, │
     │ supply        │ + Wi-Fi AP      │  crystal)        │  SNMP/CLI)        │ netem bridge)│
     ▼               ▼                 ▼                  ▼                   ▼              │
 DUT edge box    ESP32 DUT ◄──────── pulse lines ───────  LAN ◄──────────────────────────────┘
 (reference      (real firmware,     + independent counter
  hardware)      real PCNT/ISR)      cross-check
        ▲                                                  
        └─ serial console + hardware power meter (log current draw) + USB-serial/IPMI (if present) to capture boot logs
```

| Element | Choice | Notes |
|---|---|---|
| DUT edge box | Each **reference hardware SKU** (08-deployment/03, not chosen yet), with the exact SSD model that ships | Evidence is valid per SKU+SSD. A different SSD means re-run. The rig holds at least one P2 appliance image box and one P1 generic box |
| Pulse generator | A separate microcontroller board (a Raspberry Pi Pico class RP2040 with PIO is a plausible choice, **UNVERIFIED** suitability) generating pulses from a crystal-timed schedule, with a **hardware counter on its own output via a second pin looped back** as an independent count | Ground truth is two independent counts that must agree (generator firmware count and loopback counter) before the run is accepted as a valid oracle |
| ESP32 DUT | The real firmware build under test, powered from a switchable DC supply, pulse input wired to the generator | One or two units; spare units to separate "unit defect" from "design defect" |
| Power switching | **DC-side relays or solid-state relays** on the DUT's wide-range DC input (hardware doc recommends wide-range DC), controlled by GPIO relay boards. Cuts are timed by the controller to millisecond-ish accuracy; **no mains switching by us** | Mains-voltage cuts, if ever required, are done through a purpose-made, certified mains switch by a qualified person. Relay boards need a rated-current margin and flyback protection (rig build notes) |
| Power profile | Optional programmable bench supply for **slow ramp-down / brownout** profiles (supply collapses over 50 to 500 ms, as real PSUs and UPSs do) in addition to instant relay cuts | Instant cuts and slow decay stress different failure windows. UNVERIFIED which one is worse for a given SSD; both are run |
| Network | Managed switch (port on/off), impairment box (Linux, two NICs, bridged, `tc netem`), Wi-Fi AP on its own switched supply | Cable pull equals port disable plus optional relay on a patch panel |
| Observability | Serial console capture of the DUT box where available, hardware power meter or shunt logging current, controller logs everything with one monotonic clock | Boot time, power draw during recovery, and kernel messages are part of the evidence |
| Safety supervisor | Controller refuses to run unattended unless: smoke/over-temperature input is healthy (simple thermal cutoff), relay boards respond to a heartbeat, and a hard-wired master switch exists | A rig that can set fire to an office on a stuck relay is not "evidence" |

### 1.1 Ground-truth chain for the pulse meter

1. The generator emits a known number of pulses, logs `(count, schedule)` and confirms with its loopback counter.
2. The ESP32 counts in hardware (PCNT/ISR with debounce, messages.md section 2) and publishes `pulse_count`/`boot_count`.
3. The MQTT tap (file 05) records what the broker forwarded.
4. The oracle computes expected `total_volume` from (1) and (3) using the contract loss rule (file 03 section 5). The gateway's `total_volume` must equal it **exactly**.

PM-08 (file 02): ESP32 pulse accuracy against the generator.

| Test | Profile | Pass |
|---|---|---|
| PM-08a | 1,000,000 pulses at 0.1 to 50 Hz in steps, clean pulse edges | `pulse_count` delta equals generator count exactly |
| PM-08b | Contact-bounce profile: each pulse has 0 to 5 ms of bounce, pulse width 20 ms min (a typical reed switch; the real meter's electrical spec is **UNVERIFIED** and the profile must be set from the actual meter datasheet) | counted equals generator count of logical pulses, no double counts |
| PM-08c | Pulses during Wi-Fi outage (30 min), then reconnect | no pulses lost: count continues, cumulative delivery after reconnect |
| PM-08d | Pulses during gateway outage and broker outage | same |
| PM-08e | Pulse burst above the design maximum (rate where the firmware documents it saturates) | behavior is *documented*: either counted or flagged; no silent miscount. The firmware doc owns the maximum rate; **UNVERIFIED** until it is written |

## 2. Power-cut programme

### 2.1 What can break, and what we assert after each cut

| Failure mode | Assertion |
|---|---|
| Database corruption or unrecoverable WAL | PostgreSQL starts and accepts connections without manual repair; `pg_amcheck`-style integrity check passes (UNVERIFIED availability for our PostgreSQL version; fallback: query every hypertable chunk with `COUNT(*)` and run `amcheck` functions where present) |
| Half-applied config | `config_revisions` shows N or N+1, never partial; running tags match the revision; export equals the revision |
| Lost counted volume | Gateway total equals ground truth exactly when the device kept counting through the gateway outage (cumulative counter + checkpointed `last_raw`, F-CNT-05). When the ESP32 itself was cut, the loss is within the bound of the contract (pulses between the last delivered sample and the reboot) |
| Lost history | History RPO measured: rows with `ts` in the final seconds before the cut that are missing. Reported as a distribution; must be within the historian's documented flush bound (historian doc owns; UNVERIFIED value) |
| Alarm state | Active alarm before the cut is active (unacked state preserved as `active_unacked` or acked state preserved) after recovery, or re-derived correctly from the data; no alarm silently disappears; operator acks that were confirmed to the user (HTTP 200) before the cut survive |
| Writes | A write the API acknowledged as `ok` through `write_completed` before the cut is not lost; a `pending` write is either applied once or reported failed, never applied twice |
| Boot to healthy | Box boots without human input; all containers healthy; API ready within the budget below |
| Time | After boot with no RTC battery or a stale RTC, the gateway does not write plausible-looking history with wrong time (F-CLK-06); chrony steps and the system recovers |
| Credentials, certificates | Still valid and unchanged after the cut (secret store not corrupted) |
| Filesystem | Read-only root image unchanged; data partition mounts clean; `fsck` needed rate recorded (any manual fsck requirement is a failure) |
| A/B update | Cut at each stage of an update (download, write inactive slot, switch, first boot, commit) leaves a bootable system on a known slot (secure-update doc) |

### 2.2 Cut modes

| Mode | Description |
|---|---|
| Instant cut | Relay opens; supply collapses at PSU's natural rate |
| Slow ramp | Programmable supply ramps down over 50 to 500 ms (and a variant that dips and recovers: brownout) |
| Reboot loop | Cut for 1 to 10 s, power on for 5 to 60 s, repeated (F-DEV-05 on the box) |
| Phase-targeted | Cut at a chosen moment of the box's activity (below) |
| Graceful signal | Power-fail input asserted (UPS signal), box shuts down cleanly, then power removed; verifies the *graceful* path is clean and fast |
| Dirty then graceful | Cut during the graceful shutdown |

**Phase-targeted cuts**, so the dangerous windows are hit deliberately, not by luck:

| Phase | How the controller detects it | Cut at |
|---|---|---|
| Heavy history write | Controller watches API/`_system` write-queue depth rising during a burst; or runs during F-VAL-06 burst | peak of batch flush |
| TimescaleDB chunk compression/retention job | Soak lifecycle profile (file 07) makes jobs frequent; poll PostgreSQL job status through the judge account | job running |
| Config apply | Controller issues `apply` via API and cuts after a randomized delay from 0 to 2 s | during apply |
| Checkpoint of `derived_state` | Cut at randomized offsets across a 1 s grid (checkpoint period, tag-model.md section 7) | uniformly random offset |
| Container start (during DB recovery after the previous cut) | Cut again as soon as the DB container starts | second cut during recovery |
| ESP32: boot, NVS write | Test-firmware build raises a GPIO pin around the `boot_count` NVS commit; controller cuts on that edge | exactly in the write |

Random cuts sample the space uniformly. Targeted cuts guarantee the worst windows are covered. Both are reported separately.

### 2.3 Programme and counts

| Level | Cuts per SKU+SSD | When |
|---|---|---|
| Nightly subset | 30 (random + phase-targeted, mixed modes) | nightly on `lab-hil` |
| Weekly | 200 | weekly |
| **Release** | **1,000** (random, targeted and reboot-loop in fixed proportions) over about 24 h | each release candidate, per supported SKU+SSD |
| Pre-first-customer | 5,000 | once per SKU+SSD |

How to state results honestly: zero failures in *n* independent cuts supports the statement "failure probability per cut is below about 3/n at 95% confidence" (the "rule of three"). 1,000 clean cuts means "below about 0.3% per cut", **not** "never". The evidence pack states n and the bound, not "bulletproof". A single failure in a release run is a release blocker until root-caused; it is never re-run until green.

### 2.4 Cut loop control (controller sketch)

```python
# test/hil/power_cut_loop.py  (runs on the rig controller; Python as in the repo layout)
import random, time, json
from rig import Rig, Phase           # rig = relay boards, switch, generator, tap, API client

def run(rig: Rig, n: int, seed: int, out: str) -> None:
    rnd = random.Random(seed)                      # seed logged; schedule reproducible
    rig.generator.start(profile="steady-12lpm")
    results = []
    for i in range(n):
        mode = rnd.choices(["instant", "ramp", "loop", "phase"], weights=[40, 20, 15, 25])[0]
        rig.wait_healthy(timeout=rig.limits.boot_to_healthy_s)          # also records boot time
        pre = rig.snapshot()                       # config revision, alarms, writes acked, totals, last history ts
        phase = rig.cut(mode, rnd)                 # returns what was targeted, timestamps of cut and restore
        rig.power_on(after=rnd.uniform(1, 30))
        t_boot = rig.wait_healthy(timeout=rig.limits.boot_to_healthy_s)
        post = rig.snapshot()
        verdict = rig.check(pre, post, phase)      # section 2.1 assertions, one result per assertion
        results.append({"i": i, "mode": mode, "phase": phase, "boot_s": t_boot, "verdict": verdict})
        if not verdict.ok:
            rig.preserve_artifacts(i)              # full disk image of the data partition if possible, logs, serial capture
            if rig.stop_on_first_failure: break
    json.dump({"seed": seed, "results": results, "generator": rig.generator.final_counts()}, open(out, "w"))
```

On the first failure the rig **stops and preserves the failed state** (data-partition image, serial log, tap capture). Continuing would overwrite the evidence.

### 2.5 Boot-to-healthy budget

Measured on every cut: time from power applied until the API reports healthy and all configured connections are `connected`. Budget (proposed, UNVERIFIED until measured): p99 <= 120 s on the P2 appliance, <= 180 s on a generic P1 box. Reported as a histogram in the evidence pack. A rising trend between releases is a regression.

### 2.6 Storage variants

The same programme runs against: the shipping industrial SSD with power-loss protection; a **consumer SSD without PLP** as a negative control (we expect more history loss; if it shows corruption, the documentation of "recommended hardware" is justified with data, and if it shows none, we say so honestly); and never an SD card (P3 demo kit, hardware doc: SD-card PostgreSQL unsupported). The negative control is what makes the PLP recommendation evidence instead of folklore.

## 3. ESP32 firmware and device-level cases

| ID | Case | Pass |
|---|---|---|
| HIL-ESP-01 | Power cut during normal counting, 500 cycles | each reboot: `boot_count` +1 exactly, `pulse_count` restarts from 0, gateway records one reset per epoch seen; conservation vs oracle with the contract loss bound |
| HIL-ESP-02 | Power cut **inside the `boot_count` NVS commit** (GPIO-flagged test build) | after boot, `boot_count` is N or N+1, never corrupt, never lower than N. **See Open question 1** for the case where it stays N |
| HIL-ESP-03 | Wi-Fi AP off 5 min, 1 h, 24 h while counting | pulses counted throughout; on reconnect new `bdSeq`, same `boot_count`, no reset event (F-DEV-02); `pulse_count` continues; reconnect backoff does not hammer the AP |
| HIL-ESP-04 | Broker unreachable but AP up | same as above; device does not reboot-loop (watchdog policy: firmware doc) |
| HIL-ESP-05 | SNTP unreachable at boot | timestamps reported are rejected by skew rule; samples use `ts_gateway`; history not polluted (F-CLK-01) |
| HIL-ESP-06 | Brownout (supply dips to threshold for 100 ms) | brownout detector reboots cleanly or device rides through; never a hung state with link up and no data (heartbeat guarantees detection, F-DEV-03/04) |
| HIL-ESP-07 | OTA update interrupted by power cut | device boots old or new image, never bricked (OTA design is firmware doc scope; the test exists so the design must answer) |
| HIL-ESP-08 | Wi-Fi signal degradation (RF attenuator or distance; optional) | reconnects; `rssi_dbm` reported; no counter damage |
| HIL-ESP-09 | 7-day run counting continuously at a mixed profile | zero count error vs generator (two independent counters agree); heap and uptime counters sane; no unexpected reboot (`boot_count` constant) |
| HIL-ESP-10 | Firmware profile conformance PM-01..PM-07 against the real device | all pass (file 02 section 4) |

## 4. Network chaos on real hardware

| ID | Chaos | Method | Expected (catalogue ref) |
|---|---|---|---|
| HIL-NET-01 | Switch port to the DUT box down 10 s, 5 min, 1 h | managed switch | F-NET-01/02; all tags `bad_not_connected`; recovery bound; conservation |
| HIL-NET-02 | 5% then 20% packet loss, 200 ms latency with jitter, on the broker path | impairment box (`tc netem`) | F-NET-04/05; no state flapping; counters exact |
| HIL-NET-03 | True half-open: impairment box silently drops all packets in both directions | netem `loss 100%` | F-NET-02 detection within 60 s (DC-CONN-04) |
| HIL-NET-04 | Duplicate and reorder packets | netem duplicate/reorder | F-VAL-07/08 |
| HIL-NET-05 | Flapping link 2 s down / 5 s up for 30 min | switch port scheduler | F-NET-06; no memory growth on the real box (RSS slope as in file 07) |
| HIL-NET-06 | DHCP lease change or IP change of the broker | rig DHCP server | reconnect via DNS/config, no crash |
| HIL-NET-07 | NTP source loss and recovery; clock step forward 1 h and backward 10 min | chrony manipulation, `date -s` | F-CLK-03/04 on real hardware (note F-CLK-04 contract issue) |
| HIL-NET-08 | Second NIC, plant LAN vs enterprise LAN: the gateway must not bridge them | traffic probes on both sides | segmentation claims in 07-security/05 hold |

## 5. Other hardware-dependent cases

| ID | Case | Pass |
|---|---|---|
| HIL-HW-01 | **Hardware watchdog**: freeze the supervising agent and, separately, make the kernel hang (`echo c > /proc/sysrq-trigger` on a rig box) | box reboots on its own within the documented watchdog window and returns healthy; skipped with explicit result `NO_WATCHDOG` on hardware without one (the hardware doc says many cheap boxes lack a usable one, so that result is expected evidence of why not to ship them) |
| HIL-HW-02 | Disk full on the real data volume (F-DB-05) | as catalogue; gateway keeps live values; alarm raised; recovers when space is freed |
| HIL-HW-03 | Thermal: run the release soak at the ambient the datasheet claims | CPU frequency throttling events recorded; latency S-9 holds. **No temperature-range claim is made unless a thermal chamber run was done** (not in v1; the evidence pack says "tested at lab ambient 20 to 25 C" in that case) |
| HIL-HW-04 | Clean graceful shutdown on power-fail signal from the UPS | shuts down in the documented time; next boot has no recovery |
| HIL-HW-05 | USB/serial removal, NIC unplug with hot-plug | no crash |
| HIL-HW-06 | Factory restore from the recovery partition | box returns to factory image; customer data policy per deployment doc |
| HIL-HW-07 | SSD SMART read after 1,000 cuts | unsafe-shutdown count and wear recorded in the report (value is informational) |

## 6. Operating the rig

- Results go to the CI system as artifacts (JSON, graphs, serial logs, tap captures) from the `lab-hil` runner. The runner only accepts jobs from protected branches and tags (self-hosted runners must never run fork PRs; CI file 10).
- The rig has its own **calibration check** before each release run: relay timing measured with a logic analyzer or the controller's loopback input (cut-to-zero-volts latency recorded), generator loopback counts agree, tap `records_dropped` zero. A run with a failed calibration is void.
- Rig faults are handled as test bugs: a relay board that sticks is replaced, and runs since the last good calibration are flagged.
- Safety rules for anyone working on the rig are in the rig README (to be written with the build).

## Decisions made
1. The rig switches **DC** supplies with relays under controller control, uses an independent two-counter pulse generator as ground truth, and records every run with one monotonic clock.
2. Power cuts are a programme: random, phase-targeted (history flush, compression job, config apply, checkpoint, recovery, NVS write), slow-ramp and reboot-loop, per reference SKU+SSD, with 1,000 cuts per release.
3. Results are stated with their statistics (rule of three), not as absolutes. A single failure blocks a release until root-caused and is never re-run until green.
4. The rig stops on the first failure and preserves the failed disk and logs.
5. A consumer SSD without PLP runs as a negative control so hardware recommendations in the deployment doc are backed by data.
6. No environmental claims (temperature range) without a thermal chamber run.

## Alternatives rejected and why
- **Simulating power loss with `kill -9` or `docker kill`**: tests process death only. It does not exercise SSD cache flush, filesystem journaling or PostgreSQL's real recovery. Useful at L4 (F-GW-04); not a substitute.
- **Mains switching in the rig**: hazard and certification burden for no added test value over DC cuts at the box's DC input. The AC-fed PSU's decay behavior is covered when the PSU is part of the DUT.
- **Cloud-hosted hardware test services**: cannot cut power or hold a specific SSD model.
- **One universal reference box**: customers will have different hardware; evidence is per SKU+SSD and the pack says so.
- **Counting pulses only with the ESP32 itself as the oracle**: not independent.
- **Manual cut tests before releases**: not repeatable, not enough n.

## Open questions
1. **Contract issue (boot_count commit).** messages.md section 2 increments `boot_count` in NVS "before connecting" and `counter_total` detects a reboot by `epoch != last_epoch`. If power is cut so that the NVS commit did not persist, the next boot publishes the **same** `boot_count` with `pulse_count` restarting from 0. With `modulus = 2^32` (the water meter template), the gateway reads `raw < last_raw` and the epoch unchanged as a **rollover**: `delta = (modulus - last_raw) + raw`, about 4.29 billion, which `max_delta` turns into `implausible` and delta 0 (the sample is ignored and counts since the reboot are lost until the algorithm re-baselines; see also F-CNT-03 open question). With `max_delta` unset, the total jumps by about 4.29 billion liters. Proposal: firmware writes `boot_count` with verify-after-write and refuses to publish if the commit failed; and/or the contract states `max_delta` is mandatory when `modulus > 0` and the template sets a safe default. HIL-ESP-02 exists to find this.
2. Reference hardware, SSD models and the UPS (with or without a power-fail signal) are not chosen; the rig counts and budgets are placeholders until then.
3. Whether the Raspberry Pi Pico class device is acceptable as the generator (jitter of PIO-generated pulses, loopback counting) is UNVERIFIED and should be validated with a logic analyzer before the rig is trusted.
4. The historian's flush bound (history RPO) and WAL durability settings are not yet documented; assertions on history loss reference them symbolically.
5. Maximum pulse rate and debounce parameters of the firmware are firmware-doc decisions and PM-08b/e must be re-derived from the real meter datasheet.
6. Rig safety documentation, electrical review and insurance implications are out of scope here and need a qualified person.
