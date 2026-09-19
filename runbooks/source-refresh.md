# Source Refresh Troubleshooting

Data ingestion procedures and troubleshooting for the deposit_loader.

---

## Overview

The deposit_loader is a Python CLI tool that loads CSV files from source systems into Snowflake's DEPOSIT schema. It handles validation, staging, loading, and audit tracking.

---

## Daily Workflow

### 1. Drop Files in Inbox

```
~/ditteau_data/inbox/
├── powerfaids/
│   ├── awards.csv
│   └── students.csv
├── workday_student/
│   ├── students.csv
│   └── academic_records.csv
└── slate/
    └── applications.csv
```

Filenames (without `.csv`) must match table keys in `source_registry.yml`.

### 2. Validate Headers

```bash
cd ~/ditteau_data_infra/deposit_loader
python deposit_loader.py --school MERRIMACK validate powerfaids awards ~/inbox/powerfaids/awards.csv
```

### 3. Load Data

```bash
# Single file — append (default)
python deposit_loader.py --school MERRIMACK load powerfaids awards ~/inbox/powerfaids/awards.csv

# Single file — replace
python deposit_loader.py --school MERRIMACK load powerfaids awards ~/inbox/powerfaids/awards.csv --truncate

# All files for a source
python deposit_loader.py --school MERRIMACK load-all workday_student ~/inbox/workday_student/
```

### 4. Verify

```bash
python deposit_loader.py --school MERRIMACK status
python deposit_loader.py --school MERRIMACK staged
```

---

## Commands Reference

| Command | Purpose |
|---------|---------|
| `create-tables <source>` | Create DEPOSIT tables from registry |
| `list-tables <source>` | Compare database vs registry |
| `validate <source> <table> <file>` | Check CSV headers |
| `preview <source> <table> <file>` | Show first 10 rows |
| `load <source> <table> <file>` | Load single CSV |
| `load-all <source> <directory>` | Load all CSVs in directory |
| `status` | Show recent load history |
| `staged` | Show files in stage |

---

## Load Modes

### Append (Default)

Adds rows to existing data:

```bash
python deposit_loader.py --school MERRIMACK load powerfaids awards awards.csv
```

**Use for:** Incremental exports, daily feeds, audit logs.

### Full Replace (`--truncate`)

Truncates table before loading:

```bash
python deposit_loader.py --school MERRIMACK load powerfaids awards awards.csv --truncate
```

**Use for:** Full-refresh exports, initial loads, corrections.

**Warning:** If COPY INTO fails after truncate, the table is empty. Test in DEV first.

### Surgical Replace (`--delete-where`)

Deletes matching rows before loading:

```bash
python deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  load ipeds ipeds_fall_enrollment ef2024.csv \
  --delete-where "survey_year = 2024"
```

**Use for:** Multi-year tables, partial corrections.

`--truncate` and `--delete-where` are mutually exclusive.

---

## Global Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--school CODE` | `$DD_SCHOOL_CODE` | School code (e.g., `MERRIMACK`) |
| `--env ENV` | `DEV` | Environment: `DEV`, `TEST`, `PROD` |
| `--connection NAME` | `dd_prod` | Connection name in `config.toml` |
| `--database DB` | `{SCHOOL}_DD_{ENV}` | Override target database |
| `--schema SCHEMA` | `DEPOSIT` | Override target schema |
| `--role ROLE` | `{SCHOOL}_WRITE_{ENV}` | Override Snowflake role |
| `--warehouse WH` | `{SCHOOL}_TRANSFORM_{ENV}` | Override warehouse |

---

## Loading to DITTEAU_SHARED

For IPEDS and College Scorecard (platform-wide data):

```bash
python deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role DITTEAU_WRITE --warehouse DITTEAU_TRANSFORM \
  load ipeds ipeds_fall_enrollment EF2024.csv
```

---

## Load Tracking

Every load is recorded in `{DATABASE}.{SCHEMA}._LOAD_HISTORY`:

| Column | Description |
|--------|-------------|
| `load_id` | Auto-increment identifier |
| `source_system` | Source system label |
| `source_file` | Staged file path |
| `target_table` | Target table name |
| `load_status` | `PENDING` → `SUCCESS`/`PARTIAL`/`FAILED` |
| `rows_loaded` | Rows inserted |
| `rows_rejected` | Rows rejected |
| `error_message` | First error (if any) |
| `notes` | Audit trail (truncate/delete-where used) |

### Query Load History

```sql
SELECT *
FROM MERRIMACK_DD_DEV.DEPOSIT._LOAD_HISTORY
ORDER BY load_started_at DESC
LIMIT 20;
```

---

## Troubleshooting

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `snowflake-connector-python is not installed` | Missing dependency | `pip install snowflake-connector-python` |
| `config.toml not found` | Missing credentials | Create `~/.snowflake/config.toml` |
| `--school is required` | No school specified | Add `--school CODE` or set `$DD_SCHOOL_CODE` |
| `Cannot determine Snowflake role` | Role not derivable | Add `--role` flag |
| `Unknown source system` | Typo in source name | Check `source_registry.yml` keys |
| `No registry match` | Filename doesn't match | Ensure filename matches table key |
| `Number of columns in file does not match` | Schema mismatch | Run `validate`, update registry, re-run `create-tables` |

### COPY INTO Loads 0 Rows

1. Run `validate` to check headers match
2. Run `preview` to inspect the file
3. Check for encoding issues (UTF-8 expected)
4. Check for empty file

### Permission Errors

Verify role has:
- `READ` on stage
- `INSERT`/`DELETE` on target tables
- `INSERT`/`UPDATE` on `_LOAD_HISTORY`
- `USAGE` on warehouse

Resume suspended warehouse if needed.

### Slate Table Names

Slate uses dot-notation (`APPLICATION.BIN`). The loader automatically converts to underscores (`APPLICATION_BIN`).

---

## What Happens Under the Hood

```
[1/5] INSERT into _load_history     → Records PENDING with load_id
[2/5] PUT file to @ingest_stage     → Upload + compress
[2b]  TRUNCATE (if --truncate)      → Empty table
[2c]  DELETE WHERE (if --delete-where) → Surgical delete
[3/5] COPY INTO target table        → Load data
[4/5] UPDATE _load_history          → Record outcome
[5/5] REMOVE staged file            → Cleanup
```

### Automatic Metadata Columns

Every row receives:

| Column | Value |
|--------|-------|
| `_source_system` | Registry label (e.g., `POWERFAIDS`) |
| `_source_file` | Stage path |
| `_loaded_at` | COPY INTO timestamp |
| `_load_id` | Links to `_load_history` |

---

## Adding New Tables

### Same Source System

1. Add entry to `source_registry.yml`
2. Run `create-tables` for that source
3. Validate and load

### New Source System

1. Add top-level key to `source_registry.yml`
2. Set `source_system_label`, `default_file_format`
3. Add tables with column definitions
4. Create inbox folder
5. Run `create-tables new_system`

---

## Related Documentation

- [External Data Sources](external-data-sources.md)
- [Source Systems](../source-systems/README.md)
- [dbt Conventions](../dbt/conventions.md)
