# ADR-003: Domain-Scoped Row Access Policy Architecture

**Status:** Accepted — enforcement live in DEMEAU PROD since 2026-08-18; not enabled in any tenant PROD

**Date:** 2026-08-10 (last updated 2026-08-26)

**Author:** LVP

---

### Amendment 2026-08-26 — deployment state corrected

**The status line and the deployment block below were wrong, and the specific error was
that enforcement had never run.** It had. This is recorded here rather than edited away
because the claim survived three revisions of the access policy on the strength of a
handoff note marked "confirmed", and the correction is more useful than a clean document.

Measured 2026-08-26 from `SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES` against
`DEMEAU_DD_PROD`:

- **9 row access policy attachments across 3 policies.** `rap_student_academic` on
  `dim_student`, `fact_student_term`, `fact_enrollment`, `mart_student_at_risk`,
  `mart_academic_progress` and `mart_registration_holds`; `rap_admissions` on
  `dim_applicant` and `fact_application`; `rap_financial_aid` on `fact_aid_award`.
- **7 masking attachments across 4 policies.** `mask_name` on three columns and
  `mask_dob` on one, both on `dim_student`; `mask_financial_amount` on two columns of
  `fact_aid_award`; `mask_email` on `mart_academic_progress`.

`scripts/run_demeau.sh` forces `enable_row_level_security` and
`enable_masking_policies` true whenever `ENV=prod`, so every DEMEAU PROD build has
enabled both. Enforcement was measured working on 2026-08-18: four persona sessions,
four different answers, zero errors, with the spread only executing policies produce.

**What is still true:** no *tenant* database enforces anything. Enforcement has only
ever run in DEMEAU. ⚠️ But note DEMEAU is **not** synthetic — it is pseudonymised
Anselm-derived data sharing all 14,150 student ids with `ANSELM_DD_DEV` and carrying
real dates of birth for 94 per cent of them. Use is authorised by Saint Anselm. Treat
`DEMEAU_DD_*` as real institutional data.

**Two things the original block got right and are worth re-reading:** school staff still
hold no Snowflake accounts, so `advisor_username_crosswalk` remains unpopulated and
`SCOPED` remains untestable against real principals; and the warning about a BI tool on
a shared service account collapsing all advisors into one identity still stands.

**Other state changes since 2026-08-12:**

- The grid is no longer 360 rows. A privilege-escalation defect was found on 2026-08-24 —
  every business persona role inherited `{CODE}_REPORTING_PROD`, which itself held `FULL`
  on all domains, so any persona reached `FULL` with one `USE ROLE`. Five non-persona
  roles were removed from the grid across all nine databases, taking each to 20 rows.
  `DEMEAU_DD_DEV` is now 36 rows across 9 roles after the persona work of 2026-08-26.
- **The follow-up this ADR owed has landed.** It asked for "a test asserting cross-domain
  tier coherence"; `assert_cross_domain_tier_coherence` exists and passes. It has already
  earned its place twice — it refused the Admissions and Finance personas on 2026-08-26
  for the same shape that produced the `FA_ROLE` defect recorded below.
- Ten governance assertions now run on every build, nine passing and one warning by
  design.
- Two later decisions extend this architecture: **ADR-005** supplies the precondition it
  assumed — that every model declares a domain — and **ADR-006** resolves the
  one-policy-per-table constraint recorded below for models whose content spans domains.

---

> *Historical, retained for the record. Superseded by the amendment above — the
> enforcement claim in the paragraph below is false.*

**Deployment state as of 2026-08-12:** All governance objects exist in all nine
`{SCHOOL}_DD_{ENV}` databases for MERRIMACK, ANSELM, and DEMEAU — 12 business
roles, 360 `role_domain_access` tier rows, the two mapping tables, three refresh
procedures, and nine `rap_student_academic` policies. Tier logic is validated
end to end. Enforcement remains **off everywhere**: `enable_row_level_security`
is `false` in DEV/TEST/PROD, so no policy is attached to any table, and
`advisor_username_crosswalk` is unpopulated. Two gates remain before PROD
enablement — populate the crosswalk from each school's advisor roster, and
obtain sign-off from KKM (Kelly, Director of Data Governance).

