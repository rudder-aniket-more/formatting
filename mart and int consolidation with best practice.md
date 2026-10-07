# Same-grain models: what can be merged, and what is stopping the rest

**Scope:** the dbt project as it is on branch `fix/mart_refactoring` (commit `57b8a7ea`):
135 models — 35 staging, 77 intermediate, 23 reports (marts).
Every statement below was taken from the model SQL. Where the answer depends on
the data rather than the code, it says so and gives the query to check it.

---

## How to read this document

**Grain** = what one row of a model represents (e.g. *one row per employee per month*).
Two models with the same grain are candidates for merging. Whether they *can* be
merged depends on one more question: **do they apply the same rules and filters?**

Every same-grain group falls into one of three buckets:

| Bucket | Meaning | Effect on reports if merged | Groups |
|---|---|---|---|
| **1 — Merge** | Same grain, same rules. The merge only moves SQL; it cannot change a number. | None | 11 |
| **2 — Merge after a data check** | Same rules, but one detail (a tie, a duplicate, a population edge) means the result is only identical if the data behaves as expected. | None, *if the check returns 0 rows* | 4 |
| **3 — Do not merge without a business decision** | Same grain, but the business rule or filter is different. Merging forces one rule onto both reports. | Numbers change in at least one report | 14 |

### How this follows the project guidelines

Every proposal here was checked against [`data-analytics-architecture.md`](data-analytics-architecture.md)
(**AA**) and [`dbt-best-practices.md`](dbt-best-practices.md) (**BP**):

| Guideline | How it is applied here |
|---|---|
| **AA §4.2 single-definition rule** — every business term has one definition site; "which record is current" is applied once | Bucket 1 duplicates (A1–A6) and Bucket 2 are exactly this rule. Every Bucket 3 group is a **current violation** of it: two definitions of one term. AA requires them to be resolved, so the decisions at the end are not optional clean-up. |
| **AA §4.3 ladder** — keep logic inline (rung 3) unless a second consumer, a grain change, unit-testable branching or readability justifies its own model | Trivial helpers with one reader are folded (A7, A9, A10). Helpers with their own rule or complex logic stay (rung 2, bullets 3–4); see the "Rule for helpers" under Bucket 1. |
| **BP §4 / AA §3.2** — intermediate isolates complex operations so they can be tested and read on their own | Same as above: the kept helpers are listed with the reason. |
| **AA §3.2 folders** — `logic/` = ≥2 consumers or complex enough to test alone; `pipelines/<family>/` = one consumer chain | Models that absorb others stay in their current folder; the new shared model proposed in C12 goes in `logic/`. |
| **BP §4 naming** — `int_<entity>s_<verb>` | New model names follow it (C12: `int_major_medical_benefits_in_force_today`). |
| **BP rule 47 / W** — a CTE duplicated across models becomes its own intermediate model | Applied in C12 (the "in force today" rule is written three times) and B4 (copies inside other SQL). |
| **AA §7 / BP §10** — `not_null` on join keys; grain asserted with `unique_combination_of_columns` | Every merge moves the removed model's key tests to the model that absorbs it, and B3 adds the missing `unique` test. |
| **AA §3.1** — no `cast(... as date/boolean)` outside staging | No proposed SQL adds a cast outside staging. Copied existing lines (e.g. `cast(op.is_active as boolean)` in B1/C12 snippets) are quoted as they are today, not new code. |

### Summary

| | Models |
|---|---|
| Intermediate models today | **77** |
| Removed by Bucket 1 | **−9** (duplicates −5, simple helpers −4) |
| Removed by Bucket 1 (optional, see A11) | −1 |
| **Intermediate models after Bucket 1** | **68** (67 with the optional one) |
| Bucket 2 | removes duplicated *rules*, not models (count unchanged) |
| Bucket 3 | only after a business decision; see "If merged" for each group |

---

## Bucket 1 — same grain, same rules: safe to merge

Two kinds of merge are in this bucket:

* **Duplicates** — two models compute the same thing (A1–A6).
* **Simple helpers at their reader's grain** — a model read by exactly one other
  model, at the same grain, whose SQL is only a pass-through, a rename or a
  single simple pick/aggregation. It becomes a step (CTE) inside its reader (A7–A11).

**Rule for helpers:** a same-grain model that holds **its own complex logic or its
own business rule stays a separate model**, even with a single reader. It can then
be read, reviewed and tested on its own. Only trivial helpers are folded. The
helpers kept for this reason are listed under A7–A10.

### A1. Majority county per ZIP — two identical models

| | |
|---|---|
| **Grain** | ZIP × plan year |
| **Models** | `int_majority_fips_by_zip`, `int_plan_pricing_zip_majority_fips` |
| **Final reports** | `mrt_lowest_cost_plan` (first), `mrt_issuers_and_plan_counts` (second) |
| **Rules compared** | Identical: same source (`int_plan_pricing_zip_reference`), same `count(*)` per ZIP/year/FIPS, same tie-break. The only textual difference is `order by fips_count desc, fips_code` vs `fips_code asc` — `asc` is the default, so they are the same. |
| **If merged** | No change. |

SQL change — delete `int_plan_pricing_zip_majority_fips`; in `int_plan_pricing_zip_issuer_plans`:

```sql
-- before
inner join {{ ref('int_plan_pricing_zip_majority_fips') }} ppzmf
-- after
inner join {{ ref('int_majority_fips_by_zip') }} ppzmf
```

### A2. FIPS → county lookup — a projection of the ZIP/county lookup

| | |
|---|---|
| **Grain** | `int_plan_pricing_zip`: ZIP × FIPS × county. `int_fips_county_map`: FIPS × county |
| **Final reports** | `int_fips_county_map` → `mrt_compliance_lcsp_report`. `int_plan_pricing_zip` → 7 enrollment reports |
| **Rules compared** | Both are `select distinct` over `int_plan_pricing_zip_reference`. The county map is the same distinct list with the ZIP column dropped and null FIPS removed. |
| **If merged** | No change. A FIPS code with two county names returns two rows today and still would. |

SQL change — delete `int_fips_county_map`; in `int_compliance_employee` (`location_pivot`):

