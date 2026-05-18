# dbt Conventions

Standard patterns and conventions for dbt development on the Ditteau platform.

---

## Project Structure

```
ditteau_dbt/
├── models/
│   ├── staging/           # 1:1 with source tables
│   │   ├── jenzabar_cx/
│   │   ├── workday/
│   │   └── banner/
│   ├── intermediate/      # Cross-source joins, complex logic
│   └── marts/             # Business-ready tables
│       ├── core/          # Student, course, enrollment
│       ├── finance/       # Financial aid, billing
│       └── analytics/     # Aggregates, metrics
├── macros/
├── tests/
└── seeds/
```

---

## Naming Conventions

| Layer | Prefix | Example |
|---|---|---|
| Staging | `stg_` | `stg_jcx__student` |
| Intermediate | `int_` | `int_enrollment_spine` |
| Marts | `dim_`, `fct_`, `agg_` | `dim_student`, `fct_enrollment` |

### Source naming pattern

```
stg_<source>__<entity>
```

- `source`: Short code for the system (e.g., `jcx`, `wd`, `ban`)
- `entity`: Table/object name in snake_case

---

## Materializations

| Layer | Materialization | Notes |
|---|---|---|
| Staging | `view` | Lightweight; always current |
| Intermediate | `ephemeral` or `view` | Depends on reuse |
| Marts | `table` | Performance for consumers |

---

## Required Metadata Columns

All staging models must include these 5 columns via the `add_source_metadata()` macro:

| Column | Description |
|---|---|
| `_source_system` | e.g., `'jenzabar_cx'` |
| `_source_table` | Original table name |
| `_loaded_at` | Fivetran load timestamp |
| `_row_hash` | MD5 of business key columns |
| `_dbt_updated_at` | `current_timestamp()` |

---

## Var Guard Pattern

For optional sources, use the var guard:

```sql
{{ config(enabled=var('has_jenzabar_cx', false)) }}
```

This allows the same project to run across schools with different SIS systems.

---

## Tags

Staging models should include:

```yaml
tags: ['staging', '<source_system>']
```

Example:
```yaml
tags: ['staging', 'jenzabar_cx']
```

---

## Related Docs

- [Project Structure](project-structure.md)
- [Testing Strategy](testing.md)
- [Macros Reference](macros.md)
