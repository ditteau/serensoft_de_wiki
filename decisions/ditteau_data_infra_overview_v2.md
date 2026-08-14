# ditteau_data_infra — Project Overview

**Regenerated:** 2026-08-13 from deployed state
**Source:** Repository scan + Snowflake queries
**Status:** Replaces prior `ditteau_data_infra_overview.md`

---

## 1. Project Summary

`ditteau_data_infra` is the infrastructure provisioning and data ingestion repository for the Ditteau Data platform. It contains:

- **School setup generator** — Produces SQL scripts for new school provisioning
- **Governance migrations** — Post-provisioning updates for ADR-003 phases
- **Deposit loader** — CLI tool for CSV ingestion into the deposit layer
- **Platform setup** — Account-level roles and tag definitions

---

## 2. Repository Structure

```
ditteau_data_infra/
├── school_setup/
│   ├── generate_school.py       # Main generator script
│   ├── run_migration.py         # Migration runner
│   ├── README.md                # Setup instructions
│   ├── platform/                # Account-level setup
│   │   ├── 00_platform_roles.sql
│   │   └── governance_platform.sql
│   ├── migrations/              # Post-provisioning updates (8 files)
│   ├── anselm/                  # School SQL snapshots
│   ├── demeau/
│   └── merrimack/
├── deposit_loader/
│   ├── deposit_loader.py        # Main CLI tool
│   ├── source_registry.yml      # Table definitions per source system
│   ├── generate_registry_entries.py
│   ├── merge_registry.py
│   └── test_registry.py
└── SCHOOL_INVENTORY.csv         # Deployed school manifest
```

---

## 3. School Setup Generator

### 3.1 Usage

```bash
python generate_school.py \
    --name "Holy Cross" \
    --code HOLYCROSS \
    --domain holycross.edu \
    [--credit-limit 100] \
    [--retention-days 1] \
    [--alert-email alerts@ditteau.com] \
    [--ip-allowlist "192.168.1.0/24,10.0.0.0/8"]
```

### 3.2 Required Arguments

| Argument | Description | Example |
|----------|-------------|---------|
| `--name` | Full school name (used in comments) | "Holy Cross" |
| `--code` | Uppercase prefix, max 20 chars | HOLYCROSS |
| `--domain` | Email domain for service accounts | holycross.edu |

### 3.3 Optional Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--credit-limit` | 100 | Monthly credit budget |
| `--retention-days` | 1 | PROD Time Travel days (0-90) |
| `--alert-email` | — | Resource monitor notifications |
| `--ip-allowlist` | — | CIDRs for network policy (generates 06) |

### 3.4 Output Files

Generated to `./output/{code}/`:

| File | Contents | Required Role |
|------|----------|---------------|
| `01_databases_warehouses.sql` | 3 databases, 4 schemas each, 4 warehouses | SYSADMIN |
| `02_rbac.sql` | 21 roles, grants, role hierarchy | USERADMIN / SECURITYADMIN |
| `03_users.sql` | 3 dbt service accounts | USERADMIN |
| `04_resource_monitors.sql` | 4 monitors (one per warehouse) | ACCOUNTADMIN |
| `05_governance.sql` | Tags, tier tables, RAP function, role mapping | ACCOUNTADMIN / SYSADMIN |
| `06_network_policy.sql` | IP allowlist (only if `--ip-allowlist`) | ACCOUNTADMIN |
| `07_internal_stages.sql` | File formats, stage, load history table | SYSADMIN |

---

## 4. Numbered SQL Files — What Each Creates

### 4.1 `01_databases_warehouses.sql`

**Databases (3):**
- `{CODE}_DD_DEV` — Development
- `{CODE}_DD_TEST` — Test/CI
- `{CODE}_DD_PROD` — Production

**Schemas per database (4):**
- `deposit` — Raw data landing
- `deterge` — Cleaned/integrated
- `distribute` — Analytics-ready
- `governance` — Tier tables, policies

**Warehouses (4):**

| Warehouse | Size | Auto-Suspend | Purpose |
|-----------|------|--------------|---------|
| `{CODE}_TRANSFORM_DEV` | Small | 60s | dbt development |
| `{CODE}_TRANSFORM_TEST` | X-Small | 60s | CI/CD pipelines |
| `{CODE}_TRANSFORM_PROD` | Medium | 300s | Scheduled dbt |
| `{CODE}_ANALYTICS_PROD` | Small | 120s | BI tools, analysts |