```sql
-- before
LEFT JOIN {{ ref('int_fips_county_map') }} fcm ON fcm.fips_key = l.res_fips
-- after
LEFT JOIN (
    select distinct fips_code as fips_key, county_name
    from {{ ref('int_plan_pricing_zip') }}
    where fips_code is not null
) fcm ON fcm.fips_key = l.res_fips
```

### A3. "AOR changed" per employer and year — re-reads the plan-year spine's source

| | |
|---|---|
| **Grain** | Employer × plan year |
| **Models** | `int_employer_aor_changed_by_year` vs `int_employer_plan_year_spine` (one row per OEP, already carries `is_aor_changed`) |
| **Final reports** | `mrt_employer_enrollment_participation` |
| **Rules compared** | Both read every row of `stg_open_enrollment_period` with no filter. The AOR model groups by employer and year and takes "any OEP flagged". |
| **If merged** | No change: the same group-by over the same rows. |

SQL change — delete `int_employer_aor_changed_by_year`; in `int_employer_participation_by_year`:

```sql
-- add at the top
with aor_changed as (
    select
        employer_id,
        year(effective_from)                                 as plan_year,   -- spine.plan_year is varchar
        max(case when is_aor_changed then 1 else 0 end) = 1  as is_aor_changed
    from {{ ref('int_employer_plan_year_spine') }}
    group by employer_id, year(effective_from)
),
-- and replace
left join {{ ref('int_employer_aor_changed_by_year') }} aor_next
-- with
left join aor_changed aor_next
```

### A4. Enrollment-team list — an employer attribute kept in its own model

| | |
|---|---|
| **Grain** | Employer (same as `int_employer_master`) |
| **Models** | `int_enrollment_team` vs `int_employer_master` |
| **Final reports** | `mrt_dependent_age_transition_26`, `mrt_medicare_dependent_age_transition_65`, `mrt_employer_level_details`, `mrt_main_enrollment_report`, `mrt_ops_and_actual_premium`, `mrt_ops_and_projected_premium` |
| **Rules compared** | No rule conflict: it is one more column about the employer. All 4 readers already read the master, directly or through `int_employee_enrollment_base`. |
| **If merged** | No change. The list is grouped per employer before the join, so the master stays one row per employer. |

SQL change — delete `int_enrollment_team`; in `int_employer_master` add:

```sql
enrollment_teams as (
    select ete.employer_id,
           array_join(array_agg(distinct ag.name order by ag.name), ' | ') as enrollment_team_list
    from {{ ref('stg_enrollment_team_to_employer') }} ete
    left join {{ ref('stg_agency') }} ag on ete.agency_id = ag.agency_id
    group by ete.employer_id
)
-- select: et.enrollment_team_list
-- from:   left join enrollment_teams et on et.employer_id = er.employer_id
```

Then `int_employee_enrollment_base` adds `emp.enrollment_team_list`. The 4 readers
replace `team.enrollment_team_list` with `er.enrollment_team_list` (or
`base.enrollment_team_list`) and drop the `int_enrollment_team` join.

### A5. NAICS industry title — the same lookup written twice

| | |
|---|---|
| **Grain** | Employer |
| **Models** | `int_participation_employers` (`nm_detailed`) and `int_employer_level_details` (`ni`) |
| **Final reports** | `mrt_employer_enrollment_participation`, `mrt_employer_level_details` |
| **Rules compared** | Identical: `left join stg_naics_code_to_industry_mapping on naics_code = employer.naics_code`, taking `naics_title`. |
| **If merged** | No change. The NAICS seed (2,125 rows) has no repeated codes, so neither copy duplicates an employer today. |
| **Models removed** | 0. This removes a duplicated rule, not a model. |

SQL change — in `int_employer_master`, add `naics_industry`:

```sql
left join (
    select naics_code, max(naics_title) as naics_title      -- max() guards against a future repeated code
    from {{ ref('stg_naics_code_to_industry_mapping') }}
    group by naics_code
) nx on nx.naics_code = er.naics_code
-- select: nx.naics_title as naics_industry
```

Both readers then use `naics_industry` from the master and drop their own join.

### A6. Plan start date + ages (Compliance) — one model read only to feed the other

| | |
|---|---|
| **Grain** | Employee × plan year |
| **Models** | `int_lcsp_employee_plan_start`, `int_lcsp_age_calculations` |
| **Final reports** | `mrt_compliance_lcsp_report` |
| **Rules compared** | The ages model is `int_lcsp_base_employees left join int_lcsp_employee_plan_start`, and the plan start is used only to compute the age. The plan-start model has one other reader (`int_compliance_employee`), which takes the same date. |
| **If merged** | No change. |

SQL change — delete `int_lcsp_employee_plan_start`; `int_lcsp_age_calculations` becomes:

```sql
with plan_start as (
    select employee_id, plan_year, min(det_date) as actual_plan_start_date
    from {{ ref('int_lcsp_monthly_eligibility_status') }}
    where is_eligible = 'Yes'
    group by employee_id, plan_year
)
select
    b.employee_id,
    b.plan_year,
    ps.actual_plan_start_date,                                   -- new column
    {{ age_at_date('b.date_of_birth', "cast(concat(cast(b.plan_year as varchar), '-01-01') as date)") }} as age_jan_1,
    ... age_at_plan_start unchanged ...
from {{ ref('int_lcsp_base_employees') }} b
left join plan_start ps on ps.employee_id = b.employee_id and ps.plan_year = b.plan_year
```

`int_compliance_employee` drops its `int_lcsp_employee_plan_start` join and reads
`a.actual_plan_start_date`.

### A7. Payment report — one simple helper folded, three kept

| | |
|---|---|
| **Grain** | Pull expectation (all five models) |
| **Final reports** | `mrt_payment_report` |
| **Fold** | `int_payment_expected_amount_latest`: picks the highest version per expectation (one `row_number`). **Models removed: 1.** |
| **Keep separate (own logic)** | `int_payment_expectation_bounds`: business-day window that skips weekends and federal holidays. `int_payment_settled_actuals`: nets disbursements against returns over confirmed matches, with carrier mapping. `int_payment_candidate_matches`: rule "candidate links not already confirmed for the same pair". |
| **If merged** | No change. |

