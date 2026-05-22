# External Data Sources Runbook

Operational guide for loading and maintaining external reference data in the Ditteau Data platform. External sources are public or third-party datasets shared across all Ditteau schools — they live in `DITTEAU_SHARED` rather than any school-specific database.

**Last updated:** May 2026
**Maintained by:** LVP

---

## Contents

- [A. Architecture Overview](#a-architecture-overview)
- [B. CPI Index](#b-cpi-index)
- [C. IPEDS](#c-ipeds)
- [D. College Scorecard](#d-college-scorecard)
- [E. Adding a New External Source](#e-adding-a-new-external-source)
- [F. Planned Sources](#f-planned-sources)
- [G. Troubleshooting](#g-troubleshooting)

---

## A. Architecture Overview

### A.1 DITTEAU_SHARED Database

External data that is identical for all schools is stored in a single shared Snowflake database rather than being duplicated per school.

| Schema | Purpose |
|--------|---------|
| `DITTEAU_SHARED.REFERENCE` | dbt-managed seed tables (small, infrequently updated) |
| `DITTEAU_SHARED.IPEDS` | NCES IPEDS survey tables (loaded via Deposit Loader) |
| `DITTEAU_SHARED.COLLEGE_SCORECARD` | ED College Scorecard field-of-study data |

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

Large external datasets are loaded using the Deposit Loader in `ditteau_data_infra/deposit_loader/`. Because these loads target `DITTEAU_SHARED` rather than a school database, `--school` is not required — pass `--database`, `--schema`, `--role`, and `--warehouse` explicitly instead.

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
seeds:
  ditteau_data_transform:
    shared:
      seed_cpi_index:
        +database: "{{ var('external_sources_database') }}"
        +schema: reference
```

Models reference it via `{{ ref('seed_cpi_index') }}`. The companion macros `cpi_adjust()` and `cpi_adjust_joins()` in `macros/finance/cpi_adjust.sql` handle the join pattern. See `docs/examples/cpi_adjust_usage_example.sql` for usage.

### B.3 Annual Update (January)

BLS releases the prior-year annual average each January. Effort: ~10 minutes.

1. Get the new annual average from [bls.gov/cpi](https://www.bls.gov/cpi/) — Series `CUUR0000SA0`, Annual Data tab.

2. Append a new row to `seeds/shared/seed_cpi_index.csv`:

    ```
    2024,314.175,1982-84,CUUR0000SA0 annual average
    ```

3. Seed into `DITTEAU_SHARED.REFERENCE`:

    ```bash
    PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
      bash scripts/run_merrimack_dev.sh seed --select seed_cpi_index
    ```

4. Verify:

    ```sql
    select * from DITTEAU_SHARED.REFERENCE.SEED_CPI_INDEX
    order by calendar_year desc limit 3;
    ```

> **Note:** The 2025 value in the initial seed (~319.1) is a mid-year estimate. Replace it with the official BLS figure when published in January 2026.

### B.4 Downstream Models

○ `mart_financial_aid_trend` — expresses aid figures in real dollars
○ Any financial mart with a 10-year trend view should use `cpi_adjust()`

---

## C. IPEDS

### C.1 What It Is

Integrated Postsecondary Education Data System (IPEDS), published annually by NCES. Used for peer benchmarking, compliance reporting, and Common Data Set metric validation.

**Source:** [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data)

| | |
|--|--|
| **Update cadence** | Annual — NCES releases final data November–January |
| **FERPA** | None — all IPEDS data is public |
| **Cost** | Free |

| Survey | NCES File | Snowflake Table | Rows (2023) |
|--------|-----------|-----------------|-------------|
| Institutional Characteristics (HD) | `HD{YYYY}.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_INSTITUTIONAL_CHARACTERISTICS` | ~6,163 |
| Fall Enrollment Part A (EF) | `EF{YYYY}A.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_FALL_ENROLLMENT` | ~115,156 |
| Graduation Rates (GR) | `GR{YYYY}.csv` | `DITTEAU_SHARED.IPEDS.IPEDS_GRADUATION_RATES` | ~51,368 |

### C.2 Important Data Notes

Things that aren't obvious from the NCES documentation — learned during the initial load in May 2026. Read this before touching the data.

**IPEDS unitids are permanent.**
Once NCES assigns a unitid it never changes — even through name changes, mergers, or closures. Verify unitids at [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data) → Look Up an Institution before adding new peer rows. Do not guess.

**The HD file has a BOM character.**
Strip it before loading:

```bash
sed -i '' 's/ï»¿//' hd{YYYY}_prepped.csv
```

Verify: `head -1 hd{YYYY}_prepped.csv | cut -c1-40` should start with `survey_year,UNITID`.

**NCES renames Carnegie classification columns between vintages.**
The 2023 HD file uses `c21basic`, `c21ipug`, etc. (Carnegie 2021 vintage). Earlier files used `c18basic`. Check column $50 of the HD file each year:

```bash
head -1 hd{YYYY}_prepped.csv | tr ',' '\n' | nl | sed -n '48,56p'
```

**GR file `grtype` column is VARCHAR, not integer.**
NCES uses letter suffixes (e.g., `29A`, `31A`) for associate's cohort subtypes. Type as `VARCHAR` in registry. Our staging model filters to `grtype IN ('2','3','4')` for bachelor's only.

**The NCES file does not include a `survey_year` column.**
`add_yr.py` (at `~/ditteau_data_inbox/IPEDS/`) adds it. Run before every load.

**The Deposit Loader appends by default.**
Always pass `--truncate` on annual refreshes.

**Known loader bug: single-quoted error messages.**
When COPY INTO rejects rows with errors like `"Numeric value '29A' is not recognized"`, the single quotes break the post-load history `UPDATE`. Data loads correctly — only the `_LOAD_HISTORY` record fails. Fix tracked in infra repo (line 589, WDT to resolve).

### C.3 How It Works in dbt

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

Peer group membership is managed in `seeds/shared/seed_ipeds_peer_group.csv` and seeded per school. RDT owns peer group content; LVP owns the file structure.

### C.4 One-Time Snowflake Setup

Completed May 2026. Documented for reference if `DITTEAU_SHARED` ever needs to be rebuilt.

```sql
create database DITTEAU_SHARED;
create schema DITTEAU_SHARED.REFERENCE;
create schema DITTEAU_SHARED.IPEDS;
create stage DITTEAU_SHARED.IPEDS.INGEST_STAGE;
create file format DITTEAU_SHARED.IPEDS.csv_standard
    like MERRIMACK_DD_DEV.DEPOSIT.csv_standard;
```

Create load history table and IPEDS tables per the original build notes.

### C.5 Annual Update (November–January)

Use final NCES data only (not provisional). Estimated effort: ~90 minutes.

**Step 1 — Download files from NCES**

Go to [nces.ed.gov/ipeds/use-the-data](https://nces.ed.gov/ipeds/use-the-data) → **Complete Data Files**. Download:

○ `HD{YYYY}.csv` — Institutional Characteristics
○ `EF{YYYY}A.csv` — Fall Enrollment, Part A
○ `GR{YYYY}.csv` — Graduation Rates

> ⚠️ Always use **Complete Data Files**, not Custom Data Files. Always grab **Part A** of EF — Parts B and C have different grain.

**Step 2 — Prep the files**

```bash
cd ~/ditteau_data_inbox/IPEDS
python3 add_yr.py        # edit year constants first

sed -i '' 's/ï»¿//' hd{YYYY}_prepped.csv
```

**Step 3 — Check for column changes**

```bash
head -1 hd{YYYY}_prepped.csv | tr ',' '\n' | nl | sed -n '48,56p'
```

If Carnegie columns have changed, update `source_registry.yml` and `stg_ipeds__institutional_characteristics.sql`, then drop and recreate the HD table.

**Step 4 — Validate headers**

> ⚠️ All three must show `✓ All N columns match` before loading anything.

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

Expected row counts:

| Table | Expected |
|-------|---------|
| `IPEDS_INSTITUTIONAL_CHARACTERISTICS` | ~6,000–6,200 |
| `IPEDS_FALL_ENROLLMENT` | ~115,000–120,000 |
| `IPEDS_GRADUATION_RATES` | ~50,000–55,000 |

The GR load will report ~10,000 rejected rows (associate's cohort subtypes like `29A`). This is expected — bachelor's rows load correctly.

**Step 6 — Rebuild dbt models**

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

select source_file, load_status, rows_loaded, rows_rejected, load_completed_at
from DITTEAU_SHARED.IPEDS._LOAD_HISTORY
order by load_started_at desc limit 5;

-- Sanity check
select institution_name, is_own_institution, survey_year,
       headcount_total, bach_grad_rate_150pct, peer_rank_by_grad_rate
from MERRIMACK_DD_DEV.DISTRIBUTE.MART_IPEDS_PEER_COMPARISON
where is_own_institution = true
order by survey_year desc;
```

### C.6 Managing the Peer Group Seed

The peer group is defined in `seeds/shared/seed_ipeds_peer_group.csv`. RDT owns content; LVP owns file structure.

**When to update:** school requests peer changes, a peer institution closes, or a new Ditteau school is onboarded.

**How to update:**
1. Edit `seed_ipeds_peer_group.csv` — verify unitids at NCES lookup
2. `dbt seed --select seed_ipeds_peer_group`
3. `dbt run --select dim_institution mart_ipeds_peer_comparison`

> ⚠️ Do not guess unitids. Wrong unitids produce silently incorrect peer comparisons.

### C.7 Downstream Models

| Model | Layer | Description |
|-------|-------|-------------|
| `stg_ipeds__institutional_characteristics` | `deterge/staging/ipeds` | HD survey — cleaned institution attributes |
| `stg_ipeds__fall_enrollment` | `deterge/staging/ipeds` | EF survey — enrollment counts by level/race/gender |
| `stg_ipeds__graduation_rates` | `deterge/staging/ipeds` | GR survey — 150% graduation rate cohort data |
| `dim_institution` | `distribute/dimensions` | Peer universe dimension |
| `mart_ipeds_peer_comparison` | `distribute/summaries` | Side-by-side enrollment and grad rate benchmarks |

---

## D. College Scorecard

### D.1 What It Is

The U.S. Department of Education College Scorecard field-of-study dataset. Provides program-level post-graduation earnings, federal debt load, and loan repayment behavior for all Title IV-participating institutions. Used for program ROI analysis, peer program benchmarking, debt load context for `snap_aid_term`, and CDR compliance monitoring.

**Source:** [collegescorecard.ed.gov/data/](https://collegescorecard.ed.gov/data/)

| | |
|--|--|
| **Update cadence** | Annual — ED releases October–November |
| **FERPA** | None — all Scorecard data is public. Cells with n<30 are suppressed by ED before release. |
| **Cost** | Free |

| Snowflake Table | Source File | Rows (2024 release) |
|-----------------|-------------|---------------------|
| `DITTEAU_SHARED.COLLEGE_SCORECARD.SCORECARD_FIELD_OF_STUDY` | `Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv` | ~220,960 |

Companion seed:

○ `seeds/shared/seed_scorecard_opeid_crosswalk.csv` → `DITTEAU_SHARED.REFERENCE`
○ Maps 8-digit FSA OPEID codes to IPEDS UNITIDs — enables joins between federal financial aid data (OPEID-keyed) and IPEDS data (UNITID-keyed)

### D.2 Important Data Notes

Learned during the initial load in May 2026. Read this before touching the data.

**ED appends national rollup rows at the end of the file.**
These rows have `UNITID='NA'` — they are aggregate statistics across all institutions, not real schools. They must be stripped before loading. Always use the `_clean` file, never the raw `_prepped` file:

```bash
awk -F',' 'NR==1 || $2!="NA"' Most-Recent-Cohorts-Field-of-Study_prepped.csv \
  > Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv
```

**`CREDLEV` contains `'NA'` values and `credlev=99`.**
`credlev='NA'` appears in the national rollup rows (stripped by the `_clean` step). `credlev=99` means "Not Classified" — real rows that load fine as `VARCHAR`. The staging model guards against both.

**`CONTROL` is text, not integer.**
Values are `"Public"`, `"Private nonprofit"`, `"Private for-profit"` — not the integer 1/2/3 used in IPEDS. Typed `VARCHAR` in the registry.

**`CIPCODE` is 4-digit format.**
Scorecard uses `"0109"` not the 6-digit `"09.0101"` format. May require format normalization when joining to `dim_program.cip_code`.

**Four null sentinels, not one.**
ED uses `NULL`, `PrivacySuppressed`, `NA`, and `PS` to indicate suppressed or not-applicable cells. All four are handled by the `sc_num()` macro in the staging model — they all become SQL `NULL`.

**The file does not include a `data_year` column.**
`prep_scorecard.py` adds it as the first column. Always run the prep script before loading. Use the ending year of the academic window (e.g., `2024` for the Most-Recent-Cohorts file).

**`BBRR` columns are loan repayment behavior rates, not completion rates.**
`bbrr1_fed_comp_makeprog`, `bbrr4_fed_comp_dflt` etc. track what borrowers are doing with their loans 1–4 years after entering repayment. They are not graduation or program completion metrics.

**The national benchmark columns are very useful.**
`earn_mdn_4yr_nat`, `earn_p25_4yr_nat`, `earn_p75_4yr_nat` provide ED-published national medians and percentiles per CIP code. Use these to position a program vs. the national baseline — works even when the peer group is too small to rank.

**Source files downloaded May 2026 are in:**
`~/ditteau_data_inbox/CSC/College_Scorecard_Raw_Data_03232026/`

### D.3 How It Works in dbt

```yaml
sources:
  - name: college_scorecard
    database: "{{ var('external_sources_database') }}"
    schema: college_scorecard
```

The `sc_num()` macro is defined inline in `stg_scorecard__field_of_study.sql` — it handles all four ED null sentinels and wraps every outcome column cast.

Model lineage:

```
DITTEAU_SHARED.COLLEGE_SCORECARD.*
  └── stg_scorecard__field_of_study         (view, deterge/staging/scorecard)
        └── mart_scorecard_program_outcomes  (table, distribute/summaries)
```

Both models join to `dim_institution` via `ipeds_unitid`. The staging model uses an `INNER JOIN` to scope to the peer universe only, keeping the materialized mart small.

### D.4 One-Time Snowflake Setup

Completed May 2026.

```sql
create schema DITTEAU_SHARED.COLLEGE_SCORECARD
    comment = 'U.S. Department of Education College Scorecard field-of-study data';
create stage DITTEAU_SHARED.COLLEGE_SCORECARD.INGEST_STAGE;
create file format DITTEAU_SHARED.COLLEGE_SCORECARD.csv_standard
    like MERRIMACK_DD_DEV.DEPOSIT.csv_standard;
```

```bash
# Create table from registry
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema COLLEGE_SCORECARD \
  --role SYSADMIN --warehouse PLATFORM_WH \
  create-tables college_scorecard
```

### D.5 Annual Update (October–November)

Estimated effort: ~45 minutes.

**Step 1 — Download the new file**

Go to [collegescorecard.ed.gov/data/](https://collegescorecard.ed.gov/data/) → Field of Study Data → download `Most-Recent-Cohorts-Field-of-Study.zip`. Extract to `~/ditteau_data_inbox/CSC/`.

Also download the latest crosswalk `Crosswalks/CW{YYYY}_prelim.xlsx` — update the OPEID seed if institutions have changed.

**Step 2 — Prep the file**

```bash
cd ~/ditteau_data_inbox/CSC/
python3 prep_scorecard.py Most-Recent-Cohorts-Field-of-Study.csv <YYYY>

# Strip national rollup rows
awk -F',' 'NR==1 || $2!="NA"' Most-Recent-Cohorts-Field-of-Study_prepped.csv \
  > Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv
```

`prep_scorecard.py` lives at `~/ditteau_data_inbox/CSC/prep_scorecard.py`.

**Step 3 — Check for column changes**

```bash
head -1 Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv | tr ',' '\n' | wc -l
```

Expected: `179`. If different, run validate (Step 4) and compare the mismatch report against `source_registry.yml`.

**Step 4 — Validate**

```bash
cd /Users/laurievanpelt/ditteau_data_infra/deposit_loader

/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema COLLEGE_SCORECARD \
  --role SYSADMIN --warehouse PLATFORM_WH \
  validate college_scorecard scorecard_field_of_study \
  ~/ditteau_data_inbox/CSC/Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv
```

Must show `✓ All 179 columns match` before loading.

**Step 5 — Load**

```bash
/Users/laurievanpelt/testenv/bin/python3 deposit_loader.py \
  --database DITTEAU_SHARED --schema COLLEGE_SCORECARD \
  --role SYSADMIN --warehouse PLATFORM_WH \
  load college_scorecard scorecard_field_of_study \
  ~/ditteau_data_inbox/CSC/Most-Recent-Cohorts-Field-of-Study_prepped_clean.csv \
  --truncate
```

Expected: ~220,000–230,000 rows loaded, 0 rejected.

**Step 6 — Update the OPEID crosswalk seed (if new CW file available)**

```bash
cd ~/ditteau_data_inbox/CSC/
python3 convert_crosswalk.py Crosswalks/CW{YYYY}_prelim.xlsx <YYYY>
# Copy seed_scorecard_opeid_crosswalk.csv to seeds/shared/ in dbt repo

cd /Users/laurievanpelt/ditteau_data_transform
PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh seed --select seed_scorecard_opeid_crosswalk
```

**Step 7 — Rebuild dbt models for all active schools**

```bash
cd /Users/laurievanpelt/ditteau_data_transform

PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_merrimack_dev.sh build \
  --select "stg_scorecard__field_of_study mart_scorecard_program_outcomes"

PATH="/Users/laurievanpelt/testenv/bin:$PATH" \
  bash scripts/run_anselm_dev.sh build \
  --select "stg_scorecard__field_of_study mart_scorecard_program_outcomes"
```

**Step 8 — Verify**

```sql
select count(*) from DITTEAU_SHARED.COLLEGE_SCORECARD.SCORECARD_FIELD_OF_STUDY;

select source_file, load_status, rows_loaded, rows_rejected, load_completed_at
from DITTEAU_SHARED.COLLEGE_SCORECARD._LOAD_HISTORY
order by load_started_at desc limit 3;

-- Sanity check: own institution programs visible
select institution_name, cip_code, cip_desc, credential_level_desc,
       earn_median_4yr, earn_median_4yr_national, debt_median_all,
       earnings_to_debt_ratio_4yr, peer_rank_earnings_4yr
from MERRIMACK_DD_DEV.DISTRIBUTE.MART_SCORECARD_PROGRAM_OUTCOMES
where is_own_institution = true
  and credential_level_code = 3
order by earn_median_4yr desc nulls last
limit 20;
```

### D.6 Downstream Models

| Model | Layer | Description |
|-------|-------|-------------|
| `stg_scorecard__field_of_study` | `deterge/staging/scorecard` | Cleaned field-of-study data — earnings, debt, repayment by CIP and credential |
| `mart_scorecard_program_outcomes` | `distribute/summaries` | Program-level benchmarks with peer ranks, ROI ratios, national context |
| `seed_scorecard_opeid_crosswalk` | `seeds/shared` | OPEID→UNITID mapping for FSA data joins |

---

## E. Adding a New External Source

Use this checklist when onboarding a new external dataset (e.g., NSC Clearinghouse, BLS OEWS, FSA Open Data).

### E.1 Snowflake Setup

```sql
create schema DITTEAU_SHARED.<SOURCE_NAME>
    comment = '<Description of source>';
create stage DITTEAU_SHARED.<SOURCE_NAME>.INGEST_STAGE;
create file format DITTEAU_SHARED.<SOURCE_NAME>.csv_standard
    like MERRIMACK_DD_DEV.DEPOSIT.csv_standard;
```

### E.2 Register in Deposit Loader

Add to `ditteau_data_infra/deposit_loader/source_registry.yml`:

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

> ⚠️ **Column types matter.** Use `VARCHAR` for any column that might contain letter suffixes, mixed types, or ED/NCES null sentinels. Use `INTEGER` only when you are certain the column contains only integers. For any ED or federal source, assume null sentinels exist — implement `sc_num()` or equivalent in the staging model.

### E.3 Add dbt Source Definition

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

### E.4 Write Staging Model

Follow the IPEDS or Scorecard staging models as a template:

○ File: `models/deterge/staging/<source>/stg_<source>__<entity>.sql`
○ Rename and cast all columns; no business logic in staging
○ Add `{{ add_source_metadata() }}` macro at the end of the select
○ Document grain, source file, and any quirks in the header comment block
○ For ED/federal sources: implement `sc_num()` or equivalent for null sentinel handling

### E.5 Validate Before You Load

Always run `validate` before `load`. The validate command catches column count mismatches and name drift before any data touches Snowflake — a 30-second check that saves an hour of cleanup.

### E.6 Document Here

Add a new lettered section to this runbook covering:

○ What the data is and where it comes from
○ Update cadence and trigger
○ Where it lives in Snowflake
○ Data quirks discovered during initial load
○ How it connects to dbt models
○ Annual/periodic update steps with exact commands

---

## F. Planned Sources

Next in the external data build queue per the [External Data Integration Plan](https://docs.google.com/document/d/1Fu2BDcSRhgJrUVpD4N70ziuBtYr39F9dNev7sF3Dxb8/edit?usp=sharing).

| Source | Priority | KPIs Served | Status |
|--------|----------|-------------|--------|
| FSA Open Data | Tier 1 | FAFSA completion rate (KPI 3 gap) | Not started |
| BLS OEWS | Tier 1 | Program ROI, post-grad earnings | Not started |
| ACS 5-year (Census) | Tier 2 | Territory analysis, first-gen proxy | Not started |
| BEA Regional Economic Data | Tier 2 | Enrollment demand modeling | Not started |
| State Commission Reports (MA/NH) | Tier 2 | State grant benchmarking | Not started |
| Common Data Set aggregates | Tier 2 | Peer admissions benchmarking | Not started |

---

## G. Troubleshooting

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| `Object 'DITTEAU_SHARED.*' does not exist` | School dbt role lacks `SELECT` on `DITTEAU_SHARED` | Grant `USAGE` on database/schema and `SELECT` on tables to the school's transform role |
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
| Scorecard load: rows rejected, `'NA'` not recognized | NA-unitid national rollup rows not stripped | Use `_prepped_clean.csv`, not `_prepped.csv`; run the `awk` strip command |
| Scorecard: `credlev` cast error | `credlev` typed `INTEGER` but file has `'NA'` values | `credlev` must be `VARCHAR` in registry; staging model guards with `CASE` |
| Scorecard: 179 columns expected but N found | ED added/removed columns in new release | Run `validate`; compare mismatch report to registry; update registry and staging model |
| Scorecard: `earn_mdn_4yr` all `NULL` for a school | All programs suppressed (small peer cohorts) | Expected for small programs; use `earn_median_4yr_national` for national context instead |
