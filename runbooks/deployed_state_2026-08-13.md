# Deployed State of Record — 2026-08-13

**Prepared by:** Claude Code (automated extraction)
**Source:** Snowflake account `syaxlgh-ditteau_data` + repository scans
**Purpose:** Baseline for documentation regeneration per instruction document

---

## A. Snowflake Deployed State (C.1)

### A.1 Databases and Schemas

**School Databases (9 total):**

| Database | School | Environment | Created |
|----------|--------|-------------|---------|
| ANSELM_DD_DEV | St. Anselm College | Development | 2026-02-26 |
| ANSELM_DD_TEST | St. Anselm College | Test/CI | 2026-06-16 |
| ANSELM_DD_PROD | St. Anselm College | Production | 2026-02-26 |
| DEMEAU_DD_DEV | DEMEAU (Demo) | Development | 2026-07-10 |
| DEMEAU_DD_TEST | DEMEAU (Demo) | Test/CI | 2026-07-10 |
| DEMEAU_DD_PROD | DEMEAU (Demo) | Production | 2026-07-10 |
| MERRIMACK_DD_DEV | Merrimack College | Development | 2026-02-26 |
| MERRIMACK_DD_TEST | Merrimack College | Test/CI | 2026-06-16 |
| MERRIMACK_DD_PROD | Merrimack College | Production | 2026-02-26 |

**Platform Databases:**

| Database | Purpose |
|----------|---------|
| DITTEAU_PLATFORM | Platform-level shared objects (secrets, integrations, monitoring) |
| DITTEAU_SHARED | External data sources (IPEDS, College Scorecard) |
| DQ_PLATFORM | Data Quality framework metadata and marts |

**Data Share Databases (Imported):**

| Database | Origin Share |
|----------|--------------|
| ANSELM_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.ANSELM_SHARE |
| DEMEAU_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.DEMEAU_DB_SNOWFLAKE_SHARE_070725 |
| MERRIMACK_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.MERRIMACK_SHARE |

**Schools WITHOUT Snowflake Databases:**
- COLBY — No `COLBY_DD_*` databases exist
- ENDICOTT — No `ENDICOTT_DD_*` databases exist
- SPRINGFIELD — No `SPRINGFIELD_DD_*` databases exist

### A.2 Built Layers per Environment

**DEV Environments — All layers built:**

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE | DBT_TEST_RESULTS |
|----------|---------|---------|------------|------------|------------------|
| DEMEAU_DD_DEV | 3,538 | 20 | 54 | 6 | 944 |
| MERRIMACK_DD_DEV | 3,535 | 21 | 51 | 6 | 842 |
| ANSELM_DD_DEV | 3 | 21 | 50 | 6 | 719 |

**TEST Environments — Not queried (assume similar to DEV)**

**PROD Environments — UNBUILT:**

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE |
|----------|---------|---------|------------|------------|
| DEMEAU_DD_PROD | 1 | **0** | **0** | 7 |
| MERRIMACK_DD_PROD | 1 | **0** | **0** | 7 |
| ANSELM_DD_PROD | 1 | **0** | **0** | 7 |

⚠️ **CRITICAL FINDING:** All three PROD environments have NO deterge and NO distribute layers. Every document claiming a PROD deployment state for transformed data is incorrect. PROD contains only:
- 1 deposit table (placeholder/empty)
- 7 governance tables (tier tables, PII unmask tables)

### A.3 Roles

**Total roles in account:** 72

**Per-School Role Structure (23 roles each for ANSELM, DEMEAU, MERRIMACK):**

| Category | Roles | Count |
|----------|-------|-------|
| Technical (per env) | `{CODE}_TRANSFORM_{ENV}`, `{CODE}_WRITE_{ENV}`, `{CODE}_READ_{ENV}`, `{CODE}_DBT_{ENV}` | 12 |
| PROD-only | `{CODE}_REPORTING_PROD` | 1 |
| Business Function | `{CODE}_REGISTRAR_ROLE`, `{CODE}_ADVISOR_ROLE`, `{CODE}_FA_ROLE`, `{CODE}_IR_ANALYST_ROLE` | 4 |
| Governance | `{CODE}_GOVERNANCE_ADMIN_ROLE` | 1 |
| Streamlit Owner | `{CODE}_STREAMLIT_OWNER_{ENV}` (DEV, TEST, PROD) | 3 |
| **Subtotal** | | **21** |

