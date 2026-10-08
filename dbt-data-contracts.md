# Zorro Data Contracts — Rules for dbt Models

🌻 The testable rules extracted from the `zorro-ts` schema, organised the way
dbt consumes them: **grain**, **relationships**, **accepted values**, and the
**derived-status logic** warehouse models must replicate.

**Generated from `main` at commit `f9cd6df3a` (2026-10-08).** Companion to
[`schema-and-business-rules.md`](./schema-and-business-rules.md) (full column
reference).

## Why this file exists

Four of dbt's generic tests map directly onto schema facts:

| dbt test | Source in this file |
|---|---|
| `unique` / `dbt_utils.unique_combination_of_columns` | Part A — grain |
| `relationships` | Part B — foreign keys |
| `accepted_values` | Part C — enum columns |
| `not_null` | the column reference in the companion file |

Part D covers what dbt **cannot** get from the schema: derived statuses whose
logic lives in SQL functions and views. Recomputing those by hand is where
warehouse models most often silently diverge from the product.

---

# Part A0 — Join cardinality: where you can skip dedup

**The highest-value section for model authoring.** A unique constraint that
*covers the foreign-key columns* guarantees at most one child row per parent —
so the join is 1:1 and needs no `row_number()`, `qualify`, `distinct`, or
pre-aggregation. Where no such constraint exists, the join fans out and you must
aggregate or rank.

Derived mechanically: a relationship is 1:1 when some UNIQUE set on the child
table is a **subset** of its FK columns.

**Of 139 declared relationships: 32 are 1:1, 107 fan out.**

## How to use this

- **Safe-join list** → join directly, select child columns inline, no window
  functions. Asserting `unique` on the FK column in dbt documents the guarantee.
- **Fan-out list** → the parent grain is *not* preserved. Aggregate
  (`count`, `sum`, `bool_or`) or pick one row deliberately with an explicit
  `ORDER BY`. Never assume "there's only one in practice" — that is precisely
  the assumption `onboarding_period` and `allowance_model_item` break.

## Worth noting in the safe list

- **`payment_request` → its four subtype tables** (`contribution_request`,
  `allocation_request`, `deallocation_request`, `book_transfer_request`) are all
  1:1 on `request_id`. This is a table-per-subtype pattern: join all four and
  coalesce, rather than ranking.
- **`employee` → `leave_of_absence`** is 1:1 — there is at most **one LOA row
  per employee, ever**, not one per leave. Historical leaves are not retained.
- **`onboarding_period` → `qualifying_life_event`** is 1:1, so QLE fields can be
  joined inline. This is why `is_qle_enrollment` is just `qle.id IS NOT NULL`.
- **`user` → `employee`** and **`user` → `agent`** are both 1:1, so a user is at
  most one of each.
- **`sf_medical_plans` → `sf_plan_benefit_standardized`** share the composite key
  `(plan_id, plan_year)` — a clean 1:1 spine for plan models.

## Worth noting in the fan-out list

- **`employer` → `allowance_model`** fans out: one row **per year**. Joining
  without a year predicate silently multiplies rows by the number of plan years.
  Add `AND allowance_model.year = <plan year>` and the *result* is unique —
  that's the `(employer_id, year)` constraint doing the work.
- **`allowance_model` → `allowance_model_item`** fans out with **no uniqueness at
  all**. The rate grid can hold overlapping age bands or duplicate
  `(class, family_unit)` rows. Resolving an employee's allowance needs explicit
  tie-breaking, not a bare join.
- **`onboarding_period` → `benefit`** fans out — multiple products per period.
  Filter `benefit_type = 'MAJOR_MEDICAL'` to approach one row, but even that is
  not guaranteed (see below).
- **`onboarding_period` → `insured`** fans out — one row per covered person.
- **`employee` → `onboarding_period`** fans out — QLEs create extra periods.

## The "unique after a filter" pattern

Several fan-out joins become effectively 1:1 once you add the predicate the
business rule implies. These are the cases where a constraint buys you a
dedup-free model *if you filter correctly*:

| Parent → child | Add predicate | Then unique because |
|---|---|---|
| `employer` → `allowance_model` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `employer` → `allowance_model_v2` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `employer` → `class_definition` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `onboarding_period` → `insured` | `type = 'EMPLOYEE'` | app-enforced only — **not** a DB guarantee |
| `onboarding_period` → `insured` | `dependent_id = :id` | `UNIQUE (period_id, dependent_id)` |
| `employee` → `onboarding_period` | `is_active` | app-enforced only — **not** a DB guarantee |
| `onboarding_period` → `benefit` | `benefit_type = 'MAJOR_MEDICAL'` | **no constraint** — see caveat |
| `pull_expectation` → `expected_amount_version` | `version = max(version)` | `UNIQUE (expectation_id, version)` |

**Two of these are traps.** Filtering `insured` to `type = 'EMPLOYEE'`, or
`onboarding_period` to `is_active`, *looks* like it yields one row — and does in
healthy data — but neither is backed by a constraint. A duplicate would silently
fan out a model that assumes singularity.

The major-medical filter is the weakest: nothing prevents several
`MAJOR_MEDICAL` benefits in one period. The product's own
`intercom_sync_report` handles this with
`ORDER BY created_at, id LIMIT 1` and a migration comment recording that
production had zero such periods *as verified on 2026-07-29* — i.e. it is an
observed fact, not an invariant. Rank explicitly.


## Safe joins — at most one child row per parent (no dedup needed)

| parent | child | join on | child FK nullable |
|---|---|---|---|
| `agency` | `intercom_sync_status` | agency_id | yes |
| `allowance_model` | `allowance_model_source_snapshot` | allowance_model_id | no |
| `allowance_model` | `quote` | allowance_model_id | yes |
| `allowance_model` | `quote_v2` | allowance_model_id | yes |
| `contribution_by_class` | `quote_plan_design` | first_contribution_id | yes |
| `contribution_by_class` | `quote_plan_design` | second_contribution_id | yes |
| `employee` | `employee_info` | employee_id, employer_id | no |
| `employee` | `employee_payments_account` | employee_id, employer_id, user_id | no |
| `employee` | `existing_major_medical_plan` | employee_id, employer_id | no |
| `employee` | `finch_employee` | employee_id | no |
| `employee` | `finch_employee_identity` | employee_id | no |
| `employee` | `leave_of_absence` | employee_id, employer_id | no |
| `employer` | `employer_payments_account` | employer_id | no |
| `employer` | `enrollment_instructions` | employer_id | no |
| `employer` | `finch_employer` | employer_id | no |
| `employer` | `intercom_sync_status` | employer_id | yes |
| `employer_payments_account` | `lynx_employer_credentials` | employer_id | no |
| `finch_employer` | `finch_employer_eligibility` | employer_id, finch_employer_id | no |
| `onboarding_period` | `decision_factors_preference` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `payment_method` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `qualifying_life_event` | employee_id, employer_id, onboarding_period_id | no |
| `payment_request` | `allocation_request` | request_id | no |
| `payment_request` | `book_transfer_request` | request_id | no |
| `payment_request` | `contribution_request` | request_id | no |
| `payment_request` | `deallocation_request` | request_id | no |
| `payment_request` | `open_enrollment_period_payment_request` | request_id | no |
| `plan_exclusion_filter_set` | `employer` | plan_exclusion_filter_set_id | yes |
| `quote` | `quote_preference` | employer_id, quote_id | no |
| `quote` | `quote_session` | quote_id | no |
| `sf_medical_plans` | `sf_plan_benefit_standardized` | plan_id, plan_year | no |
| `user` | `agent` | user_id | no |
| `user` | `employee` | user_id | no |

## Fan-out joins — many child rows per parent (aggregate or rank)

