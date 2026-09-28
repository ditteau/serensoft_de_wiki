# Silent Failure Modes

Patterns that compile and run without error but produce wrong results. Each has been observed in this platform; the CLAUDE.md line or source file is cited.

**Last updated:** September 2026

---

## A. dbt / Jinja Patterns

### A.1 Unquoted `var()` in distribute models

A `var()` call renders as its value. In a WHERE clause, an unquoted school code becomes a bare SQL identifier:

```sql
-- ❌ WRONG: renders as WHERE school_code = DEMEAU (identifier, not string)
WHERE school_code = {{ var("school_code") }}

-- ✅ CORRECT: renders as WHERE school_code = 'DEMEAU'
WHERE school_code = '{{ var("school_code") }}'
```

The unquoted form compiles, runs, and returns zero rows — Snowflake reads `DEMEAU` as a column reference that does not exist, and the equality fails silently.

**Source:** `ditteau_data_transform/CLAUDE.md` lines 261–263

---

### A.2 `dim_program` joins require a `program_type` predicate

`program_code` is not unique across PROGRAM and SUBPROGRAM rows. A join on `program_code` alone fans out silently:

```sql
-- ❌ WRONG: fans out across PROGRAM and SUBPROGRAM
JOIN dim_program ON fact.program_code = dim_program.program_code

-- ✅ CORRECT: specify the program type
JOIN dim_program ON fact.program_code = dim_program.program_code
                AND dim_program.program_type = 'PROGRAM'
```

**Source:** `ditteau_data_transform/CLAUDE.md` lines 268–271

---

### A.3 `dim_program` column names

The actual column names are `academic_level` and `required_hours`. The columns `school_college` and `degree_type` do not exist — using them causes a compile error, but referencing the wrong names in documentation or queries against stale snapshots propagates quietly.

**Source:** `ditteau_data_transform/CLAUDE.md` lines 268–271 (implicit)

---

### A.4 JCX withdrawal sentinel

In Jenzabar CX, `withdrawal_code = '2'` means **not withdrawn**, not a withdrawal. Treating it as a withdrawal code inverts the logic:

```sql
-- ❌ WRONG: excludes non-withdrawn students
WHERE withdrawal_code != '2'

-- ✅ CORRECT: '2' is the sentinel for "not withdrawn"
WHERE withdrawal_code = '2'  -- active enrollment
```

**Source:** `ditteau_data_transform/CLAUDE.md` line 276

---

### A.5 Use `target.database`, never reconstruct from vars

The `var('env')` variable is not defined. Reconstructing database names from `var('school_code')` and `var('env')` fails at runtime:

```sql
-- ❌ WRONG: var('env') is undefined
{{ var('school_code') }}_DD_{{ var('env') }}.schema.table

-- ✅ CORRECT: use the target context object
{{ target.database }}.schema.table
```

The `target` context object is always available and contains the database from `profiles.yml`.

**Source:** `ditteau_data_transform/CLAUDE.md` lines 278–283

---

### A.6 Use the testenv dbt binary

The system dbt at `/opt/homebrew/bin/dbt` is the Cloud CLI, which fails on local credentials. Always use the project's virtual environment:

```bash
# ✅ CORRECT
PATH="/Users/laurievanpelt/testenv/bin:$PATH" bash scripts/run_demeau_dev.sh build

# ❌ WRONG: Cloud CLI, wrong authentication flow
/opt/homebrew/bin/dbt build
```

Never pass `--vars` directly to the run scripts — each script's own `--vars` block comes last and overwrites yours without warning.

**Source:** `ditteau_data_transform/CLAUDE.md` lines 124–147

---

### A.7 `dim_student` is SCD Type 2

`dim_student` maintains history. A plain join without `is_current` fans out across all versions:

```sql
-- ❌ WRONG: returns one row per historical version
SELECT COUNT(*) FROM fact_enrollment f
JOIN dim_student d ON f.student_id = d.student_id

-- ✅ CORRECT: filter to current version
SELECT COUNT(DISTINCT f.student_id) FROM fact_enrollment f
JOIN dim_student d ON f.student_id = d.student_id
  AND d.is_current = TRUE
```

For identifier counts, always use `COUNT(DISTINCT ...)` with an `is_current` predicate.

**Source:** `ditteau_data_transform/CLAUDE.md` lines 107, 117

