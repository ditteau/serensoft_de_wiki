# SQL Style Guide

Snowflake SQL conventions for the Ditteau platform.

---

## Core Principles

1. **Consistency** — Follow the same patterns across all code
2. **Readability** — Code should be scannable and self-documenting
3. **Maintainability** — Easy to modify and extend

---

## Casing

### Keywords: UPPERCASE

```sql
SELECT
    student_id,
    first_name,
    last_name
FROM dim_student
WHERE is_active = TRUE
ORDER BY last_name
```

### Identifiers: snake_case

```sql
-- Tables and columns
dim_student
fact_enrollment
student_key
enrollment_date
is_current

-- NOT camelCase or PascalCase
StudentKey     -- Wrong
enrollmentDate -- Wrong
```

---

## Indentation

Use **2 spaces** for indentation (no tabs).

```sql
SELECT
  student_id,
  CASE
    WHEN enrollment_status = 'E' THEN 'Enrolled'
    WHEN enrollment_status = 'W' THEN 'Withdrawn'
    ELSE 'Unknown'
  END AS enrollment_status_desc
FROM fact_enrollment
WHERE term_key IS NOT NULL
```

---

## SELECT Statements

### Column Alignment

One column per line, comma-leading or comma-trailing (be consistent within a file):

```sql
-- Comma-trailing (preferred)
SELECT
  student_key,
  student_id,
  first_name,
  last_name,
  email
FROM dim_student
```

### SELECT *

Avoid `SELECT *` in production code. Explicitly list columns:

```sql
-- Good
SELECT
  student_key,
  student_id,
  first_name
FROM dim_student

-- Avoid in production
SELECT * FROM dim_student
```

---

## CTEs (Common Table Expressions)

Always use named CTEs for readability. This is the single most important pattern:

```sql
WITH source AS (
  SELECT *
  FROM {{ source('jenzabar_cx', 'id_rec') }}
),

filtered AS (
  SELECT *
  FROM source
  WHERE status = 'A'
),

renamed AS (
  SELECT
    id AS student_id,
    fname AS first_name,
    lname AS last_name
  FROM filtered
)

SELECT * FROM renamed
```

### CTE Naming Conventions

| Name | Purpose |
|------|---------|
| `source` | Raw source data |
| `filtered` | After WHERE clauses |
| `renamed` | After column renaming |
| `joined` | After JOIN operations |
| `aggregated` | After GROUP BY |
| `final` | Final transformations |

---

## JOINs

### Explicit JOIN Syntax

Always use explicit `JOIN` keywords:

```sql
-- Good
SELECT
  s.student_id,
  t.term_name
FROM fact_enrollment e
INNER JOIN dim_student s ON e.student_key = s.student_key
LEFT JOIN dim_term t ON e.term_key = t.term_key

-- Avoid implicit joins
SELECT s.student_id, t.term_name
FROM fact_enrollment e, dim_student s, dim_term t
WHERE e.student_key = s.student_key
```

### JOIN Order

1. Start with the fact/main table
2. Join dimensions in logical order
3. Use table aliases consistently

---

## WHERE Clauses

### Multiple Conditions

One condition per line:

```sql
WHERE is_active = TRUE
  AND enrollment_status IN ('E', 'R')
  AND term_key IS NOT NULL
  AND credit_hours > 0
```

### NULL Handling

Be explicit about NULL behavior:

```sql
-- Good
WHERE email IS NOT NULL

-- Avoid
WHERE email != ''  -- This doesn't catch NULLs
```

---

## CASE Expressions

Align WHEN/THEN/ELSE:

```sql
CASE
  WHEN enrollment_status = 'E' THEN 'Enrolled'
  WHEN enrollment_status = 'W' THEN 'Withdrawn'
  WHEN enrollment_status = 'G' THEN 'Graduated'
  ELSE 'Unknown'
END AS enrollment_status_desc
```

For simple cases, single line is acceptable:

```sql
CASE WHEN is_active THEN 'Yes' ELSE 'No' END AS active_flag
```

---

## Comments

### Inline Comments

Use `--` for inline comments:

```sql
SELECT
  student_id,
  credit_hours,  -- Total attempted hours
  gpa            -- Cumulative GPA
FROM dim_student
```

### Block Comments

Avoid `/* */` block comments. Use multiple `--` lines:

```sql
-- Calculate the enrollment census count
-- This includes only active students
-- as of the census date for each term
SELECT ...
```

---

## Aggregations

### GROUP BY

Reference columns by name, not position:

```sql
-- Good
SELECT
  term_code,
  COUNT(*) AS student_count
FROM fact_enrollment
GROUP BY term_code

-- Avoid
GROUP BY 1
```

### Snowflake GROUP BY with CASE

In Snowflake, GROUP BY must repeat the full CASE expression, not the alias:

```sql
-- Correct
SELECT
  CASE WHEN gpa >= 3.0 THEN 'Dean''s List' ELSE 'Regular' END AS standing,
  COUNT(*) AS student_count
FROM dim_student
GROUP BY CASE WHEN gpa >= 3.0 THEN 'Dean''s List' ELSE 'Regular' END

-- Wrong (causes SQL compilation error)
GROUP BY standing
```

---

## Snowflake-Specific Patterns

### QUALIFY

Use `QUALIFY` for window function filtering:

```sql
SELECT
  student_id,
  term_code,
  gpa
FROM fact_student_term
QUALIFY ROW_NUMBER() OVER (
  PARTITION BY student_id
  ORDER BY term_code DESC
) = 1
```

### FLATTEN for Arrays

```sql
SELECT
  f.value::STRING AS course_code
FROM dim_student,
LATERAL FLATTEN(input => courses_array) f
```

### TRY_ Functions

Use `TRY_` variants for safe casting:

```sql
TRY_TO_NUMBER(credit_hours)    -- Returns NULL on failure
TRY_TO_DATE(date_string)       -- Returns NULL on failure
TRY_PARSE_JSON(json_string)    -- Returns NULL on failure
```

---

## Anti-Patterns to Avoid

| Avoid | Prefer |
|-------|--------|
| `SELECT *` | Explicit column list |
| Implicit joins | Explicit `JOIN` syntax |
| `GROUP BY 1, 2` | `GROUP BY column_name` |
| Nested subqueries | CTEs |
| `/* block comments */` | `-- inline comments` |
| Mixed case keywords | UPPERCASE keywords |

---

## dbt-Specific Patterns

### ref() and source()

```sql
FROM {{ ref('dim_student') }}
FROM {{ source('jenzabar_cx', 'id_rec') }}
```

### Jinja in SQL

Keep Jinja readable:

```sql
SELECT
  {{ dbt_utils.generate_surrogate_key(['student_id', 'term_code']) }} AS enrollment_key,
  student_id,
  {% if var('include_pii', false) %}
  first_name,
  last_name,
  {% endif %}
  term_code
FROM {{ ref('stg_jcx__enrollments') }}
```

---

## Related Documentation

- [dbt Conventions](../dbt/conventions.md) — Model standards
- [Performance Tips](performance.md) — Query optimization
