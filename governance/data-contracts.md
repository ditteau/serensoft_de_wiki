# Data Contracts

Schema contracts and field-level governance in the Ditteau Schema Registry.

---

## Overview

The Ditteau Schema Registry (`ditteau_schema_registry`) provides a YAML-first, Git-native catalog of source system schemas. It documents every field in every source system with data governance metadata.

The registry is **upstream** of `ditteau_data_transform` — it informs dbt source definitions and staging model design.

---

## Contract Enforcement

### JSON Schema Validation

Four JSON schemas validate all YAML files:

| File Pattern | Schema |
|--------------|--------|
| `sources/*/tables/*.yml`, `sources/*/views/*.yml` | `table.schema.json` |
| `sources/*/_source_meta.yml` | `source_meta.schema.json` |
| `schools/*/_school_meta.yml` | `school_meta.schema.json` |
| `schools/*/<system>.yml` | `school_override.schema.json` |

Validation runs at CI time:

```bash
python tools/validate.py
python tools/validate.py --file sources/jenzabar_cx/tables/id_rec.yml
```

---

## Table-Level Contracts

Each table YAML includes:

```yaml
source_system: jenzabar_cx
table_name: id_rec
status: complete        # stub | partial | complete
description: "Master student identity record"
data_owner: REGISTRAR
data_class: RESTRICTED

columns:
  - name: id
    data_type: NUMBER(10,0)
    is_nullable: false
    is_pii: true
    ...
```

### Required Fields

| Field | Description |
|-------|-------------|
| `source_system` | Source system identifier |
| `table_name` | Table name in source |
| `status` | Documentation completeness |
| `description` | Human-readable description |
| `data_owner` | Responsible data owner |
| `columns` | Column definitions |

### Status Values

| Status | Meaning |
|--------|---------|
| `stub` | Auto-generated, minimal documentation |
| `partial` | Some columns documented |
| `complete` | Fully documented and reviewed |

---

## Field-Level Contracts

Each column includes governance metadata:

```yaml
columns:
  - name: ssn_last4
    data_type: CHAR(4)
    is_nullable: true
    is_pii: true
    data_class: RESTRICTED
    description: "Last 4 digits of Social Security Number"
    ditteau_canonical: ssn_last4
    known_values: []
```

### Required Column Fields

| Field | Description |
|-------|-------------|
| `name` | Column name |
| `data_type` | Source data type |
| `is_nullable` | NULL allowed |
| `is_pii` | Personally identifiable information |

### Optional Column Fields

| Field | Description |
|-------|-------------|
| `data_class` | Classification level |
| `description` | Human-readable description |
| `ditteau_canonical` | Mapping to unified model field |
| `known_values` | Enumerated values for code fields |
| `safe_col` | School-specific custom field flag |

---

## Canonical Fields

The `canonical_fields.yml` file defines the unified model vocabulary:

```yaml
canonical_fields:
  - name: student_id
    domain: student
    description: "Source-system student identifier"
    data_type: varchar(20)
    data_class: RESTRICTED
    is_pii: true

  - name: first_name
    domain: person
    description: "Person's legal first name"
    data_type: varchar(50)
    data_class: RESTRICTED
    is_pii: true
```

### Canonical Field Domains

| Domain | Scope |
|--------|-------|
| `person` | Identity, demographics |
| `student` | Student-specific attributes |
| `enrollment` | Course enrollment |
| `course` | Course catalog |
| `term` | Academic terms |
| `program` | Academic programs |
| `award` | Financial aid awards |
| `application` | Admissions applications |

### Using Canonical Mappings

In table YAML:

```yaml
columns:
  - name: FNAME
    ditteau_canonical: first_name
```

The validator confirms `first_name` exists in `canonical_fields.yml`.

---

## Data Classification

Four classification levels:

| Level | Description | Examples |
|-------|-------------|----------|
| `PUBLIC` | No restrictions | Course catalog, term dates |
| `INTERNAL` | Internal use only | Enrollment counts |
| `CONFIDENTIAL` | Sensitive business data | GPA, academic standing |
| `RESTRICTED` | Highly sensitive, regulated | SSN, DOB, financial aid |

Classification drives:
- Row access policy tiers
- Masking policy application
- Audit logging requirements

---

## Data Owners

Standard data owners:

| Owner | Responsible For |
|-------|-----------------|
| `REGISTRAR` | Student records, enrollment, grades |
| `FINANCIAL_AID` | Aid awards, ISIR, packaging |
| `BURSAR` | Billing, payments |
| `ADMISSIONS` | Applications, decisions |
| `INSTITUTIONAL_RESEARCH` | Reporting, analytics |
| `COMMON` | Shared reference data |

---

## PII Indicators

The `is_pii` flag marks personally identifiable information:

```yaml
columns:
  - name: birth_date
    is_pii: true
    data_class: RESTRICTED
```

PII fields:
- Trigger masking policy application
- Require RESTRICTED classification
- Are logged in PII access monitoring

---

## Known Values

For code/flag fields, document valid values:

```yaml
columns:
  - name: gender_code
    known_values:
      - value: "M"
        description: "Male"
      - value: "F"
        description: "Female"
      - value: "U"
        description: "Unknown/Undisclosed"
      - value: "N"
        description: "Non-binary"
```

---

## School-Specific Overrides

Schools can override field definitions:

```yaml
# schools/merrimack/jenzabar_cx.yml
overrides:
  id_rec:
    columns:
      - name: CUSTOM_FIELD_1
        description: "Merrimack-specific custom field"
        safe_col: true
```

The `safe_col: true` flag indicates school-specific custom fields that may not exist in other schools' sources.

---

## Documenting a Table

1. Open `sources/<system>/tables/<table>.yml`
2. Replace `description: "# TODO"` with real descriptions
3. Add `ditteau_canonical` for unified model mappings
4. Add `known_values` for code/flag fields
5. Set `status: partial` or `complete`
6. Run `python tools/validate.py --file <path>`

---

## Stub Generators

Auto-generate stubs from source documentation:

```bash
# Jenzabar CX (from Informix syscolumns)
python tools/parse_cx_listing.py --input CXdblisting.txt

# Jenzabar One (from MSSQL Excel)
python tools/parse_j1_listing.py --input "J1 Data Dictionary.xlsx"

# Slate (from OpenQuery Excel)
python tools/parse_slate_listing.py --input "Slate Data Dictionary.xlsx"

# PowerFAIDS (from MSSQL Excel)
python tools/parse_powerfaids_listing.py --input "PFAIDS data dictionary.xlsx"
```

---

## Registry Statistics

| Source System | Tables | Status |
|---------------|--------|--------|
| Jenzabar CX | 1,483 | Mostly stubs |
| Jenzabar One | 2,679 | Mostly stubs |
| PowerFAIDS | 544 | Mostly stubs |
| Slate | 311 | Mostly stubs |
| Workday Student | 2 | Partial |
| Banner | Varies | Mostly stubs |

---

## Related Documentation

- [dbt Conventions](../dbt/conventions.md) — How contracts inform staging models
- [DQ Framework](dq-framework.md) — Data quality rules
- [Governance Overview](governance-and-production-readiness-review.md)
