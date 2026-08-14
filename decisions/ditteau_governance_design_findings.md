# Ditteau Governance Design Findings

**Purpose:** This document records design decisions and reasoning produced during ADR-003 Phase 6 governance implementation that exist in no repository. Without it, the deliberate inconsistencies documented here will be "fixed" by someone who reads them as errors.

**Date:** 2026-08-13
**Author:** LVP (with Phase 6 implementation context)
**Status:** Active — these are current design constraints, not historical notes

---

## 1. CURRENT_ROLE() Cannot Select Tiers in Streamlit in Snowflake

### The Problem

Inside a Streamlit in Snowflake (SiS) application, `CURRENT_ROLE()` returns the **app owner role**, not the viewer's role. This is documented Snowflake behavior for owner's-rights execution:

> When a Streamlit app runs, it uses the privileges of the role that owns the app, not the role of the user viewing the app.
>
> — Snowflake Documentation: [Streamlit in Snowflake Security Model](https://docs.snowflake.com/en/developer-guide/streamlit/owners-rights)

### The Implication

A row access policy that resolves tiers via `CURRENT_ROLE()` would apply the **same tier to every viewer** — the app owner's tier. If the app owner role (`{CODE}_STREAMLIT_OWNER_{ENV}`) had a tier of `FULL`, every SiS viewer would see all rows and unmasked PII, regardless of their actual authorization level.

### The Solution

The governance framework resolves tiers **user-first**:

```sql
COALESCE(
    (SELECT access_tier FROM user_domain_access WHERE snowflake_username = CURRENT_USER() ...),
    (SELECT access_tier FROM role_domain_access WHERE role_name = CURRENT_ROLE() ...)
)
```

- `user_domain_access` takes precedence over `role_domain_access`
- `CURRENT_USER()` returns the actual viewer's username, even under owner's-rights execution
- SiS viewers must be explicitly provisioned in `user_domain_access` to receive row access

This is why `user_domain_access` exists. It was not added for administrative convenience — it is architecturally required for SiS to function safely.

---

## 2. Streamlit Owner Roles Are Deliberately Absent from role_domain_access

### The Observed State

The `role_domain_access` tier grid contains 40 rows: 10 roles × 4 domains. The grid is fully populated except that `{CODE}_STREAMLIT_OWNER_{ENV}` roles do **not** appear.

This is not an oversight. It is deliberate.

### The Reasoning

An explicit `NONE` row for the Streamlit owner role would deny access identically to absence today. But it creates a different failure mode tomorrow:

1. A well-meaning administrator sees an explicit `NONE` and changes it to `FULL`
2. Because `CURRENT_ROLE()` returns the app owner for all SiS viewers, that single change grants **every SiS viewer** full row access AND unmasked PII
3. The change looks routine (updating a tier) rather than catastrophic (compromising all viewers)

Absence is the safer encoding:
- If the Streamlit owner role is absent from `role_domain_access`, the role-keyed lookup returns NULL
- `COALESCE(NULL, NULL)` returns NULL, and the policy's trailing `ELSE FALSE` denies
- There is no row to edit, so the failure mode requires adding a row — a more visible action

### The Invariant Test

`tests/governance/assert_no_app_owner_in_role_domain_access.sql` enforces this invariant:

```sql
SELECT role_name, data_domain, access_tier
FROM {{ target.database }}.governance.role_domain_access
WHERE role_name ILIKE '%STREAMLIT_OWNER%'
```

This test returns 0 rows on success. Any Streamlit owner row causes test failure.

The `tests/governance/` directory is CODEOWNERS-protected. Deleting the test is the quiet way to undo the control — the CODEOWNERS protection ensures that attempt is visible.

---

## 3. Fallback Governance View — Considered and Ruled Out

### The Alternative Considered

During Phase 6 design, a fallback governance view was specified as a remedy if the `COALESCE` of two subqueries proved costly:

```sql
CREATE VIEW governance.v_effective_tier AS
SELECT CURRENT_USER() AS username, CURRENT_ROLE() AS role_name, data_domain,
       COALESCE(u.access_tier, r.access_tier) AS effective_tier
FROM ...
```

The concern was that two scalar subqueries inside a row access policy might execute per row, causing O(n) lookups on large tables.

### The Finding

The concern was unfounded. Both tier subqueries are **uncorrelated**:
- They reference only `CURRENT_USER()`, `CURRENT_ROLE()`, and a literal domain string
- They do not reference any column from the protected table
- Snowflake evaluates uncorrelated subqueries **once per query**, not per row

Verified empirically on DEMEAU_DD_DEV, 2026-08-12: EXPLAIN ANALYZE showed single-evaluation behavior.

### The Caveat

The `SCOPED` branch **is** row-correlated:

```sql
WHEN 'SCOPED' THEN EXISTS (
    SELECT 1 FROM advisor_student_map
    WHERE snowflake_username = CURRENT_USER()
      AND student_id = row_student_id  -- <-- row reference
      AND (expiration_date IS NULL OR expiration_date >= CURRENT_DATE())
)
```

This branch executes a lookup per row when the viewer's tier is `SCOPED`. Performance on large tables with SCOPED advisors has not been timed. The mitigation path, if needed, is an index on `advisor_student_map(snowflake_username, student_id)`.

The fallback view is recorded as **considered and ruled out**, with the SCOPED caveat documented for future reference.

---

## 4. Policy Signatures Diverge by Domain — This Is Correct

### The Observed State

Row access policy signatures differ by domain:

| Policy | Signature Type | Target Columns |
|--------|----------------|----------------|
| `rap_student_academic` | `NUMBER(38,0)` | `student_id` (integer) |
| `rap_financial_aid` | `VARCHAR` | `student_key` (MD5 text) |
| `rap_admissions` | `VARCHAR` | `applicant_key` (MD5 text) |

### The Reasoning

**Applicants are not students.** An applicant who applies but never enrolls has no `student_id` at all. The admissions domain tracks `applicant_key`, which is an MD5 surrogate derived from application attributes.

Similarly, `fact_aid_award` carries only `student_key` (MD5 surrogate), not a natural `student_id`. This is a modeling choice: financial aid awards link to the student dimension via surrogate key.

A shared `NUMBER` signature would impose false uniformity on genuinely different entity grains. Snowflake error 003554 ("Policy signature does not match column type") would block attachment anyway — the divergence is required, not merely stylistic.

### The Unreferenced Parameter

In `rap_financial_aid` and `rap_admissions`, the signature parameter (`row_student_key`, `row_applicant_key`) is **declared but never read** in the policy body. With no `SCOPED` branch, nothing uses the row value — only type-compatibility for attachment matters.

The parameter exists solely to satisfy Snowflake's policy attachment requirement. It is not dead code in the conventional sense; it is a type declaration with no runtime behavior.

---

## 5. Snowflake Matches Signatures on Type Family, Not Precision or Scale

### The Finding

A policy with signature `NUMBER(38,0)` attaches successfully to columns of type:
- `NUMBER(38,0)` (exact match)
- `NUMBER(38,5)` (different scale)
- `NUMBER(10,0)` (different precision)
- `INTEGER` (alias for NUMBER)

But it **fails** to attach to `VARCHAR` or `TEXT` columns (error 003554).

### Verification

Verified empirically on DEMEAU_DD_DEV, 2026-08-12:
- `rap_student_academic` (NUMBER signature) attached to `dim_student.student_id` at `NUMBER(38,5)` — success
- Same policy attached to `fact_student_term.student_id` at `NUMBER(38,0)` — success
- Attempted attachment to `fact_aid_award.student_key` (TEXT) — error 003554

### The Implication

One `NUMBER(38,0)` signature serves all numeric `student_id` columns regardless of their declared precision. Policy authors do not need to match precision exactly — family match is sufficient.

This is why `rap_student_academic` uses a single signature for all student-academic tables, even though `student_id` precision varies between dimensions and facts.

---

## 6. AGGREGATED Tier Denies at Row Grain by Design

### The Observed State

The `AGGREGATED` tier maps to `FALSE` in all row access policies:

```sql
WHEN 'AGGREGATED' THEN FALSE
```

This means an `IR_ANALYST_ROLE` with `AGGREGATED` access to `student_academic` sees **zero rows** from `fact_student_term`.

### The Reasoning

A row access policy filters **before** aggregation. The policy cannot know whether the query will aggregate; it sees individual rows.

If `AGGREGATED` returned `TRUE`, the user would see individual student records — which defeats the purpose of the tier. There is no way to make a row access policy say "deny if selecting individual rows, allow if aggregating."

### The Correct Pattern

`AGGREGATED` users consume data through **pre-aggregated marts** that do not carry a row access policy:

- `mart_enrollment_census` — headcount aggregated by term, program, level
- `mart_retention_cohort_summary` — retention rates aggregated by cohort
- `mart_admissions_funnel` — funnel counts aggregated by term and stage

These marts are not student-grained, so they carry no RAP. Access flows through the role hierarchy (`IR_ANALYST_ROLE` inherits from `REPORTING_PROD`), and the marts expose only counts, not individuals.

The `AGGREGATED` tier on the row-grain tables is not "allow counting" — it is "deny row access, consume via marts."

---

## 7. The Exists-Versus-Observed Pattern

### The Problem

Four ADR-003 Phase 4 acceptance criteria were recorded as complete because code had been written:

1. `rap_student_academic` function created — ✓ (exists in governance schema)
2. `apply_rap()` macro defined — ✓ (exists in macros/governance/)
3. Post-hook configured in `dbt_project.yml` — ✓ (configuration present)
4. `role_domain_access` populated — ✓ (40 rows exist)

All four criteria had evidence: a file existed, a schema object existed, a configuration was present.

### The Defect

Three of the four could only fail at **execution**, and nothing executed them:

- The post-hook contained `var('env')`, which does not exist. The hook would raise "Required var 'env' not found" if executed.
- `enable_row_level_security` was `false` in all environments, so the hook never ran.
- Policy attachment was never tested because no execution path triggered it.

The code sat unexecuted from Phase 4 (2026-08-10) until Phase 6 (2026-08-13). The `var('env')` defect was discovered only when Phase 6 work attempted to verify attachment.

### The Lesson

**Existence is not observation.** Writing code is not deploying a control. A schema object that exists but has never executed proves nothing about whether it works.

The deployed-state document now requires:
- `[DEPLOYED]` — observed to operate (query result or test run)
- `[BUILT, NOT ACTIVE]` — exists but never executed

The distinction matters because three Phase 4 criteria looked complete. They were not.

---

## 8. The `var('env')` Defect and Target Database Resolution

### The Defect

The original `apply_rap()` macro attempted to reconstruct the database name:

```jinja
{% set rap_fqn = var('school_code') ~ '_DD_' ~ var('env') ~ '.governance.rap_' ~ domain %}
```

This failed because `var('env')` is not defined in `dbt_project.yml` and the `scripts/run_<school>_dev.sh` wrappers do not supply it.

### The Fix

Use `target.database` directly:

```jinja
{% set rap_fqn = target.database ~ '.governance.rap_' ~ domain %}
```

The `target` context object is always available in dbt. `target.database` contains the fully-qualified database name from `profiles.yml` (e.g., `DEMEAU_DD_DEV`).

### The Convention

**Prefer `target.database` over reconstructing database names from vars.**

Environment isolation is handled by database naming (`{SCHOOL}_DD_{ENV}`) via the profile target, not by a separate `env` variable. The `school_code` var exists for other purposes (seed selection, school-specific logic) — not for database name construction.

This convention is now documented in the code comments and should be added to `ditteau_dbt_conventions_lvp_march26.md`.

---

## Summary of Deliberate Inconsistencies

| Item | Appears To Be | Actually Is |
|------|---------------|-------------|
| Streamlit owner roles absent from tier grid | Missing data | Deliberate security encoding |
| Policy signatures differ by domain | Inconsistency | Required by entity grain differences |
| Signature parameter unreferenced in body | Dead code | Type declaration for attachment |
| `AGGREGATED` returns FALSE | Broken tier | Correct design (use marts) |
| `var('env')` not defined | Missing var | Intentionally absent (use target.database) |

Do not "fix" these without understanding the reasoning documented above.
