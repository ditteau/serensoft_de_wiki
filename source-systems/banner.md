# Ellucian Banner

Enterprise ERP/SIS from Ellucian.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Ellucian |
| Product | Banner |
| Database | Oracle |
| Schools | Springfield (primary) |
| Status | Active — onboarding |

Banner is one of the most widely-used SIS platforms in higher education, particularly at larger institutions.

---

## Connection Method

**CSV Deposit** via deposit_loader

Banner exports are loaded to the DEPOSIT schema:

```bash
python deposit_loader.py --school SPRINGFIELD load banner students ~/inbox/banner/students.csv
```

---

## Key Tables

Banner uses a naming convention where tables are prefixed by module:

### Student Module (S)

| Table | Purpose |
|-------|---------|
| `SPRIDEN` | Person identification |
| `SPBPERS` | Person biographical |
| `SGBSTDN` | Student general |
| `SHRTCKN` | Student course registration |
| `SHRTCKG` | Course registration grades |

### Academic History Module (SH)

| Table | Purpose |
|-------|---------|
| `SHRTRIT` | Transfer institution |
| `SHRTCKN` | Course registration (enrollment) |
| `SHRTRCE` | Transfer credit |

### General Module (G)

| Table | Purpose |
|-------|---------|
| `GOREMAL` | Email addresses |
| `GORPRAC` | Race codes |
| `STVATYP` | Address types |

---

## Schema Notes

### Table Naming

Banner uses 7-character table names:
- First 3 chars: Module (SPR, SGB, SHR, GOR, etc.)
- Last 4 chars: Entity identifier

### Common Patterns

- `_PIDM`: Person ID Master (foreign key to person)
- `_CODE`: Code lookup value
- `_DESC`: Description field
- `_IND`: Indicator (Y/N flag)
- `_DATE`: Date field

### Character Data

- Fixed-width CHAR fields common
- Often requires TRIM()
- NULL vs empty string handling varies

---

## Staging Models

Located in `models/deterge/staging/banner/`:

```
stg_banner__spriden.sql
stg_banner__spbpers.sql
stg_banner__sgbstdn.sql
...
```

### Source Definition

```yaml
# _banner_sources.yml
sources:
  - name: banner
    database: "{{ target.database }}"
    schema: deposit
    tables:
      - name: spriden
      - name: spbpers
      ...
```

### Var Guard

```sql
{{ config(enabled=var('has_banner', false)) }}
```

---

## Primary SIS Configuration

Springfield uses Banner as primary SIS:

```yaml
# In run script vars
primary_sis: BANNER
```

---

## Data Loading

### Create Tables

```bash
python deposit_loader.py --school SPRINGFIELD create-tables banner
```

### Load Data

```bash
python deposit_loader.py --school SPRINGFIELD load banner spriden spriden.csv
python deposit_loader.py --school SPRINGFIELD load-all banner ~/inbox/banner/
```

---

## ERD Reference

Banner has extensive documentation. Key relationships:

```
SPRIDEN (person)
  └─ SPBPERS (biographical)
  └─ SGBSTDN (student record)
       └─ SHRTCKN (course registration)
            └─ SHRTCKG (grades)
```

The `PIDM` (Person ID Master) is the primary key linking all person-related tables.

---

## Common Transformations

### Handle PIDM Joins

```sql
SELECT
  s.spriden_pidm,
  s.spriden_id AS student_id,
  p.spbpers_birth_date AS birth_date
FROM {{ source('banner', 'spriden') }} s
JOIN {{ source('banner', 'spbpers') }} p
  ON s.spriden_pidm = p.spbpers_pidm
```

### Indicator Fields

```sql
CASE WHEN veteran_ind = 'Y' THEN TRUE ELSE FALSE END AS is_veteran
```

---

## Related Documentation

- [Source Systems Overview](README.md)
- [Source Refresh Runbook](../runbooks/source-refresh.md)