| parent | child | join on | child FK nullable |
|---|---|---|---|
| `actual_transaction` | `actual_transaction_match` | transaction_id | no |
| `agency` | `agent` | agency_id | no |
| `agency` | `employer` | producer_id | yes |
| `agency` | `employer` | zorro_partner_id | yes |
| `agency` | `onboarding_period` | enrollment_team_id | yes |
| `agent` | `benefit` | enrollment_agent_id | yes |
| `agent` | `employer` | producer_agent_id | yes |
| `allowance_model` | `allowance_model_item` | allowance_model_id | no |
| `benefit` | `benefit_payment_request` | benefit_id, employee_id, employer_id | no |
| `benefit` | `healthsherpa_application` | benefit_id | yes |
| `class_definition` | `class_definition_item` | class_definition_id, employer_id | no |
| `dependent` | `insured` | dependent_id | yes |
| `employee` | `dependent` | employee_id, employer_id | no |
| `employee` | `eligibility_history` | employee_id, employer_id | no |
| `employee` | `finch_employee_notification` | employee_id, employer_id | no |
| `employee` | `onboarding_period` | employee_id, employer_id | no |
| `employee` | `payroll_individual` | employee_id, employer_id | no |
| `employee_payments_account` | `allocation_request` | employee_id, employer_id | no |
| `employee_payments_account` | `deallocation_request` | employee_id, employer_id | no |
| `employee_payments_account` | `employee_payments_alert` | employee_id, employer_id | no |
| `employer` | `allowance_model` | employer_id | no |
| `employer` | `allowance_model_item` | employer_id | no |
| `employer` | `class_definition` | employer_id | no |
| `employer` | `employee` | employer_id | no |
| `employer` | `employer_beneficial_owner` | employer_id | no |
| `employer` | `employer_contacts` | employer_id | no |
| `employer` | `employer_document` | employer_id | no |
| `employer` | `finch_employee` | employer_id | no |
| `employer` | `finch_employee_identity` | employer_id | no |
| `employer` | `open_enrollment_period` | employer_id | no |
| `employer` | `payroll_pay_group` | employer_id | no |
| `employer` | `quote` | employer_id | no |
| `employer` | `quote_session` | employer_id | no |
| `employer` | `quote_v2` | employer_id | no |
| `employer` | `sub_entity` | employer_id | no |
| `employer` | `sub_entity_2` | employer_id | no |
| `employer` | `user_activation_link` | employer_id | yes |
| `employer_payments_account` | `allocation_request` | employer_id | no |
| `employer_payments_account` | `book_transfer_request` | employer_id | no |
| `employer_payments_account` | `contribution_request` | employer_id | no |
| `employer_payments_account` | `deallocation_request` | employer_id | no |
| `employer_payments_account` | `employee_payments_account` | employer_id | no |
| `employer_payments_account` | `employee_payments_alert` | employer_id | no |
| `finch_deduction` | `finch_enrollment` | employer_id, finch_deduction_id | no |
| `finch_employee` | `finch_employee_notification` | finch_employee_id | yes |
| `finch_employee` | `finch_enrollment` | employer_id, finch_employee_id | no |
| `finch_employer` | `finch_deduction` | employer_id, finch_employer_id | no |
| `finch_employer` | `finch_employee_notification` | employer_id, finch_employer_id | no |
| `finch_employer` | `finch_employer_entity` | employer_id, finch_employer_id | no |
| `finch_employer` | `finch_payment` | employer_id, finch_employer_id | no |
| `finch_employer_eligibility` | `custom_class_finch_mapping` | eligibility_id, employer_id | no |
| `finch_employer_entity` | `finch_deduction` | finch_employer_entity_id | no |
| `finch_employer_entity` | `finch_employee` | finch_employer_entity_id | no |
| `finch_employer_entity` | `finch_payment` | finch_employer_entity_id | no |
| `finch_provider` | `finch_employer` | provider_id | yes |
| `onboarding_period` | `benefit` | employee_id, employer_id, period_id | no |
| `onboarding_period` | `benefit_document` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `cost_preference` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `drug_preference` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `electronic_signature` | employee_id, employer_id, period_id | no |
| `onboarding_period` | `enrollment_activity_log` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `enrollment_comment` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `finch_enrollment` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `insured` | employee_id, employer_id, period_id | no |
| `onboarding_period` | `provider_preference` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `review_snapshot` | employee_id, employer_id, period_id | no |
| `onboarding_period` | `saved_plan` | employee_id, employer_id, onboarding_period_id | no |
| `onboarding_period` | `shopping_preference` | employee_id, employer_id, onboarding_period_id | no |
| `open_enrollment_period` | `employer_document` | open_enrollment_period_id | yes |
| `open_enrollment_period` | `onboarding_period` | enrollment_period_id | no |
| `open_enrollment_period` | `open_enrollment_period_payment_request` | employer_id, open_enrollment_period_id | no |
| `payment_request` | `benefit_payment_request` | request_id | no |
| `payment_request` | `pull_expectation` | request_id | no |
| `payroll_pay_group` | `payroll_individual` | employer_id, pay_group_id | yes |
| `payroll_pay_group` | `payroll_pay_period` | employer_id, pay_group_id | no |
| `plan_exclusion_filter_set` | `plan_exclusion_filter` | plan_exclusion_filter_set_id | no |
| `provider_preference` | `provider_preference_address` | employee_id, employer_id, onboarding_period_id, provider_npi | no |
| `pull_expectation` | `actual_transaction_match` | expectation_id | no |
| `pull_expectation` | `expected_amount_version` | expectation_id | no |
| `quote` | `census_employee` | employer_id, quote_id | no |
| `quote` | `census_plan` | employer_id, quote_id | no |
| `quote` | `quote_age_cost_unit` | employer_id, quote_id | no |
| `quote` | `quote_employee` | employer_id, quote_id | no |
| `quote` | `quote_plan` | employer_id, quote_id | no |
| `quote` | `quote_plan_diversity` | employer_id, quote_id | no |
| `quote_plan_design` | `quote_allowance_model_row` | employer_id, quote_plan_design_id | no |
| `quote_v2` | `census_employee_v2` | employer_id, quote_id | no |
| `quote_v2` | `census_plan_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_age_cost_unit_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_age_premium_pool_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_allowance_model_row` | employer_id, quote_id | no |
| `quote_v2` | `quote_employee_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_plan_design` | employer_id, quote_id | no |
| `quote_v2` | `quote_plan_diversity_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_plan_v2` | employer_id, quote_id | no |
| `quote_v2` | `quote_presets` | employer_id, quote_id | no |
| `review_snapshot` | `electronic_signature` | review_snapshot_id | yes |
| `role` | `user_role` | role_id | no |
| `sf_medical_plans` | `sf_plan_pricing_zip` | plan_id, plan_year | no |
| `shopping_preference_option` | `shopping_preference` | type, value, version | no |
| `sub_entity` | `employee` | sub_entity_id | yes |
| `sub_entity_2` | `employee` | sub_entity_2_id | yes |
| `user` | `employer` | csm_user_id | yes |
| `user` | `user_activation_link` | user_id | no |
| `user` | `user_role` | user_id | no |
| `workflow_execution` | `workflow_task` | workflow_execution_id | no |
| `workflow_task` | `workflow_task_error` | task_id | no |

---

# Part A — Grain and uniqueness

Every table's primary key plus any additional UNIQUE sets. The second column is
your `unique` test; the third is `dbt_utils.unique_combination_of_columns`.

## A.1 How to read the UNIQUE sets

Three different kinds appear, and they are **not** equally useful:

**Business keys — genuinely useful.** `(employer_id, year)`,
`(name, state)`, `(zip_code, state_code)`. These express a real rule and make
good natural-key tests.

**Vendor idempotency keys — useful for dedup.**
`(vendor, vendor_correlation_id, vendor_account_id)`,
`(provider, external_event_id)`, `(employer_id, idempotency_key)`. These
guarantee at-most-once ingestion of an external event.

**Compound-FK targets — NOT business rules.** Any UNIQUE set containing `id`
(e.g. `benefit (id, employee_id, employer_id)`) exists only so a child table can
reference it on a compound key for Row-Level Security. It is implied by the PK
and tells you nothing. **Do not build a dbt test around these** — they will
always pass and they hide the fact that no real uniqueness exists.

`onboarding_period` is the cautionary case: it has two such UNIQUE sets and
still permits many rows per employee per enrollment period.

## A.2 The business-key rules worth testing

| Table | Rule | What it means |
|---|---|---|
| `allowance_model` | `(employer_id, year)` | one rate table per employer per plan year |
| `allowance_model_v2` | `(employer_id, year)` | same, V2 pipeline — independently unique |
| `class_definition` | `(employer_id, year)` | one class scheme per employer per plan year |
| `carrier_info` | `(name, state)` | carrier facts are per-state |
| `major_medical_carrier` | `(name, state)` | same |
| `state_appointments` | `name`, `code` | one row per state |
| `blended_rate_decisions` | `(state_code, hios, year)` | one rate decision per plan per year |
| `sub_entity` / `sub_entity_2` | `(name, employer_id)` | sub-entity names unique within employer |
| `class_definition_item` | `(class_definition_id, class_name)` | no duplicate class names in a scheme |
| `existing_major_medical_plan` | `(employee_id, employer_id)` | at most one prior plan per employee |
| `leave_of_absence` | `(employee_id, employer_id)`, `employee_id` | **at most one LOA per employee, ever** |
| `qualifying_life_event` | `onboarding_period_id` | at most one QLE per period |
| `decision_factors_preference` | `(onboarding_period_id, employee_id, employer_id)` | one per period |
| `saved_plan` | `(plan_id, onboarding_period_id)` | a plan is saved at most once per period |
| `insured` | `(period_id, dependent_id)` | a dependent appears at most once per period |
| `payment_method` (Echo) | `onboarding_period_id` | one Echo payment method per period |
| `employee_payments_account` | `(employee_id, employer_id)`, `user_id` | one Lynx account per employee |
| `pull_expectation` | `(employee_id, plan_year, coverage_month, expected_carrier)` | one expectation per member-month-carrier |
| `expected_amount_version` | `(expectation_id, version)` | version numbers don't repeat |
| `actual_transaction_match` | `(transaction_id, expectation_id)` | a transaction matches an expectation once |
| `benefit_payment_request` | `(request_id, benefit_id)` | |
| `payment_request` | `(employer_id, idempotency_key)` | idempotent submission |
| `quote_age_premium_pool_v2` | `(quote_id, scenario, aggregator, state_code, rating_area, age)` | the quoting grain |
| `quote_allowance_model_row` | `(quote_plan_design_id, quote_id, state_code, rating_area, age, employment_type, wage_type)` | the allowance-row grain |
| `quote_presets` | `(quote_id, preset)` | |
| `payroll_pay_group` | `(employer_id, provider_employer_entity_id, provider_pay_group_id)` | |
| `payroll_pay_period` | `(pay_group_id, start_date)` | no overlapping period starts |
| `payroll_individual` | `(employer_id, provider_individual_id, provider_employer_entity_id)` | |
| `finch_deduction` | `(finch_employer_id, finch_employer_entity_id, hsa_eligible, tax_type)` | one deduction per tax treatment |
| `finch_enrollment` | `(finch_employee_id, finch_deduction_id)` | |
| `finch_payment` | `(finch_payment_id, pay_group_id)` | |
| `finch_employee_notification` | `(employee_id, finch_employer_id, notification_type)` | notification sent once |
| `custom_class_finch_mapping` | `(eligibility_id, category_name_zorro)` | |
| `webhook_event` | `(provider, external_event_id)` | at-most-once webhook ingestion |
| `actual_transaction` | `(vendor, vendor_correlation_id, vendor_account_id)` | at-most-once transaction ingestion |
| `carrier_vendor_mapping` | `(vendor, vendor_merchant_id)` | |
| `special_case_zip_codes` | `(zip_code, state_code)` | |
| `workflow_task` | `task_hash` | task dedup |
| `workflow_execution` | `graphile_job_id` | |

