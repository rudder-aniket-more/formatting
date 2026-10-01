# Grain Consolidation — Employee Coverage Facts

**Status:** built side-by-side, awaiting parity sign-off · October 2026
**Audience:** anyone reviewing, signing off or cutting over the LCSP / compliance / PCORI reports
**Companion docs:** [`data-analytics-architecture.md`](data-analytics-architecture.md) (target layers, the dbt/Metabase boundary) · [`dbt-best-practices.md`](dbt-best-practices.md)

---

## 1. Summary

Five reports — monthly LCSP, all-employer LCSP, insured coverage months, PCORI and compliance — were fed by **four parallel `int_lcsp_*` chains** (31 intermediate models). Each chain re-implemented the same five rules with its own variations. These are the rules every one of the reports depends on:

| Rule | Copies before | Single definition now |
|---|---|---|
| Eligibility on a date (record window ∪ history) | 4 | `int_employee_eligibility_windows` → `int_employee_months.is_eligible` |
| Plan start + age at plan start | 3 | `int_employee_plan_years` |
| 3-tier address fallback (insured → change log → employee) | 5 | `int_employee_addresses_resolved_monthly` |
| Benefit-class resolution | 2 | `int_employee_classes_resolved_monthly` |
| LCSP ranking (ZIP/FIPS + rating age, cheapest silver) | 4 | `int_employee_lcsp_ranked_monthly` |
| Onboarding-period selection / coverage | 3 | `int_employee_coverage_resolved_monthly` |

This change adds a conformed layer: **11 intermediate models** in `intermediate/logic/employee_coverage/` and **3 facts**, one per grain, in `marts/core/`. Each of the five reports has a **v2**: a thin projection of the facts, with the same columns in the same order as v1 (verified mechanically). **Nothing existing changes behaviour.** v1 keeps its relation name, `ref()` still resolves to v1, and Metabase is unaffected until each report is signed off.

---

## 2. Target lineage

```
staging (stg_*)
   │
   ├─ int_plan_year_months ─────────────┐   month spine, uncapped (is_elapsed_month)
   ├─ int_employer_plan_years ──────────┤   population gate: 1 OEP per employer-year
   ├─ int_employee_eligibility_windows ─┤   record + history windows
   │                                    ▼
   │                     int_employee_plan_years          (employee_id, plan_year)
   │                        plan start · ages · population
   │                                    ▼
   │                     int_employee_months              (employee_id, determination_date)
   │                        is_eligible · is_employed · is_within_oep
   │            ┌───────────────┬───────┴────────┬──────────────────────┐
   │            ▼               ▼                ▼                      ▼
   │  int_employee_coverage  int_employee_    int_employee_classes   int_dependent_
   │  _resolved_monthly      addresses_       _resolved_monthly      onboarding_periods
   │  (OP selection,         resolved_        (class scoring)        (insured_id, period_id)
   │   is_covered)           monthly               ▼
   │            └──────────▶ (3-tier)         int_employee_allowances_monthly
   │                            ▼
   │                  int_employee_lcsp_ranked_monthly
   │                            ▼
   ├──────────▶ fct_employee_months   fct_employee_plan_years   fct_dependent_coverage
   │                      │                    │                         │
   └──────────▶ mrt_monthly_lcsp_report v2 ────┤                         │
                mrt_insured_coverage_months v2 ┼─────────────────────────┤
                mrt_lcsp_all_employer_report v2┤                         │
                mrt_pcori_report v2            ┼── pivots (→ exports/) ──┤
                mrt_compliance_lcsp_report v2 ─┘                         │
```

Each fact assembles its columns with LEFT JOINs on its own full key, so an enrichment can neither fan a row out nor drop it. Every intermediate model's output grain is asserted with `unique_combination_of_columns`.

---

## 3. Report → model matrix

### 3.1 Consolidated in this change

