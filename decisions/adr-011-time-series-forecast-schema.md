# ADR-011: Time-Series Forecast Schema Design

**Status:** Proposed

**Date:** 2026-09-07

**Author:** LVP

---

### Context

The Predictive Analytics Meet (scheduled for Q4 2026) establishes forecasting as
the first predictive capability to operationalize. A proof-of-concept using
Snowflake Cortex `SNOWFLAKE.ML.FORECAST` has validated that institution-level
enrollment forecasting is viable:

- 18 years of clean annual data (post-ACYR fix)
- 2.3% MAPE on headcount forecasts
- Cortex handles model training and inference in SQL

Moving from prototype to production requires decisions about where forecast
objects live, how they are named, and how they integrate with existing marts.

**Objects to place:**

| Object Type | Example | Purpose |
|-------------|---------|---------|
| Training tables | `FORECAST_TRAINING_HEADCOUNT` | Input data for model training |
| Cortex models | `FORECAST_MODEL_HEADCOUNT` | Trained ML models |
| Forecast output tables | `FORECAST_HEADCOUNT_INSTITUTION` | Predictions for consumption |
| Backtest results | `FORECAST_BACKTEST_HEADCOUNT` | Model validation history |

**Constraints:**

1. **Cortex models are immutable.** New data requires training a new model, which
   creates a new object. Model versioning is external to Snowflake.

2. **Models are per-series.** Multi-series forecasting requires one model per
   time series (e.g., one per load_status segment).

3. **Grain viability varies.** Institution and load_status grains are 100%
   viable; finer grains drop below the 12-observation floor.

4. **Forecasts are aggregates, not student-level.** They do not require the
   governance framework for derived predictive attributes.

---

### Decision

**Place all forecast objects in `DISTRIBUTE` schema, using a `FORECAST_` prefix
convention.**

#### 1. Schema location

All forecast objects live in `{SCHOOL}_DD_{ENV}.DISTRIBUTE`:

- Consistent with marts (consumer-facing layer)
- Accessible to analytics roles without additional grants
- Avoids a new schema for a single capability

#### 2. Naming convention

| Object Type | Pattern | Example |
|-------------|---------|---------|
| Training table | `FORECAST_TRAINING_{METRIC}` | `FORECAST_TRAINING_HEADCOUNT` |
| Model | `FORECAST_MODEL_{METRIC}` | `FORECAST_MODEL_HEADCOUNT` |
| Model (segmented) | `FORECAST_MODEL_{METRIC}_{SEGMENT}` | `FORECAST_MODEL_HEADCOUNT_FT` |
| Output table | `FORECAST_{METRIC}_{GRAIN}` | `FORECAST_HEADCOUNT_INSTITUTION` |
| Backtest table | `FORECAST_BACKTEST_{METRIC}` | `FORECAST_BACKTEST_HEADCOUNT` |

Segment abbreviations: `FT` (full-time), `PT` (part-time), `HT` (half-time),
`LHT` (less-than-half-time), `UNK` (unknown).

#### 3. Training table structure

Training tables are materialized by dbt and serve as input to Cortex models:

```sql
-- dbt model: forecast_training_headcount.sql
select
    date_from_parts(
        2000 + left(academic_year, 2)::int,
        8,
        1
    )                           as forecast_date,
    sum(headcount)              as headcount
from {{ ref('mart_enrollment_census') }}
group by academic_year
```

**Columns:**
- `forecast_date` — proper timestamp for Cortex (August 1 of start year)
- `{metric}` — the target column (headcount, fte, etc.)
- Optional: segment columns for multi-series training

#### 4. Forecast output table structure

Output tables store predictions for BI consumption:

| Column | Type | Description |
|--------|------|-------------|
| `forecast_date` | DATE | Future period (August 1 of academic year) |
| `academic_year` | VARCHAR(4) | Academic year code ('2425', '2526', etc.) |
| `forecast_value` | NUMBER | Point estimate |
| `lower_bound_95` | NUMBER | 95% prediction interval lower |
| `upper_bound_95` | NUMBER | 95% prediction interval upper |
| `model_name` | VARCHAR | Source model name |
| `model_trained_at` | TIMESTAMP | When the model was trained |
| `forecast_generated_at` | TIMESTAMP | When this forecast was generated |

#### 5. Model lifecycle

