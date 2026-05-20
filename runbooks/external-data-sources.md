# External Data Sources Runbook

Operational guide for loading and maintaining external reference data in the Ditteau Data platform. External sources are public or third-party datasets shared across all Ditteau schools — they live in `DITTEAU_SHARED` rather than any school-specific database.

**Last updated:** May 2026  
**Maintained by:** LVP

---

## Contents

- [A. Architecture Overview](#a-architecture-overview)
- [B. CPI Index](#b-cpi-index)
- [C. IPEDS](#c-ipeds)
- [D. Adding a New External Source](#d-adding-a-new-external-source)
- [E. Planned Sources](#e-planned-sources)
- [F. Troubleshooting](#f-troubleshooting)

---

## A. Architecture Overview

### A.1 DITTEAU_SHARED Database

External data that is identical for all schools is stored in a single shared Snowflake database rather than being duplicated per school.

| Schema | Purpose |
|--------|---------|
| `DITTEAU_SHARED.REFERENCE` | dbt-managed seed tables (small, infrequently updated) |
| `DITTEAU_SHARED.IPEDS` | NCES IPEDS survey tables (loaded via Deposit Loader) |

New external source systems should get their own schema here (e.g., `DITTEAU_SHARED.BLS`, `DITTEAU_SHARED.NSC`).

### A.2 dbt Integration

The dbt var `external_sources_database` controls where staging models and seeds read from. It defaults to `DITTEAU_SHARED` in `dbt_project.yml` and is passed explicitly in every school's run script.

```yaml
# dbt_project.yml
vars:
  external_sources_database: "DITTEAU_SHARED"
```

```bash
# scripts/run_merrimack_dev.sh (and all other school scripts)
--vars '{ ..., "external_sources_database": "DITTEAU_SHARED" }'
```

### A.3 Deposit Loader

Large external datasets (IPEDS survey files) are loaded using the Deposit Loader in `ditteau_data_infra/deposit_loader/`. Because these loads target `DITTEAU_SHARED` rather than a school database, `--school` is not required — pass `--database`, `--schema`, `--role`, and `--warehouse` explicitly instead.

```bash
# Pattern for all DITTEAU_SHARED loads
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED \
  --schema <SCHEMA> \
  --role SYSADMIN \
  --warehouse PLATFORM_WH \
  <command> <source_system> ...
```

See the [Deposit Loader README](../../ditteau_data_infra/deposit_loader/Deposit_Loader_README.md) for full command reference.

> ⚠️ **Always use `--truncate` on annual refresh loads.** The loader appends by default — omitting `--truncate` will duplicate every row and silently corrupt all downstream benchmarking outputs. Ask us how we know.

---

## B. CPI Index

### B.1 What It Is

The U.S. Bureau of Labor Statistics (BLS) Consumer Price Index for All Urban Consumers (CPI-U), annual averages. Series: `CUUR0000SA0` (All items, U.S. city average, 1982–84=100). Used to inflation-adjust financial KPIs across the 10-year historical lookback window — tuition, net tuition revenue, aid disbursements.

| | |
|--|--|
| **Source file** | `ditteau_data_transform/seeds/shared/seed_cpi_index.csv` |
| **Snowflake** | `DITTEAU_SHARED.REFERENCE.SEED_CPI_INDEX` |
| **dbt ref** | `{{ ref('seed_cpi_index') }}` |
| **Update cadence** | Annual — January |
| **Cost** | Free |

### B.2 How It Works

`seed_cpi_index` is a dbt seed — a CSV file that dbt manages directly. The `dbt_project.yml` config routes it to `DITTEAU_SHARED.REFERENCE` for all schools:

```yaml
# dbt_project.yml
seeds:
  ditteau_data_transform:
    shared:
      seed_cpi_index:
        +database: "{{ var('external_sources_database') }}"
        +schema: reference
```

Models reference it via `{{ ref('seed_cpi_index') }}` — dbt resolves this to `DITTEAU_SHARED.REFERENCE.SEED_CPI_INDEX` regardless of which school is running.

The companion macros `cpi_adjust()` and `cpi_adjust_joins()` in `macros/finance/cpi_adjust.sql` handle the join pattern. See `docs/examples/cpi_adjust_usage_example.sql` for usage.

### B.3 Annual Update (January)

BLS releases the prior-year annual average each January. Effort: ~10 minutes.

1. Get the new annual average from [bls.gov/cpi](https://www.bls.gov/cpi/) — Series `CUUR0000SA0`, Annual Data tab.

2. Append a new row to `seeds/shared/seed_cpi_index.csv`:

    ```
    2024,314.175,1982-84,CUUR0000SA0 annual average
    ```

3. Seed into `DITTEAU_SHARED.REFERENCE` (run from `ditteau_data_transform/`):

    ```bash
    PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
      bash scripts/run_merrimack_dev.sh seed --select seed_cpi_index
    ```

    Any school script works — the seed always writes to `DITTEAU_SHARED`.

4. Verify:

    ```sql
    select * from DITTEAU_SHARED.REFERENCE.SEED_CPI_INDEX
    order by calendar_year desc
    limit 3;
    ```

> **Note:** The 2025 value in the initial seed (~319.1) is a mid-year estimate. Replace it with the official BLS figure when published in January 2026.

### B.4 Downstream Models

○ `mart_financial_aid_trend` — expresses aid figures in real dollars  
○ Any financial mart with a 10-year trend view should use `cpi_adjust()`

---

## C. IPEDS

### C.1 What It Is

Integrated Postsecondary Education Data System (IPEDS), published annually by the National Center for Education Statistics (NCES). Used for peer benchmarking, compliance reporting, and Common Data Set metric validation.

**Source:** [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data)

| | |
|--|--|
| **Update cadence** | Annual — NCES releases final data November–January for the prior fall |
| **FERPA** | None — all IPEDS data is public |
| **Cost** | Free |

The three surveys we load:

| Survey | NCES File | Snowflake Table | Rows (2023) |
|--------|-----------|-----------------|-------------|
| Institutional Characteristics (HD) | `HD{YYYY}.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_INSTITUTIONAL_CHARACTERISTICS` | ~6,163 |
| Fall Enrollment Part A (EF) | `EF{YYYY}A.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_FALL_ENROLLMENT` | ~115,156 |
| Graduation Rates (GR) | `GR{YYYY}.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_GRADUATION_RATES` | ~51,368 |

### C.2 Important Data Notes

Things that aren't obvious from the NCES documentation — learned during the initial load in May 2026. Read this before touching the data.

**IPEDS unitids are permanent.**  
Once NCES assigns a unitid to an institution it never changes — even through name changes, mergers, or closures. The `seed_ipeds_peer_group.csv` file is the authoritative unitid reference for all Ditteau peer institutions. Verify unitids at [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data) → Look Up an Institution before adding new peer rows.

**The HD file has a BOM character.**  
`HD{YYYY}.csv` ships with a UTF-8 byte order mark on the first column name. Strip it before loading:

```bash
sed -i '' 's/ï»¿//' hd{YYYY}_prepped.csv
```

Verify: `head -1 hd{YYYY}_prepped.csv | cut -c1-40` should start with `survey_year,UNITID`.

**NCES renames Carnegie classification columns between vintages.**  
The 2023 HD file uses `c21basic`, `c21ipug`, `c21ipgrd`, `c21ugprf`, `c21enprf`, `c21szset` (Carnegie 2021 vintage). Earlier files used `c18basic`, `c18ipug`, etc. The `source_registry.yml` column list and `stg_ipeds__institutional_characteristics.sql` must match the actual file vintage. Check column $50 of the HD file if in doubt (see Step 3 of the [annual update](#c5-annual-update-novemberjanuaryjanuary)).

**GR file `grtype` column is VARCHAR, not integer.**  
NCES uses letter suffixes (e.g., `29A`, `31A`) for associate's and certificate-seeking cohort subtypes. The column must be typed `VARCHAR` in `source_registry.yml`. Our staging model filters to `grtype IN ('2','3','4')` for bachelor's-seeking cohorts only — the associate's rows are loaded but ignored downstream.

**The NCES file does not include a `survey_year` column.**  
The `add_yr.py` prep script (at `~/ditteau_data_inbox/IPEDS/add_yr.py`) adds it as the first column. Run this before every load. The script handles `latin-1` encoding which NCES uses.

**The Deposit Loader appends by default.**  
Always pass `--truncate` on annual refreshes or you will duplicate every row. See the warning in [Section A.3](#a3-deposit-loader).

**Known loader bug: single-quoted error messages.**  
When COPY INTO rejects rows with errors like `"Numeric value '29A' is not recognized"`, the single quotes break the post-load history `UPDATE`. The data loads correctly — only the `_LOAD_HISTORY` record fails to update. Fix is tracked in the infra repo (line 589 of `deposit_loader.py` — escape single quotes before SQL interpolation). WDT to resolve.

### C.3 How It Works in dbt

The `_ipeds_sources.yml` source definition points to `DITTEAU_SHARED.IPEDS` via the `external_sources_database` var:

```yaml
sources:
  - name: ipeds
    database: "{{ var('external_sources_database') }}"
    schema: ipeds
```

Model lineage:

```
DITTEAU_SHARED.IPEDS.*
  └── stg_ipeds__institutional_characteristics  (view, deterge/staging/ipeds)
  └── stg_ipeds__fall_enrollment               (view, deterge/staging/ipeds)
  └── stg_ipeds__graduation_rates              (view, deterge/staging/ipeds)
        └── dim_institution                    (table, distribute/dimensions)
              └── mart_ipeds_peer_comparison   (table, distribute/summaries)
```

Peer group membership is managed in `seeds/shared/seed_ipeds_peer_group.csv` and seeded per school (not to `DITTEAU_SHARED`), since peer designations are school-specific configuration. RDT owns peer group content; LVP owns the file structure.

### C.4 One-Time Snowflake Setup

Completed during initial platform build — May 2026. Documented here for reference if `DITTEAU_SHARED` ever needs to be rebuilt.

```sql
create database DITTEAU_SHARED;
create schema DITTEAU_SHARED.REFERENCE;
create schema DITTEAU_SHARED.IPEDS;
create stage DITTEAU_SHARED.IPEDS.INGEST_STAGE;
create file format DITTEAU_SHARED.IPEDS.csv_standard
    like MERRIMACK_DD_DEV.DEPOSIT.csv_standard;
```

Create the load history table:

```bash
# From ditteau_data_infra/deposit_loader/
/Users/laurievanpelt/testenv/bin/python3 - << 'EOF'
import argparse, sys
sys.argv = ['deposit_loader.py']
from deposit_loader import get_config, connect
args = argparse.Namespace(
    school=None, env="DEV", connection="dd_prod",
    database="DITTEAU_SHARED", schema="IPEDS",
    stage="INGEST_STAGE", role="SYSADMIN", warehouse="PLATFORM_WH"
)
config = get_config(args)
conn = connect(config)
cur = conn.cursor()
cur.execute("""
    CREATE TABLE IF NOT EXISTS DITTEAU_SHARED.IPEDS._LOAD_HISTORY (
        load_id             NUMBER AUTOINCREMENT PRIMARY KEY,
        source_system       VARCHAR(100)  NOT NULL,
        source_file         VARCHAR(500)  NOT NULL,
        target_table        VARCHAR(200)  NOT NULL,
        load_status         VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
        rows_loaded         NUMBER,
        rows_rejected       NUMBER,
        file_size_bytes     NUMBER,
        load_started_at     TIMESTAMP_NTZ NOT NULL DEFAULT CURRENT_TIMESTAMP(),
        load_completed_at   TIMESTAMP_NTZ,
        error_message       VARCHAR(2000),
        loaded_by           VARCHAR(100)  DEFAULT CURRENT_USER(),
        environment         VARCHAR(10)   DEFAULT 'DEV',
        notes               VARCHAR(1000)
    )
""")
print("Done")
cur.close(); conn.close()
EOF
```

Create IPEDS tables from the registry:

```bash
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  create-tables ipeds
```

### C.5 Annual Update (November–January)

NCES releases provisional data around November and final data around January. **Use final data only.** Estimated effort: ~90 minutes end to end.

**Step 1 — Download files from NCES**

Go to [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data) → **Complete Data Files** (not Custom Data Files). Download for the new survey year:

○ `HD{YYYY}.csv` — Institutional Characteristics  
○ `EF{YYYY}A.csv` — Fall Enrollment, Part A  
○ `GR{YYYY}.csv` — Graduation Rates  

>  **Always use Complete Data Files, not Custom Data Files.** Custom exports cap variables at 250 and will silently drop columns we need. Always grab Part A of the EF survey — Part B and Part C have different grain and are not used.

**Step 2 — Prep the files**

Edit the year constants at the bottom of `add_yr.py`, then run:

```bash
cd ~/ditteau_data_inbox/IPEDS
python3 add_yr.py

# Strip BOM from HD file
sed -i '' 's/ï»¿//' hd{YYYY}_prepped.csv
```

**Step 3 — Check for column changes**

NCES occasionally renames columns between vintages. Spot-check column $50 of the HD file before validating:

```bash
head -1 hd{YYYY}_prepped.csv | tr ',' '\n' | nl | sed -n '48,56p'
```

If Carnegie column names have changed (e.g., `c21*` → `c24*`), update:

○ `source_registry.yml` — `ipeds_institutional_characteristics` columns list  
○ `stg_ipeds__institutional_characteristics.sql` — column references  
○ Drop and recreate `DITTEAU_SHARED.IPEDS.IPEDS_INSTITUTIONAL_CHARACTERISTICS`  

**Step 4 — Validate headers**

```bash
cd /Users/laurievanpelt/ditteau_data_infra/deposit_loader

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  validate ipeds ipeds_institutional_characteristics \
  ~/ditteau_data_inbox/IPEDS/hd{YYYY}_prepped.csv

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  validate ipeds ipeds_fall_enrollment \
  ~/ditteau_data_inbox/IPEDS/ef{YYYY}a_prepped.csv

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  validate ipeds ipeds_graduation_rates \
  ~/ditteau_data_inbox/IPEDS/gr{YYYY}_prepped.csv
```

>  **All three must show `✓ All N columns match` before you load anything.** Do not skip validate. Do not assume last year's file layout is unchanged.

**Step 5 — Load**

```bash
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  load ipeds ipeds_institutional_characteristics \
  ~/ditteau_data_inbox/IPEDS/hd{YYYY}_prepped.csv --truncate

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  load ipeds ipeds_fall_enrollment \
  ~/ditteau_data_inbox/IPEDS/ef{YYYY}a_prepped.csv --truncate

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  load ipeds ipeds_graduation_rates \
  ~/ditteau_data_inbox/IPEDS/gr{YYYY}_prepped.csv --truncate
```

Expected row counts (stable year-over-year):

| Table | Expected Rows |
|-------|--------------|
| `IPEDS_INSTITUTIONAL_CHARACTERISTICS` | ~6,000–6,200 |
| `IPEDS_FALL_ENROLLMENT` | ~115,000–120,000 |
| `IPEDS_GRADUATION_RATES` | ~50,000–55,000 |

The GR load will report ~10,000 rejected rows — these are associate's/certificate cohort subtypes (`grtype` values like `29A`, `31A`) that the loader flags as non-numeric. The bachelor's-seeking rows (`grtype IN ('2','3','4')`) load correctly. This is expected and acceptable.

**Step 6 — Rebuild dbt models for all active schools**

```bash
cd /Users/laurievanpelt/ditteau_data_transform

PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh build \
  --select "path:models/deterge/staging/ipeds dim_institution mart_ipeds_peer_comparison"

PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_anselm_dev.sh build \
  --select "path:models/deterge/staging/ipeds dim_institution mart_ipeds_peer_comparison"
```

**Step 7 — Verify**

```sql
select count(*) from DITTEAU_SHARED.IPEDS.IPEDS_INSTITUTIONAL_CHARACTERISTICS;
select count(*) from DITTEAU_SHARED.IPEDS.IPEDS_FALL_ENROLLMENT;
select count(*) from DITTEAU_SHARED.IPEDS.IPEDS_GRADUATION_RATES;

-- Check load history
select source_file, load_status, rows_loaded, rows_rejected, load_completed_at
from DITTEAU_SHARED.IPEDS._LOAD_HISTORY
order by load_started_at desc
limit 5;

-- Sanity check: own institutions present in peer comparison
select institution_name, is_own_institution, survey_year,
       headcount_total, bach_grad_rate_150pct, peer_rank_by_grad_rate
from MERRIMACK_DD_DEV.DISTRIBUTE.MART_IPEDS_PEER_COMPARISON
where is_own_institution = true
order by survey_year desc;
```

### C.6 Managing the Peer Group Seed

The peer group is defined in `seeds/shared/seed_ipeds_peer_group.csv`. LVP owns the content (which schools are peers); LVP owns the file structure.

**When to update:**

○ A partner school requests peer group changes (add/remove institutions)  
○ A peer institution closes (unitids are permanent, but institutions do close)  
○ A new Ditteau school is onboarded — add 8–10 peer rows for the new `school_code`  

**How to update:**

1. Edit `seed_ipeds_peer_group.csv` — verify unitids at [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data) → Look Up an Institution
2. Re-seed: `dbt seed --select seed_ipeds_peer_group`
3. Re-run: `dbt run --select dim_institution mart_ipeds_peer_comparison`

> **Do not guess unitids.** Every unitid in this file was manually verified against the NCES lookup. Wrong unitids produce silently incorrect peer comparisons — no error, just wrong data.

### C.7 Downstream Models

| Model | Layer | Description |
|-------|-------|-------------|
| `stg_ipeds__institutional_characteristics` | `deterge/staging/ipeds` | HD survey — cleaned institution attributes |
| `stg_ipeds__fall_enrollment` | `deterge/staging/ipeds` | EF survey — enrollment counts by level/race/gender |
| `stg_ipeds__graduation_rates` | `deterge/staging/ipeds` | GR survey — 150% graduation rate cohort data |
| `dim_institution` | `distribute/dimensions` | Peer universe dimension — own schools + all peer institutions |
| `mart_ipeds_peer_comparison` | `distribute/summaries` | Side-by-side enrollment and grad rate benchmarks |

---

## D. Adding a New External Source

Use this checklist when onboarding a new external dataset (e.g., College Scorecard, NSC Clearinghouse, BLS OEWS, FSA Open Data).

### D.1 Snowflake Setup

```sql
-- Create a schema in DITTEAU_SHARED
create schema DITTEAU_SHARED.<SOURCE_NAME>
    comment = '<Description of source>';
create stage DITTEAU_SHARED.<SOURCE_NAME>.INGEST_STAGE;
create file format DITTEAU_SHARED.<SOURCE_NAME>.csv_standard
    like MERRIMACK_DD_DEV.DEPOSIT.csv_standard;
```

### D.2 Register in Deposit Loader

Add the source system to `ditteau_data_infra/deposit_loader/source_registry.yml`:

```yaml
<source_key>:
  source_system_label: <LABEL>
  default_file_format: csv_standard
  default_delimiter: ","
  tables:
    <table_key>:
      target_table: <table_name>
      description: >
        ...
      data_owner: INSTITUTIONAL_RESEARCH
      data_class: PUBLIC
      columns:
        - { name: <col>, type: VARCHAR }
```

>**Column types matter.** Use `VARCHAR` for any column that might contain letter suffixes or mixed types (learned from IPEDS `grtype`). Use `INTEGER` only when you are certain the column contains only integers. When in doubt, use `VARCHAR` — you can always cast downstream in the staging model.

Create the tables:

```bash
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema <SOURCE_NAME> \
  --role SYSADMIN --warehouse PLATFORM_WH \
  create-tables <source_key>
```

### D.3 Add dbt Source Definition

Create `models/deterge/staging/<source>/_<source>_sources.yml`:

```yaml
sources:
  - name: <source>
    database: "{{ var('external_sources_database') }}"
    schema: <source_name>
    tables:
      - name: <table_name>
        meta:
          source_system: <SOURCE>
          data_classification: PUBLIC
          is_pii: false
```

### D.4 Write Staging Model

Follow the IPEDS staging models as a template:

○ File: `models/deterge/staging/<source>/stg_<source>__<entity>.sql`  
○ Rename and cast all columns; no business logic in staging  
○ Add `{{ add_source_metadata() }}` macro at the end of the select  
○ Document grain, source file, and any quirks in the header comment block  

### D.5 Validate Before You Load

Always run `validate` before `load`. The validate command catches column count mismatches and name drift before any data touches Snowflake — a 30-second check that saves an hour of cleanup.

### D.6 Document Here

Add a new lettered section to this runbook covering:

○ What the data is and where it comes from  
○ Update cadence and trigger  
○ Where it lives in Snowflake  
○ Any data quirks discovered during initial load  
○ How it connects to dbt models  
○ Annual/periodic update steps with exact commands  

---

## E. Planned Sources

Next in the external data build queue per the [External Data Integration Plan](https://docs.google.com/document/d/1Fu2BDcSRhgJrUVpD4N70ziuBtYr39F9dNev7sF3Dxb8/edit?usp=sharing).

| Source | Priority | KPIs Served | Status |
|--------|----------|-------------|--------|
| College Scorecard (ED) | Tier 1 | Outcomes, earnings, debt load | Not started |
| FSA Open Data | Tier 1 | FAFSA completion rate (KPI 3 gap) | Not started |
| BLS OEWS | Tier 1 | Program ROI, post-grad earnings | Not started |
| ACS 5-year (Census) | Tier 2 | Territory analysis, first-gen proxy | Not started |
| BEA Regional Economic Data | Tier 2 | Enrollment demand modeling | Not started |
| State Commission Reports (MA/NH) | Tier 2 | State grant benchmarking | Not started |

---

## F. Troubleshooting

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| `Object 'DITTEAU_SHARED.IPEDS.*' does not exist` | School dbt role lacks `SELECT` on `DITTEAU_SHARED` | Grant `USAGE` on database/schema and `SELECT` on tables to the school's transform role |
| `No active warehouse selected` | Warehouse doesn't exist or role lacks `USAGE` | Use `PLATFORM_WH`; verify with `SHOW WAREHOUSES` |
| `Role does not exist or not authorized` | Using school-specific role for shared DB | Use `SYSADMIN` or create a dedicated `DITTEAU_SHARED_WRITE` role |
| Row counts 2x or 3x expected | Load run multiple times without `--truncate` | `DROP` and recreate table, reload with `--truncate` |
| COPY INTO loads 0 rows | Column mismatch or file format issue | Run `validate` command; check file format definition |
| `dbt ref()` resolves to wrong database | `external_sources_database` var not set | Verify var in `dbt_project.yml` and run script |
| HD file first column shows as `ï»¿UNITID` | BOM not stripped before load | Run: `sed -i '' 's/ï»¿//' hd{YYYY}_prepped.csv` |
| Column mismatch at position $50 in HD | Carnegie columns renamed in new vintage | Check actual file headers; update registry and staging model |
| Load history `UPDATE` crashes after GR load | Single-quote bug in loader line 589 | Known issue — data loads correctly; fix tracked in infra repo (WDT) |
| `KeyError: 'source_system_label'` | `source_registry.yml` uses wrong field name | Field must be `source_system_label`, not `label` |
| `TypeError: string indices must be integers` | Columns in registry are plain strings, not dicts | Each column must be `{ name: <col>, type: <TYPE> }` |
