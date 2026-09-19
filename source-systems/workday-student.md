# Workday Student

Cloud-based Student Information System from Workday.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Workday |
| Product | Workday Student |
| Platform | Cloud (SaaS) |
| Schools | Saint Anselm (primary), Colby, Endicott |
| Status | Active |

Workday Student is part of Workday's cloud HCM/ERP suite, providing modern SIS capabilities.

---

## Connection Method

**CSV Deposit** via deposit_loader

Workday exports are loaded to the DEPOSIT schema:

```bash
python deposit_loader.py --school ANSELM load workday_student students ~/inbox/wd/students.csv
```

---

## Registry Statistics

From `source_registry.yml`:

| Metric | Count |
|--------|-------|
| Tables | 2 |
| `students` columns | 79 |
| `academic_records` columns | 92 |

Workday's API provides denormalized exports rather than raw tables.

---

## Key Tables

### Students

| Column | Description |
|--------|-------------|
| `STUDENT_ID` | Workday student identifier |
| `FIRST_NAME`, `LAST_NAME` | Legal name |
| `PREFERRED_NAME` | Preferred first name |
| `EMAIL` | Institutional email |
| `DATE_OF_BIRTH` | Birth date |
| `GENDER` | Gender code |
| `ETHNICITY` | IPEDS ethnicity |
| `CITIZENSHIP_STATUS` | Citizenship code |
| `PROGRAM_OF_STUDY` | Current academic program |
| `ACADEMIC_LEVEL` | UNDG, GRAD, etc. |

### Academic Records

| Column | Description |
|--------|-------------|
| `STUDENT_ID` | Links to students |
| `TERM_CODE` | Academic term |
| `COURSE_ID` | Course identifier |
| `CREDITS_ATTEMPTED` | Credit hours attempted |
| `CREDITS_EARNED` | Credit hours earned |
| `GRADE` | Final grade |
| `GPA_TERM` | Term GPA |
| `GPA_CUMULATIVE` | Cumulative GPA |

---

## Schema Notes

### Denormalized Structure

Workday exports are denormalized for simplicity:
- Student demographics in one wide table
- Academic records flattened

### Data Quality

Generally clean data:
- Proper NULL handling
- Consistent date formats (ISO 8601)
- Standardized code values

### ID Mapping

Workday IDs may differ from legacy system IDs. The `int_ditteau_id_registry` model handles cross-system identity resolution.

---

## Staging Models

Located in `models/deterge/staging/workday/`:

```
stg_wd__students.sql
stg_wd__academic_records.sql
```

### Source Definition

```yaml
# _wd_sources.yml
sources:
  - name: workday_student
    database: "{{ target.database }}"
    schema: deposit
    tables:
      - name: students
      - name: academic_records
```

### Var Guard

```sql
{{ config(enabled=var('has_workday', false)) }}
```

---

## Primary SIS Configuration

Saint Anselm uses Workday as primary SIS:

```yaml
# In run script vars
primary_sis: WORKDAY
```

---

## School-Specific Seeds

Saint Anselm has Workday-specific mappings:

```
seeds/anselm/
├── seed_anselm_wd_program_of_study.csv   # Program code crosswalk
└── seed_anselm_ipeds_peer_group.csv      # Peer institution list
```

---

## Data Loading

### Create Tables

```bash
python deposit_loader.py --school ANSELM create-tables workday_student
```

### Load Data

```bash
python deposit_loader.py --school ANSELM load workday_student students students.csv
python deposit_loader.py --school ANSELM load workday_student academic_records records.csv
```

---

## Integration Notes

### Cross-System Joins

Schools with both Workday and legacy JCX data need careful identity resolution:

```sql
-- In int_ditteau_id_registry
SELECT
  COALESCE(wd.student_id, jcx.id) AS resolved_student_id,
  ...
FROM stg_wd__students wd
FULL OUTER JOIN stg_jcx__id_rec jcx
  ON matching_criteria
```

### Historical Data

Some schools maintain JCX archives for historical data while using Workday for current students.

---

## Related Documentation

- [Source Systems Overview](README.md)
- [Source Refresh Runbook](../runbooks/source-refresh.md)
- [Identity Resolution](../architecture/data-flow.md#identity-resolution)
