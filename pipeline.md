# Onboarding — understanding this repository

Three parts: a short overview, a detailed overview, and a staged learning path with
realistic time estimates.

**The single most useful thing to know before you start:** this repo contains **two
independent pipelines** that share a codebase but no data. Almost all early confusion comes
from reading a file without knowing which of the two it belongs to. Check that first, every
time.

---

# Part 1 — The short overview

*Read this in two minutes.*

It is a **data warehouse**. It copies data from two places into cloud storage, cleans it up,
and makes it queryable for reports and for the app.

### The two pipelines

| | **Zorro CDC** | **Ideon → Tornado** |
|---|---|---|
| Where the data comes from | Your own app's database (Aurora PostgreSQL) | An outside supplier, Ideon |
| What arrives | Every insert, update and delete, continuously | CSV files of health plans and prices |
| How often | Non-stop, all day | Once a day at 6am UTC |
| What it produces | Tables for reports (enrolment, payments, compliance) | The plan catalogue and prices employees see when shopping |
| Who reads the result | Analysts, via Metabase | The Zorro app itself |

They share the same code repo, the same Spark setup and the same Airflow instance — but
**no tables**. A change to one cannot affect the other's data.

### How data moves (both pipelines)

```
raw data  →  bronze  →  silver  →  gold  →  consumers
             (as-is)   (cleaned)  (final)
```

This is the "medallion" pattern. Each layer is a set of tables; each job moves data from one
layer to the next.

### The technology, in one line each

- **Airflow** — the scheduler. Decides what runs and when.
- **AWS Glue** — runs the actual data processing (Spark/Python).
- **S3 + Apache Iceberg** — where tables are stored.
- **Athena** — lets you query those tables with SQL.
- **dbt** — builds reporting views on top of the finished tables.

### Repo size

| | |
|---|---|
| Python in `src/` | ~10,900 lines |
| Airflow pipelines (DAGs) | 7 |
| dbt models / macros | 125 / 19 |
| Tests | 358 |
| Documentation | ~5,700 lines |

**It is well documented.** Most files carry a docstring explaining not just what they do but
why they were written that way. Read those before reading the code.

---

# Part 2 — The detailed overview

*Read this in about twenty minutes.*

## 2.1 Pipeline one — Zorro CDC

**Question it answers:** *what is happening in our business right now?*

```
Aurora PostgreSQL (the live app database)
      │  AWS DMS streams every change as it happens
      ▼
S3 landing zone      Parquet files, each row tagged I / U / D
      │  job_bronze_zorro.py
      ▼
bronze.<table>       every change ever received, nothing removed
      │  job_gold_zorro.py
      ▼
gold.<table>         one row per record: its current state
      │  dbt
      ▼
gold.stg_ / int_ / mrt_    views that reports read
```

Three things about it surprise nearly everyone:

**Silver is switched off.** The medallion pattern says bronze → silver → gold, and the file
`job_silver_zorro.py` exists and is complete — but nothing runs it. Gold reads bronze
directly. The file is kept so the layer can be restored later.

**Bronze never deletes anything.** It is an event log. If a row was deleted in the app, bronze
records a delete *event* and keeps the earlier versions. Gold is where deletes are applied.

**The pipeline runs continuously.** Not hourly, not daily — it restarts the moment it
finishes. So the gold tables are usually seconds to minutes behind the app.

### The other pieces of this pipeline

- **`dag_generate_pk_mapping`** — every hour, asks the app database which column uniquely
  identifies each table, and saves it to S3. Bronze, gold and validation all read that file.
  Without it, gold cannot work out which row is "current".
- **`dag_validation_silver_rds`** — every night, compares the finished tables against the live
  database, row by row, and records anything that does not match.
- **`dag_zorro_dms_reload`** — a manual recovery procedure for when the data feed breaks. Only
  a person starts this.

## 2.2 Pipeline two — Ideon → Tornado

**Question it answers:** *what health plans exist, and what do they cost?*

```
Ideon's S3 bucket      CSV files, one folder per state and year
      │  job_bronze_ideon_s3_to_iceberg.py
      ▼
bronze.cdc_*           9 tables, loaded as-is
      │  three cleaning jobs run in parallel
      ▼
silver.dim_plans / plan_benefit_design / plan_pricing_detailed
      │  a pivot job, then four gold jobs
      ▼
gold.medical_plans, plan_pricing_zip, state_benefits_params, plan_benefits_standardized
      │  a person runs an export by hand
      ▼
CSV files  →  loaded into the quoting database the app uses
```

