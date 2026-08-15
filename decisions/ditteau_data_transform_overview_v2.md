# ditteau_data_transform — Project Overview

**Regenerated:** 2026-08-13 from deployed state
**Source:** Repository scan + Snowflake queries
**Status:** Replaces prior `ditteau_data_transform_overview.md`

---

## 1. Project Summary

`ditteau_data_transform` is a dbt project for **Ditteau Data** — a higher education data platform built on Snowflake. It transforms raw institutional data from multiple source systems into analytics-ready dimensional models.

**Key Statistics:**
- **Total Models:** 296
- **Source Systems:** 8 (Jenzabar CX, Jenzabar One, Workday, Banner, PowerFAIDS, Slate, IPEDS, College Scorecard)
- **Schools Configured:** 6 (3 with deployed databases, 3 with scripts only)
- **Environments:** DEV, TEST, PROD per school

---

## 2. Architecture: Three-Layer Medallion

| Layer | Folder | Schema | Purpose |
|-------|--------|--------|---------|
| **Deposit** | `models/deposit/` | `deposit` | Raw data landing zone — sourced via stages, shares, or CSV deposit |
| **Deterge** | `models/deterge/` | `deterge` | Cleaned, conformed, and cross-source integrated data |
| **Distribute** | `models/distribute/` | `distribute` | Analytics-ready dimensional models (star schema) |

### 2.1 Deterge Sub-Layers

| Sub-Layer | Materialization | Purpose |
|-----------|-----------------|---------|
| `deterge/staging/` | **view** | Lightweight source conformance, one model per source table |
| `deterge/intermediate/` | **table** | Cross-source joins and business logic |

### 2.2 Distribute Sub-Layers

| Sub-Layer | Materialization | Purpose |
|-----------|-----------------|---------|
| `distribute/dimensions/` | **table** | SCD-aware dimension tables |
| `distribute/facts/` | **table** | Fact tables with documented grain |
| `distribute/marts/` | **table** / **incremental** | Business data marts and snapshots |

---

## 3. Model Inventory

### 3.1 Counts by Layer

| Layer | Sub-Layer | Model Count | Materialization |
|-------|-----------|-------------|-----------------|
| DEPOSIT | — | 0 | N/A (source definitions only) |
| DETERGE | Staging | 228 | view |
| DETERGE | Intermediate | 24 | table |
| DISTRIBUTE | Dimensions | 16 | table |
| DISTRIBUTE | Facts | 4 | table |
| DISTRIBUTE | Snapshots | 5 | incremental |
| DISTRIBUTE | Summary Marts | 19 | table |
| **Total** | | **296** | |

### 3.2 Staging Models by Source System

| Source System | Model Count | Prefix |
|---------------|-------------|--------|
| Jenzabar CX | 45 | `stg_jcx__` |
| Slate | 129 | `stg_slate__` |
| PowerFAIDS | 25 | `stg_pf__` |
| Jenzabar One | 16 | `stg_j1__` |
| IPEDS | 3 | `stg_ipeds__` |
| College Scorecard | 1 | `stg_scorecard__` |
| Workday | 2 | `stg_wd__` |
| Banner | 0 | `stg_banner__` (feature-flagged) |
| School Overrides | 7 | `anselm__`, `merrimack__` |

### 3.3 Distribute Layer Detail

**Dimensions (16):**
- `dim_student`, `dim_applicant`, `dim_term`, `dim_program`
- `dim_institution`, `dim_date`, `dim_aid_type`, `dim_gender`
- `dim_state`, `dim_major`, `dim_degree`, `dim_department`
- `dim_concentration`, `dim_minor`, `dim_subprogram`, `dim_integration_ids`

**Facts (4):**
- `fact_student_term` — grain: student × term
- `fact_enrollment` — grain: enrollment record
- `fact_application` — grain: application record
- `fact_aid_award` — grain: aid award record

**Snapshots (5, incremental):**
- `snap_admissions_weekly`, `snap_aid_term`, `snap_cohort_milestone`
- `snap_enrollment_term`, `snap_retention_term`

