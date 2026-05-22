# ADR-002: Delete-Where Flag for Deposit Loader

**Status:** Accepted

**Date:** 2026-05-22

**Author:** LVP

---

### Context

The Ditteau platform ingests external reference data from federal sources like IPEDS and College Scorecard into the `DITTEAU_SHARED` database using `deposit_loader.py`. These datasets arrive annually and contain multi-year historical data. For example, IPEDS Fall Enrollment surveys include records for multiple survey years (2020, 2021, 2022, 2023, 2024), and each annual release only provides new or corrected data for the most recent year.

Before this decision, the deposit loader offered only two loading modes:
1. **Append mode** (default) - adds new rows without removing existing data
2. **Truncate mode** (`--truncate`) - deletes all rows before loading

Neither mode was appropriate for multi-year tables. Append mode caused duplicate rows when reloading corrected data for a year already in the table. Truncate mode required operators to reload the complete multi-year dataset every time they wanted to refresh a single year, which was both time-consuming and fragile (if the load failed after truncation, all historical data would be lost).

The platform serves 2+ schools across DEV/TEST/PROD environments, and IPEDS alone includes 20+ multi-year survey tables. Without a surgical deletion option, annual IPEDS refreshes required either manual pre-deletion SQL (error-prone) or full table reloads from a consolidated multi-year CSV (operational overhead).

---

### Decision

Added a `--delete-where` flag to `deposit_loader.py` that accepts a SQL predicate string and executes a targeted `DELETE` statement before the `COPY INTO` operation. This enables partition-level replacement for multi-year tables.

**Implementation details:**
- Flag added to both `load` and `load-all` subcommands (lines 938-948, 958-968 in `deposit_loader.py`)
- Mutually exclusive with `--truncate` - validation exits with error if both flags are passed (lines 518-523)
- DELETE statement executes as step [2c/5] in the load sequence, immediately before COPY INTO (lines 588-598)
- Single-quote escaping applied to predicate before SQL interpolation (line 672) for basic SQL injection mitigation
- Predicate and deleted row count recorded in `_LOAD_HISTORY.notes` column for audit trail (lines 664-675, 683-684)
- Status command updated to display notes column (lines 822, 834-835)

**Example usage:**
```bash
# Replace only 2024 survey year data
python deposit_loader.py \
  --database DITTEAU_SHARED --schema IPEDS \
  --role SYSADMIN --warehouse PLATFORM_WH \
  load ipeds ipeds_fall_enrollment ef2024a_prepped.csv \
  --delete-where "survey_year = 2024"
```

**Scope boundaries:**
- This addresses multi-year external data tables only (IPEDS, Scorecard). School-specific source systems (Jenzabar, Workday, Slate) continue using append or truncate modes.
- Predicate validation is not enforced - operators must supply syntactically correct SQL.

---

### Consequences

#### Pros

- Annual IPEDS refreshes now require loading only the new year's files (~5-10 CSVs), not reconstituting the entire multi-year dataset
- Iterative corrections to a single year no longer risk data loss (pre-2024 data remains untouched)
- Audit trail in `_LOAD_HISTORY.notes` records exactly which rows were deleted during each load
- Eliminates the risk of truncate-then-fail scenarios for multi-year tables (if COPY fails, historical data remains intact)
- Consistent operational pattern for all multi-year tables (IPEDS surveys, Scorecard field-of-study, future external sources)
- Load-all batch operations can now use surgical deletion across all files (`--delete-where "survey_year = 2024"` applies to every table in the directory)

#### Cons

- SQL injection risk if loader ever gets an API or web interface (acceptable for CLI-only operator use; documented as requiring allowlist validation before API exposure)
- Operators must know which tables are multi-year and which are single-snapshot to choose the correct mode
- No validation that the predicate column exists in the target table (will fail at DELETE execution time)
- Adds a third loading mode to document and train operators on
- Notes column was added to existing `_LOAD_HISTORY` tables across all schools/environments (backward-compatible but not retroactively populated for old loads)

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Manual DELETE + load workflow** | No code changes needed; operators already know SQL | Error-prone (easy to forget DELETE step); no audit trail; inconsistent process across operators |
| **Keep using --truncate only** | Simplest mental model (only two modes) | Forces full multi-year reloads; high risk of data loss if load fails; 5-10× more operational work per IPEDS cycle |
| **Add --delete-where flag** (chosen) | Surgical deletion preserves historical data; auditable; consistent with CLI patterns | Adds SQL injection surface (mitigated for CLI use); operators must understand when to use it |
| **Separate "partition-replace" command** | Could enforce column allowlist for predicates; clearer intent | More complex CLI surface; would still need to handle arbitrary year values; over-engineered for current scale |

---

### References

- [deposit_loader.py](file:///Users/laurievanpelt/ditteau_data_infra/deposit_loader/deposit_loader.py)
- [DELETE_WHERE_IMPLEMENTATION.md](file:///Users/laurievanpelt/ditteau_data_infra/deposit_loader/DELETE_WHERE_IMPLEMENTATION.md) - Implementation summary and test checklist
- [Deposit Loader README](file:///Users/laurievanpelt/ditteau_data_infra/deposit_loader/README.md) - Updated usage documentation
- [External Data Sources Runbook](runbooks/external-data-sources.md)