⚠️ **ANALYTICS warehouse is PROD-only.** DEV and TEST have no analytics warehouse.

### 4.2 `02_rbac.sql`

**Technical Roles (12):** 4 per environment × 3 environments

| Role Pattern | Purpose |
|--------------|---------|
| `{CODE}_TRANSFORM_{ENV}` | Read deposit, write deterge/distribute |
| `{CODE}_WRITE_{ENV}` | Write deposit only |
| `{CODE}_READ_{ENV}` | Read-only all layers |
| `{CODE}_DBT_{ENV}` | dbt service account role |

**PROD-Only Role (1):**

| Role | Purpose |
|------|---------|
| `{CODE}_REPORTING_PROD` | Read distribute layer (BI tools) |

**Business Function Roles (4, PROD only):**

| Role | Domain Access |
|------|---------------|
| `{CODE}_REGISTRAR_ROLE` | FULL student_academic, FULL admissions |
| `{CODE}_ADVISOR_ROLE` | SCOPED student_academic |
| `{CODE}_FA_ROLE` | FULL financial_aid, FULL student_academic |
| `{CODE}_IR_ANALYST_ROLE` | AGGREGATED student_academic, AGGREGATED financial_aid |

⚠️ **Business roles have no DEV/TEST compute.** They exist only in PROD and have warehouse grants only to `{CODE}_ANALYTICS_PROD`.

**Governance Role (1, PROD only):**

| Role | Purpose |
|------|---------|
| `{CODE}_GOVERNANCE_ADMIN_ROLE` | Write to tier tables (unassigned by default) |

**Streamlit Owner Roles (3):**

| Role | Purpose |
|------|---------|
| `{CODE}_STREAMLIT_OWNER_{ENV}` | App execution context (deliberately absent from tier grid) |

### 4.3 `03_users.sql`

**Service Accounts (3):**

| User | Default Role | Purpose |
|------|--------------|---------|
| `svc_{code}_dbt_dev` | `{CODE}_DBT_DEV` | dbt development |
| `svc_{code}_dbt_test` | `{CODE}_DBT_TEST` | CI/CD |
| `svc_{code}_dbt_prod` | `{CODE}_DBT_PROD` | Scheduled production |

Temporary passwords set; recommend key-pair auth for PROD.

### 4.4 `04_resource_monitors.sql`

**Monitors (4):** One per warehouse

| Monitor | Credit Distribution | Thresholds |
|---------|---------------------|------------|
| `{CODE}_MONITOR_TRANSFORM_DEV` | 15% | 75%→90%→100% |
| `{CODE}_MONITOR_TRANSFORM_TEST` | 10% | 75%→90%→100% |
| `{CODE}_MONITOR_TRANSFORM_PROD` | 50% | 75%→90%→100% |
| `{CODE}_MONITOR_ANALYTICS_PROD` | 25% | 75%→90%→100% |

### 4.5 `05_governance.sql`

**Tags (3):**
- `school` — School identifier
- `data_classification` — FERPA_PROTECTED | INTERNAL | PUBLIC
- `cost_center` — Billing attribution

**Tier Tables (6):**
- `role_domain_access` — Role → (domain, tier) mapping
- `user_domain_access` — User → (domain, tier) override
- `advisor_student_map` — Username → student assignments for SCOPED
- `advisor_username_crosswalk` — Advisor ID → Snowflake username
- `role_pii_unmask` — Role → PII unmask grants
- `user_pii_unmask` — User → PII unmask grants

**Row Access Policy:**
- `rap_student_academic` — NUMBER(38,0) signature, FULL/SCOPED/AGGREGATED/NONE tiers

**Role Mapping:**
- Populates `role_domain_access` with initial 40-row grid (10 roles × 4 domains)

### 4.6 `06_network_policy.sql` (Optional)

Only generated if `--ip-allowlist` provided:
- Creates `{CODE}_NETWORK_POLICY` with allowed IP ranges
- Applies to dbt service accounts

### 4.7 `07_internal_stages.sql`