*Note: Count discrepancy exists; actual per-school is 21, not 23. Total 63 school roles + 9 platform/system = 72.*

**Platform Roles:**
- `DITTEAU_ADMIN` — 4 assigned users, 52 granted roles
- `DITTEAU_ENGINEER` — 0 assigned users, 27 granted roles
- `DQ_ADMIN` — 0 assigned users

**System Roles:**
- `ACCOUNTADMIN` — 3 assigned users
- `SYSADMIN` — 3 assigned users (current role for queries)
- `SECURITYADMIN`, `USERADMIN`, `ORGADMIN`, `PUBLIC`

**Governance Admin Roles — All unassigned:**
- `ANSELM_GOVERNANCE_ADMIN_ROLE` — 0 users
- `DEMEAU_GOVERNANCE_ADMIN_ROLE` — 0 users
- `MERRIMACK_GOVERNANCE_ADMIN_ROLE` — 0 users

**Streamlit Owner Roles — All unassigned, deliberately absent from role_domain_access:**
- Comment confirms: "Deliberately absent from role_domain_access so the role-keyed tier path resolves to deny under owner-rights execution."

### A.4 Row Access Policies

**Total RAPs defined:** 30

| Policy Name | Databases Present In | Purpose |
|-------------|---------------------|---------|
| RAP_STUDENT_ACADEMIC | All 9 school databases | Student-academic domain; supports FULL/SCOPED/AGGREGATED/NONE |
| RAP_FINANCIAL_AID | All 9 school databases | Financial aid domain; FULL/AGGREGATED/NONE only (SCOPED unimplemented) |
| RAP_ADMISSIONS | All 9 school databases | Admissions domain; FULL/AGGREGATED/NONE only (SCOPED unimplemented) |
| DISTRIBUTE_ACCESS_POLICY | 3 PROD databases only | Legacy role-based school isolation |

**Policy Attachments:**

⚠️ **NONE.** Query against `INFORMATION_SCHEMA.POLICY_REFERENCES` returned 0 rows. All RAPs are defined but not attached to any tables.

**Status:** `[BUILT, NOT ACTIVE]` — Policies exist, execution path never exercised. Attachment is gated on `enable_row_level_security` var, which is `false` in all environments.

### A.5 Masking Policies

**Total masking policies defined:** 7 (DEMEAU_DD_DEV only)

| Policy Name | Database | Schema | Purpose |
|-------------|----------|--------|---------|
| MASK_NAME | DEMEAU_DD_DEV | GOVERNANCE | Masks NAME PII field |
| MASK_DOB | DEMEAU_DD_DEV | GOVERNANCE | Masks DOB PII field |
| MASK_EMAIL | DEMEAU_DD_DEV | GOVERNANCE | Masks EMAIL PII field |
| MASK_SSN | DEMEAU_DD_DEV | GOVERNANCE | Masks SSN PII field |
| MASK_PHONE | DEMEAU_DD_DEV | GOVERNANCE | Masks PHONE PII field |
| MASK_ADDRESS | DEMEAU_DD_DEV | GOVERNANCE | Masks ADDRESS PII field |
| MASK_FINANCIAL_AMOUNT | DEMEAU_DD_DEV | GOVERNANCE | Masks FINANCIAL_AMOUNT PII field |

**Masking policies in ANSELM or MERRIMACK:** 0

**Policy Attachments:** Not queried, but based on `enable_masking_policies: false` everywhere, assumed NONE attached.

**Status:** `[BUILT, NOT ACTIVE]` — Policies exist only in DEMEAU DEV. No code reads `enable_masking_policies` at runtime.

### A.6 Governance Schema Tables