## A.3 Composite primary keys — the natural grain

These tables have **no surrogate `id`**; the PK *is* the business grain. They are
the cleanest dbt sources because the key needs no derivation:

| Table | Composite PK |
|---|---|
| `sf_medical_plans` | `(plan_id, plan_year)` |
| `sf_plan_benefit_standardized` | `(plan_id, plan_year)` |
| `sf_plan_pricing_zip` | `(plan_id, plan_year, zip_code, age, fips_code)` |
| `sf_premium_age_factor` | `(state_code, age)` |
| `sf_state_benefit_params` | `(state_code, plan_year)` |
| `zip_county` | `(zip_code, state_code, county_name, plan_year)` |
| `zip_state` | `(zip_code, state_code)` |
| `plan_year_data` | `year` |
| `state_carrier_rate_change` | `(state_code, carrier_name)` |
| `dw_blended_rate_decision_versions` | `(state_code, hios, year, valid_from)` — **SCD2, see below** |
| `user_role` | `(user_id, role_id)` |
| `cost_preference` | `(onboarding_period_id, attribute)` |
| `drug_preference` | `(external_id, onboarding_period_id)` |
| `provider_preference` | `(npi, onboarding_period_id)` |
| `provider_preference_address` | `(id, provider_npi, onboarding_period_id)` |
| `shopping_preference` | `(onboarding_period_id, type)` |
| `shopping_preference_option` | `(type, version, value)` |
| `recommendation_data` | `(plan_year, plan_id, care_level_1..4_count)` |
| `workflow_rate_limit` | `(task_type, window_start)` |

**`dw_blended_rate_decision_versions` is already a type-2 slowly-changing
dimension** — it carries `valid_from` / `valid_to` (`Timestamptz`), with
`valid_to IS NULL` marking the current row. Do not re-snapshot it; filter it.
Its sibling `blended_rate_decisions` holds only the current state.

## A.4 Tables with NO uniqueness beyond the surrogate key

These accept unlimited rows per logical entity. Any dbt model that assumes one
row per parent here will silently fan out:

`allowance_model_item` · `agency` · `employer_contacts` ·
`employer_beneficial_owner` · `enrollment_activity_log` · `enrollment_comment` ·
`eligibility_history` · `employee_change_log` · `benefit_document` ·
`electronic_signature` · `review_snapshot` · `census_employee(_v2)` ·
`census_plan(_v2)` · `quote_employee(_v2)` · `quote_plan(_v2)` ·
`quote_age_cost_unit(_v2)` · `quote_plan_diversity(_v2)` ·
`contribution_by_class` · `audit` · `data_change_audit` · `event` ·
`address_resolve_responses` · `request_context` · `workflow_task_error` ·
`plan_exclusion_filter` · `specialty`

Two to watch especially:

- **`allowance_model_item`** — the actual rate grid. Nothing prevents
  overlapping age bands or duplicate `(class, family_unit)` rows. A duplicate
  here silently changes an employee's allowance.
- **`eligibility_history`** / **`employee_change_log`** — append-only histories;
  expect many rows per employee and dedupe deliberately.

---

# Part B — Foreign keys and delete behaviour

139 declared relations. Each is a `relationships` test candidate. The `on delete`
column matters for warehouse modelling: **92 are `Cascade`**, so parent deletes
remove children outright — there is no tombstone to replicate.

## B.1 Delete-behaviour summary

| Behaviour | Count | Warehouse implication |
|---|---|---|
| `Cascade` | 92 | child rows vanish with the parent; incremental models need a full-refresh or delete-detection strategy |
| `SetNull` | some | FK becomes `NULL`, row survives — e.g. `onboarding_period.enrollment_team_id`, `insured.dependent_id` |
| default (`Restrict`) | rest | delete blocked while children exist |

**There is effectively no soft delete.** Only `enrollment_comment.deleted_at`
exists. Snapshot-based dbt models cannot detect hard deletes from the source
alone — plan for periodic full refreshes on cascade-heavy trees, or rely on CDC.

## B.2 Compound foreign keys

Several FKs are multi-column because of Row-Level Security denormalisation —
your `relationships` tests must match on **all** columns, not just the obvious
one:

| Child | FK columns | Parent |
|---|---|---|
| `insured` | `period_id, employee_id, employer_id` | `onboarding_period` |
| `benefit` | `period_id, employee_id, employer_id` | `onboarding_period` |
| `employee` | `id, employer_id` | — (target of others) |
| `allocation_request` | `employee_id, employer_id` | `employee_payments_account` |
| `qualifying_life_event` | `onboarding_period_id, employee_id, employer_id` | `onboarding_period` |

The full child→parent matrix is in Part F below.

## B.3 The join table with no model

`_benefit_to_insured` exists in no `.prisma` file (Prisma implicit M2M):

```sql
"A" UUID -> benefit(id)   ON DELETE CASCADE
"B" UUID -> insured(id)   ON DELETE CASCADE
PRIMARY KEY ("A","B")
```

Quote the column names — unquoted `A`/`B` will not resolve. It carries **no
`employer_id`**, so it has no RLS policy of its own and must be joined through
`benefit` or `insured` for tenant filtering.

---

# Part C — Accepted values

**242 enum-typed columns** across **115 enums**. Every one is an
`accepted_values` test; the full value lists are in the companion file's
Appendix, and the per-column map is in Part G below.

Two cautions:

**Enums change, and values get removed.** Postgres cannot drop an enum value, so
removals are done by rename-and-swap. `SubmissionType` lost `JOINT_SESSION` on
2026-09-23 (PR #9122): historical rows were rewritten to `BY_OPERATOR` with a new
`is_impersonated = true` flag, and two new values `BY_EMPLOYER_ADMIN` /
`BY_AGENT` were added with **no history behind them**. Hard-coded
`accepted_values` lists need a review step when the schema changes, and
time-series splits by such a column will show discontinuities at the migration
date rather than real behaviour change.

**Array-typed enum columns** need `accepted_values` applied per element, not to
the column: `enrollment_tags` (`EnrollmentTag[]`), `benefit_types`
(`BenefitType[]`), `decision_factors` (`DecisionFactors[]`), `priorities`
(`QuotePlanPriorities[]`).

---

# Part D — Derived statuses (dbt must replicate these exactly)

None of these are stored. If a warehouse model recomputes them by hand it will
diverge from the product. **Prefer calling the database functions or selecting
from the views.**

## D.1 `enrollment_status` — the most important one

`calculate_enrollment_status(onboarding_period_id, is_active, last_user_login,
onboarding_until, benefit_id, is_waived, benefit_effective_from,
benefit_effective_until, benefit_status, today)` → `EnrollmentStatus`
(`IMMUTABLE`, so it is safe in indexes and `GROUP BY`).

Evaluated **in order** — the first matching branch wins:

| # | Condition | Result |
|---|---|---|
| 1 | `onboarding_period_id IS NULL` | `NO_ENROLLMENTS` |
| 2 | `is_waived` AND `today < effective_from` | `WAIVED_ELECTION` |
| 3 | `is_waived` (otherwise) | `WAIVED_COVERAGE` |
| 4 | `is_active` AND `last_user_login IS NOT NULL` | `ELECTION_ACTIVE_STARTED` |
| 5 | `is_active` (otherwise) | `ELECTION_ACTIVE_HAS_NOT_STARTED` |
| 6 | `benefit_id IS NULL` OR `status = IN_CART`, AND `today < onboarding_until` | `PENDING_ELECTION_WINDOW` |
| 7 | `benefit_id IS NULL` OR `status = IN_CART` (otherwise) | `DEADLINE_PASSED` |
| 8 | `status IN (READY_TO_APPLY, AWAITING_MEDICAL, AWAITING_EMPLOYEE)` | `ELECTION_SUBMITTED` |
| 9 | `status IN (CARRIER_APPLICATION_SENT, AWAITING_PAYMENT)` | `CARRIER_APPLICATION_SENT` |
| 10 | `status IN (ENROLLMENT_CONFIRMED, ACTIVE, ENDED)` AND `today < effective_from` | `ENROLLMENT_CONFIRMED` |
| 11 | same, AND `today BETWEEN effective_from AND effective_until` | `ACTIVE_COVERAGE` |
| 12 | same, otherwise | `COVERAGE_ENDED` |
| 13 | fallback | `NO_ENROLLMENTS` |

Five consequences for dbt:

1. **It takes `today`.** The same unchanged row returns a different status
   tomorrow. A table materialisation silently freezes a date; an incremental
   model freezes a *different* date per partition. Either pass an explicit
   as-of date or re-derive on read.
2. **Ordering is load-bearing.** Waiver beats activity, activity beats benefit
   status. A `CASE` written in a different order gives different answers.
3. **The buckets are coarser than `BenefitStatus`.** Nine benefit statuses
   collapse into these thirteen, which also encode waiver and activity. The
   mapping is many-to-one and not invertible.
4. **`benefit_status` and `enrollment_status` can disagree and both be right** —
   a benefit sits in `ACTIVE` while showing `COVERAGE_ENDED` between
   `effective_until` passing and the nightly cron flipping it to `ENDED`.
5. **`ELECTION_ACTIVE` is in the enum but unreachable** from this function — only
   the two `_STARTED` / `_HAS_NOT_STARTED` variants are produced, so expect zero
   rows. Application code still lists it in membership arrays (e.g.
   `NOT_SUBMITTED_ELECTION_STATUSES`) for exhaustiveness, so keep it in any
   `accepted_values` list, but treat a non-zero count as a red flag.

## D.2 `benefit_eligibility` (`EmploymentStatus`)

`calculate_employment_status(eligible_from, eligible_until, termination_date,
leave_of_absence_start_date, leave_of_absence_end_date, today)`, also in order:

| # | Condition | Result |
|---|---|---|
| 1 | `eligible_until < today` AND `termination_date < today` | `INELIGIBLE_TERMINATED` |
| 2 | `eligible_until < today` (otherwise) | `INELIGIBLE` |
| 3 | LOA window contains `today` | `INELIGIBLE_ON_LEAVE` |
| 4 | eligible started, `eligible_until IS NULL`, future LOA exists | `ELIGIBLE_UPCOMING_LEAVE` |
| 5 | eligible started, `eligible_until IS NULL` | `ELIGIBLE_EMPLOYED` |
| 6 | eligible started, `eligible_until >= today` | `ELIGIBLE_ENDING` |
| 7 | `eligible_from > eligible_until` | `INELIGIBLE` |
| 8 | `eligible_from > today` | `INELIGIBLE_WAITING` |
| 9 | fallback | `NULL` |

Note branch 1: `INELIGIBLE_TERMINATED` requires **both** expired eligibility and
a past termination date. An employee with a termination date but unexpired
eligibility is still `ELIGIBLE_*`. Also note the function **can return `NULL`**.

## D.3 `renewal_status`

From `enrollment_report`, in order:

| # | Condition | Result |
|---|---|---|
| 1 | `is_waived` | `WAIVED` |
| 2 | no benefit, or benefit is not `MAJOR_MEDICAL` | `NOT_APPLICABLE` |
| 3 | no previous benefit found | `NOT_APPLICABLE` |
| 4 | **all** of: same `external_plan_id`; same insured count; previous `effective_until + 1 day = effective_from`; **and** identity matches | `RENEWED` |
| 5 | otherwise | `CHANGED` |

Identity matches when *either* every insured has an SSN and the **SSN
fingerprint** equals the previous one, *or* the **name fingerprint** matches
(name + date of birth). This is a genuine fuzzy-identity rule — reimplementing
it loosely will misclassify renewals. Note it is **major-medical only** and
requires contiguous coverage to the day.

## D.4 `aor_cancellation_status`

| Condition | Result |
|---|---|
| OEP `is_aor_changed` AND **not** special enrollment AND `benefit.is_aor_confirmed_by_employee` | `CONFIRMED_BY_EMPLOYEE` |
| OEP `is_aor_changed` AND not special enrollment, otherwise | `AWAITING_EMPLOYEE` |
| otherwise | `NOT_APPLICABLE` |

## D.5 `carrier_request_status`

| # | Condition | Result |
|---|---|---|
| 1 | `is_waived`, or no benefit | `NOT_APPLICABLE` |
| 2 | employee state unknown, or no matching `carrier_info` | **`NULL`** |
| 3 | carrier has special requests AND `is_special_requests_confirmed` | `CONFIRMED_BY_EMPLOYEE` |
| 4 | carrier has special requests, otherwise | `AWAITING_EMPLOYEE` |
| 5 | otherwise | `NOT_APPLICABLE` |

Branch 2 is the trap: `NULL` here means *"cannot determine"*, which is
semantically different from `NOT_APPLICABLE`. `carrier_info` is joined on
`(name, state)`, so a carrier-name mismatch produces nulls, not falses.

## D.6 `payment_method_type` (`employee_report`)

| Condition | Result |
|---|---|
| no period, no benefit, or waived | `NULL` |
| `self_pay_amount` is `NULL` or `0` | `ZORRO_PAY` |
| `self_pay_amount = premium` | `SELF_PAY` |
| `0 < self_pay_amount < premium` | `COMBINED` |
| fallback | `ZORRO_PAY` |

## D.7 `family_unit`

Derived from `_benefit_to_insured`, **not** from the period's insured rows:

| Condition (over insured joined through the junction) | Result |
|---|---|
| has spouse AND has child | `FAMILY` |
| has spouse | `EMPLOYEE_SPOUSE` |
| has child | `EMPLOYEE_CHILD` |
| `count(*) > 0` | `EMPLOYEE_ONLY` |
| else | `NULL` |

It was a stored `benefit.family_unit` column until Feb 2026 (PR #7134), then
dropped and inverted into this derivation. **It selects the allowance tier, so
junction rows determine money.**

Note `employee_report` uses a subtly different variant whose final branch is
`ELSE 'EMPLOYEE_ONLY'` rather than `WHEN count(*) > 0 ... ELSE NULL` — so a
benefit covering nobody reads as `EMPLOYEE_ONLY` there and `NULL` in
`enrollment_report`. Reconcile deliberately if you union them.

## D.8 Simple derived flags

| Column | Rule |
|---|---|
| `is_qle_enrollment` | `qle.id IS NOT NULL` |
| `has_user_logged_in` | `op.last_user_login IS NOT NULL` |
| `has_previous_eligibility` | `EXISTS` row in `eligibility_history` |
| `has_qualifying_benefit` | `COALESCE(status IN (ENROLLMENT_CONFIRMED, ACTIVE, ENDED), FALSE)` |
| `enrollment_type` | `SPECIAL` if `is_special_enrollment` else `OPEN_ENROLLMENT` |
| `enrollment_submission_date` | `COALESCE(benefit.submitted_at, period.waived_at)` |
| FPL year | `resolve_fpl_year(year, start_of_coverage)` — prior year if coverage starts <6 months after publication |

## D.9 The row-grain selectors — read this before counting anything

`employee_report` is **one row per onboarding period**, not per employee. An
employee with a QLE has several rows. Four mutually-exclusive boolean selectors
exist, each a `row_number() OVER (PARTITION BY open_enrollment_period_id,
employee_id ...)` = 1:

| Selector | Picks |
|---|---|
| `is_open_enrollment_view` | the non-special (OE) period |
| `is_latest_coverage_view` | latest period in a coverage status |
| `is_enrollments_in_process_view` | earliest period not yet in coverage |
| `is_all_view` | latest period overall |

**Counting employees without filtering on one of these multi-counts anyone who
has had a QLE.** `intercom_sync_report` solves it differently, with
`LEFT JOIN LATERAL ... ORDER BY created_at, id LIMIT 1`.

Also note `employee_report` joins `benefit` with
`benefit_type = 'MAJOR_MEDICAL'` only — supplementals never appear there, and
`onboarding_period_id_key` is `COALESCE(onboarding_period_id, '000...0')` so the
unique key survives employees with no period.

---

# Part E — Rules that are NOT in the schema

dbt tests cannot be derived for these; they need bespoke assertions.

| Rule | Where it lives | Suggested test |
|---|---|---|
| Exactly one `EMPLOYEE` insured per period | app code only | `count(*) filter (where type='EMPLOYEE') = 1` per `period_id` |
| At most one spouse per period | app code only | same shape for `SPOUSE_OR_DOMESTIC_PARTNER` |
| One active period per employee | app code only | `count(*) filter (where is_active) <= 1` per `employee_id` |
| A benefit covers ≥1 insured | app code only | anti-join `benefit` to `_benefit_to_insured` |
| `effective_from <= effective_until` | nowhere | explicit range test |
| Non-negative money | nowhere | explicit test on premium/contribution columns |
| Valid status transitions | app code only | not testable from a snapshot |
| Allowance = ER + EE reconciliation | app code | `employer_contribution + employee_contribution = premium` where allocated |

Five of these are exactly the integrity checks the schema permits and the
application cannot see.

---

# Part F — Full foreign-key matrix

| child table | FK columns | parent table | parent columns | on delete | nullable |
|---|---|---|---|---|---|
| `actual_transaction_match` | expectation_id | `pull_expectation` | id | Cascade | no |
| `actual_transaction_match` | transaction_id | `actual_transaction` | id | Cascade | no |
| `agent` | agency_id | `agency` | id | Cascade | no |
| `agent` | user_id | `user` | id | Cascade | no |
| `allocation_request` | employee_id, employer_id | `employee_payments_account` | employee_id, employer_id | (default: Restrict) | no |
| `allocation_request` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `allocation_request` | request_id | `payment_request` | id | (default: Restrict) | no |
| `allowance_model` | employer_id | `employer` | id | Cascade | no |
| `allowance_model_item` | allowance_model_id | `allowance_model` | id | Cascade | no |
| `allowance_model_item` | employer_id | `employer` | id | Cascade | no |
| `allowance_model_source_snapshot` | allowance_model_id | `allowance_model` | id | Cascade | no |
| `benefit` | enrollment_agent_id | `agent` | id | SetNull | yes |
| `benefit` | period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `benefit_document` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `benefit_payment_request` | benefit_id, employee_id, employer_id | `benefit` | id, employee_id, employer_id | Cascade | no |
| `benefit_payment_request` | request_id | `payment_request` | id | Cascade | no |
| `book_transfer_request` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `book_transfer_request` | request_id | `payment_request` | id | (default: Restrict) | no |
| `census_employee` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `census_employee_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `census_plan` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `census_plan_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `class_definition` | employer_id | `employer` | id | Cascade | no |
| `class_definition_item` | class_definition_id, employer_id | `class_definition` | id, employer_id | Cascade | no |
| `contribution_request` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `contribution_request` | request_id | `payment_request` | id | (default: Restrict) | no |
| `cost_preference` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `custom_class_finch_mapping` | eligibility_id, employer_id | `finch_employer_eligibility` | id, employer_id | Cascade | no |
| `deallocation_request` | employee_id, employer_id | `employee_payments_account` | employee_id, employer_id | (default: Restrict) | no |
| `deallocation_request` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `deallocation_request` | request_id | `payment_request` | id | (default: Restrict) | no |
| `decision_factors_preference` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `dependent` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `drug_preference` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `electronic_signature` | period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `electronic_signature` | review_snapshot_id | `review_snapshot` | id | (default: Restrict) | yes |
| `eligibility_history` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `employee` | employer_id | `employer` | id | Cascade | no |
| `employee` | sub_entity_2_id | `sub_entity_2` | id | Restrict | yes |
| `employee` | sub_entity_id | `sub_entity` | id | Restrict | yes |
| `employee` | user_id | `user` | id | Cascade | no |
| `employee_info` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `employee_payments_account` | employee_id, employer_id, user_id | `employee` | id, employer_id, user_id | (default: Restrict) | no |
| `employee_payments_account` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `employee_payments_alert` | employee_id, employer_id | `employee_payments_account` | employee_id, employer_id | (default: Restrict) | no |
| `employee_payments_alert` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `employer` | csm_user_id | `user` | id | Restrict | yes |
| `employer` | plan_exclusion_filter_set_id | `plan_exclusion_filter_set` | id | SetNull | yes |
| `employer` | producer_agent_id | `agent` | id | Restrict | yes |
| `employer` | producer_id | `agency` | id | Restrict | yes |
| `employer` | zorro_partner_id | `agency` | id | Restrict | yes |
| `employer_beneficial_owner` | employer_id | `employer` | id | Cascade | no |
| `employer_contacts` | employer_id | `employer` | id | Cascade | no |
| `employer_document` | employer_id | `employer` | id | Cascade | no |
| `employer_document` | open_enrollment_period_id | `open_enrollment_period` | id | Cascade | yes |
| `employer_payments_account` | employer_id | `employer` | id | (default: Restrict) | no |
| `enrollment_activity_log` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `enrollment_comment` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `enrollment_instructions` | employer_id | `employer` | id | Cascade | no |
| `existing_major_medical_plan` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `expected_amount_version` | expectation_id | `pull_expectation` | id | Cascade | no |
| `finch_deduction` | finch_employer_entity_id | `finch_employer_entity` | id | Cascade | no |
| `finch_deduction` | finch_employer_id, employer_id | `finch_employer` | id, employer_id | Cascade | no |
| `finch_employee` | employee_id | `employee` | id | Restrict | no |
| `finch_employee` | employer_id | `employer` | id | Restrict | no |
| `finch_employee` | finch_employer_entity_id | `finch_employer_entity` | id | Cascade | no |
| `finch_employee_identity` | employee_id | `employee` | id | Cascade | no |
| `finch_employee_identity` | employer_id | `employer` | id | Restrict | no |
| `finch_employee_notification` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `finch_employee_notification` | finch_employee_id | `finch_employee` | id | Cascade | yes |
| `finch_employee_notification` | finch_employer_id, employer_id | `finch_employer` | id, employer_id | Cascade | no |
| `finch_employer` | employer_id | `employer` | id | Restrict | no |
| `finch_employer` | provider_id | `finch_provider` | id | SetNull | yes |
| `finch_employer_eligibility` | finch_employer_id, employer_id | `finch_employer` | id, employer_id | Cascade | no |
| `finch_employer_entity` | finch_employer_id, employer_id | `finch_employer` | id, employer_id | Cascade | no |
| `finch_enrollment` | finch_deduction_id, employer_id | `finch_deduction` | id, employer_id | Restrict | no |
| `finch_enrollment` | finch_employee_id, employer_id | `finch_employee` | id, employer_id | Restrict | no |
| `finch_enrollment` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Restrict | no |
| `finch_payment` | finch_employer_entity_id | `finch_employer_entity` | id | Cascade | no |
| `finch_payment` | finch_employer_id, employer_id | `finch_employer` | id, employer_id | Cascade | no |
| `healthsherpa_application` | benefit_id | `benefit` | id | SetNull | yes |
| `insured` | dependent_id | `dependent` | id | SetNull | yes |
| `insured` | period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `intercom_sync_status` | agency_id | `agency` | id | Cascade | yes |
| `intercom_sync_status` | employer_id | `employer` | id | Cascade | yes |
| `leave_of_absence` | employee_id, employer_id | `employee` | id, employer_id | (default: Restrict) | no |
| `lynx_employer_credentials` | employer_id | `employer_payments_account` | employer_id | (default: Restrict) | no |
| `onboarding_period` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `onboarding_period` | enrollment_period_id | `open_enrollment_period` | id | Cascade | no |
| `onboarding_period` | enrollment_team_id | `agency` | id | SetNull | yes |
| `open_enrollment_period` | employer_id | `employer` | id | Cascade | no |
| `open_enrollment_period_payment_request` | open_enrollment_period_id, employer_id | `open_enrollment_period` | id, employer_id | Cascade | no |
| `open_enrollment_period_payment_request` | request_id | `payment_request` | id | Cascade | no |
| `payment_method` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `payroll_individual` | employee_id, employer_id | `employee` | id, employer_id | Cascade | no |
| `payroll_individual` | pay_group_id, employer_id | `payroll_pay_group` | id, employer_id | (default: Restrict) | yes |
| `payroll_pay_group` | employer_id | `employer` | id | Cascade | no |
| `payroll_pay_period` | pay_group_id, employer_id | `payroll_pay_group` | id, employer_id | Cascade | no |
| `plan_exclusion_filter` | plan_exclusion_filter_set_id | `plan_exclusion_filter_set` | id | Cascade | no |
| `provider_preference` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `provider_preference_address` | provider_npi, onboarding_period_id, employer_id, employee_id | `provider_preference` | npi, onboarding_period_id, employer_id, employee_id | Cascade | no |
| `pull_expectation` | request_id | `payment_request` | id | Restrict | no |
| `qualifying_life_event` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `quote` | allowance_model_id | `allowance_model` | id | Restrict | yes |
| `quote` | employer_id | `employer` | id | Cascade | no |
| `quote_age_cost_unit` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `quote_age_cost_unit_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_age_premium_pool_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_allowance_model_row` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_allowance_model_row` | quote_plan_design_id, employer_id | `quote_plan_design` | id, employer_id | Cascade | no |
| `quote_employee` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `quote_employee_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_plan` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `quote_plan_design` | first_contribution_id | `contribution_by_class` | id | (default: Restrict) | yes |
| `quote_plan_design` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_plan_design` | second_contribution_id | `contribution_by_class` | id | (default: Restrict) | yes |
| `quote_plan_diversity` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `quote_plan_diversity_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_plan_v2` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_preference` | quote_id, employer_id | `quote` | id, employer_id | Cascade | no |
| `quote_presets` | quote_id, employer_id | `quote_v2` | id, employer_id | Cascade | no |
| `quote_session` | employer_id | `employer` | id | Cascade | no |
| `quote_session` | quote_id | `quote` | id | Cascade | no |
| `quote_v2` | allowance_model_id | `allowance_model` | id | Restrict | yes |
| `quote_v2` | employer_id | `employer` | id | Cascade | no |
| `review_snapshot` | period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `saved_plan` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `sf_plan_benefit_standardized` | plan_id, plan_year | `sf_medical_plans` | plan_id, plan_year | (default: Restrict) | no |
| `sf_plan_pricing_zip` | plan_id, plan_year | `sf_medical_plans` | plan_id, plan_year | (default: Restrict) | no |
| `shopping_preference` | onboarding_period_id, employee_id, employer_id | `onboarding_period` | id, employee_id, employer_id | Cascade | no |
| `shopping_preference` | type, version, value | `shopping_preference_option` | type, version, value | (default: Restrict) | no |
| `sub_entity` | employer_id | `employer` | id | Cascade | no |
| `sub_entity_2` | employer_id | `employer` | id | Cascade | no |
| `user_activation_link` | employer_id | `employer` | id | Cascade | yes |
| `user_activation_link` | user_id | `user` | id | Cascade | no |
| `user_role` | role_id | `role` | id | Restrict | no |
| `user_role` | user_id | `user` | id | Cascade | no |
| `workflow_task` | workflow_execution_id | `workflow_execution` | id | Cascade | no |
| `workflow_task_error` | task_id | `workflow_task` | id | Cascade | no |

# Part G — Grain matrix (all tables)

| table | grain (PK) | additional UNIQUE sets |
|---|---|---|
| `actual_transaction` | id | (vendor, vendor_correlation_id, vendor_account_id) |
| `actual_transaction_match` | id | (transaction_id, expectation_id) |
| `address_resolve_responses` | id | — |
| `agency` | id | — |
| `agent` | id | user_id |
| `allocation_request` | id | request_id |
| `allowance_model` | id | (employer_id, year) |
| `allowance_model_item` | id | — |
| `allowance_model_source_snapshot` | id | allowance_model_id |
| `audit` | id | — |
| `benefit` | id | (id, employee_id, employer_id) |
| `benefit_document` | id | document_url |
| `benefit_payment_request` | id | (request_id, benefit_id) |
| `blended_rate_decisions` | id | (state_code, hios, year) |
| `book_transfer_request` | id | request_id |
| `cache_data` | id | — |
| `carrier_info` | id | (name, state) |
| `carrier_vendor_mapping` | id | (vendor, vendor_merchant_id) |
| `census_employee` | id | — |
| `census_employee_v2` | id | — |
| `census_plan` | id | — |
| `census_plan_v2` | id | — |
| `class_definition` | id | (id, employer_id) · (employer_id, year) |
| `class_definition_item` | id | (class_definition_id, class_name) |
| `contribution_by_class` | id | — |
| `contribution_request` | id | request_id |
| `cost_preference` | onboarding_period_id, attribute | — |
| `custom_class_finch_mapping` | id | (eligibility_id, category_name_zorro) |
| `data_change_audit` | id | — |
| `deallocation_request` | id | request_id |
| `decision_factors_preference` | onboarding_period_id | (onboarding_period_id, employee_id, employer_id) |
| `dependent` | id | — |
| `drug_preference` | external_id, onboarding_period_id | — |
| `dw_blended_rate_decision_versions` | state_code, hios, year, valid_from | — |
| `electronic_signature` | id | — |
| `eligibility_history` | id | — |
| `employee` | id | (id, employer_id) · (id, employer_id, user_id) · user_id · email |
| `employee_change_log` | id | — |
| `employee_info` | employee_id | (employee_id, employer_id) |
| `employee_payments_account` | employee_id | (employee_id, employer_id) · (employee_id, employer_id, user_id) · user_id |
| `employee_payments_alert` | id | (type, idempotency_key) |
| `employee_report` *(view)* | — | (employee_id, onboarding_period_id_key) |
| `employer` | id | id · plan_exclusion_filter_set_id |
| `employer_beneficial_owner` | id | — |
| `employer_contacts` | id | — |
| `employer_document` | id | document_url |
| `employer_payments_account` | employer_id | — |
| `enrollment_activity_log` | id | — |
| `enrollment_comment` | id | — |
| `enrollment_instructions` | id | employer_id |
| `enrollment_report` *(view)* | — | (id) |
| `event` | id | — |
| `existing_major_medical_plan` | employee_id | (employee_id, employer_id) |
| `expected_amount_version` | id | (expectation_id, version) |
| `finch_deduction` | id | (id, employer_id) · (finch_employer_id, finch_employer_entity_id, hsa_eligible, tax_type) |
| `finch_employee` | id | (id, employer_id) · employee_id · finch_individual_id |
| `finch_employee_identity` | id | finch_individual_id · employee_id |
| `finch_employee_notification` | id | (employee_id, finch_employer_id, notification_type) |
| `finch_employer` | id | (id, employer_id) · (employer_id, session_id) · (connection_id, access_token) · employer_id |
| `finch_employer_eligibility` | id | (id, employer_id) · (finch_employer_id, employer_id) · finch_employer_id |
| `finch_employer_entity` | id | — |
| `finch_enrollment` | id | (finch_employee_id, finch_deduction_id) |
| `finch_payment` | id | (finch_payment_id, pay_group_id) |
| `finch_provider` | id | — |
| `healthsherpa_application` | id | application_id |
| `insured` | id | (period_id, dependent_id) |
| `intercom_sync_report` *(view)* | — | (id) |
| `intercom_sync_status` | id | employer_id · agency_id |
| `leave_of_absence` | id | (employee_id, employer_id) · employee_id |
| `lynx_employer_credentials` | employer_id | — |
| `major_medical_carrier` | id | (name, state) |
| `manually_validated_addresses` | id | formattedAddress |
| `onboarding_period` | id | (id, employer_id) · (id, employee_id, employer_id) |
| `open_enrollment_period` | id | (id, employer_id) |
| `open_enrollment_period_payment_request` | id | request_id · flow_id |
| `payment_method` | id | (onboarding_period_id, employee_id, employer_id) · onboarding_period_id |
| `payment_request` | id | (employer_id, idempotency_key) |
| `payroll_individual` | id | (employer_id, provider_individual_id, provider_employer_entity_id) |
| `payroll_pay_group` | id | (employer_id, provider_employer_entity_id, provider_pay_group_id) · (id, employer_id) |
| `payroll_pay_period` | id | (pay_group_id, start_date) |
| `plan_exclusion_filter` | id | — |
| `plan_exclusion_filter_set` | id | — |
| `plan_year_data` | year | — |
| `prospect_report` *(view)* | — | (id) |
| `provider_preference` | npi, onboarding_period_id | (npi, onboarding_period_id, employer_id, employee_id) |
| `provider_preference_address` | id, provider_npi, onboarding_period_id | — |
| `pull_expectation` | id | (employee_id, plan_year, coverage_month, expected_carrier) |
| `qualifying_life_event` | id | (onboarding_period_id, employee_id, employer_id) · onboarding_period_id |
| `quote` | id | (id, employer_id) · allowance_model_id |
| `quote_age_cost_unit` | id | — |
| `quote_age_cost_unit_v2` | id | — |
| `quote_age_premium_pool_v2` | id | (quote_id, scenario, aggregator, state_code, rating_area, age) |
| `quote_allowance_model_row` | id | (quote_plan_design_id, quote_id, state_code, rating_area, age, employment_type, wage_type) |
| `quote_employee` | id | — |
| `quote_employee_v2` | id | — |
| `quote_plan` | id | — |
| `quote_plan_design` | id | (id, employer_id) · first_contribution_id · second_contribution_id |
| `quote_plan_diversity` | id | — |
| `quote_plan_diversity_v2` | id | — |
| `quote_plan_v2` | id | — |
| `quote_preference` | — | (quote_id, employer_id) · quote_id |
| `quote_presets` | id | (quote_id, preset) |
| `quote_session` | quote_id | quote_id |
| `quote_v2` | id | (id, employer_id) · allowance_model_id |
| `recommendation_data` | plan_year, plan_id, care_level_1_count, care_level_2_count, care_level_3_count, care_level_4_count | — |
| `request_context` | id | — |
| `review_snapshot` | id | — |
| `role` | id | frontegg_id |
| `saved_plan` | id | (plan_id, onboarding_period_id) |
| `sf_medical_plans` | plan_id, plan_year | — |
| `sf_plan_benefit_standardized` | plan_id, plan_year | — |
| `sf_plan_pricing_zip` | plan_id, plan_year, zip_code, age, fips_code | — |
| `sf_premium_age_factor` | state_code, age | — |
| `sf_state_benefit_params` | state_code, plan_year | — |
| `shopping_preference` | onboarding_period_id, type | — |
| `shopping_preference_option` | type, version, value | — |
| `special_case_zip_codes` | id | (zip_code, state_code) |
| `specialty` | id | — |
| `state_appointments` | id | name · code |
| `state_carrier_rate_change` | state_code, carrier_name | — |
| `sub_entity` | id | (name, employer_id) · id |
| `sub_entity_2` | id | (name, employer_id) · id |
| `user` | id | email |
| `user_activation_link` | id | — |
| `user_role` | user_id, role_id | — |
| `webhook_event` | id | (provider, external_event_id) |
| `workflow_execution` | id | graphile_job_id |
| `workflow_rate_limit` | task_type, window_start | — |
| `workflow_task` | id | (task_hash) · graphile_job_id |
| `workflow_task_error` | id | — |
| `zip_county` | zip_code, state_code, county_name, plan_year | — |
| `zip_state` | zip_code, state_code | — |

# Part H — Enum column map (accepted_values targets)

Total enum-typed columns: 242

| table | column | enum | #values | nullable | array |
|---|---|---|---|---|---|
| `actual_transaction` | `category` | `ActualTransactionCategory` | 2 | no |  |
| `actual_transaction` | `payment_method` | `CarrierPaymentMethod` | 2 | yes |  |
| `actual_transaction` | `settlement_status` | `SettlementStatus` | 4 | no |  |
| `actual_transaction` | `vendor` | `PaymentsVendor` | 1 | no |  |
| `actual_transaction_match` | `status` | `ActualTransactionMatchStatus` | 2 | no |  |
| `address_resolve_responses` | `outcome` | `AddressResolveOutcome` | 4 | no |  |
| `address_resolve_responses` | `provider` | `AddressResolveProvider` | 4 | no |  |
| `agency` | `type` | `AgencyType` | 3 | no |  |
| `agent` | `access_level` | `AccessLevel` | 3 | no |  |
| `allowance_model_item` | `family_unit` | `FamilyUnit` | 4 | no |  |
| `benefit` | `application_submission_method` | `ApplicationSubmissionMethod` | 7 | yes |  |
| `benefit` | `auto_pay_status` | `AutoPayStatus` | 4 | yes |  |
| `benefit` | `benefit_type` | `BenefitType` | 11 | no |  |
| `benefit` | `initial_premium_payment_status` | `InitialPremiumPaymentStatus` | 6 | yes |  |
| `benefit` | `payment_method_override` | `CarrierPaymentMethod` | 2 | yes |  |
| `benefit` | `plan_market` | `PlanMarket` | 3 | yes |  |
| `benefit` | `plan_policy_type` | `PolicyType` | 3 | yes |  |
| `benefit` | `plan_self_enroll_type` | `SelfEnrollType` | 3 | no |  |
| `benefit` | `self_report_type` | `SelfReportType` | 4 | no |  |
| `benefit` | `status` | `BenefitStatus` | 9 | no |  |
| `benefit` | `submission_type` | `SubmissionType` | 7 | yes |  |
| `benefit_document` | `type` | `BenefitDocumentType` | 9 | yes |  |
| `benefit_payment_request` | `coverage_month` | `Month` | 12 | yes |  |
| `benefit_payment_request` | `payments_cycle` | `PaymentsCycle` | 3 | no |  |
| `blended_rate_decisions` | `estimate_source` | `BlendedRateEstimateSource` | 3 | yes |  |
| `blended_rate_decisions` | `issuer_match_type` | `BlendedRateIssuerMatchType` | 2 | yes |  |
| `blended_rate_decisions` | `lookup_method` | `BlendedRateLookupMethod` | 3 | no |  |
| `blended_rate_decisions` | `state_code` | `USState` | 51 | no |  |
| `book_transfer_request` | `originating_account_type` | `AccountType` | 2 | no |  |
| `book_transfer_request` | `receiving_account_type` | `AccountType` | 2 | no |  |
| `carrier_info` | `auto_pay` | `PaymentResponsibility` | 2 | yes |  |
| `carrier_info` | `initial_payment` | `PaymentResponsibility` | 2 | yes |  |
| `carrier_info` | `state` | `USState` | 51 | no |  |
| `carrier_vendor_mapping` | `vendor` | `PaymentsVendor` | 1 | no |  |
| `census_employee` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `census_employee` | `family_unit` | `FamilyUnit` | 4 | no |  |
| `census_employee` | `wage_type` | `WageType` | 2 | yes |  |
| `census_employee_v2` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `census_employee_v2` | `family_unit` | `FamilyUnit` | 4 | no |  |
| `census_employee_v2` | `wage_type` | `WageType` | 2 | yes |  |
| `census_plan` | `network_type` | `NetworkType` | 5 | no |  |
| `census_plan_v2` | `network_type` | `NetworkType` | 5 | no |  |
| `class_definition_item` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `class_definition_item` | `state` | `USState` | 51 | yes |  |
| `class_definition_item` | `wage_type` | `WageType` | 2 | yes |  |
| `contribution_request` | `status` | `ContributionRequestStatus` | 4 | no |  |
| `cost_preference` | `attribute` | `CostAttribute` | 3 | no |  |
| `cost_preference` | `direction` | `PreferredDirection` | 2 | no |  |
| `cost_preference` | `importance` | `CostPreferenceImportance` | 3 | no |  |
| `decision_factors_preference` | `decision_factors` | `DecisionFactors` | 5 | no | yes |
| `dependent` | `citizenship_status` | `CitizenshipStatus` | 3 | yes |  |
| `dependent` | `county_resolution_source` | `CountyResolutionSource` | 5 | yes |  |
| `dependent` | `gender` | `Gender` | 2 | yes |  |
| `dependent` | `state` | `USState` | 51 | yes |  |
| `dependent` | `sub_type` | `InsuredSubtype` | 4 | yes |  |
| `dependent` | `type` | `InsuredType` | 3 | no |  |
| `dw_blended_rate_decision_versions` | `estimate_source` | `BlendedRateEstimateSource` | 3 | yes |  |
| `dw_blended_rate_decision_versions` | `issuer_match_type` | `BlendedRateIssuerMatchType` | 2 | yes |  |
| `dw_blended_rate_decision_versions` | `lookup_method` | `BlendedRateLookupMethod` | 3 | no |  |
| `dw_blended_rate_decision_versions` | `state_code` | `USState` | 51 | no |  |
| `employee` | `citizenship_status` | `CitizenshipStatus` | 3 | yes |  |
| `employee` | `county_resolution_source` | `CountyResolutionSource` | 5 | yes |  |
| `employee` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `employee` | `gender` | `Gender` | 2 | yes |  |
| `employee` | `marital_status` | `MaritalStatus` | 3 | yes |  |
| `employee` | `state` | `USState` | 51 | yes |  |
| `employee` | `wage_type` | `WageType` | 2 | yes |  |
| `employee_change_log` | `county_resolution_source` | `CountyResolutionSource` | 5 | yes |  |
| `employee_change_log` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `employee_change_log` | `state` | `USState` | 51 | yes |  |
| `employee_change_log` | `wage_type` | `WageType` | 2 | yes |  |
| `employee_payments_account` | `creation_status` | `CreationStatus` | 3 | no |  |
| `employee_payments_alert` | `type` | `EmployeePaymentsAlertType` | 5 | no |  |
| `employee_report` | `auto_pay_status` | `AutoPayStatus` | 4 | yes |  |
| `employee_report` | `benefit_eligibility` | `EmploymentStatus` | 7 | yes |  |
| `employee_report` | `benefit_family_unit` | `FamilyUnit` | 4 | yes |  |
| `employee_report` | `benefit_status` | `BenefitStatus` | 9 | yes |  |
| `employee_report` | `benefit_type` | `BenefitType` | 11 | yes |  |
| `employee_report` | `employee_role` | `EmployeeRole` | 3 | no |  |
| `employee_report` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `employee_report` | `enrollment_status` | `EnrollmentStatus` | 13 | no |  |
| `employee_report` | `enrollment_submission_type` | `SubmissionType` | 7 | yes |  |
| `employee_report` | `enrollment_type` | `OnboardingType` | 2 | yes |  |
| `employee_report` | `initial_premium_payment_status` | `InitialPremiumPaymentStatus` | 6 | yes |  |
| `employee_report` | `invitation_status` | `UserStatus` | 4 | no |  |
| `employee_report` | `medical_plan_market` | `PlanMarket` | 3 | yes |  |
| `employee_report` | `medical_plan_type` | `PolicyType` | 3 | yes |  |
| `employee_report` | `medical_self_enroll` | `SelfEnrollType` | 3 | yes |  |
| `employee_report` | `medical_self_report_type` | `SelfReportType` | 4 | yes |  |
| `employee_report` | `payment_method_type` | `EmployeePaymentMethodType` | 3 | yes |  |
| `employee_report` | `qle_type` | `QualifyingLifeEventType` | 7 | yes |  |
| `employee_report` | `state` | `USState` | 51 | yes |  |
| `employer` | `business_entity_type` | `BusinessEntityType` | 10 | yes |  |
| `employer` | `payroll_cycle` | `PayrollCycle` | 6 | yes |  |
| `employer` | `signup_status` | `EmployerSignupStatus` | 6 | no |  |
| `employer` | `state_of_incorporation` | `USState` | 51 | yes |  |
| `employer` | `tax_id_type` | `TaxIdType` | 2 | yes |  |
| `employer` | `waiting_period` | `WaitingPeriod` | 4 | yes |  |
| `employer_contacts` | `type` | `ContactType` | 4 | no |  |
| `employer_document` | `type` | `DocumentType` | 5 | no |  |
| `employer_payments_account` | `creation_status` | `CreationStatus` | 3 | no |  |
| `enrollment_activity_log` | `activity` | `EnrollmentActivity` | 31 | no |  |
| `enrollment_activity_log` | `benefit_types` | `BenefitType` | 11 | no | yes |
| `enrollment_activity_log` | `submission_type` | `SubmissionType` | 7 | yes |  |
| `enrollment_report` | `aor_cancellation_status` | `AorCancellationStatus` | 3 | no |  |
| `enrollment_report` | `auto_pay_status` | `AutoPayStatus` | 4 | yes |  |
| `enrollment_report` | `benefit_status` | `BenefitStatus` | 9 | yes |  |
| `enrollment_report` | `carrier_request_status` | `CarrierRequestStatus` | 3 | yes |  |
| `enrollment_report` | `employee_state` | `USState` | 51 | yes |  |
| `enrollment_report` | `enrollment_status` | `EnrollmentStatus` | 13 | no |  |
| `enrollment_report` | `enrollment_tags` | `EnrollmentTag` | 9 | no | yes |
| `enrollment_report` | `family_unit` | `FamilyUnit` | 4 | yes |  |
| `enrollment_report` | `initial_premium_payment_status` | `InitialPremiumPaymentStatus` | 6 | yes |  |
| `enrollment_report` | `metal_level` | `MetalLevel` | 6 | yes |  |
| `enrollment_report` | `plan_market` | `PlanMarket` | 3 | yes |  |
| `enrollment_report` | `policy_type` | `PolicyType` | 3 | yes |  |
| `enrollment_report` | `qle_type` | `QualifyingLifeEventType` | 7 | yes |  |
| `enrollment_report` | `renewal_status` | `RenewalStatus` | 4 | no |  |
| `enrollment_report` | `self_enroll` | `SelfEnrollType` | 3 | yes |  |
| `event` | `source` | `EventSource` | 2 | no |  |
| `expected_amount_version` | `source` | `ExpectedAmountSource` | 2 | no |  |
| `finch_deduction` | `action` | `FinchDeductionAction` | 3 | no |  |
| `finch_deduction` | `hsa_eligible` | `FinchDeductionHsaEligible` | 3 | no |  |
| `finch_deduction` | `job_status` | `FinchJobStatus` | 6 | no |  |
| `finch_deduction` | `tax_type` | `FinchDeductionTax` | 3 | no |  |
| `finch_employee` | `pay_frequency` | `FinchPayFrequency` | 9 | yes |  |
| `finch_employee` | `sync_type` | `FinchEmployeeSyncType` | 3 | no |  |
| `finch_employee_notification` | `notification_type` | `FinchNotificationType` | 4 | no |  |
| `finch_employer` | `connection_status` | `FinchConnectionStatus` | 5 | yes |  |
| `finch_employer` | `deduction_action` | `FinchDeductionAction` | 3 | no |  |
| `finch_employer` | `integration_type` | `FinchIntegrationType` | 2 | yes |  |
| `finch_employer` | `last_sync_status` | `FinchSyncStatus` | 2 | yes |  |
| `finch_employer` | `pay_group_behavior` | `FinchPayGroupBehavior` | 3 | no |  |
| `finch_enrollment` | `action` | `FinchEnrollmentAction` | 2 | no |  |
| `finch_enrollment` | `job_status` | `FinchJobStatus` | 6 | no |  |
| `finch_enrollment` | `unenroll_reason` | `FinchUnenrollReason` | 5 | yes |  |
| `finch_payment` | `pay_frequency` | `FinchPayFrequency` | 9 | yes |  |
| `finch_provider` | `deduction_action` | `FinchDeductionAction` | 3 | no |  |
| `healthsherpa_application` | `payment_status` | `HealthSherpaPaymentStatus` | 6 | yes |  |
| `healthsherpa_application` | `policy_status` | `HealthSherpaPolicyStatus` | 8 | yes |  |
| `healthsherpa_application` | `provider` | `HealthSherpaApplicationProvider` | 2 | no |  |
| `healthsherpa_application` | `status` | `HealthSherpaApplicationStatus` | 9 | no |  |
| `insured` | `anticipated_care_level` | `AnticipatedCareLevel` | 9 | yes |  |
| `insured` | `citizenship_status` | `CitizenshipStatus` | 3 | yes |  |
| `insured` | `county_resolution_source` | `CountyResolutionSource` | 5 | yes |  |
| `insured` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `insured` | `gender` | `Gender` | 2 | yes |  |
| `insured` | `marital_status` | `MaritalStatus` | 3 | yes |  |
| `insured` | `state` | `USState` | 51 | yes |  |
| `insured` | `subtype` | `InsuredSubtype` | 4 | yes |  |
| `insured` | `type` | `InsuredType` | 3 | no |  |
| `insured` | `wage_type` | `WageType` | 2 | yes |  |
| `intercom_sync_report` | `aor_cancellation_status` | `AorCancellationStatus` | 3 | no |  |
| `intercom_sync_report` | `auto_pay_status` | `AutoPayStatus` | 4 | yes |  |
| `intercom_sync_report` | `carrier_request_status` | `CarrierRequestStatus` | 3 | yes |  |
| `intercom_sync_report` | `employee_citizenship_status` | `CitizenshipStatus` | 3 | yes |  |
| `intercom_sync_report` | `employee_employment_type` | `EmploymentType` | 2 | yes |  |
| `intercom_sync_report` | `employee_marital_status` | `MaritalStatus` | 3 | yes |  |
| `intercom_sync_report` | `employee_wage_type` | `WageType` | 2 | yes |  |
| `intercom_sync_report` | `enrollment_status` | `EnrollmentStatus` | 13 | no |  |
| `intercom_sync_report` | `enrollment_tags` | `EnrollmentTag` | 9 | no | yes |
| `intercom_sync_report` | `family_unit` | `FamilyUnit` | 4 | yes |  |
| `intercom_sync_report` | `initial_premium_payment_status` | `InitialPremiumPaymentStatus` | 6 | yes |  |
| `intercom_sync_report` | `payment_method_status` | `EmployeePaymentMethodStatus` | 4 | yes |  |
| `intercom_sync_report` | `plan_market` | `PlanMarket` | 3 | yes |  |
| `intercom_sync_report` | `qle_type` | `QualifyingLifeEventType` | 7 | yes |  |
| `intercom_sync_report` | `self_enroll` | `SelfEnrollType` | 3 | yes |  |
| `intercom_sync_report` | `self_report_type` | `SelfReportType` | 4 | yes |  |
| `intercom_sync_report` | `submission_type` | `SubmissionType` | 7 | yes |  |
| `leave_of_absence` | `payment_by` | `PaymentBy` | 2 | yes |  |
| `major_medical_carrier` | `payment_method` | `CarrierPaymentMethod` | 2 | no |  |
| `manually_validated_addresses` | `state_code` | `USState` | 51 | no |  |
| `onboarding_period` | `enrollment_tags` | `EnrollmentTag` | 9 | no | yes |
| `onboarding_period` | `type` | `OnboardingPeriodType` | 2 | no |  |
| `open_enrollment_period` | `payment_method` | `OEPPaymentMethod` | 2 | yes |  |
| `open_enrollment_period_payment_request` | `coverage_month` | `Month` | 12 | yes |  |
| `open_enrollment_period_payment_request` | `payments_cycle` | `PaymentsCycle` | 3 | no |  |
| `payment_method` | `status` | `EmployeePaymentMethodStatus` | 4 | no |  |
| `payment_request` | `status` | `PaymentRequestStatus` | 8 | no |  |
| `payment_request` | `type` | `PaymentRequestType` | 4 | no |  |
| `payroll_pay_group` | `pay_frequency` | `PayrollPayFrequency` | 4 | no |  |
| `payroll_pay_period` | `source` | `PayrollPayPeriodSource` | 2 | no |  |
| `provider_preference_address` | `state` | `USState` | 51 | no |  |
| `pull_expectation` | `coverage_month` | `Month` | 12 | no |  |
| `pull_expectation` | `payment_type` | `PaymentsCycle` | 3 | no |  |
| `qualifying_life_event` | `type` | `QualifyingLifeEventType` | 7 | no |  |
| `quote_age_premium_pool_v2` | `aggregator` | `Aggregator` | 3 | no |  |
| `quote_age_premium_pool_v2` | `scenario` | `Scenario` | 14 | no |  |
| `quote_allowance_model_row` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `quote_allowance_model_row` | `wage_type` | `WageType` | 2 | yes |  |
| `quote_employee` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `quote_employee` | `family_unit` | `FamilyUnit` | 4 | no |  |
| `quote_employee` | `metal_level` | `MetalLevel` | 6 | no |  |
| `quote_employee` | `wage_type` | `WageType` | 2 | yes |  |
| `quote_employee_v2` | `employment_type` | `EmploymentType` | 2 | yes |  |
| `quote_employee_v2` | `family_unit` | `FamilyUnit` | 4 | no |  |
| `quote_employee_v2` | `metal_level` | `MetalLevel` | 6 | no |  |
| `quote_employee_v2` | `scenario` | `Scenario` | 14 | no |  |
| `quote_employee_v2` | `wage_type` | `WageType` | 2 | yes |  |
| `quote_plan` | `network_type` | `NetworkType` | 5 | no |  |
| `quote_plan_design` | `contribution_group` | `ContributionGroup` | 2 | yes |  |
| `quote_plan_design` | `contribution_mode` | `ContributionMode` | 2 | yes |  |
| `quote_plan_design` | `contribution_type` | `AllowanceUnit` | 2 | yes |  |
| `quote_plan_design` | `entity_view` | `EntityView` | 2 | yes |  |
| `quote_plan_design` | `family_unit` | `FamilyUnitDistribution` | 2 | yes |  |
| `quote_plan_design` | `geo_distribution` | `GeoDistribution` | 3 | yes |  |
| `quote_plan_design` | `priorities` | `QuotePlanPriorities` | 6 | no | yes |
| `quote_plan_design` | `scenario` | `Scenario` | 14 | yes |  |
| `quote_plan_design` | `selected_preset` | `ContributionPreset` | 6 | yes |  |
| `quote_plan_design` | `work_status` | `ClassType` | 3 | yes |  |
| `quote_plan_diversity` | `metal_level` | `MetalLevel` | 6 | yes |  |
| `quote_plan_diversity` | `network_type` | `NetworkType` | 5 | no |  |
| `quote_plan_diversity` | `plan_type` | `PlanType` | 2 | no |  |
| `quote_plan_diversity_v2` | `metal_level` | `MetalLevel` | 6 | yes |  |
| `quote_plan_diversity_v2` | `network_type` | `NetworkType` | 5 | no |  |
| `quote_plan_diversity_v2` | `plan_type` | `PlanType` | 2 | no |  |
| `quote_plan_v2` | `metal_level` | `MetalLevel` | 6 | yes |  |
| `quote_plan_v2` | `network_type` | `NetworkType` | 5 | no |  |
| `quote_preference` | `contribution_type` | `AllowanceUnit` | 2 | no |  |
| `quote_preference` | `family_units` | `FamilyUnitDistribution` | 2 | no |  |
| `quote_preference` | `geo_distribution` | `GeoDistribution` | 3 | no |  |
| `quote_preference` | `selected_class` | `ClassType` | 3 | yes |  |
| `quote_presets` | `preset` | `ContributionPreset` | 6 | no |  |
| `quote_presets` | `unit` | `AllowanceUnit` | 2 | no |  |
| `quote_v2` | `renewal_baseline_scenario` | `Scenario` | 14 | yes |  |
| `quote_v2` | `setup_status` | `SetupStatus` | 3 | no |  |
| `review_snapshot` | `type` | `ReviewSnapshotType` | 3 | no |  |
| `role` | `id` | `Role` | 9 | no |  |
| `sf_medical_plans` | `network_type` | `NetworkType` | 5 | no |  |
| `sf_plan_benefit_standardized` | `network_type` | `NetworkType` | 5 | no |  |
| `shopping_preference` | `type` | `ShoppingPreferenceType` | 1 | no |  |
| `shopping_preference_option` | `type` | `ShoppingPreferenceType` | 1 | no |  |
| `state_appointments` | `auto_pay` | `PaymentResponsibility` | 2 | yes |  |
| `state_appointments` | `code` | `USState` | 51 | no |  |
| `state_appointments` | `initial_payment` | `PaymentResponsibility` | 2 | yes |  |
| `state_carrier_rate_change` | `state_code` | `USState` | 51 | no |  |
| `user` | `status` | `UserStatus` | 4 | no |  |
| `user_activation_link` | `redirect_to` | `ActivationRedirectUrlOptions` | 4 | no |  |
| `user_role` | `role_id` | `Role` | 9 | no |  |
| `workflow_execution` | `status` | `WorkflowStatus` | 5 | no |  |
| `workflow_task` | `status` | `WorkflowTaskStatus` | 5 | no |  |
| `workflow_task_error` | `severity` | `ErrorSeverity` | 3 | no |  |
