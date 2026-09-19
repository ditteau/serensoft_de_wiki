# PowerFAIDS

Financial aid management system from Ellucian.

---

## Overview

| Attribute | Value |
|-----------|-------|
| Vendor | Ellucian (originally PowerFAIDS Inc.) |
| Product | PowerFAIDS |
| Database | Microsoft SQL Server |
| Schools | DEMEAU (demo) |
| Status | Active in demo environment |

PowerFAIDS is a financial aid management system widely used in higher education for need analysis, packaging, and disbursement.

---

## Connection Method

**CSV Deposit** via deposit_loader

PowerFAIDS exports are loaded to the DEPOSIT schema:

```bash
python deposit_loader.py --school DEMEAU load powerfaids awards ~/inbox/pf/awards.csv
```

---

## Registry Statistics

From `ditteau_schema_registry`:

| Metric | Count |
|--------|-------|
| Tables | 544 |

---

## Key Tables

### Student & Demographics

| Table | Purpose |
|-------|---------|
| `STUDENTS` | Student master record |
| `STUDENT_DEMOGRAPHICS` | Demographic information |
| `STUDENT_ADDRESSES` | Address records |

### Financial Aid

| Table | Purpose |
|-------|---------|
| `AWARDS` | Aid awards |
| `AWARD_PERIODS` | Award period definitions |
| `FUND_SOURCES` | Fund sources and types |
| `PACKAGES` | Award packages |
| `DISBURSEMENTS` | Disbursement records |

### Need Analysis

| Table | Purpose |
|-------|---------|
| `ISIR` | ISIR (FAFSA) records |
| `EFC` | Expected Family Contribution |
| `BUDGET` | Cost of attendance budgets |

### Lookups

| Table | Purpose |
|-------|---------|
| `AWARD_TYPES` | Award type codes |
| `FUND_CODES` | Fund code definitions |
| `AID_YEARS` | Aid year definitions |

---

## Schema Notes

### Aid Year Structure

PowerFAIDS uses aid years (AY) aligned to federal regulations:
- AY 2024-2025 starts July 1, 2024
- ISIR data is year-specific

### Fund Hierarchy

```
FUND_SOURCES
  └─ FUND_CODES
       └─ AWARDS
            └─ DISBURSEMENTS
```

### Amount Fields

Financial amounts are stored as:
- Decimal/numeric types
- Positive values (credits)
- NULL for no award

---

## Staging Models

Located in `models/deterge/staging/powerfaids/`:

```
stg_pf__students.sql
stg_pf__awards.sql
stg_pf__isir.sql
...
```

### Source Definition

```yaml
# _pf_sources.yml
sources:
  - name: powerfaids
    database: "{{ target.database }}"
    schema: deposit
    tables:
      - name: students
      - name: awards
      ...
```

### Var Guard

```sql
{{ config(enabled=var('has_powerfaids', false)) }}
```

---

## Current Status

**DEMEAU only as of September 2026.**

Previous Merrimack PowerFAIDS data was discovered to be Faker-generated test data. The flags were corrected:

```yaml
# Merrimack/Anselm
has_powerfaids: false

# DEMEAU only
has_powerfaids: true
```

---

## Data Loading

### Create Tables

```bash
python deposit_loader.py --school DEMEAU create-tables powerfaids
```

### Load Data

```bash
python deposit_loader.py --school DEMEAU load powerfaids awards awards.csv
python deposit_loader.py --school DEMEAU load-all powerfaids ~/inbox/pf/
```

---

## Common Transformations

### Aid Year Mapping

```sql
-- Map aid year to academic year
CASE
  WHEN aid_year = '2425' THEN '2024-2025'
  WHEN aid_year = '2324' THEN '2023-2024'
  ...
END AS academic_year
```

### Fund Crosswalk

The `seed_aid_fund_crosswalk` seed maps PowerFAIDS fund codes to standard categories:

```sql
SELECT
  a.award_id,
  a.amount,
  c.federal_category,
  c.aid_type
FROM {{ ref('stg_pf__awards') }} a
LEFT JOIN {{ ref('seed_aid_fund_crosswalk') }} c
  ON a.fund_code = c.source_fund_code
```

### CPI Adjustment

Use `cpi_adjust()` macro for inflation-adjusted comparisons:

```sql
{{ cpi_adjust('award_amount', 'award_year', 2024) }} AS award_amount_2024_dollars
```

---

## Governance Notes

Financial aid data is classified as RESTRICTED:
- Domain: `financial_aid`
- Row access policy: `rap_financial_aid`
- Masking: `FINANCIAL_AMOUNT` class for dollar amounts

---

## Related Documentation

- [Source Systems Overview](README.md)
- [Source Refresh Runbook](../runbooks/source-refresh.md)
- [Financial Aid Trend Mart](../architecture/data-flow.md)