| Report (v1) | Grain | v1 chain (retired at cutover) | v2 reads | Shape |
|---|---|---|---|---|
| `mrt_monthly_lcsp_report` | employee × month | `int_lcsp_eligible_employees` → `_covered_data`, `_ecl_address`, `_plan_start` → `_employee_resolved` → `_monthly_ranked` | `fct_employee_months` where `is_eligible` | mart |
| `mrt_insured_coverage_months` | member × month | `int_lcsp_member_coverage`, `_attributes`, `_resolved`, `_ranked`, `_allowance`, `_final` | `fct_employee_months` + `fct_dependent_coverage` on `onboarding_period_id` | mart |
| `mrt_lcsp_all_employer_report` | employee × plan_year | `int_lcsp_all_employees_eligible` → `_all_employees_ranked` | `fct_employee_plan_years` where `is_eligible_in_plan_year` | mart |
| `mrt_pcori_report` | person × plan_year | `int_lcsp_base_employees`, `int_employee_benefit_coverage_monthly` + inline dependents | `fct_employee_months` + `fct_dependent_coverage` (by `dependent_person_id`) | export (pivot) |
| `mrt_compliance_lcsp_report` | person × plan_year | `int_compliance_employee`, `_dependent_coverage`, `int_lcsp_monthly_*`, `_age_calculations`, `_employee_plan_start` | all three facts | export (pivot) |

### 3.2 Not consolidated — phase 2 candidates

| Report | Grain | Why not now | Phase 2 target |
|---|---|---|---|
| `mrt_main_enrollment_report`, `mrt_employee_level_premium`, `mrt_employee_enrollment_status_names_invoices`, `mrt_ops_and_actual_premium`, `mrt_ops_and_projected_premium`, `mrt_employee_employer_ytd_contribution` | employee × plan_year | Six "latest enrollment" rankings with **different** partitions (OEP year vs `coverage_year`), gates (`is_real_employer` vs `is_not_demo`, Medicare exclusion) and the deliberate 2025 in-flight exception. Converging them changes row counts, so it needs its own sign-off. | `employee_enrollments` (architecture §3.3) at employee × plan_year, extending `fct_employee_plan_years` |
| `mrt_dependent_age_transition_26`, `mrt_medicare_dependent_age_transition_65` | insured person, current state | Not plan-year scoped, includes EMPLOYEE insureds, and uses the raw `signup_status` (sign-off pending, see `int_insured_age_transition_base`). | `insured_coverage` (architecture §3.3) |

### 3.3 Retained as-is — already one mart per grain

`mrt_employer_level_details`, `mrt_employer_enrollment_participation` (employer × plan_year) · `mrt_payment_report` (expectation / transaction) · `mrt_daily_enrollment_status_snapshot`, `mrt_*_current_month` (incremental snapshots) · `mrt_lowest_cost_plan`, `mrt_issuers_and_plan_counts`, `mrt_carrier_and_plan_id_reference` (plan / ZIP grains). Each already projects a single intermediate chain at its own grain. Consolidating them would mean renaming, not removing redundancy.

---

## 4. Rule decisions — expected v1 → v2 differences

Picking one definition per rule means the reports that used a different variant will change. **These are the differences parity should show. Any other difference is a defect in v2.** Each one needs sign-off from the report owner before that report cuts over.

| # | Rule | v1 behaviour | v2 (canonical) | Reports affected |
|---|---|---|---|---|
| D1 | Eligibility vs OEP end | Compliance/PCORI ignored `oep_effective_until`; monthly LCSP respected it | Eligible only inside the OEP (mid-year cutoff respected) | compliance (Eligibility, ZIP/FIPS, LCSP, Allowance, Contribution after an OEP end) |
| D2 | Month → plan year | Monthly LCSP gave months after Dec 31 to the OEP's start year | Calendar plan years. A warn-level test flags any OEP that crosses a year end | monthly LCSP, insured months (only if such an OEP exists) |
| D3 | Onboarding-period selection | Monthly LCSP kept only active-or-waived OPs and ranked non-waived first | Every OP in the month; active coverage first, then non-waived, then latest update | monthly LCSP `effective_from/until`, and the insured-address tier behind it |
| D4 | Compliance address | Insured address from *any* covering active OP (arbitrary pick when there were several) | Insured address from the month's selected OP | compliance ZIP/FIPS/County |
| D5 | Class resolution (compliance allowance) | Every past month's change-log attributes competed in scoring | Latest attributes as of the month only | compliance Allowance, EE Only Contribution |
| D6 | Age fallback | Compliance fell back to Jan 1 when the employee had no eligible month | Falls back to the OEP start (monthly/member rule) | compliance "age on plan start", "Plan start month employee" |
| D7 | All-employer plan start | One `resolved_eligible_from`, so re-hires used the wrong window; address from a year-level OP pick | First eligible month across **all** windows; address and LCSP of the plan-start month | all-employer LCSP |
| D8 | Masking | `eligible_from/until` visible in the monthly LCSP report | Masked [c], as in insured months (strictest level wins) | monthly LCSP (masked employer only) |
| D9 | `employee_name` | `concat()` — null when either part is null | `concat_ws()` — null-safe | monthly LCSP, all-employer LCSP |
| D10 | Fan-out guards | Duplicate EMPLOYEE insured rows or duplicate MAJOR_MEDICAL benefits per OP could duplicate rows or pick arbitrarily | One row kept deterministically (by `insured_id`; by coverage status, then `created_at`) | all five, rare |

