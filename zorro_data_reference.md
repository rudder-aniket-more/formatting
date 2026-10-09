# Zorro Schema, Business Rules & Data Contracts

🌻 The single reference for the `zorro-ts` database: what every table and column
is, what constrains it, and which rules a warehouse model must replicate.

**Generated from `main` at commit `f9cd6df3a` (2026-10-08).** Schema facts
(columns, types, nullability, defaults, keys, indexes, cascades, join
cardinality) are extracted mechanically from
`apps/monolith/prisma/schema/*.prisma` and are complete. Business rules are
hand-verified against migrations, SQL functions and service code; where a rule
lives only in application code and not the database, that is stated.

**Scope:** 132 tables/views · 1,755 columns · 115 enums · 242 enum-typed columns
· 139 foreign keys · 7 CHECK constraints · 12 schema files.

> [`entities.md`](./entities.md) and [`business-flows.md`](./business-flows.md)
> predate this file (Dec 2025) and have drifted — prefer this file where they
> disagree. [`payments.md`](./payments.md) and
> [`user-roles.md`](./user-roles.md) remain current for their topics.

## Contents

| Part | Content | Use for |
|---|---|---|
| **1** | Notation | reading the tables below |
| **2** | Rules the database enforces | the only hard guarantees |
| **3** | Join cardinality — 1:1 vs fan-out | deciding where dedup is needed |
| **4** | Grain and uniqueness | `unique` tests, natural keys |
| **5** | Foreign keys and delete behaviour | `relationships` tests, CDC strategy |
| **6** | Accepted values | `accepted_values` tests |
| **7** | Derived statuses | logic a model must replicate exactly |
| **8** | Rules enforced only in app code | bespoke assertions |
| **9** | Global column conventions | money, dates, naming traps |
| **10** | Per-table column reference | every table, every column |
| **11** | Appendix — all enum values | |

Four of dbt's generic tests map directly onto Parts 4–6; Part 3 tells you where
a join preserves grain; Part 7 is what dbt cannot derive from the schema.

---

# Part 1 — Notation

| Notation | Meaning |
|---|---|
| **PK** | primary key (`@id`) |
| **UQ** | column-level unique (`@unique`) |
| `NOT NULL` / `null` | nullability |
| def `x` | database default |
| `auto` | `@updatedAt` — written on every update |
| `Uuid`, `Decimal(12, 2)` | explicit Postgres type (`@db.*`) |
| (enum) | enum column — values in Part 11 |
| `-> Model` | relation, not a stored column |

Column names shown are the **database** names (snake_case). The Prisma model
field name differs where `@map()` is used — the model name appears under each
heading in Part 10.

---

# Part 2 — Rules the database enforces

These are the only hard guarantees. Everything in Part 8 can be violated by a
direct write.

## 2.1 CHECK constraints — all 7 in the entire database

| Table | Rule | Meaning |
|---|---|---|
| `benefit` | `monthly_auto_pay_date BETWEEN 1 AND 28` | auto-pay never lands on a day some months lack |
| `benefit` | `NOT (is_monthly_auto_pay_date_end_of_month = true AND monthly_auto_pay_date IS NOT NULL)` | end-of-month and a fixed day are mutually exclusive |
| `employer` | `is_employment_sync_on = false OR is_payroll_sync_on = true` | employment sync requires payroll sync |
| `enrollment_activity_log` | `submission_type = 'AUTOMATIC' OR (performed_by_id IS NOT NULL AND performed_by_email IS NOT NULL)` | every non-automatic action must name an actor |
| `qualifying_life_event` | `type <> 'OTHER' OR otherReason IS NOT NULL` | QLE type `OTHER` requires a free-text reason |
| `quote_plan_design` | `work_status IS NULL`, or `= 'NONE'` with exactly one contribution, or `<> 'NONE'` with both | a discriminated quote needs two contribution rates |
| `blended_rate_decisions` | `lookup_method <> 'ESTIMATED' OR rate_change_pct IS NOT NULL` | an estimated rate must carry its percentage |

Note what is **absent**: no CHECK enforces date ordering (`effective_from <=
effective_until`), non-negative money, or valid status transitions. All of that
is application-level — see Part 8.

## 2.2 Partial unique indexes

Only two, both on `payroll_individual` — uniqueness applies *only to matched
rows*, so many unmatched candidates may coexist:

```sql
CREATE UNIQUE INDEX "payroll_individual_matched_employee_id_key"
  ON "payroll_individual"("employee_id") WHERE "is_matched";
CREATE UNIQUE INDEX "payroll_individual_matched_provider_individual_id_key"
  ON "payroll_individual"("employer_id","provider_individual_id") WHERE "is_matched";
```

## 2.3 Compound foreign keys — the multi-tenancy pattern

Several tables denormalise `employer_id` / `employee_id` and reference parents on
compound keys — e.g. `insured.period_id + employee_id + employer_id` →
`onboarding_period(id, employee_id, employer_id)`. That is why
`onboarding_period` carries `UNIQUE (id, employer_id)` and
`UNIQUE (id, employee_id, employer_id)`: they exist as **FK targets**, not as
business keys.

**Consequence:** a `@@unique` containing `id` tells you nothing about business
cardinality. `onboarding_period` has two such constraints and still permits many
rows per employee per enrollment period. Do not build a uniqueness test on them —
it will always pass and will hide the absence of any real constraint.

The denormalised tenant columns also back Row-Level Security, which is enabled,
so query results depend on the connecting role's context.

## 2.4 The join table with no model

`_benefit_to_insured` exists in no `.prisma` file (Prisma implicit many-to-many):

```sql
"A" UUID -> benefit(id)   ON DELETE CASCADE
"B" UUID -> insured(id)   ON DELETE CASCADE
PRIMARY KEY ("A","B")
```

Quote the column names — unquoted `A`/`B` will not resolve. It carries **no
`employer_id`**, so it has no RLS policy of its own and must be filtered through
`benefit` or `insured`. It answers "who is covered by this benefit", and it
drives `family_unit`, which selects the allowance tier — so **junction rows
determine money** (Part 7.7).

---
# Part 3 — Join cardinality: where you can skip dedup

**The most actionable section.** A unique constraint that *covers the
foreign-key columns* guarantees at most one child row per parent — so the join
is 1:1 and needs no `row_number()`, `qualify`, `distinct`, or pre-aggregation.
Where no such constraint exists, the join fans out.

Derived mechanically: a relationship is 1:1 when some UNIQUE set on the child
table is a **subset** of its FK columns.

**Of 139 declared relationships: 32 are 1:1, 107 fan out.**

## How to use this

- **Safe-join list** → join directly, select child columns inline, no window
  functions.
- **Fan-out list** → the parent grain is *not* preserved. Aggregate
  (`count`, `sum`, `bool_or`) or pick one row with an explicit `ORDER BY`. Never
  assume "there's only one in practice" — that is exactly the assumption
  `onboarding_period` and `allowance_model_item` break.

## Worth noting in the safe list

- **`payment_request` → its four subtype tables** (`contribution_request`,
  `allocation_request`, `deallocation_request`, `book_transfer_request`) are all
  1:1 on `request_id`. Table-per-subtype: join all four and coalesce, don't rank.
- **`employee` → `leave_of_absence`** is 1:1 — at most **one LOA row per
  employee, ever**, not one per leave. Historical leaves are not retained.
- **`onboarding_period` → `qualifying_life_event`** is 1:1, so QLE fields join
  inline. That is why `is_qle_enrollment` is just `qle.id IS NOT NULL`.
- **`user` → `employee`** and **`user` → `agent`** are both 1:1.
- **`sf_medical_plans` → `sf_plan_benefit_standardized`** share the composite key
  `(plan_id, plan_year)` — a clean 1:1 spine for plan models.

## Worth noting in the fan-out list

- **`employer` → `allowance_model`** fans out: one row **per year**. Joining
  without a year predicate silently multiplies rows by the number of plan years.
- **`allowance_model` → `allowance_model_item`** fans out with **no uniqueness at
  all**. The rate grid can hold overlapping age bands or duplicate
  `(class, family_unit)` rows, so resolving an allowance needs explicit
  tie-breaking.
- **`onboarding_period` → `benefit`** fans out — multiple products per period.
- **`onboarding_period` → `insured`** fans out — one row per covered person.
- **`employee` → `onboarding_period`** fans out — QLEs create extra periods.

## The "unique after a filter" pattern

Several fan-out joins become effectively 1:1 once you add the predicate the
business rule implies. This is where a constraint buys you a dedup-free model —
*if you filter correctly*:

| Parent → child | Add predicate | Then unique because |
|---|---|---|
| `employer` → `allowance_model` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `employer` → `allowance_model_v2` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `employer` → `class_definition` | `year = :plan_year` | `UNIQUE (employer_id, year)` |
| `onboarding_period` → `insured` | `dependent_id = :id` | `UNIQUE (period_id, dependent_id)` |
| `pull_expectation` → `expected_amount_version` | `version = max(version)` | `UNIQUE (expectation_id, version)` |
| `onboarding_period` → `insured` | `type = 'EMPLOYEE'` | **app-enforced only — not a DB guarantee** |
| `employee` → `onboarding_period` | `is_active` | **app-enforced only — not a DB guarantee** |
| `onboarding_period` → `benefit` | `benefit_type = 'MAJOR_MEDICAL'` | **no constraint at all — see below** |

**The last three are traps.** Filtering `insured` to `type = 'EMPLOYEE'`, or
`onboarding_period` to `is_active`, *looks* like it yields one row — and does in
healthy data — but neither is backed by a constraint.

The major-medical filter is weakest: nothing prevents several `MAJOR_MEDICAL`
benefits in one period. The product's own `intercom_sync_report` handles it with
`ORDER BY created_at, id LIMIT 1`, and a migration comment records that
production had zero such periods *as verified on 2026-07-29* — an observed fact
with a date on it, not an invariant. Rank explicitly.


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

# Part 4 — Grain and uniqueness

## 4.1 Three kinds of UNIQUE set — not equally useful

**Business keys — worth testing.** `(employer_id, year)`, `(name, state)`,
`(zip_code, state_code)`. These express a real rule and make good natural-key
tests.

**Vendor idempotency keys — useful for dedup.**
`(vendor, vendor_correlation_id, vendor_account_id)`,
`(provider, external_event_id)`, `(employer_id, idempotency_key)`. These
guarantee at-most-once ingestion of an external event.

**Compound-FK targets — not business rules.** Any UNIQUE set containing `id`;
see Part 2.3.

## 4.2 The business-key rules worth testing

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
| `leave_of_absence` | `employee_id` | **at most one LOA per employee, ever** |
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

## 4.3 Composite primary keys — the natural grain

These have **no surrogate `id`**; the PK *is* the business grain, making them the
cleanest sources:

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
| `dw_blended_rate_decision_versions` | `(state_code, hios, year, valid_from)` — **SCD2** |
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
dimension** — `valid_from` / `valid_to` (`Timestamptz`), with `valid_to IS NULL`
marking the current row. Do not re-snapshot it; filter it. Its sibling
`blended_rate_decisions` holds only current state.

## 4.4 Tables with NO uniqueness beyond the surrogate key

These accept unlimited rows per logical entity. Any model assuming one row per
parent here will silently fan out:

`allowance_model_item` · `agency` · `employer_contacts` ·
`employer_beneficial_owner` · `enrollment_activity_log` · `enrollment_comment` ·
`eligibility_history` · `employee_change_log` · `benefit_document` ·
`electronic_signature` · `review_snapshot` · `census_employee(_v2)` ·
`census_plan(_v2)` · `quote_employee(_v2)` · `quote_plan(_v2)` ·
`quote_age_cost_unit(_v2)` · `quote_plan_diversity(_v2)` ·
`contribution_by_class` · `audit` · `data_change_audit` · `event` ·
`address_resolve_responses` · `request_context` · `workflow_task_error` ·
`plan_exclusion_filter` · `specialty`

