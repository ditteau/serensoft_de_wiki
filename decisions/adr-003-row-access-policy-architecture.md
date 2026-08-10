# ADR-003: Domain-Scoped Row Access Policy Architecture

**Status:** Proposed

**Date:** 2026-08-10

**Author:** LVP

---

### Context

The Ditteau platform serves multiple higher-education institutions and exposes
student, financial aid, admissions, and institutional budget data to a variety
of internal consumers — advisors, registrars, institutional research analysts,
financial aid officers, cost-center managers, and executive leadership. Each
consumer group has a legitimately different scope of row access, and several
scenarios involve data that one role should see but another explicitly should not.

Governance gap G-03 (Access Control, MEDIUM, P2) documents that no Row Access
Policies exist anywhere in the platform. The feature flag `enable_row_level_security`
is defined in `dbt_project.yml` but no downstream logic consumes it. Distribution-
layer models contain student-key-level FERPA data with only a documentation note
that "row access policy applies in PROD." In practice, any Snowflake role with
SELECT on a distribute-layer table can read all rows across all students and,
in a shared-database scenario, across institutions.

A naive RAP design — one monolithic policy function that encodes every role's
predicate logic — would be expensive to evaluate, fragile to extend, and
difficult to test. The platform must also distinguish row-level filtering from
column-level masking: the scenario where a president sees a student's full name
but an IR analyst sees only a hashed identifier is a masking concern, not a RAP
concern. Conflating the two leads to overbuilt RAP logic that should instead be
handled by Dynamic Data Masking (DDM) policies.

The governance policy (`docs/governance/data_governance_policy.md`) names five
target Snowflake roles — `ANALYST_ROLE`, `ADVISOR_ROLE`, `REGISTRAR_ROLE`,
`FA_ROLE`, `ADMIN_ROLE` — but these are aspirational. No seeds, no Snowflake
objects, and no dbt grants configuration exist to back them.

---

### Decision

We adopt a **three-layer access control architecture**. Each layer is independent
and separately configurable:

**Layer 1 — Object grants (Snowflake RBAC)**
Controls which tables and views a role can SELECT from at all. Implemented via
dbt `+grants:` config on distribute-layer models. This layer determines
object-level visibility — a role not granted SELECT never reaches layers 2 or 3.

**Layer 2 — Row Access Policies (RAPs)**
Controls which rows within a permitted object are visible to the current session.
Policies are **domain-scoped** — one RAP function per data domain
(`rap_student_academic`, `rap_budget`, `rap_financial_aid`), applied to all
tables in that domain via dbt post-hook macro. A single mega-RAP across all
domains is explicitly rejected (see Alternatives).

**Layer 3 — Dynamic Data Masking (DDM)**
Controls what column values are visible within accessible rows. Implemented
separately as G-01 (P1, HIGH). Examples: SSN masked for non-registrar roles,
DOB masked for analysts, student name hashed for IR cohort queries.

#### Governance schema

RAP functions and their supporting mapping tables live in each institution's own
database: `{SCHOOL}_DD_{ENV}.governance`. This is a deliberate choice against
placing them in `DITTEAU_SHARED` — access control configuration is inherently
institutional, not platform-wide, and per-school placement contains blast radius,
allows cross-school policy divergence, and keeps RAP function resolution within
a single database boundary.

`DITTEAU_SHARED` retains a `governance.role_definitions` reference table
documenting the standard platform roles and their intended purpose — a catalogue
only, not used for enforcement.

Per-school governance tables:

| Table | Grain | Purpose |
|---|---|---|
| `role_domain_access` | role × data_domain | Maps each Snowflake role to an access tier for this institution; schools may deviate from platform defaults |
| `advisor_student_map` | snowflake_username × student_id | Advisor roster for this institution; refreshed from SIS on each PROD build |
| `cost_center_manager_map` | snowflake_username × cost_center_code | Budget scope per manager for this institution |

Access tiers in `role_domain_access`:

| Tier | Meaning |
|---|---|
| `FULL` | All rows in the domain |
| `SCOPED` | Rows matching a user-specific predicate (advisor roster, cost center, etc.) |
| `AGGREGATED` | No row access — consume only through mart aggregations |
| `NONE` | Access denied at this layer |

A domain RAP function evaluates as follows:
1. Look up `CURRENT_ROLE()` in this school's `role_domain_access` for the relevant domain
2. If `FULL` → return TRUE
3. If `SCOPED` → join to the appropriate mapping table and evaluate the predicate
4. If `AGGREGATED` or `NONE` → return FALSE

