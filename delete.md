## Summary
Consolidates the intermediate layer and makes the LCSP change-log picks deterministic.

**Removed models**: each was a duplicate or a small helper with one reader, and its SQL now sits in that reader:
- `int_plan_pricing_zip_majority_fips` → `int_majority_fips_by_zip` (exact duplicate)
- `int_lcsp_employee_plan_start` → `int_lcsp_age_calculations` (new `actual_plan_start_date` column)
- `int_fips_county_map` → CTE in `int_compliance_employee`
- `int_employer_aor_changed_by_year` → grouped from `int_employer_plan_year_spine`
- `int_enrollment_team` → `int_employer_master.enrollment_team_list`
- `int_payment_expected_amount_latest`, `int_employer_allowance_model_by_year`, `int_latest_application_sent` → CTEs in the model that reads each one
- `int_medical_plan_level` → `int_employee_enrollment_enriched` joins `stg_medical_plans` directly

**Other changes**
- `int_employer_master` gains `naics_industry`.
- New grain test on `int_employer_plan_year_spine` (`employer_id`, `plan_year`).
- `stg_employee_change_log` exposes `change_log_id`, tested `not_null` and `unique`.
- 5 LCSP change-log rankings get a final tie-breaker: when two rows share a timestamp, the later `created_at` wins, then the highest `change_log_id`.
- `int_compliance_employee` calculates `cont_*` in the final select, so it always uses the `lcsp` and `alw` on the same row.

## Output impact
- **Removing the models changes no output:** the SQL moves without any change to what it selects or filters.
- **The tie-break affects only employees with change-log rows sharing a timestamp**, in `mrt_compliance_lcsp_report`, `mrt_monthly_lcsp_report`, `mrt_insured_coverage_months` and `mrt_lcsp_all_employer_report`. Their values could already change between runs; they now always resolve to the same row.
- **Everyone else is unaffected.**

## How to test
- [ ] `dbt build` on the affected marts.
- [ ] Gold vs PR `EXCEPT` diff in both directions. Every difference should belong to an employee with tied change-log rows.
- [ ] The new spine grain test and the `change_log_id` tests pass.

## Decision needed
- Business to confirm the tie rule: when two change-log entries share a timestamp, the later-entered one wins.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
