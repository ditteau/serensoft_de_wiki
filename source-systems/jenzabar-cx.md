# Jenzabar CX

Student Information System (SIS) built on Informix database.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Jenzabar |
| Product | CX (formerly CARS) |
| Database | Informix |
| Schools | Saint Anselm (archive), Colby (archive), Endicott (archive), Merrimack (archive), DEMEAU |
| Status | Legacy — most schools have migrated to other SIS |

Jenzabar CX is an older SIS that many schools are migrating away from. However, historical data remains critical for longitudinal reporting.

---

## Connection Method

**Snowflake Data Share** via ORGDATACLOUD

```
Database: ORGDATACLOUD$INTERNAL$<SCHOOL>_CX_DATA
```

For Merrimack, an archive copy exists:
```
Database: MERRIMACK_CX_ARCHIVE
Schema: DITTEAU_ARCHIVE
```

The Data Share provides near-real-time access without ETL infrastructure.

---

## Key Tables

### Identity & Demographics

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `ID_REC` | Master student record | `ID`, `FNAME`, `LNAME`, `DOB`, `GENDER` |
| `PROFILE_REC` | Extended demographics | `ETHNIC`, `CTZNSHIP`, `STATE` |
| `AA_REC` | Academic advisors | `ID`, `AA` (advisor code), `BEG_DATE` |

### Academic Records

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `STU_ACAD_REC` | Academic standing | `ID`, `ACAD_STAT`, `GPA` |
| `STU_SERV_REC` | Student services | `ID`, `SESS`, `STAT` |
| `PROG_ENR_REC` | Program enrollment | `ID`, `PROG`, `MAJOR1`, `CONC1` |
| `ACAD_CAL_REC` | Academic calendar | `SESS`, `YR`, `BEG_DATE`, `END_DATE` |

### Enrollment

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `STU_CRS_REC` | Course enrollments | `ID`, `CRS_NO`, `GRADE`, `HRS_ATTEMPT` |
| `CRS_REC` | Course catalog | `CRS_NO`, `TITLE`, `DEPT`, `CREDIT` |
| `SEC_REC` | Course sections | `CRS_NO`, `SEC_NO`, `YR`, `SESS` |

### Financial

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `STU_BILL_REC` | Student billing | `ID`, `ITEM`, `AMT` |
| `GLA_REC` | General ledger | `ACCT`, `AMT`, `TRANS_DATE` |

---

## Schema Notes

### Field Quirks

- **Character padding**: Many fields are padded with spaces (e.g., `SESS = 'FA  '`)
- **Null handling**: Empty strings used instead of NULL in many places
- **Date formats**: Mix of DATE and VARCHAR date storage
- **Case sensitivity**: Data is often uppercase

### Code Tables

Many fields use code lookups. Common patterns:
- `STAT` values: `A` (active), `I` (inactive), `G` (graduated), `W` (withdrawn)
- `GENDER` values: `M`, `F`, `U` (unknown)
- `SESS` values: `FA`, `SP`, `SU`, `WI` (padded to 4 chars)

### Term/Session Structure

```
YR   = 4-digit year (e.g., 2024)
SESS = 2-char session code + padding (e.g., 'FA  ')
SUBSESS = Sub-session code (often blank)
```

---

## Registry Statistics

From `ditteau_schema_registry`:

| Metric | Count |
|--------|-------|
| Tables | 1,483 |
| Views | 35 |
| Status | Most are stubs |

---

## Staging Models

Located in `models/deterge/staging/jenzabar_cx/`:

```
stg_jcx__id_rec.sql
stg_jcx__profile_rec.sql
stg_jcx__stu_acad_rec.sql
stg_jcx__prog_enr_rec.sql
stg_jcx__stu_crs_rec.sql
...
```

### Source Definition

```yaml
# _jcx_sources.yml
sources:
  - name: jenzabar_cx
    database: "{{ var('jcx_share_database') }}"
    schema: "{{ var('jcx_share_schema') }}"
    tables:
      - name: id_rec
      - name: profile_rec
      ...
```

### Var Guard

Models are enabled with:

```sql
{{ config(enabled=var('has_jenzabar_cx', false)) }}
```

---

## Common Transformations

### Trim Character Padding

```sql
TRIM(SESS) AS term_session
```

### Handle Empty Strings as NULL

```sql
NULLIF(TRIM(ETHNIC), '') AS ethnicity_code
```

### Parse Term Codes

```sql
YR || '-' || TRIM(SESS) AS term_code
```

---

## Data Quality Issues

| Issue | Mitigation |
|-------|------------|
| Padded fields | TRIM() in staging |
| Empty vs NULL | NULLIF(TRIM(...), '') |
| Mixed case | UPPER() or LOWER() standardization |
| Legacy codes | Code translation seeds |
| Missing dates | Sentinel dates (1900-01-01) |

---

## Related Documentation

- [Source Systems Overview](README.md)
- [dbt Conventions](../dbt/conventions.md)
- [Data Flow](../architecture/data-flow.md)