SQL change in `int_payment_expectation_rows`:

```sql
with expected_amount_latest as (
    select expectation_id, amount_in_cents as expected_amount_in_cents
    from (select *, row_number() over (partition by expectation_id order by version desc) as rn
          from {{ ref('stg_expected_amount_version') }})
    where rn = 1
)
...
left join expected_amount_latest le on le.expectation_id = e.expectation_id   -- was ref('int_payment_expected_amount_latest')
```

### A8. Employer participation — both helpers kept

`int_employer_enrolled_by_year` and `int_employer_headcount_by_year` share the
employer × plan year grain and have one reader (`int_employer_participation_by_year`),
but **both stay separate**:

* Enrolled holds the status list and the calendar-year containment rule, which differs from Employer Level Details (C1).
* Headcount is short SQL, but it is the named "employed the whole year" measure, which differs from two similar counts (C3).

Models removed: 0.

### A9. Employer details — allowance-model upload date

| | |
|---|---|
| **Grain** | Employer × plan year |
| **Fold** | `int_employer_allowance_model_by_year`: `min(created_at)` per employer and year, read only by `int_employer_level_details`. **Models removed: 1.** |
| **Keep separate (own logic)** | `int_employer_plan_year_counts`: five counts with status lists and scoping flags. |
| **Final reports** | `mrt_employer_level_details` |
| **If merged** | No change. |

```sql
-- in int_employer_level_details
allowance_model as (
    select employer_id, cast(year as varchar) as plan_year, min(created_at) as allowance_model_uploaded_date
    from {{ ref('stg_allowance_model') }}
    group by employer_id, cast(year as varchar)
)
-- replace ref('int_employer_allowance_model_by_year') amf with allowance_model amf
```

### A10. Main Enrollment — two simple helpers folded, one kept

| | |
|---|---|
| **Grain** | Onboarding period (enrollment) |
| **Fold** | `int_care_preferences`: a two-column copy of `stg_decision_factors_preference`, read by the mart. `int_latest_application_sent`: latest `APPLICATION_SENT` per enrollment (one `row_number`), read only by `int_employee_enrollment_enriched`. **Models removed: 2.** |
| **Keep separate (own logic)** | `int_onboarding_period_counts`: three aggregations (providers, drugs, documents) unioned and joined. |
| **Final reports** | `mrt_main_enrollment_report` |
| **If merged** | No change. |

SQL change:

```sql
-- mrt_main_enrollment_report
left join {{ ref('int_care_preferences') }} cp            -- before
left join {{ ref('stg_decision_factors_preference') }} cp -- after (same two columns)

-- int_employee_enrollment_enriched: add a CTE and repoint the join
latest_application_sent as (
    select onboarding_period_id, employee_last_application_sent_to_carrier_utc, employee_last_application_sent_performed_by
    from (select onboarding_period_id,
                 created_at as employee_last_application_sent_to_carrier_utc,
                 concat(performed_by_first_name, ' ', performed_by_last_name) as employee_last_application_sent_performed_by,
                 row_number() over (partition by onboarding_period_id order by created_at desc) as rn
          from {{ ref('stg_enrollment_activity_log') }}
          where activity = 'APPLICATION_SENT')
    where rn = 1
)
left join latest_application_sent eal on eal.onboarding_period_id = base.employee_op_id
```

### A11. *(Optional)* OPS & Projected Premium — rate-change pass-through

`int_requested_rate_change` only renames `year` to `plan_year` from
`stg_requested_rate_change`, and only `mrt_ops_and_projected_premium` reads it. It
can be folded in (**−1**), but that report is deprecated: branch
`origin/fix/remove_op_and_projected_model` removes the report and this model
together, which is the better route.

---

## Bucket 2 — same rules, merge after a data check

Each of these groups has the same rule written more than once. The merge is
identical *if* the check returns no rows. These merges remove duplicated rules,
not models.

### B1. "Deactivated employer" — two versions of the latest OEP

| | |
|---|---|
| **Grain** | Employer |
| **Models** | `int_employer_master` (used by 17 reports) vs `int_current_month_active_employers` |
| **Final reports of the second** | `mrt_active_covered_employees_current_month`, `mrt_ajg_current_month_coverage_active_employees`, `mrt_count_of_active_covered_employees_current_month` |
| **Rule difference** | Master: ACTIVE becomes Deactivated when the **latest-starting** OEP has ended (`row_number() … order by effective_from desc, oep_id desc`). Current-month: when the **latest end date** of any OEP has passed (`max(effective_until)`). Label `'Deactivated'` vs `'DEACTIVATED'` (cosmetic). Filter `signup_status = 'ACTIVE' and is_demo = false` in both. |
| **Identical when** | No employer has an earlier-starting OEP that ends after its latest-starting one, i.e. no overlapping OEPs. |
| **If merged and the check fails** | Those employers flip between ACTIVE and DEACTIVATED in the three current-month snapshot reports. |

```sql
-- Data check (expect 0 rows)
select employer_id
from stg_open_enrollment_period
group by employer_id
having max(effective_until) <> max_by(effective_until, effective_from);
```

SQL change — `int_current_month_active_employers` becomes a filter over the master:

```sql
select
    employer_id, employer_name, legal_name as employer_legal_name,
    state as employer_hq_state, producer_id,
    case when signup_status = 'Deactivated' then 'DEACTIVATED' else signup_status_raw end as employer_status
from {{ ref('int_employer_master') }}
where signup_status_raw = 'ACTIVE' and is_not_demo
```

### B2. Enrollment status label — computed twice with the same macro

| | |
|---|---|
| **Grain** | Onboarding period × major-medical benefit |
| **Models** | `int_employee_enrollment_base` and `int_employer_plan_year_op_status` both call `employee_enrollment_status(...)` with the same inputs and `as_of_date = current_date`. The `month_restricted` argument they pass differently is ignored by the macro (its header says so). |
| **Final reports** | Base → 7 enrollment reports. Op-status → `mrt_employer_level_details` |
| **Identical when** | Every onboarding period's employee belongs to the OEP's employer. Base reaches the OP through the employee; op-status reaches it through the OEP. Base also excludes prospect employers, but the Employer Level Details mart filters prospects out anyway. Base duplicates rows per county / QLE, which is harmless because op-status is only counted with `count(distinct employee_id)`. |
| **If merged and the check fails** | Those enrollments drop out of Employer Level Details' enrolled / waived / shopping counts. |