#### dbt integration

- `macros/governance/apply_rap.sql` — post-hook macro that runs
  `ALTER TABLE ... ADD ROW ACCESS POLICY` conditionally on
  `var('enable_row_level_security', false)`
- The macro resolves the fully-qualified RAP function path using dbt vars:
  `{{ var('school_code') }}_DD_{{ var('env') }}.governance.rap_student_academic`
  — consistent with how all other school-scoped objects are addressed in this project
- Models declare their domain via `meta: {data_domain: 'student_academic'}` in YAML
- `enable_row_level_security` remains `false` in dev/test; set to `true` in PROD

#### Scope of this ADR

This ADR covers the student-academic domain as the first implementation target.
Budget and financial-aid domains follow the same pattern with their own mapping
tables and are out of scope here. Column-level masking (Layer 3 / G-01) is a
separate workstream and is not governed by this decision.

---

### Implementation Checklist

This ADR also serves as a work guide. Tasks are sequenced by dependency.

**Phase 1 — Prerequisites (complete)**
- [x] Surface `advisor_id` in `int_students` as canonical coalesced column
      (JCX primary; J1 stub ready for when `stg_j1__students` is populated)
- [x] Promote `advisor_id` to `dim_student` via `enrollment_attrs` CTE
- [x] Promote `advisor_id` to `fact_student_term` via `advisor_attrs` CTE + left join

**Phase 2 — Role taxonomy and grants (complete)**
- [x] Define and create Snowflake roles: `{CODE}_REGISTRAR_ROLE`, `{CODE}_ADVISOR_ROLE`,
      `{CODE}_FA_ROLE`, `{CODE}_IR_ANALYST_ROLE` — added to `generate_school.py`
      (`generate_rbac()` function) for all future schools; migration script at
      `school_setup/migrations/add_business_roles.sql` applies them to existing schools
- [x] Business-function roles inherit distribute access via `{CODE}_REPORTING_PROD`
      role hierarchy — FUTURE GRANTS flow through automatically; no per-table grants
      needed. This supersedes the `+grants:` dbt config approach: dbt grants would be
      redundant and Jinja role names are not supported in `dbt_project.yml` grants blocks.
- [x] Added roles to `row_access_role_map` INSERT in `generate_governance()` so they
      are permitted by the existing binary RAP until Phase 3 replaces it
- [x] Add `seeds/shared/seed_rbac_role_definitions.csv` — platform-wide role catalogue
      with default access tiers per data domain; loaded into
      `DITTEAU_SHARED.governance.role_definitions` as documentation only

**Phase 3 — Per-school governance schema and mapping tables**
- [ ] Create `{SCHOOL}_DD_{ENV}.governance` schema for each active school
      (DDL script: `scripts/governance/create_governance_schema.sql`,
      parameterised by `school_code` and `env`)
- [ ] Create and seed `role_domain_access` per school from `seed_rbac_role_definitions`
      defaults; document any school-specific tier overrides in the seed file
- [ ] Create `advisor_student_map` per school; write refresh procedure sourced
      from `stg_jcx__students.advisor_id` (triggered on each PROD dbt build)
- [ ] Validate mapping table coverage against live advisor roster before enabling
      enforcement

**Phase 4 — RAP functions and macro**
- [ ] Write Snowflake RAP function `rap_student_academic` in
      `scripts/governance/create_rap_functions.sql` (parameterised; deployed
      per school into their governance schema)
- [ ] Write `macros/governance/apply_rap.sql` — post-hook wrapper resolving
      the RAP path via `var('school_code')` and `var('env')`, guarded by
      `var('enable_row_level_security')`
- [ ] Apply macro to `dim_student`, `fact_student_term`, `fact_enrollment`
      via `+post-hook:` in their model YAML or `dbt_project.yml` config block
- [ ] Test: run as `ADVISOR_ROLE` user — confirm only advisee rows returned
- [ ] Test: run as `REGISTRAR_ROLE` user — confirm full row access
- [ ] Test: run as `IR_ANALYST_ROLE` user — confirm zero row access
      (aggregated-only tier)

**Phase 5 — Documentation and sign-off**
- [ ] Update `data_governance_policy.md` to mark G-03 in progress / complete
- [ ] Add runbook: `runbooks/row-access-policies.md`
      (how to add a user to advisor_student_map, how to test RAP as a role,
      how to deploy to a new school)