Two things to understand here:

**Everything is rebuilt from scratch, every day.** There is no "process only what changed".
Each run replaces the tables entirely.

**The hardest code in the repo lives here.** `job_silver_ideon_normalize_benefit_design.py`
(823 lines) reads free-text descriptions like
`"In-Network: $30 after deductible / Out-of-Network: 40%"` and works out what a member
actually pays. Leave it until last.

There is also a **blending** job used during the months when insurers have not all published
next year's prices: it fills the gaps with this year's prices adjusted by an agreed
percentage, so quotes can still cover every carrier.

## 2.3 How a job actually runs

The same code runs in three environments, and the difference is handled in exactly one place.

| | On a laptop (`dev`) | On AWS (`test` / `prod`) |
|---|---|---|
| Scheduler | Airflow in Docker | MWAA (managed Airflow) |
| Processing | A local Glue container | AWS Glue |
| Storage | MinIO (S3 look-alike) | Real S3 |

`helpers/task_factory.py` decides which of those to use. `helpers/resolver.py` plus
`resolver.json` supply every bucket name and endpoint.

> **The rule the whole repo follows:** jobs never know which environment they are in.
> Everything external is passed to them as an argument. Only the scheduling layer knows.

Knowing this rule explains a lot of otherwise puzzling code.

## 2.4 The dbt layer

Separate from the pipelines above, and **not run by Airflow**. It deploys from GitHub Actions.

It reads the finished `gold.*` tables and builds three tiers of views:

| Tier | Count | Purpose |
|---|---|---|
| `stg_` | 33 | Thin wrappers over a source table — rename, cast, nothing clever |
| `int_` | 69 | The business logic — joins, calculations, reusable pieces |
| `mrt_` | 19 | The finished reports analysts actually open |

All are **views**, so they cost nothing to store and always reflect current data.

## 2.5 Where things are

```
src/dags/         the schedules — what runs, and when
src/dags/helpers/ shared plumbing (config, task creation)
src/sources/      getting data IN     (zorro/ and ideon/)
src/pipelines/    turning it into finished tables  (tornado/)
src/utils/        small shared helpers
dbt/              the reporting views
tests/            mirrors src/
docs/             architecture and design decisions
.agents/          deeper architecture reference
```

A useful habit: the filename tells you the layer.
`job_<layer>_<source>_<what>.py` → `job_silver_ideon_normalize_plans.py` is the *silver* stage
of the *ideon* pipeline.

---

# Part 3 — The learning path

**Follow the data, not the folder listing.** Reading files alphabetically is the single
biggest cause of confusion here.

## Stage 0 — Orientation · half a day

**Goal:** know what exists and be able to find things.

1. `AGENTS.md` — the documentation index (10 min)
2. `docs/pipelines-overview.md` — the best single description of both pipelines (30 min)
3. `docs/src_file_reference.csv` — skim the `how_it_is_used` column for all 51 rows (20 min)
4. Draw the two pipelines on paper from memory. Check against the doc. Repeat until right.

✅ **You can:** say what the two pipelines do and roughly name their stages.

## Stage 1 — Make it run · half a day

**Goal:** see it working before reading more code.

```bash
docker compose up          # Airflow 8080, MinIO 9001, Jupyter 8888
```

Open Airflow, trigger `dag_ideon_tornado`, watch the tasks go green. Then open MinIO and look
at the files that appeared.

Follow `.claude/commands/ideon-pipeline.md` — it is the runbook for exactly this.

✅ **You can:** run the Ideon pipeline locally and see its output.

## Stage 2 — Follow one record end to end · 2 days

**Goal:** understand the Zorro pipeline properly. Do this one *slowly*.

Read in this order — it is the order the data moves:

1. `src/dags/dag_zorro_etl.py` (173 lines) — the schedule
2. `src/sources/zorro/job_bronze_zorro.py` — read the docstring, then `process_table`
3. `src/sources/zorro/job_gold_zorro.py` (293) — read `process_table` closely; the
   `row_number` + delete-filter is the heart of the whole pipeline
4. `src/sources/zorro/common.py` — just the watermark functions

Then answer these without looking:

- Why does bronze never delete anything?
- How does gold decide which row is current?
- What is a watermark, and what happens if a job fails halfway?

✅ **You can:** explain how a change in the app database reaches a report.

## Stage 3 — The Ideon pipeline · 2 days

