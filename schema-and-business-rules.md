# Zorro Schema & Business Rules Reference

🌻 A complete per-table, per-column reference for the `zorro-ts` database, with
the business rules that govern each one.

**Generated from `main` at commit `f9cd6df3a` (2026-10-08).** Schema facts below
(columns, types, nullability, defaults, keys, indexes, cascades) are extracted
mechanically from `apps/monolith/prisma/schema/*.prisma` and are complete.
Business rules are hand-verified against migrations and service code; where a
rule lives only in application code and not the database, that is stated.

**Scope:** 132 tables/views · 1,755 columns · 115 enums · 12 schema files.

> **Companion:** [`data-and-business-guide.md`](./data-and-business-guide.md)
> explains *how the business works* (ICHRA, lifecycles, money flow). This file is
> the *reference*: what each table and column is, and what constrains it.
> [`entities.md`](./entities.md) and [`business-flows.md`](./business-flows.md)
> are stale (Dec 2025) — prefer these two.

---

# Part 1 — How to read the column tables

| Notation | Meaning |
|---|---|
| **PK** | primary key (`@id`) |
| **UQ** | column-level unique (`@unique`) |
| `NOT NULL` / `null` | nullability |
| def `x` | database default |
| `auto` | `@updatedAt` — written on every update |
| `Uuid`, `Decimal(12, 2)` | explicit Postgres type (`@db.*`) |
| (enum) | enum column — allowed values in the Appendix |
| `-> Model` | relation, not a stored column |

Column names shown are the **database** names (snake_case). The Prisma model
field name differs where `@map()` is used — see the model name under each
heading.

---

# Part 2 — Rules the database actually enforces

These are the only hard guarantees. Everything in Part 3 can be violated by a
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
is application-level.

## 2.2 Partial unique indexes

Only two, both on `payroll_individual` — uniqueness applies *only to matched
rows*, so many unmatched candidates may coexist:

```sql
CREATE UNIQUE INDEX "payroll_individual_matched_employee_id_key"
  ON "payroll_individual"("employee_id") WHERE "is_matched";
CREATE UNIQUE INDEX "payroll_individual_matched_provider_individual_id_key"
  ON "payroll_individual"("employer_id","provider_individual_id") WHERE "is_matched";
```

## 2.3 Year-scoped uniqueness — the renewal grain

Three tables share `UNIQUE (employer_id, year)`, the structural expression of
"rates reset each plan year":

| Table | Constraint |
|---|---|
| `allowance_model` | `UNIQUE (employer_id, year)` |
| `allowance_model_v2` | `UNIQUE (employer_id, year)` |
| `class_definition` | `UNIQUE (employer_id, year)` |

At most one of each per employer per year. `allowance_model` and
`allowance_model_v2` are independently unique — an employer may hold a row in
both for the same year.

## 2.4 Cascade deletes — 92 relations

Deletes are **hard** and they propagate. Soft delete exists in exactly one
place: `enrollment_comment.deleted_at`. Deleting an employer removes its
employees, periods, benefits, insured, quotes and payment records. There is no
`deleted_at` filter to remember and no recovery.

Counts by schema: administration 32 · prospects 22 · integrations 12 ·
shopping 11 · payments 7 · payroll 3 · infra 2 · schema 2 · payments-v1 1.

## 2.5 Compound foreign keys — the multi-tenancy pattern

Several tables denormalise `employer_id` / `employee_id` and reference parents on
compound keys — e.g. `insured.period_id + employee_id + employer_id` →
`onboarding_period(id, employee_id, employer_id)`. That is why
`onboarding_period` carries `UNIQUE (id, employer_id)` and
`UNIQUE (id, employee_id, employer_id)`: they exist as **FK targets**, not as
business keys.

**Consequence:** a `@@unique` containing `id` tells you nothing about business
cardinality. `onboarding_period` has two such constraints and still permits many
rows per employee per enrollment period.

The denormalised tenant columns also back Row-Level Security, which is enabled —
query results depend on the connecting role's context.

---

# Part 3 — Business rules enforced only in application code

212 `*Error` classes encode invariants the database does not. The ones that
change how you should read the data:

## 3.1 Cardinality