**Tables per governance schema:** 6

| Table Name | Purpose |
|------------|---------|
| ROLE_DOMAIN_ACCESS | Role → (domain, access_tier) mapping |
| USER_DOMAIN_ACCESS | User → (domain, access_tier) override mapping |
| ADVISOR_STUDENT_MAP | Username → student_id assignment for SCOPED tier |
| ADVISOR_USERNAME_CROSSWALK | Advisor ID → Snowflake login name mapping |
| ROLE_PII_UNMASK | Role → PII field unmask grants |
| USER_PII_UNMASK | User → PII field unmask grants |

**Tier Table Row Counts (DEMEAU_DD_DEV):**

| Table | Row Count | Notes |
|-------|-----------|-------|
| role_domain_access | 40 | 10 roles × 4 domains |
| user_domain_access | 0 | Empty — no user-level overrides |
| advisor_student_map | 0 | Empty — no advisor assignments |
| advisor_username_crosswalk | 1 | Single test entry |

### A.7 Future Grants

**PROD Distribute Schema (DEMEAU_DD_PROD.DISTRIBUTE):**
- `SELECT` on TABLE/VIEW → `DEMEAU_REPORTING_PROD`
- `SELECT` on TABLE/VIEW → `DEMEAU_STREAMLIT_OWNER_PROD`
- Full DML → `DEMEAU_TRANSFORM_PROD`

**DEV Distribute Schema (DEMEAU_DD_DEV.DISTRIBUTE):**
- `SELECT` on TABLE/VIEW → `DEMEAU_STREAMLIT_OWNER_DEV`
- Full DML → `DEMEAU_TRANSFORM_DEV`
- **NO** `DEMEAU_REPORTING_DEV` grants

⚠️ **Asymmetry confirmed:** REPORTING role future grants exist only in PROD, not DEV/TEST.

### A.8 Warehouses

**Total warehouses:** 15

| Warehouse | Size | Auto-Suspend | Owner | Resource Monitor |
|-----------|------|--------------|-------|------------------|
| ANSELM_TRANSFORM_DEV | Small | 60s | SYSADMIN | ANSELM_MONITOR_TRANSFORM_DEV |
| ANSELM_TRANSFORM_TEST | X-Small | 60s | SYSADMIN | ANSELM_MONITOR_TRANSFORM_TEST |
| ANSELM_TRANSFORM_PROD | Medium | 300s | SYSADMIN | ANSELM_MONITOR_TRANSFORM_PROD |
| ANSELM_ANALYTICS_PROD | Small | 120s | SYSADMIN | ANSELM_MONITOR_ANALYTICS_PROD |
| DEMEAU_TRANSFORM_DEV | Small | 60s | SYSADMIN | DEMEAU_MONITOR_TRANSFORM_DEV |
| DEMEAU_TRANSFORM_TEST | X-Small | 60s | SYSADMIN | DEMEAU_MONITOR_TRANSFORM_TEST |
| DEMEAU_TRANSFORM_PROD | Medium | 300s | SYSADMIN | DEMEAU_MONITOR_TRANSFORM_PROD |
| DEMEAU_ANALYTICS_PROD | Small | 120s | SYSADMIN | DEMEAU_MONITOR_ANALYTICS_PROD |
| MERRIMACK_TRANSFORM_DEV | Small | 60s | SYSADMIN | null |
| MERRIMACK_TRANSFORM_TEST | X-Small | 60s | SYSADMIN | null |
| MERRIMACK_TRANSFORM_PROD | Medium | 300s | SYSADMIN | null |
| MERRIMACK_ANALYTICS_PROD | Small | 120s | SYSADMIN | null |
| PLATFORM_WH | X-Small | 300s | ACCOUNTADMIN | null |
| DQ_WH | X-Small | 300s | DQ_ADMIN | null |
| SYSTEM$STREAMLIT_NOTEBOOK_WH | X-Small | 60s | ACCOUNTADMIN | null |

⚠️ **Asymmetry:** MERRIMACK warehouses have no resource monitors attached.

