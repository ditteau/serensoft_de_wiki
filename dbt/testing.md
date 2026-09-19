# dbt Testing Strategy

Testing patterns, data quality validation, and governance assertions for the Ditteau platform.

---

## Test Framework Overview

| Component | Purpose |
|-----------|---------|
| dbt native tests | `unique`, `not_null`, `accepted_values`, `relationships` |
| `dbt_expectations` | Advanced data quality tests (distributions, patterns, ranges) |
| Governance assertions | Custom tests ensuring access control integrity |
| Source freshness | Alerting on stale source data |

---

## Test Configuration

### Store Failures

All test failures are persisted for debugging:

```yaml
# dbt_project.yml
tests:
  +store_failures: true
  +schema: dbt_test_results
```

Failed rows are written to `{DATABASE}.DBT_TEST_RESULTS.{test_name}`.

### Source Freshness

Configured per source table:

```yaml
sources:
  - name: jenzabar_cx
    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    loaded_at_field: _loaded_at
```

Run freshness checks:

```bash
dbt source freshness --select source:jenzabar_cx
```

---

## Test Types

### Schema Tests

Applied to columns in YAML:

```yaml
models:
  - name: dim_student
    columns:
      - name: student_key
        tests:
          - unique
          - not_null
      - name: gender_code
        tests:
          - accepted_values:
              values: ['M', 'F', 'U', 'N']
      - name: term_key
        tests:
          - relationships:
              to: ref('dim_term')
              field: term_key
```

### Data Tests

Standalone SQL tests in `tests/`:

```sql
-- tests/assert_no_orphan_enrollments.sql
SELECT *
FROM {{ ref('fact_enrollment') }} e
LEFT JOIN {{ ref('dim_student') }} s ON e.student_key = s.student_key
WHERE s.student_key IS NULL
```

A test passes if it returns **zero rows**.

### dbt_expectations Tests

For advanced data quality:

```yaml
models:
  - name: fact_enrollment
    tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          max_value: 1000000
    columns:
      - name: credit_hours
        tests:
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 24
      - name: email
        tests:
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"
```

---

## Governance Assertion Tests

Custom tests ensuring access control integrity. Located in `tests/governance/`:

### Key Governance Tests

| Test | Purpose |
|------|---------|
| `assert_public_role_holds_no_tier.sql` | Ensures PUBLIC role has no domain access |
| `assert_no_app_owner_in_role_domain_access.sql` | Streamlit owner roles cannot hold tiers directly |
| `assert_separation_of_duties.sql` | Validates principle 8 separation |
| `assert_no_orphan_role_grants.sql` | All granted roles exist in the grid |

### Example: Separation of Duties

```sql
-- tests/governance/assert_separation_of_duties.sql
-- Principle 8: No role holds FULL on both student_academic and financial_aid
-- with identity unmask, unless explicitly allowlisted

SELECT role_name
FROM {{ ref('seed_persona_domain_grid') }}
WHERE student_academic_tier = 'FULL'
  AND financial_aid_tier = 'FULL'
  AND (unmask_name = TRUE OR unmask_dob = TRUE)
  AND role_name NOT IN (
    SELECT role_name FROM {{ ref('seed_separation_allowlist') }}
  )
```

---

## Running Tests

### All Tests

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh test
```

### Specific Model Tests

```bash
# Tests for one model
bash scripts/run_merrimack_dev.sh test --select dim_student

# Tests for a model and its downstream
bash scripts/run_merrimack_dev.sh test --select dim_student+
```

### Build (Run + Test)

```bash
# Build runs the model and its tests together
bash scripts/run_merrimack_dev.sh build --select mart_enrollment_census
```

---

## Test Severity

Control test behavior with severity:

```yaml
columns:
  - name: student_id
    tests:
      - not_null:
          severity: error    # Fails the run
      - unique:
          severity: warn     # Logs warning, continues
```

Use `warn` for:
- Known data quality issues being addressed
- Non-critical validations
- Monitoring without blocking

Use `error` for:
- Primary key integrity
- Foreign key relationships
- Critical business rules

---

## Common Test Patterns

### Referential Integrity

```yaml
columns:
  - name: student_key
    tests:
      - relationships:
          to: ref('dim_student')
          field: student_key
          severity: error
```

### Accepted Values with Null Handling

```yaml
columns:
  - name: enrollment_status
    tests:
      - accepted_values:
          values: ['ENROLLED', 'WITHDRAWN', 'GRADUATED', 'LEAVE']
          config:
            where: "enrollment_status IS NOT NULL"
```

### Row Count Sanity

```yaml
models:
  - name: dim_student
    tests:
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          strictly: false
```

### Unique Combination

```yaml
models:
  - name: fact_enrollment
    tests:
      - dbt_utils.unique_combination_of_columns:
          combination_of_columns:
            - student_key
            - term_key
            - course_key
```

---

## Debugging Test Failures

### View Failed Rows

```sql
SELECT *
FROM MERRIMACK_DD_DEV.DBT_TEST_RESULTS.<test_name>
LIMIT 100;
```

### Investigate with Audit Helper

Compare current vs expected:

```sql
{{ audit_helper.compare_relations(
    a_relation=ref('dim_student'),
    b_relation=ref('dim_student_expected'),
    primary_key='student_key'
) }}
```

---

## Test Organization

```
tests/
├── generic/
│   ├── test_valid_email.sql
│   └── test_positive_amount.sql
├── governance/
│   ├── assert_public_role_holds_no_tier.sql
│   ├── assert_no_app_owner_in_role_domain_access.sql
│   └── assert_separation_of_duties.sql
└── staging/
    └── assert_no_duplicate_source_ids.sql
```

---

## Related Documentation

- [Conventions](conventions.md) — Model standards
- [DQ Framework](../governance/dq-framework.md) — Data quality framework
- [Macros Reference](macros.md) — Test helper macros