```sql
-- Data check (expect 0 rows)
select op.onboarding_period_id
from stg_onboarding_period op
join stg_open_enrollment_period oep on oep.oep_id = op.enrollment_period_id
join stg_employee e on e.employee_id = op.employee_id
where e.employer_id <> oep.employer_id;
```

SQL change — `int_employer_plan_year_op_status` reads `employee_enrollment_status` from
`int_employee_enrollment_base` (joined on `employee_op_id`) instead of recomputing it
from `stg_benefit`.

### B3. The employee's own insured record on an enrollment — written three times

| | |
|---|---|
| **Grain** | Onboarding period (the `EMPLOYEE` row of `stg_insured`) |
| **Models** | `int_insured_employee_address` (`select distinct … where type = 'EMPLOYEE'`), `int_employee_enrollment_base` (`left join stg_insured ee_ins … and ee_ins.type = 'EMPLOYEE'`, no distinct), and `int_lcsp_monthly_locations` (inline, see C9) |
| **Final reports** | Address model → Monthly LCSP, Insured Coverage Months, All-Employer LCSP. Base → 7 enrollment reports |
| **Identical when** | Each enrollment has exactly one `EMPLOYEE` insured row. No test enforces this today (`int_insured_employee_address.period_id` has `not_null` but no `unique`). |
| **If merged and the check fails** | Base has one row per `EMPLOYEE` insured row today; reading the distinct model would remove those duplicate rows from the 7 enrollment reports. |

```sql
-- Data check (expect 0 rows)
select period_id, count(*) from stg_insured
where type = 'EMPLOYEE' group by period_id having count(*) > 1;
```

SQL change — in `int_employee_enrollment_base`:

```sql
left join {{ ref('stg_insured') }} ee_ins on ee_ins.period_id = op.onboarding_period_id and ee_ins.type = 'EMPLOYEE'  -- before
left join {{ ref('int_insured_employee_address') }} ee_ins on ee_ins.period_id = op.onboarding_period_id             -- after
```

Add a `unique` test on `int_insured_employee_address.period_id` at the same time.

### B4. Same rule copied inside other SQL

These copies sit inside a model or report rather than in a model of their own, so no model is removed:

| Where | Same rule as | Final report | SQL change | Identical? |
|---|---|---|---|---|
| `int_lcsp_monthly_lcsp_values` joins `stg_plan_pricing_zip` + `stg_medical_plans` `where lower(metal_level) = 'silver' and on_market = true` | `int_silver_plan_candidates` (same filter, same join) | `mrt_compliance_lcsp_report` | Read `int_silver_plan_candidates` (`pricing_zip`, `pricing_fips`, `age`, `base_premium`) | Yes, by construction |
| `mrt_employee_enrollment_status_names_invoices` recomputes `row_number() over (partition by employee_id order by oep from desc, op from desc, op created desc)` | `int_employee_enrollment_base.latest_employee_id` (same partition, same order) | Enrollment Status (Invoices) | `where latest_employee_id = 1` on the base column | Yes, except exact ties: both copies pick arbitrarily among tied rows |
| `mrt_employee_employer_ytd_contribution` `oep_per_year` (latest OEP per employer × year; only employer and year are used) | `int_employer_plan_year_spine` | YTD Contribution | `select distinct employer_id, year(effective_from) as plan_year from spine where effective_from is not null`; drop `and oep.rn = 1` | Yes |

---

## Bucket 3 — same grain, different rules or filters: do not merge without a decision

Each group below has models with the same grain doing the same job, but with a
different rule or filter. Merging means choosing one rule, and the reports on
the other rule change. For each group:

* **Today** shows the exact difference in SQL.
* **If merged** shows what changes, and where.

### C1. "Enrolled employees" per employer and plan year — two definitions

| | `int_employer_enrolled_by_year` | `int_employer_plan_year_counts.number_of_enrolled` |
|---|---|---|
| **Final report** | `mrt_employer_enrollment_participation` | `mrt_employer_level_details` |
| Which statuses count | Benefit status `AWAITING_PAYMENT`, `ENROLLMENT_CONFIRMED`, `ACTIVE`, `ENDED` | Status label Election Submitted (`READY_TO_APPLY`, `AWAITING_MEDICAL`, `AWAITING_EMPLOYEE`), Enrollment in Progress (`CARRIER_APPLICATION_SENT`, `AWAITING_PAYMENT`), Enrollment Confirmed, Coverage Active, Coverage Ended |
| How an enrollment is assigned to a year | The OP window lies **inside the calendar year** (`op.effective_from >= Jan 1 and op.effective_until <= Dec 31`) | The OP belongs to **that year's OEP**, then mode rules (pre-OE: starts on the OEP start; during year: still in force today; after year: latest OP only) |
| Employers | Non-demo, non-prospect | All employers with an OEP (the mart removes prospects; **demo employers stay**) |

```sql
-- Participation
and benf.status in ('AWAITING_PAYMENT', 'ENROLLMENT_CONFIRMED', 'ACTIVE', 'ENDED')
and op.effective_from >= y.year_start and op.effective_until <= y.year_end
-- Employer Level Details
when in_scope_enrolled_waived
 and employee_enrollment_status in ('Election Submitted','Enrollment in Progress',
                                    'Enrollment Confirmed','Coverage Active','Coverage Ended')
```

**If merged:**
* Using the Level Details rule makes Participation count employees whose election is submitted but not yet with the carrier. Mid-year enrollments that cross 31 Dec would also count.
* Using the Participation rule makes Level Details drop enrollments that are still in progress.
* **Decision needed:** which statuses mean "enrolled".

### C2. Employer × year row list — two spines

| | Participation spine (inside `int_employer_participation_by_year`) | `int_employer_plan_year_spine` |
|---|---|---|
| **Final report** | `mrt_employer_enrollment_participation` | `mrt_employer_level_details` |
| Rows | Every participation employer × **every year in which any employer has an OEP** (`cross join`) | One row per **the employer's own** OEP |
| Effect | Employers appear in years before they joined, with zero counts | Only years in which the employer had an open enrollment |