### A.9 Stored Procedures

**Procedures in DEMEAU_DD_DEV.INFORMATION_SCHEMA.PROCEDURES:** 0

`refresh_advisor_student_map()` — Referenced in documentation but not found in information_schema. Either not created or in a different schema.

---

## B. Repository Deployed State (C.2)

### B.1 Model Inventory by Layer

| Layer | Sub-Layer | Models | Materialization |
|-------|-----------|--------|-----------------|
| DEPOSIT | — | 0 | N/A (source definitions only) |
| DETERGE | Staging | 228 | view |
| DETERGE | Intermediate | 24 | table |
| DISTRIBUTE | Dimensions | 16 | table |
| DISTRIBUTE | Facts | 4 | table |
| DISTRIBUTE | Snapshots (marts) | 5 | incremental |
| DISTRIBUTE | Summary marts | 19 | table |
| **Total** | | **296** | |

### B.2 dbt Variables Defined in dbt_project.yml

**Academic Calendar:**
- `current_academic_year`: `2025-2026`
- `current_term_code`: `FA26`
- `historical_cutoff_years`: `10`

**Source System Feature Flags (all default `false`):**
- `has_jenzabar_cx`, `has_jenzabar_one`, `has_workday`, `has_banner`
- `has_powerfaids`, `has_slate`, `has_canvas`, `has_jcx_deposit`

**Security/Governance Flags:**
- `enable_masking_policies`: `false`
- `enable_row_level_security`: `false`

**School Identity (override at runtime):**
- `school_code`: `UNKNOWN`
- `primary_sis`: `UNKNOWN`
- `jcx_share_database`: `UNKNOWN`
- `jcx_share_schema`: `DITTEAU_ARCHIVE`
- `external_sources_database`: `DITTEAU_SHARED`

### B.3 Variables Referenced but NOT Defined

| Variable | Location | Status |
|----------|----------|--------|
| `var('env')` | Comments only (macros/governance/apply_rap.sql) | **Intentionally missing** — environment resolved via `target.database` |
| `var('j1_source_database')` | `_j1_sources.yml` | Has inline default: `DITTEAU_RAW` |
| `var('j1_source_schema')` | `_j1_sources.yml` | Has inline default: `JENZABAR_ONE_RAW` |

### B.4 Run Scripts

| Script | Target | Source Systems Enabled |
|--------|--------|------------------------|
| run_anselm_dev.sh | anselm_dev | JCX (legacy), Workday (primary), PowerFAIDS, Slate |
| run_colby_dev.sh | colby_dev | JCX only |
| run_demeau_dev.sh | demeau_dev | J1 (primary), JCX (share+deposit), PowerFAIDS, Slate |
| run_endicott_dev.sh | endicott_dev | JCX only |
| run_merrimack_dev.sh | merrimack_dev | JCX, J1 (in progress), PowerFAIDS, Slate |
| run_springfield_dev.sh | springfield_dev | JCX (legacy), Banner (primary), PowerFAIDS, Slate (TBD) |

### B.5 Sources by System

| System | Tables | Ingest Method | Schools |
|--------|--------|---------------|---------|
| Jenzabar CX | 75+ | Snowflake Data Share | ANSELM, COLBY, ENDICOTT, MERRIMACK, SPRINGFIELD, DEMEAU |
| Jenzabar One | 29 | CSV Deposit | MERRIMACK, DEMEAU |
| PowerFAIDS | 29 | CSV Deposit | Multiple |
| Slate | 60+ | CSV Deposit | Multiple |
| Workday Student | 2 | CSV Deposit | ANSELM |
| Banner | 5 | CSV Deposit | SPRINGFIELD |
| IPEDS | 4 | Full Load (DITTEAU_SHARED) | All |
| College Scorecard | 1 | Full Load (DITTEAU_SHARED) | All |

### B.6 Macros