**Also note:** `onboarding_period_id` is visible in `fct_employee_months` as a join key, the same treatment as `employee_id`. The ops marts mask `employee_op_id` as [f]. Confirm this with whoever owns the masking list before exposing the facts in Metabase.

---

## 5. Migration and deprecation checklist

### Phase 0 — Merge (this change; no consumer impact)
- [ ] `dbt build --select +fct_employee_months +fct_employee_plan_years +fct_dependent_coverage` in `pr_<n>`: models, grain tests and both new unit tests green.
- [ ] `dbt build --select mrt_monthly_lcsp_report mrt_lcsp_all_employer_report mrt_insured_coverage_months mrt_pcori_report mrt_compliance_lcsp_report` builds v1 **and** v2. Every v1 data test also runs against v2.
- [ ] Review the `oep_within_calendar_year_int_employer_plan_years` warning. If it fires, D2 is live; resolve it before cutover.

### Phase 1 — Parity, per report
- [ ] Run `analyses/grain_consolidation/parity_<report>.sql` (`dbt compile`, then run the compiled SQL in Athena) for every plan year (`--vars '{test_years_back: 99}'` for the tests).
- [ ] `only_in_v1` / `only_in_v2` / `mismatch:` rows are each explained by D1–D10, or fixed in v2.
- [ ] Masked employer: row counts per plan year match v1 exactly (the parity keys exclude it).
- [ ] Report owner signs off the D-items that apply to their report.

### Phase 2 — Cutover, per report (one PR each)
- [ ] In `_mrt_models.yml`: set `latest_version: 2`; move the unsuffixed alias to v2 (`config: alias: <name>` under `v: 2`); give v1 alias `<name>_v1` and a `deprecation_date` 30 days out.
- [ ] Metabase: check the questions on the report still run. The relation name and columns are unchanged, so normally nothing needs editing.
- [ ] PCORI and compliance: per architecture §3.4, rename v2 into `models/exports/` (`export_pcori`, `export_compliance_lcsp`) when the exports folder is introduced. They are already pivot-only projections of the facts.

### Phase 3 — Deprecate (after the deprecation date)
- [ ] Delete the v1 SQL and the `versions:` block; rename `<name>_v2.sql` → `<name>.sql`.
- [ ] Delete the retired chain once nothing `ref()`s it (`dbt ls --select +<model>+`):
      `int_lcsp_base_employees`, `int_lcsp_eligible_employees`, `int_lcsp_generated_months`, `int_lcsp_plan_start`, `int_lcsp_employee_plan_start`, `int_lcsp_age_calculations`, `int_lcsp_monthly_eligibility_status`, `int_lcsp_monthly_locations`, `int_lcsp_ecl_address`, `int_lcsp_member_resolved`, `int_lcsp_all_employees_eligible`, `int_employee_benefit_coverage_monthly`, `int_insured_employee_address`, and all of `pipelines/lcsp/` and `pipelines/compliance/`.
- [ ] Remove their entries from `_int_models.yml`, and move `ut_monthly_pivot_dependent_coverage` in `_int_unit_tests.yml` onto a surviving `monthly_pivot()` consumer.
- [ ] `dbt run-operation drop_orphaned_relations` (dry run first). The prefix guard now includes `fct_`.
- [ ] Update `docs/dbt-lineage.md` and the routing in `AGENTS.md` / `.agents/architecture.md`.

---

## 6. Alignment with the architecture doc — open points

| Point | This change | Architecture doc | Decision needed |
|---|---|---|---|
| Fact naming | `fct_employee_months` etc. (as requested) | §3.3 proposes un-prefixed entities (`employee_enrollment_months`, `insured_coverage_months`) | Pick one before the facts reach Metabase. Renaming later breaks questions. |
| Schemas | No per-model `+schema`; the facts inherit `marts/`, the ints inherit `intermediate/` | §6.1 per-layer schemas, not yet implemented | None. The new models move with their folders when §6.1 lands. |
| Pivots | PCORI and compliance v2 pivot inside `marts/` | §3.4: pivots belong in `exports/` only | Covered by Phase 2. |
| Migration plan | — | `dbt-migration-plan.md` is referenced but not in the repo | Restore or drop the reference. |
