# Slate CRM

Admissions and enrollment management platform from Technolutions.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Technolutions |
| Product | Slate |
| Platform | Cloud (SaaS) |
| Schools | DEMEAU (demo) |
| Status | Active in demo environment |

Slate is the leading CRM platform for higher education admissions, used for recruitment, applications, and enrollment management.

---

## Connection Method

**CSV Deposit** via deposit_loader

Slate exports are loaded to the DEPOSIT schema:

```bash
python deposit_loader.py --school DEMEAU load slate applications ~/inbox/slate/applications.csv
```

---

## Registry Statistics

From `ditteau_schema_registry`:

| Metric | Count |
|--------|-------|
| Tables | 311 |

---

## Key Tables

### Person & Identity

| Table | Purpose |
|-------|---------|
| `PERSON` | Master person record |
| `PERSON_NAME` | Name history |
| `PERSON_EMAIL` | Email addresses |
| `PERSON_ADDRESS` | Addresses |
| `PERSON_PHONE` | Phone numbers |

### Applications

| Table | Purpose |
|-------|---------|
| `APPLICATION` | Application records |
| `APPLICATION_STATUS` | Status history |
| `APPLICATION_CHECKLIST` | Required documents |
| `APPLICATION_DECISION` | Admission decisions |

### Lookups

| Table | Purpose |
|-------|---------|
| `LOOKUP_ROUND` | Application rounds |
| `LOOKUP_ACADEMIC_YEAR` | Academic years |
| `LOOKUP_PROGRAM` | Academic programs |
| `LOOKUP_TERM` | Entry terms |

---

## Schema Notes

### Dot-Notation Table Names

Slate uses dot-notation in table names (e.g., `APPLICATION.BIN`, `BOT.INTENT`).

**Important:** These are automatically sanitized to underscores during table creation:
- `APPLICATION.BIN` → `APPLICATION_BIN`
- `BOT.INTENT` → `BOT_INTENT`

The registry preserves original names for documentation.

### UUID Foreign Keys

Slate uses UUIDs for foreign keys:
- `academic_year` field stores a UUID FK to `LOOKUP_ACADEMIC_YEAR`
- `round` field stores a UUID FK to `LOOKUP_ROUND`

### Term Resolution

Application term routing goes through lookup tables:
```
APPLICATION.round → LOOKUP_ROUND.id → LOOKUP_ACADEMIC_YEAR
```

---

## Staging Models

Located in `models/deterge/staging/slate/`:

```
stg_slate__person.sql
stg_slate__application.sql
stg_slate__lookup__round.sql
stg_slate__lookup__academic__term.sql
...
```

### Source Definition

```yaml
# _slate_sources.yml
sources:
  - name: slate
    database: "{{ target.database }}"
    schema: deposit
    tables:
      - name: person
      - name: application
      ...
```

### Var Guard

```sql
{{ config(enabled=var('has_slate', false)) }}
```

---

## Current Status

**DEMEAU only as of September 2026.**

Previous Merrimack Slate data was discovered to be Faker-generated test data, not real source data. The flags were corrected:

```yaml
# Merrimack/Anselm
has_slate: false

# DEMEAU only
has_slate: true
```

---

## Data Quality Notes

### DEMEAU Fixture Limitations

The DEMEAU Slate fixture is generated from the models it tests:
- `emit_slate.py` sets `application.round` to the academic-term ID
- `SLT_LOOKUP_ROUND` and `SLT_LOOKUP_ACADEMIC_YEAR` are empty

A fixture derived from the model it tests cannot fully test that model. Real Slate data routes terms through lookup tables.

---

## Data Loading

### Create Tables

```bash
python deposit_loader.py --school DEMEAU create-tables slate
```

Note: Dot-notation names are automatically converted to underscores.

### Load Data

```bash
python deposit_loader.py --school DEMEAU load slate person person.csv
python deposit_loader.py --school DEMEAU load-all slate ~/inbox/slate/
```

---

## Common Transformations

### Join Through Lookups

```sql
SELECT
  a.application_id,
  r.round_name,
  ay.academic_year
FROM {{ source('slate', 'application') }} a
LEFT JOIN {{ source('slate', 'lookup_round') }} r
  ON a.round = r.id
LEFT JOIN {{ source('slate', 'lookup_academic_year') }} ay
  ON r.academic_year_id = ay.id
```

### Handle Table Name Sanitization

Table names with dots become underscores:
```sql
-- In Snowflake
FROM {{ source('slate', 'application_bin') }}  -- Was APPLICATION.BIN
```

---

## Related Documentation

- [Source Systems Overview](README.md)
- [Source Refresh Runbook](../runbooks/source-refresh.md)
- [Admissions Funnel](../architecture/data-flow.md)