**File Formats (3):**
- `csv_standard` — Comma-delimited
- `csv_pipe_delimited` — Jenzabar format
- `csv_tab_delimited` — Database exports

**Stage:**
- `INGEST_STAGE` — Internal, folder-organized by source system

**Load Tracking:**
- `_load_history` table — Tracks all loads
- `_staged_files` view — Lists files in stage

---

## 5. Roles per School — Actual Counts

Per Snowflake query (`SHOW ROLES`), each deployed school has **21 roles**:

| Category | Roles | Count |
|----------|-------|-------|
| Technical (per env) | TRANSFORM, WRITE, READ, DBT × 3 envs | 12 |
| PROD Reporting | REPORTING_PROD | 1 |
| Business Function | REGISTRAR, ADVISOR, FA, IR_ANALYST | 4 |
| Governance Admin | GOVERNANCE_ADMIN_ROLE | 1 |
| Streamlit Owner | STREAMLIT_OWNER × 3 envs | 3 |
| **Total** | | **21** |

**Platform Roles (account-level):**
- `DITTEAU_ADMIN` — Full access, all schools
- `DITTEAU_ENGINEER` — Dev/Test full, Prod read-only
- `DQ_ADMIN` — Data Quality framework

---

## 6. Naming Conventions

### 6.1 Databases

| Pattern | Example |
|---------|---------|
| `{CODE}_DD_{ENV}` | `MERRIMACK_DD_PROD` |
| `{CODE}_CX_ARCHIVE` | `MERRIMACK_CX_ARCHIVE` |

### 6.2 Warehouses

| Pattern | Example |
|---------|---------|
| `{CODE}_TRANSFORM_{ENV}` | `MERRIMACK_TRANSFORM_DEV` |
| `{CODE}_ANALYTICS_PROD` | `MERRIMACK_ANALYTICS_PROD` |

### 6.3 Roles

| Pattern | Example |
|---------|---------|
| Technical | `{CODE}_{TYPE}_{ENV}` | `MERRIMACK_DBT_PROD` |
| Business | `{CODE}_{FUNCTION}_ROLE` | `MERRIMACK_REGISTRAR_ROLE` |
| Streamlit | `{CODE}_STREAMLIT_OWNER_{ENV}` | `MERRIMACK_STREAMLIT_OWNER_DEV` |

### 6.4 Users

| Pattern | Example |
|---------|---------|
| Service account | `svc_{code}_dbt_{env}` | `svc_merrimack_dbt_prod` |

### 6.5 Resource Monitors

| Pattern | Example |
|---------|---------|
| `{CODE}_MONITOR_{PURPOSE}_{ENV}` | `MERRIMACK_MONITOR_TRANSFORM_PROD` |

---

## 7. Deposit Loader CLI

### 7.1 Location

`deposit_loader/deposit_loader.py`

### 7.2 Global Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--school CODE` | — | School code (required unless `--database`) |
| `--env ENV` | DEV | DEV, TEST, or PROD |
| `--connection NAME` | dd_prod | config.toml connection |
| `--database DB` | `{SCHOOL}_DD_{ENV}` | Override target database |
| `--schema SCHEMA` | DEPOSIT | Override target schema |
| `--stage STAGE` | INGEST_STAGE | Override stage name |
| `--role ROLE` | `{SCHOOL}_WRITE_{ENV}` | Override Snowflake role |
| `--warehouse WH` | `{SCHOOL}_TRANSFORM_{ENV}` | Override warehouse |

### 7.3 Commands

| Command | Arguments | Description |
|---------|-----------|-------------|
| `create-tables` | `SOURCE [--table KEY]` | CREATE TABLE IF NOT EXISTS for registry tables |
| `list-tables` | `SOURCE` | Compare database vs registry |
| `validate` | `SOURCE TABLE FILE` | Check CSV headers without loading |
| `preview` | `SOURCE TABLE FILE` | Show first 10 rows |
| `load` | `SOURCE TABLE FILE [--truncate] [--delete-where "PRED"] [--keep-staged]` | Load single CSV |
| `load-all` | `SOURCE DIR [--keep-staged]` | Load all CSVs in directory |
| `status` | `[--last N]` | Show recent load history |
| `staged` | — | Show files in stage |

### 7.4 Load Modes