**Summary Marts (19):**
- Admissions: `mart_admissions_class_profile`, `mart_admissions_funnel`, `mart_admissions_progression`
- Enrollment: `mart_enrollment_census`, `mart_enrollment_census_ntr`, `mart_enrollment_demographics`, `mart_executive_summary`, `mart_retention_cohort_summary`
- Financial Aid: `mart_aid_leveraging`, `mart_aid_summary`, `mart_financial_aid_trend`
- Academic: `mart_academic_progress`, `mart_registration_holds`, `mart_section_utilization`, `mart_student_at_risk`
- IPEDS/Scorecard: `mart_ipeds_fall_enrollment`, `mart_ipeds_peer_comparison`, `mart_ipeds_reporting`, `mart_scorecard_program_outcomes`

---

## 4. Data Sources

### 4.1 Institutional SIS/Source Systems

| System | Prefix | Ingest Type | Schools | Feature Flag |
|--------|--------|-------------|---------|--------------|
| Jenzabar CX | `jcx` | Snowflake Data Share | ANSELM, COLBY, DEMEAU, ENDICOTT, MERRIMACK, SPRINGFIELD | `has_jenzabar_cx` |
| Jenzabar One | `j1` | CSV Deposit | DEMEAU, MERRIMACK | `has_jenzabar_one` |
| PowerFAIDS | `pf` | CSV Deposit | ANSELM, DEMEAU, MERRIMACK | `has_powerfaids` |
| Workday Student | `wd` | CSV Deposit | ANSELM | `has_workday` |
| Banner | `banner` | CSV Deposit | SPRINGFIELD | `has_banner` |
| Slate | `slate` | CSV Deposit | ANSELM, DEMEAU, MERRIMACK | `has_slate` |
| Canvas | `canvas` | CSV Deposit | (none currently) | `has_canvas` |

### 4.2 External Shared Data

| System | Prefix | Database | Schema | Update Cadence |
|--------|--------|----------|--------|----------------|
| IPEDS | `ipeds` | DITTEAU_SHARED | ipeds | Annual (Nov-Jan) |
| College Scorecard | `scorecard` | DITTEAU_SHARED | college_scorecard | Annual (Oct-Nov) |

### 4.3 JCX Data Share Configuration

Each school's CX data arrives via Snowflake org data share. The archive database is configured per school:

| School | Archive Database | Share Origin |
|--------|------------------|--------------|
| ANSELM | ANSELM_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.ANSELM_SHARE |
| DEMEAU | DEMEAU_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.DEMEAU_DB_SNOWFLAKE_SHARE_070725 |
| MERRIMACK | MERRIMACK_CX_ARCHIVE | SYAXLGH.DITTEAUEAST.MERRIMACK_SHARE |
| COLBY | COLBY_CX_ARCHIVE | **Does not exist** |
| ENDICOTT | (not configured) | **Does not exist** |
| SPRINGFIELD | SPRINGFIELD_CX_ARCHIVE | **Does not exist** |

---

## 5. Schools and Environments

### 5.1 School Deployment Status

| School | Code | Snowflake Databases | CX Archive | Status |
|--------|------|---------------------|------------|--------|
| St. Anselm College | ANSELM | ✅ DEV/TEST/PROD | ✅ Exists | **Active** |
| DEMEAU (Demo) | DEMEAU | ✅ DEV/TEST/PROD | ✅ Exists | **Active** |
| Merrimack College | MERRIMACK | ✅ DEV/TEST/PROD | ✅ Exists | **Active** |
| Colby College | COLBY | ❌ None | ❌ None | Scripts only |
| Endicott College | ENDICOTT | ❌ None | ❌ None | Scripts only |
| Springfield College | SPRINGFIELD | ❌ None | ❌ None | Scripts only |

⚠️ **COLBY, ENDICOTT, and SPRINGFIELD have run scripts but no deployed Snowflake infrastructure.** The run scripts reference CX archives that do not exist.

### 5.2 Database Naming Convention

Databases follow the pattern `{SCHOOL_CODE}_DD_{ENV}`:

| Environment | Database Example | Purpose |
|-------------|------------------|---------|
| DEV | `MERRIMACK_DD_DEV` | Development and testing |
| TEST | `MERRIMACK_DD_TEST` | CI/CD pipelines |
| PROD | `MERRIMACK_DD_PROD` | Production deployment |