**If merged:**
* Using the OEP spine removes Participation rows for years an employer had no OEP. Those rows show `total_employees` but no enrollment.
* Using the cross join adds empty years to Level Details.

### C3. Headcounts with similar names but different measures

| Model | Measure | Final report |
|---|---|---|
| `int_employer_headcount_by_year` | Employed the **whole calendar year**: `hire_date < Jan 1 and termination_date > Dec 31` | Employer Enrollment Participation |
| `int_employer_plan_year_counts.number_of_eligible` | **Eligible** on the plan year's as-of date (employee record only) | Employer Level Details |
| `int_count_of_active_covered_employees_current_month.all_hired_not_terminated_employee_count` | Hired by the 1st of this month and not terminated before it | Count of Active Covered Employees |

These are different measures, so they should not be merged. The only action
needed is to keep the column names distinct.

### C4. Employee × plan year population — Compliance/PCORI vs All-Employer LCSP

| | `int_lcsp_base_employees` | `int_lcsp_all_employees_eligible` |
|---|---|---|
| **Final reports** | `mrt_compliance_lcsp_report`, `mrt_pcori_report` | `mrt_lcsp_all_employer_report` |
| Eligibility window tested | Calendar year: **1 Jan – 31 Dec** | **OEP start – 31 Dec** |
| OEPs in a year | Every OEP (two OEPs in a year would give two rows) | Earliest OEP only (`row_number() … order by effective_from`, keep 1) |
| Extra filter | — | Computed plan start must fall on or before 31 Dec |

```sql
-- Compliance / PCORI
COALESCE(e.eligible_from, DATE '1900-01-01') <= <Dec 31> AND COALESCE(e.eligible_until, DATE '9999-12-31') >= <Jan 1>
-- All-Employer
coalesce(b.eligible_from, date '1900-01-01') <= b.year_end and coalesce(b.eligible_until, date '9999-12-31') >= b.oep_effective_from
... where plan_start_date <= year_end
```

**If merged:**
* Using the Compliance rule adds employees to All-Employer LCSP whose eligibility ended between 1 Jan and the OEP start.
* Using the All-Employer rule removes them from Compliance and PCORI.

### C5. Monthly eligibility — two models

| | `int_lcsp_eligible_employees` | `int_lcsp_monthly_eligibility_status` |
|---|---|---|
| **Final reports** | `mrt_monthly_lcsp_report`, `mrt_insured_coverage_months` | `mrt_compliance_lcsp_report` (and its plan start, ages, allowance, LCSP) |
| Rows | **Eligible months only** | Every month of the year, `'Yes'` / `'No'` |
| Month must be inside the OEP | **Yes**: `determination_date between oep.effective_from and oep.effective_until` | **Start only**: `det_date < oep_effective_from → 'No'`; no end bound |
| Which OEP's year | Any OEP covering the month (an OEP starting in 2024 that runs into 2025 is included) | OEP plan years from 2025 |

**If merged:**
* With one rule, Compliance shows `'No'` for months after an OEP has ended when it ends before 31 Dec. Today it shows `'Yes'`.
* That also moves Compliance's plan start, ages, allowance and LCSP for those employees.
* **Decision needed:** does eligibility stop when the employer's OEP ends?

### C6. Plan start date — three rules

| Model | Rule | Fallback when no start | Final reports |
|---|---|---|---|
| `int_lcsp_plan_start` | First eligible month **inside the OEP** (from C5 left) | OEP start date (for age) | Monthly LCSP, Insured Coverage Months |
| `int_lcsp_employee_plan_start` | First `'Yes'` month (from C5 right; no OEP end bound) | **1 Jan** (age) and **'Jan'** (displayed month) | Compliance LCSP |
| inline in `int_lcsp_all_employees_eligible` | `greatest(OEP start, eligible_from)`, rounded up to the next 1st of the month. `eligible_from` falls back to the **latest** history start, and `eligible_until` is ignored | — | All-Employer LCSP |

**If merged:** plan start, and so `age_at_plan_start` and the LCSP premium (rated
by age), change for employees who:
* become eligible after the OEP ends; or
* have more than one eligibility window; or
* have no eligible month at all.

### C7. Which enrollment represents an employee-month — three rules

| | `int_lcsp_covered_data` | `int_lcsp_member_coverage` | `int_employee_benefit_coverage_monthly` |
|---|---|---|---|
| **Final reports** | Monthly LCSP | Insured Coverage Months | Compliance LCSP, PCORI |
| OPs considered | Covering the month **and** (coverage-status benefit **or** waived) | **Every** OP covering the month | Every OP of the OEP (no selection: one row per OP per benefit) |
| Order | Non-waived first → `updated_at desc` → `effective_from desc` → `op_id` | Coverage-status first → non-waived → `updated_at desc` → `op_id` | — (covered = any OP qualifies) |
| "Covered" requires closed enrollment (`is_active = false`) | n/a | **No** (`has_active_coverage` ignores `is_active`) | **Yes** |

```sql
-- covered_data: filter, then rank
where (benf.benefit_id is not null and benf.status in (<5 coverage statuses>)) or op.is_waived = true
order by case when op.is_waived = false then 0 else 1 end, op.updated_at desc, op.effective_from desc, op.onboarding_period_id
-- member_coverage: no filter
order by case when <coverage status> then 0 else 1 end, case when op.is_waived = false then 0 else 1 end,
         op.updated_at desc, op.onboarding_period_id
```

Concrete case: a month with an **abandoned** (non-waived, not coverage-status) enrollment and a **waived** one.
* Monthly LCSP picks the **waived** enrollment.
* Insured Coverage Months picks the **abandoned** one.

That changes `effective_from/until`, the address (C9) and, in Insured Coverage
Months, the dependents listed.

**If merged:** Monthly LCSP gains coverage dates for months with only an abandoned
enrollment, or Insured Coverage Months loses them. **Decision needed:** one
selection order (the logic review recommended coverage → waived → other).

### C8. Resolved employee-month and LCSP ranking — blocked only by C6/C7/C9

