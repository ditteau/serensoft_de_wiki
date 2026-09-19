# dbt Conventions

Standards and patterns for dbt development in the Ditteau Data platform.

---

## Architecture: Three-Layer Medallion

| Layer | Folder | Purpose |
|-------|--------|---------|
| Deposit | `models/deposit/` | Raw data landing zone — sourced via stages, shares, or ETL |
| Deterge | `models/deterge/` | Cleaned, conformed, cross-source integrated data |
| Distribute | `models/distribute/` | Analytics-ready dimensional models (star schema) |

### Deterge Sub-Layers

- **`deterge/staging/`** — Materialized as **views**. Lightweight source conformance, one model per source table.
- **`deterge/intermediate/`** — Materialized as **tables**. Cross-source joins and business logic.

### Distribute Sub-Layers

- **`distribute/dimensions/`** — SCD-aware dimension tables
- **`distribute/facts/`** — Fact tables with documented grain
- **`distribute/marts/`** — Business data marts organized by domain (enrollment, financial_aid, academic, etc.)

---

## Naming Conventions

### Models

| Type | Pattern | Example |
|------|---------|---------|
| Staging | `stg_{source}__{entity}.sql` | `stg_jcx__students.sql` |
| Intermediate | `int_{entity}.sql` | `int_students.sql` |
| Dimension | `dim_{entity}.sql` | `dim_student.sql` |
| Fact | `fact_{entity}.sql` | `fact_enrollment.sql` |
| Mart | `mart_{domain}_{metric}.sql` | `mart_enrollment_census.sql` |

### YAML Files

- Sources: `_{source}_sources.yml`
- Model docs: `_{layer}_models.yml` or `_{source}_models.yml`

### Columns

| Type | Pattern | Example |
|------|---------|---------|
| Surrogate key | `{entity}_key` | `student_key` |
| Natural/business key | `{entity}_code` | `student_code` |
| Timestamps | `creation_timestamp`, `last_modified_timestamp` | |
| SCD flags | `is_current`, `is_active` | |
| Source dates | `source_active_date`, `source_inactive_date` | |

---

## Materializations

| Layer | Materialization | Rationale |
|-------|-----------------|-----------|
| Staging | `view` | Lightweight, no storage cost, always current |
| Intermediate | `table` or `ephemeral` | Tables for frequently-joined data; ephemeral for single-use CTEs |
| Dimensions | `table` | Persisted for fast lookups |
| Facts | `table` | Persisted for analytical queries |
| Marts | `table` | Pre-aggregated for dashboard performance |

---

## Tagging Strategy

All models receive tags based on layer and source system:

```yaml
models:
  - name: stg_jcx__students
    config:
      tags: ['staging', 'jenzabar_cx']
```

Standard tags:
- Layer: `staging`, `intermediate`, `dimension`, `fact`, `mart`
- Source: `jenzabar_cx`, `jenzabar_one`, `workday`, `banner`, `slate`, `powerfaids`
- Special: `snapshot`, `scd_type_2`, `governance`

---

## Var Guards for Optional Sources

Schools have different source systems. Use var guards to conditionally enable models:

```sql
{{ config(
    enabled = var('has_slate', false)
) }}
```

In `dbt_project.yml` or run scripts, set the appropriate flags:

```yaml
vars:
  has_jenzabar_cx: true
  has_jenzabar_one: true
  has_slate: false
  has_powerfaids: false
  has_workday: false
  has_banner: false
```

**Critical:** A `has_<source>` flag is a claim, not a measurement. Always verify the deposit tables actually exist:

```sql
SELECT table_name
FROM {DB}.INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'DEPOSIT';
```

Presence is not authenticity — tables may exist but contain generated test data rather than real source data. Look at the rows.

---

## Required Metadata Columns

Every model in Deterge and Distribute layers must include five metadata columns via the `add_source_metadata()` macro:

| Column | Type | Description |
|--------|------|-------------|
| `_source_system` | VARCHAR | Source system identifier (e.g., `JENZABAR_CX`) |
| `_source_table` | VARCHAR | Source table name |
| `_loaded_at` | TIMESTAMP_NTZ | When the record was loaded |
| `_dbt_updated_at` | TIMESTAMP_NTZ | When dbt last touched the record |
| `_row_hash` | VARCHAR | MD5 hash of business columns for change detection |

Usage in a staging model:

```sql
SELECT
    student_id,
    first_name,
    last_name,
    {{ add_source_metadata('jenzabar_cx', 'id_rec') }}
FROM {{ source('jenzabar_cx', 'id_rec') }}
```

---

## Dimension Model Standards

Every dimension should include:

- `{entity}_key` — Surrogate key (MD5 via `dbt_utils.generate_surrogate_key`)
- `{entity}_code` — Natural/business key
- `source_active_date`, `source_inactive_date`
- `is_active` (boolean)
- `is_current` (boolean, for SCD Type 2 — currently always TRUE)
- `creation_timestamp`, `last_modified_timestamp`

---

## Surrogate Key Generation

Use `dbt_utils.generate_surrogate_key` for all surrogate keys:

```sql
{{ dbt_utils.generate_surrogate_key(['student_id', 'source_system']) }} AS student_key
```

This produces an MD5 hash, ensuring consistent keys across rebuilds.

---

## Primary SIS Selection

Most schools carry a Jenzabar CX archive alongside their primary SIS. The `primary_sis` variable controls which system anchors identity resolution:

| School | Primary SIS | `primary_sis` value |
|--------|-------------|---------------------|
| Saint Anselm | Workday | `WORKDAY` |
| Merrimack | Jenzabar One | `JENZABAR_ONE` |
| DEMEAU (demo) | Jenzabar One | `JENZABAR_ONE` |
| Springfield | Banner | `BANNER` |

This affects `int_ditteau_id_registry`, `dim_integration_ids`, and cross-source identity resolution.

---

## Key Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `school_code` | Institution identifier | `MERRIMACK` |
| `env` | Environment | `dev`, `test`, `prod` |
| `current_academic_year` | Current AY | `2025-2026` |
| `current_term_code` | Current term | `FA26` |
| `deposit_database` | Source database | `{school_code}_DD_{env}` |
| `external_sources_database` | Shared external data | `DITTEAU_SHARED` |
| `historical_cutoff_years` | Lookback window | `10` |

---

## Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `dbt-labs/dbt_utils` | 1.1.1 | Utilities (`generate_surrogate_key`, `date_spine`, etc.) |
| `metaplane/dbt_expectations` | 0.10.1 | Data quality testing |
| `dbt-labs/audit_helper` | 0.9.0 | Comparing relations during refactoring |
| `dbt-labs/codegen` | 0.12.1 | Generating source/model YAML scaffolding |

---

## Tests

- `+store_failures: true` — All test failures are persisted
- Test results schema: `dbt_test_results`
- Source freshness: warn at 24h, error at 48h

See [Testing Strategy](testing.md) for detailed testing patterns.

---

## Related Documentation

- [Project Structure](project-structure.md) — Directory layout and organization
- [Testing Strategy](testing.md) — Test patterns and data quality
- [Macros Reference](macros.md) — Available macros and usage
