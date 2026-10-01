# Grain Consolidation — What Changes and the Rollout Plan

**Status:** ready for review · October 2026
**For:** the data team lead approving the change, and the owners of the five affected reports
**Design detail:** [`dbt-grain-consolidation.md`](dbt-grain-consolidation.md) (lineage, report matrix, rule decisions D1–D10)

---

## 1. In one paragraph

Five reports (Monthly LCSP, All-Employer LCSP, Insured Coverage Months, PCORI and Compliance LCSP) currently compute the same business rules four separate times, in four chains of 31 intermediate models. The chains have drifted apart: the same employee can get a different address, age or LCSP depending on which report you open. This change computes each rule **once** and stores the results in **three shared fact tables**, one per data grain. Each report is then rebuilt as a short view on top of those facts. The new versions run **alongside** the current reports. Nobody sees a change until each report owner has compared old and new and signed off.

---

## 2. What changes

### 2.1 Now, in this change — added, nothing replaced

| Area | Added | Purpose |
|---|---|---|
| Shared business rules | 11 models in `dbt/models/intermediate/logic/employee_coverage/` | Eligibility, plan start and age, 3-tier address, benefit class, allowance, LCSP ranking and coverage selection, each defined once |
| Fact tables | `fct_employee_months`, `fct_employee_plan_years`, `fct_dependent_coverage` in `dbt/models/marts/core/` | One table per grain: employee × month, employee × plan year, dependent × onboarding period. PII masked at the strictest level any report uses |
| New report versions | `mrt_<report>_v2.sql` for each of the five reports | Same columns, in the same order, as today's report, built on the facts |
| Comparison queries | 5 files in `dbt/analyses/grain_consolidation/` + the `version_parity` macro | Old vs new, row by row and column by column |
| Shared macro | `is_covered_in_month` | One definition of "a dependent was covered in month m" |
| Tests & docs | Grain/not-null tests on every new model, 2 unit tests, full column docs | Duplicates or dropped rows fail loudly |

**Existing files touched (157 lines added, 4 changed):**

| File | Change | Effect on today's reports |
|---|---|---|
| `models/marts/_mrt_models.yml` | Adds a `versions:` block (v1 = today, v2 = new) to the five reports | **None.** v1 keeps its table name; `ref()` and Metabase still read v1 |
| `models/intermediate/_int_unit_tests.yml` | Two new unit tests appended | None |
| `macros/_macros.yml` | Docs for the two new macros | None |
| `macros/drop_orphaned_relations.sql` | Recognises the `fct_` prefix | Lets the cleanup job remove deleted `fct_` tables. Still a dry run by default |

**Not changed:** any existing SQL model, any of the other 18 marts, Metabase, schemas, materializations, Airflow / Glue.

### 2.2 Later, at each report's cutover — what report users will notice

Columns, column order and table names stay the same. Some **values** change where the old chains disagreed and one rule had to win:

| Report | What may change for users | Ref |
|---|---|---|
| **Compliance LCSP** | Months after an employer's open-enrollment end date show "No" eligibility (and no LCSP/allowance). ZIP/county comes from the employee's selected enrollment, not an arbitrary one. Allowance uses the employee's current class attributes. "Age on plan start" for never-eligible employees is taken at the company plan start instead of 1 Jan | D1, D4, D5, D6 |
| **Monthly LCSP** | Coverage start/end now also shows enrollments that were neither active nor waived. Names no longer go blank when a first or last name is missing. For the masked employer, eligibility dates are masked | D3, D8, D9 |
| **All-Employer LCSP** | Re-hired employees get the correct plan start, and therefore the correct age and LCSP. Address is taken at the plan-start month | D7, D9 |
| **Insured Coverage Months** | Only if an employer's open enrollment crosses a year end (a test will tell us) | D2 |
| **PCORI** | No intended change. Counts should match exactly | — |
| All five | Rare duplicate rows caused by duplicate source records disappear | D10 |

After cutover, the retired chain is deleted: roughly **31 intermediate models and their YAML**, replaced by 11.

---

## 3. The plan

| Phase | What happens | Owner | Done when | Consumer impact |
|---|---|---|---|---|
| **0. Review & merge** | Code review; `dbt build` in the PR schema builds v1 and v2 and runs all tests on both | Data engineering | PR merged, all tests green | None |
| **1. Parity** | Run the comparison query for each report across all plan years; explain every difference against D1–D10 or fix v2 | Data engineering | Every difference is explained or fixed | None |
| **2. Sign-off** | Each report owner reviews the differences for their report (§2.2) | Report owners | Written approval per report | None |
| **3. Cutover** | One small PR per report switches the report's table name to v2; v1 remains for 30 days as `<report>_v1` | Data engineering | Metabase questions on the report confirmed working | **Values change as in §2.2** |
| **4. Clean-up** | After 30 days: delete v1 and the old chain, remove their YAML, run orphan cleanup, update lineage docs | Data engineering | Old models gone, docs updated | None |

**Suggested cutover order:** PCORI first, then Monthly LCSP, Insured Coverage Months, All-Employer LCSP, and Compliance last. That runs from fewest expected differences to most.

---

## 4. Rollback

Before cutover, nothing to roll back: v2 only exists alongside v1. After cutover, the report is reverted by flipping `latest_version` and the table alias back to v1 in `_mrt_models.yml`. That is a one-line PR, valid for the full 30-day window while v1 is kept.

---

## 5. Risks and open decisions

| Item | Risk / decision | Who decides |
|---|---|---|
| Not yet run on real data | The change compiles and every SQL file parses, but there were no warehouse credentials in the build session. Phase 0 is the first real run | — |
| Value changes D1–D10 | A rule picked as canonical may be wrong for one report; sign-off exists to catch that | Report owners |
| Fact naming | `fct_*` (as requested) vs the un-prefixed names proposed in `data-analytics-architecture.md` §3.3. Must be settled before the facts are exposed in Metabase, since renaming later breaks questions | Data team lead |
| Onboarding-period id | Visible in the facts as a join key; the ops reports mask it | Owner of the masking list |
| Query cost | The facts are views, so each report query recomputes the chain. Promote a fact to a table if a dashboard is slow (architecture §5.2) | Data engineering |
| Scope | The six enrollment reports and the two age-transition reports are a separate phase 2 (design doc §3.2) | Data team lead |