| Same grain (employee × month) | Final reports | Difference |
|---|---|---|
| `int_lcsp_employee_resolved` vs `int_lcsp_member_resolved` | Monthly LCSP vs Insured Coverage Months | **The SQL is the same** except the coverage source (`int_lcsp_covered_data` vs `int_lcsp_member_coverage`, C7). The member version adds `id_from_employer`, `gender`, `ssn`, `allowance` and `has_active_coverage`. |
| `int_lcsp_monthly_ranked` vs `int_lcsp_member_ranked` vs `int_lcsp_monthly_lcsp_values` | Monthly LCSP vs Insured Coverage Months vs Compliance LCSP | **The ranking rule is the same:** silver, on-market, `zip_fips_match`, `rating_age`, cheapest premium, tie on plan id. Only the inputs differ: address (C9) and age at plan start (C6). |

**If merged:** once C6, C7 and C9 are decided, these models collapse into one
monthly model and one ranking model, removing **3 models** (two resolved → one,
three ranked → one). Merging them before then forces C7's rule onto one report.

### C9. Employee address per month — three rules

| | Monthly LCSP / Insured Coverage Months | Compliance (`int_lcsp_monthly_locations`) | All-Employer (inline) |
|---|---|---|---|
| Step 1 | `EMPLOYEE` insured on **the month's selected enrollment** (C7) | `EMPLOYEE` insured on **any** enrollment with a coverage-status benefit covering the month: not limited to the OEP, and **arbitrary** if several | `EMPLOYEE` insured on **the year's** selected enrollment |
| Step 2 | Latest change log `created_at <=` month | Same | Latest change log `<=` plan start |
| Step 3 | Employee record | Employee record, **eligible months only** | Employee record |

**If merged:** Compliance's ZIP / FIPS / county (and so LCSP) changes for employees
with more than one enrollment covering a month.

### C10. Benefit class and allowance — two rules

| | `int_lcsp_member_attributes` + `int_lcsp_member_allowance` | `int_lcsp_monthly_allowances` |
|---|---|---|
| **Final reports** | Insured Coverage Months | Compliance LCSP |
| Employee attributes used | **Latest** change-log row on or before the month | Latest row **of every earlier calendar month**; class matching then picks the **best score across all of them** |
| Age for the allowance band | `age_at_plan_start` with OEP-start fallback (C6) | `age_at_plan_start` with 1 Jan fallback (C6) |

```sql
-- Insured Coverage Months: one row, the latest
row_number() over (partition by employee_id, determination_date order by event_timestamp desc) ... and rn = 1
-- Compliance: one row per earlier month, all kept
ROW_NUMBER() OVER (PARTITION BY employee_id, date_trunc('month', event_timestamp) ORDER BY event_timestamp DESC) ... ecl.rn = 1
   -- joined with event_timestamp <= det_date: every earlier month's row survives
```

**If merged:** with the "latest" rule, Compliance's allowance can change for
employees whose class attributes changed during the year. Today Compliance can
pick an out-of-date class.

### C11. Dependent coverage — Compliance vs PCORI

| | `int_compliance_dependent_coverage` | Dependent branch inside `mrt_pcori_report` |
|---|---|---|
| Grain | **Insured record** × plan year | **Person** (employee + name + relationship) × plan year |
| Filter | Every non-employee insured on the OEP's enrollments | Only insured people **linked to a benefit** (`join stg__benefit_to_insured`) |
| Coverage rule | **Same**: `ec.is_covered` and `is_dependent_covered(family unit of that benefit, type)` | Same |

**If merged:**
* Moving PCORI to the insured-record grain counts the same child on two enrollments twice, which over-counts covered lives in the PCORI fee base.
* Moving Compliance to the person grain merges them into one row, and adding the link filter drops unlinked dependents from Compliance.

**Recommendation:** share the *coverage rule* (it is already a shared macro) and keep both grains.

### C12. "Covered today" — the same rule, three populations

| Model | Population | Selection | Final reports |
|---|---|---|---|
| `int_current_month_active_covered_base` | ACTIVE (not deactivated), non-demo employers; employees eligible today | Every enrollment/benefit in force today | Active Covered Employees, AJG |
| CTE `active_covered_base` in `int_count_of_active_covered_employees_current_month` | ACTIVE **or two override employers** (`is_active_or_override_employer`) | Same | Count of Active Covered Employees |
| `int_latest_coverage_active` | **All employers, all employees** (no eligibility test) | **Latest** in-force enrollment (`order by op.effective_from desc`; ties arbitrary) | Dependent Age Transition (26), Medicare & Dependent Age Transition (65) |

The coverage rule is **identical** in all three:

```sql
op.is_active = false and op.is_waived = false and benf.benefit_id is not null
and benf.benefit_type = 'MAJOR_MEDICAL' and benf.status != 'IN_CART'
and op.effective_from <= current_date and op.effective_until >= current_date
```

**If merged into one model:**
* The Count report loses the two override employers' covered employees.
* The age-transition reports would only see employees eligible today at ACTIVE employers.

**Recommendation:** one shared model, `intermediate/logic/int_major_medical_benefits_in_force_today`
(one row per enrollment × major-medical benefit, no population filter), with each
reader applying its own population. This adds one model and removes the three
copies of the rule. It changes no numbers. This is AA §4.3 rung 2, first bullet:
a second (and third) model needs it.

### C13. "Eligible today" — the same rule, different employer lists

| Model | Employers | Rule | Final reports |
|---|---|---|---|
| `int_current_month_eligible_employees` | `employer_status = 'ACTIVE'` | Employee record covers today; **eligibility history ignored** | Active Covered Employees, AJG |
| CTE `all_employees` in `int_count_of_active_covered_employees_current_month` | ACTIVE **or override employers** | Same | Count of Active Covered Employees |
| `int_employer_eligible_employees_by_year` | Every employer with an OEP | Same rule, on the plan year's as-of date | Employer Level Details |
| LCSP models (C5) | Non-demo, non-prospect | Record **or eligibility history** | LCSP reports |

**If merged:**
* Moving the first three to the LCSP rule adds employees whose eligibility was restarted in advance. In that case the app moves the still-running window into eligibility history.
* Merging the first two removes the override employers from the Count report.