### 5.3 Built Layers per Environment

**DEV environments — All layers built:**

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE |
|----------|---------|---------|------------|------------|
| DEMEAU_DD_DEV | 3,538 | 20 | 54 | 6 |
| MERRIMACK_DD_DEV | 3,535 | 21 | 51 | 6 |
| ANSELM_DD_DEV | 3 | 21 | 50 | 6 |

**PROD environments — UNBUILT:**

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE |
|----------|---------|---------|------------|------------|
| DEMEAU_DD_PROD | 1 | **0** | **0** | 7 |
| MERRIMACK_DD_PROD | 1 | **0** | **0** | 7 |
| ANSELM_DD_PROD | 1 | **0** | **0** | 7 |

⚠️ **All PROD environments have no deterge or distribute layers.** Transformed data exists only in DEV.

### 5.4 Merrimack SIS Migration Seam

Merrimack is configured with **both** Jenzabar CX and Jenzabar One enabled:

```json
{
  "primary_sis": "JENZABAR_ONE",
  "has_jenzabar_cx": true,
  "has_jenzabar_one": true
}
```

This is a migration seam:
- **Jenzabar One is the primary SIS.** Corrected 2026-08-14; this previously read
  `JENZABAR_CX`, which did not match the institution's actual system of record.
- J1 staging models read from `MERRIMACK_DD_DEV.DEPOSIT`
- The CX archive remains enabled and feeds the unified deterge/distribute layers

⚠️ The J1 deposit is currently sparse — 500 rows in `stg_j1__students` against 74,846 in
`stg_jcx__students`. Because identity anchors never re-anchor once minted
(`int_ditteau_id_registry.sql`), Merrimack keeps ~71,400 CX-anchored identities while new
persons anchor to J1. That split is expected mid-migration and matches Anselm's shape
(35,277 CX vs 6,468 Workday); it resolves as the J1 load fills in.

---

## 6. Run Scripts

### 6.1 Script Inventory

| Script | School | Target | Primary SIS | Runs today? |
|--------|--------|--------|-------------|---|
| `run_anselm_dev.sh` | St. Anselm College | `anselm_dev` | WORKDAY | Yes |
| `run_demeau_dev.sh` | DEMEAU (Demo) | `demeau_dev` | JENZABAR_ONE | Yes |
| `run_merrimack_dev.sh` | Merrimack College | `merrimack_dev` | JENZABAR_ONE | Yes |
| `run_colby_dev.sh` | Colby College | `colby_dev` | WORKDAY | No — no target |
| `run_endicott_dev.sh` | Endicott College | `endicott_dev` | WORKDAY | No — no target |
| `run_springfield_dev.sh` | Springfield College | `springfield_dev` | BANNER | No — no target |

Merrimack, Colby and Endicott were corrected on 2026-08-14 — all three previously read
`JENZABAR_CX`, which matched neither the institution's system of record nor, for Colby
and Endicott, any enabled source (`has_workday` was `false` alongside a Workday primary).

`profiles.yml` defines only `{merrimack,anselm,demeau}_{dev,test,prod}`. The other three
scripts fail immediately on a missing target and those schools have no Snowflake
databases — the scripts are provisioning scaffolding, not runnable configuration.

### 6.2 Source Systems per School

| Script | JCX | J1 | WD | Banner | PF | Slate | Canvas |
|--------|-----|----|----|--------|----|----|--------|
| run_anselm_dev.sh | ✅ | ❌ | ✅ | ❌ | ✅ | ✅ | ❌ |
| run_colby_dev.sh | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| run_demeau_dev.sh | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| run_endicott_dev.sh | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| run_merrimack_dev.sh | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | ❌ |
| run_springfield_dev.sh | ✅ | ❌ | ❌ | ✅ | ❌ | ❌ | ❌ |

### 6.3 Usage Pattern

```bash
# Standard invocation
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_<school>_dev.sh <dbt_command> [flags]

# With extra vars (merged, not replacing)
EXTRA_VARS='{"enable_row_level_security": true}' \
  bash scripts/run_merrimack_dev.sh build --select dim_student

# Multi-value --select (must be quoted)
bash scripts/run_merrimack_dev.sh build --select "seed_cpi_index mart_financial_aid_trend"
```

