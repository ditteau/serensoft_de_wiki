# Row Access Policies Runbook

Operational guide for the domain-scoped Row Access Policy (RAP) architecture defined in [ADR-003](../decisions/adr-003-row-access-policy-architecture.md). RAPs are Layer 2 of the platform's three-layer access control model: Snowflake object grants restrict *which tables* a role can see, RAPs restrict *which rows within them*, and dynamic data masking (G-01) restricts *which column values*.

**Last updated:** August 2026
**Maintained by:** LVP

---

## Contents

- [A. Architecture Overview](#a-architecture-overview)
- [B. Current Deployment State](#b-current-deployment-state)
- [C. Managing Advisor Assignments](#c-managing-advisor-assignments)
- [D. Changing a Role's Access Tier](#d-changing-a-roles-access-tier)
- [E. Testing a RAP as a Role](#e-testing-a-rap-as-a-role)
  - [E.4 Policy evaluation cost](#e4-policy-evaluation-cost)
  - [E.5 Standing DEV grants for DEMEAU](#e5-standing-dev-grants-for-demeau)
- [F. Enabling Enforcement in PROD](#f-enabling-enforcement-in-prod)
  - [F.0 The run scripts silently discard any var you pass](#f0-the-run-scripts-silently-discard-any-var-you-pass)
  - [F.0.1 Verify attachment, never infer it](#f01-verify-attachment-never-infer-it)
- [G. Deploying to a New School](#g-deploying-to-a-new-school)
- [H. Adding a New Domain RAP](#h-adding-a-new-domain-rap)
- [I. Troubleshooting](#i-troubleshooting)

---

## A. Architecture Overview

### A.1 Why per-school governance schemas

Governance objects live in `{SCHOOL}_DD_{ENV}.governance`, **not** in `DITTEAU_SHARED`. This is deliberate: it contains blast radius (a bad tier row affects one school), and it lets schools diverge on policy without forking a shared table.

The cost is that every change is a nine-way deployment (3 schools × 3 environments), which is why all governance DDL is scripted rather than applied by hand.

### A.2 The four access tiers

Each role gets one tier per data domain, stored in `role_domain_access`:

| Tier | Meaning |
|------|---------|
| `FULL` | All rows in the domain |
| `SCOPED` | Only rows matching a user-specific predicate (for `student_academic`, the user's advisees) |
| `AGGREGATED` | No row access at all — the role is expected to read marts, which are pre-aggregated |
| `NONE` | Denied |

`AGGREGATED` and `NONE` behave identically at the RAP level (both return `FALSE`). They are distinguished for documentation and audit: `AGGREGATED` means "this role has a legitimate path to the data via marts", `NONE` means "this role has no business need".

Roles absent from `role_domain_access` fall through the policy's `ELSE` to `FALSE`. **The default is deny** — adding a role to Snowflake does not grant it row access.

### A.3 Objects per school per environment

| Object | Type | Purpose |
|--------|------|---------|
| `role_domain_access` | table | `(role_name, data_domain)` → `access_tier`. 40 rows: 10 roles × 4 domains |
| `advisor_username_crosswalk` | table | SIS `advisor_id` → Snowflake `snowflake_username`. Manually maintained |
| `advisor_student_map` | table | `snowflake_username` → advisee `student_id`. Derived, do not hand-edit |
| `refresh_advisor_student_map()` | procedure | Rebuilds `advisor_student_map`. PROD only |
| `rap_student_academic` | row access policy | Enforces the tiers for the `student_academic` domain |

### A.4 How the policy evaluates

```sql
CURRENT_ROLE() IN ('ACCOUNTADMIN', 'SYSADMIN')     -- bypass
OR CASE COALESCE(
     -- user-keyed tier: the only path that resolves inside Streamlit
     (SELECT access_tier FROM user_domain_access
       WHERE snowflake_username = CURRENT_USER()
         AND data_domain = 'student_academic'
         AND (expiration_date IS NULL OR expiration_date >= CURRENT_DATE())),
     -- role-keyed tier: direct SQL and per-user BI connections
     (SELECT access_tier FROM role_domain_access
       WHERE role_name = CURRENT_ROLE()
         AND data_domain = 'student_academic'))
     WHEN 'FULL'   THEN TRUE
     WHEN 'SCOPED' THEN EXISTS (SELECT 1 FROM advisor_student_map
                                WHERE snowflake_username = CURRENT_USER()
                                  AND student_id = row_student_id
                                  AND (expiration_date IS NULL
                                       OR expiration_date >= CURRENT_DATE()))
     WHEN 'AGGREGATED' THEN FALSE
     WHEN 'NONE'       THEN FALSE
     ELSE FALSE
   END
```

Three things to read out of this:

**Tier resolves user-first, then role.** A `user_domain_access` row always wins over a `role_domain_access` row — that is what `COALESCE` ordering means, and it is deliberate. An expired user row is ignored and resolution falls back to the role tier.

**The user-keyed path exists because of Streamlit.** Under owner's-rights execution, `CURRENT_ROLE()` returns the app owner role for every viewer, so the role-keyed path cannot be trusted there. `{CODE}_STREAMLIT_OWNER_{ENV}` is deliberately absent from `role_domain_access` so that path dead-ends, forcing tier to come from `user_domain_access`. That absence is enforced by `tests/governance/assert_no_app_owner_in_role_domain_access.sql`.

**Tier comes from identity; the predicate comes from the user.** Two advisors sharing `{CODE}_ADVISOR_ROLE` see different rows, because `SCOPED` joins `advisor_student_map` on `CURRENT_USER()`.

Both tier subqueries are **uncorrelated** — they never reference the row parameter — so Snowflake evaluates them once per query rather than per row. The `SCOPED` branch is the exception and *is* row-correlated. See [E.4](#e4-policy-evaluation-cost).

### A.5 Known environment asymmetries

DEV, TEST and PROD are **not** symmetric. Each of these has already caused a wrong assumption during this work, so they are recorded here rather than rediscovered a fourth time.

| Asymmetry | Consequence |
|---|---|
| `{CODE}_ANALYTICS_PROD` exists; there is no `ANALYTICS_DEV` or `ANALYTICS_TEST` | Anything specified as "grant usage on `{CODE}_ANALYTICS_PROD`" only works in PROD. DEV and TEST have `{CODE}_TRANSFORM_{ENV}` as their only warehouse — that is what `STREAMLIT_OWNER_DEV` and `_TEST` were granted. |
| Future grants on `DISTRIBUTE` go to `REPORTING_PROD` in **PROD only**; DEV and TEST have them for `TRANSFORM` only | `AGGREGATED` roles automatically hold `select` on every distribute model in PROD and **nothing** in DEV. This is why testing in DEV needs hand-granted access ([E.5](#e5-standing-dev-grants-for-demeau)) and why no dbt `+grants:` config is needed in PROD. |
| `refresh_advisor_student_map()` exists in **PROD only** | DEV and TEST have `advisor_student_map` as a table but no procedure. Populate those by hand when testing. |
| The business roles hold `{CODE}_ANALYTICS_PROD` by default and no DEV compute | DEV testing requires temporary or standing grants; see [E.5](#e5-standing-dev-grants-for-demeau). |

The pattern behind all four: the school-setup scripts provision PROD as the real environment and DEV/TEST as thinner development copies. Assume nothing carries across without checking.

### A.6 Attachment is owned by dbt

The policy is created by infra scripts but attached by dbt, via the `apply_rap()` post-hook on `dim_student`, `fact_student_term`, and `fact_enrollment`. The hook is a no-op unless `enable_row_level_security` is `true`.

The hook runs `DROP ALL ROW ACCESS POLICIES` before `ADD`, so it is idempotent across `--full-refresh` rebuilds that drop and recreate the table.

> **Never `ALTER TABLE ... ADD ROW ACCESS POLICY` on those three models by hand.** dbt reapplies on every build, so manual attachments either get clobbered or drift silently. Attach to a scratch table instead — see [E](#e-testing-a-rap-as-a-role).

The post-hooks live in `dbt_project.yml`, not in the model YAML. Jinja inside a model-YAML `config:` block is evaluated at parse time, before project macros are loaded, so `post-hook: "{{ apply_rap() }}"` there fails with `'apply_rap' is undefined`.

---

## B. Current Deployment State

As of **2026-08-12**:

| Item | State |
|------|-------|
| Schools covered | MERRIMACK, ANSELM, DEMEAU (all 3 envs each) |
| Business roles | 12 created (`REGISTRAR`, `ADVISOR`, `FA`, `IR_ANALYST` per school) |
| `role_domain_access` | 360 rows (40 × 9 databases) |
| `rap_student_academic` | 9 policies deployed |
| Policies attached to tables | **None** |
| `enable_row_level_security` | `false` in DEV, TEST, and PROD |
| `advisor_username_crosswalk` | **Empty in all 9 databases** |

Enforcement is therefore inactive platform-wide. Two gates remain: load each school's advisor roster ([C](#c-managing-advisor-assignments)) and obtain sign-off from KKM (Kelly, Director of Data Governance).

A third prerequisite sits behind the roster gate and is easy to miss: **as of 2026-08-13 no school staff have Snowflake accounts.** All 12 business roles show `assigned_to_users = 0`, and the account holds four human users, all Serensoft staff. `advisor_username_crosswalk` maps `advisor_id` to a Snowflake `login_name`, so there is nothing to map to yet. Enforcement would work if enabled today, but would not govern anyone.

This also constrains the design: the `SCOPED` predicate keys on `CURRENT_USER()`, so it only distinguishes advisors who each hold their own Snowflake login. If school access is ever delivered through a BI tool using a single shared service account, every advisor collapses into one identity and `SCOPED` silently returns that account's mapped rows to everybody. Confirm the access model before relying on it.

Default tiers as seeded, from `seed_rbac_role_definitions.csv`:

| Role suffix | student_academic | financial_aid | financial | admissions |
|-------------|------------------|---------------|-----------|------------|
| `REGISTRAR_ROLE` | FULL | NONE | NONE | FULL |
| `ADVISOR_ROLE` | SCOPED | NONE | NONE | NONE |
| `FA_ROLE` | SCOPED | FULL | NONE | NONE |
| `IR_ANALYST_ROLE` | AGGREGATED | AGGREGATED | NONE | NONE |

Only `student_academic` has a RAP function. The other three domains' tier rows are inert until one is written ([H](#h-adding-a-new-domain-rap)).

---

## C. Managing Advisor Assignments

`advisor_student_map` is **derived** — never edit it directly. The editable table is `advisor_username_crosswalk`.

### C.1 Onboarding an advisor

Find the advisor's SIS ID and confirm their Snowflake login name:

```sql
-- Advisors present in the SIS with advisees, not yet mapped
SELECT s.advisor_id, COUNT(*) AS advisee_count
FROM MERRIMACK_DD_PROD.deterge.stg_jcx__students s
LEFT JOIN MERRIMACK_DD_PROD.governance.advisor_username_crosswalk x
       ON s.advisor_id = x.advisor_id
WHERE s.advisor_id IS NOT NULL
  AND x.advisor_id IS NULL
GROUP BY s.advisor_id
ORDER BY advisee_count DESC;

-- Confirm the exact login_name; CURRENT_USER() returns login_name, not display name
SHOW USERS LIKE '%SMITH%';
```

Insert the mapping, then refresh:

```sql
USE ROLE SYSADMIN;

INSERT INTO MERRIMACK_DD_PROD.governance.advisor_username_crosswalk
    (advisor_id, snowflake_username, display_name, is_active)
VALUES (48812, 'JSMITH', 'Jordan Smith', TRUE);

CALL MERRIMACK_DD_PROD.governance.refresh_advisor_student_map();
-- returns e.g. 'refreshed 1284 advisor-student assignments'
```

Grant them the advisor role if they don't have it:

```sql
USE ROLE SECURITYADMIN;
GRANT ROLE MERRIMACK_ADVISOR_ROLE TO USER JSMITH;
```

### C.2 Offboarding an advisor

Set `is_active = FALSE` rather than deleting — the row is audit evidence of who had access:

```sql
UPDATE MERRIMACK_DD_PROD.governance.advisor_username_crosswalk
SET is_active = FALSE
WHERE advisor_id = 48812;

CALL MERRIMACK_DD_PROD.governance.refresh_advisor_student_map();
```

The refresh filters on `is_active = TRUE`, so their `advisor_student_map` rows disappear on the next call. Revoke the role separately:

```sql
USE ROLE SECURITYADMIN;
REVOKE ROLE MERRIMACK_ADVISOR_ROLE FROM USER JSMITH;
```

### C.3 How the refresh works

`refresh_advisor_student_map()` is `TRUNCATE` + `INSERT`, so it fully rebuilds from `stg_jcx__students` joined to the crosswalk. Consequences:

- Stale assignments are removed automatically when a student changes advisors.
- It reads the **deterge** layer, so it must run *after* staging models are built.
- It exists in PROD only. DEV and TEST have the table but no procedure — populate those by hand when testing.
- `advisor_id` and `student_id` are both `NUMBER(38,0)`, matching the SIS.

### C.4 Coverage check before enabling enforcement

```sql
SELECT
    COUNT(DISTINCT s.advisor_id)                             AS advisors_in_sis,
    COUNT(DISTINCT x.advisor_id)                             AS advisors_mapped,
    COUNT(DISTINCT CASE WHEN x.advisor_id IS NULL
                        THEN s.advisor_id END)               AS advisors_unmapped
FROM MERRIMACK_DD_PROD.deterge.stg_jcx__students s
LEFT JOIN MERRIMACK_DD_PROD.governance.advisor_username_crosswalk x
       ON s.advisor_id = x.advisor_id
WHERE s.advisor_id IS NOT NULL;
```

An unmapped advisor is not a security hole — they simply see zero rows. But it is a support ticket waiting to happen, so drive `advisors_unmapped` to zero before flipping the flag.

---

## D. Changing a Role's Access Tier

Tiers are data, not code — no redeploy needed. The policy reads `role_domain_access` at query time, so a tier change takes effect on the next query.

```sql
USE ROLE SYSADMIN;

UPDATE MERRIMACK_DD_PROD.governance.role_domain_access
SET access_tier = 'FULL'
WHERE role_name = 'MERRIMACK_ADVISOR_ROLE'
  AND data_domain = 'student_academic';
```

Apply the same change to DEV and TEST so environments don't drift. If a change should apply to *all* schools, update `seeds/shared/seed_rbac_role_definitions.csv` in `ditteau_data_transform` as well — that file is the documented default for schools onboarded later, and the tables will otherwise diverge from it silently.

Valid tiers are `FULL`, `SCOPED`, `AGGREGATED`, `NONE`. There is no check constraint; a typo behaves as deny.

---

## E. Testing a RAP as a Role

### E.1 Four constraints that produce misleading results

1. **A row access policy is not a callable UDF.** `SELECT rap_student_academic('123')` fails with SQL compilation error `002141`. The body only evaluates when the policy is attached to a table.
2. **Never test as `SYSADMIN` or `ACCOUNTADMIN`.** The policy short-circuits to `TRUE`, so every query returns all rows and the test appears to pass regardless of tier logic.
3. **The business roles have no DEV/TEST compute.** They are granted `{CODE}_ANALYTICS_PROD` only, so DEV testing needs temporary warehouse, database, and schema grants.
4. **Setting the role at connect time does not restrict the session.** `LVANPELT` has `DEFAULT_SECONDARY_ROLES = ["ALL"]`, so `ACCOUNTADMIN` and `DITTEAU_ADMIN` are active as *secondary* roles in every session no matter what primary role is requested. Object access resolves against the union of all active roles, so constraint 2 is not satisfied merely by passing `--role` or setting `role:` in a profile.

   Every session that tests access **must** begin:

   ```sql
   USE SECONDARY ROLES NONE;
   ```

   Confirm it took effect before trusting any result:

   ```sql
   SELECT CURRENT_ROLE(), CURRENT_SECONDARY_ROLES();
   -- expect the requested role and {"roles":"","value":"NONE"}
   ```

   Found 2026-08-15 while verifying `add_service_role_source_grants.sql`. Three deliberate cross-tenant reads — `DEMEAU_DBT_DEV` against `MERRIMACK_CX_ARCHIVE` and two others — all **succeeded**, returning 507,026 Merrimack rows to a DEMEAU role. Tenant isolation was in fact correct; the test was measuring the user's privilege rather than the role's. With `USE SECONDARY ROLES NONE` the same three reads fail with `002003` "Database does not exist or not authorized," which is the correct result.

   This is the same failure shape as the false attachment validation in F.0: a check that reports success while measuring nothing. Note the direction of the error — it produced a **false alarm** here, but the identical mechanism produces a **false pass** whenever the test expects access to be allowed.

   Dedicated service users (`SVC_{CODE}_DBT_{ENV}`) are the durable answer, since each holds exactly one role. They are not usable yet — see E.6.

### E.6 Service users exist but cannot authenticate

Every school has `SVC_{CODE}_DBT_{DEV,TEST,PROD}` with the matching role as `DEFAULT_ROLE`. Running dbt as these users — rather than as a person with `--role` — is the designed mechanism and the only one that makes constraint 4 unnecessary.

Two things block it, as observed 2026-08-15 on all three DEMEAU service users:

| Property | Current | Needed |
|---|---|---|
| `RSA_PUBLIC_KEY` | `null` | a registered public key |
| `TYPE` | `PERSON` | `SERVICE` |

`profiles.yml` uses `authenticator: snowflake_jwt`, which requires key-pair auth. These users have passwords only and no key, so no dbt target can currently connect as them. Separately, `TYPE = PERSON` subjects them to MFA policy, which will break unattended runs regardless of the key.

`DEFAULT_SECONDARY_ROLES = ["ALL"]` is also set on all three, but is currently harmless: each service user holds exactly one role, so `ALL` resolves to that role alone. It becomes a live risk the moment any additional role is granted to one of them, and setting it to `NONE` costs nothing today.

There is also a type constraint: the policy signature is `NUMBER(38,0)` and the column it attaches to must be in the NUMBER family. Snowflake matches on type *family*, not precision or scale — a `NUMBER(38,0)` policy attaches to `dim_student.student_id` at `NUMBER(38,5)` without complaint, but would be rejected by a `VARCHAR` column.

### E.2 Scratch-table test procedure

Run with `ditteau_data_infra/school_setup/run_migration.py`, which keeps all statements in one session so `USE ROLE` switching works.

```sql
USE ROLE SYSADMIN;

-- Fixture: 100 real student_ids, native column type, no cast
CREATE OR REPLACE TABLE DEMEAU_DD_DEV.governance.rap_test_students AS
SELECT DISTINCT student_id
FROM DEMEAU_DD_DEV.distribute.dim_student
WHERE student_id IS NOT NULL AND student_id <> '-1'
LIMIT 100;

ALTER TABLE DEMEAU_DD_DEV.governance.rap_test_students
    ADD ROW ACCESS POLICY DEMEAU_DD_DEV.governance.rap_student_academic
    ON (student_id);

-- DEV access for the business roles.
-- These are already in place for DEMEAU (see E.5) — included here for other schools.
GRANT USAGE ON WAREHOUSE DEMEAU_TRANSFORM_DEV TO ROLE DEMEAU_ADVISOR_ROLE;
GRANT USAGE ON DATABASE DEMEAU_DD_DEV          TO ROLE DEMEAU_ADVISOR_ROLE;
GRANT USAGE ON SCHEMA DEMEAU_DD_DEV.governance TO ROLE DEMEAU_ADVISOR_ROLE;
GRANT SELECT ON TABLE DEMEAU_DD_DEV.governance.rap_test_students
                                               TO ROLE DEMEAU_ADVISOR_ROLE;
-- ... repeat per role under test

-- Seed the SCOPED predicate for the *testing* user
INSERT INTO DEMEAU_DD_DEV.governance.advisor_username_crosswalk
    (advisor_id, snowflake_username, display_name, is_active)
VALUES (999001, 'LVANPELT', 'RAP test advisor', TRUE);

INSERT INTO DEMEAU_DD_DEV.governance.advisor_student_map
    (snowflake_username, student_id, effective_date)
SELECT 'LVANPELT', student_id, CURRENT_DATE()
FROM DEMEAU_DD_DEV.governance.rap_test_students
LIMIT 5;
```

Then, in the same session:

```sql
USE ROLE DEMEAU_REGISTRAR_ROLE;   -- FULL       -> 100
SELECT COUNT(*) FROM DEMEAU_DD_DEV.governance.rap_test_students;
USE ROLE DEMEAU_ADVISOR_ROLE;     -- SCOPED     -> 5
SELECT COUNT(*) FROM DEMEAU_DD_DEV.governance.rap_test_students;
USE ROLE DEMEAU_FA_ROLE;          -- SCOPED     -> 5
SELECT COUNT(*) FROM DEMEAU_DD_DEV.governance.rap_test_students;
USE ROLE DEMEAU_IR_ANALYST_ROLE;  -- AGGREGATED -> 0
SELECT COUNT(*) FROM DEMEAU_DD_DEV.governance.rap_test_students;
USE ROLE DEMEAU_READ_DEV;         -- unmapped   -> 0
SELECT COUNT(*) FROM DEMEAU_DD_DEV.governance.rap_test_students;
```

Results recorded 2026-08-12 (DEMEAU DEV): 100 / 100 / 5 / 5 / 0 / 0 — all as expected, including the SYSADMIN bypass baseline.

### E.3 Cleanup

```sql
USE ROLE SYSADMIN;
DROP TABLE IF EXISTS DEMEAU_DD_DEV.governance.rap_test_students;
DELETE FROM DEMEAU_DD_DEV.governance.advisor_student_map
WHERE snowflake_username = 'LVANPELT';
DELETE FROM DEMEAU_DD_DEV.governance.advisor_username_crosswalk
WHERE advisor_id = 999001;
```

Drop the fixture *before* re-running any script that does `CREATE OR REPLACE ROW ACCESS POLICY` — Snowflake refuses to replace a policy that is still attached to something (`error 003531: cannot be dropped/replaced as it is associated with one or more entities`).

Note that the cleanup above deliberately does **not** revoke the DEV grants from [E.2](#e2-scratch-table-test-procedure). See [E.5](#e5-standing-dev-grants-for-demeau).

### E.4 Policy evaluation cost

Measured 2026-08-13 on DEMEAU DEV, scanning a 89,122-row copy of `fact_student_term` as `DEMEAU_REGISTRAR_ROLE` (tier `FULL`, so the `CASE` resolves through the tier lookup rather than short-circuiting on the SYSADMIN bypass). Five repetitions, `USE_CACHED_RESULT = FALSE`.

| Condition | Median | Min | Max |
|---|---|---|---|
| No policy attached | 352 ms | 240 | 418 |
| Phase 4 body — one scalar subquery | 501 ms | 360 | 576 |
| Phase 6 body — `COALESCE` of two | 240 ms | 227 | 333 |

**Read this as "no measurable difference," not as a speedup.** Phase 6 cannot genuinely be faster than no policy at all; the ordering is an artifact. Run-to-run variance on the unprotected baseline alone spans 178 ms, which is wider than any gap between conditions, so the policy overhead sits below this measurement's noise floor.

The mechanism explains why. Both tier subqueries are **uncorrelated** — they reference `CURRENT_USER()`, `CURRENT_ROLE()` and a literal domain string, never the row parameter. Snowflake evaluates them once per query, not once per row. Going from one subquery to two therefore adds a constant, not a per-row cost, and at any realistic table size that constant disappears into query overhead.

Consequence: **the fallback governance view unioning both tier tables is unnecessary.** It was specified as the remedy if the regression proved material. It did not.

**This conclusion is scoped to the *uncorrelated* tier lookups, and the `SCOPED` branch is already the exception.** Its `EXISTS` against `advisor_student_map` joins on `row_student_id`, so it *is* row-correlated. It did not appear in these numbers because the measurement ran as a `FULL` role, which resolves at the first `WHEN` and never reaches the `SCOPED` branch.

> **Before enforcement goes live, re-time as `DEMEAU_ADVISOR_ROLE` against `fact_student_term`.** That is the path where per-row cost is real, and it is the path advisors will actually use every day. Nothing above tells you what it costs.

Use `EXECUTION_TIME` from `QUERY_HISTORY` rather than client wall clock for that measurement — wall clock here includes connection and result round-trip, which is part of why the variance swamped the signal.

**On the fallback governance view — considered and ruled out, not overlooked.** The Phase 6 plan specified a view unioning `user_domain_access` and `role_domain_access` with a precedence column, queried once, as the remedy if two subqueries proved materially more expensive than one. It is unnecessary *because* the tier lookups are uncorrelated: a second uncorrelated subquery adds a constant, not a per-row cost, so collapsing two into one saves a constant that is already invisible. If a future slowdown appears on the tier lookup path, that view is the known remedy and this is the reason it was not built pre-emptively. Do not reinvent it without first confirming the lookups are still uncorrelated — if one becomes row-correlated, the view solves a different and much larger problem.

### E.5 Standing DEV grants for DEMEAU

The DEMEAU business roles have been left holding DEV access so this test is repeatable without re-granting each time. Deliberate, not drift — but it is a divergence from what `generate_school.py` produces, so it will not exist for other schools:

| Role | Standing grants in DEV |
|------|------------------------|
| `DEMEAU_REGISTRAR_ROLE` | USAGE on `DEMEAU_TRANSFORM_DEV`, `DEMEAU_DD_DEV`, `DEMEAU_DD_DEV.GOVERNANCE` |
| `DEMEAU_ADVISOR_ROLE` | same |
| `DEMEAU_FA_ROLE` | same |
| `DEMEAU_IR_ANALYST_ROLE` | same |
| `DEMEAU_READ_DEV` | USAGE on `DEMEAU_TRANSFORM_DEV`, `DEMEAU_DD_DEV.GOVERNANCE` |

Granted 2026-08-12 by SYSADMIN. The `SELECT` on the scratch table is not standing — it disappears with the table each time it is dropped.

This is safe because DEMEAU is synthetic demo data and the grants are read-only in DEV. **Do not copy this pattern to a school holding real student data** — there, grant DEV access only for the duration of a test and revoke it afterwards:

```sql
USE ROLE SYSADMIN;
REVOKE USAGE ON SCHEMA {SCHOOL}_DD_DEV.governance FROM ROLE {SCHOOL}_ADVISOR_ROLE;
REVOKE USAGE ON DATABASE {SCHOOL}_DD_DEV          FROM ROLE {SCHOOL}_ADVISOR_ROLE;
REVOKE USAGE ON WAREHOUSE {SCHOOL}_TRANSFORM_DEV  FROM ROLE {SCHOOL}_ADVISOR_ROLE;
```

To audit what a role actually holds at any point:

```sql
SHOW GRANTS TO ROLE DEMEAU_ADVISOR_ROLE;
```

---

## F. Enabling Enforcement in PROD

### F.0 The run scripts silently discard any var you pass

> ⚠️ **You cannot enable this flag through `scripts/run_<school>_dev.sh`.** Every script is shaped `dbt "$@" --target … --vars '{…}'` — its own `--vars` comes **after** `"$@"`, and dbt honours only the last `--vars`. Anything you pass is dropped without warning.

This produced a false pass on 2026-08-13. A full DEMEAU build invoked as
`run_demeau_dev.sh build --vars '{"enable_row_level_security": true}'` returned
`PASS=1213 WARN=16 ERROR=0` — and attached nothing, because the flag was still false and
`apply_rap()` no-opped. The build looked like successful validation and proved nothing.

To pass an extra var, replicate the school's var block and merge the key in, invoking
`dbt` directly. Or temporarily edit `dbt_project.yml` and revert. Do not assume a clean
build means the flag took effect — **confirm the flag reached dbt by observing its
effect**, per F.0.1.

### F.0.1 Verify attachment, never infer it

A clean build proves the hook did not raise. It does not prove attachment. Always
observe:

```sql
SELECT POLICY_NAME, REF_ENTITY_NAME, REF_ENTITY_DOMAIN, REF_ARG_COLUMN_NAMES
FROM TABLE({SCHOOL}_DD_{ENV}.INFORMATION_SCHEMA.POLICY_REFERENCES(
    POLICY_NAME => '{SCHOOL}_DD_{ENV}.governance.rap_student_academic'));
```

Note the column is `REF_ARG_COLUMN_NAMES` (a JSON array), not `ref_column_name`.

**Validation record — DEMEAU DEV, 2026-08-13.** First time the Phase 4 attachment path
ever executed. DEMEAU's CX share is de-identified and authorised for demonstration use,
which is what makes it the correct place to run this.

| Policy | Attached to | On column |
|---|---|---|
| `RAP_STUDENT_ACADEMIC` | `DIM_STUDENT` | `STUDENT_ID` |
| `RAP_STUDENT_ACADEMIC` | `FACT_ENROLLMENT` | `STUDENT_ID` |
| `RAP_STUDENT_ACADEMIC` | `FACT_STUDENT_TERM` | `STUDENT_ID` |

Row-correlated `SCOPED` path measured against the live attachment on
`fact_student_term` (89,122 rows, 500 advisee mappings, 5 reps, cache off):

| Tier | Median | Rows visible |
|---|---|---|
| `FULL` (uncorrelated path only) | 623 ms | 89,122 |
| `SCOPED` (row-correlated `EXISTS`) | 493 ms | 3,306 |

**The functional result is the finding, not the timing.** `SCOPED` correctly filtered
89,122 rows to the 3,306 belonging to the mapped advisees — the row-correlated predicate
works at scale against a real fact table. The timing cannot isolate policy overhead
because `SCOPED` returns 27× less data and therefore aggregates less; the two effects run
opposite directions. What it does establish is that there is **no material regression** at
this volume. Re-measure with `EXECUTION_TIME` from `QUERY_HISTORY` if the number ever
gates a decision.

After validating, the flag was removed and detachment confirmed — `POLICY_REFERENCES`
returned no rows. A normal `table`-materialisation rebuild detaches, because
`CREATE OR REPLACE TABLE` drops policy attachments.

### F.1 Pre-flight

Pre-flight, per school:

- [ ] `advisors_unmapped = 0` from the query in [C.4](#c4-coverage-check-before-enabling-enforcement)
- [ ] Tier assignments in `role_domain_access` reviewed against the school's actual org chart
- [ ] Scratch-table test passed in that school's DEV
- [ ] Sign-off obtained from KKM (Kelly, Director of Data Governance) — ADR-003 Phase 5 gate. This is the formal approval to enforce row-level access on live student data; get it in writing before the flag is flipped, not after.
- [ ] A rollback window agreed — enforcement changes what dashboards show

Then run a PROD build with the flag on:

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh build \
    --select dim_student fact_student_term fact_enrollment \
    --target merrimack_prod \
    --vars '{"enable_row_level_security": true}'
```

Note `--select` with multiple values must be one quoted string in some dbt versions; if the second model is rejected as an unknown argument, quote it as `--select "dim_student fact_student_term fact_enrollment"`.

Confirm attachment:

```sql
SELECT *
FROM TABLE(MERRIMACK_DD_PROD.information_schema.policy_references(
    policy_name => 'MERRIMACK_DD_PROD.governance.rap_student_academic'));
```

To make enforcement permanent, set `enable_row_level_security: true` in the PROD target's vars rather than passing it per-run — otherwise the next scheduled build detaches the policy, because `apply_rap()` becomes a no-op and the table is rebuilt without it.

### F.2 Rollback

```bash
# Rebuild without the flag; apply_rap() no-ops and the policy is not reattached
bash scripts/run_merrimack_dev.sh build \
  --select "dim_student fact_student_term fact_enrollment" \
  --target merrimack_prod
```

For an emergency detach without a rebuild:

```sql
USE ROLE SYSADMIN;
ALTER TABLE MERRIMACK_DD_PROD.distribute.dim_student DROP ALL ROW ACCESS POLICIES;
```

---

## G. Deploying to a New School

Schools created by `generate_school.py` get everything automatically — `generate_governance()` emits the tier table, both mapping tables, the refresh procedure, the RAP, and the APPLY grants as Sections 3 and 4 of `05_governance.sql`.

For a school created before August 2026, run the migrations in this order. Order matters: `role_domain_access.role_name` is a plain `VARCHAR` with no foreign key, so seeding tiers before the roles exist succeeds silently and leaves 40 rows pointing at nothing.

```bash
cd ~/ditteau_data_infra/school_setup

# 1. Business-function roles (Phase 2)
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  python run_migration.py migrations/add_business_roles.sql --dry-run
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  python run_migration.py migrations/add_business_roles.sql

# 2. Governance tables + refresh proc (Phase 3)
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  python run_migration.py migrations/add_governance_phase3_tables.sql

# 3. RAP function + APPLY grants (Phase 4)
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  python run_migration.py migrations/add_governance_phase4_rap.sql
```

All three are idempotent and safe to re-run. `run_migration.py` splits on `$$`-quoted procedure bodies correctly — a naive split on `;` shreds them — and reports per-statement status. Always `--dry-run` first.

The migration files are hard-coded to MERRIMACK, ANSELM, and DEMEAU; adding a fourth school means adding a block, or preferably generating `05_governance.sql` and running that instead.

Finally, verify:

```sql
SELECT COUNT(*) FROM {SCHOOL}_DD_PROD.governance.role_domain_access;  -- expect 40
SHOW ROW ACCESS POLICIES IN SCHEMA {SCHOOL}_DD_PROD.governance;       -- expect 1
SHOW PROCEDURES LIKE 'REFRESH_ADVISOR_STUDENT_MAP'
  IN SCHEMA {SCHOOL}_DD_PROD.governance;                              -- expect 1
```

---

## H. Adding a New Domain RAP

`financial_aid`, `financial`, and `admissions` have tier rows seeded but no policy. To add one:

1. **Choose the attachment column.** It must exist on every target model and be a natural key present in whatever mapping table the `SCOPED` predicate reads. Check the type first — the policy signature must match the column's type family:

   ```sql
   SELECT table_name, column_name, data_type, numeric_precision, numeric_scale
   FROM MERRIMACK_DD_PROD.INFORMATION_SCHEMA.COLUMNS
   WHERE table_schema = 'DISTRIBUTE' AND column_name = '<your_key>';
   ```

   If the models disagree on type family, fix that before writing the policy. `fact_enrollment` had to gain a natural `student_id` for exactly this reason — attaching to the surrogate `enrollment_student_key` would have compiled and then matched nothing in `advisor_student_map`, silently returning zero rows for every SCOPED user.

2. **Decide what `SCOPED` means for the domain**, or drop the tier. For `financial_aid` there is no advisor analogue; `SCOPED` may need its own mapping table, or the tier should be collapsed to `FULL`/`NONE`.

3. **Write the policy** into `generate_school.py` Section 4 *and* a new migration for existing schools. Keep both in sync — the generator is the source of truth for new schools, the migration for old ones.

4. **Grant APPLY to the dbt service roles**, not to `SYSADMIN`. `SYSADMIN` owns the policies and already holds APPLY implicitly, so granting to it is a no-op; the role that runs the post-hook is `{CODE}_DBT_{ENV}`, which reaches the governance schema via `{CODE}_TRANSFORM_{ENV}`.

5. **Extend `apply_rap()`** or add a sibling macro. The current macro hard-codes `rap_student_academic` and `ON (student_id)`; a second domain needs either parameters or its own macro, plus `+post-hook:` entries in `dbt_project.yml`.

6. **Tag the models** with `meta: data_domain: <domain>` so the mapping from model to policy is discoverable in the manifest.

---

## I. Troubleshooting

### A role sees zero rows and shouldn't

Work down the evaluation chain:

```sql
-- 1. Is there a tier row at all? Missing -> ELSE FALSE -> deny
SELECT * FROM MERRIMACK_DD_PROD.governance.role_domain_access
WHERE role_name = 'MERRIMACK_ADVISOR_ROLE';

-- 2. Exact string match? The lookup is =, not ILIKE
SELECT DISTINCT role_name FROM MERRIMACK_DD_PROD.governance.role_domain_access;

-- 3. For SCOPED: does this *user* have advisees?
SELECT COUNT(*) FROM MERRIMACK_DD_PROD.governance.advisor_student_map
WHERE snowflake_username = 'JSMITH';

-- 4. Is the crosswalk entry active, and did the refresh run since?
SELECT * FROM MERRIMACK_DD_PROD.governance.advisor_username_crosswalk
WHERE snowflake_username = 'JSMITH';
```

The most common cause is a `snowflake_username` that holds a display name instead of a login name. `CURRENT_USER()` returns `login_name`; confirm with `SHOW USERS`.

### `'apply_rap' is undefined`

The post-hook is in a model YAML `config:` block. Move it to `dbt_project.yml` — see [A.5](#a5-attachment-is-owned-by-dbt).

### `Column 'X' data type 'NUMBER(38,5)' does not match with Row access policy data type 'VARCHAR(...)'`

The policy signature and column type are in different families. Type family must match; precision and scale need not. See [E.1](#e1-three-constraints-that-produce-misleading-results).

### `Policy RAP_STUDENT_ACADEMIC cannot be dropped/replaced as it is associated with one or more entities`

Something still has the policy attached — usually a leftover test fixture. Find it and detach:

```sql
SELECT * FROM TABLE(MERRIMACK_DD_PROD.information_schema.policy_references(
    policy_name => 'MERRIMACK_DD_PROD.governance.rap_student_academic'));
```

### Policy vanished after a rebuild

Expected if `enable_row_level_security` was not `true` for that run. `apply_rap()` no-ops and the recreated table has no policy. Set the var in the PROD target rather than passing it ad hoc.

### `refresh_advisor_student_map()` returns 0 assignments

Either the crosswalk is empty, every entry is `is_active = FALSE`, or `stg_jcx__students` has not been built in that environment. The procedure reads the deterge layer, so it must run after staging.

---

## References

- [ADR-003: Domain-Scoped Row Access Policy Architecture](../decisions/adr-003-row-access-policy-architecture.md)
- [Snowflake: Row Access Policies](https://docs.snowflake.com/en/user-guide/security-row-intro)
- `ditteau_data_infra/school_setup/run_migration.py` — migration runner
- `ditteau_data_infra/school_setup/generate_school.py` — `generate_governance()`, Sections 3 and 4
- `ditteau_data_transform/macros/governance/apply_rap.sql` — attachment post-hook
- `ditteau_data_transform/seeds/shared/seed_rbac_role_definitions.csv` — default tiers
- `ditteau_data_transform/docs/governance/data_governance_policy.md` — gap G-03
