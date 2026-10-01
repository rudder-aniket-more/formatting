# Grain Consolidation — Report-by-Report Impact

**Status:** for report-owner review before sign-off · October 2026
**For:** the owner of each of the five affected reports, deciding whether to approve its switch to v2
**Related:** [`dbt-grain-consolidation-change-plan.md`](dbt-grain-consolidation-change-plan.md) (what changes and the plan) · [`dbt-grain-consolidation.md`](dbt-grain-consolidation.md) (design; the D1–D10 table)

Each section below covers one report:

1. **Earlier logic:** how the report works today (v1).
2. **New logic:** how v2 computes the same report.
3. **What changes if we switch:** which rows and values will differ, with examples.
4. **What stays the same.**
5. **How to check:** what to look at in the comparison (parity) query.

The column names, column order and table name of every report stay the same. Only values change, and only where noted. Everything under "What changes" is **expected from reading the code**. The parity run (change plan, Phase 1) confirms it on real data, and any difference not listed here is treated as a bug.

---

## Summary

| Report | Rows change? | Values change? | Size of change | Items |
|---|---|---|---|---|
| PCORI | No | No (except rare duplicate-record cases) | None expected | D10 |
| Insured Coverage Months | No | Rarely | Very small | D2, D10 |
| Monthly LCSP | No | Yes, in specific months | Small | D3, D8, D9, D10 |
| All-Employer LCSP | Yes, some employees added or removed | Yes | Moderate | D7, D9, D10 |
| Compliance LCSP | No | Yes, in several monthly blocks | Largest | D1, D4, D5, D6, D10 |

---

## Terms used

| Term | Meaning |
|---|---|
| **OEP** | The employer's open enrollment period for a plan year (start and end date) |
| **OP** | An employee's onboarding period: one enrollment attempt inside an OEP. An employee can have several in a year |
| **Coverage status** | The OP's major-medical benefit is Carrier Application Sent, Awaiting Payment, Enrollment Confirmed, Active or Ended |
| **Eligibility window** | A from/until date range the employee is ICHRA-eligible. Comes from the employee record or the eligibility history; re-hires have several |
| **3-tier address** | The employee's address on their enrollment (insured record) → else the latest employee change-log entry → else the employee record |
| **Rating age** | Age at plan start capped to 0–64, the age plan prices are filed at |
| **LCSP** | The cheapest on-market silver plan for the employee's ZIP + county at their rating age |
| **Masked employer** | MyZorro Inc (US Entity), whose personal data is blanked in reports |

---

## 1. PCORI Report (`mrt_pcori_report`)

One row per person (employee or covered-benefit dependent) per plan year, with a 0/1 coverage flag for each of the 12 months. Used for the PCORI fee filing.

### Earlier logic (v1)
1. **Population:** every employee of a non-demo, non-prospect employer that has an OEP from 2025 on, if any of the employee's eligibility windows overlaps that calendar year.
2. **Employee coverage per month:** covered (1) if **any** of the employee's OPs in that OEP has closed enrollment, a coverage-status benefit, and a date range that includes the month. Months not reached yet show 0.
3. **Dependents:** every non-employee insured on any of those OPs who is linked to a benefit. Covered in a month if that OP is covered that month **and** the benefit's family unit includes them (Family → everyone; Employee + Spouse → spouse; Employee + Child → children).
4. **People, not records:** the same dependent on two OPs is merged into one row by grouping on employee, name and relationship.
5. **Masking:** names are blanked for the masked employer *after* grouping, so blanked names cannot merge two people.

### New logic (v2)
- Employee months come from `fct_employee_months`. It uses the same population, and its `is_covered` is the same any-OP rule.
- Dependents come from `fct_dependent_coverage`. It stores each dependent's covered date range per OP plus the family-unit flag, and the month check is the shared `is_covered_in_month` macro.
- People are merged by `dependent_person_id`, an ID built from the same employee + name + relationship grouping before any masking. The grouping is identical to v1, and masked rows cannot collapse.

### What changes if we switch
- **Expected: nothing.** Row counts and every 0/1 flag should match v1.
- **Only exception (D10):** an OP with **two** major-medical benefit records. v1 checked both; v2 keeps one, preferring the one in coverage status. A dependent's flag could differ only if the two records have different family units. This should be very rare.

### What stays the same
Population, coverage rule, family-unit rule, person merging, masking, columns.

### How to check
Parity should show `only_in_v1 = 0`, `only_in_v2 = 0` and zero mismatches in every `Count - <month>` column. Separately, confirm the masked employer's row count per plan year equals v1's. **This is the recommended first cutover.**

---

## 2. Insured Coverage Months (`mrt_insured_coverage_months`)

