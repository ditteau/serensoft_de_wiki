# Data Quality Framework

Standards, practices, and automated controls for ensuring data quality across the Ditteau platform.

---

## Overview

The DQ framework operates at three levels:

| Level | Scope | Enforcement |
|-------|-------|-------------|
| **Schema Tests** | Column-level validity | dbt native tests, dbt_expectations |
| **Business Rules** | Cross-table consistency | Custom data tests |
| **Governance Assertions** | Access control integrity | `tests/governance/` suite |

All tests run as part of `dbt build` and failures block deployment.

---

## DQ Dimensions

| Dimension | Definition | Example Checks |
|-----------|------------|----------------|
| **Completeness** | Required fields are populated | `not_null` on required columns |
| **Uniqueness** | No duplicate records | `unique` on primary keys |
| **Validity** | Values conform to expected formats | `accepted_values`, regex patterns, ranges |
| **Accuracy** | Data reflects reality | Cross-source reconciliation |
| **Timeliness** | Data is current enough | Source freshness tests |
| **Consistency** | Same data = same value everywhere | Referential integrity, cross-table checks |

---

## Test Packages

### dbt Native Tests

Built-in tests for common validations:

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

### dbt_expectations

Advanced data quality tests from the `metaplane/dbt_expectations` package (v0.10.1):

```yaml
models:
  - name: fact_enrollment
    tests:
      # Table-level
      - dbt_expectations.expect_table_row_count_to_be_between:
          min_value: 1000
          max_value: 10000000

    columns:
      - name: credit_hours
        tests:
          # Range validation
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
              max_value: 24
              strictly: false

      - name: email
        tests:
          # Pattern validation
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\\.[a-zA-Z0-9-.]+$"

      - name: enrollment_date
        tests:
          # Temporal validation
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: "'2000-01-01'"
              max_value: "current_date()"
```

### Available dbt_expectations Tests

| Test | Purpose |
|------|---------|
| `expect_table_row_count_to_be_between` | Row count bounds |
| `expect_column_values_to_be_between` | Numeric/date ranges |
| `expect_column_values_to_match_regex` | Pattern matching |
| `expect_column_values_to_be_unique` | Uniqueness |
| `expect_column_values_to_not_be_null` | Completeness |
| `expect_column_distinct_count_to_equal` | Cardinality |
| `expect_column_mean_to_be_between` | Statistical bounds |
| `expect_column_proportion_of_unique_values_to_be_between` | Uniqueness ratio |

---

## Tests by Layer

### Staging Layer

| Test Type | Purpose | Example |
|-----------|---------|---------|
| `not_null` | Required fields | Primary keys, foreign keys |
| `unique` | No duplicates | Natural keys |
| `accepted_values` | Valid codes | Status codes, gender codes |

```yaml
# models/deterge/staging/jenzabar_cx/_jcx_models.yml
models:
  - name: stg_jcx__id_rec
    columns:
      - name: student_id
        tests:
          - unique
          - not_null
      - name: status_code
        tests:
          - accepted_values:
              values: ['A', 'I', 'G', 'W', 'D']
```

### Intermediate Layer

| Test Type | Purpose | Example |
|-----------|---------|---------|
| `relationships` | Referential integrity | FK to staging models |
| Row count comparisons | No data loss | Source vs destination counts |
| Cross-source reconciliation | Identity resolution | ID registry validation |

### Distribute Layer

| Test Type | Purpose | Example |
|-----------|---------|---------|
| Business rules | Domain logic | GPA ranges, credit limits |
| Grain validation | Correct aggregation | Unique combination of keys |
| Completeness | Required dimensions | No orphan facts |

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

## Governance Assertion Tests

The `tests/governance/` directory contains 18 assertion tests that validate access control integrity. These are not data quality tests — they ensure the governance model is correctly configured.

### Access Control Assertions

| Test | Assertion ID | Purpose |
|------|--------------|---------|
| `assert_public_role_holds_no_tier` | N-12 | PUBLIC role has no domain access (fail-closed default) |
| `assert_no_app_owner_in_role_domain_access` | N-13 | Streamlit owner roles require explicit exceptions |
| `assert_separation_of_duties` | N-8 | No unauthorized academic + financial aid access |
| `assert_grid_contains_only_leaf_roles` | N-12 | Only leaf roles in entitlement grid |
| `assert_deployed_grid_is_authorised` | N-16a | Deployed tiers match authorized grid |

### Aggregate Model Assertions

| Test | Assertion ID | Purpose |
|------|--------------|---------|
| `assert_aggregate_register_covers_all_marts` | N-7 | All aggregates registered for suppression review |
| `assert_aggregate_register_has_no_orphans` | — | No stale register entries |
| `assert_aggregate_models_cleared` | — | Registered models cleared for AGGREGATED access |
| `assert_privacy_block_reaches_aggregates` | — | FERPA blocks propagate to aggregates |

### Grid Integrity Assertions

| Test | Assertion ID | Purpose |
|------|--------------|---------|
| `assert_grid_overrides_are_valid` | N-16c | WIDEN overrides have justification |
| `assert_cross_domain_tier_coherence` | N-14 | Related domains have coherent tiers |
| `assert_no_unimplemented_scoped_tiers` | — | SCOPED tiers have implementation |
| `assert_entitlement_exceptions_expire` | — | Exceptions have review dates |

