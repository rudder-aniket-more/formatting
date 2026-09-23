# Monitoring & Data-Quality Plan

Alerting and validation for the zorro CDC branch (Postgres → DMS → bronze → gold → Athena),
with notes where the Ideon/Tornado branch differs.

Status legend: **[none]** not implemented · **[partial]** exists but incomplete ·
**[planned]** designed here, not built.

**Item ID legend.** IDs are local to this document — they are shorthand so the coverage
matrix and build order can reference an item without restating it. They are not a repo or
industry convention.

| Prefix | Meaning |
|---|---|
| **A** | **Alert** — notifies a human when it fires (A1–A6) |
| **T** | **Test** — checks data correctness (T1, T3, T4) |

Three items carry a `T` for packaging reasons rather than because they are tests: **T2** is a
cost optimization, **T5** is an enabler that T2 depends on, and **T6** is observability. They
ship alongside the tests in the same files, which is why they are grouped here.

---

## 1. What exists today — verified against the code

Every item below was checked against the working tree before this plan was written.

| Capability | Status | Evidence |
|---|---|---|
| Any notification integration (SNS / Slack / email / PagerDuty) | **[none]** | No match in `src/` for `on_failure_callback`, `sns`, `send_email`, `slack`, `webhook`. The only hit is [dag_zorro_dms_reload.py:1272](../src/dags/dag_zorro_dms_reload.py#L1272): *"Deliberately no notification integration: the repo has no SNS/Slack wiring"* |
| Airflow failure callbacks | **[none]** | No `default_args` with `on_failure_callback` in any of the 7 DAGs |
| Freshness / lag monitoring DAG | **[none]** | `src/dags/` contains 7 DAGs; none monitors freshness |
| DMS landing-zone silence check | **[none]** | No match for a "files arriving?" check anywhere in `src/` |
| Alert on validation `DISCREPANCY` | **[none]** | [run():738-747](../src/sources/zorro/job_validation_zorro_silver_rds.py#L738) raises only on `failures` (exceptions). A `DISCREPANCY` writes a log row and exits **green** |
| Grace keyed to the gold watermark | **[none]** | [line 686](../src/sources/zorro/job_validation_zorro_silver_rds.py#L686): `cutoff = rds_now - grace_period_seconds` |
| Validation skips unchanged tables | **[none]** | [line 705](../src/sources/zorro/job_validation_zorro_silver_rds.py#L705): `tables = sorted(self.tables_pk_mapping)` — no watermark filter; every table is fully scanned every run |
| `dbt source freshness` | **[none]** | No `freshness:` or `loaded_at_field:` in `_gold__sources.yml` or `_tornado__sources.yml` |
| Run-over-run row-count guard (Tornado) | **[none]** | No such check in `src/pipelines/tornado/` |
| DMS CloudWatch alarms | **unknown** | Provisioned outside this repo — cannot be verified here. Listed as planned in `.agents/architecture.md` |

**What does exist:** 469 dbt schema tests (deploy-time), 356 pytest tests (CI-time), the daily
RDS reconciliation job (writes findings, alerts nobody), per-table failure isolation that fails
the Glue job, and the Athena pending-gate in `dag_zorro_etl`.

**Summary:** the repo can *detect* a great deal and *notify* nothing.

---

## 2. Failure modes and target coverage

| # | Failure | Athena symptom | Today | Covered by |
|---|---|---|---|---|
| 1 | Gold Glue job errors | Stale table | Task fails, no alert | A1 |
| 2 | Bronze Glue job errors | Stale table | Task fails, no alert | A1 |
| 3 | **DMS replication dies** | Stale tables, all DAGs green | **Nothing** | **A3** (+ A6 early warning) |
| 4 | Gold lagging behind bronze | Stale table | Retried, but unbounded | **A2** |
| 5 | Loaded but values wrong | Fresh, wrong | `DISCREPANCY`, green DAG | **A4** |
| 6 | dbt deploy fails | Views stale | GH Actions email to committer | A1 (partial) |
| 7 | Gold table empty / collapsed | 0 rows | Nothing | A5, T3 |
| 8 | Ideon CSV truncated | Tornado gold shrinks | Nothing | **T3** |
| 9 | Hot rows never validated | Silent blind spot | **Structural gap** | **T1** |

Modes 3, 8 and 9 are the ones with no signal at all today.

---

## 3. The cadence problem — the constraint that shapes everything

The ~120 replicated Postgres tables do **not** update at the same rate. Some change every
second; some have not changed since the initial `LOAD`. This is not incidental — it rules out
the obvious design.

### Why "time since last load" is the wrong metric

Bronze writes a log row only when it actually processes files. A table with no new files is
`skipped_no_files` ([job_bronze_zorro.py:406](../src/sources/zorro/job_bronze_zorro.py#L406)) —
no log row, watermark frozen.

So `now() - max(processed_at)` is **ambiguous**: it means either "no new data arrived" (healthy)
or "the pipeline is broken" (not healthy). And with a five-order-of-magnitude spread in cadence,
no single threshold works:

| Threshold | Effect |
|---|---|
| 2 hours | Every quiet table breaches permanently. A reference table untouched since the initial load is *always* months stale → ~60 alerts on day one → channel muted by day two |
| 24 hours | The busiest table can be dead for 23 hours in silence — exactly the outage you are trying to catch |

### The metric that does work: **lag, not age**

Ask *"has bronze delivered something gold has not absorbed, and for how long?"*

This is **cadence-independent**. A table that never changes never produces a bronze row, so it
can never appear in the result. The alert only fires when there is real, unabsorbed work.

> **Design rule:** measure the pipeline's lag, not the data's age. Age is a property of the
> business; lag is a property of the pipeline, and it means the same thing for a table updated
> every second and one updated every year.

The complement — "is anything arriving at all?" — is checked **once per replication**, not per
table. Across 120 tables of mixed cadence, *some* table changes within an hour; total silence
means DMS is down, not that a table is quiet.

---

## 4. The plan

### Phase 1 — Notification foundation (½ day)

#### A1. SNS topic + `on_failure_callback` **[planned]**

**Covers:** modes 1, 2, 6. **Effort:** ~1 hour.

Nothing can alert until there is a channel. Add one SNS topic per environment, then wire a
callback into every DAG's `default_args`.

```python
# src/dags/helpers/notify.py  (new)
import boto3, logging
from helpers.resolver import config

logger = logging.getLogger(__name__)

def _publish(subject: str, message: str) -> None:
    topic = config.get("alerts_topic_arn")
    if not topic:                      # dev: no topic, log only
        logger.warning("ALERT (no topic configured): %s\n%s", subject, message)
        return
    boto3.client("sns", region_name=config["region"]).publish(
        TopicArn=topic, Subject=subject[:100], Message=message
    )

def alert_on_failure(context) -> None:
    # ENVIRONMENT comes from Airflow config, not resolver.json — there is no
    # `environment` key there. Same source every DAG already uses.
    ti = context["task_instance"]
    _publish(
        f"[{ENVIRONMENT}] Airflow failure: {ti.dag_id}.{ti.task_id}",
        f"Run:  {context['run_id']}\nTry:  {ti.try_number}\nLog:  {ti.log_url}",
    )
```

```python
# in each @dag(...)
default_args={"on_failure_callback": alert_on_failure}
```

Add `alerts_topic_arn` to `resolver.json` — `null` in the `dev` section, a real ARN per AWS
environment. This matches the existing convention: dev degrades to a log line, AWS publishes.

**Requires:** `sns:Publish` on the MWAA execution role.

> ⚠️ `.agents/architecture.md` lists `sla_miss_callback` among planned monitoring. Verify before
> relying on it — SLAs were removed in Airflow 3 in favour of Deadline Alerts, and this project
> runs 3.2.2. A2 below gives the same coverage without that dependency.

---

### Phase 2 — The core: pipeline lag and source silence (2–3 days)

A new DAG, `dag_data_freshness`, scheduled every 30 minutes, **independent of the pipeline it
watches**. This is a dead man's switch: alerting wired only into pipeline tasks goes quiet
exactly when the pipeline dies.

#### A2. Pipeline-lag alert **[planned]**

**Covers:** modes 1, 2, 4, 7 (partially). **Effort:** ~1 day.

**What it does.** Finds `(source, table)` pairs where the bronze watermark is ahead of the gold
watermark and has been for longer than one threshold.

**Why it is cadence-independent.** The row only exists if bronze actually processed files. No
files → no row → no alert, no matter how long the table has been quiet.

```sql
WITH b AS (
  SELECT dms_prefix, src_schema, table_name, max(processed_at) AS ts
  FROM etl.zorro_etl_logs WHERE layer = 'bronze' GROUP BY 1,2,3),
g AS (
  SELECT dms_prefix, src_schema, table_name, max(processed_at) AS ts
  FROM etl.zorro_etl_logs WHERE layer = 'gold'   GROUP BY 1,2,3)
SELECT b.dms_prefix, b.src_schema, b.table_name,
       b.ts AS bronze_ts, g.ts AS gold_ts,
       date_diff('minute', b.ts, current_timestamp) AS minutes_behind
FROM b LEFT JOIN g USING (dms_prefix, src_schema, table_name)
WHERE (g.ts IS NULL OR b.ts > g.ts)
  AND date_diff('minute', b.ts, current_timestamp) > 60
ORDER BY minutes_behind DESC
```

This is the existing `CHECK_PENDING_QUERY` from
[dag_zorro_etl.py:43](../src/dags/dag_zorro_etl.py#L43) plus a duration filter — the gate query
and the alert query are the same question asked with different patience.

##### Worked example

State of `etl.zorro_etl_logs` at 14:30:

| table | cadence | bronze_ts | gold_ts | minutes_behind | Alert? |
|---|---|---|---|---|---|
| `employee` | every few min | 14:28 | 14:29 | — | No — gold caught up |
| `benefit` | hourly | 13:05 | 13:06 | — | No — caught up |
| `audit` | bursty | **12:40** | **12:05** | **110** | 🔴 **Yes** — 110 min behind |
| `employer` | daily | 06:15 | 06:16 | — | No |
| `carrier_vendor_mapping` | **never changed since LOAD** | 2026-05-01 | 2026-05-01 | — | **No** — equal watermarks, invisible to the query |
| `insured` | every few min | 14:27 | *(null)* | 63 | 🔴 **Yes** — gold has never run |

The static table from May is correctly silent. A naive "stale > 2h" check would have alerted on
it every 30 minutes for four months.

**Alert payload** — one digest per check, never one alert per table:

```
[prod] PIPELINE LAG: 2 table(s) behind > 60 min

  audit    110 min behind   (bronze 12:40 → gold 12:05)
  insured   63 min behind   (gold has never run)

Check: MWAA dag_zorro_etl recent runs, then the Glue job logs.
```

#### A3. Source-silence alert **[planned]**

**Covers:** mode 3 — the `dms_error.md` outage, which today produces **zero** signal.

**Why A2 cannot catch it.** If DMS dies, no files arrive → bronze processes nothing → no new
bronze rows → nothing is pending → gold is legitimately caught up → **A2 stays silent and every
DAG is green.** The pipeline is perfectly healthy and perfectly useless.

**What it does.** Checks the DMS landing zone directly, once per replication:

```python
@task
def check_source_silence():
    s3 = create_s3_client(config["s3_endpoint"])
    for source in all_sources().values():
        newest = newest_object_mtime(
            s3, config["dms_bucket"],
            prefix=f"{source['dms_prefix']}/{source['src_schema']}/",
        )
        quiet_for = datetime.now(timezone.utc) - newest
        if quiet_for > timedelta(hours=1):
            alert(
                f"[{env}] NO SOURCE DATA — {source['name']}",
                f"No DMS files in {quiet_for}. Newest object: {newest}.\n"
                f"Check the DMS replication status and CDCLatencySource.",
            )
```

**Why one hour is safe here.** This threshold is *not* per table — it is across all ~120 tables
of a replication. Heterogeneous cadence is what makes it robust: in a healthy system some table
always changes within the hour. A full hour of silence across every table means the source is
broken, not that traffic is quiet.

Tune from history if you want: the longest observed gap between *any* two DMS files over the
last 30 days, × 3.

##### Worked example

`dms_error.md` scenario — WAL slot invalidated at 12:12 UTC:

| Time | DMS | Pipeline | A2 | A3 |
|---|---|---|---|---|
| 12:12 | CDC dies | — | — | — |
| 12:20 | No files | Bronze: 0 files, success. Gate: nothing pending. Gold skipped. **All green** | Silent | Silent (< 1h) |
| 13:15 | No files | Still green | Silent | 🔴 **"NO SOURCE DATA — 63 min"** |

A3 is the *only* detector that fires. Detection in ~1 hour instead of whenever a human noticed.

---

### Phase 3 — Fix the validation job (2–3 days)

#### A4. Alert on `DISCREPANCY` **[planned]**

**Covers:** mode 5. **Effort:** ~2 hours.

Today a run that finds 50,000 missing rows writes `status = 'DISCREPANCY'` and exits green
([run():738](../src/sources/zorro/job_validation_zorro_silver_rds.py#L738) raises only on
exceptions). Nothing queries the log table.

```python
# at the end of run(), after the failures block
discrepant = [r for r in self._results if r.status == STATUS_DISCREPANCY]
if discrepant:
    alert(f"[{env}] GOLD/RDS DIVERGENCE: {len(discrepant)} table(s)", digest(discrepant))
```

**Notify, do not raise.** `FAILED` (the check itself broke — flaky JDBC, OOM) and `DISCREPANCY`
(the data is wrong) have different owners and different urgency. Conflating them means a
transient JDBC timeout pages someone about data corruption.

Example payload:

```
[prod] GOLD/RDS DIVERGENCE: 2 table(s)

  employee   missing=0  extra=1,204  mismatched=0
             → 1,204 deletes never reached bronze
             WHERE id IN ('e-8841','e-8842', …)

  benefit    missing=0  extra=0      mismatched=37
             mismatch_columns: amount:37
```

#### T1. Fix the grace-period blind spot **[planned]** — highest-value correctness fix

**Covers:** mode 9. **Effort:** ~1 day including tests.

**The defect.** `_apply_grace` excludes every PK whose `updated_at` is within
`grace_period_seconds` (600s) of RDS `now()`, from **both** sides. A row updated more often than
every 10 minutes is inside that window on *every* run, so it is **never validated — not rarely,
structurally never.**

Validation coverage is therefore **inversely proportional to update frequency**:

| Table cadence | Rows in the 600s window | Coverage |
|---|---|---|
| Every few minutes | Most or all, always | **~0%** |
| Hourly | Some | Partial |
| Weekly | Almost none | ~100% |
| Static | None | 100% |

The hottest tables — the most CDC events, the most chances to drop a delete, usually the most
business-critical — get the least checking. The fully-covered tables are the ones least likely
to be wrong.

##### Worked example

`employee` row `e-42`, updated by a background job every ~5 minutes. Its salary has been wrong
in gold since 2026-09-01 (a dropped CDC event).

| Run date | `updated_at` | RDS now | Cutoff (now − 600s) | In window? | Validated? |
|---|---|---|---|---|---|
| 09-02 | 03:56 | 04:00 | 03:50 | Yes | ❌ Skipped |
| 09-03 | 03:58 | 04:00 | 03:50 | Yes | ❌ Skipped |
| … every run … | always < 10 min old | | | Yes | ❌ Skipped |

The corruption is invisible forever.

##### The fix

Key the cutoff to the **gold watermark**, not the wall clock. A row that was already correct as
of the gold snapshot is comparable no matter how recently it changed; only changes *after* that
snapshot are genuinely un-propagated.

```python
# current
self._cutoff = rds_now - timedelta(seconds=self.grace_period_seconds)

# proposed, per table
gold_wm = get_layer_watermarks(..., layer="gold")[table_name]
cutoff  = gold_wm + timedelta(seconds=safety_margin)   # small, e.g. 60s
```

Re-run the example: on 09-02 the gold watermark is 03:45, so only rows changed after 03:45 are
excluded. `e-42` last changed at 03:56 — still excluded that run. But the *next* run, gold's
watermark has advanced past 03:56, and `e-42` becomes comparable. The corruption surfaces within
one cycle instead of never.

Same false-positive protection, no permanent blind spot.

#### T2. Skip unchanged tables **[planned]** — cost reduction

**Effort:** ~½ day. **Saving:** est. 60–80% of validation cost.

Today every table is fully scanned daily regardless of change
([line 705](../src/sources/zorro/job_validation_zorro_silver_rds.py#L705)). A static reference
table is re-read from RDS every day to prove it still matches.

```python
last_validated = read_last_validation_timestamps()   # from the log table
bronze_wm      = get_layer_watermarks(..., layer="bronze")
tables = [t for t in sorted(self.tables_pk_mapping)
          if force_full or bronze_wm.get(t, MIN) > last_validated.get(t, MIN)]
```

Keep a **weekly full pass** (`--force_full`) so nothing goes permanently unchecked. Combined with
moving `audit` (39 of 55 minutes) to weekly, the daily run should drop to roughly 10 minutes.

⚠️ T2 depends on the log table retaining history. It is currently
`CREATE OR REPLACE`d each run — see T5.

---

### Phase 4 — Coverage extensions (1–2 days)

#### A5. Gold row-count sanity **[planned]**
Alert when a gold table's row count drops more than *N%* versus the previous run, or hits zero.
Catches mode 7 (collapsed/empty table) which lag monitoring cannot see — a table can be perfectly
fresh and perfectly empty. Store counts in the freshness DAG's own small state table.

#### A6. DMS CloudWatch alarms **[planned, infra — outside this repo]**
Alarms on `CDCLatencySource` / `CDCLatencyTarget` going flat or to zero. This is the *root cause*
of mode 3, detectable before it propagates, and it is inherently cadence-independent. Earliest
possible signal; complements rather than replaces A3.

#### T3. Tornado run-over-run row-count guard **[planned]**
**Covers:** mode 8 — the biggest unmonitored gap on the Ideon side.

Ideon silver and Tornado gold are full `createOrReplace` rebuilds with **no** run-over-run
comparison anywhere. If Ideon ships a truncated `plans.csv`, gold shrinks by 90%, every gate
passes, and the bad catalog flows to the seed export and into quoting.

Add a gate after the four gold jobs, in the style of the existing `BlendValidationError` checks:

```python
prev = read_previous_counts()          # small gold.tornado_run_counts table
for table, count in current_counts.items():
    if prev.get(table) and count < prev[table] * 0.8:
        raise TornadoValidationError(
            f"{table}: {count} rows, down {1 - count/prev[table]:.0%} from {prev[table]}"
        )
```

Fail the DAG, consistent with how the blend gates behave — a bad plan catalog must not reach
`zorro-ts`.

#### T4. `dbt source freshness` **[planned]**
Add to `_gold__sources.yml` (gold jobs already stamp `_created_at`):

```yaml
sources:
  - name: gold
    loaded_at_field: _created_at
    freshness:
      warn_after:  {count: 2, period: hour}
      error_after: {count: 6, period: hour}
```

Checks freshness *at the layer consumers actually query*, with no new infrastructure.
**Complement, not a replacement for A2/A3** — it cannot distinguish "no source data" from
"pipeline broken", and per-source overrides are needed for legitimately quiet tables.

#### T5. Retain validation history **[planned]**
`etl.zorro_gold_validation_logs` is `CREATE OR REPLACE`d each run, so you cannot answer *"when
did this discrepancy start?"* or *"is it getting worse?"* Switch to append + prune (the
`airflow_run_id` column already exists). **T2 depends on this.**

#### T6. Tornado orphan-count logging **[planned]**
`create_plan_pricing_zip`'s docstring claims *"The dropped-row count is logged"* — it is not; only
the surviving count is. Orphan pricing is discarded with no visibility. One extra `.count()` and
a log line.

---

## 5. Priority and sequencing

| Order | Item | Covers | Effort | Why this order |
|---|---|---|---|---|
| 1 | **A1** SNS + failure callback | 1, 2, 6 | 1 h | Nothing can alert without a channel |
| 2 | **A3** Source silence | **3** | ½ day | The only detector for the outage that already happened |
| 3 | **A2** Pipeline lag | 1, 2, 4 | 1 day | Cause-agnostic; catches failures you have not imagined |
| 4 | **T1** Grace fix | **9** | 1 day | Restores validation on your most critical tables |
| 5 | **A4** DISCREPANCY alert | 5 | 2 h | Nearly free — the data is already computed |
| 6 | **T3** Tornado count guard | **8** | ½ day | Largest gap on the Ideon side |
| 7 | **T5** → **T2** | cost | 1 day | History first, then skip-unchanged |
| 8 | A5, A6, T4, T6 | 7, defence in depth | 1–2 days | Diminishing returns |

Items 1–3 are two days of work and close every failure mode that is currently **completely**
silent.

---

## 6. Design rules

1. **Measure lag, not age.** The only metric that means the same thing for a table updated every
   second and one updated every year.
2. **The monitor must not depend on what it monitors.** `dag_data_freshness` runs on its own
   schedule so a dead pipeline cannot suppress its own alarm.
3. **One digest per check, not one alert per table.** With ~120 tables, per-table alerting
   guarantees the channel gets muted.
4. **Separate "the check broke" from "the data is wrong."** Different owners, different urgency.
5. **Do not alert from inside `dag_zorro_etl`.** Under `@continuous` a transient failure would
   fire every few minutes.
6. **Thresholds derive from observed history, not intuition.** `etl.zorro_etl_logs` already holds
   each table's cadence:

   ```sql
   SELECT table_name,
          approx_percentile(gap_min, 0.95) AS p95_gap,
          max(gap_min) AS max_gap
   FROM (SELECT table_name,
                date_diff('minute',
                  lag(processed_at) OVER (PARTITION BY table_name ORDER BY processed_at),
                  processed_at) AS gap_min
         FROM etl.zorro_etl_logs WHERE layer = 'bronze')
   GROUP BY table_name
   ```

   Only needed if the cadence-independent alerts prove insufficient — they usually do not.
7. **Dev degrades gracefully.** No topic configured → log a warning. Never make local runs
   depend on AWS.

---

## 7. Rollout

1. Ship A1–A3 in **test** first. Run one week; count alerts. Zero alerts in a week means the
   thresholds are too loose — verify by deliberately pausing `dag_zorro_etl`.
2. Route to a **non-paging** channel initially. Promote to paging only after a week with no
   false positives.
3. Tune A2's 60-minute threshold against observed lag. If `audit` legitimately runs 90 minutes
   behind after a burst, raise the global threshold rather than adding a per-table exception.
4. Add T1 with a regression test: a fixture row updated inside the old grace window must now be
   compared.