One row per insured member per month: an employee row for each eligible employee-month, plus a row for each benefit-linked dependent on that month's enrollment. Basis of the Bernie & Phyl's monthly LCSP Metabase question.

### Earlier logic (v1)
1. **Population:** employee-months where the month falls inside the employer's OEP **and** the employee is eligible on the 1st of that month (record window or eligibility history).
2. **Enrollment for the month:** among the employee's OPs covering the month, pick: coverage-status first, then non-waived, then most recently updated.
3. **Address:** 3-tier, using the insured record of that chosen OP.
4. **Age:** age at the first eligible month of the plan year (else at OEP start). The LCSP uses the rating age.
5. **Class and allowance:** the employee's latest change-log attributes as of the month (else the employee record) are scored against the employer's class definitions. The EMPLOYEE_ONLY stipend for the best-matching class and the employee's age band is the allowance.
6. **LCSP:** cheapest silver plan for the ZIP + county at the rating age; blank if none.
7. **Dependents:** non-employee insureds on the chosen OP who are linked to a benefit, inheriting the employee's age, address and LCSP.

### New logic (v2)
This report's rules were chosen as **the canonical rules** for all five reports, so v2 applies the same steps. They are now taken from the shared models: `fct_employee_months` filtered to eligible months, joined to `fct_dependent_coverage` on the month's chosen OP.

### What changes if we switch
- **D2, only if it occurs:** an OEP that crosses 31 December (e.g. Jul 2025 – Jun 2026). v1 attributed Jan–Jun 2026 to plan year 2025. v2 attributes months to their calendar year. A warning test reports whether any such OEP exists; production data has shown none so far.
- **D10:** an OP with two employee insured records holding **different** addresses previously could produce two rows for the same month. v2 keeps one (the first by record id).
- Otherwise no change is expected.

### What stays the same
Everything else, including the 4-digit employee id, relationship codes, monthly allowance on the employee row only, and masking.

### How to check
Parity should show no key differences and no mismatches. The masked employer is excluded from the key match (its names are blank), so compare its row counts per plan year separately.

---

## 3. Monthly LCSP Report (`mrt_monthly_lcsp_report`)

One row per eligible employee per month, with address, age, class and that month's LCSP and premium.

### Earlier logic (v1)
1. **Population:** same as Insured Coverage Months, step 1.
2. **Enrollment for the month:** a **different** rule from the other reports. Only OPs that are in coverage status **or waived** are considered, ranked non-waived first, then most recently updated. An employee whose only OP in the month is neither (e.g. an abandoned or in-progress enrollment) gets **no** enrollment: coverage dates are blank.
3. **Address:** 3-tier, using the insured record of that OP.
4. **Age and LCSP:** same as Insured Coverage Months.
5. **Name:** `first_name + ' ' + last_name`. Blank if either part is missing.
6. **Masking (masked employer):** name, class, age, address, ZIP, FIPS, state and premium blanked; **eligibility dates visible**.

### New logic (v2)
`fct_employee_months` filtered to eligible months. It uses the canonical enrollment rule (Insured Coverage Months, step 2), the 3-tier address on that OP, and the shared LCSP ranking.

### What changes if we switch
| Item | Situation | v1 shows | v2 shows |
|---|---|---|---|
| **D3** | Employee's only OP that month is not in coverage status and not waived | Blank `effective_from/until` | That OP's dates |
| **D3** | Employee has a waived OP **and** an abandoned non-waived OP that month, nothing in coverage status | The waived OP's dates | The abandoned OP's dates |
| **D3, knock-on** | In the two cases above, the insured address may come from a different OP | Address of the v1 OP (or change log / employee record) | Address of the v2 OP |
| **D3, knock-on** | If that address has a different ZIP or county | LCSP for the old ZIP | LCSP for the new ZIP |
| **D9** | Employee with a missing first or last name | Blank `employee_name` | e.g. `Smith` |
| **D8** | Masked employer only | Eligibility dates visible | Eligibility dates blanked |
| **D10** | Duplicate employee insured records on one OP | Possible duplicate rows | One row |

**Rows:** no change expected. Which employee-months appear is decided by eligibility, and that rule is unchanged.

**Practical effect:** months where an employee had no active coverage now show the dates of whatever enrollment they had, instead of a blank. Months with active coverage are unaffected. `has_active_coverage` in Insured Coverage Months is the way to tell the two apart.

### What stays the same
Population, eligibility, age, LCSP ranking, all other masking, columns.

### How to check
Parity mismatches are expected only in `effective_from`, `effective_until`, `eligible_from/until` (masked employer, excluded from parity), and occasionally `address`, `zip_code`, `fips_code`, `lcsp_id` and `lcsp_premium` in the same months. Spot-check a few mismatched rows: each should be a month without coverage-status enrollment.