Two to watch: **`allowance_model_item`** (a duplicate silently changes an
employee's allowance) and **`eligibility_history`** / **`employee_change_log`**
(append-only histories — expect many rows per employee and dedupe deliberately).

## 4.5 Full grain matrix

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

---

# Part 5 — Foreign keys and delete behaviour

139 declared relations, each a `relationships` test candidate.

## 5.1 Delete behaviour

| Behaviour | Count | Warehouse implication |
|---|---|---|
| `Cascade` | **92** | child rows vanish with the parent; incremental models need full-refresh or delete-detection |
| `SetNull` | some | FK becomes `NULL`, row survives — e.g. `onboarding_period.enrollment_team_id`, `insured.dependent_id` |
| default (`Restrict`) | rest | delete blocked while children exist |

Cascades by schema: administration 32 · prospects 22 · integrations 12 ·
shopping 11 · payments 7 · payroll 3 · infra 2 · schema 2 · payments-v1 1.

**There is effectively no soft delete.** Only `enrollment_comment.deleted_at`
exists. Deleting an employer removes its employees, periods, benefits, insured,
quotes and payment records. There is no `deleted_at` filter to remember and no
recovery — and snapshot-based models cannot detect hard deletes from the source
alone. Plan for periodic full refreshes on cascade-heavy trees, or rely on CDC.

## 5.2 Compound foreign keys

`relationships` tests must match on **all** columns, not just the obvious one:

| Child | FK columns | Parent |
|---|---|---|
| `insured` | `period_id, employee_id, employer_id` | `onboarding_period` |
| `benefit` | `period_id, employee_id, employer_id` | `onboarding_period` |
| `qualifying_life_event` | `onboarding_period_id, employee_id, employer_id` | `onboarding_period` |
| `allocation_request` | `employee_id, employer_id` | `employee_payments_account` |
| `employee_payments_account` | `employee_id, employer_id, user_id` | `employee` |

## 5.3 Full foreign-key matrix

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

---

# Part 6 — Accepted values

**242 enum-typed columns** across **115 enums**. Each is an `accepted_values`
test; full value lists are in Part 11, the per-column map below.

Two cautions:

**Enums change, and values get removed.** Postgres cannot drop an enum value, so
removals are done by rename-and-swap. `SubmissionType` lost `JOINT_SESSION` on
2026-09-23 (PR #9122): historical rows were rewritten to `BY_OPERATOR` with a new
`is_impersonated = true` flag, and two new values `BY_EMPLOYER_ADMIN` /
`BY_AGENT` were added with **no history behind them**. So hard-coded
`accepted_values` lists need review when the schema changes, and a time series
split by such a column shows a discontinuity at the migration date rather than
real behaviour change.

**Array-typed enum columns** need `accepted_values` applied per element, not to
the column: `enrollment_tags` (`EnrollmentTag[]`), `benefit_types`
(`BenefitType[]`), `decision_factors` (`DecisionFactors[]`), `priorities`
(`QuotePlanPriorities[]`).

## 6.1 Enum column map

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

---

# Part 7 — Derived statuses

None of these are stored. A model that recomputes them by hand will diverge from
the product. **Prefer calling the database functions or selecting from the
views.**

## 7.1 `enrollment_status` — the most important one

`calculate_enrollment_status(onboarding_period_id, is_active, last_user_login,
onboarding_until, benefit_id, is_waived, benefit_effective_from,
benefit_effective_until, benefit_status, today)` → `EnrollmentStatus`
(`IMMUTABLE`, so safe in indexes and `GROUP BY`).

Evaluated **in order** — first match wins:

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

Five consequences:

1. **It takes `today`.** The same unchanged row returns a different status
   tomorrow. A table materialisation silently freezes a date; an incremental
   model freezes a *different* date per partition. Pass an explicit as-of date or
   re-derive on read.
2. **Ordering is load-bearing.** Waiver beats activity, activity beats benefit
   status. A `CASE` in another order gives different answers.
3. **Buckets are coarser than `BenefitStatus`** — nine benefit statuses collapse
   into thirteen enrollment statuses that also encode waiver and activity. The
   mapping is many-to-one and not invertible.
4. **`benefit_status` and `enrollment_status` can disagree and both be right** —
   a benefit sits in `ACTIVE` while showing `COVERAGE_ENDED` between
   `effective_until` passing and the nightly cron flipping it to `ENDED`.
5. **`ELECTION_ACTIVE` is in the enum but unreachable** from this function — only
   the two `_STARTED` / `_HAS_NOT_STARTED` variants are produced, so expect zero
   rows. Application code still lists it in membership arrays (e.g.
   `NOT_SUBMITTED_ELECTION_STATUSES`) for exhaustiveness, so keep it in any
   `accepted_values` list, but treat a non-zero count as a red flag.

## 7.2 `benefit_eligibility` (`EmploymentStatus`)

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

Branch 1 requires **both** expired eligibility and a past termination date — an
employee with a termination date but unexpired eligibility is still `ELIGIBLE_*`.
The function **can return `NULL`**.

## 7.3 `renewal_status`

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
(name + date of birth). A genuine fuzzy-identity rule — reimplementing it loosely
will misclassify renewals. Major-medical only, and requires coverage contiguous
to the day.

## 7.4 `aor_cancellation_status`

| Condition | Result |
|---|---|
| OEP `is_aor_changed` AND **not** special enrollment AND `benefit.is_aor_confirmed_by_employee` | `CONFIRMED_BY_EMPLOYEE` |
| OEP `is_aor_changed` AND not special enrollment, otherwise | `AWAITING_EMPLOYEE` |
| otherwise | `NOT_APPLICABLE` |

## 7.5 `carrier_request_status`

| # | Condition | Result |
|---|---|---|
| 1 | `is_waived`, or no benefit | `NOT_APPLICABLE` |
| 2 | employee state unknown, or no matching `carrier_info` | **`NULL`** |
| 3 | carrier has special requests AND `is_special_requests_confirmed` | `CONFIRMED_BY_EMPLOYEE` |
| 4 | carrier has special requests, otherwise | `AWAITING_EMPLOYEE` |
| 5 | otherwise | `NOT_APPLICABLE` |

Branch 2 is the trap: `NULL` means *"cannot determine"*, semantically different
from `NOT_APPLICABLE`. `carrier_info` joins on `(name, state)`, so a carrier-name
mismatch produces nulls, not falses.

## 7.6 `payment_method_type` (`employee_report`)

| Condition | Result |
|---|---|
| no period, no benefit, or waived | `NULL` |
| `self_pay_amount` is `NULL` or `0` | `ZORRO_PAY` |
| `self_pay_amount = premium` | `SELF_PAY` |
| `0 < self_pay_amount < premium` | `COMBINED` |
| fallback | `ZORRO_PAY` |

## 7.7 `family_unit`

Derived from `_benefit_to_insured`, **not** from the period's insured rows:

| Condition (insured joined through the junction) | Result |
|---|---|
| has spouse AND has child | `FAMILY` |
| has spouse | `EMPLOYEE_SPOUSE` |
| has child | `EMPLOYEE_CHILD` |
| `count(*) > 0` | `EMPLOYEE_ONLY` |
| else | `NULL` |

It was a stored `benefit.family_unit` column until Feb 2026 (PR #7134), then
dropped and inverted into this derivation — because a four-value label cannot
express *which* people are covered. **It selects the allowance tier, so junction
rows determine money.**

**The two views disagree on the final branch.** `enrollment_report` ends
`WHEN count(*) > 0 THEN 'EMPLOYEE_ONLY' ELSE NULL`; `employee_report` ends plain
`ELSE 'EMPLOYEE_ONLY'`. A benefit covering nobody reads `NULL` in one and
`EMPLOYEE_ONLY` in the other. Reconcile deliberately if you union them.

## 7.8 Simple derived flags

| Column | Rule |
|---|---|
| `is_qle_enrollment` | `qle.id IS NOT NULL` |
| `has_user_logged_in` | `op.last_user_login IS NOT NULL` |
| `has_previous_eligibility` | `EXISTS` row in `eligibility_history` |
| `has_qualifying_benefit` | `COALESCE(status IN (ENROLLMENT_CONFIRMED, ACTIVE, ENDED), FALSE)` |
| `enrollment_type` | `SPECIAL` if `is_special_enrollment` else `OPEN_ENROLLMENT` |
| `enrollment_submission_date` | `COALESCE(benefit.submitted_at, period.waived_at)` |
| FPL year | `resolve_fpl_year(year, start_of_coverage)` — prior year if coverage starts <6 months after publication |

## 7.9 View row-grain — read before counting anything

`employee_report` is **one row per onboarding period**, not per employee. An
employee with a QLE has several rows. Four mutually-exclusive boolean selectors,
each a `row_number() OVER (PARTITION BY open_enrollment_period_id, employee_id
...)` = 1:

| Selector | Picks |
|---|---|
| `is_open_enrollment_view` | the non-special (OE) period |
| `is_latest_coverage_view` | latest period in a coverage status |
| `is_enrollments_in_process_view` | earliest period not yet in coverage |
| `is_all_view` | latest period overall |

**Counting employees without filtering on one of these multi-counts anyone who
has had a QLE.** `intercom_sync_report` instead forces one row per period with
`LEFT JOIN LATERAL ... ORDER BY created_at, id LIMIT 1`.

Also: `employee_report` joins `benefit` with `benefit_type = 'MAJOR_MEDICAL'`
only — supplementals never appear there — and `onboarding_period_id_key` is
`COALESCE(onboarding_period_id, '000...0')` so the unique key survives employees
with no period.

---

# Part 8 — Rules enforced only in application code

212 `*Error` classes encode invariants the database does not. dbt tests cannot be
derived for these; they need bespoke assertions.

## 8.1 Cardinality

| Rule | Enforced by | DB guarantee? | Suggested assertion |
|---|---|---|---|
| Exactly one `EMPLOYEE` insured per period | `InsuredEmployeeAlreadyExistsError`, `CannotDeleteInsuredEmployeeError`, `InsuredNotFoundError` | **No** — no `UNIQUE (period_id, type)` | `count(*) filter (where type='EMPLOYEE') = 1` per `period_id` |
| At most one spouse per period | `InsuredPeopleDto` resolves with `.find()` | **No** — duplicates silently ignored | same shape for `SPOUSE_OR_DOMESTIC_PARTNER` |
| One active period per employee | `findActivePeriodOrThrow` uses `findFirstOrThrow` | **No** — would pick arbitrarily | `count(*) filter (where is_active) <= 1` per `employee_id` |
| A benefit covers ≥1 insured | `CannotDeleteAllInsuredOfBenefitError` | **No** — empty junction is a valid state | anti-join `benefit` to `_benefit_to_insured` |

Note the shared failure mode: both `insured` lookups use `.find()`, so a
duplicate is **silently masked** rather than raising — the API looks healthy
while a row sits invisible.

## 8.2 Status transitions

`BenefitStatus` has no transition table. Rules live in
`benefit-status.service.ts`:

- `IN_CART` is terminal-backwards — nothing transitions into it (encoded as
  `Exclude<BenefitStatus, 'IN_CART'>` on the workflow task payload)
- `AWAITING_MEDICAL` → `CARRIER_APPLICATION_SENT` is "not possible" (code
  comment); it must pass through `READY_TO_APPLY`
- `ACTIVE` / `ENDED` are written **only** by the cron workflow:
  `ENROLLMENT_CONFIRMED AND effective_from <= today` → `ACTIVE`;
  `ACTIVE AND effective_until < today` → `ENDED`. Note `<=` versus `<`, so a
  benefit stays `ACTIVE` through its final covered day.
- `AWAITING_EMPLOYEE` / `AWAITING_PAYMENT` writes are feature-flag gated and
  **fail silently** (plain return, no error, no log) when the flag is off

Everything from `READY_TO_APPLY` through `ENROLLMENT_CONFIRMED` is a **human
action** by an agent or operator — which is why those transitions carry
`EnrollmentActivityLog` entries and revert paths.

## 8.3 Date rules

Computed in `compute-enrollment-effective-dates.ts`, not constrained in SQL:

- standard: `effective_from` = latest of expected start, OEP start, eligibility
  start
- QLE: first-of-month after expected start, and first-of-month after
  (now + **3 business days**)
- QLE waiver: as QLE but **no** 3-business-day buffer
- birth QLE on a renewed plan: the **event date exactly**, no rounding
- `effective_until` = earliest of OEP end and eligibility end
- then trimmed to avoid overlapping another period holding a
  carrier-app-sent-or-later benefit (lower bound exclusive)
- **frozen at submission** — reused, not recomputed, once major medical is
  submitted-or-later

Guarding errors: `CoverageStartExceedsCoverageEndError`,
`EffectiveDatesOverlapError`, `ConflictingEnrollmentsError`,
`FinalizedPeriodOverlapError`, `EmployeeEligibilityEndLaterThanCoverageError`.

## 8.4 Money rules

- allowance derives from the **family unit of submitted medical benefits**, wiped
  to `NULL` when no submitted medical remains
- allocation order: medical, then dental (incl. combined dental/vision), then
  standalone vision; employee-covered first, then by `plan.external_id`
- `ER = min(remaining_allowance, premium)` and `EE = max(premium - ER, 0)`
- all money math uses `Decimal` with banker's rounding (`ROUND_HALF_EVEN`, 2dp)
- purse funding: normal benefits at carrier application; **self-enroll benefits
  at election submission**

Medical consuming the allowance first is a business rule, not an implementation
detail — supplementals are far more likely to fall on the employee.

## 8.5 Payment platform exclusivity

Echo (`payments-v1`) and Lynx (`payments`) are **mutually exclusive per
employer**, gated by the `lynx_payments_enabled` flag. No Echo call may be made
for a Lynx employer at any layer. Lynx is the source of truth for payment data —
local tables hold reference keys only. Not expressible in schema.

## 8.6 Unenforced invariants worth testing anyway

| Rule | Where it lives | Suggested test |
|---|---|---|
| `effective_from <= effective_until` | nowhere | explicit range test |
| Non-negative money | nowhere | test premium / contribution columns |
| `employer_contribution + employee_contribution = premium` | app code | reconciliation test where allocated |
| Valid status transitions | app code | not testable from a snapshot |

---

# Part 9 — Global column conventions

**Money** (see [`decimal-precision.md`](../tech/guidelines/decimal-precision.md)):

| Semantic | Type |
|---|---|
| Dollars (premiums, contributions, wages) | `Decimal @db.Decimal(12, 2)` |
| Rates, factors, percentages | `Decimal @db.Decimal(38, 18)` |
| Payments ledger | `Int`, column suffixed `_in_cents` |

Never `Float`; never a bare `@db.Decimal` (Prisma defaults it to `(65,30)`,
exceeding the Iceberg/Parquet precision cap of 38 and breaking DMS replication
typing). Scale is permanent once data reaches the warehouse — Iceberg can widen
precision but never change scale.

**Violations still present:** `open_enrollment_period.initial_draw_amount`,
`.reserve_amount` and `.threshold_amount` are `Float`.

**Dates.** Calendar dates are `String` (`YYYY-MM-DD`), cast to `DATE` inside SQL
— `effective_from`, `onboarding_until`, `date_of_birth`, `eligible_from`. True
instants are `DateTime` — `created_at`, `submitted_at`, `application_sent_at`. A
`*_date` or `*_from` suffix does **not** imply a temporal type.

**Demo data lives in production.** `employer.is_demo`, surfaced as
`is_demo_employer` on the reports. Exclude it from business metrics.

**`is_active` on `onboarding_period` means "election window open"**, not
"coverage active". Active coverage is `enrollment_status = 'ACTIVE_COVERAGE'`.

**`V2` models are the current ones** in `prospects` (`QuoteV2`,
`CensusEmployeeV2`, …) — the unsuffixed models are legacy. This is the opposite
of `BenefitStatus`, where the `V2` suffix was *dropped* in Aug 2026 when the
legacy enum was retired.

**`EnrollmentActivityLog` is the audit trail.** 31 `EnrollmentActivity` values
record who did what and when, including reverts, contribution changes, tag
changes and call logs. It is the best source for reconstructing history, since
most status columns carry only current state.

**Two statuses are mid-rollout.** `AWAITING_EMPLOYEE` is live but flag-gated per
employer (`awaiting_employee_benefit_status_enabled`, default off);
`AWAITING_PAYMENT` has a label, a wire value and a flag but **no logic writes
it** — its project (CORE-787) is recorded in commit history as paused. Zero rows
in either is expected, not a data bug.

---

# Part 10 — Per-table column reference

Ordered by domain: core, then election, money, quoting, reference data,
integrations, infra, views.

## administration - core domain

<sub>`administration.prisma` - 34 tables</sub>

### `agency`

<sub>model `Agency`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `Uuid` |
| `name` | `String` | NOT NULL |
| `legal_name` | `String` | null |
| `type` | `AgencyType` (enum) | NOT NULL |
| `contact_email` | `String` | null |
| `notification_emails` | `String` [] | NOT NULL |
| `business_address` | `String` | null |
| `phone_number` | `String` | null |
| `npn` | `String` | null |
| `main_agent_full_name` | `String` | null |
| `main_agent_npn` | `String` | null |
| `logo_url` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: agents->Agent[]; assignableAgencies->Agency[]; assignableBy->Agency[]; zorroPartnerEmployers->Employer[]; producerEmployers->Employer[]; enrollmentTeamEmployers->Employer[]; onboardingPeriods->OnboardingPeriod[]; intercomSyncStatus->IntercomSyncStatus</sub>

### `agent`

<sub>model `Agent`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `user_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `first_name` | `String` | NOT NULL |
| `last_name` | `String` | NOT NULL |
| `access_level` | `AccessLevel` (enum) | NOT NULL |
| `email` | `String` | NOT NULL |
| `agency_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: user->User; agency->Agency; producerAgentEmployers->Employer[]; benefits->Benefit[]</sub>

### `allowance_model`

<sub>model `AllowanceModel`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `year` | `Int` | NOT NULL |
| `use_excess_stipend` | `Boolean` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employer_id, year)

<sub>FK/relations: employer->Employer; allowanceModelItem->AllowanceModelItem[]; Quote->Quote; QuoteV2->QuoteV2; sourceSnapshot->AllowanceModelSourceSnapshot</sub>

### `allowance_model_item`

<sub>model `AllowanceModelItem`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `VarChar(255)` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `allowance_model_id` | `String` | NOT NULL, `Uuid` |
| `age_from` | `Int` | NOT NULL |
| `age_to` | `Int` | NOT NULL |
| `stipend` | `Int` | NOT NULL |
| `class` | `String` | NOT NULL |
| `family_unit` | `FamilyUnit` (enum) | NOT NULL |
| `number_of_dependents` | `Int` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employer->Employer; allowanceModel->AllowanceModel</sub>

### `allowance_model_source_snapshot`

<sub>model `AllowanceModelSourceSnapshot`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `allowance_model_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `plan_design_id` | `String` | null, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `quoting_year` | `Int` | NOT NULL |
| `is_hidden` | `Boolean` | NOT NULL, def `true` |
| `manual_upload` | `Boolean` | NOT NULL, def `false` |
| `adjustment_note` | `String` | null |
| `locked_at` | `DateTime` | NOT NULL, def `now()` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `snapshot_key` | `String` | NOT NULL |

<sub>FK/relations: allowanceModel->AllowanceModel</sub>

### `benefit`

<sub>model `Benefit`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `benefit_type` | `BenefitType` (enum) | NOT NULL |
| `period_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `member_id` | `String` | null |
| `status` | `BenefitStatus` (enum) | NOT NULL |
| `submission_type` | `SubmissionType` (enum) | null |
| `is_impersonated` | `Boolean` | null |
| `external_plan_id` | `String` | null |
| `premium` | `Decimal` | NOT NULL, `Decimal(12, 2)` |
| `self_pay_amount` | `Decimal` | null, `Decimal(12, 2)` |
| `is_hsa_eligible` | `Boolean` | null |
| `is_plan_manually_entered` | `Boolean` | NOT NULL, def `false` |
| `plan_name` | `String` | null |
| `carrier_name` | `String` | NOT NULL |
| `deductible` | `Decimal` | null, `Decimal(12, 2)` |
| `max_out_of_pocket` | `Decimal` | null, `Decimal(12, 2)` |
| `plan_benefits_summary_url` | `String` | null |
| `employee_monthly_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `employer_monthly_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `self_report_type` | `SelfReportType` (enum) | NOT NULL, def `NOT_APPLICABLE` |
| `is_combined_plan` | `Boolean` | NOT NULL, def `false` |
| `extended_plan_details` | `String` | null |
| `plan_market` | `PlanMarket` (enum) | null |
| `plan_self_enroll_type` | `SelfEnrollType` (enum) | NOT NULL, def `UNKNOWN` |
| `plan_policy_type` | `PolicyType` (enum) | null |
| `is_special_requests_confirmed` | `Boolean` | NOT NULL, def `false` |
| `is_aor_confirmed_by_employee` | `Boolean` | null |
| `enrollment_agent_id` | `String` | null, `Uuid` |
| `submitted_at` | `DateTime` | null |
| `application_sent_at` | `DateTime` | null |
| `application_sent_by_first_name` | `String` | null |
| `application_sent_by_last_name` | `String` | null |
| `application_submission_method` | `ApplicationSubmissionMethod` (enum) | null |
| `enrollment_confirmed_at` | `DateTime` | null |
| `confirmed_by_first_name` | `String` | null |
| `confirmed_by_last_name` | `String` | null |
| `initial_premium_payment_status` | `InitialPremiumPaymentStatus` (enum) | null |
| `auto_pay_status` | `AutoPayStatus` (enum) | null |
| `monthly_auto_pay_date` | `Int` | null, `SmallInt` |
| `is_monthly_auto_pay_date_end_of_month` | `Boolean` | NOT NULL, def `false` |
| `payment_method_override` | `CarrierPaymentMethod` (enum) | null |

- `UNIQUE` (id, employee_id, employer_id)
- `idx` (employee_id)
- `idx` (employer_id)
- `idx` (enrollment_agent_id)

<sub>FK/relations: period->OnboardingPeriod; insureds->Insured[]; enrollmentAgent->Agent; carrierApplications->CarrierApplication[]; benefitPaymentRequests->BenefitPaymentRequest[]</sub>

### `healthsherpa_application`

<sub>model `CarrierApplication`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `benefit_id` | `String` | null, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `provider` | `HealthSherpaApplicationProvider` (enum) | NOT NULL |
| `application_id` | `String` | **UQ**, null |
| `policy_id` | `String` | null |
| `status` | `HealthSherpaApplicationStatus` (enum) | NOT NULL |
| `policy_status` | `HealthSherpaPolicyStatus` (enum) | null |
| `policy_status_updated_date` | `String` | null |
| `payment_status` | `HealthSherpaPaymentStatus` (enum) | null |
| `payment_status_updated_date` | `String` | null |
| `autopay_indicator` | `Boolean` | null |
| `submitted_at` | `DateTime` | null |
| `effectuated_at` | `DateTime` | null |
| `terminated_at` | `DateTime` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (benefit_id)
- `idx` (employer_id)
- `idx` (employee_id)
- `idx` (status)

<sub>FK/relations: benefit->Benefit</sub>

### `class_definition`

<sub>model `ClassDefinition`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `year` | `Int` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (employer_id, year)

<sub>FK/relations: items->ClassDefinitionItem[]; employer->Employer</sub>

### `class_definition_item`

<sub>model `ClassDefinitionItem`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `class_definition_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `class_name` | `String` | NOT NULL |
| `state` | `USState` (enum) | null |
| `rating_area` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `custom_class_category` | `String` | null |

- `UNIQUE` (class_definition_id, class_name)
- `idx` (employer_id)

<sub>FK/relations: classDefinition->ClassDefinition</sub>

### `dependent`

<sub>model `Dependent`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `first_name` | `String` | NOT NULL |
| `last_name` | `String` | NOT NULL |
| `gender` | `Gender` (enum) | null |
| `date_of_birth` | `String` | NOT NULL |
| `is_pregnant` | `Boolean` | null |
| `is_smoker` | `Boolean` | null |
| `is_disabled` | `Boolean` | NOT NULL, def `false` |
| `ssn` | `String` | null |
| `citizenship_status` | `CitizenshipStatus` (enum) | null |
| `residential_address` | `String` | null |
| `zip_code` | `String` | null |
| `fips_code` | `String` | null |
| `state` | `USState` (enum) | null |
| `rating_area` | `String` | null |
| `locality` | `String` | null |
| `street_name` | `String` | null |
| `street_number` | `String` | null |
| `subpremise` | `String` | null |
| `latitude` | `Float` | null |
| `longitude` | `Float` | null |
| `county_resolution_source` | `CountyResolutionSource` (enum) | null |
| `address_resolved_at` | `DateTime` | null |
| `type` | `InsuredType` (enum) | NOT NULL |
| `sub_type` | `InsuredSubtype` (enum) | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)

<sub>FK/relations: employee->Employee; insureds->Insured[]</sub>

### `employee`

<sub>model `Employee`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `user_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `sub_entity_id` | `String` | null, `Uuid` |
| `sub_entity_2_id` | `String` | null, `Uuid` |
| `first_name` | `String` | NOT NULL |
| `last_name` | `String` | NOT NULL |
| `date_of_birth` | `String` | NOT NULL |
| `id_from_employer` | `String` | null |
| `termination_date` | `String` | null |
| `termination_report_date` | `String` | null |
| `gender` | `Gender` (enum) | null |
| `address` | `String` | null |
| `mailing_address` | `String` | null |
| `custom_class_category` | `String` | null |
| `is_address_manually_verified` | `Boolean` | null, def `false` |
| `state` | `USState` (enum) | null |
| `email` | `String` | **UQ**, NOT NULL |
| `personal_email` | `String` | null |
| `phone` | `String` | null |
| `class` | `String` | null |
| `fips_code` | `String` | null |
| `zip_code` | `String` | null |
| `rating_area` | `String` | null |
| `locality` | `String` | null |
| `street_name` | `String` | null |
| `street_number` | `String` | null |
| `subpremise` | `String` | null |
| `latitude` | `Float` | null |
| `longitude` | `Float` | null |
| `county_resolution_source` | `CountyResolutionSource` (enum) | null |
| `address_resolved_at` | `DateTime` | null |
| `salary` | `Float` | null |
| `hire_date` | `String` | null |
| `eligible_from` | `String` | null |
| `eligible_until` | `String` | null |
| `ssn` | `String` | null |
| `wage_type` | `WageType` (enum) | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `is_pregnant` | `Boolean` | null |
| `is_smoker` | `Boolean` | null |
| `citizenship_status` | `CitizenshipStatus` (enum) | null |
| `marital_status` | `MaritalStatus` (enum) | null |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (id, employer_id, user_id)
- `idx` (employer_id)

<sub>FK/relations: user->User; employer->Employer; subEntity->SubEntity; subEntity2->SubEntity2; info->EmployeeInfo; existingPlan->ExistingMajorMedicalPlan; onboardingPeriods->OnboardingPeriod[]; leaveOfAbsence->LeaveOfAbsence; finchEmployee->FinchEmployee; finchEmployeeNotification->FinchEmployeeNotification[]; Dependent->Dependent[]; eligibilityHistory->eligibilityHistory[]; FinchEmployeeIdentity->FinchEmployeeIdentity; payrollIndividuals->PayrollIndividual[]; paymentAccount->EmployeePaymentsAccount</sub>

### `employee_change_log`

<sub>model `EmployeeChangeLog`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | null, `Uuid` |
| `residential_address` | `String` | null |
| `fips_code` | `String` | null |
| `county_resolution_source` | `CountyResolutionSource` (enum) | null |
| `zip_code` | `String` | null |
| `state` | `USState` (enum) | null |
| `rating_area` | `String` | null |
| `wage_type` | `WageType` (enum) | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `class` | `String` | null |
| `custom_class_category` | `String` | null |
| `event_timestamp` | `DateTime` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `idx` (employee_id, eventTimestamp(sort: Desc))
- `idx` (employer_id)

### `employee_info`

<sub>model `EmployeeInfo`</sub>

| column | type | rules |
|---|---|---|
| `employee_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `notes` | `String` | null, `Text` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employee_id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: employee->Employee</sub>

### `employer`

<sub>model `Employer`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, **UQ**, NOT NULL, `Uuid` |
| `name` | `String` | NOT NULL |
| `legal_name` | `String` | null |
| `business_entity_type` | `BusinessEntityType` (enum) | null |
| `address` | `String` | null |
| `mailing_address` | `String` | null |
| `zip_code` | `String` | null |
| `state` | `String` | null |
| `phone` | `String` | null |
| `logo_url` | `String` | null |
| `payroll_cycle` | `PayrollCycle` (enum) | null |
| `hris_provider` | `String` | null, `VarChar(255)` |
| `tax_id_type` | `TaxIdType` (enum) | null |
| `tax_id` | `String` | null |
| `waiting_period` | `WaitingPeriod` (enum) | null |
| `is_applicable_large_employer` | `Boolean` | null |
| `signup_status` | `EmployerSignupStatus` (enum) | NOT NULL, def `DRAFT` |
| `is_demo` | `Boolean` | NOT NULL, def `false` |
| `is_payroll_sync_on` | `Boolean` | NOT NULL, def `false` |
| `is_partners_hub_on` | `Boolean` | NOT NULL, def `true` |
| `is_employment_sync_on` | `Boolean` | NOT NULL, def `false` |
| `prospect_transition_time` | `String` | null |
| `prospect_coverage_start_date` | `String` | null |
| `agreement_signed_date` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `crm_record_id` | `String` | null |
| `zorro_partner_id` | `String` | null, `Uuid` |
| `producer_id` | `String` | null, `Uuid` |
| `cc_email_list` | `String` [] | NOT NULL |
| `producer_agent_id` | `String` | null, `Uuid` |
| `csm_user_id` | `String` | null, `Uuid` |
| `invitation_link` | `String` | null |
| `consultation_link` | `String` | null |
| `monthly_sep_deadline` | `Int` | null, `SmallInt` |
| `is_monthly_sep_deadline_end_of_month` | `Boolean` | NOT NULL |
| `estimated_employees` | `Int` | null |
| `estimated_participation_rate` | `Int` | null |
| `naics_code` | `String` | null |
| `plan_exclusion_filter_set_id` | `String` | **UQ**, null, `Uuid` |
| `website_url` | `String` | null |
| `state_of_incorporation` | `USState` (enum) | null |
| `years_in_business` | `Int` | null |
| `enrollment_team_responsibilities` | `String` | null |

<sub>FK/relations: subEntities->SubEntity[]; subEntities2->SubEntity2[]; employerDocuments->EmployerDocument[]; allowanceModel->AllowanceModel[]; allowanceModelItem->AllowanceModelItem[]; classDefinitions->ClassDefinition[]; openEnrollmentPeriods->OpenEnrollmentPeriod[]; contacts->EmployerContacts[]; employees->Employee[]; quotes->Quote[]; zorroPartner->Agency; producer->Agency; enrollmentTeams->Agency[]; producerAgent->Agent; csmUser->User; enrollmentInstructions->EnrollmentInstructions[]; userActivationLink->UserActivationLink[]; FinchEmployer->FinchEmployer; quoteSessions->QuoteSession[]; FinchEmployee->FinchEmployee[]; QuoteV2->QuoteV2[]; planExclusionFilterSet->PlanExclusionFilterSet; intercomSyncStatus->IntercomSyncStatus; FinchEmployeeIdentity->FinchEmployeeIdentity[]; beneficialOwners->EmployerBeneficialOwner[]; paymentAccount->EmployerPaymentsAccount; payrollPayGroups->PayrollPayGroup[]</sub>

### `employer_beneficial_owner`

<sub>model `EmployerBeneficialOwner`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `first_name` | `String` | null |
| `last_name` | `String` | null |
| `email` | `String` | null |
| `phone` | `String` | null |
| `address` | `String` | null |
| `date_of_birth` | `String` | null |
| `ssn` | `String` | null |
| `is_officer` | `Boolean` | NOT NULL, def `false` |
| `is_primary` | `Boolean` | NOT NULL, def `false` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)

<sub>FK/relations: employer->Employer</sub>

### `employer_contacts`

<sub>model `EmployerContacts`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `type` | `ContactType` (enum) | NOT NULL |
| `first_name` | `String` | null |
| `last_name` | `String` | null |
| `phone` | `String` | null |
| `email` | `String` | null |
| `is_admin` | `Boolean` | NOT NULL |

<sub>FK/relations: employer->Employer</sub>

### `employer_document`

<sub>model `EmployerDocument`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `DocumentType` (enum) | NOT NULL, def `PLAN` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `document_url` | `String` | **UQ**, NOT NULL |
| `file_name` | `String` | NOT NULL |
| `is_legal_agreement` | `Boolean` | null |
| `description` | `String` | null |
| `open_enrollment_period_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employer->Employer; openEnrollmentPeriod->OpenEnrollmentPeriod</sub>

### `enrollment_activity_log`

<sub>model `EnrollmentActivityLog`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `benefit_ids` | `String` [] | NOT NULL, def `[]`, `Uuid` |
| `benefit_types` | `BenefitType` (enum) [] | NOT NULL, def `[]` |
| `activity` | `EnrollmentActivity` (enum) | NOT NULL |
| `performed_by_id` | `String` | null, `Uuid` |
| `performed_by_email` | `String` | null |
| `performed_by_first_name` | `String` | null |
| `performed_by_last_name` | `String` | null |
| `submission_type` | `SubmissionType` (enum) | null |
| `is_impersonated` | `Boolean` | NOT NULL, def `false` |
| `correlation_id` | `String` | null |
| `flow_id` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `metadata` | `Json` | null, `JsonB` |

- `idx` (onboarding_period_id)
- `idx` (employee_id)
- `idx` (employer_id)
- `idx` (correlation_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `enrollment_comment`

<sub>model `EnrollmentComment`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `content` | `Json` | NOT NULL |
| `tenant_id` | `String` | null, `Uuid` |
| `performed_by_id` | `String` | NOT NULL, `Uuid` |
| `performed_by_email` | `String` | null |
| `performed_by_first_name` | `String` | null |
| `performed_by_last_name` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `deleted_at` | `DateTime` | null |

- `idx` (onboarding_period_id, deleted_at)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `enrollment_instructions`

<sub>model `EnrollmentInstructions`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `content` | `Json` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employer->Employer</sub>

### `existing_major_medical_plan`

<sub>model `ExistingMajorMedicalPlan`</sub>

| column | type | rules |
|---|---|---|
| `employee_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `year` | `Int` | NOT NULL, `SmallInt` |
| `external_id` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employee_id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: employee->Employee</sub>

### `leave_of_absence`

<sub>model `LeaveOfAbsence`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `start_date` | `String` | NOT NULL |
| `end_date` | `String` | NOT NULL |
| `payment_by` | `PaymentBy` (enum) | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `employee_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |

- `UNIQUE` (employee_id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: employee->Employee</sub>

### `major_medical_carrier`

<sub>model `MajorMedicalCarrier`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `state` | `String` | NOT NULL |
| `echo_code` | `String` | NOT NULL |
| `payment_method` | `CarrierPaymentMethod` (enum) | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (name, state)

### `onboarding_period`

<sub>model `OnboardingPeriod`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `OnboardingPeriodType` (enum) | NOT NULL |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `enrollment_period_id` | `String` | NOT NULL, `Uuid` |
| `is_active` | `Boolean` | NOT NULL |
| `onboarding_from` | `String` | NOT NULL |
| `onboarding_until` | `String` | NOT NULL |
| `effective_from` | `String` | NOT NULL |
| `effective_until` | `String` | NOT NULL |
| `waived_at` | `DateTime` | null |
| `allowance` | `Int` | null |
| `is_special_enrollment` | `Boolean` | NOT NULL |
| `is_waived` | `Boolean` | NOT NULL, def `false` |
| `waived_reason` | `String` | null |
| `last_user_login` | `DateTime` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `class` | `String` | null |
| `enrollment_team_id` | `String` | null, `Uuid` |
| `enrollment_tags` | `EnrollmentTag` (enum) [] | NOT NULL, def `[]` |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (id, employee_id, employer_id)
- `idx` (employee_id)
- `idx` (enrollment_period_id, employee_id)
- `idx` (employer_id)

<sub>FK/relations: employee->Employee; enrollmentPeriod->OpenEnrollmentPeriod; insured->Insured[]; decisionFactorsPreference->DecisionFactorsPreference; providerPreferences->ProviderPreference[]; drugPreferences->DrugPreference[]; shoppingPreferences->ShoppingPreference[]; costPreferences->CostPreference[]; benefits->Benefit[]; electronicSignatures->ElectronicSignature[]; reviewSnapshots->ReviewSnapshot[]; qualifyingLifeEvent->QualifyingLifeEvent; paymentMethod->PaymentMethod; benefitDocuments->BenefitDocument[]; enrollmentActivityLogs->EnrollmentActivityLog[]; enrollmentComments->EnrollmentComment[]; finchEnrollment->FinchEnrollment[]; savedPlans->SavedPlan[]; enrollmentTeam->Agency</sub>

### `open_enrollment_period`

<sub>model `OpenEnrollmentPeriod`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `effective_from` | `String` | NOT NULL |
| `effective_until` | `String` | NOT NULL |
| `onboarding_from` | `String` | NOT NULL |
| `onboarding_until` | `String` | NOT NULL |
| `payment_method` | `OEPPaymentMethod` (enum) | null |
| `initial_draw_amount` | `Float` | null |
| `initial_draw_date` | `String` | null |
| `reserve_amount` | `Float` | null |
| `threshold_amount` | `Float` | null |
| `is_aor_changed` | `Boolean` | null |
| `is_domestic_partner_allowed` | `Boolean` | NOT NULL, def `true` |
| `is_dental_enabled` | `Boolean` | NOT NULL, def `false` |
| `is_vision_enabled` | `Boolean` | NOT NULL, def `false` |
| `is_qme_enabled` | `Boolean` | NOT NULL, def `false` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (id, employer_id)

<sub>FK/relations: employer->Employer; onboardingPeriods->OnboardingPeriod[]; paymentRequests->OpenEnrollmentPeriodPaymentRequest[]; EmployerDocument->EmployerDocument[]</sub>

### `plan_exclusion_filter`

<sub>model `PlanExclusionFilter`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `plan_exclusion_filter_set_id` | `String` | NOT NULL, `Uuid` |
| `expression` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (plan_exclusion_filter_set_id)

<sub>FK/relations: exclusionFilterSet->PlanExclusionFilterSet</sub>

### `plan_exclusion_filter_set`

<sub>model `PlanExclusionFilterSet`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employer->Employer; exclusionFilters->PlanExclusionFilter[]</sub>

### `qualifying_life_event`

<sub>model `QualifyingLifeEvent`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `QualifyingLifeEventType` (enum) | NOT NULL |
| `onboarding_period_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `other_reason` | `String` | null |
| `occurred_on` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (onboarding_period_id, employee_id, employer_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `role`

<sub>model `Roles`</sub>

| column | type | rules |
|---|---|---|
| `id` | `Role` (enum) | **PK**, NOT NULL |
| `frontegg_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: userRoles->UserRole[]</sub>

### `sub_entity`

<sub>model `SubEntity`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, **UQ**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `employer_id` | `String` | NOT NULL, `Uuid` |

- `UNIQUE` (name, employer_id)

<sub>FK/relations: employer->Employer; employees->Employee[]</sub>

### `sub_entity_2`

<sub>model `SubEntity2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, **UQ**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `employer_id` | `String` | NOT NULL, `Uuid` |

- `UNIQUE` (name, employer_id)

<sub>FK/relations: employer->Employer; employees->Employee[]</sub>

### `user_activation_link`

<sub>model `UserActivationLink`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `redirect_to` | `ActivationRedirectUrlOptions` (enum) | NOT NULL |
| `employer_id` | `String` | null, `Uuid` |
| `employer_name` | `String` | null |
| `is_disabled` | `Boolean` | NOT NULL, def `false` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: user->User; employer->Employer</sub>

### `user_role`

<sub>model `UserRole`</sub>

| column | type | rules |
|---|---|---|
| `user_id` | `String` | NOT NULL, `Uuid` |
| `tenant_id` | `String` | NOT NULL, `Uuid` |
| `role_id` | `Role` (enum) | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (user_id, role_id)

<sub>FK/relations: user->User; fronteggRole->Roles</sub>

### `eligibility_history`

<sub>model `eligibilityHistory`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `eligible_from` | `String` | NOT NULL |
| `eligible_until` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)
- `idx` (employee_id)

<sub>FK/relations: employee->Employee</sub>

## shopping - employee election flow

<sub>`shopping.prisma` - 13 tables</sub>

### `benefit_document`

<sub>model `BenefitDocument`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `document_url` | `String` | **UQ**, NOT NULL |
| `file_name` | `String` | NOT NULL |
| `is_shared_with_employee` | `Boolean` | NOT NULL, def `true` |
| `type` | `BenefitDocumentType` (enum) | null, def `OTHER` |
| `uploaded_by_user_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `cost_preference`

<sub>model `CostPreference`</sub>

| column | type | rules |
|---|---|---|
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `attribute` | `CostAttribute` (enum) | NOT NULL |
| `direction` | `PreferredDirection` (enum) | NOT NULL |
| `importance` | `CostPreferenceImportance` (enum) | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (onboarding_period_id, attribute)
- `idx` (onboarding_period_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `decision_factors_preference`

<sub>model `DecisionFactorsPreference`</sub>

| column | type | rules |
|---|---|---|
| `onboarding_period_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `decision_factors` | `DecisionFactors` (enum) [] | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (onboarding_period_id, employee_id, employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `drug_preference`

<sub>model `DrugPreference`</sub>

| column | type | rules |
|---|---|---|
| `external_id` | `String` | NOT NULL, `VarChar(255)` |
| `med_id` | `Int` | NOT NULL |
| `rx_cui_id` | `String` | null |
| `name` | `String` | NOT NULL |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (external_id, onboarding_period_id)
- `idx` (onboarding_period_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `electronic_signature`

<sub>model `ElectronicSignature`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `ip` | `String` | NOT NULL |
| `user_agent` | `String` | NOT NULL |
| `user_time_zone` | `String` | NOT NULL |
| `presented_date` | `String` | NOT NULL |
| `agreement_date_time` | `String` | NOT NULL |
| `signed_name` | `String` | NOT NULL |
| `signer_name` | `String` | null |
| `signer_email` | `String` | null |
| `signer_user_id` | `String` | null, `Uuid` |
| `signer_roles` | `String` [] | NOT NULL |
| `is_impersonated` | `Boolean` | null |
| `review_snapshot_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (period_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: period->OnboardingPeriod; reviewSnapshot->ReviewSnapshot</sub>

### `insured`

<sub>model `Insured`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `InsuredType` (enum) | NOT NULL |
| `first_name` | `String` | NOT NULL |
| `last_name` | `String` | NOT NULL |
| `date_of_birth` | `String` | NOT NULL |
| `gender` | `Gender` (enum) | null |
| `residential_address` | `String` | null |
| `is_smoker` | `Boolean` | NOT NULL |
| `is_pregnant` | `Boolean` | NOT NULL |
| `is_disabled` | `Boolean` | NOT NULL, def `false` |
| `ssn` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `period_id` | `String` | NOT NULL, `Uuid` |
| `anticipated_care_level` | `AnticipatedCareLevel` (enum) | null |
| `citizenship_status` | `CitizenshipStatus` (enum) | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `mailing_address` | `String` | null |
| `wage_type` | `WageType` (enum) | null |
| `subtype` | `InsuredSubtype` (enum) | null |
| `company_email` | `String` | null |
| `fips_code` | `String` | null |
| `personal_email` | `String` | null |
| `phone_number` | `String` | null |
| `rating_area` | `String` | null |
| `state` | `USState` (enum) | null |
| `zip_code` | `String` | null |
| `locality` | `String` | null |
| `street_name` | `String` | null |
| `street_number` | `String` | null |
| `subpremise` | `String` | null |
| `latitude` | `Float` | null |
| `longitude` | `Float` | null |
| `county_resolution_source` | `CountyResolutionSource` (enum) | null |
| `address_resolved_at` | `DateTime` | null |
| `custom_class_category` | `String` | null |
| `marital_status` | `MaritalStatus` (enum) | null |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `dependent_id` | `String` | null, `Uuid` |

- `UNIQUE` (period_id, dependent_id)
- `idx` (period_id)
- `idx` (employee_id)
- `idx` (employer_id)
- `idx` (dependent_id)

<sub>FK/relations: dependent->Dependent; benefits->Benefit[]; period->OnboardingPeriod</sub>

### `provider_preference`

<sub>model `ProviderPreference`</sub>

| column | type | rules |
|---|---|---|
| `npi` | `Int` | NOT NULL |
| `name` | `String` | null |
| `specialty` | `String` | null |
| `phone` | `String` | null |
| `provider_type` | `String` | null, `VarChar(16)` |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (npi, onboarding_period_id)
- `UNIQUE` (npi, onboarding_period_id, employer_id, employee_id)
- `idx` (onboarding_period_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod; addresses->ProviderPreferenceAddress[]</sub>

### `provider_preference_address`

<sub>model `ProviderPreferenceAddress`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | NOT NULL, `VarChar(255)` |
| `street_line_1` | `String` | NOT NULL |
| `city` | `String` | NOT NULL |
| `state` | `USState` (enum) | NOT NULL |
| `zip_code` | `String` | NOT NULL |
| `provider_npi` | `Int` | NOT NULL |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `is_selected` | `Boolean` | NOT NULL, def `true` |
| `address_confidence` | `String` | null, `VarChar(16)` |

- `PK` (id, provider_npi, onboarding_period_id)
- `idx` (onboarding_period_id)

<sub>FK/relations: providerPreference->ProviderPreference</sub>

### `recommendation_data`

<sub>model `RecommendationData`</sub>

| column | type | rules |
|---|---|---|
| `plan_id` | `String` | NOT NULL, `VarChar(255)` |
| `plan_year` | `Int` | NOT NULL |
| `care_level_1_count` | `Int` | NOT NULL |
| `care_level_2_count` | `Int` | NOT NULL |
| `care_level_3_count` | `Int` | NOT NULL |
| `care_level_4_count` | `Int` | NOT NULL |
| `medical_expenses` | `Float` | NOT NULL |
| `risk` | `Float` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (plan_year, plan_id, care_level_1_count, care_level_2_count, care_level_3_count, care_level_4_count)

### `review_snapshot`

<sub>model `ReviewSnapshot`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `ReviewSnapshotType` (enum) | NOT NULL |
| `period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `payload` | `Json` | NOT NULL |
| `content_hash` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `idx` (period_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: period->OnboardingPeriod; electronicSignatures->ElectronicSignature[]</sub>

### `saved_plan`

<sub>model `SavedPlan`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `plan_id` | `String` | NOT NULL |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `UNIQUE` (plan_id, onboarding_period_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

### `shopping_preference`

<sub>model `ShoppingPreference`</sub>

| column | type | rules |
|---|---|---|
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `type` | `ShoppingPreferenceType` (enum) | NOT NULL |
| `version` | `String` | NOT NULL |
| `value` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `PK` (onboarding_period_id, type)
- `idx` (onboarding_period_id)

<sub>FK/relations: option->ShoppingPreferenceOption; onboardingPeriod->OnboardingPeriod</sub>

### `shopping_preference_option`

<sub>model `ShoppingPreferenceOption`</sub>

| column | type | rules |
|---|---|---|
| `type` | `ShoppingPreferenceType` (enum) | NOT NULL |
| `version` | `String` | NOT NULL |
| `value` | `String` | NOT NULL |
| `label` | `String` | null |
| `display_order` | `Int` | NOT NULL |

- `PK` (type, version, value)

<sub>FK/relations: answers->ShoppingPreference[]</sub>

## payments - Lynx (current)

<sub>`payments.prisma` - 16 tables</sub>

### `actual_transaction`

<sub>model `ActualTransaction`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `vendor` | `PaymentsVendor` (enum) | NOT NULL, def `LYNX` |
| `vendor_correlation_id` | `String` | NOT NULL |
| `vendor_account_id` | `String` | NOT NULL |
| `vendor_pull_group_id` | `String` | null |
| `category` | `ActualTransactionCategory` (enum) | NOT NULL |
| `amount_in_cents` | `Int` | NOT NULL |
| `transaction_date` | `DateTime` | NOT NULL, `Date` |
| `actual_carrier_raw` | `String` | null |
| `actual_carrier_vendor_id` | `String` | null |
| `payment_method` | `CarrierPaymentMethod` (enum) | null |
| `card_last_four` | `String` | null |
| `vendor_status` | `String` | NOT NULL |
| `settlement_status` | `SettlementStatus` (enum) | NOT NULL |
| `settled_at` | `DateTime` | null |
| `reserve_used` | `Boolean` | NOT NULL, def `false` |
| `reserve_amount_in_cents` | `Int` | NOT NULL, def `0` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (vendor, vendor_correlation_id, vendor_account_id)
- `idx` (vendor, vendor_pull_group_id)

<sub>FK/relations: matches->ActualTransactionMatch[]</sub>

### `actual_transaction_match`

<sub>model `ActualTransactionMatch`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `transaction_id` | `String` | NOT NULL, `Uuid` |
| `expectation_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `status` | `ActualTransactionMatchStatus` (enum) | NOT NULL |
| `allocated_amount_in_cents` | `Int` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (transaction_id, expectation_id)
- `idx` (expectation_id)

<sub>FK/relations: transaction->ActualTransaction; expectation->PullExpectation</sub>

### `allocation_request`

<sub>model `AllocationRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `amount_in_cents` | `Int` | NOT NULL |
| `external_id` | `String` | null |
| `short_description` | `String` | null |
| `request_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)
- `idx` (employee_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; employeePaymentsAccount->EmployeePaymentsAccount; request->PaymentRequest</sub>

### `benefit_payment_request`

<sub>model `BenefitPaymentRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `request_id` | `String` | NOT NULL, `Uuid` |
| `benefit_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `payments_cycle` | `PaymentsCycle` (enum) | NOT NULL |
| `plan_year` | `Int` | null |
| `coverage_month` | `Month` (enum) | null |
| `flow_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (request_id, benefit_id)
- `idx` (benefit_id)
- `idx` (flow_id)

<sub>FK/relations: paymentRequest->PaymentRequest; benefit->Benefit</sub>

### `book_transfer_request`

<sub>model `BookTransferRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `amount_in_cents` | `Int` | NOT NULL |
| `originating_account_type` | `AccountType` (enum) | NOT NULL |
| `receiving_account_type` | `AccountType` (enum) | NOT NULL |
| `external_id` | `String` | null |
| `short_description` | `String` | null |
| `request_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; request->PaymentRequest</sub>

### `carrier_vendor_mapping`

<sub>model `CarrierVendorMapping`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `vendor` | `PaymentsVendor` (enum) | NOT NULL, def `LYNX` |
| `vendor_merchant_id` | `String` | NOT NULL |
| `normalized_carrier` | `String` | null |
| `observed_name` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (vendor, vendor_merchant_id)
- `idx` (normalized_carrier)

### `contribution_request`

<sub>model `ContributionRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `amount_in_cents` | `Int` | NOT NULL |
| `external_id` | `String` | null |
| `settled_at` | `DateTime` | null |
| `short_description` | `String` | null |
| `skip_settlement_wait` | `Boolean` | NOT NULL, def `false` |
| `status` | `ContributionRequestStatus` (enum) | NOT NULL, def `PENDING` |
| `request_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; request->PaymentRequest</sub>

### `deallocation_request`

<sub>model `DeallocationRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `amount_in_cents` | `Int` | NOT NULL |
| `external_id` | `String` | null |
| `short_description` | `String` | null |
| `request_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)
- `idx` (employee_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; employeePaymentsAccount->EmployeePaymentsAccount; request->PaymentRequest</sub>

### `employee_payments_account`

<sub>model `EmployeePaymentsAccount`</sub>

| column | type | rules |
|---|---|---|
| `employee_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `user_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `creation_status` | `CreationStatus` (enum) | NOT NULL, def `PENDING` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employee_id, employer_id)
- `UNIQUE` (employee_id, employer_id, user_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; employee->Employee; allocationRequests->AllocationRequest[]; deallocationRequests->DeallocationRequest[]; paymentsAlerts->EmployeePaymentsAlert[]</sub>

### `employee_payments_alert`

<sub>model `EmployeePaymentsAlert`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `type` | `EmployeePaymentsAlertType` (enum) | NOT NULL |
| `idempotency_key` | `String` | NOT NULL |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `metadata` | `Json` | NOT NULL, def `"{}"`, `JsonB` |
| `performed_by_id` | `String` | null, `Uuid` |
| `performed_by_email` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (type, idempotency_key)
- `idx` (employer_id)
- `idx` (employee_id)

<sub>FK/relations: employerPaymentsAccount->EmployerPaymentsAccount; employeePaymentsAccount->EmployeePaymentsAccount</sub>

### `employer_payments_account`

<sub>model `EmployerPaymentsAccount`</sub>

| column | type | rules |
|---|---|---|
| `employer_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `creation_status` | `CreationStatus` (enum) | NOT NULL, def `PENDING` |
| `is_ichra_only` | `Boolean` | NOT NULL, def `true` |
| `reserve_amount_in_cents` | `Int` | NOT NULL |
| `threshold_amount_in_cents` | `Int` | NOT NULL |
| `bank_account_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employer->Employer; lynxEmployerCredentials->LynxEmployerCredentials; employeePaymentsAccounts->EmployeePaymentsAccount[]; contributionRequests->ContributionRequest[]; allocationRequests->AllocationRequest[]; deallocationRequests->DeallocationRequest[]; bookTransferRequests->BookTransferRequest[]; paymentsAlerts->EmployeePaymentsAlert[]</sub>

### `expected_amount_version`

<sub>model `ExpectedAmountVersion`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `expectation_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `version` | `Int` | NOT NULL |
| `amount_in_cents` | `Int` | NOT NULL |
| `source` | `ExpectedAmountSource` (enum) | NOT NULL |
| `created_by_id` | `String` | null, `Uuid` |
| `created_by_email` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `UNIQUE` (expectation_id, version)

<sub>FK/relations: expectation->PullExpectation</sub>

### `lynx_employer_credentials`

<sub>model `LynxEmployerCredentials`</sub>

| column | type | rules |
|---|---|---|
| `employer_id` | `String` | **PK**, NOT NULL, `Uuid` |
| `client_id` | `String` | NOT NULL |
| `client_secret` | `String` | NOT NULL, `Text` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employerPaymentAccount->EmployerPaymentsAccount</sub>

### `open_enrollment_period_payment_request`

<sub>model `OpenEnrollmentPeriodPaymentRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `request_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `open_enrollment_period_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `payments_cycle` | `PaymentsCycle` (enum) | NOT NULL |
| `plan_year` | `Int` | null |
| `coverage_month` | `Month` (enum) | null |
| `flow_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (open_enrollment_period_id, payments_cycle)

<sub>FK/relations: paymentRequest->PaymentRequest; openEnrollmentPeriod->OpenEnrollmentPeriod</sub>

### `payment_request`

<sub>model `PaymentRequest`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `schedule_date` | `DateTime` | null |
| `status` | `PaymentRequestStatus` (enum) | NOT NULL, def `PENDING` |
| `type` | `PaymentRequestType` (enum) | NOT NULL |
| `idempotency_key` | `String` | NOT NULL, `Uuid` |
| `submitted_at` | `DateTime` | null |
| `manual_retry_count` | `Int` | NOT NULL, def `0` |
| `last_failed_at` | `DateTime` | null |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employer_id, idempotency_key)
- `idx` (status, schedule_date)
- `idx` (status, created_at)

<sub>FK/relations: dependsOn->PaymentRequest[]; dependents->PaymentRequest[]; contributionRequest->ContributionRequest; allocationRequest->AllocationRequest; deallocationRequest->DeallocationRequest; bookTransferRequest->BookTransferRequest; benefitPaymentRequests->BenefitPaymentRequest[]; openEnrollmentPeriodPaymentRequest->OpenEnrollmentPeriodPaymentRequest; pullExpectations->PullExpectation[]</sub>

### `pull_expectation`

<sub>model `PullExpectation`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_year` | `Int` | NOT NULL |
| `coverage_month` | `Month` (enum) | NOT NULL |
| `expected_carrier` | `String` | NOT NULL |
| `expected_date` | `String` | null |
| `payment_type` | `PaymentsCycle` (enum) | NOT NULL |
| `request_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employee_id, plan_year, coverage_month, expected_carrier)
- `idx` (employer_id, plan_year, coverage_month)
- `idx` (request_id)

<sub>FK/relations: paymentRequest->PaymentRequest; expectedAmounts->ExpectedAmountVersion[]; transactionMatches->ActualTransactionMatch[]</sub>

## payments-v1 - Echo (legacy)

<sub>`payments-v1.prisma` - 1 tables</sub>

### `payment_method`

<sub>model `PaymentMethod`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `onboarding_period_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `status` | `EmployeePaymentMethodStatus` (enum) | NOT NULL, def `EMPTY` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (onboarding_period_id, employee_id, employer_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: onboardingPeriod->OnboardingPeriod</sub>

## prospects - quoting / pre-sale

<sub>`prospects.prisma` - 21 tables</sub>

### `census_employee`

<sub>model `CensusEmployee`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `id_from_employer` | `String` | NOT NULL |
| `name` | `String` | null |
| `date_of_birth` | `String` | NOT NULL |
| `family_unit` | `FamilyUnit` (enum) | NOT NULL |
| `zip_code` | `String` | NOT NULL |
| `current_premium` | `Float` | null |
| `current_allowance` | `Float` | null |
| `current_plan` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `w2_wage` | `Float` | null |
| `spouse_date_of_birth` | `String` | null |
| `dependents_dates_of_birth` | `String` [] | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (zip_code)
- `idx` (current_plan)
- `idx` (quote_id)
- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `census_employee_v2`

<sub>model `CensusEmployeeV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `id_from_employer` | `String` | NOT NULL |
| `name` | `String` | null |
| `date_of_birth` | `String` | NOT NULL |
| `family_unit` | `FamilyUnit` (enum) | NOT NULL |
| `zip_code` | `String` | NOT NULL |
| `current_premium` | `Float` | null |
| `current_allowance` | `Float` | null |
| `current_plan` | `String` | null |
| `current_plan_id` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `w2_wage` | `Float` | null |
| `spouse_date_of_birth` | `String` | null |
| `dependents_dates_of_birth` | `String` [] | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (zip_code)
- `idx` (current_plan)
- `idx` (current_plan_id)
- `idx` (quote_id)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `census_plan`

<sub>model `CensusPlan`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_name` | `String` | NOT NULL |
| `carrier` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |
| `deductible` | `Float` | NOT NULL |
| `moop` | `Float` | NOT NULL |
| `pcp` | `Float` | null |
| `specialist` | `Float` | null |
| `rx1` | `Float` | null |
| `rx2` | `Float` | null |
| `rx3` | `Float` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (plan_name)
- `idx` (network_type)
- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `census_plan_v2`

<sub>model `CensusPlanV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_name` | `String` | NOT NULL |
| `carrier` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |
| `deductible` | `Float` | NOT NULL |
| `moop` | `Float` | NOT NULL |
| `pcp` | `Float` | null |
| `specialist` | `Float` | null |
| `rx1` | `Float` | null |
| `rx2` | `Float` | null |
| `rx3` | `Float` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (plan_name)
- `idx` (network_type)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `contribution_by_class`

<sub>model `ContributionByClass`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `employee_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `spouse_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `child_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |

- `idx` (employer_id)

<sub>FK/relations: firstContributionPlanDesign->QuotePlanDesign; secondContributionPlanDesign->QuotePlanDesign</sub>

### `quote`

<sub>model `Quote`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `pepm_fee` | `Float` | null |
| `fixed_platform_fee` | `Float` | null |
| `quoting_year` | `Int` | null |
| `exclude_affordability` | `Boolean` | null |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `target_plan_ids` | `String` [] | NOT NULL |
| `last_run_at` | `DateTime` | null |
| `last_updated_at` | `DateTime` | NOT NULL, def `now()` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `allowance_model_id` | `String` | **UQ**, null, `Uuid` |

- `UNIQUE` (id, employer_id)
- `idx` (target_plan_ids)

<sub>FK/relations: employer->Employer; censusEmployees->CensusEmployee[]; censusPlans->CensusPlan[]; quoteEmployees->QuoteEmployee[]; QuotePreferences->QuotePreference; quotePlans->QuotePlan[]; quoteAgeCostUnits->QuoteAgeCostUnits[]; quotePlansDiversity->QuotePlanDiversity[]; allowanceModel->AllowanceModel; quoteSessions->QuoteSession[]</sub>

### `quote_age_cost_unit`

<sub>model `QuoteAgeCostUnits`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `band_type` | `String` | NOT NULL |
| `state_code` | `String` | NOT NULL |
| `band` | `String` | NOT NULL |
| `cost_units` | `Float` | NOT NULL |

- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `quote_age_cost_unit_v2`

<sub>model `QuoteAgeCostUnitsV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `band_type` | `String` | NOT NULL |
| `state_code` | `String` | NOT NULL |
| `band` | `String` | NOT NULL |
| `cost_units` | `Float` | NOT NULL |

- `idx` (quote_id, band_type, band, state_code)
- `idx` (quote_id, state_code, cost_units)
- `idx` (quote_id, band_type, band, cost_units)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_age_premium_pool_v2`

<sub>model `QuoteAgePremiumPoolV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `scenario` | `Scenario` (enum) | NOT NULL |
| `aggregator` | `Aggregator` (enum) | NOT NULL |
| `state_code` | `String` | NOT NULL |
| `rating_area` | `String` | null |
| `age` | `Int` | NOT NULL |
| `avg_premium` | `Float` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `UNIQUE` (quote_id, scenario, aggregator, state_code, rating_area, age)
- `idx` (quote_id, scenario, aggregator, age)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_allowance_model_row`

<sub>model `QuoteAllowanceModelRow`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_plan_design_id` | `String` | NOT NULL, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `state_code` | `String` | null |
| `rating_area` | `String` | null |
| `age` | `Int` | NOT NULL |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `employee_allowance` | `Int` | NOT NULL |
| `employee_spouse_allowance` | `Int` | NOT NULL |
| `employee_child_allowance` | `Int` | NOT NULL |
| `family_allowance` | `Int` | NOT NULL |
| `family_1_allowance` | `Int` | NOT NULL |
| `family_2_allowance` | `Int` | NOT NULL |
| `family_3_allowance` | `Int` | NOT NULL |
| `employee_child_1_allowance` | `Int` | NOT NULL |
| `employee_child_2_allowance` | `Int` | NOT NULL |
| `employee_child_3_allowance` | `Int` | NOT NULL |

- `UNIQUE` (quote_plan_design_id, quote_id, state_code, rating_area, age, employment_type, wage_type)
- `idx` (employer_id)

<sub>FK/relations: quotePlanDesign->QuotePlanDesign; quote->QuoteV2</sub>

### `quote_employee`

<sub>model `QuoteEmployee`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `scenario` | `String` | NOT NULL |
| `plan_id` | `String` | NOT NULL |
| `employee_id` | `String` | NOT NULL |
| `id_from_employer` | `String` | NOT NULL |
| `name` | `String` | null |
| `family_unit` | `FamilyUnit` (enum) | NOT NULL |
| `state_code` | `String` | NOT NULL |
| `medicare_eligible` | `Boolean` | NOT NULL |
| `age` | `Int` | NOT NULL |
| `spouse_age` | `Int` | null |
| `age_band_5` | `String` | NOT NULL |
| `age_band_1` | `String` | NOT NULL |
| `plan_name` | `String` | NOT NULL |
| `zorro_carrier` | `String` | NOT NULL |
| `metal_level` | `MetalLevel` (enum) | NOT NULL |
| `employee_cost_units` | `Float` | NOT NULL |
| `base_premium` | `Float` | NOT NULL |
| `premium` | `Float` | NOT NULL |
| `expected_cost` | `Float` | NOT NULL |
| `cost_units` | `Float` | NOT NULL |
| `premium_cost_units` | `Float` | NOT NULL |
| `children` | `Int` | NOT NULL |
| `current_premium` | `Float` | null |
| `current_allowance` | `Float` | null |
| `current_plan_name` | `String` | null |
| `current_carrier` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `w2_wage` | `Float` | null |
| `state_price_factor` | `Float` | NOT NULL |
| `rating_area` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `quote_employee_v2`

<sub>model `QuoteEmployeeV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `scenario` | `Scenario` (enum) | NOT NULL |
| `plan_id` | `String` | NOT NULL |
| `employee_id` | `String` | NOT NULL |
| `id_from_employer` | `String` | NOT NULL |
| `name` | `String` | null |
| `family_unit` | `FamilyUnit` (enum) | NOT NULL |
| `state_code` | `String` | NOT NULL |
| `medicare_eligible` | `Boolean` | NOT NULL |
| `age` | `Int` | NOT NULL |
| `employee_age` | `Int` | NOT NULL |
| `spouse_age` | `Int` | null |
| `age_band_5` | `String` | NOT NULL |
| `age_band_1` | `String` | NOT NULL |
| `plan_name` | `String` | NOT NULL |
| `zorro_carrier` | `String` | NOT NULL |
| `metal_level` | `MetalLevel` (enum) | NOT NULL |
| `employee_cost_units` | `Float` | NOT NULL |
| `base_premium` | `Float` | NOT NULL |
| `premium` | `Float` | NOT NULL |
| `expected_cost` | `Float` | NOT NULL |
| `cost_units` | `Float` | NOT NULL |
| `premium_cost_units` | `Float` | NOT NULL |
| `children` | `Int` | NOT NULL |
| `is_participating` | `Boolean` | null |
| `current_premium` | `Float` | null |
| `current_allowance` | `Float` | null |
| `current_plan_name` | `String` | null |
| `current_carrier` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `wage_type` | `WageType` (enum) | null |
| `w2_wage` | `Float` | null |
| `state_price_factor` | `Float` | NOT NULL |
| `rating_area` | `String` | null |
| `pricing_fips_code` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `idx` (quote_id, scenario, age, state_code)
- `idx` (quote_id, scenario, rating_area)
- `idx` (quote_id, scenario, state_code, rating_area, family_unit, age, children)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_plan`

<sub>model `QuotePlan`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_name` | `String` | NOT NULL |
| `carrier` | `String` | NOT NULL |
| `deductible` | `Int` | NOT NULL |
| `medical_moop` | `Int` | NOT NULL |
| `pcp_cost` | `String` | null |
| `generic_drugs_cost` | `String` | null |
| `specialist_cost` | `String` | null |
| `plan_id` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |

- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `quote_plan_design`

<sub>model `QuotePlanDesign`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `priorities` | `QuotePlanPriorities` (enum) [] | NOT NULL |
| `scenario` | `Scenario` (enum) | null |
| `savingSplit` | `Int` | null |
| `work_status` | `ClassType` (enum) | null |
| `geo_distribution` | `GeoDistribution` (enum) | null |
| `contribution_group` | `ContributionGroup` (enum) | null |
| `family_unit` | `FamilyUnitDistribution` (enum) | null |
| `contribution_type` | `AllowanceUnit` (enum) | null |
| `contribution_mode` | `ContributionMode` (enum) | null |
| `entity_view` | `EntityView` (enum) | null |
| `selected_preset` | `ContributionPreset` (enum) | null |
| `first_contribution_id` | `String` | **UQ**, null, `Uuid` |
| `second_contribution_id` | `String` | **UQ**, null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `name` | `String` | null |
| `is_completed` | `Boolean` | NOT NULL, def `false` |

- `UNIQUE` (id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2; firstContributionByClass->ContributionByClass; secondContributionByClass->ContributionByClass; quoteAllowanceModels->QuoteAllowanceModelRow[]</sub>

### `quote_plan_diversity`

<sub>model `QuotePlanDiversity`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_type` | `PlanType` (enum) | NOT NULL |
| `plan_id` | `String` | NOT NULL |
| `plan_name` | `String` | NOT NULL |
| `metal_level` | `MetalLevel` (enum) | null |
| `issuer_name` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |

- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `quote_plan_diversity_v2`

<sub>model `QuotePlanDiversityV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_type` | `PlanType` (enum) | NOT NULL |
| `plan_id` | `String` | NOT NULL |
| `plan_name` | `String` | NOT NULL |
| `metal_level` | `MetalLevel` (enum) | null |
| `issuer_name` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |

- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_plan_v2`

<sub>model `QuotePlanV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `plan_name` | `String` | NOT NULL |
| `carrier` | `String` | NOT NULL |
| `deductible` | `Int` | NOT NULL |
| `medical_moop` | `Int` | NOT NULL |
| `pcp_cost` | `String` | null |
| `generic_drugs_cost` | `String` | null |
| `specialist_cost` | `String` | null |
| `plan_id` | `String` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |
| `metal_level` | `MetalLevel` (enum) | null |
| `benefits_summary_url` | `String` | null |

- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_preference`

<sub>model `QuotePreference`</sub>

| column | type | rules |
|---|---|---|
| `quote_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `selected_scenario` | `String` | NOT NULL |
| `contribution_type` | `AllowanceUnit` (enum) | NOT NULL |
| `use_classes` | `Boolean` | NOT NULL |
| `selected_class` | `ClassType` (enum) | null |
| `main_allowance_employee_only_contribution` | `Float` | NOT NULL |
| `main_allowance_employee_spouse_contribution` | `Float` | NOT NULL |
| `main_allowance_employee_children_contribution` | `Float` | NOT NULL |
| `main_allowance_employee_family_contribution` | `Float` | NOT NULL |
| `secondary_allowance_employee_only_contribution` | `Float` | NOT NULL |
| `secondary_allowance_employee_spouse_contribution` | `Float` | NOT NULL |
| `secondary_allowance_employee_children_contribution` | `Float` | NOT NULL |
| `secondary_allowance_employee_family_contribution` | `Float` | NOT NULL |
| `geo_distribution` | `GeoDistribution` (enum) | NOT NULL |
| `family_units` | `FamilyUnitDistribution` (enum) | NOT NULL |

- `UNIQUE` (quote_id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: quote->Quote</sub>

### `quote_presets`

<sub>model `QuotePresets`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `quote_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `preset` | `ContributionPreset` (enum) | NOT NULL |
| `employee_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `spouse_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `child_contribution` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `unit` | `AllowanceUnit` (enum) | NOT NULL |

- `UNIQUE` (quote_id, preset)
- `idx` (employer_id)

<sub>FK/relations: quote->QuoteV2</sub>

### `quote_session`

<sub>model `QuoteSession`</sub>

| column | type | rules |
|---|---|---|
| `quote_id` | `String` | **PK**, **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `quote_data` | `Json` | NOT NULL |

<sub>FK/relations: quote->Quote; employer->Employer</sub>

### `quote_v2`

<sub>model `QuoteV2`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `pepm_fee` | `Float` | null |
| `fixed_platform_fee` | `Float` | null |
| `exclude_affordability` | `Boolean` | null |
| `quoting_year` | `Int` | null |
| `start_of_coverage` | `String` | null |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `target_plan_ids` | `String` [] | NOT NULL |
| `renewal_mode` | `Boolean` | NOT NULL, def `false` |
| `renewal_baseline_scenario` | `Scenario` (enum) | null |
| `use_prior_year_lcsp` | `Boolean` | NOT NULL, def `false` |
| `last_run_at` | `DateTime` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `allowance_model_id` | `String` | **UQ**, null, `Uuid` |
| `setup_status` | `SetupStatus` (enum) | NOT NULL |

- `UNIQUE` (id, employer_id)
- `idx` (target_plan_ids)

<sub>FK/relations: employer->Employer; censusEmployees->CensusEmployeeV2[]; censusPlans->CensusPlanV2[]; quoteEmployees->QuoteEmployeeV2[]; quotePlans->QuotePlanV2[]; quoteAgeCostUnits->QuoteAgeCostUnitsV2[]; quoteAgePremiumPools->QuoteAgePremiumPoolV2[]; quotePlansDiversity->QuotePlanDiversityV2[]; allowanceModel->AllowanceModel; planDesigns->QuotePlanDesign[]; quoteAllowanceModelRows->QuoteAllowanceModelRow[]; quotePresets->QuotePresets[]</sub>

## external - plan catalog & rating data

<sub>`external.prisma` - 12 tables</sub>

### `blended_rate_decisions`

<sub>model `BlendedRateDecision`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `state_code` | `USState` (enum) | NOT NULL |
| `carrier_name` | `String` | NOT NULL, `VarChar(255)` |
| `hios` | `String` | NOT NULL, `VarChar(255)` |
| `rate_change_pct` | `Decimal` | null, `Decimal(38, 18)` |
| `lookup_method` | `BlendedRateLookupMethod` (enum) | NOT NULL |
| `estimate_source` | `BlendedRateEstimateSource` (enum) | null |
| `issuer_match_type` | `BlendedRateIssuerMatchType` (enum) | null |
| `adjustment_pp` | `Decimal` | null, `Decimal(38, 18)` |
| `year` | `Int` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (state_code, hios, year)

### `carrier_info`

<sub>model `CarrierInfo`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `state` | `USState` (enum) | NOT NULL |
| `is_zorro_appointed` | `Boolean` | NOT NULL |
| `is_hs_eap` | `Boolean` | NOT NULL |
| `is_pdf_application_available` | `Boolean` | NOT NULL, def `true` |
| `website_url` | `String` | NOT NULL |
| `phone_number` | `String` | NOT NULL |
| `provider_search_url` | `String` | NOT NULL |
| `medication_search_url` | `String` | NOT NULL |
| `initial_payment` | `PaymentResponsibility` (enum) | null |
| `initial_payment_comments` | `String` | null |
| `auto_pay` | `PaymentResponsibility` (enum) | null |
| `auto_pay_comments` | `String` | null |
| `special_requests_election_submitted` | `String` | null |
| `special_requests_carrier_app_sent` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (name, state)

### `dw_blended_rate_decision_versions`

<sub>model `DwBlendedRateDecisionVersion`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `USState` (enum) | NOT NULL |
| `carrier_name` | `String` | NOT NULL, `VarChar(255)` |
| `hios` | `String` | NOT NULL, `VarChar(255)` |
| `rate_change_pct` | `Decimal` | null, `Decimal(38, 18)` |
| `lookup_method` | `BlendedRateLookupMethod` (enum) | NOT NULL |
| `estimate_source` | `BlendedRateEstimateSource` (enum) | null |
| `issuer_match_type` | `BlendedRateIssuerMatchType` (enum) | null |
| `adjustment_pp` | `Decimal` | null, `Decimal(38, 18)` |
| `year` | `Int` | NOT NULL |
| `valid_from` | `DateTime` | NOT NULL, `Timestamptz(6)` |
| `valid_to` | `DateTime` | null, `Timestamptz(6)` |

- `PK` (state_code, hios, year, valid_from)

### `special_case_zip_codes`

<sub>model `SpecialCaseZipCode`</sub>

| column | type | rules |
|---|---|---|
| `id` | `Int` | **PK**, NOT NULL, def `autoincrement()` |
| `zip_code` | `Int` | NOT NULL |
| `state_code` | `String` | NOT NULL, `VarChar(2)` |

- `UNIQUE` (zip_code, state_code)
- `idx` (zip_code)

### `state_appointments`

<sub>model `StateAppointments`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | **UQ**, NOT NULL |
| `code` | `USState` (enum) | **UQ**, NOT NULL |
| `is_appointed` | `Boolean` | NOT NULL |
| `website` | `String` | null |
| `initial_payment` | `PaymentResponsibility` (enum) | null |
| `initial_payment_comments` | `String` | null |
| `auto_pay` | `PaymentResponsibility` (enum) | null |
| `auto_pay_comments` | `String` | null |
| `special_requests_election_submitted` | `String` | null |
| `special_requests_carrier_app_sent` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

### `sf_medical_plans`

<sub>model `sfMedicalPlan`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `String` | NOT NULL |
| `metal_level` | `String` | NOT NULL |
| `issuer_name` | `String` | null |
| `plan_id` | `String` | NOT NULL, `VarChar(255)` |
| `network_type` | `NetworkType` (enum) | NOT NULL |
| `sbc_url` | `String` | null |
| `adult_dental` | `Boolean` | null |
| `child_dental` | `Boolean` | null |
| `medical_deductible_individual` | `Int` | null |
| `drug_deductible_individual` | `Int` | null |
| `medical_deductible_family` | `Int` | null |
| `drug_deductible_family` | `Int` | null |
| `medical_maximum_out_of_pocket_individual` | `Int` | null |
| `drug_maximum_out_of_pocket_individual` | `Int` | null |
| `medical_maximum_out_of_pocket_family` | `Int` | null |
| `drug_maximum_out_of_pocket_family` | `Int` | null |
| `plan_name` | `String` | NOT NULL |
| `hsa_eligible` | `Boolean` | null |
| `on_market` | `Boolean` | null |
| `off_market` | `Boolean` | null |
| `embedded_deductible` | `String` | null |
| `gated` | `Boolean` | null |
| `network_size` | `Int` | null |
| `plan_market_var` | `String` | null |
| `coinsurance` | `Int` | null |
| `data_source` | `String` | NOT NULL |
| `plan_year` | `Int` | NOT NULL |
| `is_available` | `Boolean` | NOT NULL, def `true` |

- `PK` (plan_id, plan_year)
- `idx` (state_code, plan_year)
- `idx` (metal_level)
- `idx` (plan_year)

<sub>FK/relations: pricingZip->sfPlanPricingZip[]; sfPlanBenefitStandardized->sfPlanBenefitStandardized[]</sub>

### `sf_plan_benefit_standardized`

<sub>model `sfPlanBenefitStandardized`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `String` | NOT NULL |
| `plan_id` | `String` | NOT NULL, `VarChar(255)` |
| `deductible` | `Int` | NOT NULL |
| `max_out_of_pocket` | `Int` | NOT NULL |
| `network_type` | `NetworkType` (enum) | NOT NULL |
| `pcp` | `String` | null |
| `specialist` | `String` | null |
| `rx1` | `String` | null |
| `rx2` | `String` | null |
| `rx3` | `String` | null |
| `deductible_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `max_out_of_pocket_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `pcp_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `specialist_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `rx1_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `rx2_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `rx3_stand` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `plan_type_stand` | `Int` | NOT NULL |
| `plan_year` | `Int` | NOT NULL |
| `score` | `Decimal` | NOT NULL, def `0`, `Decimal(38, 18)` |

- `PK` (plan_id, plan_year)
- `idx` (state_code)
- `idx` (plan_year)
- `idx` (state_code, plan_year)

<sub>FK/relations: sfMedicalPlan->sfMedicalPlan</sub>

### `sf_plan_pricing_zip`

<sub>model `sfPlanPricingZip`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `String` | NOT NULL |
| `county_name` | `String` | NOT NULL |
| `fips_code` | `Int` | NOT NULL |
| `zip_code` | `Int` | NOT NULL |
| `state_price_factor` | `Float` | NOT NULL |
| `plan_id` | `String` | NOT NULL, `VarChar(255)` |
| `base_premium` | `Decimal` | NOT NULL, `Decimal(12, 2)` |
| `plan_year` | `Int` | NOT NULL |
| `rating_area` | `String` | null |
| `base_premium_stand` | `Decimal` | NOT NULL, def `0`, `Decimal(38, 18)` |
| `age` | `Int` | NOT NULL |

- `PK` (plan_id, plan_year, zip_code, age, fips_code)
- `idx` (zip_code, plan_year, age)
- `idx` (plan_year)
- `idx` (fips_code, plan_year)

<sub>FK/relations: sfMedicalPlan->sfMedicalPlan</sub>

### `sf_premium_age_factor`

<sub>model `sfPremiumAgeFactor`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `String` | NOT NULL, `VarChar(255)` |
| `age` | `Int` | NOT NULL |
| `premium_factor` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `band_5` | `String` | NOT NULL |
| `band_10` | `String` | NOT NULL |
| `band_5_factor` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `band_10_factor` | `Decimal` | NOT NULL, `Decimal(38, 18)` |

- `PK` (state_code, age)
- `idx` (age)
- `idx` (state_code, age)

### `sf_state_benefit_params`

<sub>model `sfStateBenefitParams`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `String` | NOT NULL, `VarChar(255)` |
| `plan_year` | `Int` | NOT NULL |
| `avg_deductible` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_deductible` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_max_out_of_pocket` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_max_out_of_pocket` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_pcp` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_pcp` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_specialist` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_specialist` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_rx1` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_rx1` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_rx2` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_rx2` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `avg_rx3` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `std_rx3` | `Decimal` | NOT NULL, `Decimal(38, 18)` |

- `PK` (state_code, plan_year)

### `zip_county`

<sub>model `zipCounty`</sub>

| column | type | rules |
|---|---|---|
| `zip_code` | `Int` | NOT NULL |
| `fips_code` | `Int` | NOT NULL |
| `state_code` | `String` | NOT NULL, `VarChar(2)` |
| `county_name` | `String` | NOT NULL, `VarChar(255)` |
| `plan_year` | `Int` | NOT NULL |

- `PK` (zip_code, state_code, county_name, plan_year)
- `idx` (zip_code)
- `idx` (plan_year)

### `zip_state`

<sub>model `zipState`</sub>

| column | type | rules |
|---|---|---|
| `zip_code` | `Int` | NOT NULL |
| `state_code` | `String` | NOT NULL, `VarChar(2)` |
| `state_price_factor` | `Float` | NOT NULL |

- `PK` (zip_code, state_code)
- `idx` (zip_code)

## insurance - regulatory constants

<sub>`insurance.prisma` - 1 tables</sub>

### `plan_year_data`

<sub>model `PlanYearData`</sub>

| column | type | rules |
|---|---|---|
| `year` | `Int` | **PK**, NOT NULL |
| `fpl_publication_date` | `DateTime` | null, `Date` |
| `alaska_poverty_line` | `Int` | NOT NULL |
| `hawaii_poverty_line` | `Int` | NOT NULL |
| `poverty_line` | `Int` | NOT NULL |
| `medicare_cost` | `Int` | NOT NULL |
| `affordability_percentage` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `new_york_employee_child_count` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `new_york_family_child_count` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `vermont_employee_child_count` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `vermont_family_child_count` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `general_child_count` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `hsa_individual_medical_deductible_threshold` | `Int` | NOT NULL, def `0` |
| `hsa_individual_medical_max_out_of_pocket_threshold` | `Int` | NOT NULL, def `0` |
| `hsa_family_medical_deductible_threshold` | `Int` | NOT NULL, def `0` |
| `hsa_family_medical_max_out_of_pocket_threshold` | `Int` | NOT NULL, def `0` |
| `hsa_embedded_individual_medical_deductible_threshold` | `Int` | NOT NULL, def `0` |

## integrations - Finch / Intercom

<sub>`integrations.prisma` - 13 tables</sub>

### `custom_class_finch_mapping`

<sub>model `CustomClassFinchMapping`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `eligibility_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `category_name_zorro` | `String` | NOT NULL |
| `expression` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (eligibility_id, category_name_zorro)
- `idx` (employer_id)

<sub>FK/relations: eligibility->FinchEmployerEligibility</sub>

### `finch_deduction`

<sub>model `FinchDeduction`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `finch_employer_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `finch_employer_entity_id` | `String` | NOT NULL, `Uuid` |
| `hsa_eligible` | `FinchDeductionHsaEligible` (enum) | NOT NULL |
| `tax_type` | `FinchDeductionTax` (enum) | NOT NULL, def `NOT_APPLICABLE` |
| `benefit_id` | `String` | NOT NULL |
| `job_id` | `String` | NOT NULL, `Uuid` |
| `job_status` | `FinchJobStatus` (enum) | NOT NULL |
| `action` | `FinchDeductionAction` (enum) | NOT NULL, def `UNKNOWN` |
| `request` | `Json` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (finch_employer_id, finch_employer_entity_id, hsa_eligible, tax_type)
- `idx` (employer_id)

<sub>FK/relations: finchEmployer->FinchEmployer; finchEmployerEntity->FinchEmployerEntity; finchEnrollment->FinchEnrollment[]</sub>

### `finch_employee`

<sub>model `FinchEmployee`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `finch_individual_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `finch_employer_entity_id` | `String` | NOT NULL, `Uuid` |
| `sync_type` | `FinchEmployeeSyncType` (enum) | NOT NULL |
| `pay_group_id` | `String` | null, `Uuid` |
| `pay_frequency` | `FinchPayFrequency` (enum) | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (id, employer_id)

<sub>FK/relations: employee->Employee; employer->Employer; finchEmployerEntity->FinchEmployerEntity; finchEnrollment->FinchEnrollment[]; finchEmployeeNotification->FinchEmployeeNotification[]</sub>

### `finch_employee_identity`

<sub>model `FinchEmployeeIdentity`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `finch_individual_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employee_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

<sub>FK/relations: employee->Employee; employer->Employer</sub>

### `finch_employee_notification`

<sub>model `FinchEmployeeNotification`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `finch_employer_id` | `String` | NOT NULL, `Uuid` |
| `finch_employee_id` | `String` | null, `Uuid` |
| `notification_type` | `FinchNotificationType` (enum) | NOT NULL |
| `last_sent_at` | `String` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (employee_id, finch_employer_id, notification_type)
- `idx` (employer_id)

<sub>FK/relations: employee->Employee; finchEmployer->FinchEmployer; finchEmployee->FinchEmployee</sub>

### `finch_employer`

<sub>model `FinchEmployer`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `session_id` | `String` | NOT NULL, `Uuid` |
| `connection_status` | `FinchConnectionStatus` (enum) | null |
| `last_sync_status` | `FinchSyncStatus` (enum) | null |
| `last_successful_sync_date_time` | `String` | null |
| `integration_type` | `FinchIntegrationType` (enum) | null |
| `connection_id` | `String` | null, `Uuid` |
| `provider_id` | `String` | null, `VarChar(255)` |
| `access_token` | `String` | null, `Uuid` |
| `client_type` | `String` | null |
| `deduction_action` | `FinchDeductionAction` (enum) | NOT NULL, def `UNKNOWN` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `earliest_enrollments_sync_date` | `String` | null |
| `early_enrollments_offset` | `Int` | NOT NULL, def `1` |
| `enforce_post_tax_deductions` | `Boolean` | NOT NULL, def `true` |
| `is_hsa_deductions` | `Boolean` | NOT NULL, def `false` |
| `pay_group_behavior` | `FinchPayGroupBehavior` (enum) | NOT NULL, def `DEFAULT` |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (employer_id, session_id)
- `UNIQUE` (connection_id, access_token)

<sub>FK/relations: employer->Employer; provider->FinchProvider; finchPayments->FinchPayment[]; finchDeductions->FinchDeduction[]; FinchEmployerEntity->FinchEmployerEntity[]; finchEmployerEligibility->FinchEmployerEligibility; finchEmployeeNotification->FinchEmployeeNotification[]</sub>

### `finch_employer_eligibility`

<sub>model `FinchEmployerEligibility`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `finch_employer_id` | `String` | **UQ**, NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `eligibility_logic` | `String` | NOT NULL |
| `full_time_mapping` | `String` | null |
| `part_time_mapping` | `String` | null |
| `salary_mapping` | `String` | null |
| `hourly_mapping` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

- `UNIQUE` (id, employer_id)
- `UNIQUE` (finch_employer_id, employer_id)
- `idx` (employer_id)

<sub>FK/relations: finchEmployer->FinchEmployer; customClassMappings->CustomClassFinchMapping[]</sub>

### `finch_employer_entity`

<sub>model `FinchEmployerEntity`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `Uuid` |
| `name` | `String` | NOT NULL |
| `finch_employer_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `estimate_pay_groups_count` | `Int` | NOT NULL, def `4` |
| `is_enabled` | `Boolean` | NOT NULL, def `true` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `idx` (employer_id)

<sub>FK/relations: finchEmployer->FinchEmployer; FinchPayment->FinchPayment[]; FinchEmployee->FinchEmployee[]; FinchDeduction->FinchDeduction[]</sub>

### `finch_enrollment`

<sub>model `FinchEnrollment`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `finch_employee_id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `onboarding_period_id` | `String` | NOT NULL, `Uuid` |
| `finch_deduction_id` | `String` | NOT NULL, `Uuid` |
| `job_id` | `String` | NOT NULL, `Uuid` |
| `job_status` | `FinchJobStatus` (enum) | NOT NULL |
| `job_index` | `Int` | null |
| `action` | `FinchEnrollmentAction` (enum) | NOT NULL |
| `is_correction_needed` | `Boolean` | NOT NULL, def `false` |
| `employee_deduction_amount` | `String` | NOT NULL |
| `company_contribution_amount` | `String` | NOT NULL |
| `desired_enroll_effective_date` | `String` | NOT NULL |
| `desired_enroll_date` | `String` | NOT NULL |
| `desired_unenroll_effective_date` | `String` | NOT NULL |
| `desired_unenroll_date` | `String` | NOT NULL |
| `last_enroll_date` | `String` | null |
| `last_unenroll_date` | `String` | null |
| `unenroll_reason` | `FinchUnenrollReason` (enum) | null |
| `request` | `Json` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (finch_employee_id, finch_deduction_id)
- `idx` (employee_id)
- `idx` (employer_id)

<sub>FK/relations: finchEmployee->FinchEmployee; onboardingPeriod->OnboardingPeriod; finchDeduction->FinchDeduction</sub>

### `finch_payment`

<sub>model `FinchPayment`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `finch_payment_id` | `String` | NOT NULL, `Uuid` |
| `finch_employer_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `finch_employer_entity_id` | `String` | NOT NULL, `Uuid` |
| `is_valid` | `Boolean` | NOT NULL, def `false` |
| `pay_group_id` | `String` | null, `Uuid` |
| `pay_frequency` | `FinchPayFrequency` (enum) | null |
| `start_date` | `String` | null |
| `end_date` | `String` | null |
| `individual_ids` | `String` [] | NOT NULL, def `[]` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (finch_payment_id, pay_group_id)
- `idx` (employer_id)

<sub>FK/relations: finchEmployer->FinchEmployer; finchEmployerEntity->FinchEmployerEntity</sub>

### `finch_provider`

<sub>model `FinchProvider`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `VarChar(255)` |
| `display_name` | `String` | NOT NULL |
| `logo_url` | `String` | NOT NULL |
| `icon_url` | `String` | NOT NULL |
| `deduction_action` | `FinchDeductionAction` (enum) | NOT NULL, def `UNKNOWN` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

<sub>FK/relations: FinchEmployer->FinchEmployer[]</sub>

### `intercom_sync_report` - VIEW

<sub>model `IntercomSyncReport`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | NOT NULL, `Uuid` |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `enrollment_status` | `EnrollmentStatus` (enum) | NOT NULL |
| `aor_cancellation_status` | `AorCancellationStatus` (enum) | NOT NULL |
| `carrier_request_status` | `CarrierRequestStatus` (enum) | null |
| `payment_method_status` | `EmployeePaymentMethodStatus` (enum) | null |
| `employee_wage_type` | `WageType` (enum) | null |
| `employee_employment_type` | `EmploymentType` (enum) | null |
| `employee_citizenship_status` | `CitizenshipStatus` (enum) | null |
| `employee_marital_status` | `MaritalStatus` (enum) | null |
| `created_at` | `DateTime` | NOT NULL |
| `onboarding_from` | `String` | NOT NULL |
| `onboarding_until` | `String` | NOT NULL |
| `effective_from` | `String` | NOT NULL |
| `effective_until` | `String` | NOT NULL |
| `enrollment_effective_from` | `String` | NOT NULL |
| `is_special_enrollment` | `Boolean` | NOT NULL |
| `is_waived` | `Boolean` | NOT NULL |
| `has_qualifying_benefit` | `Boolean` | NOT NULL |
| `allowance` | `Int` | null |
| `period_class` | `String` | null |
| `enrollment_tags` | `EnrollmentTag` (enum) [] | NOT NULL |
| `enrollment_team_name` | `String` | null |
| `enrollment_agent_email` | `String` | null |
| `qle_type` | `QualifyingLifeEventType` (enum) | null |
| `qle_occurred_on` | `String` | null |
| `family_unit` | `FamilyUnit` (enum) | null |
| `premium` | `Decimal` | null, `Decimal(12, 2)` |
| `employer_monthly_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `employee_monthly_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `self_pay_amount` | `Decimal` | null, `Decimal(12, 2)` |
| `carrier_name` | `String` | null |
| `plan_name` | `String` | null |
| `is_plan_manually_entered` | `Boolean` | null |
| `is_combined_plan` | `Boolean` | null |
| `self_report_type` | `SelfReportType` (enum) | null |
| `plan_market` | `PlanMarket` (enum) | null |
| `self_enroll` | `SelfEnrollType` (enum) | null |
| `member_id` | `String` | null |
| `auto_pay_status` | `AutoPayStatus` (enum) | null |
| `initial_premium_payment_status` | `InitialPremiumPaymentStatus` (enum) | null |
| `submission_type` | `SubmissionType` (enum) | null |

- `UNIQUE` (id)

### `specialty`

<sub>model `Specialty`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `VarChar(255)` |
| `name` | `String` | NOT NULL |
| `category` | `String` | NOT NULL |
| `code` | `String` | NOT NULL |
| `sub_specialty` | `String` | null |
| `updated_at` | `DateTime` | NOT NULL, auto |

## payroll - direct payroll integrations

<sub>`payroll.prisma` - 3 tables</sub>

### `payroll_individual`

<sub>model `PayrollIndividual`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `provider_individual_id` | `String` | NOT NULL, `VarChar(255)` |
| `provider_employer_entity_id` | `String` | NOT NULL, `VarChar(255)` |
| `is_active` | `Boolean` | NOT NULL |
| `provider_metadata` | `Json` | NOT NULL |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `is_matched` | `Boolean` | NOT NULL |
| `pay_group_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (employer_id, provider_individual_id, provider_employer_entity_id)
- `idx` (employee_id)

<sub>FK/relations: employee->Employee; payGroup->PayrollPayGroup</sub>

### `payroll_pay_group`

<sub>model `PayrollPayGroup`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `provider_employer_entity_id` | `String` | NOT NULL, `VarChar(255)` |
| `provider_pay_group_id` | `String` | NOT NULL, `VarChar(255)` |
| `pay_frequency` | `PayrollPayFrequency` (enum) | NOT NULL |
| `is_active` | `Boolean` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (employer_id, provider_employer_entity_id, provider_pay_group_id)
- `UNIQUE` (id, employer_id)

<sub>FK/relations: employer->Employer; payPeriods->PayrollPayPeriod[]; individuals->PayrollIndividual[]</sub>

### `payroll_pay_period`

<sub>model `PayrollPayPeriod`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `pay_group_id` | `String` | NOT NULL, `Uuid` |
| `start_date` | `DateTime` | NOT NULL, `Date` |
| `end_date` | `DateTime` | NOT NULL, `Date` |
| `pay_date` | `DateTime` | null, `Date` |
| `source` | `PayrollPayPeriodSource` (enum) | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, def `now()`, auto |

- `UNIQUE` (pay_group_id, start_date)
- `idx` (employer_id)

<sub>FK/relations: payGroup->PayrollPayGroup</sub>

## infra - workflows, audit, events

<sub>`infra.prisma` - 11 tables</sub>

### `address_resolve_responses`

<sub>model `AddressResolveResponse`</sub>

| column | type | rules |
|---|---|---|
| `id` | `BigInt` | **PK**, NOT NULL, def `autoincrement()` |
| `resolution_id` | `String` | NOT NULL, `Uuid` |
| `correlation_id` | `String` | null, `Uuid` |
| `input_address` | `String` | NOT NULL |
| `provider` | `AddressResolveProvider` (enum) | NOT NULL |
| `outcome` | `AddressResolveOutcome` (enum) | NOT NULL |
| `http_status` | `Int` | null |
| `duration_ms` | `Int` | NOT NULL |
| `retry_count` | `Int` | NOT NULL, def `0` |
| `request` | `Json` | NOT NULL |
| `response` | `Json` | null |
| `error` | `Json` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `idx` (created_at)
- `idx` (resolution_id)
- `idx` (input_address)

### `audit`

<sub>model `Audit`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `user_roles` | `String` [] | NOT NULL |
| `url` | `String` | NOT NULL |
| `http_method` | `String` | NOT NULL |
| `request_body` | `Json` | NOT NULL |
| `response_status_code` | `Int` | null |
| `response_body` | `Json` | null |
| `error` | `String` | null |
| `referer` | `String` | null |
| `time_taken_milliseconds` | `Int` | NOT NULL |
| `correlation_id` | `String` | null, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `idx` (createdAt(sort: Desc))
- `idx` (correlation_id)

### `cache_data`

<sub>model `CacheData`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `VarChar(255)` |
| `data` | `Json` | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

### `data_change_audit`

<sub>model `DataChangeAudit`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `action` | `String` | NOT NULL |
| `table_name` | `String` | NOT NULL |
| `row_id` | `String` | NOT NULL |
| `args` | `Json` | null |
| `data` | `Json` | null |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `user_roles` | `String` [] | NOT NULL |
| `url` | `String` | null |
| `correlation_id` | `String` | null, `Uuid` |

- `idx` (table_name, row_id)
- `idx` (correlation_id)
- `idx` (createdAt(sort: Desc))

### `event`

<sub>model `Event`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `name` | `String` | NOT NULL |
| `source` | `EventSource` (enum) | NOT NULL, def `BROWSER` |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `pageUrl` | `String` | NOT NULL |
| `user_agent` | `String` | null |
| `referer` | `String` | null |
| `sessionId` | `String` | NOT NULL |
| `ip` | `String` | null |
| `employer_id` | `String` | null, `Uuid` |
| `employee_id` | `String` | null, `Uuid` |
| `is_impersonated` | `Boolean` | null |
| `payload` | `Json` | null |
| `sent_at` | `DateTime` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

### `request_context`

<sub>model `RequestContext`</sub>

| column | type | rules |
|---|---|---|
| `id` | `BigInt` | **PK**, NOT NULL, def `autoincrement()` |
| `correlation_id` | `String` | NOT NULL, `Uuid` |
| `user_id` | `String` | null |
| `user_roles` | `String` [] | NOT NULL |
| `url` | `String` | null |
| `route` | `String` | null |
| `origin` | `String` | NOT NULL |
| `source_ip` | `String` | null |
| `job_id` | `String` | null |
| `job_name` | `String` | null |
| `trace_id` | `String` | null |
| `app_version` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |

- `idx` (created_at)

### `webhook_event`

<sub>model `WebhookEvent`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `provider` | `String` | NOT NULL |
| `external_event_id` | `String` | NOT NULL |
| `external_event_type` | `String` | NOT NULL |
| `received_at` | `DateTime` | NOT NULL, def `now()` |
| `payload` | `Json` | NOT NULL |

- `UNIQUE` (provider, external_event_id)
- `idx` (received_at)

### `workflow_execution`

<sub>model `WorkflowExecution`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `workflow_type` | `String` | NOT NULL |
| `status` | `WorkflowStatus` (enum) | NOT NULL, def `PENDING` |
| `tenant_id` | `String` | NOT NULL, `Uuid` |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `payload` | `Json` | NOT NULL, def `"{}"` |
| `total_items` | `Int` | NOT NULL, def `0` |
| `processed_items` | `Int` | NOT NULL, def `0` |
| `failed_items` | `Int` | NOT NULL, def `0` |
| `warning_items` | `Int` | NOT NULL, def `0` |
| `correlation_id` | `String` | NOT NULL, `Uuid` |
| `graphile_job_id` | `String` | **UQ**, null |
| `execution_mode` | `String` | NOT NULL, def `"sequential"` |
| `stalled_replay_count` | `Int` | NOT NULL, def `0` |
| `last_processed_items_snapshot` | `Int` | null |
| `auth_token` | `String` | null, `Text` |
| `error_code` | `String` | null |
| `error_message` | `String` | null, `Text` |
| `original_error` | `Json` | null |
| `error_stack_trace` | `String` | null, `Text` |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |
| `completed_at` | `DateTime` | null |

- `idx` (correlation_id)
- `idx` (tenant_id)
- `idx` (user_id)
- `idx` (status, created_at)
- `idx` (workflow_type, status)

<sub>FK/relations: tasks->WorkflowTask[]</sub>

### `workflow_rate_limit`

<sub>model `WorkflowRateLimit`</sub>

| column | type | rules |
|---|---|---|
| `task_type` | `String` | NOT NULL |
| `window_start` | `DateTime` | NOT NULL |
| `counter` | `Int` | NOT NULL, def `0` |

- `PK` (task_type, window_start)
- `idx` (window_start)

### `workflow_task`

<sub>model `WorkflowTask`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `workflow_execution_id` | `String` | NOT NULL, `Uuid` |
| `task_type` | `String` | NOT NULL |
| `graphile_job_id` | `String` | **UQ**, null |
| `item_index` | `Int` | null |
| `payload` | `Json` | NOT NULL |
| `result` | `Json` | null |
| `task_hash` | `String` | null |
| `sequence_number` | `Int` | NOT NULL, def `0` |
| `scheduled_for` | `DateTime` | null |
| `status` | `WorkflowTaskStatus` (enum) | NOT NULL, def `PENDING` |
| `correlation_id` | `String` | NOT NULL, `Uuid` |
| `started_at` | `DateTime` | NOT NULL, def `now()` |
| `completed_at` | `DateTime` | null |

- `UNIQUE` (task_hash)
- `idx` (workflow_execution_id, status)
- `idx` (correlation_id)
- `idx` (graphile_job_id)

<sub>FK/relations: workflowExecution->WorkflowExecution; errors->WorkflowTaskError[]</sub>

### `workflow_task_error`

<sub>model `WorkflowTaskError`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `task_id` | `String` | NOT NULL, `Uuid` |
| `timestamp` | `DateTime` | NOT NULL, def `now()` |
| `attempt_number` | `Int` | NOT NULL, def `1` |
| `code` | `String` | NOT NULL |
| `message` | `String` | NOT NULL, `Text` |
| `severity` | `ErrorSeverity` (enum) | NOT NULL |
| `original_error` | `Json` | null |
| `stack_trace` | `String` | null, `Text` |
| `retryable` | `Boolean` | NOT NULL |
| `suggested_action` | `String` | null, `Text` |
| `documentation_url` | `String` | null |
| `item_data` | `Json` | null |
| `related_ids` | `Json` | null |
| `correlation_id` | `String` | NOT NULL, `Uuid` |

- `idx` (task_id)
- `idx` (task_id, attempt_number)
- `idx` (code)
- `idx` (correlation_id)

<sub>FK/relations: task->WorkflowTask</sub>

## schema - users & shared lookups

<sub>`schema.prisma` - 4 tables</sub>

### `intercom_sync_status`

<sub>model `IntercomSyncStatus`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `employer_id` | `String` | **UQ**, null, `Uuid` |
| `agency_id` | `String` | **UQ**, null, `Uuid` |
| `last_successful_sync` | `DateTime` | null |

<sub>FK/relations: employer->Employer; agency->Agency</sub>

### `state_carrier_rate_change`

<sub>model `StateCarrierRateChange`</sub>

| column | type | rules |
|---|---|---|
| `state_code` | `USState` (enum) | NOT NULL |
| `rate_change` | `Decimal` | NOT NULL, `Decimal(38, 18)` |
| `carrier_name` | `String` | NOT NULL, `VarChar(255)` |

- `PK` (state_code, carrier_name)

### `user`

<sub>model `User`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, `Uuid` |
| `tenant_id` | `String` | NOT NULL, `Uuid` |
| `first_name` | `String` | null |
| `last_name` | `String` | null |
| `email` | `String` | **UQ**, NOT NULL |
| `profile_picture_url` | `String` | null |
| `phone_number` | `String` | null |
| `status` | `UserStatus` (enum) | NOT NULL |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

<sub>FK/relations: employee->Employee; agent->Agent; userActivationLink->UserActivationLink[]; userRoles->UserRole[]; csmEmployers->Employer[]</sub>

### `manually_validated_addresses`

<sub>model `manuallyValidatedAddresses`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | **PK**, NOT NULL, def `uuid()`, `Uuid` |
| `formattedAddress` | `String` | **UQ**, NOT NULL |
| `state_code` | `USState` (enum) | NOT NULL |
| `city` | `String` | NOT NULL |
| `zip_code` | `String` | NOT NULL |
| `fips_county_code` | `String` | NOT NULL |
| `street_name` | `String` | NOT NULL |
| `street_number` | `String` | NOT NULL |
| `subpremise` | `String` | null |
| `created_at` | `DateTime` | NOT NULL, def `now()` |
| `updated_at` | `DateTime` | NOT NULL, auto |

## reports - database VIEWS

<sub>`reports.prisma` - 3 tables</sub>

### `employee_report` - VIEW

<sub>model `EmployeeReport`</sub>

| column | type | rules |
|---|---|---|
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employee_id_from_employer` | `String` | null |
| `employee_first_name` | `String` | NOT NULL |
| `employee_last_name` | `String` | NOT NULL |
| `employee_full_name` | `String` | NOT NULL |
| `email` | `String` | NOT NULL |
| `date_of_birth` | `String` | NOT NULL |
| `employee_custom_class_category` | `String` | null |
| `employee_eligible_from` | `String` | null |
| `employee_eligible_until` | `String` | null |
| `employee_termination_date` | `String` | null |
| `employee_created_at` | `DateTime` | NOT NULL |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `sub_entity_name` | `String` | null |
| `sub_entity_name_2` | `String` | null |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `invitation_status` | `UserStatus` (enum) | NOT NULL |
| `employee_role` | `EmployeeRole` (enum) | NOT NULL |
| `leave_of_absence_start_date` | `String` | null |
| `leave_of_absence_end_date` | `String` | null |
| `benefit_eligibility` | `EmploymentStatus` (enum) | null |
| `has_previous_eligibility` | `Boolean` | null |
| `onboarding_period_id` | `String` | null, `Uuid` |
| `onboarding_period_id_key` | `String` | NOT NULL, `Uuid` |
| `is_special_enrollment` | `Boolean` | null |
| `onboarding_period_is_active` | `Boolean` | null |
| `onboarding_from` | `String` | null |
| `onboarding_until` | `String` | null |
| `class` | `String` | null |
| `onboarding_period_allowance` | `Int` | null |
| `enrollment_submission_date` | `DateTime` | null |
| `onboarding_period_is_waived` | `Boolean` | null |
| `enrollment_type` | `OnboardingType` (enum) | null |
| `open_enrollment_period_id` | `String` | null, `Uuid` |
| `open_enrollment_effective_from` | `String` | null |
| `open_enrollment_effective_until` | `String` | null |
| `benefit_id` | `String` | null, `Uuid` |
| `benefit_type` | `BenefitType` (enum) | null |
| `benefit_family_unit` | `FamilyUnit` (enum) | null |
| `benefit_status` | `BenefitStatus` (enum) | null |
| `enrollment_effective_from` | `String` | null |
| `enrollment_effective_until` | `String` | null |
| `enrollment_submission_type` | `SubmissionType` (enum) | null |
| `enrollment_status` | `EnrollmentStatus` (enum) | NOT NULL |
| `medical_plan_id` | `String` | null |
| `medical_plan_name` | `String` | null |
| `medical_carrier` | `String` | null |
| `medical_premium` | `Decimal` | null, `Decimal(12, 2)` |
| `medical_employee_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `medical_employer_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `medical_self_pay_amount` | `Decimal` | null, `Decimal(12, 2)` |
| `medical_is_hsa_eligible` | `Boolean` | null |
| `medical_is_combined_plan` | `Boolean` | null |
| `medical_plan_market` | `PlanMarket` (enum) | null |
| `medical_self_report_type` | `SelfReportType` (enum) | null |
| `medical_self_enroll` | `SelfEnrollType` (enum) | null |
| `medical_plan_type` | `PolicyType` (enum) | null |
| `state` | `USState` (enum) | null |
| `dependent_count` | `Int` | null |
| `qle_id` | `String` | null, `Uuid` |
| `qle_type` | `QualifyingLifeEventType` (enum) | null |
| `qle_other_reason` | `String` | null |
| `qle_occurred_on` | `String` | null |
| `initial_premium_payment_status` | `InitialPremiumPaymentStatus` (enum) | null |
| `auto_pay_status` | `AutoPayStatus` (enum) | null |
| `payment_method_type` | `EmployeePaymentMethodType` (enum) | null |
| `enrollment_team_id` | `String` | null, `Uuid` |
| `enrollment_team_name` | `String` | null |
| `enrollment_agent_first_name` | `String` | null |
| `enrollment_agent_last_name` | `String` | null |
| `enrollment_agent_name` | `String` | null |
| `enrollment_agent_email` | `String` | null |
| `address` | `String` | null |
| `phone` | `String` | null |
| `personal_email` | `String` | null |
| `ssn` | `String` | null |
| `employment_type` | `EmploymentType` (enum) | null |
| `hire_date` | `String` | null |
| `is_open_enrollment_view` | `Boolean` | null |
| `is_latest_coverage_view` | `Boolean` | null |
| `is_enrollments_in_process_view` | `Boolean` | null |
| `is_all_view` | `Boolean` | null |

- `UNIQUE` (employee_id, onboarding_period_id_key)

### `enrollment_report` - VIEW

<sub>model `EnrollmentReport`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | NOT NULL, `Uuid` |
| `allowance` | `Int` | null |
| `is_special_enrollment` | `Boolean` | NOT NULL |
| `enrollment_tags` | `EnrollmentTag` (enum) [] | NOT NULL |
| `onboarding_from` | `String` | NOT NULL |
| `onboarding_until` | `String` | NOT NULL |
| `is_combined_plan` | `Boolean` | null |
| `plan_market` | `PlanMarket` (enum) | null |
| `employee_full_name` | `String` | NOT NULL |
| `is_qle_enrollment` | `Boolean` | NOT NULL |
| `enrollment_agent_name` | `String` | null |
| `is_active` | `Boolean` | NOT NULL |
| `has_user_logged_in` | `Boolean` | NOT NULL |
| `effective_from` | `String` | NOT NULL |
| `effective_until` | `String` | NOT NULL |
| `benefit_status` | `BenefitStatus` (enum) | null |
| `enrollment_status` | `EnrollmentStatus` (enum) | NOT NULL |
| `renewal_status` | `RenewalStatus` (enum) | NOT NULL |
| `aor_cancellation_status` | `AorCancellationStatus` (enum) | NOT NULL |
| `carrier_request_status` | `CarrierRequestStatus` (enum) | null |
| `employee_id` | `String` | NOT NULL, `Uuid` |
| `employee_first_name` | `String` | NOT NULL |
| `employee_last_name` | `String` | NOT NULL |
| `email` | `String` | NOT NULL |
| `date_of_birth` | `String` | NOT NULL |
| `employee_state` | `USState` (enum) | null |
| `employee_class` | `String` | NOT NULL |
| `employee_id_from_employer` | `String` | null |
| `user_id` | `String` | NOT NULL, `Uuid` |
| `employee_eligible_from` | `String` | null |
| `employee_eligible_until` | `String` | null |
| `employee_custom_class_category` | `String` | null |
| `employee_termination_date` | `String` | null |
| `enrollment_period_id` | `String` | NOT NULL, `Uuid` |
| `enrollment_period_effective_from` | `String` | null |
| `employer_id` | `String` | NOT NULL, `Uuid` |
| `employer_name` | `String` | NOT NULL |
| `is_demo_employer` | `Boolean` | NOT NULL |
| `family_unit` | `FamilyUnit` (enum) | null |
| `is_waived` | `Boolean` | NOT NULL |
| `benefit_id` | `String` | null, `Uuid` |
| `major_medical_plan_benefit_id` | `String` | null, `Uuid` |
| `plan_id` | `String` | null |
| `carrier_name` | `String` | null |
| `plan_name` | `String` | null |
| `premium` | `Decimal` | null, `Decimal(12, 2)` |
| `employee_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `employer_contribution` | `Decimal` | null, `Decimal(12, 2)` |
| `self_enroll` | `SelfEnrollType` (enum) | null |
| `policy_type` | `PolicyType` (enum) | null |
| `qle_type` | `QualifyingLifeEventType` (enum) | null |
| `enrollment_team_name` | `String` | null |
| `enrollment_team_id` | `String` | null, `Uuid` |
| `enrollment_agent_first_name` | `String` | null |
| `enrollment_agent_last_name` | `String` | null |
| `enrollment_agent_email` | `String` | null |
| `enrollment_agent_id` | `String` | null, `Uuid` |
| `initial_premium_payment_status` | `InitialPremiumPaymentStatus` (enum) | null |
| `auto_pay_status` | `AutoPayStatus` (enum) | null |
| `monthly_auto_pay_date` | `Int` | null, `SmallInt` |
| `is_monthly_auto_pay_date_end_of_month` | `Boolean` | null |
| `metal_level` | `MetalLevel` (enum) | null |
| `leave_of_absence_start_date` | `String` | null |
| `leave_of_absence_end_date` | `String` | null |
| `submitted_at` | `DateTime` | null |
| `enrollment_confirmed_at` | `DateTime` | null |
| `application_sent_at` | `DateTime` | null |
| `application_sent_by_name` | `String` | null |
| `confirmed_by_name` | `String` | null |

- `UNIQUE` (id)

### `prospect_report` - VIEW

<sub>model `ProspectReport`</sub>

| column | type | rules |
|---|---|---|
| `id` | `String` | NOT NULL, `Uuid` |
| `is_demo` | `Boolean` | NOT NULL |
| `name` | `String` | NOT NULL |
| `state` | `String` | null |
| `created_at` | `DateTime` | NOT NULL |
| `prospect_coverage_start_date` | `String` | null |
| `estimated_employees` | `Int` | null |
| `estimated_participation_rate` | `Int` | null |
| `zorro_partner_name` | `String` | null |
| `producer_name` | `String` | null |
| `producer_agent_name` | `String` | null |
| `enrollment_team_names` | `String` [] | NOT NULL |

- `UNIQUE` (id)


# Part 11 — Appendix: all 115 enum values

**`AccessLevel`** (3) - `STANDARD`, `ACCOUNT_MANAGEMENT`, `ADMIN`

**`AccountType`** (2) - `CONTRIBUTION_FBO`, `RESERVE_FBO`

**`ActivationRedirectUrlOptions`** (4) - `ZORRO_UI`, `ZORRO_OPS`, `ADD_EMPLOYER_GET_STARTED`, `CANCEL_INVITATION`

**`ActualTransactionCategory`** (2) - `DISBURSEMENT`, `RETURN`

**`ActualTransactionMatchStatus`** (2) - `CANDIDATE`, `CONFIRMED`

**`AddressResolveOutcome`** (4) - `SUCCESS`, `EMPTY`, `PROVIDER_ERROR`, `TIMEOUT`

**`AddressResolveProvider`** (4) - `GOOGLE_ADDRESS_VALIDATION`, `CENSUS_GEOCODER`, `GOOGLE_GEOCODING`, `IDEON`

**`AgencyType`** (3) - `PRODUCER`, `ENROLLMENT_TEAM`, `ZORRO_PARTNER`

**`Aggregator`** (3) - `STATE`, `RATING_AREA`, `NONE`

**`AllowanceUnit`** (2) - `PERCENT`, `DOLLAR`

**`AnticipatedCareLevel`** (9) - `VERY_LOW`, `LOW`, `MEDIUM`, `HIGH`, `VERY_HIGH`, `ESSENTIAL`, `MODERATE`, `ABOVE_AVERAGE`, `EXTENSIVE`

**`AorCancellationStatus`** (3) - `NOT_APPLICABLE`, `CONFIRMED_BY_EMPLOYEE`, `AWAITING_EMPLOYEE`

**`ApplicationSubmissionMethod`** (7) - `MANUAL`, `PDF`, `BULK_CSV`, `SELF_ENROLLED`, `DEEPLINK`, `ENROLL_CONNECT`, `OSCAR`

**`AutoPayStatus`** (4) - `NOT_SET_UP_BY_EMPLOYEE`, `NOT_SET_UP_BY_AGENT`, `SET_UP_BY_EMPLOYEE`, `SET_UP_BY_AGENT`

**`BenefitDocumentType`** (9) - `SELF_REPORT`, `APPLICATION`, `ENROLLMENT`, `WAIVE`, `CONFIRMATION`, `QLE`, `ELECTION_CONSENT`, `APPLICATION_CONSENT`, `OTHER`

**`BenefitStatus`** (9) - `IN_CART`, `AWAITING_MEDICAL`, `READY_TO_APPLY`, `AWAITING_EMPLOYEE`, `CARRIER_APPLICATION_SENT`, `AWAITING_PAYMENT`, `ENROLLMENT_CONFIRMED`, `ACTIVE`, `ENDED`

**`BenefitType`** (11) - `MAJOR_MEDICAL`, `SUPPLEMENTAL_ACCIDENT`, `SUPPLEMENTAL_ACCIDENTAL_DEATH_AND_DISMEMBERMENT`, `SUPPLEMENTAL_COMBINED_DENTAL_VISION`, `SUPPLEMENTAL_CRITICAL_ILLNESS`, `SUPPLEMENTAL_DENTAL`, `SUPPLEMENTAL_FLEXIBLE_SPENDING_ACCOUNT`, `SUPPLEMENTAL_HOSPITAL`, `SUPPLEMENTAL_LIFE`, `SUPPLEMENTAL_OTHER`, `SUPPLEMENTAL_VISION`

**`BlendedRateEstimateSource`** (3) - `CARRIER_SPECIFIC_REQUESTED`, `CARRIER_SPECIFIC_APPROVED`, `STATE_MEDIAN`

**`BlendedRateIssuerMatchType`** (2) - `HIOS_SPECIFIC`, `PARENT_ISSUER`

**`BlendedRateLookupMethod`** (3) - `ESTIMATED`, `ACTUAL`, `REMOVED`

**`BusinessEntityType`** (10) - `C_CORPORATION`, `S_CORPORATION`, `NON_PROFIT_ORGANIZATION`, `PARTNERSHIP`, `LIMITED_LIABILITY_CORPORATION`, `LIMITED_LIABILITY_PARTNERSHIP`, `SOLE_PROPRIETORSHIP`, `UNION`, `GOVERNMENT_AGENCY`, `OTHER`

**`CarrierPaymentMethod`** (2) - `ACH`, `CARD`

**`CarrierRequestStatus`** (3) - `AWAITING_EMPLOYEE`, `CONFIRMED_BY_EMPLOYEE`, `NOT_APPLICABLE`

**`CitizenshipStatus`** (3) - `US_CITIZEN`, `RESIDENT_ALIEN`, `OTHER`

**`ClassType`** (3) - `EMPLOYMENT_TYPE`, `WAGE_TYPE`, `NONE`

**`ContactType`** (4) - `LEGAL`, `HR`, `FINANCE`, `ONBOARDING`

**`ContributionGroup`** (2) - `INDIVIDUALS`, `TIERS`

**`ContributionMode`** (2) - `SIMPLE`, `ADVANCED`

**`ContributionPreset`** (6) - `RECOMMENDED`, `INDUSTRY`, `CURRENT`, `COMPLIANT`, `GENEROUS`, `NONE`

**`ContributionRequestStatus`** (4) - `PENDING`, `SENT`, `SETTLED`, `REJECTED`

**`CostAttribute`** (3) - `DEDUCTIBLE`, `OOP_MAX`, `EMPLOYEE_MONTHLY_CONTRIBUTION`

**`CostPreferenceImportance`** (3) - `LOW`, `MEDIUM`, `HIGH`

**`CountyResolutionSource`** (5) - `US_CENSUS`, `GOOGLE_GEOCODER`, `DB_AUTO`, `DB_USER`, `MANUAL_OVERRIDE`

**`CreationStatus`** (3) - `PENDING`, `CREATED`, `ACTIVATED`

**`DecisionFactors`** (5) - `MINIMIZE_ALL_IN_COST`, `MINIMIZE_RISK`, `KEEP_DOCTORS`, `KEEP_MEDICATIONS`, `FREE_REFERRALS`

**`DocumentType`** (5) - `PLAN`, `ADMIN`, `COMPLIANCE`, `ZORRO_PAY`, `INVOICE`

**`EmployeePaymentMethodStatus`** (4) - `EMPTY`, `VALID`, `INVALID`, `NEEDS_HANDLING`

**`EmployeePaymentMethodType`** (3) - `ZORRO_PAY`, `COMBINED`, `SELF_PAY`

**`EmployeePaymentsAlertType`** (5) - `COVERAGE_DATES_CHANGED`, `BENEFIT_NO_LONGER_FUNDABLE`, `ZORRO_PAY_AMOUNT_CHANGED`, `TRANSACTION_DECLINED`, `TRANSACTION_RETURNED`

**`EmployeeRole`** (3) - `EMPLOYEE`, `EMPLOYER_ADMIN`, `OTHER`

**`EmployerSignupStatus`** (6) - `PROSPECT`, `DRAFT`, `WAITING_FOR_EMPLOYER`, `NEEDS_REVIEW`, `IN_ZORRO_SETUP`, `ACTIVE`

**`EmploymentStatus`** (7) - `ELIGIBLE_EMPLOYED`, `ELIGIBLE_UPCOMING_LEAVE`, `INELIGIBLE_WAITING`, `INELIGIBLE_ON_LEAVE`, `ELIGIBLE_ENDING`, `INELIGIBLE_TERMINATED`, `INELIGIBLE`

**`EmploymentType`** (2) - `FULL_TIME`, `PART_TIME`

**`EnrollmentActivity`** (31) - `SUBMITTED`, `WAIVED`, `WAIVER_UPDATE`, `RESET`, `APPLICATION_SENT`, `ENROLLMENT_CONFIRMED`, `ENROLLMENT_DETAILS_UPDATE`, `PREMIUM_ADJUSTMENT`, `ELECTION_DETAILS_UPDATE`, `APPLICATION_REVERTED`, `ENROLLMENT_CONFIRMATION_REVERTED`, `EFFECTIVE_FROM_UPDATE`, `EFFECTIVE_UNTIL_UPDATE`, `EMPLOYEE_MONTHLY_CONTRIBUTION_UPDATE`, `EMPLOYER_MONTHLY_CONTRIBUTION_UPDATE`, `SELF_PAY_AMOUNT_UPDATE`, `PREMIUM_UPDATE`, `ENROLLMENT_CREATED`, `ENROLLMENT_TEAM_ASSIGNED`, `ENROLLMENT_AGENT_ASSIGNED`, `ENROLLMENT_TAGS_APPLIED`, `ENROLLMENT_TAGS_REMOVED`, `COVERED_MEMBERS_UPDATE`, `BENEFIT_REMOVED`, `INITIAL_PREMIUM_PAYMENT_COMPLETED`, `ENROLLMENT_CALL_LOGGED`, `INITIAL_PREMIUM_PAYMENT_STATUS_UPDATE`, `AUTO_PAY_STATUS_UPDATE`, `MONTHLY_AUTO_PAY_DATE_UPDATE`, `ALLOWANCE_UPDATE`, `CARRIER_APPLICATION_DETAILS_UPDATE`

**`EnrollmentStatus`** (13) - `PENDING_ELECTION_WINDOW`, `ELECTION_ACTIVE`, `ELECTION_ACTIVE_HAS_NOT_STARTED`, `ELECTION_ACTIVE_STARTED`, `ELECTION_SUBMITTED`, `WAIVED_ELECTION`, `DEADLINE_PASSED`, `ENROLLMENT_CONFIRMED`, `CARRIER_APPLICATION_SENT`, `COVERAGE_ENDED`, `ACTIVE_COVERAGE`, `WAIVED_COVERAGE`, `NO_ENROLLMENTS`

**`EnrollmentTag`** (9) - `PENDING_CARRIER`, `PENDING_EMPLOYEE`, `PENDING_EMPLOYER`, `PENDING_AGENT`, `SUBMIT_FAILED`, `ADDRESS_ISSUE`, `PM_IN_HANDLING`, `READY_TO_SUBMIT`, `AOR_NEEDED`

**`EntityView`** (2) - `EMPLOYER`, `EMPLOYEE`

**`ErrorSeverity`** (3) - `WARNING`, `ERROR`, `CRITICAL`

**`EventSource`** (2) - `BROWSER`, `SERVER`

**`ExpectedAmountSource`** (2) - `INITIAL_EXPECTATION`, `CASE_HANDLING`

**`FamilyUnit`** (4) - `EMPLOYEE_ONLY`, `EMPLOYEE_SPOUSE`, `EMPLOYEE_CHILD`, `FAMILY`

**`FamilyUnitDistribution`** (2) - `EIGHT`, `FOUR`

**`FinchConnectionStatus`** (5) - `CONNECTED`, `PENDING`, `DISCONNECTED`, `REAUTHENTICATE`, `ERROR`

**`FinchDeductionAction`** (3) - `CREATE`, `REGISTER`, `UNKNOWN`

**`FinchDeductionHsaEligible`** (3) - `NOT_APPLICABLE`, `ELIGIBLE`, `NOT_ELIGIBLE`

**`FinchDeductionTax`** (3) - `NOT_APPLICABLE`, `PRE_TAX`, `POST_TAX`

**`FinchEmployeeSyncType`** (3) - `SSN`, `EMAIL`, `MANUAL`

**`FinchEnrollmentAction`** (2) - `ENROLL`, `UNENROLL`

**`FinchIntegrationType`** (2) - `AUTOMATED`, `MANUAL`

**`FinchJobStatus`** (6) - `pending`, `in_progress`, `error`, `complete`, `reauth_error`, `permissions_error`

**`FinchNotificationType`** (4) - `MATCHED_EMPLOYEE_INACTIVE_IN_FINCH`, `EMPLOYEE_INACTIVE_IN_FINCH`, `EMPLOYEE_MATCHED_BY_EMAIL`, `EMPLOYEE_NOT_MATCHED_IN_FINCH`

**`FinchPayFrequency`** (9) - `annually`, `semi_annually`, `quarterly`, `monthly`, `semi_monthly`, `bi_weekly`, `weekly`, `daily`, `other`

**`FinchPayGroupBehavior`** (3) - `DEFAULT`, `SINGLE_PAY_GROUP`, `ESTIMATE_MULTI_PAY_GROUP`

**`FinchSyncStatus`** (2) - `SUCCESS`, `FAILURE`

**`FinchUnenrollReason`** (5) - `SCHEDULE`, `ENROLLMENT_CHANGE`, `ENROLLMENT_REMOVED`, `EMPLOYEE_REMOVED`, `RESET`

**`Gender`** (2) - `MALE`, `FEMALE`

**`GeoDistribution`** (3) - `OFF`, `AREA`, `STATES`

**`HealthSherpaApplicationProvider`** (2) - `ENROLL_CONNECT`, `DEEPLINK`

**`HealthSherpaApplicationStatus`** (9) - `PENDING_CREATE`, `LINK_GENERATED`, `DRAFT`, `PENDING_EFFECTUATION`, `EFFECTUATED`, `SUBMISSION_FAILED`, `PENDING_CANCELLATION`, `CANCELLED`, `TERMINATED`

**`HealthSherpaPaymentStatus`** (6) - `UNPAID_BINDER`, `PAID_BINDER`, `PAID`, `PAST_DUE`, `CANCELLED`, `TERMINATED`

**`HealthSherpaPolicyStatus`** (8) - `DRAFT`, `PENDING_EFFECTUATION`, `EFFECTUATED`, `SUBMISSION_FAILED`, `CANCELLED`, `TERMINATED`, `SEP_DOCS_REQUIRED`, `SEP_DOCS_UNDER_REVIEW`

**`InitialPremiumPaymentStatus`** (6) - `COMPLETED_BY_EMPLOYEE`, `COMPLETED_BY_AGENT`, `NOT_SET_UP_BY_EMPLOYEE`, `NOT_SET_UP_BY_AGENT`, `SET_UP_BY_EMPLOYEE`, `SET_UP_BY_AGENT`

**`InsuredSubtype`** (4) - `SPOUSE`, `DOMESTIC_PARTNER`, `CHILD`, `OTHER_DEPENDENT`

**`InsuredType`** (3) - `EMPLOYEE`, `SPOUSE_OR_DOMESTIC_PARTNER`, `CHILD_OR_OTHER_DEPENDENT`

**`MaritalStatus`** (3) - `SINGLE`, `MARRIED`, `IN_DOMESTIC_PARTNERSHIP`

**`MetalLevel`** (6) - `CATASTROPHIC`, `BRONZE`, `EXPANDED_BRONZE`, `SILVER`, `GOLD`, `PLATINUM`

**`Month`** (12) - `JANUARY`, `FEBRUARY`, `MARCH`, `APRIL`, `MAY`, `JUNE`, `JULY`, `AUGUST`, `SEPTEMBER`, `OCTOBER`, `NOVEMBER`, `DECEMBER`

**`NetworkType`** (5) - `HMO`, `POS`, `EPO`, `PPO`, `INDEMNITY`

**`OEPPaymentMethod`** (2) - `SELF_PAY`, `ZORRO_PAY`

**`OnboardingPeriodType`** (2) - `SHOPPING`, `REPORTING`

**`OnboardingType`** (2) - `OPEN_ENROLLMENT`, `SPECIAL`

**`PaymentBy`** (2) - `EMPLOYER`, `EMPLOYEE`

**`PaymentRequestStatus`** (8) - `SCHEDULED`, `AWAITING_DEPENDENCY`, `PENDING`, `PROCESSING`, `SUBMITTED`, `SUCCEEDED`, `FAILED`, `CANCELED`

**`PaymentRequestType`** (4) - `TRANSFER_CONTRIBUTION`, `TRANSFER_ALLOCATION`, `TRANSFER_DEALLOCATION`, `BOOK_TRANSFER`

**`PaymentResponsibility`** (2) - `ENROLLMENT_AGENT`, `EMPLOYEE`

**`PaymentsCycle`** (3) - `INITIAL`, `MONTHLY`, `IMMEDIATE`

**`PaymentsVendor`** (1) - `LYNX`

**`PayrollActionType`** (6) - `CHANGE_AMOUNTS`, `CHANGE_AMOUNTS_MID_MONTH`, `ADD_AMOUNTS`, `REMOVE_AMOUNTS`, `REMOVE_AMOUNTS_MID_MONTH`, `RETROACTIVE_CHANGE`

**`PayrollCycle`** (6) - `WEEKLY`, `WEEKLY_DEDUCTIONS`, `BIWEEKLY`, `BIWEEKLY_DEDUCTIONS`, `MONTHLY`, `SEMI_MONTHLY`

**`PayrollPayFrequency`** (4) - `WEEKLY`, `BI_WEEKLY`, `SEMI_MONTHLY`, `MONTHLY`

**`PayrollPayPeriodSource`** (2) - `PROVIDER_PUBLISHED`, `PROJECTED`

**`PlanMarket`** (3) - `OFF_EXCHANGE`, `ON_EXCHANGE`, `BOTH_MARKETS`

**`PlanType`** (2) - `INDIVIDUAL_MARKET`, `CURRENT`

**`PolicyType`** (3) - `MEDICARE`, `IFP`, `MEDICARE_IFP`

**`PreferredDirection`** (2) - `PREFER_LOWER`, `PREFER_HIGHER`

**`QualifyingLifeEventType`** (7) - `MARRIED`, `DIVORCED`, `NEW_DEPENDENT`, `DEATH_IN_FAMILY`, `RELOCATION`, `ELIGIBILITY`, `OTHER`

**`QuotePlanPriorities`** (6) - `COMPANY_COSTS`, `TOP_BENEFITS`, `EMPLOYEE_COSTS`, `COMPLIANCE`, `INDUSTRY_STANDARD`, `SIMILAR_CARE`

**`RenewalStatus`** (4) - `NOT_APPLICABLE`, `WAIVED`, `RENEWED`, `CHANGED`

**`ReviewSnapshotType`** (3) - `ELECTION`, `RENEWAL`, `MANUAL_ELECTION`

**`Role`** (9) - `ANY`, `OMNIPOTENT_ADMIN`, `OPERATOR`, `AGENCY_ADMIN`, `AGENT`, `ACCOUNT_SUPERVISOR`, `EMPLOYER_ADMIN`, `EMPLOYEE`, `USER`

**`Scenario`** (14) - `BASIC`, `ENHANCED`, `ESSENTIAL`, `STANDARD`, `EXCEPTIONAL`, `LOW_COST_GOLD`, `LOW_COST_SILVER`, `LOW_COST_PLATINUM`, `ON_MARKET_LOW_COST_SILVER`, `SIMILAR_CARE`, `SAME_ICHRA_PLAN`, `TARGET_PLAN`, `RENEWAL_BASELINE`, `PRIOR_YEAR_ON_MARKET_LOW_COST_SILVER`

**`SelfEnrollType`** (3) - `SELF_ENROLL`, `NOT_SELF_ENROLL`, `UNKNOWN`

**`SelfReportType`** (4) - `NEW_ENTRY`, `CONFIRMED`, `UPDATED`, `NOT_APPLICABLE`

**`SettlementStatus`** (4) - `PENDING`, `SETTLED`, `RETURNED`, `FAILED`

**`SetupStatus`** (3) - `DRAFT`, `PROCESSING`, `COMPLETED`

**`ShoppingPreferenceType`** (1) - `FINANCIAL_RISK_TOLERANCE`

**`SubmissionType`** (7) - `BY_EMPLOYEE`, `BY_EMPLOYER_ADMIN`, `BY_AGENT`, `BY_OPERATOR`, `SELF_REPORTED`, `AUTOMATIC`, `NA`

**`TaxIdType`** (2) - `EIN`, `ITIN`

**`USState`** (51) - `AL`, `AK`, `AZ`, `AR`, `CA`, `CO`, `CT`, `DC`, `DE`, `FL`, `GA`, `HI`, `ID`, `IL`, `IN`, `IA`, `KS`, `KY`, `LA`, `ME`, `MD`, `MA`, `MI`, `MN`, `MS`, `MO`, `MT`, `NE`, `NV`, `NH`, `NJ`, `NM`, `NY`, `NC`, `ND`, `OH`, `OK`, `OR`, `PA`, `RI`, `SC`, `SD`, `TN`, `TX`, `UT`, `VT`, `VA`, `WA`, `WV`, `WI`, `WY`

**`UserStatus`** (4) - `PENDING_INVITATION`, `PENDING_LOGIN`, `ACTIVATED`, `NOT_ACTIVATED`

**`WageType`** (2) - `SALARY`, `HOURLY`

**`WaitingPeriod`** (4) - `NONE`, `THIRTY_DAYS`, `SIXTY_DAYS`, `NINETY_DAYS`

**`WorkflowStatus`** (5) - `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`, `CANCELLED`

**`WorkflowTaskStatus`** (5) - `PENDING`, `DEFERRED`, `RUNNING`, `COMPLETED`, `FAILED`

**`YesNoNotSure`** (3) - `YES`, `NO`, `NOT_SURE`