### C14. Other pairs with the same grain but different filters

| Models (grain) | Final reports | Difference | If merged |
|---|---|---|---|
| `int_medical_plan_level` vs `int_medical_plan_reference` (plan × year) | Main Enrollment vs Carrier and Plan ID Reference | Level: `where plan_id not like '%-%'`; key (plan_id, plan_year). Reference: no filter; adds issuer, plan name, **state**, market type; plan_year as varchar | Main Enrollment's metal-level join fans out by state (duplicate rows), or the Carrier reference loses plan IDs containing `-` |
| `int_ops_latest_enrollment` and `int_daily_enrollment_snapshot_base` vs `int_employee_enrollment_base` (enrollment × benefit) | OPS reports vs Daily Snapshot vs 7 enrollment reports | OPS: real employers, 4 statuses, ranked by year of the **enrollment start**. Snapshot: non-demo, OEP present, ranked by year of the enrollment (else OEP) start. Base: no filter, ranked by year of the **OEP start** | The "latest enrollment" pick changes for enrollments that start in a different year from their OEP |
| "Latest enrollment per employee and year" in `mrt_main_enrollment_report`, `mrt_employee_level_premium` vs `int_employee_enrollment_base.latest_employee_id_by_effective_year` | Main Enrollment, Employee Level Premium | Same order, but each report **filters first** (real employers and OEP before 1 Nov; non-demo, OEP before 1 Dec, non-Medicare carrier), then ranks | A Medicare or late-OEP enrollment could become the employee's row |

---

## Mart consolidation

The same question applied to the 23 reports. The project architecture
([`data-analytics-architecture.md`](data-analytics-architecture.md)) defines
what a mart should be:

* one business entity at one grain (§3.3);
* report-specific filters such as `days_until_26 <= 90` belong in Metabase, over a dbt-computed column (§2.3);
* fixed regulatory layouts are `exports/`, not marts (§3.4).

Two marts whose **only** differences are report filters are therefore one mart
plus two Metabase questions.

**Cost of every mart merge, even when numbers do not change:** the Metabase
questions on the old marts must be rebuilt with the filters moved into them.
Exposures are deferred (architecture §2.2), so dbt cannot list the affected
dashboards; they have to be found in Metabase first.

| | Reports |
|---|---|
| Today | **23** |
| After M1 + M2 (no business decision needed) | **21** |
| After M3–M5 (needs the C1/C2, C7 and C14 decisions) | **about 17** |

The architecture's full entity set can also *add* marts (every grain a report
reads must be a mart, §2.1), so the long-term number depends on those decisions.

### M1. Age transitions 26 and 65 — one mart, filters move to Metabase

| | |
|---|---|
| **Marts** | `mrt_dependent_age_transition_26`, `mrt_medicare_dependent_age_transition_65` |
| **Grain** | Both read `int_insured_age_transition_base` and end with `select distinct` over person-level columns: one row per person (employee, name, date of birth, type). |
| **Differences (all filters)** | **26:** `not is_prospect`, `type = 'CHILD_OR_OTHER_DEPENDENT'`, `days_until_26 between 0 and 90`, and the person is on the enrollment that holds the current coverage (`inner join stg_benefit … benefit_id = base.benefit_id`). **65:** `signup_status = 'ACTIVE' and has_active_oep`, types EMPLOYEE / SPOUSE / CHILD, `days_until_65 between 0 and 90`. |
| **Column differences** | 26 adds `employee_name` and `market_type`; 65 has no extra columns. |

SQL of the consolidated mart (sketch):

```sql
with people as (
    select
        base.employer_id, base.employee_id, base.employee_email,
        concat(base.employee_first_name, ' ', base.employee_last_name) as employee_name,
        base.partner_name, base.producer_name, base.enrollment_team_list, base.employer_name,
        base.full_name, base.type, base.date_of_birth, base.age,
        base.days_until_26, base.days_until_65,
        base.carrier_name, base.plan_market, base.employee_termination_date, base.employee_ops_url,
        -- gates as columns: the 26 / 65 questions filter on these
        not base.is_prospect                                  as is_not_prospect_employer,
        base.signup_status = 'ACTIVE' and base.has_active_oep as is_active_employer_with_oep,
        -- the 90-day windows as UNMASKED booleans (see "Masking" below)
        base.days_until_26 between 0 and 90                   as is_turning_26_within_90_days,
        base.days_until_65 between 0 and 90                   as is_turning_65_within_90_days,
        -- 26's inner join to stg_benefit, kept as a flag at person level
        bool_or(benf.benefit_id is not null)                  as is_on_current_coverage_enrollment
    from {{ ref('int_insured_age_transition_base') }} base
    left join {{ ref('stg_benefit') }} benf
        on benf.period_id = base.period_id
       and benf.benefit_id = base.benefit_id
       and benf.benefit_type = 'MAJOR_MEDICAL'
    group by 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22
)
select distinct   -- distinct AFTER masking, as today
    ...the same mask_pii(...) columns as today (both year_month_turn_26 and _65)...,
    is_not_prospect_employer, is_active_employer_with_oep, is_on_current_coverage_enrollment,
    is_turning_26_within_90_days, is_turning_65_within_90_days
from people
```

The `bool_or` at person level matters. Each person has one insured row per
enrollment, and only one of those rows is on the current-coverage enrollment. A
per-row flag would split a person into several rows, and the 65 question, which
never filters on the flag, would show duplicates.

Metabase questions:

* **Age 26:** `is_on_current_coverage_enrollment and is_not_prospect_employer and type = 'CHILD_OR_OTHER_DEPENDENT' and is_turning_26_within_90_days`
* **Age 65:** `is_active_employer_with_oep and is_turning_65_within_90_days`

**Masking: why the window is a boolean column, not a Metabase filter on
`days_until_*`.** Today both marts filter on the real `days_until_*` and then mask
it, together with names and date of birth, for the masked employer. In a
consolidated mart `days_until_*` is masked, so a Metabase filter on it would see
NULL and silently drop every row of that employer. The boolean window flags are
not masked. They reveal nothing new: today's report already shows that those rows
exist. This is a documented exception to architecture §2.3 (the "90 days" policy
value lives in dbt rather than in the question), forced by masking. If ops want
to tune the window without a PR, the alternative is a dbt `var`.