**A prerequisite behind the roster gate:** as of 2026-08-13 no school staff hold
Snowflake accounts. All 12 business roles report `assigned_to_users = 0` and the
account contains four human users, all Serensoft. Because
`advisor_username_crosswalk` maps `advisor_id` to a Snowflake `login_name`, the
crosswalk cannot be populated until staff are provisioned — this is a
provisioning dependency, not a data-entry task. It also bounds the design: the
`SCOPED` predicate reads `CURRENT_USER()`, so it only separates advisors who each
hold an individual login. Delivering school access through a BI tool on a shared
service account would collapse all advisors into one identity and break the tier.
Confirm the intended access model before enabling enforcement.

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
- [~] Add `seeds/shared/seed_rbac_role_definitions.csv` — platform-wide role catalogue
      with default access tiers per data domain; ~~loaded into
      `DITTEAU_SHARED.governance.role_definitions` as documentation only~~
      — **partially complete.** The CSV exists in the repo and was used as the source
      for the per-school `role_domain_access` seeding, but
      `DITTEAU_SHARED.GOVERNANCE.ROLE_DEFINITIONS` **does not exist** in Snowflake
      (verified 2026-08-13). Nothing was ever loaded there. Either load it or drop the
      claim; a reference table cited in the Decision section but absent from the account
      is worse than no reference table.

**Phase 3 — Per-school governance schema and mapping tables**
- [x] Create `{SCHOOL}_DD_{ENV}.governance` schema for each active school
      — already created by the original school-setup run (`05_governance.sql`);
      verified present in all 9 databases, no new DDL needed
- [x] Create and seed `role_domain_access` per school from `seed_rbac_role_definitions`
      defaults; document any school-specific tier overrides in the seed file
      — 40 rows per school per env (360 total); no school overrides yet
- [~] Create `advisor_student_map` per school; ~~write refresh procedure sourced
      from `stg_jcx__students.advisor_id` (triggered on each PROD dbt build)~~
      — `refresh_advisor_student_map()` created in each PROD, joining through
      `advisor_username_crosswalk` (see Decision: advisor identity resolution).
      **The tables are deployed; the procedure does not run.** First execution on
      2026-08-13 failed:
      `Object 'DEMEAU_DD_PROD.DETERGE.STG_JCX__STUDENTS' does not exist`.
      Root cause is not the procedure — **no PROD database has ever been built.**
      All three hold only `deposit` and `governance`; there is no `deterge` or
      `distribute` layer in any PROD. The procedure is PROD-only by design and
      targets a layer that does not exist there, so it could never have succeeded.
      It was marked complete on the basis of being created, never called.
- [ ] Validate mapping table coverage against live advisor roster before enabling
      enforcement — **outstanding; `advisor_username_crosswalk` is empty in all
      environments, so the SCOPED tier currently grants zero rows**

Deviation from the proposed layout: the DDL lives in
`ditteau_data_infra/school_setup/migrations/add_governance_phase3_tables.sql`
(one-time, for the three existing schools) plus `generate_governance()`
Section 3 in `generate_school.py` (for schools onboarded later), rather than a
single parameterised `scripts/governance/` script. This matches the dual-track
pattern already established by `add_business_roles.sql`.

**Phase 4 — RAP functions and macro**
- [x] Write Snowflake RAP function `rap_student_academic` in
      `scripts/governance/create_rap_functions.sql` (parameterised; deployed
      per school into their governance schema)
      — actual path: `migrations/add_governance_phase4_rap.sql` + generator
      Section 4; deployed to all 9 governance schemas
- [x] Write `macros/governance/apply_rap.sql` — post-hook wrapper resolving
      the RAP path via `var('school_code')` and `var('env')`, guarded by
      `var('enable_row_level_security')`