| Mode | Flag | Use Case |
|------|------|----------|
| Append | (default) | Incremental adds |
| Replace | `--truncate` | Single-year snapshots |
| Surgical | `--delete-where "PRED"` | Update specific rows |

### 7.5 Source Systems in Registry

| Key | System | Tables | Target |
|-----|--------|--------|--------|
| `jenzabar_one` | Jenzabar One | 2,679 | `{SCHOOL}_DD_{ENV}.DEPOSIT` |
| `powerfaids` | PowerFAIDS | 544 | `{SCHOOL}_DD_{ENV}.DEPOSIT` |
| `slate` | Slate CRM | 311 | `{SCHOOL}_DD_{ENV}.DEPOSIT` |
| `workday_student` | Workday | 2 | `{SCHOOL}_DD_{ENV}.DEPOSIT` |
| `ipeds` | IPEDS | 3 | `DITTEAU_SHARED.IPEDS` |
| `college_scorecard` | Scorecard | 1 | `DITTEAU_SHARED.COLLEGE_SCORECARD` |

**Note:** Jenzabar CX arrives via Snowflake org share, not deposit_loader.

### 7.6 Usage Examples

```bash
# Create tables
python deposit_loader.py --school MERRIMACK create-tables powerfaids

# Validate before loading
python deposit_loader.py --school MERRIMACK validate powerfaids awards ~/awards.csv

# Load (append mode)
python deposit_loader.py --school MERRIMACK load powerfaids awards ~/awards.csv

# Load to PROD
python deposit_loader.py --school MERRIMACK --env PROD load powerfaids awards awards.csv

# Replace entire table
python deposit_loader.py --school MERRIMACK load powerfaids awards awards.csv --truncate

# Surgical delete + load
python deposit_loader.py --school MERRIMACK load powerfaids awards awards_2024.csv \
    --delete-where "academic_year = 2024"

# Load IPEDS to shared database
python deposit_loader.py \
    --database DITTEAU_SHARED --schema IPEDS \
    --role DITTEAU_WRITE --warehouse PLATFORM_WH \
    load ipeds ipeds_fall_enrollment EF2023A.csv

# Check load history
python deposit_loader.py --school MERRIMACK status --last 20
```

---

## 8. Migrations

Post-provisioning updates in `school_setup/migrations/`:

| Migration | Date | Purpose |
|-----------|------|---------|
| `add_business_roles.sql` | 2026-08-10 | Add 4 business function roles per school |
| `add_governance_phase3_tables.sql` | 2026-08-10 | Create tier tables (role_domain_access, etc.) |
| `add_governance_phase4_rap.sql` | 2026-08-10 | Create rap_student_academic function |
| `add_governance_phase4b_retype_student_id.sql` | 2026-08-10 | Fix signature to NUMBER(38,0) |
| `add_governance_phase6_user_keyed_tiers.sql` | 2026-08-13 | Add user_domain_access table |
| `add_governance_phase6b_sibling_domain_policies.sql` | 2026-08-13 | Add rap_financial_aid, rap_admissions |
| `add_governance_phase6c_fa_role_tier.sql` | 2026-08-13 | Populate FA_ROLE tier rows |
| `add_governance_phase6d_pii_unmask.sql` | 2026-08-13 | Add PII unmask tables |

### 8.1 Running Migrations

```bash
/Users/laurievanpelt/testenv/bin/python run_migration.py migrations/add_business_roles.sql

# Options
--dry-run          # Show statements without executing
--continue-on-error # Collect errors instead of stopping
--target <name>    # Use different dbt profile target
```

---

## 9. Environment Asymmetries

| Feature | DEV | TEST | PROD |
|---------|-----|------|------|
| **Deterge/Distribute layers** | ✅ Built | (varies) | ❌ Empty |
| **ANALYTICS warehouse** | ❌ None | ❌ None | ✅ Exists |
| **REPORTING role** | ❌ None | ❌ None | ✅ With grants |
| **Future grants on distribute** | ❌ None | ❌ None | ✅ Configured |
| **Business function roles** | ❌ No compute | ❌ No compute | ✅ Has warehouse |
| **`refresh_advisor_student_map()`** | ❌ Not found | ❌ Not found | ? |

⚠️ **All PROD environments have no deterge/distribute tables built.** Governance schemas exist, but no transformed data.

---

## 10. Built Layers per Environment