**Training cadence:** Annually, after Fall census date (October 15).

**Versioning:** Model names do not include version numbers. Each training
replaces the previous model (`CREATE OR REPLACE`). The `model_trained_at`
timestamp in output tables provides audit trail.

**Retention:** Training tables are retained indefinitely (small, useful for
backtesting). Old forecast outputs may be archived after 3 years.

#### 6. Orchestration

Model training is **not** part of the standard dbt build. It runs as a separate
scheduled task after the Fall census mart refresh:

```
mart_enrollment_census refresh (dbt)
    ↓
forecast_training_headcount refresh (dbt)
    ↓
FORECAST_MODEL_HEADCOUNT training (Snowflake Task)
    ↓
FORECAST_HEADCOUNT_INSTITUTION generation (Snowflake Task)
```

Rationale: Cortex model training is expensive relative to dbt model builds and
only needs to run annually.

#### 7. Multi-series (segmented) forecasts

For load_status segmentation (5 viable series), train separate models:

| Series | Model Name |
|--------|------------|
| Full-time | `FORECAST_MODEL_HEADCOUNT_FT` |
| Part-time | `FORECAST_MODEL_HEADCOUNT_PT` |
| Half-time | `FORECAST_MODEL_HEADCOUNT_HT` |
| Less-than-half-time | `FORECAST_MODEL_HEADCOUNT_LHT` |
| Unknown | `FORECAST_MODEL_HEADCOUNT_UNK` |

Alternative: Use Cortex's multi-series syntax with `SERIES_COLNAME`. Deferred
pending validation.

#### 8. BI integration

Forecast tables are exposed to Tableau/Power BI via the standard analytics role.
Forecasts appear alongside actuals in enrollment dashboards:

```sql
-- Combined view for BI
select
    academic_year,
    headcount,
    null as forecast_value,
    'actual' as data_type
from mart_enrollment_census

union all

select
    academic_year,
    null as headcount,
    forecast_value,
    'forecast' as data_type
from forecast_headcount_institution
```

---

### Consequences

#### Pros

- **Minimal new infrastructure.** Uses existing `DISTRIBUTE` schema and naming
  conventions.
- **Clear separation.** `FORECAST_` prefix makes forecast objects immediately
  identifiable.
- **BI-ready.** Output tables are structured for direct dashboard consumption.
- **Audit trail.** Timestamps on output tables track model lineage.
- **Flexible orchestration.** Decoupling from dbt allows annual training without
  daily overhead.

#### Cons

- **No model versioning in Snowflake.** Rollback requires retraining from
  historical training table.
- **Manual orchestration.** Task scheduling is outside dbt; requires separate
  monitoring.
- **One model per series.** Segmented forecasts multiply the number of model
  objects.
- **Schema crowding.** Adds 10-15 objects to `DISTRIBUTE` per school (acceptable
  given current object count).

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **`DISTRIBUTE` schema with prefix (chosen)** | Consistent with marts; no new grants | Mixes forecast objects with marts |
| Dedicated `FORECAST` schema | Clean separation | New schema to grant and maintain |
| `DETERGE` schema (silver layer) | Keeps `DISTRIBUTE` clean | Forecasts are consumer-facing, not intermediate |
| Model names with version suffix (`_V1`, `_V2`) | Explicit versioning | Breaks `CREATE OR REPLACE` pattern; accumulates objects |

---

### Open Questions

| # | Question | Owner |
|---|----------|-------|
| TS-1 | What is the exact Fall census date to trigger annual retraining? | RDT |
| TS-2 | Should multi-series training use `SERIES_COLNAME` or separate models? | LVP |
| TS-3 | How should forecast accuracy be monitored over time? (MAPE drift) | RDT/LVP |
| TS-4 | What happens when a school has <12 years of data? | RDT |

---

### References

- Cortex forecast prototype: `ditteau_data_forecast/docs/cortex_forecast_prototype.md`
- Forecast readiness findings: `ditteau_data_forecast/docs/forecast_readiness_findings.md`
- Platform gap analysis: `ditteau_data_forecast/docs/platform_capability_gaps.md`
- ACYR corruption fix: `ditteau_data_forecast/docs/acyr_corruption_investigation.md`
- Snowflake Cortex ML Forecasting: https://docs.snowflake.com/en/user-guide/ml-functions/forecasting