---

## 4. All-Employer LCSP Report (`mrt_lcsp_all_employer_report`)

One row per employee per plan year, with age, address and the LCSP at the employee's plan start.

### Earlier logic (v1)
1. **OEP:** the earliest OEP per employer per year (in practice there is only one).
2. **Population:** employee whose record window **or** any history window overlaps the period from OEP start to 31 Dec.
3. **Plan start:** a single "eligible from" date is used: the employee record's, or if blank, the **latest** eligible-from in the history. Plan start = the first 1st-of-month on or after the later of that date and the OEP start. Employees whose plan start falls after 31 Dec are dropped. The window's **end** date is not checked.
4. **Enrollment for the address:** one OP for the **whole year**, among OPs in coverage status or waived: coverage-status first, then non-waived, then most recently updated (no final tie-break, so ties are unpredictable).
5. **Address:** 3-tier, using that year-level OP's insured record, then the change log as of plan start, then the employee record.
6. **Age:** age at plan start → rating age → LCSP at the plan-start address.
7. **Name:** `first + ' ' + last`, blank if either part is missing.

### New logic (v2)
`fct_employee_plan_years`, filtered to employees with a plan start in the year.
- **Plan start** = the first 1st-of-month that falls inside **an eligibility window (any of them) and the OEP**, within the year.
- **Address and LCSP** are those of the employee's **plan-start month**, read from the same monthly models as Monthly LCSP. An employee's annual LCSP therefore always equals their first month in the Monthly LCSP report.

### What changes if we switch
**Rows (D7):**

| Situation | v1 | v2 |
|---|---|---|
| Re-hire whose current record window starts next year but who had a history window this year | Dropped (plan start after 31 Dec) | **Included**, plan start from the earlier window |
| Eligibility window that contains no 1st-of-month inside the OEP (e.g. eligible 15–25 Mar) | Included with plan start 1 Apr, although not eligible then | **Removed** |
| Window that ended before its computed plan start (e.g. eligible until 10 Jun, plan start computed as 1 Jul) | Included | **Removed** if no earlier month qualifies |

**Values:**

| Item | Situation | Effect |
|---|---|---|
| **D7, plan start** | Re-hire with blank record "eligible from": v1 used the **latest** history date | v2 uses the first eligible month in the year → earlier plan start → age may differ by one → **rating age and LCSP premium may change** |
| **D7, address** | Coverage starts later than plan start (e.g. plan start Mar, enrollment from Apr) | v1 used the insured address of the year's enrollment. v2 has no enrollment in March, so it uses the change log, then the employee record. Usually the same address; if not, ZIP and LCSP change |
| **D7, address** | Several enrollments in the year with different addresses | v1 picked one for the year, sometimes arbitrarily. v2 uses the one covering the plan-start month |
| **D9** | Missing first or last name | Name now shows |
| **D10** | Duplicate employee insured records | One row |

### What stays the same
Start year (2025, same variable), demo/prospect exclusion, LCSP ranking rule, masking, columns.

### How to check
Expect a small number of `only_in_v1` / `only_in_v2` rows; each should be a re-hire or short-eligibility-window employee. Mismatches are expected mostly in `age`, `lcsp_premium` and occasionally in the address columns and `lcsp_id`. Sample a few and confirm the v2 plan start against the employee's eligibility history.

---

## 5. Compliance LCSP Report (`mrt_compliance_lcsp_report`)

One row per employee per plan year (plus one per dependent), with 12-month blocks: Employment, Eligibility, ZIP, County, FIPS, LCSP, Allowance, EE-only Contribution, Coverage, and Actual Premium / ER / EE Contributions. Used for ACA affordability compliance. 191 columns, 160 masked for the masked employer.

### Earlier logic (v1)
1. **Population:** as PCORI step 1. Dependents: every non-employee insured on any of the employee's OPs that year, covered or not.
2. **Header:** name, SSN, current address, state, ZIP, email and phone from the employee record. Age on 1 Jan.
   - **Age on plan start** = age at the first eligible month already reached this year, else **at 1 Jan**.
   - **Plan start month employee** = that month, else "Jan".