---

### A.8 `snap_*` is not a reliable class — three of five carry no history

The `snap_` prefix reads as "snapshot", and platform notes have described the
family as "SCD Type 2 with `dbt_valid_from` timestamps". **Neither is reliably
true.**

None of the five `snap_*` models carries `dbt_valid_from` or `dbt_valid_to`.
They are not dbt snapshots. They are incremental merge models with a
`snapshot_date` column, and whether that column holds history depends entirely
on where its value comes from.

Measured 2026-09-28, both tenants:

| Model | `snapshot_date` source | Merrimack | DEMEAU | Historical? |
|---|---|---|---|---|
| `snap_enrollment_term` | `dim_term.term_beg_date` | 205 dates from 2002-05-28 | 164 from 2007-01-15 | **Yes** |
| `snap_cohort_milestone` | academic calendar | 23 dates from 2002-09-04 | 28 from 1997-09-01 | **Yes** |
| `snap_retention_term` | `current_date` | **1 date** | 2 dates, same month | **No** |
| `snap_admissions_weekly` | `current_date` | **1 date** | 2 dates, same month | **No** |
| `snap_aid_term` | `current_date` | **1 date** | **1 date** | **No** |

The bottom three stamp `snapshot_date` with **the day dbt ran**. They accumulate
history forward from first run and carry nothing before it. The DEMEAU pairs have
two dates only because dbt happened to run twice that month — that is the
mechanism visible, not a counterexample.

**Two consequences:**

1. **A `--full-refresh` on one of the bottom three destroys its accumulated
   history**, because there is no source to reproduce it from. On the top two a
   full refresh is safe: the dates come from the academic calendar, so a rebuild
   reproduces every one.
2. **Querying one of the bottom three "as of" a past date silently returns the
   current state**, because only one snapshot date exists to match.

**How to tell, for any `snap_*` model:**

```sql
-- If this returns 1, or returns only dates in the recent past,
-- the model carries no history.
SELECT COUNT(DISTINCT snapshot_date), MIN(snapshot_date), MAX(snapshot_date)
FROM <db>.DISTRIBUTE.<snap_model>;
```

Then check the model source — a `snapshot_date` derived from `dim_term` or the
academic calendar is real history; `current_date AS snapshot_date` is not.

**When adding a new `snap_*` model:** derive `snapshot_date` from the data's own
calendar wherever the grain allows it. Reach for `current_date` only when the
measurement genuinely has no date but today, and say so in the model header —
otherwise the prefix promises something the model does not deliver.

**Source:** measured against both tenant warehouses 2026-09-28; recorded in
`ditteau_data_transform/docs/forecasting/point_in_time_findings.md` §E

---

## B. Snowflake Patterns

*Verified 2026-09-26 against DEMEAU_DD_PROD.*

### B.1 `SYSADMIN` cannot see DISTRIBUTE objects

`DISTRIBUTE` schema objects are owned by `{CODE}_DBT_{ENV}` roles. `SYSADMIN` sees an empty result from `SHOW TABLES` / `INFORMATION_SCHEMA.TABLES` — not an error, just zero rows.

Use `DITTEAU_ADMIN` for verification queries against the modelled layer.

**Source:** `~/ditteau_context_regen/09_snowflake_verification.md` Section A

---

### B.2 `policy_references` is a table function, not a view

Querying `information_schema.policy_references` as a view returns "does not exist or not authorized." It must be called as a table function:

```sql
-- ❌ WRONG: looks like a missing object
SELECT * FROM DEMEAU_DD_PROD.INFORMATION_SCHEMA.POLICY_REFERENCES;

-- ✅ CORRECT: table function with entity parameters
SELECT policy_name, policy_kind, ref_column_name, policy_status
FROM TABLE(
    DEMEAU_DD_PROD.INFORMATION_SCHEMA.POLICY_REFERENCES(
        REF_ENTITY_NAME   => 'DEMEAU_DD_PROD.DISTRIBUTE.DIM_STUDENT',
        REF_ENTITY_DOMAIN => 'table'
    )
);
```

**Source:** `~/ditteau_context_regen/09_snowflake_verification.md` Section K.1

---

## Related Documentation

- [dbt Conventions](conventions.md) — Var guards and model patterns
- [Row Access Policies Runbook](../runbooks/row-access-policies.md) — Policy attachment verification