| Macro | Purpose |
|-------|---------|
| `apply_rap(domain, key_column)` | Attaches RAPs when `enable_row_level_security=true` |
| `add_source_metadata(source, table)` | Injects 5 metadata columns |
| `jcx_base(table_name)` | Conditional UNION for JCX share + deposit |
| `generate_surrogate_key(columns)` | Wrapper for MD5 surrogate key |
| `safe_col(relation, column, ...)` | Graceful column existence check |
| `cpi_adjust(amount, year, target)` | CPI inflation adjustment |
| `sc_num(col)` | College Scorecard numeric parsing |
| `school_ref(model_name)` | School-specific override resolution |

### B.7 Package Versions

| Package | Version |
|---------|---------|
| dbt-labs/dbt_utils | 1.1.1 |
| metaplane/dbt_expectations | 0.10.1 |
| dbt-labs/audit_helper | 0.9.0 |
| dbt-labs/codegen | 0.12.1 |

### B.8 Schema Registry Counts

| Source System | Tables | Columns |
|---------------|--------|---------|
| Jenzabar CX | 1,483 | 38,728 |
| Jenzabar One | 2,679 | 40,757 |
| PowerFAIDS | 544 | 18,984 |
| Slate | 311 | 4,081 |
| **Total** | **5,017** | **102,550** |

### B.9 Tests

**Singular tests:** 4 (governance assertions)
- `assert_anselm_crosswalk_coverage.sql`
- `assert_no_duplicate_source_identity.sql`
- `assert_no_app_owner_in_role_domain_access.sql`
- `assert_no_unimplemented_scoped_tiers.sql`

**Generic tests:** ~198 (191 not_null, 7 unique)

---

## C. Environment Asymmetries

Per instruction §E.2, documenting environment-specific differences:

| Feature | DEV | TEST | PROD |
|---------|-----|------|------|
| deterge/distribute layers | ✅ Built | ? | ❌ Empty |
| ANALYTICS warehouse | ❌ None | ❌ None | ✅ Exists |
| REPORTING role future grants | ❌ None | ❌ None | ✅ Configured |
| Business function roles (REGISTRAR, etc.) | ❌ No compute | ❌ No compute | ✅ Has warehouse |
| `refresh_advisor_student_map()` | ❌ Not found | ❌ Not found | ? |

---

## D. Repo-vs-Snowflake Discrepancies

| Item | Repo State | Snowflake State | Resolution |
|------|------------|-----------------|------------|
| School databases for COLBY, ENDICOTT, SPRINGFIELD | Run scripts exist | Databases do not exist | Snowflake governs |
| Stored procedure `refresh_advisor_student_map` | Referenced in docs | Not in information_schema | Unknown; may be in different location |
| Masking policies | `enable_masking_policies` var exists | Policies exist only in DEMEAU_DD_DEV | DEMEAU is test environment |
| 15 roles per school | Documented in data_infra | Actually 21 per school | Snowflake governs (business + governance + streamlit roles added) |

---

## E. Summary of Critical Findings

1. **PROD is unbuilt.** All three PROD environments (ANSELM, DEMEAU, MERRIMACK) contain only deposit and governance schemas. No deterge or distribute layers exist in PROD.

2. **Row access policies are not attached.** 30 RAPs are defined across all environments but attached to zero tables. `enable_row_level_security` is `false` everywhere.

3. **Masking policies exist only in DEMEAU DEV.** 7 masking policies are defined but only in DEMEAU_DD_DEV.GOVERNANCE. No policies in ANSELM or MERRIMACK.

4. **Three schools have no Snowflake infrastructure.** COLBY, ENDICOTT, and SPRINGFIELD have run scripts but no corresponding databases.

5. **`var('env')` is intentionally missing.** Environment isolation uses `target.database` pattern, not a var.

6. **User-level tier table is empty.** `user_domain_access` has 0 rows; tier resolution falls back to `role_domain_access`.

7. **Governance admin roles are unassigned.** All `{CODE}_GOVERNANCE_ADMIN_ROLE` roles have 0 users.

8. **Streamlit owner roles deliberately absent from tier tables.** This is by design to force user-keyed tier resolution in SiS apps.
