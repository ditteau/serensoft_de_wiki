# DEMEAU Tenant Documentation

**Purpose:** DEMEAU is a full three-environment tenant that appears in none of the eight project context documents, yet serves as the governance validation environment and demo tenant. This document establishes the authoritative record of its configuration and role.

**Date:** 2026-08-13
**Author:** LVP
**Status:** Active — provenance section pending KKM-approved language

---

## 1. Overview

DEMEAU is a demonstration school tenant in the Ditteau Data platform. It is:
- A **full three-environment tenant** (DEV, TEST, PROD databases)
- The **governance validation environment** for all ADR-003 work
- The **demo tenant** for client presentations and dashboard prototyping
- The only tenant with **masking policies deployed** (DEV only)

DEMEAU is not a production school. It does not represent a real institution's data.

---

## 2. Databases and Schemas

### 2.1 School Databases

| Database | Environment | Created | Purpose |
|----------|-------------|---------|---------|
| DEMEAU_DD_DEV | Development | 2026-07-10 | Primary development, governance testing |
| DEMEAU_DD_TEST | Test/CI | 2026-07-10 | Integration testing |
| DEMEAU_DD_PROD | Production | 2026-07-10 | Production deployment (unbuilt) |

### 2.2 Data Share Database

| Database | Type | Origin Share |
|----------|------|--------------|
| DEMEAU_CX_ARCHIVE | Imported | SYAXLGH.DITTEAUEAST.DEMEAU_DB_SNOWFLAKE_SHARE_070725 |

### 2.3 Built Layers per Environment

**DEMEAU_DD_DEV (fully built):**

| Schema | Table Count | Notes |
|--------|-------------|-------|
| DEPOSIT | 3,538 | Raw data landing zone |
| DETERGE | 20 | Staging views + intermediate tables |
| DISTRIBUTE | 54 | Dimensions, facts, marts, seeds |
| GOVERNANCE | 6 | Tier tables, PII unmask tables |
| DBT_TEST_RESULTS | 944 | Test failure storage |

**DEMEAU_DD_TEST:** Similar structure (not queried).

**DEMEAU_DD_PROD (unbuilt):**

| Schema | Table Count | Notes |
|--------|-------------|-------|
| DEPOSIT | 1 | Placeholder only |
| GOVERNANCE | 7 | Tier tables (populated), PII unmask tables |
| DETERGE | 0 | **Not built** |
| DISTRIBUTE | 0 | **Not built** |

⚠️ DEMEAU PROD has no transformed data. The deterge and distribute layers have never been built in PROD.

---

## 3. Source System Configuration

From `scripts/run_demeau_dev.sh`:

```json
{
  "school_code": "DEMEAU",
  "primary_sis": "JENZABAR_ONE",
  "has_jenzabar_cx": true,
  "has_jcx_deposit": true,
  "has_jenzabar_one": true,
  "has_powerfaids": true,
  "has_workday": false,
  "has_slate": true,
  "has_canvas": false,
  "jcx_share_database": "DEMEAU_CX_ARCHIVE",
  "jcx_share_schema": "DITTEAU_ARCHIVE",
  "j1_source_database": "DEMEAU_DD_DEV",
  "j1_source_schema": "DEPOSIT",
  "jcx_deposit_database": "DEMEAU_DD_DEV",
  "external_sources_database": "DITTEAU_SHARED"
}
```

### 3.1 Source Systems Enabled

| System | Enabled | Database | Schema | Notes |
|--------|---------|----------|--------|-------|
| Jenzabar CX (share) | Yes | DEMEAU_CX_ARCHIVE | DITTEAU_ARCHIVE | Data share from DITTEAUEAST |
| Jenzabar CX (deposit) | Yes | DEMEAU_DD_DEV | DEPOSIT | Supplement rows via `jcx_base()` macro |
| Jenzabar One | Yes | DEMEAU_DD_DEV | DEPOSIT | Primary SIS |
| PowerFAIDS | Yes | DEMEAU_DD_DEV | DEPOSIT | Financial aid |
| Slate | Yes | DEMEAU_DD_DEV | DEPOSIT | Admissions CRM |
| Workday | No | — | — | Not applicable |
| Canvas | No | — | — | Not applicable |

### 3.2 The jcx_base() UNION Pattern

DEMEAU is the only tenant with `has_jcx_deposit: true`. This enables the `jcx_base()` macro to UNION rows from both the CX share and the deposit schema:

```sql
-- jcx_base('id_rec') expands to:
SELECT * FROM DEMEAU_CX_ARCHIVE.DITTEAU_ARCHIVE.id_rec
UNION ALL
SELECT * FROM DEMEAU_DD_DEV.DEPOSIT.id_rec
```

For all other schools, `jcx_base()` returns only the share rows.

---

## 4. Role in the Platform

### 4.1 Governance Validation Environment

DEMEAU serves as the validation environment for all ADR-003 governance work:

- **Row access policies** were first created and tested in DEMEAU_DD_DEV
- **Masking policies** exist only in DEMEAU_DD_DEV (7 policies)
- **Tier table population** was validated in DEMEAU before applying to other schools
- **Phase 6 COALESCE tier logic** was verified empirically on DEMEAU

All governance migrations are applied to DEMEAU first, then to MERRIMACK and ANSELM.

### 4.2 Demo Tenant

DEMEAU is used for:

- **Client demos** — showing dashboard capabilities without exposing real student data
- **Dashboard prototyping** — `streamlit/demeau_enrollment_dashboard_v2.py` and related files
- **Mart development** — new marts are built and tested against DEMEAU before other schools
- **Documentation screenshots** — DEMEAU data appears in runbook examples

### 4.3 Unique Characteristics

| Feature | DEMEAU | Other Schools |
|---------|--------|---------------|
| Masking policies | 7 deployed | 0 |
| `has_jcx_deposit` | true | false |
| Network policy | Yes (06_network_policy.sql) | No |
| Data span | 18 years (2006-2024) | Varies |
| Students per term | ~2,200 | Varies |

---

## 5. Data Provenance

⚠️ **PENDING KKM-APPROVED LANGUAGE**

This section must be written in language KKM has reviewed and approved. The placeholder below indicates what must be documented; the actual content requires KKM sign-off.

### 5.1 Stub — Awaiting KKM Language

```
[PENDING KKM LANGUAGE]

This section must document:
- The origin of data in DEMEAU_CX_ARCHIVE
- The origin of data in DEMEAU_DD_DEV.DEPOSIT
- Whether any data is derived from real institutional records
- What transformations or synthesis were applied
- The safety premises that depend on this provenance

Do not draft this section from any prior description. See Section 6 for why.
```

### 5.2 What Is Known from Deployed State

The following is observable from configuration, not from prior documentation:

1. `run_demeau_dev.sh` sets `jcx_share_database: DEMEAU_CX_ARCHIVE`
2. `stg_jcx__students` reads from that archive via `jcx_base()`
3. Therefore, `DEMEAU_DD_DEV.distribute` is derived from `DEMEAU_CX_ARCHIVE`
4. The data share origin is `SYAXLGH.DITTEAUEAST.DEMEAU_DB_SNOWFLAKE_SHARE_070725`

The provenance of the data in that share — whether synthetic, anonymized, or derived from real records — is not documented in any repository and must come from KKM.

---

## 6. Documentation Defect Record

### 6.1 The Prior Description

A two-lane account of DEMEAU data was previously in circulation:

> **Lane 1:** Terminal and CX-only — [description of first lane]
> **Lane 2:** Fully synthetic — [description of second lane]

This description implied that certain DEMEAU data paths were isolated from real institutional records.

### 6.2 Why It Was Wrong

The prior description does not match deployed reality:

1. `run_demeau_dev.sh` explicitly sets `jcx_share_database: DEMEAU_CX_ARCHIVE`
2. The `stg_jcx__*` staging models read from this archive
3. `DEMEAU_DD_DEV.distribute` tables are therefore derived from `DEMEAU_CX_ARCHIVE`
4. The share's origin (`SYAXLGH.DITTEAUEAST.DEMEAU_DB_SNOWFLAKE_SHARE_070725`) indicates it came from the DITTEAUEAST account

**The "Lane 2 fully synthetic" premise did not hold.** Data in `DEMEAU_DD_DEV.distribute` flows through the CX archive, not through a synthetic generation path.

### 6.3 Impact

Reasoning from the prior description produced a **false safety premise** in Phase 6 planning:

- Phase 6 work assumed certain DEMEAU data paths were safe for testing because they were "fully synthetic"
- This assumption informed decisions about what validation could be performed on DEMEAU vs. other schools
- The assumption was not verified against deployed configuration

### 6.4 Lesson

