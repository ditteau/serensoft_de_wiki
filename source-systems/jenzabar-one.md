# Jenzabar One

Modern ERP/SIS platform from Jenzabar.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Jenzabar |
| Product | Jenzabar One (J1) |
| Database | Microsoft SQL Server |
| Schools | Merrimack (primary), DEMEAU (demo) |
| Status | Active — primary SIS for Merrimack |

Jenzabar One is Jenzabar's modern cloud-based ERP/SIS, replacing the legacy CX product.

---

## Connection Method

**CSV Deposit** via deposit_loader

Files are exported from J1 and loaded to the DEPOSIT schema:

```bash
python deposit_loader.py --school MERRIMACK load jenzabar_one students ~/inbox/j1/students.csv
```

---

## Registry Statistics

From `ditteau_schema_registry` and `source_registry.yml`:

| Metric | Count |
|--------|-------|
| Tables | 2,679 |
| Registered in deposit_loader | Yes |

---

## Key Tables

### Student Records

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `STUDENTS` | Master student record | `STUDENT_ID`, `FIRST_NAME`, `LAST_NAME`, `BIRTH_DATE` |
| `STUDENT_ADDRESSES` | Address history | `STUDENT_ID`, `ADDRESS_TYPE`, `STREET`, `CITY`, `STATE` |
| `STUDENT_EMAILS` | Email addresses | `STUDENT_ID`, `EMAIL_TYPE`, `EMAIL_ADDRESS` |

### Academic Records

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `ACADEMIC_CREDITS` | Credit records | `STUDENT_ID`, `TERM_ID`, `CREDITS_ATTEMPTED`, `CREDITS_EARNED` |
| `PROGRAMS` | Academic programs | `PROGRAM_ID`, `PROGRAM_NAME`, `DEGREE_TYPE` |
| `ACADEMIC_TERMS` | Term definitions | `TERM_ID`, `TERM_CODE`, `START_DATE`, `END_DATE` |

### Enrollment

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `ENROLLMENTS` | Course enrollments | `STUDENT_ID`, `SECTION_ID`, `STATUS`, `GRADE` |
| `SECTIONS` | Course sections | `SECTION_ID`, `COURSE_ID`, `TERM_ID`, `INSTRUCTOR_ID` |
| `COURSES` | Course catalog | `COURSE_ID`, `COURSE_NAME`, `CREDITS`, `DEPARTMENT` |

---

## Schema Notes

### Modern Design

Unlike CX, J1 uses:
- Proper NULL handling
- Surrogate keys
- Normalized tables
- Standard SQL data types

### Naming Conventions

- Tables: PascalCase or UPPER_SNAKE_CASE
- Columns: snake_case or PascalCase (varies)
- IDs: Often GUIDs or integer sequences

---

## Staging Models

Located in `models/deterge/staging/jenzabar_one/`:

```
stg_j1__students.sql
stg_j1__enrollments.sql
stg_j1__academic_terms.sql
...
```

### Source Definition

```yaml
# _j1_sources.yml
sources:
  - name: jenzabar_one
    database: "{{ var('j1_source_database') }}"
    schema: deposit
    tables:
      - name: students
      - name: enrollments
      ...
```

### Var Guard

```sql
{{ config(enabled=var('has_jenzabar_one', false)) }}
```

---

## Primary SIS Configuration

Merrimack and DEMEAU use J1 as their primary SIS:

```yaml
# In run script vars
primary_sis: JENZABAR_ONE
```

This affects identity resolution in `int_ditteau_id_registry`.

---

## Data Loading

### Create Tables

```bash
python deposit_loader.py --school MERRIMACK create-tables jenzabar_one
```

Note: 2,679 tables may take 3-5 minutes to create.

### Load Data

```bash
# Single file
python deposit_loader.py --school MERRIMACK load jenzabar_one students students.csv

# All files in directory
python deposit_loader.py --school MERRIMACK load-all jenzabar_one ~/inbox/j1/
```

---

## Related Documentation

- [Source Systems Overview](README.md)
- [Source Refresh Runbook](../runbooks/source-refresh.md)
- [Jenzabar CX](jenzabar-cx.md) — Legacy comparison