Same approach, different shape — this one is about *transformation*, not capture.

1. `src/dags/dag_ideon_tornado.py` — note the fan-out and fan-in
2. `src/sources/ideon/job_bronze_ideon_s3_to_iceberg.py`
3. `src/sources/ideon/job_silver_ideon_normalize_plans.py` (254) — the gentlest cleaning job
4. `src/pipelines/tornado/job_gold_tornado_medical_plans.py` (72) — the smallest gold job
5. `src/sources/ideon/README.md` and the `*_SPEC.md` files

**Skip for now:** `job_silver_ideon_normalize_benefit_design.py` and
`job_gold_tornado_blended_plan_data.py`.

✅ **You can:** explain how a supplier CSV becomes a price an employee sees.

## Stage 4 — The dbt layer · 2 days

1. `docs/data-analytics-architecture.md` — why the layers are split as they are
2. `dbt/models/staging/gold/stg_employee.sql` — see how thin a staging model is
3. Pick one `mrt_` report and trace it back through its `int_` models to `stg_`
4. Skim `docs/dbt-best-practices.md` — use it as reference, do not read it end to end

✅ **You can:** add a column to a report and know which models to change.

## Stage 5 — How it is deployed and operated · 1 day

1. `.github/workflows/test.yml` — what CI checks
2. `.github/workflows/build-sync-s3.yml` — how pipeline code reaches AWS
3. `.github/workflows/dbt-deploy.yml` — how dbt deploys, and the PR schema trick
4. `.agents/TESTING.md` — how to write and run tests

✅ **You can:** open a pull request with confidence about what will happen to it.

## Stage 6 — Operations and failure · 2 days

Now the awkward parts, because you will meet them during an incident.

1. `docs/aws-architecture.md` — the AWS layout, §Monitoring especially
2. `dms_error.md` — a real incident post-mortem; the best single page on how this breaks
3. `src/sources/zorro/job_validation_zorro_silver_rds.py` — read the module docstring, then
   `_apply_grace` and `_row_compare`
4. `src/dags/dag_zorro_dms_reload.py` — read only the module docstring and the task order at
   the bottom. **Do not attempt all 1,368 lines yet.**

✅ **You can:** work out why the pipeline has stopped and what to do about it.

## Stage 7 — The deep end · ongoing

Only when you need them:

- `job_silver_ideon_normalize_benefit_design.py` (823) — the benefit text parser
- `job_gold_tornado_blended_plan_data.py` (500) — the blending rules
- `dag_zorro_dms_reload.py` (1,368) — the full recovery choreography
- `job_silver_zorro.py` — SCD2, currently switched off

These are specialist areas. Nobody holds all of them in their head.

---

# Part 4 — How long it takes

Honest estimates for someone comfortable with Python, SQL and cloud basics.

| Goal | Time |
|---|---|
| Explain what the repo does | **Half a day** |
| Run it locally, find your way around | **1 day** |
| Trace a change through one pipeline | **1 week** |
| Safely make a change to a job or model | **2 weeks** |
| Debug a production problem unaided | **4–6 weeks** |
| Deep expertise in the hard areas | **2–3 months** |

**"Understanding everything" is the wrong target.** Roughly 2,700 lines — the benefit parser,
the blend, the DMS reload — are specialist code that people return to with the docs open.
Being *productive* takes about two weeks; being *fluent* takes a couple of months; total
recall is not how anyone works with this repo.

## What actually makes it faster

**Read the docstrings first.** They explain *why*, often citing the incident that prompted the
code. Nothing in the source will teach you as quickly.

**Always ask "which pipeline?"** Almost every early mistake is applying Zorro reasoning to
Ideon code, or vice versa.

**Type the questions out.** After each stage, write your answers down. If you cannot explain
watermarks in a sentence, you have not finished Stage 2.

**Trust the code over the docs.** The documentation is unusually good but has drifted in
places. `docs/REPOSITORY_GUIDE.md` lists the known discrepancies — read that before trusting
any doc claim in detail.

## Common early confusions

| You think | Actually |
|---|---|
| Silver is part of the Zorro pipeline | It exists but nothing runs it |
| The committed PK mapping file is current | It is stale; AWS reads a copy from S3 |
| dbt is run by Airflow | It deploys from GitHub Actions |
| `gold` holds only pipeline tables | It holds pipeline tables *and* dbt views |
| `job_validation_zorro_silver_rds` validates silver | It validates **gold**; the name is a fossil |