**Prior documentation is not evidence of deployed state.** The two-lane description may have been accurate at some point, or may have been aspirational, or may have described a different DEMEAU configuration. What matters is what is currently deployed:

- `run_demeau_dev.sh` is the source of truth for DEMEAU configuration
- `SHOW DATABASES` and schema queries are the source of truth for what exists
- The data share origin is observable; the data's ultimate provenance requires KKM confirmation

This document now serves as the authoritative DEMEAU reference. The prior description should not be cited.

---

## 7. Warehouses and Roles

### 7.1 Warehouses

| Warehouse | Size | Auto-Suspend | Resource Monitor |
|-----------|------|--------------|------------------|
| DEMEAU_TRANSFORM_DEV | Small | 60s | DEMEAU_MONITOR_TRANSFORM_DEV |
| DEMEAU_TRANSFORM_TEST | X-Small | 60s | DEMEAU_MONITOR_TRANSFORM_TEST |
| DEMEAU_TRANSFORM_PROD | Medium | 300s | DEMEAU_MONITOR_TRANSFORM_PROD |
| DEMEAU_ANALYTICS_PROD | Small | 120s | DEMEAU_MONITOR_ANALYTICS_PROD |

### 7.2 Roles

DEMEAU has the standard 21-role structure:

**Technical (12):** `DEMEAU_{TRANSFORM,WRITE,READ,DBT}_{DEV,TEST,PROD}`

**PROD-only (1):** `DEMEAU_REPORTING_PROD`

**Business Function (4):**
- `DEMEAU_REGISTRAR_ROLE`
- `DEMEAU_ADVISOR_ROLE`
- `DEMEAU_FA_ROLE`
- `DEMEAU_IR_ANALYST_ROLE`

**Governance (1):** `DEMEAU_GOVERNANCE_ADMIN_ROLE` (0 users assigned)

**Streamlit Owner (3):** `DEMEAU_STREAMLIT_OWNER_{DEV,TEST,PROD}` (0 users assigned, deliberately absent from tier tables)

### 7.3 Tier Table Population

`DEMEAU_DD_DEV.governance.role_domain_access` contains 40 rows (10 roles × 4 domains), matching the standard grid.

`DEMEAU_DD_DEV.governance.user_domain_access` contains 0 rows (no user-level overrides).

---

## 8. Governance Objects

### 8.1 Row Access Policies

| Policy | Signature | Status |
|--------|-----------|--------|
| RAP_STUDENT_ACADEMIC | NUMBER(38,0) | Built, not attached |
| RAP_FINANCIAL_AID | VARCHAR | Built, not attached |
| RAP_ADMISSIONS | VARCHAR | Built, not attached |
| DISTRIBUTE_ACCESS_POLICY | — | Legacy, PROD only |

### 8.2 Masking Policies (DEMEAU Only)

| Policy | PII Field | Status |
|--------|-----------|--------|
| MASK_NAME | NAME | Built, not attached |
| MASK_DOB | DOB | Built, not attached |
| MASK_EMAIL | EMAIL | Built, not attached |
| MASK_SSN | SSN | Built, not attached |
| MASK_PHONE | PHONE | Built, not attached |
| MASK_ADDRESS | ADDRESS | Built, not attached |
| MASK_FINANCIAL_AMOUNT | FINANCIAL_AMOUNT | Built, not attached |

These masking policies exist **only** in DEMEAU_DD_DEV. No other school has masking policies deployed.

---

## 9. Usage Notes

### 9.1 Running dbt for DEMEAU

```bash
# Standard pattern
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_demeau_dev.sh build --select dim_student

# With extra vars (merged, not replacing)
EXTRA_VARS='{"enable_row_level_security": true}' \
  bash scripts/run_demeau_dev.sh build --select dim_student

# DO NOT pass --vars directly (rejected with error)
bash scripts/run_demeau_dev.sh build --vars '...'  # ERROR
```

### 9.2 Querying DEMEAU Snowflake

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  python scripts/query_snowflake.py "SELECT * FROM DEMEAU_DD_DEV.DISTRIBUTE.DIM_STUDENT LIMIT 5"
```

The query helper defaults to `demeau_dev` target.

### 9.3 Dashboard Development

DEMEAU dashboards are in `streamlit/`:
- `demeau_enrollment_dashboard_v2.py` — 4-tab enrollment analytics

These read from `DEMEAU_DD_DEV.DISTRIBUTE` marts.