- [x] Apply macro to `dim_student`, `fact_student_term`, `fact_enrollment`
      via `+post-hook:` in their model YAML or `dbt_project.yml` config block
      — had to go in `dbt_project.yml`: Jinja in a model-YAML `config:` block is
      evaluated at parse time, before project macros load, so `{{ apply_rap() }}`
      there fails with `'apply_rap' is undefined`

      ~~Marked complete 2026-08-12 on the basis that the macro and hooks were
      written and wired.~~ **They were, and the macro did not work.** It resolved
      the policy path as `var('school_code') ~ '_DD_' ~ var('env')`, and there is
      no `env` var in this project — not in `dbt_project.yml`, not supplied by any
      `run_<school>_dev.sh`. Enforcement would have failed on the first PROD model
      with `Required var 'env' not found`. The defect was undetectable because the
      macro body only executes when `enable_row_level_security` is true, which it
      had never been in any environment. Fixed in Phase 6 to use `target.database`,
      which resolves from `profiles.yml` rather than being reconstructed from parts
      that may not exist. Attachment observed for the first time on 2026-08-13
      against `DEMEAU_DD_DEV` — see the Phase 6 validation record.
- [x] Test: run as `ADVISOR_ROLE` user — confirm only advisee rows returned
- [x] Test: run as `REGISTRAR_ROLE` user — confirm full row access
- [x] Test: run as `IR_ANALYST_ROLE` user — confirm zero row access
      (aggregated-only tier)

All three tests passed on 2026-08-12 against DEMEAU DEV, plus three additional
cases: SYSADMIN bypass (100/100), `FA_ROLE` SCOPED (5/100), and an unmapped role
falling through to default-deny (0/100). Method and constraints are recorded in
`runbooks/row-access-policies.md`.

Two corrections came out of Phase 4 and are worth recording:

- **Policy signature type.** The policy was first written as
  `(row_student_id VARCHAR)`, which cannot attach to any target column —
  `dim_student.student_id` is `NUMBER(38,5)`, `fact_student_term.student_id` and
  `fact_enrollment.student_id` are `NUMBER(_,0)`. Snowflake matches policy
  signatures on type *family*, not precision/scale (verified empirically), so a
  single `NUMBER(38,0)` signature attaches to all of them and no model needed
  re-typing. `advisor_student_map.student_id` and
  `advisor_username_crosswalk.advisor_id` were retyped to `NUMBER(38,0)` to match
  (`add_governance_phase4b_retype_student_id.sql`).
- **`fact_enrollment` had no natural student key.** Only the surrogate
  `enrollment_student_key` existed; attaching the policy to it would have
  compiled and then silently matched nothing, returning zero rows for every
  SCOPED user. `a.student_id` was already available from the
  `int_course_registrations` join and is now selected.

**Phase 5 — Documentation and sign-off**
- [x] Update `data_governance_policy.md` to mark G-03 in progress / complete
- [x] Add runbook: `runbooks/row-access-policies.md`
      (how to add a user to advisor_student_map, how to test RAP as a role,
      how to deploy to a new school)
- [ ] Sign-off from KKM (Kelly, Director of Data Governance) before enabling in any
      PROD environment
- [ ] Provision Snowflake accounts for school staff and assign the business roles —
      blocks the advisor roster load, and therefore the SCOPED tier
- [ ] Confirm the access model is per-user rather than a shared service account,
      since the SCOPED predicate depends on `CURRENT_USER()`

---

### Validation Methodology — added 2026-08-13

Four criteria in this checklist were marked complete on the basis that code had been
**written and wired**, not that it had been **observed to work**. Three of the four
could only fail at execution, and nothing executed them.

| Item | Marked on | Actual state when tested |
|---|---|---|
| `apply_rap()` post-hooks (Phase 4) | Macro written, hooks wired | Used a non-existent `var('env')`; would fail on first PROD model |
| `refresh_advisor_student_map()` (Phase 3) | Procedure created | Fails on first call — no PROD `deterge` layer exists |
| `DITTEAU_SHARED.governance.role_definitions` (Phase 2) | Seed CSV committed | Table never created in Snowflake |
| `enable_masking_policies` (referenced in model docs, not this checklist) | Flag defined, docs written | Flag is inert — no code reads it; zero masking policies deployed |

The common cause is that each sat behind something that prevented execution: a feature
flag that has never been true, a PROD environment that has never been built, or a
manual step nobody ran. **A feature flag guarantees its own code is untested.**

Consequences adopted going forward:

- A criterion is not complete until its **effect** is observed. For a policy, that
  means querying `information_schema.policy_references` and seeing the attachment —
  not that the hook ran without error. For a procedure, calling it. For a table,
  selecting from it in the target account.