| Rule | Enforced by | DB guarantee? |
|---|---|---|
| Exactly one `EMPLOYEE` insured per onboarding period | `InsuredEmployeeAlreadyExistsError`, `CannotDeleteInsuredEmployeeError`, `InsuredNotFoundError` | **No** — there is no `UNIQUE (period_id, type)` |
| At most one spouse per period | `InsuredPeopleDto` resolves with `.find()` | **No** — duplicates silently ignored |
| One active onboarding period per employee | `findActivePeriodOrThrow` uses `findFirstOrThrow` | **No** — would pick arbitrarily |
| A benefit must cover at least one insured | `CannotDeleteAllInsuredOfBenefitError` | **No** — an empty junction is a valid table state |

## 3.2 Status transitions

`BenefitStatus` has no transition table. The rules live in
`benefit-status.service.ts`:

- `IN_CART` is terminal-backwards — nothing transitions into it (encoded as
  `Exclude<BenefitStatus, 'IN_CART'>` on the workflow task payload)
- `AWAITING_MEDICAL` to `CARRIER_APPLICATION_SENT` is "not possible" (code
  comment); it must pass through `READY_TO_APPLY`
- `ACTIVE` / `ENDED` are written **only** by the cron workflow:
  `ENROLLMENT_CONFIRMED AND effective_from <= today` becomes `ACTIVE`;
  `ACTIVE AND effective_until < today` becomes `ENDED` (note `<=` versus `<`)
- `AWAITING_EMPLOYEE` / `AWAITING_PAYMENT` writes are feature-flag gated and
  **fail silently** (plain return, no error, no log) when the flag is off

## 3.3 Date rules

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

## 3.4 Money rules

- allowance derives from the **family unit of submitted medical benefits**, and
  is wiped to `NULL` when no submitted medical remains
- allocation order: medical, then dental (incl. combined dental/vision), then
  standalone vision; employee-covered first, then by `plan.external_id`
- `ER = min(remaining_allowance, premium)` and `EE = max(premium - ER, 0)`
- all money math uses `Decimal` with banker's rounding (`ROUND_HALF_EVEN`, 2dp)
- purse funding: normal benefits at carrier application; **self-enroll benefits
  at election submission**

## 3.5 Payment platform exclusivity

Echo (`payments-v1`) and Lynx (`payments`) are **mutually exclusive per
employer**, gated by the `lynx_payments_enabled` flag. No Echo call may be made
for a Lynx employer at any layer. Lynx is the source of truth for payment data —
local tables hold reference keys only. Not expressible in schema.

---

# Part 4 — Derived values (not stored)

Recomputing these from raw columns will diverge from the product unless you
reuse the same functions.

| Value | Source | Note |
|---|---|---|
| `enrollment_status` | `calculate_enrollment_status(...)` | takes `today` — **the same row yields different values on different days** |
| `benefit_eligibility` (EmploymentStatus) | `calculate_employment_status(...)` | takes `today` |
| `family_unit` | `bool_or` over insured types via `_benefit_to_insured` | was a stored column until Feb 2026 (PR #7134), then dropped |
| `payment_method_type` | `CASE` on `self_pay_amount` vs `premium` in `employee_report` | `ZORRO_PAY` / `COMBINED` / `SELF_PAY` |
| FPL year | `resolve_fpl_year(year, start_of_coverage)` | previous year's FPL if coverage starts under 6 months after publication |

`_benefit_to_insured` is Prisma's implicit many-to-many join table (`"A"` =
`benefit.id`, `"B"` = `insured.id`). It appears in no `.prisma` file. It has no
`employer_id`, so it carries no RLS policy of its own.

## View row-grain warning

`employee_report` emits **one row per onboarding period**, not per employee. An
employee with a QLE has several rows. Four boolean selectors pick one:
`is_open_enrollment_view`, `is_latest_coverage_view`,
`is_enrollments_in_process_view`, `is_all_view`. **Querying without one of them
multi-counts employees.** `intercom_sync_report` instead forces one row per
period with `LEFT JOIN LATERAL ... ORDER BY created_at, id LIMIT 1`.

---

# Part 5 — Global column conventions

**Money** (see [`decimal-precision.md`](../tech/guidelines/decimal-precision.md)):

| Semantic | Type |
|---|---|
| Dollars (premiums, contributions, wages) | `Decimal @db.Decimal(12, 2)` |
| Rates, factors, percentages | `Decimal @db.Decimal(38, 18)` |
| Payments ledger | `Int`, column suffixed `_in_cents` |

Never `Float`; never a bare `@db.Decimal` (Prisma defaults it to `(65,30)`,
which exceeds the Iceberg/Parquet precision cap of 38 and breaks DMS replication
typing). Scale is permanent once data reaches the warehouse.

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

---

# Part 6 — Per-table reference

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


# Appendix - all 115 enums

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

