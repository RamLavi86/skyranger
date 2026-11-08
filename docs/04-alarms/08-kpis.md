# 08 Alarm KPIs

KPIs exist to answer one question: **where is the noise coming from, and is it getting better?** They are computed from the journal (07), published as `_system/alarms/kpi/...` tags so they trend, alarm and export like everything else, and shown on an HMI "alarm health" page.

Numeric targets quoted in the alarm-management literature (alarms per operator per hour, flood thresholds, percentage splits by priority, share of time in flood) are **UNVERIFIED** here. The system measures and reports; the integrator and customer set targets in the alarm philosophy document after checking ISA-18.2 / IEC 62682 / EEMUA 191 texts. Defaults below are configurable placeholders.

## 1. KPI catalog

Per console (`Gateway.spec.alarms.consoles`, 01) plus a `_all` roll-up. Windows are rolling unless stated.

| KPI | Definition | Source | Reading it |
|---|---|---|---|
| `activation_rate_10m` | activations (`activated`, `reactivated`) in the last 10 min | journal | Peak is the number that matters for operator load |
| `activations_per_day` | per calendar day (gateway timezone), also average over 30 d | journal | Trend |
| `flood_minutes_pct` | percent of time in the last 30 d the console was in flood (04 s5) | `flood_changed` events | Placeholder target set by philosophy |
| `flood_count` | number of flood episodes, with peak count and top contributors | events | Which alarms start floods |
| `active_count` | alarms in `active_*` and `cleared_unacked`, annunciated | `alarm_state` | Operator list size |
| `unacked_count`, `oldest_unacked_age` | `active_unacked` plus `cleared_unacked`, age of oldest | state | A growing oldest age is an operator not seeing the system |
| `standing_count` | alarms `active_acked` longer than `standing_after` (default `24h`) | state | Alarms that are always on become wallpaper |
| `stale_alarm_age_p50/p95` | active age distribution | state | |
| `shelved_count`, `shelved_total_time` | currently shelved; sum of shelved time in 30 d | state, journal | Shelving as a hidden config |
| `suppressed_count` | currently suppressed | state | |
| `held_count` | currently held (input not trusted) | state | Many held alarms means a data problem somewhere |
| `out_of_service_count`, `oldest_oos_age` | disabled alarms and age | state | Over `30d` warns (04 s8) |
| `chattering_count` | alarms currently chattering; episodes in 30 d | journal | Fix list |
| `time_to_ack_p50/p95` | ack time minus activation time (07 s3), by severity | journal | Operator responsiveness, escalation tuning |
| `ack_within_target_pct` | percent acked within `rationale.response_time` (where defined) | journal | Does the response time hold up in reality |
| `severity_distribution` | count of configured alarms by severity; activations by severity | config, journal | Top-heavy configurations |
| `top_offenders` | top N alarms by activation count in 7 d / 30 d, with percent of total | journal | The 80/20 list: fix these first |
| `repeat_ratio` | activations of an alarm within 30 min of its previous clear, over total | journal | Cheap proxy for needing deadband or delay |
| `notification_latency_p50/p95` | `sent_at - active_at` per channel and severity | outbox | Real-world delay to the phone |
| `notification_failure_rate` | `dead` plus `expired` over created, per channel | outbox | Channel quality |
| `unnotified_critical` | critical activations with no `sent` notification within 5 min | outbox, journal | Must be zero. Red flag KPI |
| `rationalization_coverage` | percent of `high`/`critical` alarms with `rationale` | config | Completeness |
| `engine_input_overflow_count` | coalesced samples because of mailbox overflow | engine | Must be zero (10) |

Chattering and repeat ratio include activations while shelved or suppressed, so hidden alarms stay visible in the numbers. `suppressed activations` are counted separately (`suppressed_activation_count`) so a root-cause suppression is credited rather than counted as noise.

## 2. SQL (PostgreSQL / TimescaleDB)

Peak 10-minute activation rate per console, last 7 days (sliding window via generate_series is simpler than a continuous aggregate; the engine also keeps a live in-memory counter for the current value):

```sql
WITH a AS (
  SELECT console, ts FROM alarm_journal
  WHERE kind IN ('activated','reactivated') AND ts >= now() - interval '7 days'
)
SELECT console, max(n) AS peak_10m
FROM (
  SELECT console, ts,
         count(*) OVER (PARTITION BY console ORDER BY ts
                        RANGE BETWEEN interval '10 minutes' PRECEDING AND CURRENT ROW) AS n
  FROM a
) x
GROUP BY console;
```

Top offenders, 30 days:

```sql
SELECT alarm_id, max(path) AS path, max(alarm_name) AS name, max(severity) AS severity,
       count(*) AS activations,
       round(100.0 * count(*) / sum(count(*)) OVER (), 1) AS pct_of_total,
       count(*) FILTER (WHERE shelved) AS while_shelved
FROM alarm_journal
WHERE kind IN ('activated','reactivated') AND ts >= now() - interval '30 days'
GROUP BY alarm_id
ORDER BY activations DESC
LIMIT 20;
```

Standing alarms (current):

```sql
SELECT path, name, severity, now() - acked_at AS standing_for
FROM alarm_state
WHERE state = 'active_acked' AND acked_at < now() - interval '24 hours' AND enabled
ORDER BY standing_for DESC;
```