⚠️ **Do not pass `--vars` directly.** The scripts reject it to prevent silent variable replacement.

---

## 7. dbt Variables

### 7.1 Variables Defined in dbt_project.yml

**Academic Calendar:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `current_academic_year` | `2025-2026` | Current academic year for time-relative filters |
| `current_term_code` | `FA26` | Current term code |
| `historical_cutoff_years` | `10` | Lookback window for historical analysis |

**Source System Feature Flags:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `has_jenzabar_cx` | `false` | Enable Jenzabar CX data share |
| `has_jenzabar_one` | `false` | Enable Jenzabar One CSV deposit |
| `has_workday` | `false` | Enable Workday Student CSV deposit |
| `has_banner` | `false` | Enable Banner SIS |
| `has_powerfaids` | `false` | Enable PowerFAIDS |
| `has_slate` | `false` | Enable Slate CRM |
| `has_canvas` | `false` | Enable Canvas LMS |
| `has_jcx_deposit` | `false` | Enable JCX share + deposit UNION (DEMEAU only) |

**Security/Governance Flags:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `enable_masking_policies` | `false` | Enable column-level masking `[PLANNED]` |
| `enable_row_level_security` | `false` | Enable row access policies `[BUILT, NOT ACTIVE]` |

**School Identity (override at runtime):**

| Variable | Default | Purpose |
|----------|---------|---------|
| `school_code` | `UNKNOWN` | Institution identifier |
| `primary_sis` | `UNKNOWN` | Primary SIS system |
| `jcx_share_database` | `UNKNOWN` | JCX data share database |
| `jcx_share_schema` | `DITTEAU_ARCHIVE` | JCX share schema |
| `external_sources_database` | `DITTEAU_SHARED` | External data database |

### 7.2 Variables Referenced but NOT Defined

| Variable | Location | Status | Impact |
|----------|----------|--------|--------|
| `var('env')` | Comments in `apply_rap.sql` | **Intentionally missing** | Environment resolved via `target.database` |
| `var('j1_source_database')` | `_j1_sources.yml` | Has inline default: `DITTEAU_RAW` | Works via fallback |
| `var('j1_source_schema')` | `_j1_sources.yml` | Has inline default: `JENZABAR_ONE_RAW` | Works via fallback |

⚠️ **`var('env')` does not exist.** The original `apply_rap()` macro referenced it and would have failed if executed. Environment isolation uses `target.database` instead.

---

## 8. Macros

| Macro | Signature | Purpose |
|-------|-----------|---------|
| `apply_rap` | `(domain, key_column)` | Attaches RAPs when `enable_row_level_security=true` |
| `add_source_metadata` | `(source, table)` | Injects 5 standardized metadata columns |
| `jcx_base` | `(table_name)` | Conditional UNION for JCX share + deposit |
| `generate_surrogate_key` | `(columns)` | MD5 surrogate key wrapper |
| `safe_col` | `(relation, column, ...)` | Graceful column existence check |
| `cpi_adjust` | `(amount, year, target)` | CPI inflation adjustment |
| `cpi_adjust_joins` | `(year_col, target)` | CPI join clauses |
| `sc_num` | `(col)` | College Scorecard numeric parsing |
| `school_ref` | `(model_name)` | School-specific override resolution |
| `metadata_defaults` | `()` | Fallback metadata values |
| `metadata_column_descriptions` | `()` | Standardized column descriptions |
| `source_meta` | `(source, table)` | Resolve source meta from YAML |
| `get_retention_years` | `(school, source, table)` | Retention policy lookup |
| `generate_schema_name` | `(custom, node)` | Schema name override |

---

## 9. Packages

| Package | Version | Purpose |
|---------|---------|---------|
| dbt-labs/dbt_utils | 1.1.1 | General utilities (surrogate keys, date spine, etc.) |
| metaplane/dbt_expectations | 0.10.1 | Data quality testing |
| dbt-labs/audit_helper | 0.9.0 | Comparing relations during refactoring |
| dbt-labs/codegen | 0.12.1 | YAML scaffolding generation |

---

## 10. Seeds

### 10.1 Shared Seeds (8)

