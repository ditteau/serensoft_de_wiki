# Architecture Overview

High-level view of the Ditteau Data platform architecture.

---

## Platform Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SOURCE SYSTEMS                                     │
│   Jenzabar CX │ Jenzabar One │ Workday │ Banner │ Slate │ PowerFAIDS        │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INGESTION LAYER                                    │
│         Snowflake Data Share (JCX) │ deposit_loader CSV (others)            │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                             SNOWFLAKE                                        │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌─────────────┐   │
│  │   DEPOSIT    │   │   DETERGE    │   │  DISTRIBUTE  │   │ GOVERNANCE  │   │
│  │   (Bronze)   │ → │   (Silver)   │ → │    (Gold)    │   │  (Control)  │   │
│  │              │   │  staging +   │   │  dims/facts  │   │  RAPs, masks│   │
│  │  Raw source  │   │ intermediate │   │    marts     │   │  entitlements│  │
│  └──────────────┘   └──────────────┘   └──────────────┘   └─────────────┘   │
│                                                                              │
│                    Row Access Policies │ Masking Policies                    │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CONSUMPTION                                        │
│           Streamlit Dashboards │ Snowsight │ BI Tools │ APIs                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Principles

### 1. One Database Set per School

Multi-tenant isolation at the database level:
- `{CODE}_DD_DEV` — Development
- `{CODE}_DD_TEST` — Testing/UAT
- `{CODE}_DD_PROD` — Production

Each database has four schemas: `deposit`, `deterge`, `distribute`, `governance`.

### 2. Three-Layer Medallion Architecture

| Layer | Schema | Purpose | Materialization |
|-------|--------|---------|-----------------|
| Deposit | `deposit` | Raw source data, no transformation | Tables (loaded) |
| Deterge | `deterge` | Cleaned, conformed, integrated | Views (staging), Tables (intermediate) |
| Distribute | `distribute` | Analytics-ready dimensional models | Tables |

### 3. dbt for All Transformations

- No stored procedures
- No manual SQL transformations
- All logic version-controlled in `ditteau_data_transform`
- Reproducible builds across environments

### 4. Source-Agnostic Marts

Downstream consumers query the Distribute layer without knowing the source SIS:
- `dim_student` works regardless of Jenzabar, Workday, or Banner
- `fact_enrollment` unified across all source systems
- Identity resolution handled in Deterge layer

### 5. Governance by Default

- Row access policies on sensitive tables
- Masking policies on PII columns
- Entitlement-driven access (not role-based SELECT grants)
- FERPA compliance built into architecture

---

## Multi-Tenant Design

Each school receives complete isolation via automated provisioning:

| Component | Per School | Pattern |
|-----------|------------|---------|
| Databases | 3 | `{CODE}_DD_{ENV}` |
| Schemas | 12 | 4 per database |
| Warehouses | 4 | Transform ×3, Analytics ×1 |
| Roles | 13 | Environment-scoped |
| Service accounts | 3 | `svc_{code}_dbt_{env}` |

See [Snowflake Environment](snowflake-environment.md) for details.

---

## Data Flow

```
Source Systems
      │
      ▼
   DEPOSIT ─────────────────────────────────────
      │         Raw tables, no transformation
      ▼
   DETERGE ─────────────────────────────────────
      │         Staging (views): rename, cast, metadata
      │         Intermediate (tables): joins, identity resolution
      ▼
  DISTRIBUTE ───────────────────────────────────
      │         Dimensions, facts, marts
      │         Row access policies applied here
      ▼
  Dashboards / Reports / APIs
```

See [Data Flow](data-flow.md) for detailed documentation.

---

## Technology Stack

| Component | Technology |
|-----------|------------|
| Data Warehouse | Snowflake |
| Transformation | dbt Core |
| Ingestion | deposit_loader (CSV), Snowflake Data Share (JCX) |
| Dashboards | Streamlit in Snowflake |
| Version Control | GitHub (`ditteau` organization) |
| Schema Registry | YAML + JSON Schema validation |

---

## Active Schools

| School | Primary SIS | Status |
|--------|-------------|--------|
| Merrimack College | Jenzabar One | Production |
| Saint Anselm College | Workday Student | Production |
| Springfield College | Banner | Onboarding |
| Endicott College | Workday Student | Onboarding |
| DEMEAU | Jenzabar One | Demo environment |

---

## Related Documentation

- [Snowflake Environment](snowflake-environment.md) — Infrastructure details
- [Data Flow](data-flow.md) — Medallion architecture
- [dbt Conventions](../dbt/conventions.md) — Transformation standards
- [Roles & Permissions](../snowflake/roles-permissions.md) — RBAC model
