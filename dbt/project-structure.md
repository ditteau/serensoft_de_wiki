# dbt Project Structure

Directory layout and organization for the `ditteau_data_transform` dbt project.

---

## Directory Overview

```
ditteau_data_transform/
├── models/
│   ├── deposit/              # Source references only (no transformations)
│   ├── deterge/              # Silver layer
│   │   ├── staging/          # 1:1 source conformance
│   │   │   ├── jenzabar_cx/
│   │   │   ├── jenzabar_one/
│   │   │   ├── workday/
│   │   │   ├── banner/
│   │   │   ├── slate/
│   │   │   ├── powerfaids/
│   │   │   └── ipeds/
│   │   └── intermediate/     # Cross-source joins
│   └── distribute/           # Gold layer
│       ├── dimensions/       # Dimension tables
│       ├── facts/            # Fact tables
│       └── marts/            # Business marts
│           ├── enrollment/
│           ├── financial_aid/
│           ├── admissions/
│           ├── academic/
│           └── registration/
├── macros/
│   ├── utils/                # Utility macros
│   ├── schema/               # Schema generation
│   ├── metadata/             # Metadata column macros
│   ├── governance/           # Row access policies, masking
│   └── audit/                # Logging macros
├── seeds/
│   ├── shared/               # Cross-school seeds
│   └── {school_code}/        # School-specific seeds
├── snapshots/                # SCD Type 2 snapshots
├── tests/
│   ├── generic/              # Reusable test definitions
│   └── governance/           # Governance assertion tests
├── scripts/                  # Run scripts and utilities
├── streamlit/                # Streamlit dashboards
└── docs/                     # Additional documentation
```

---

## Models Organization

### Deposit Layer

The Deposit layer contains **source definitions only** — no transformations occur here. Data arrives via:

- Snowflake Data Share (Jenzabar CX)
- CSV files via deposit_loader (Jenzabar One, PowerFAIDS, Slate, Workday, Banner)
- External loads to `DITTEAU_SHARED` (IPEDS, College Scorecard)

### Deterge Layer: Staging

One staging model per source table. Each model:

- Renames columns to snake_case
- Casts to appropriate types
- Adds the 5 required metadata columns
- Applies no business logic

```
models/deterge/staging/jenzabar_cx/
├── _jcx_sources.yml          # Source definitions
├── _jcx_models.yml           # Model documentation
├── stg_jcx__id_rec.sql       # Student identity
├── stg_jcx__stu_acad_rec.sql # Academic records
└── ...
```

### Deterge Layer: Intermediate

Cross-source integration and business logic:

```
models/deterge/intermediate/
├── int_ditteau_id_registry.sql   # Master identity resolution
├── int_student_race.sql          # Race/ethnicity from multiple sources
├── int_program_crosswalk.sql     # Program code mapping
└── ...
```

### Distribute Layer

Analytics-ready dimensional models:

```
models/distribute/
├── dimensions/
│   ├── dim_student.sql           # Student dimension (SCD Type 2)
│   ├── dim_term.sql              # Academic term dimension
│   ├── dim_program.sql           # Academic program dimension
│   └── dim_date.sql              # Date dimension
├── facts/
│   ├── fact_enrollment.sql       # Enrollment transactions
│   ├── fact_application.sql      # Admissions applications
│   ├── fact_aid_award.sql        # Financial aid awards
│   └── fact_student_term.sql     # Student-term snapshots
└── marts/
    ├── enrollment/
    │   ├── mart_enrollment_census.sql
    │   └── mart_retention_cohort.sql
    ├── financial_aid/
    │   └── mart_financial_aid_trend.sql
    └── admissions/
        └── mart_admissions_funnel.sql
```

---

## YAML File Conventions

### Source Definitions

Located with staging models, prefixed with underscore:

```yaml
# models/deterge/staging/jenzabar_cx/_jcx_sources.yml
version: 2

sources:
  - name: jenzabar_cx
    database: "{{ var('jcx_share_database') }}"
    schema: "{{ var('jcx_share_schema') }}"
    tables:
      - name: id_rec
        description: Student identity records
        columns:
          - name: id
            description: Student ID
```

### Model Documentation

Separate YAML files for model metadata:

```yaml
# models/deterge/staging/jenzabar_cx/_jcx_models.yml
version: 2

models:
  - name: stg_jcx__id_rec
    description: Staged student identity from Jenzabar CX
    config:
      tags: ['staging', 'jenzabar_cx']
    columns:
      - name: student_id
        description: Primary student identifier
        tests:
          - not_null
          - unique
```

---

## Seeds

### Shared Seeds

Cross-school reference data in `seeds/shared/`:

| Seed | Purpose |
|------|---------|
| `dim_gender_seed` | Gender code lookup |
| `dim_state_seed` | US state lookup |
| `seed_aid_fund_crosswalk` | Financial aid fund mapping |
| `seed_cpi_index` | CPI inflation adjustment factors |
| `seed_persona_domain_grid` | Governance tier assignments |
| `seed_valid_role_types` | Valid Snowflake role types |

### School-Specific Seeds

School-specific overrides in `seeds/{school_code}/`:

```
seeds/
├── anselm/
│   ├── seed_anselm_wd_program_of_study.csv
│   ├── seed_anselm_ipeds_peer_group.csv
│   └── seed_anselm_retention_policy.csv
├── merrimack/
│   ├── seed_merrimack_ipeds_peer_group.csv
│   └── seed_merrimack_retention_policy.csv
└── shared/
    └── ...
```

School seeds are enabled via `dbt_project.yml`:

```yaml
seeds:
  ditteau_data_transform:
    anselm:
      +enabled: "{{ var('school_code') == 'ANSELM' }}"
```

---

## Run Scripts

School-specific configuration is encapsulated in run scripts:

| Script | School | Target |
|--------|--------|--------|
| `scripts/run_merrimack_dev.sh` | Merrimack | `merrimack_dev` |
| `scripts/run_anselm_dev.sh` | Saint Anselm | `anselm_dev` |
| `scripts/run_demeau_dev.sh` | DEMEAU (demo) | `demeau_dev` |
| `scripts/run_demeau.sh` | DEMEAU | Env-aware (`dev`/`test`/`prod`) |

**Always use run scripts** rather than passing vars inline — they are the source of truth for each school's configuration.

Usage:

```bash
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh build --select mart_enrollment_census
```

---

## Environments and Databases

Databases follow the pattern `{SCHOOL_CODE}_DD_{ENV}`:

| School | DEV | TEST | PROD |
|--------|-----|------|------|
| Merrimack | `MERRIMACK_DD_DEV` | `MERRIMACK_DD_TEST` | `MERRIMACK_DD_PROD` |
| Saint Anselm | `ANSELM_DD_DEV` | `ANSELM_DD_TEST` | `ANSELM_DD_PROD` |
| DEMEAU | `DEMEAU_DD_DEV` | `DEMEAU_DD_TEST` | `DEMEAU_DD_PROD` |

Schemas are consistent across environments:
- `deposit` — Raw source data
- `deterge` — Cleaned/transformed data
- `distribute` — Analytics-ready models
- `governance` — Access control objects

---

## Profile Configuration

Profiles are defined in `~/.dbt/profiles.yml` with nine targets:

- `merrimack_dev`, `merrimack_test`, `merrimack_prod`
- `anselm_dev`, `anselm_test`, `anselm_prod`
- `demeau_dev`, `demeau_test`, `demeau_prod`

TEST and PROD targets authenticate as service users (`SVC_{SCHOOL}_DBT_{ENV}`) with per-user key pairs.

---

## Related Documentation

- [Conventions](conventions.md) — Naming, materializations, tags
- [Testing Strategy](testing.md) — Test patterns
- [Macros Reference](macros.md) — Available macros