| Seed | Schema | Purpose |
|------|--------|---------|
| `dim_gender_seed` | distribute | Gender reference codes |
| `dim_state_seed` | distribute | US state codes |
| `seed_aid_fund_crosswalk` | distribute | Aid code categorization |
| `seed_cpi_index` | distribute | CPI-U annual averages |
| `seed_rbac_role_definitions` | distribute | Role-based access control |
| `seed_scorecard_opeid_crosswalk` | distribute | OPEID mapping |
| `seed_valid_role_types` | distribute | Enumerated role types |
| `seed_id_overrides` | distribute | Manual ID match escape hatch |

### 10.2 School-Specific Seeds

| School | Seeds |
|--------|-------|
| ANSELM | `seed_anselm_ipeds_peer_group`, `seed_anselm_retention_policy`, `seed_anselm_wd_program_of_study` |
| MERRIMACK | `seed_merrimack_ipeds_peer_group`, `seed_merrimack_retention_policy` |
| COLBY | `seed_colby_retention_policy` |
| ENDICOTT | `seed_endicott_retention_policy` |
| SPRINGFIELD | `seed_springfield_retention_policy` |
| DEMEAU | `seed_demeau_ipeds_peer_group`, `seed_demeau_retention_policy` |

---

## 11. Tests

### 11.1 Test Configuration

- **Store failures:** `+store_failures: true`
- **Results schema:** `dbt_test_results`
- **Source freshness:** warn at 24h, error at 48h

### 11.2 Singular Tests

| Test | Purpose |
|------|---------|
| `assert_anselm_crosswalk_coverage.sql` | Validates SIS crosswalk resolution |
| `assert_no_duplicate_source_identity.sql` | No duplicate source identity tuples |
| `assert_no_app_owner_in_role_domain_access.sql` | Streamlit owner roles absent from tier grid |
| `assert_no_unimplemented_scoped_tiers.sql` | SCOPED only for student_academic domain |

### 11.3 Generic Tests

- **not_null:** 191 assertions
- **unique:** 7 assertions

---

## 12. Governance Objects

### 12.1 Row Access Policies

| Status | Description |
|--------|-------------|
| `[BUILT, NOT ACTIVE]` | 30 RAPs defined (3 per school × 3 envs + legacy), attached to 0 tables |

Policies are gated by `enable_row_level_security`, which is `false` in all environments.

### 12.2 Masking Policies

| Status | Description |
|--------|-------------|
| `[BUILT, NOT ACTIVE]` | 7 policies in DEMEAU_DD_DEV only, attached to 0 columns |

No code reads `enable_masking_policies` at runtime.

---

## 13. Naming Conventions

### 13.1 Models

| Pattern | Example |
|---------|---------|
| Staging | `stg_{source}__{entity}.sql` |
| Intermediate | `int_{entity}.sql` |
| Dimension | `dim_{entity}.sql` |
| Fact | `fact_{entity}.sql` |
| Mart | `mart_{domain}_{entity}.sql` |
| Snapshot | `snap_{entity}_{grain}.sql` |

### 13.2 Columns

| Pattern | Example | Purpose |
|---------|---------|---------|
| `{entity}_key` | `student_key` | Surrogate key (MD5) |
| `{entity}_code` | `term_code` | Natural/business key |
| `is_current` | `is_current` | SCD Type 2 flag |
| `is_active` | `is_active` | Business active flag |
| `creation_timestamp` | `creation_timestamp` | Record creation |
| `last_modified_timestamp` | `last_modified_timestamp` | Record update |

### 13.3 YAML Files

| Pattern | Purpose |
|---------|---------|
| `_{source}_sources.yml` | Source definitions |
| `_{layer}_models.yml` | Model documentation |
| `_semantic_layer.yml` | MetricFlow definitions |
| `_metrics.yml` | Metric definitions |

---

## 14. dbt Binary

⚠️ **Always use the testenv virtual environment:**

```bash
# Correct
/Users/laurievanpelt/testenv/bin/dbt <command>

# Or prefix PATH
PATH="/Users/laurievanpelt/testenv/bin:$PATH" dbt <command>
```

`/opt/homebrew/bin/dbt` is the dbt Cloud CLI and will fail with a credentials error.