### Health Data Assertions

| Test | Assertion ID | Purpose |
|------|--------------|---------|
| `assert_student_health_unentitled` | N-21 | No persona holds student_health tier |
| `assert_health_holds_classified` | — | Health columns properly classified |

### Operational Assertions

| Test | Purpose |
|------|---------|
| `assert_dbt_run_audit_written` | Audit logging active |
| `assert_model_governance_metadata` | Models have governance tags |
| `assert_no_undeclared_null_columns` | NULL columns declared in registry |

---

## Source Freshness

Configure freshness thresholds in source YAML:

```yaml
sources:
  - name: jenzabar_cx
    freshness:
      warn_after: {count: 24, period: hour}
      error_after: {count: 48, period: hour}
    loaded_at_field: _loaded_at

    tables:
      - name: id_rec
        freshness:
          # Override for critical tables
          warn_after: {count: 12, period: hour}
```

Run freshness checks:

```bash
# All sources
dbt source freshness

# Specific source
dbt source freshness --select source:jenzabar_cx
```

---

## Test Severity

Control test behavior with severity levels:

| Severity | Behavior | Use For |
|----------|----------|---------|
| `error` | Fails the build | Critical integrity (PKs, FKs) |
| `warn` | Logs warning, continues | Known issues, monitoring |

```yaml
columns:
  - name: student_id
    tests:
      - not_null:
          severity: error
      - unique:
          severity: error
  - name: advisor_id
    tests:
      - not_null:
          severity: warn  # Known data quality issue
```

### Governance Test Severities

| Severity | Tests |
|----------|-------|
| `error` | All access control assertions (N-*) |
| `warn` | `assert_aggregate_models_cleared`, `assert_aggregate_register_has_no_orphans` |

The `warn` severities for aggregate tests are deliberate: clearance is a backlog, not a gate. If these blocked builds, they would be deleted within a week.

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

### Query Failed Rows

```sql
SELECT *
FROM MERRIMACK_DD_DEV.DBT_TEST_RESULTS.unique_dim_student_student_key
LIMIT 100;
```

---

## Running Tests

### Full Test Suite

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh test
```

### Specific Tests

```bash
# Tests for one model
bash scripts/run_merrimack_dev.sh test --select dim_student

# Governance tests only
bash scripts/run_merrimack_dev.sh test --select tag:governance

# Tests for a model and downstream
bash scripts/run_merrimack_dev.sh test --select dim_student+
```

### Build (Run + Test)

```bash
# Runs model and tests together
bash scripts/run_merrimack_dev.sh build --select mart_enrollment_census
```

---

## Test Organization

```
tests/
├── assert_anselm_crosswalk_coverage.sql
├── assert_class_level_codes_decoded.sql
├── assert_no_duplicate_source_identity.sql
├── assert_source_column_scan_is_live.sql
├── assert_suppressed_term_dates_are_declared.sql
└── governance/
    ├── assert_aggregate_models_cleared.sql
    ├── assert_aggregate_register_covers_all_marts.sql
    ├── assert_aggregate_register_has_no_orphans.sql
    ├── assert_cross_domain_tier_coherence.sql
    ├── assert_dbt_run_audit_written.sql
    ├── assert_deployed_grid_is_authorised.sql
    ├── assert_entitlement_exceptions_expire.sql
    ├── assert_grid_contains_only_leaf_roles.sql
    ├── assert_grid_overrides_are_valid.sql
    ├── assert_health_holds_classified.sql
    ├── assert_model_governance_metadata.sql
    ├── assert_no_app_owner_in_role_domain_access.sql
    ├── assert_no_undeclared_null_columns.sql
    ├── assert_no_unimplemented_scoped_tiers.sql
    ├── assert_privacy_block_reaches_aggregates.sql
    ├── assert_public_role_holds_no_tier.sql
    ├── assert_separation_of_duties.sql
    └── assert_student_health_unentitled.sql
```

The `tests/governance/` directory is CODEOWNERS-protected. Deleting tests from it is the quiet way to defeat access controls.

---

## Adding New Tests

### Schema Tests (YAML)

Add to the model's YAML file in `_models.yml`:

```yaml
columns:
  - name: new_column
    tests:
      - not_null
      - unique
```

### Data Tests (SQL)

Create a SQL file in `tests/`:

```sql
-- tests/assert_no_orphan_enrollments.sql
SELECT *
FROM {{ ref('fact_enrollment') }} e
LEFT JOIN {{ ref('dim_student') }} s ON e.student_key = s.student_key
WHERE s.student_key IS NULL
```

A test passes if it returns **zero rows**.

### Generic Tests

For reusable test logic, create in `tests/generic/`:

```sql
-- tests/generic/test_positive_amount.sql
{% test positive_amount(model, column_name) %}
SELECT *
FROM {{ model }}
WHERE {{ column_name }} < 0
{% endtest %}
```

---

## Related Documentation

- [Data Contracts](data-contracts.md) — Schema contracts and field-level governance
- [dbt Testing](../dbt/testing.md) — Testing patterns
- [Row Access Policies](../runbooks/row-access-policies.md) — RAP implementation
- [Production Operating Rules](production-operating-rules.md) — Operational controls