3. **Employment (12 months):** "Yes" if hired on or before the 1st and not terminated before it.
4. **Eligibility (elapsed months):** "No" before the OEP start. "Yes" if the record window or any history window covers the 1st. **The OEP end date is not checked**, so months after an employer's OEP ends can still show "Yes".
5. **ZIP / FIPS / County (eligible months):** insured address from **any** OP in coverage status that covers the month (if several match, one is picked arbitrarily), else the latest change-log entry, else the employee record.
6. **LCSP (eligible months):** cheapest silver price for that ZIP + county at the rating age.
7. **Allowance (eligible months):** class resolution using change-log attributes. For each **past month**, the last change of that month is kept, and **all of them compete**: the best-scoring class across the employee's whole history wins, even if those attributes are out of date. Stipend for that class, EMPLOYEE_ONLY, in the band for age on plan start.
8. **EE-only Contribution:** max(0, LCSP − Allowance).
9. **Coverage (elapsed months):** YES/NO, using the PCORI any-OP rule.
10. **Actual Premium / ER / EE contribution:** highest value among the month's covered OPs.
11. **Covered individuals:** the employee's coverage on employee rows. Dependent rows show YES/NO by family unit in months their OP is covered, blank otherwise.

### New logic (v2)
The same 191 columns and the same masking. Values now come from the shared models:
- **Header:** `fct_employee_plan_years`.
- **Monthly blocks:** `fct_employee_months`, pivoted.
- **Dependents:** `fct_dependent_coverage`.
- **Employment:** still computed for all 12 months from hire and termination dates, as before.

### What changes if we switch
| Item | Situation | v1 shows | v2 shows | Blocks affected |
|---|---|---|---|---|
| **D1** | Employer's OEP ends mid-year (e.g. 30 Sep) | Oct–Dec Eligibility "Yes" and values in the dependent blocks | Oct–Dec Eligibility **"No"**; ZIP, County, FIPS, LCSP, Allowance and Contribution **blank** | Eligibility, ZIP, County, FIPS, LCSP, Allowance, EE-only Contribution |
| **D4** | Several covered OPs in a month with different insured addresses | One picked arbitrarily | The month's chosen OP (canonical rule) | ZIP, County, FIPS → LCSP, Contribution |
| **D4** | No covered OP that month, but an abandoned or waived OP exists | Change log / employee record | The insured address on that OP, if it has one | ZIP, County, FIPS → LCSP, Contribution |
| **D5** | Employee whose attributes changed (moved state, full-time → part-time, etc.) | The class that best matches **any** past set of attributes | The class matching the attributes **current as of that month** | Allowance → EE-only Contribution |
| **D6** | Employee never eligible this year | Age on plan start at **1 Jan**; plan start month "Jan" | Age at the **company plan start** (OEP start); month "Jan" unchanged | Age on plan start; Allowance (age band) |
| **D6** | Current year: employee becomes eligible in a month not reached yet | Age at 1 Jan; month "Jan" | Age and month of the **future** plan start | Age on plan start, Plan start month employee, Allowance |
| **D1 → D6** | Employee eligible only after the OEP ended | Plan start in that month | No plan start (falls back as above) | Header ages |
| **D10** | Duplicate benefit or insured records | Possible arbitrary values | One record kept deterministically | Any |

**Rows:** no change expected. Population and dependent listing are unchanged.

**Practical effect:** the affordability numbers (LCSP, Allowance, Contribution) change for employees who changed class attributes or ZIP during the year, and for employers whose OEP ended early. For an ACA filing the v2 values are the more defensible ones: they use the attributes and address in force in each month, and nothing after the plan ended.

### What stays the same
Population, Employment, Coverage, Actual Premium / ER / EE, dependent coverage rules, masking of all 160 columns, the 191-column layout.

### How to check
- Mismatches are expected in the Eligibility / ZIP / FIPS / County / LCSP / Allowance / EE-only Contribution month columns and in the two age columns.
- **Employment, Employee Coverage, Actual Premium and Covered individuals should show zero mismatches.** Any difference there is a defect.
- Sample mismatches: each should be an OEP-end case (D1), a mid-year attribute or ZIP change (D4/D5), or a never-eligible / future-start employee (D6).

---

## What happens overall if we make these changes

**Benefits**
- The same employee gets the same eligibility, plan start, age, address and LCSP in every report.
- Business rules are maintained in one place: about 31 intermediate models are replaced by 11.
- A fix to a rule reaches every report at once.
- Each fact's grain is checked by tests, so duplicate or dropped rows fail loudly instead of silently.
- New reports at these grains need only a Metabase question over a fact, not a new pipeline.

**Costs and risks**
- Values change in Compliance, All-Employer LCSP and, to a small degree, Monthly LCSP, as listed above. Anyone comparing against previously exported files will see differences.
- A rule picked as canonical could be wrong for one report. Owner sign-off per report exists to catch that.
- The reports are still views, so query cost is similar to today. A fact can be promoted to a table if a dashboard gets slow.
- Rollback is a one-line change for 30 days after each cutover (change plan §4).