- Anything gated on `enable_row_level_security` or `enable_masking_policies` must be
  exercised on `DEMEAU_DD_DEV` with the flag temporarily true, then flipped back.
  DEMEAU's CX share is de-identified and authorised for demonstration use, which makes
  it the correct place to do this.
- Deployment assertions belong in `tests/governance/` as queries against deployed
  state. Note that a dbt test alone would not have caught three of the four above;
  they need assertions about Snowflake objects, not about data quality.
- **No PROD database has ever been built.** All three hold only `deposit` and
  `governance`. `MERRIMACK_DD_TEST` and `ANSELM_DD_TEST` are built; `DEMEAU_DD_TEST` is
  not. Any statement in this ADR about PROD behaviour is therefore projection, not
  observation, and should be read that way until a PROD build exists.

---

### Constraint — conformed dimensions can belong to only one domain

Snowflake permits **one row access policy per table**. Conformed dimensions are shared
across domains, so this forces a single domain owner per table — and that ownership
determines the tier every role gets on that table, regardless of the role's tier in any
other domain.

`dim_student` is owned by `student_academic`. It therefore cannot also carry
`rap_financial_aid`, so financial-aid access to student demographics is **not
expressible at dimension grain**. A role's `financial_aid` tier has no effect on what it
reads from `dim_student`; only its `student_academic` tier does.

This is not a limitation to work around — a second mechanism to express it would
reintroduce exactly the parallel-lookup problem this ADR removes. It is a constraint to
design the tier grid against.

**The grid must therefore be read across rows, not down columns.** It was authored
domain by domain, which is how the defect below survived review.

#### Defect found 2026-08-13: FA_ROLE saw zero students

`FA_ROLE` held `FULL` on `financial_aid` and `SCOPED` on `student_academic`. `SCOPED`
resolves through `advisor_student_map`, populated from advisor assignments — and
financial-aid counselors are not advisors. Measured on DEMEAU DEV with policies live:

| Query as `FA_ROLE` | Rows |
|---|---|
| `fact_aid_award` | 204 |
| `dim_student` | **0** |
| the two joined on `student_key` | **0** |

Every aid analysis touching demographics returns nothing, **silently** — a row access
policy yields empty results, not errors. `mart_aid_leveraging`, aid-by-ethnicity and
aid-by-first-generation would all have been unbuildable for the role that exists to use
them.

`SCOPED` was never a deliberate policy choice here. It is the same value
`ADVISOR_ROLE` carries, reached through a mechanism built for advising. The grid
expressed a mechanism, not an intent.

**Correction (KKM-approved 2026-08-13): `FA_ROLE` → `FULL` on `student_academic`.** No
caseload relationship exists to scope to: a student's aid record originates from their
own FAFSA filing, not from an advisor assignment, so the set of students an FA counselor
legitimately touches is closer to "all of them" than to any subset the data can express.

What was approved is a **widening of PII access** — under `FULL`, `FA_ROLE` reads all of
`dim_student`. Row access and column masking are separable: `FULL` row access plus masked
SSN/DOB may cover the role's work, and that is the smaller grant. Resolve it as part of
work item F rather than assuming `FULL` implies unmasked.

The other three business roles were checked across the grid at the same time and are
sound: `REGISTRAR_ROLE` is `FULL` in both domains it touches, `ADVISOR_ROLE` has access
in one domain only, and `IR_ANALYST_ROLE` is `AGGREGATED` in both — uniformly zero at row
grain, which is a coherent marts-only posture rather than a mismatch.

> **Note for IR_ANALYST_ROLE:** `mart_student_at_risk`, `mart_academic_progress` and
> `mart_registration_holds` now carry `rap_student_academic`, so `AGGREGATED` roles read
> **zero rows** from them. That is correct — they are student-grain, not pre-aggregated —
> but it is a behaviour change. Anything IR needs from those marts must come from a
> genuinely aggregated model.

**Follow-up owed:** a test asserting cross-domain tier coherence, so a role cannot hold
non-zero access in one domain while resolving to zero rows in a conformed dimension it
must join. Reading each role's row is currently a manual review step.

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