**Distinct after masking, as today.** Today, two people of the masked employer
whose visible columns are identical after masking collapse into one row. The final
`select distinct` runs after `mask_pii`, as today, and the filter flags are
constant within each question's result. So each question returns exactly today's
rows.

Milestone labels: the architecture says milestone flags are mart columns (§2.1),
so the mart keeps `age_milestone_flag` per person type
(`DEPENDENT_WILL_REACH_26_SOON`, `EMPLOYEE_WILL_REACH_65_SOON`, …), computed with
the same `case` as today.

**If merged:** no number changes, as long as the two Metabase questions apply
exactly the filters above. **Check before cutover:** for each question, compare
the row count, including the masked employer's rows, with the old mart.

### M2. OPS & Projected Premium — delete

Deprecated. Branch `origin/fix/remove_op_and_projected_model` removes
`mrt_ops_and_projected_premium` together with `int_requested_rate_change` and
`stg_requested_rate_change` (A11).

### M3. Employer reports — one `employer_plan_years` mart, after C1 / C2

| | |
|---|---|
| **Marts** | `mrt_employer_level_details` (one row per employer per OEP year) and `mrt_employer_enrollment_participation` (every participation employer × every OEP year) |
| **Blocked by** | Two definitions of "enrolled employees" (C1), and two row lists: the employer's own OEP years vs a cross join of all years (C2) |
| **If merged before deciding** | One report's enrolled count and row list change |
| **After the decision** | One mart, one row per employer per plan year. It carries both headcounts as differently-named measures (C3: whole-year headcount vs eligible on the as-of date), the participation percentages, and the employer attributes. |

### M4. Monthly LCSP + Insured Coverage Months — one `insured_coverage_months` mart, after C7

| | |
|---|---|
| **Marts** | `mrt_monthly_lcsp_report` (employee × month) and `mrt_insured_coverage_months` (employee **and dependent** × month) |
| **Same today** | Population: both start from `int_lcsp_eligible_employees`. Address, age and LCSP ranking use the same rules, applied to each report's own selected enrollment. |
| **Blocked by** | Which enrollment represents a month (C7). Also the employee name: `concat(first, ' ', last)` in Monthly LCSP vs `concat_ws(' ', first, last)` in Insured Coverage Months, which differ when a name part is null. |
| **After the decision** | Monthly LCSP = the question `relationship_type = 'EMPLOYEE'` on the one mart. Its columns are a subset under different names (`zip_code` vs `employee_zip_code`, `class` vs `employee_class`, …), so the question renames them. |

### M5. Enrollment reports — one `employee_enrollments` mart, after C14

| | |
|---|---|
| **Marts** | `mrt_employee_level_premium` (employee × plan year), `mrt_employee_enrollment_status_names_invoices` (employee, latest across all years), `mrt_ops_and_actual_premium` (employee × plan year, latest **priced** enrollment joined to pricing) |
| **Blocked by** | Each picks "the latest enrollment" its own way (C14). Level Premium removes Medicare carriers and OEPs starting after 1 Dec **before** ranking. Invoices ranks across all years. OPS ranks with unpriced enrollments last and keeps only priced ones. The architecture's single-definition rule (§4.2) requires one rule, or a documented named variant. |
| **After the decision** | One mart, one row per employee per plan year, with `is_latest_overall` for Invoices and the pricing columns for OPS. Each report becomes a filtered question. |

### Marts that stay separate

| Marts | Why |
|---|---|
| `mrt_compliance_lcsp_report`, `mrt_pcori_report` | Externally mandated layouts (12-month pivots, ~175 named columns): **exports** under architecture §3.4. They can become projections over the monthly entity, but stay separate models. |
| `mrt_lcsp_all_employer_report` | Employee × plan year on its own population and plan-start rules (C4, C6). It can join M5's entity only after those decisions. |
| `mrt_carrier_and_plan_id_reference`, `mrt_lowest_cost_plan`, `mrt_issuers_and_plan_counts` | Three different grains (plan; ZIP × metal × age; ZIP). Architecture §3.3 lists them as three entities. |
| `mrt_active_covered_employees_current_month`, `mrt_ajg_current_month_coverage_active_employees`, `mrt_count_of_active_covered_employees_current_month`, `mrt_daily_enrollment_status_snapshot` | Incremental snapshots that **store history** (`full_refresh=false`, seeded from CSVs), so merging them means migrating stored months. Logically, AJG is the covered list at insured level plus a producer filter (`pr.name like '%AJG%'`), so it could merge with Active Covered after a history migration. The Count report also includes two override employers (C12). |
| `mrt_payment_report` | Its own grain (expected payment / unexpected transaction) |
| `mrt_employee_employer_ytd_contribution` | Every enrollment overlapping the plan year, not only the latest: enrollment × plan year |
| `mrt_main_enrollment_report` | Insured person × enrollment; architecture §2.1 gives this fan-out its own insured-level mart |

---

## Checklist for every Bucket 1 / Bucket 2 merge

1. Build the current version into a separate schema.
2. Make the change and build again.
3. Compare each affected report row by row: same row count, and no row present in only one side. The report-to-model mapping is in each section above.
4. Move the key tests (`unique`, `not_null`) from the removed model to the model that absorbs it.
5. For Bucket 2: run the data check first and merge only if it returns 0 rows.

## Bucket 3 decisions, in suggested order

| # | Decision | Unblocks |
|---|---|---|
| 1 | Which enrollment represents an employee-month (C7) | C8 (−3 models), C9, mart merge M4 (−1 mart) |
| 2 | Does eligibility stop at the OEP end? (C5) | C5 (−1), C6 |
| 3 | One plan start rule (C6) | C6 (−1), C8 |
| 4 | Which attributes choose the benefit class (C10) | C10 (−1) |
| 5 | Which statuses mean "enrolled" (C1), and which employer-years are listed (C2) | Consistent numbers between the two employer reports; mart merge M3 (−1 mart) |
| 6 | One "latest enrollment" rule, or named variants (C14) | Mart merge M5 (−2 marts) |