Time to ack by severity, 30 days: the lateral join of 07 s3 grouped with `percentile_cont(0.5/0.95) WITHIN GROUP (ORDER BY time_to_ack)`.

Flood time percent: from `flood_changed` rows in `events` (`type = 'skyranger.alarm.flood_changed.v1'`), pair `flood: true` with the next `flood: false` per console and sum overlapped durations over the window; open episodes are closed at `now()`.

Heavy KPIs are computed by a **KPI aggregator** goroutine every 60 s (cheap live counters) and every 15 min (journal queries), never on the engine's path. Daily and monthly roll-ups are stored in `alarm_kpi_daily` (console, day, metrics jsonb) so reports do not scan years of journal.

## 3. `_system` KPI tags

Gateway-created, read-only, under `_system/alarms/kpi/{console}/...` (`{console}` is a console name from config, or `_all`; segment grammar). These extend the `_system` tag namespace owned by the gateway (tag-model s1: "may change between versions").

| Tag | Datatype | Unit | Update |
|---|---|---|---|
| `_system/alarms/kpi/{console}/active_count` | int64 | `1` | on change |
| `_system/alarms/kpi/{console}/unacked_count` | int64 | `1` | on change |
| `_system/alarms/kpi/{console}/oldest_unacked_age` | float64 | `s` | every 60 s |
| `_system/alarms/kpi/{console}/activation_rate_10m` | float64 | `1` | every 60 s |
| `_system/alarms/kpi/{console}/standing_count` | int64 | `1` | every 15 min |
| `_system/alarms/kpi/{console}/shelved_count` | int64 | `1` | on change |
| `_system/alarms/kpi/{console}/chattering_count` | int64 | `1` | on change |
| `_system/alarms/kpi/{console}/held_count` | int64 | `1` | on change |
| `_system/alarms/flood/{console}` | bool | | on change (01 s6) |
| `_system/alarms/notifier/{channel}/failing`, `queue_depth`, `oldest_pending_age` | bool / int64 / float64 `s` | | 06 |
| `_system/alarms/engine/input_overflow` | bool | | 10 |
| `_system/alarms/engine/queue_depth`, `eval_p99_us`, `commit_p99_ms` | int64 / float64 | `1`, `us`, `ms` (UCUM `us` for microsecond is UNVERIFIED) | every 10 s |

Because they are tags, a customer can alarm on them: for example `_system/alarms/kpi/water/oldest_unacked_age > 1800` with a `class: system` alarm "operator has not looked at the alarm list for 30 min", routed to the shift supervisor. History comes for free (the historian stores them with `on_change` plus `max_interval`).

## 4. Reports

The HMI alarm health page and a monthly PDF/CSV report for the customer:
1. Monthly activations, per day chart, per console, with peak 10 min.
2. Top 10 offenders with recommended actions (deadband, delay, rationalize, fix sensor).
3. Flood episodes with their top contributors.
4. Time-to-ack distribution by severity and escalation counts.
5. Shelved, suppressed, out-of-service lists with ages.
6. Notification delivery performance, failures, heartbeat gaps.
7. Config changes affecting alarms (journal `reconfigured` rows).

A standing monthly review of the top offenders is the process that makes the KPIs matter; the tool only supplies the list.

## Decisions made
1. KPIs are derived from the journal and state tables, never separately counted in a way that can disagree with the record.
2. Published as `_system/alarms/kpi/...` tags so they trend, alarm and export like any tag.
3. Heavy queries run in a separate aggregator goroutine on a 60 s / 15 min cadence; daily roll-ups are stored.
4. Hidden alarms (shelved, suppressed, held, chattering) stay visible in counts; suppressed activations are counted separately to credit root-cause suppression.
5. No numeric targets from standards are hard-coded; every threshold is config and marked UNVERIFIED.
6. `unnotified_critical` must be zero and is a first-class KPI.

## Alternatives rejected and why
- **A separate metrics system (Prometheus counters) as the source**: second source of truth that can disagree with the journal, plus another service. Tags and SQL are enough at this scale.
- **Computing KPIs in the engine loop**: adds latency to alarm evaluation for no benefit.
- **Hard-coding 'good' thresholds from standards**: quoting numbers we have not verified; and the right target is a plant-specific decision.
- **KPIs only on a dashboard**: can't alarm on them, can't trend them, can't export them.
- **Counting only annunciated alarms**: reports a calm plant while shelving hides the noise.

## Open questions
- Operator and console mapping: "alarm rate per operator" needs shift and operator assignment, which the HMI/security area does not define. Console is the proxy.
- `_system/alarms/kpi/...` names and `us`/`ms` units extend the tag namespace and UCUM usage (units UNVERIFIED); needs a line in tag-model's `_system` list.
- Is `standing_after` default of 24 h reasonable for water networks (many alarms are legitimately long: reservoir low for days)? Tune on first project.
- Flood definition counts activations; some guidance counts annunciated alarms per operator. Both can be computed; which one the philosophy uses is a customer decision (UNVERIFIED which the standard prefers).
- Event table retention may be shorter than the 30 d flood window; if so, flood episodes move to the journal as `kind` rows (cheap change).
