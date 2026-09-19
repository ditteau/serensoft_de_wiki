# Performance Tips

Query optimization and performance best practices for Snowflake.

---

## Warehouse Sizing

### Right-Size for Workload

| Workload | Recommended Size | Notes |
|----------|------------------|-------|
| dbt builds (small) | X-Small | Most staging/dimension models |
| dbt builds (large) | Small | Full refresh of large marts |
| Dashboard queries | Small | Concurrent users |
| Ad-hoc analysis | X-Small to Small | Scale as needed |
| Heavy aggregations | Medium | Large GROUP BY operations |

### Multi-Cluster Warehouses

For production analytics with concurrent users:

```sql
ALTER WAREHOUSE MERRIMACK_ANALYTICS_PROD SET
  MIN_CLUSTER_COUNT = 1
  MAX_CLUSTER_COUNT = 3
  SCALING_POLICY = 'STANDARD';
```

---

## Auto-Suspend and Resume

Configure appropriate suspend times:

| Warehouse Type | Auto-Suspend | Rationale |
|----------------|--------------|-----------|
| Transform | 60 seconds | Short bursts, infrequent use |
| Analytics | 300 seconds | User sessions may have gaps |

```sql
ALTER WAREHOUSE MERRIMACK_TRANSFORM_DEV SET
  AUTO_SUSPEND = 60
  AUTO_RESUME = TRUE;
```

---

## Query Optimization

### Use WHERE Clauses Early

Filter data as early as possible in CTEs:

```sql
-- Good: filter in first CTE
WITH source AS (
  SELECT *
  FROM {{ source('jenzabar_cx', 'id_rec') }}
  WHERE status = 'A'  -- Filter early
),
...

-- Avoid: filter late
WITH source AS (
  SELECT * FROM ...
),
final AS (
  SELECT * FROM source
  WHERE status = 'A'  -- Filtering millions of rows late
)
```

### Avoid SELECT *

Explicitly list needed columns:

```sql
-- Good
SELECT student_id, first_name, last_name
FROM dim_student

-- Avoid (pulls all columns into memory)
SELECT * FROM dim_student
```

### Use QUALIFY for Window Functions

More efficient than subqueries:

```sql
-- Good: QUALIFY
SELECT student_id, term_code, gpa
FROM fact_student_term
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY student_id ORDER BY term_code DESC
) = 1

-- Avoid: subquery
SELECT *
FROM (
  SELECT student_id, term_code, gpa,
    ROW_NUMBER() OVER (...) AS rn
  FROM fact_student_term
)
WHERE rn = 1
```

---

## Clustering

### When to Cluster

Consider clustering for:
- Tables > 1TB
- Frequently filtered columns
- Range queries on dates

### Clustering Keys

Choose based on query patterns:

```sql
-- Cluster on frequently filtered columns
ALTER TABLE fact_enrollment CLUSTER BY (term_key, student_key);

-- Check clustering depth
SELECT SYSTEM$CLUSTERING_DEPTH('fact_enrollment');
```

### Automatic Clustering

Enable for large, frequently-queried tables:

```sql
ALTER TABLE fact_enrollment RESUME RECLUSTER;
```

---

## Materialization Strategy

### Views vs Tables

| Use Views When | Use Tables When |
|----------------|-----------------|
| Source is small (<100K rows) | Data is queried frequently |
| Always need current data | Complex transformations |
| Simple transformations | Multiple downstream dependencies |
| Storage cost is a concern | Query performance is critical |

### Incremental Models

For large fact tables, consider incremental materialization:

```sql
{{
  config(
    materialized='incremental',
    unique_key='enrollment_key',
    incremental_strategy='merge'
  )
}}

SELECT ...
FROM {{ source('jenzabar_cx', 'enrollments') }}
{% if is_incremental() %}
WHERE _loaded_at > (SELECT MAX(_loaded_at) FROM {{ this }})
{% endif %}
```

---

## Result Caching

### Query Result Cache

Snowflake automatically caches query results for 24 hours. Identical queries return cached results instantly.

To benefit from caching:
- Use consistent SQL formatting
- Avoid non-deterministic functions in cached queries
- Don't disable caching unless necessary

### Metadata Cache

Table metadata is cached. Operations like `COUNT(*)` on unchanged tables are instant.

---

## Common Anti-Patterns

### Cartesian Joins

```sql
-- Dangerous: no join condition
SELECT * FROM table_a, table_b

-- Explicit cross join if intended
SELECT * FROM table_a CROSS JOIN table_b
```

### ORDER BY Without LIMIT

```sql
-- Avoid: sorts entire result set
SELECT * FROM dim_student ORDER BY last_name

-- Better: limit results
SELECT * FROM dim_student ORDER BY last_name LIMIT 100
```

### Functions on Indexed Columns

```sql
-- Avoid: function prevents pruning
WHERE UPPER(email) = 'TEST@EXAMPLE.COM'

-- Better: store normalized, query directly
WHERE email_normalized = 'test@example.com'
```

---

## Monitoring

### Query History

```sql
SELECT
  query_id,
  query_text,
  total_elapsed_time / 1000 AS elapsed_seconds,
  bytes_scanned / 1024 / 1024 AS mb_scanned
FROM snowflake.account_usage.query_history
WHERE warehouse_name = 'MERRIMACK_ANALYTICS_PROD'
  AND start_time > DATEADD(day, -1, CURRENT_TIMESTAMP())
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

### Warehouse Utilization

```sql
SELECT
  warehouse_name,
  DATE_TRUNC('hour', start_time) AS hour,
  SUM(credits_used) AS credits
FROM snowflake.account_usage.warehouse_metering_history
WHERE start_time > DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY 1, 2
ORDER BY 1, 2;
```

---

## dbt-Specific Tips

### Reduce Full Refreshes

Use incremental models where possible to avoid rebuilding large tables.

### Parallelize Independent Models

dbt automatically parallelizes independent models. Ensure your DAG allows parallel execution.

### Defer to Production

During development, defer to production for unchanged models:

```bash
dbt build --select my_model --defer --state prod-manifest
```

---

## Related Documentation

- [SQL Style Guide](sql-style.md) — Query formatting
- [dbt Conventions](../dbt/conventions.md) — Model standards