### 10.1 DEV Environments

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE |
|----------|---------|---------|------------|------------|
| DEMEAU_DD_DEV | 3,538 | 20 | 54 | 6 |
| MERRIMACK_DD_DEV | 3,535 | 21 | 51 | 6 |
| ANSELM_DD_DEV | 3 | 21 | 50 | 6 |

### 10.2 PROD Environments

| Database | DEPOSIT | DETERGE | DISTRIBUTE | GOVERNANCE |
|----------|---------|---------|------------|------------|
| DEMEAU_DD_PROD | 1 | **0** | **0** | 7 |
| MERRIMACK_DD_PROD | 1 | **0** | **0** | 7 |
| ANSELM_DD_PROD | 1 | **0** | **0** | 7 |

### 10.3 Schools Without Snowflake Infrastructure

| School | Run Script | Snowflake Databases | Status |
|--------|------------|---------------------|--------|
| COLBY | ✅ Exists | ❌ None | Scripts only |
| ENDICOTT | ✅ Exists | ❌ None | Scripts only |
| SPRINGFIELD | ✅ Exists | ❌ None | Scripts only |

---

## 11. Platform Setup

### 11.1 Platform Roles (`platform/00_platform_roles.sql`)

| Role | Purpose | Assigned Users |
|------|---------|----------------|
| `DITTEAU_ADMIN` | Full access, all schools | 4 |
| `DITTEAU_ENGINEER` | Dev/Test full, Prod read-only | 0 |

### 11.2 Platform Tags (`platform/governance_platform.sql`)

Located in `DITTEAU_PLATFORM.GOVERNANCE`:

| Tag | Allowed Values |
|-----|----------------|
| `SENSITIVITY_LEVEL` | PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED |
| `COMPLIANCE_TYPE` | FERPA, GDPR, CCPA, HIPAA, PCI, NONE |
| `DATA_DOMAIN` | STUDENT, FINANCIAL_AID, FINANCE, ACADEMIC, ADMISSIONS, HR, ADVANCEMENT, OPERATIONAL |
| `CONTAINS_PII` | TRUE, FALSE |
| `DATA_OWNER` | (free-text) |

---

## 12. School Inventory

From `SCHOOL_INVENTORY.csv` and Snowflake queries:

| Code | Name | Primary SIS Type | Databases | Scripts | Status |
|------|------|----------|-----------|---------|--------|
| ANSELM | Saint Anselm College | Workday | ✅ DEV/TEST/PROD | 01-05, 07 | Active |
| DEMEAU | DEMEAU (Demo) | Jenzabar One | ✅ DEV/TEST/PROD | 01-07 | Active |
| MERRIMACK | Merrimack College | Jenzabar One | ✅ DEV/TEST/PROD | 01-05, 07 | Active |
| COLBY | Colby College | Workday | ❌ None | — | Not provisioned |
| ENDICOTT | Endicott College | Workday | ❌ None | — | Not provisioned |
| SPRINGFIELD | Springfield College | Banner | ❌ None | — | Not provisioned |

---

## 13. Source Registry TODOs

From `source_registry.yml` comments and code:

| Item | Status | Notes |
|------|--------|-------|
| Banner source tables | Stub only | 5 tables defined, not production-tested |
| Canvas source tables | Not started | No tables in registry |
| Workday column validation | Partial | 2 tables (79 cols student, 92 cols acad) |
| Jenzabar CX via deposit | DEMEAU only | Other schools use org share |

---

## 14. Deleted Scripts

The following scripts referenced in prior documentation **no longer exist**:

| Script | Status | Notes |
|--------|--------|-------|
| `generate_governance.py` | Deleted | Functionality merged into `05_governance.sql` template |

---

## 15. Execution Order

New school provisioning follows this sequence:

```
1. 00_platform_roles.sql      (ONCE per account)
2. governance_platform.sql    (ONCE per account)
3. 01_databases_warehouses.sql
4. 02_rbac.sql
5. 03_users.sql
6. 04_resource_monitors.sql
7. 05_governance.sql
8. 06_network_policy.sql      (optional)
9. 07_internal_stages.sql
10. Migrations (as needed)
```

All scripts are idempotent (CREATE ... IF NOT EXISTS, CREATE OR REPLACE).