- [ ] KKM sign-off before enabling in any PROD environment

---

### Consequences

#### Pros

- **Configuration-driven:** adding a new advisor or cost-center manager requires
  only a row insert into a mapping table — no DDL or dbt changes
- **Domain isolation:** a bug or change in `rap_budget` cannot affect
  `rap_student_academic` — domains are independently deployable and testable
- **Institutional isolation:** each school's RAP functions and mapping tables are
  contained within its own database — a misconfiguration at one institution
  cannot affect another, and schools can set different access tiers for the
  same role without coordination
- **Cheap evaluation:** RAP functions join to small mapping tables on indexed keys;
  Snowflake evaluates the policy once per session, not per row scan
- **Clear separation of concerns:** row visibility (RAP) and column sensitivity
  (masking) are distinct objects with distinct owners and change cadences
- **Feature-flag safe:** `enable_row_level_security: false` in dev means no
  policy overhead during development; PROD enforcement is opt-in per environment
- **Extensible to new domains:** budget and financial-aid access follow the
  identical pattern — add a mapping table, write one RAP function per school

#### Cons

- **Mapping tables require operational ownership:** `advisor_student_map` must
  be refreshed whenever advisor assignments change in the SIS; a stale map means
  advisors silently lose access to reassigned students
- **Multi-source advisor gap:** J1 and Workday do not currently populate
  `advisor_id`; SCOPED tier enforcement is JCX-only until those sources
  are mapped
- **Per-school deployment overhead:** RAP functions and governance schema DDL
  must be deployed and maintained for every active school; adding a new school
  requires a governance schema setup step before dbt can apply policies
- **CURRENT_ROLE() vs CURRENT_USER():** Snowflake RAPs can filter on role or
  user. Role-based filtering is simpler but requires all sessions to set the
  correct role; user-based filtering is more precise but requires username
  maintenance in mapping tables. This ADR uses role-based tier lookup plus
  user-based predicate lookup — both mapping tables must be maintained
- **Out-of-band DDL:** RAP function creation and `ALTER TABLE ... ADD ROW ACCESS
  POLICY` statements are Snowflake DDL executed outside dbt's normal run; they
  must be re-applied after `--full-refresh` runs that drop and recreate tables
- **Testing complexity:** validating row access requires Snowflake sessions
  with explicit `USE ROLE` — cannot be tested via dbt tests alone; requires
  a separate test harness or manual verification

---

### Alternatives Considered

| Option | Pros | Cons |
|---|---|---|
| Status quo — no RAPs | No implementation cost | FERPA non-compliance; any role with SELECT sees all rows |
| Centralized RAPs in `DITTEAU_SHARED.governance` | Single deployment target; one set of functions to maintain | Blast radius spans all schools; cross-school permission changes couple institutions; schools cannot diverge on tier policy |
| Single mega-RAP across all domains | One function to write and maintain | Expensive to evaluate; fragile — any domain change touches the shared function; harder to test in isolation |
| View-per-role pattern | No Snowflake policy objects needed; standard SQL | Object explosion (N tables × M roles = many views); hard to maintain; consumers must know which view to use |
| Per-school governance schema with domain-scoped RAPs **(chosen)** | Blast radius contained; institutional policy independence; cheap evaluation; configuration-driven | Per-school deployment overhead; out-of-band DDL; mapping table operational burden |

---

### References

- Governance gap register: [`docs/governance/data_governance_policy.md`](/Users/laurievanpelt/ditteau_data_transform/docs/governance/data_governance_policy.md)
- RAP prerequisite work: [`models/deterge/intermediate/int_students.sql`](/Users/laurievanpelt/ditteau_data_transform/models/deterge/intermediate/int_students.sql)
- [`models/distribute/dimensions/dim_student.sql`](/Users/laurievanpelt/ditteau_data_transform/models/distribute/dimensions/dim_student.sql)
- [`models/distribute/facts/fact_student_term.sql`](/Users/laurievanpelt/ditteau_data_transform/models/distribute/facts/fact_student_term.sql)
- Snowflake RAP documentation: [docs.snowflake.com/en/user-guide/security-row-intro](https://docs.snowflake.com/en/user-guide/security-row-intro)
- Related: ADR-001 (external sources shared database — `DITTEAU_SHARED` scope boundary)
- Governance item: G-03 | Access Control | MEDIUM | P2 | Phase 2 target
