# DEMEAU Tenant Documentation

**Purpose:** DEMEAU is a full three-environment tenant that appears in none of the eight project context documents, yet serves as the governance validation environment and demo tenant. This document establishes the authoritative record of its configuration and role.

**Date:** 2026-08-13
**Author:** LVP
**Status:** Active

---

## 1. Overview

DEMEAU is a demonstration school tenant in the Ditteau Data platform. It is:
- A **full three-environment tenant** (DEV, TEST, PROD databases)
- The **governance validation environment** for all ADR-003 work
- The **demo tenant** for client presentations and dashboard prototyping
- The only tenant with **RAPs and masking policies deployed and active** (PROD)

DEMEAU is **pseudonymized Saint Anselm data**, not synthetic data. Treat DEMEAU as real institutional data for access purposes. See Section 5 for provenance details.

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

**DEMEAU_DD_PROD (fully built, verified 2026-09-26):**

| Schema | Table Count | Notes |
|--------|-------------|-------|
| DEPOSIT | 3,538 | Raw data landing zone |
| DETERGE | 22 | Staging views + intermediate tables |
| DISTRIBUTE | 64 | Dimensions, facts, marts, seeds |
| GOVERNANCE | 8 | Tier tables, PII unmask tables |
| DBT_AUDIT | 1 | Build provenance |
| DBT_TEST_RESULTS | 1,012 | Test failure storage |

✅ DEMEAU PROD is fully built with RAPs and masking policies **attached and active**.

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
| RAPs and masking | **Active in PROD** | Not deployed |
| `has_jcx_deposit` | true | false |
| Network policy | Yes (06_network_policy.sql) | No |
| Data span | 18 years (2006-2024) | Varies |
| Students per term | ~2,200 | Varies |
| Data provenance | Pseudonymized Anselm | Real institutional |

---

## 5. Data Provenance

*Measured 2026-09-26 against DEMEAU_DD_DEV and ANSELM_DD_DEV.*

### 5.1 Summary

DEMEAU is **pseudonymized Saint Anselm data**, not synthetic data:

| Metric | Value |
|--------|-------|
| Shared `student_id` values with ANSELM | **14,151** (100% overlap in both directions) |
| First name match rate | 0% |
| Last name match rate | 0% |
| DOB match rate | 0% |
| Average DOB shift | **55.4 days** (D-32 jitter, ±10–100 days) |

### 5.2 Authorization

Anselm authorized use of this pseudonymized data including third-party demos (D-21, ratified KKM 2026-08-26).

### 5.3 Access Implications

**Treat DEMEAU as real institutional data for access purposes.** The `student_id` values are identical to Anselm's, and a ±3-month DOB shift crosses a year boundary only near one — 85.8% of students keep their real birth year. Real `student_id` plus a mostly-real birth year is still a quasi-identifier. DOB masking remains active, and the `student_id` half of the O-29 gap remains open.

### 5.4 Anonymization Details

- **Names:** Replaced with Faker-generated values (0% match)
- **DOB:** Jittered ±10–100 days per D-32, verified by N-21 assertion (`ditteau_data_infra/school_setup/platform/verify_demeau_dob_anonymisation.sql`)
- **student_id:** Preserved (required for referential integrity across dimension/fact joins)

---

## 6. Documentation Defect Record

### 6.1 Summary

An earlier two-lane description of DEMEAU data (real CX plus fully synthetic) did not match deployed reality. The verification in Section 5 confirmed DEMEAU is pseudonymized Anselm data, not synthetic.

### 6.2 Impact

Reasoning from the prior description produced a **false safety premise** in Phase 6 planning:

- Phase 6 work assumed certain DEMEAU data paths were safe for testing because they were "fully synthetic"
- This assumption informed decisions about what validation could be performed on DEMEAU vs. other schools
- The assumption was not verified against deployed configuration

### 6.3 Lesson

**Prior documentation is not evidence of deployed state.** What matters is what is currently deployed:

- `run_demeau_dev.sh` is the source of truth for DEMEAU configuration
- `SHOW DATABASES` and schema queries are the source of truth for what exists
- The 2026-09-26 verification (Section 5) established the actual provenance

This document now serves as the authoritative DEMEAU reference. The prior two-lane description should not be cited or restated.

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

*Verified 2026-09-26 in DEMEAU_DD_PROD.*

| Policy | Signature | Status |
|--------|-----------|--------|
| RAP_STUDENT_ACADEMIC | NUMBER(38,0) | **Attached and active** (dim_student, fact_student_term, fact_enrollment) |
| RAP_FINANCIAL_AID | VARCHAR | **Attached and active** (fact_aid_award) |
| RAP_ADMISSIONS | VARCHAR | **Attached and active** (dim_applicant, fact_application) |
| DISTRIBUTE_ACCESS_POLICY | — | Legacy, PROD only |

### 8.2 Masking Policies (DEMEAU Only)

*Verified 2026-09-26 in DEMEAU_DD_PROD.*

| Policy | PII Field | Status |
|--------|-----------|--------|
| MASK_NAME | NAME | **Attached and active** |
| MASK_DOB | DOB | **Attached and active** |
| MASK_EMAIL | EMAIL | Built |
| MASK_SSN | SSN | Built |
| MASK_PHONE | PHONE | Built |
| MASK_ADDRESS | ADDRESS | Built |
| MASK_FINANCIAL_AMOUNT | FINANCIAL_AMOUNT | **Attached and active** |

RAPs and masking policies are **attached and active in DEMEAU_DD_PROD**. No other school has masking policies deployed.

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
