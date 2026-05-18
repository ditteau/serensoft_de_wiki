# Data Quality Framework

Standards and practices for ensuring data quality across the Ditteau platform.

---

## DQ Dimensions

| Dimension | Definition | Example Check |
|---|---|---|
| **Completeness** | Required fields are populated | `NOT NULL` constraints |
| **Uniqueness** | No duplicate records | Primary key tests |
| **Validity** | Values conform to expected formats | Date ranges, enums |
| **Accuracy** | Data reflects reality | Cross-source validation |
| **Timeliness** | Data is current enough | Freshness tests |
| **Consistency** | Same data = same value everywhere | Referential integrity |

---

## dbt Testing Strategy

### Required Tests (All Models)

```yaml
models:
  - name: dim_student
    columns:
      - name: student_id
        tests:
          - unique
          - not_null
```

### Recommended Tests by Layer

| Layer | Test Types |
|---|---|
| Staging | `unique`, `not_null` on PKs; `accepted_values` for enums |
| Intermediate | Referential integrity; row count comparisons |
| Marts | Business rule validation; range checks |

---

## Freshness Monitoring

Configure source freshness in `_sources.yml`:

```yaml
sources:
  - name: jenzabar_cx
    freshness:
      warn_after: {count: 12, period: hour}
      error_after: {count: 24, period: hour}
    loaded_at_field: _fivetran_synced
```

---

## Data Contracts

For critical mart tables, define contracts:

- Expected columns and types
- Row count thresholds
- Business rule assertions

See [Data Contracts](data-contracts.md) for templates.
