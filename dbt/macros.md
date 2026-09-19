# Macros Reference

Project-specific macros available in `ditteau_data_transform`.

---

## Directory Structure

```
macros/
├── utils/               # Utility macros
│   ├── safe_col.sql
│   ├── school_ref.sql
│   ├── jcx_base.sql
│   ├── cpi_adjust.sql
│   ├── generate_surrogate_key.sql
│   ├── sc_num.sql
│   ├── cw_status.sql
│   └── acad_yr_to_start_year.sql
├── schema/              # Schema generation
│   └── generate_schema_name.sql
├── metadata/            # Metadata columns
│   ├── add_source_metadata.sql
│   ├── source_meta.sql
│   ├── metadata_defaults.sql
│   ├── metadata_utils.sql
│   ├── document_metadata_columns.sql
│   └── get_retention_years.sql
├── governance/          # Row access policies, masking
│   ├── apply_rap.sql
│   ├── apply_masking.sql
│   ├── apply_domain_masking.sql
│   └── effective_persona_grid.sql
└── audit/               # Logging
    └── log_dbt_results.sql
```

---

## Metadata Macros

### `add_source_metadata(source_name, table_name)`

Adds standardized metadata columns to staging models. Required on all staging models.

**Output Columns:**
- `_source_system` — Source system identifier
- `_source_table` — Source table name (uppercase)
- `_source_file` — Source file path (conditional)
- `_data_classification` — Data classification level
- `_data_owner` — Data owner identifier
- `_ingest_type` — Ingestion method
- `_dbt_loaded_at` — Processing timestamp

**Usage:**

```sql
SELECT
    id_num AS student_id,
    {{ add_source_metadata('jenzabar_cx', 'id_rec') }}
FROM {{ source('jenzabar_cx', 'id_rec') }}
```

### `source_meta(source_name, table_name)`

Retrieves metadata from source YAML definition. Called internally by `add_source_metadata()`.

### `metadata_defaults()`

Returns default values when source metadata is not defined.

### `get_retention_years()`

Returns the retention window in years for historical data queries.

---

## Utility Macros

### `safe_col(source_relation, column_name, alias, default, expression)`

Returns the column if it exists in the source, otherwise returns a default value. Essential for handling school-specific custom fields that may not exist in all source systems.

**Arguments:**
- `source_relation` — A dbt relation (`source()` or `ref()`)
- `column_name` — Source column name (case-insensitive)
- `alias` — Output column alias (defaults to column_name lowercased)
- `default` — SQL expression when column absent (default: `null`)
- `expression` — Optional SQL expression wrapping the column

**Usage:**

```sql
-- Basic usage
{{ safe_col(source('jenzabar_cx', 'profile_rec'), 'ETHNIC_CODE2', 'ethnicity_code_2') }}

-- With default value
{{ safe_col(source('jenzabar_cx', 'id_rec'), 'CUSTOM_FIELD', 'custom_field', default="'N/A'") }}

-- With expression
{{ safe_col(source('jenzabar_cx', 'gla_rec'), 'SUBFUND', 'subfund_code', expression="UPPER(TRIM(SUBFUND))") }}
```

### `school_ref(table_name)`

Returns a reference to a school-specific seed or model.

### `jcx_base()`

Common base logic for Jenzabar CX staging models.

### `cpi_adjust(amount_column, year_column, target_year)`

Adjusts dollar amounts for inflation using CPI index.

**Usage:**

```sql
{{ cpi_adjust('award_amount', 'award_year', 2024) }} AS award_amount_2024_dollars
```

### `generate_surrogate_key(field_list)`

Wrapper around `dbt_utils.generate_surrogate_key` with project-specific defaults.

### `acad_yr_to_start_year(academic_year)`

Extracts the start year from an academic year string (e.g., `'2024-2025'` → `2024`).

### `sc_num(value)`

Numeric conversion with null handling for Snowflake.

### `cw_status(status_code)`

Courseware status code translation.

---

## Governance Macros

### `apply_rap(domain, key_column)`

Post-hook macro that attaches a domain row access policy to the model when `enable_row_level_security` is true.

**Arguments:**
- `domain` — Access domain: `student_academic`, `financial_aid`, or `admissions` (default: `student_academic`)
- `key_column` — Column the policy filters on (default: `student_id`)

**Usage in dbt_project.yml:**

```yaml
models:
  ditteau_data_transform:
    distribute:
      dimensions:
        dim_student:
          +post-hook: "{{ apply_rap() }}"
      facts:
        fact_aid_award:
          +post-hook: "{{ apply_rap('financial_aid', 'student_key') }}"
```

**Important:** Post-hooks must be declared in `dbt_project.yml`, NOT in model YAML config blocks. Jinja in model YAML is evaluated at parse time before project macros load.

### `apply_masking(column_name, pii_class)`

Attaches a masking policy to a column based on PII classification.

**PII Classes:**
- `NAME` — Full name masking (`***`)
- `DOB` — Date truncation to year
- `SSN` — Full redaction
- `EMAIL` — Email masking
- `FINANCIAL_AMOUNT` — Amount redaction

**Usage:**

```yaml
models:
  ditteau_data_transform:
    distribute:
      dimensions:
        dim_student:
          +post-hook:
            - "{{ apply_rap() }}"
            - "{{ apply_masking('student_full_name', 'NAME') }}"
            - "{{ apply_masking('student_dob', 'DOB') }}"
```

### `apply_domain_masking(domain)`

Applies all masking policies for a given domain.

### `effective_persona_grid()`

Returns the effective persona-domain access grid, combining base grid with exceptions.

---

## Schema Macros

### `generate_schema_name(custom_schema_name, node)`

Overrides dbt's default schema name generation. Returns the custom schema name directly (no concatenation with target schema).

This ensures models go to the intended schema:
- Staging models → `deterge`
- Dimension/fact models → `distribute`
- Governance objects → `governance`

---

## Audit Macros

### `log_dbt_results()`

On-run-end hook that logs build results to the governance schema. Captures model execution times, row counts, and success/failure status.

---

## Package Macros

In addition to project macros, these package macros are frequently used:

### dbt_utils

| Macro | Purpose |
|-------|---------|
| `generate_surrogate_key()` | MD5 hash key from columns |
| `date_spine()` | Generate date dimension |
| `star()` | Select all columns from relation |
| `union_relations()` | Union multiple relations |
| `pivot()` | Pivot rows to columns |

### dbt_expectations

| Macro | Purpose |
|-------|---------|
| `expect_table_row_count_to_be_between()` | Row count validation |
| `expect_column_values_to_be_between()` | Range validation |
| `expect_column_values_to_match_regex()` | Pattern validation |
| `expect_column_values_to_be_unique()` | Uniqueness check |

---

## Related Documentation

- [Conventions](conventions.md) — Model standards
- [Testing Strategy](testing.md) — Test patterns
- [Row Access Policies](../runbooks/row-access-policies.md) — RAP implementation
