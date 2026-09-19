# Data Flow

End-to-end data flow through the Ditteau platform's three-layer medallion architecture.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SOURCE SYSTEMS                                  │
│  Jenzabar CX │ Jenzabar One │ Workday │ Banner │ Slate │ PowerFAIDS │ IPEDS │
└──────────────┬──────────────┬─────────┬────────┬───────┬────────────┬───────┘
               │              │         │        │       │            │
               ▼              ▼         ▼        ▼       ▼            ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DEPOSIT (Bronze)                                    │
│  Raw source data: no transformations, schema matches source                  │
│  Ingest: Snowflake Share (JCX) │ deposit_loader CSV (others)                │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DETERGE (Silver)                                    │
│  ┌─────────────────────────┐    ┌───────────────────────────────────────┐   │
│  │       STAGING           │    │          INTERMEDIATE                  │   │
│  │  • 1:1 with source      │───▶│  • Cross-source joins                 │   │
│  │  • snake_case columns   │    │  • Identity resolution                │   │
│  │  • Type casting         │    │  • Business logic                     │   │
│  │  • Metadata columns     │    │  • int_ditteau_id_registry            │   │
│  │  • Materialized: VIEW   │    │  • Materialized: TABLE                │   │
│  └─────────────────────────┘    └───────────────────────────────────────┘   │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DISTRIBUTE (Gold)                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────────────┐ │
│  │  DIMENSIONS  │  │    FACTS     │  │             MARTS                  │ │
│  │  dim_student │  │ fact_enroll  │  │  mart_enrollment_census            │ │
│  │  dim_term    │  │ fact_app     │  │  mart_admissions_funnel            │ │
│  │  dim_program │  │ fact_aid     │  │  mart_financial_aid_trend          │ │
│  │  dim_date    │  │ fact_stu_trm │  │  mart_retention_cohort             │ │
│  └──────────────┘  └──────────────┘  └────────────────────────────────────┘ │
│                                                                              │
│  Row Access Policies │ Masking Policies │ Governance Enforcement             │
└──────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            CONSUMERS                                         │
│     Streamlit Dashboards │ Snowsight │ BI Tools │ Ad-hoc Queries            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Layer Details

### Deposit (Bronze)

**Purpose:** Raw data landing zone with no transformations.

**Characteristics:**
- Schema matches source system exactly
- No business logic or cleaning
- Append-only (with optional truncate/delete-where for full refreshes)
- Metadata columns: `_source_system`, `_source_file`, `_loaded_at`, `_load_id`

**Ingest Methods:**

| Source | Method | Frequency |
|--------|--------|-----------|
| Jenzabar CX | Snowflake Data Share | Real-time |
| Jenzabar One | deposit_loader CSV | Daily/on-demand |
| Workday Student | deposit_loader CSV | Daily/on-demand |
| Banner | deposit_loader CSV | Daily/on-demand |
| Slate | deposit_loader CSV | Daily/on-demand |
| PowerFAIDS | deposit_loader CSV | Daily/on-demand |
| IPEDS | deposit_loader CSV | Annual |

### Deterge (Silver)

**Purpose:** Clean, conform, and integrate data from multiple sources.

#### Staging Sub-Layer

- One model per source table
- Materialized as **views** (no storage cost, always current)
- Responsibilities:
  - Rename columns to snake_case
  - Cast to appropriate types
  - Add required metadata columns
  - NO business logic

#### Intermediate Sub-Layer

- Cross-source integration
- Materialized as **tables** (persisted for performance)
- Key models:
  - `int_ditteau_id_registry` — Master identity resolution across systems
  - `int_student_race` — Race/ethnicity consolidation
  - `int_program_crosswalk` — Program code mapping

### Distribute (Gold)

**Purpose:** Analytics-ready dimensional models with governance enforcement.

#### Dimensions

Star schema dimension tables with:
- Surrogate keys (MD5 hash)
- Natural business keys
- SCD Type 2 support (`is_current`, `source_active_date`, `source_inactive_date`)
- Descriptive attributes

#### Facts

Transactional fact tables with:
- Foreign keys to dimensions
- Measured values (credit hours, amounts, counts)
- Transaction dates

#### Marts

Pre-aggregated business views:
- Optimized for specific use cases
- May include calculated metrics
- Dashboard-ready

---

## Identity Resolution

The `int_ditteau_id_registry` model resolves student identities across source systems:

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  JCX ID     │   │  J1 ID      │   │  Workday ID │
│  12345      │   │  STU-12345  │   │  WD-12345   │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │
       └────────────────┬┴─────────────────┘
                        ▼
              ┌─────────────────┐
              │  ditteau_id     │
              │  DT-00012345    │
              └─────────────────┘
```

The `primary_sis` variable determines which source system anchors identity:

| School | Primary SIS | Anchors To |
|--------|-------------|------------|
| Merrimack | Jenzabar One | J1 student IDs |
| Saint Anselm | Workday | Workday IDs |
| Springfield | Banner | Banner IDs |

---

## Governance Enforcement

At the Distribute layer, governance controls are applied:

### Row Access Policies

Filter rows based on user/role entitlements:

| Policy | Protected Objects |
|--------|-------------------|
| `rap_student_academic` | `dim_student`, `fact_enrollment`, `fact_student_term` |
| `rap_admissions` | `dim_applicant`, `fact_application` |
| `rap_financial_aid` | `fact_aid_award` |

### Masking Policies

Redact or anonymize sensitive columns:

| Policy | Effect | Applied To |
|--------|--------|------------|
| `mask_name` | Returns `***` | `student_full_name` |
| `mask_dob` | Truncates to year | `student_dob` |
| `mask_ssn` | Full redaction | `ssn_last4` |
| `mask_email` | Domain only | `student_email` |
| `mask_amount` | Returns `NULL` | Financial amounts |

---

## Data Refresh Patterns

### Real-Time (Jenzabar CX)

Data Share provides near-real-time access to source data. Staging views reflect current state automatically.

### Daily Batch (Other Sources)

```
1. Source system exports CSV to SFTP/staging area
2. deposit_loader validates and loads to DEPOSIT schema
3. dbt build refreshes DETERGE and DISTRIBUTE
4. Dashboards reflect updated data
```

### Annual (IPEDS)

IPEDS data loads once per year after NCES releases final files:

```bash
python deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  load ipeds ipeds_fall_enrollment ef2024.csv \
  --delete-where "survey_year = 2024"
```

---

## Dependency Chain

```
DEPOSIT sources
    ↓
stg_* models (views)
    ↓
int_* models (tables)
    ↓
dim_* / fact_* models (tables)
    ↓
mart_* models (tables)
```

Build with dependencies:

```bash
# Build a mart and all upstream dependencies
dbt build --select +mart_enrollment_census
```

---

## Related Documentation

- [Snowflake Environment](snowflake-environment.md) — Infrastructure details
- [dbt Conventions](../dbt/conventions.md) — Model standards
- [Source Refresh](../runbooks/source-refresh.md) — Data loading procedures
