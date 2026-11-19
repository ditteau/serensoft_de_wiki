# ADR-014: Capture As-Known History by Daily Change Capture and Certified Runs

**Status:** Proposed

**Date:** 2026-11-19

**Author:** LVP

---

### Context

Schools and Serensoft management ask for point-in-time reporting for trend analysis
and forecasting. That covers two different kinds of history:

- **As of (valid time):** what was true for a past period. This can be rebuilt if the
  source keeps period rows or effective dates.
- **As known (knowledge time):** what the data said on a past date, such as census-day
  headcount or a frozen year-end figure. This exists only if it was captured on that
  date.

No ADR decides whether the platform keeps history. The assessment
(`ditteau_data_transform/docs/designs/point_in_time_assessment.md`, 2026-11-19) measured
the following.

**The platform holds no as-known history.**
- Every row in the five `snap_*` marts was computed between 2026-08-15 and 2026-11-11
  from current data, including rows labelled 2002.
- Those marts are incremental models. A `--full-refresh` silently rewrites them, and
  two freeze each term at the first build that sees it, with no rule tied to census.
- Today's Merrimack fall headcounts run 1.2–2.0% below census, because later
  withdrawals drop out of the count.

**The live source overwrites its own past.**
- Jenzabar CX is a frozen archive through Fall 2024, so it supports as-of history only.
- Merrimack's live source, Jenzabar One, loads by hand with `deposit_loader.py --truncate`.
- Time Travel is 1 day on every database (the account is ENTERPRISE), so each load
  destroys the previous extract within a day.
- A measured load replaced 1,510 of 34,133 `SECTION_MASTER` rows. J1 `JOB_TIME` advanced
  on only 83–90% of changed rows; `APPROWVERSION` advanced on essentially all of them.

**The published requirements need daily history.**
- The KPI library promises four daily admissions KPIs, including "Deposit Pace vs.
  Prior Year".
- It promises weekly registration and aid KPIs in season, census figures each term,
  and monthly finance.

**Why decide now.** History cannot be backfilled. Every census, term close and
month-end that passes before a school's capture starts is permanently as-of only; the
Merrimack Fall 2026 census is already in that state.

The design should therefore exist, proven, before any school's daily feed starts, so
capture begins the day the feed does. No school is committed to a date. Merrimack's
feed depends on Merrimack's own pace.

---

### Decision

The design is built and proven on **DEMEAU**, where generated data is allowed (R2) and
daily changes can be simulated. It is then **enabled per school when that school's
daily feed exists**.

**1. Daily extract.**
- In-scope tables are extracted daily into `{SCHOOL}_DD_PROD.DEPOSIT` as one consistent
  run. DEV and TEST receive Deposit by clone from PROD, as `clone_deposit.py` intends.
- Full-replace loads stay; change capture works on full-replace extracts.
- **Loads become atomic.** `deposit_loader.py` loads into a temporary table and swaps
  it in, instead of truncating and then COPYing. Today a failed COPY leaves an empty
  table, which change capture would read as every row deleted.
- Each run is verified before anything downstream runs: every table has a `SUCCESS`
  row in `_LOAD_HISTORY` for the same run, and row counts are within tolerance of the
  previous load.
- A failed or partial run stops that day's capture and raises an alert.
- Extraction runs on a schedule, not by hand. The scheduler technology is chosen when
  the scheduling chunk is built (rollout plan 0.2).

**2. Change capture into `DOCUMENT`.**
- A new schema, `{SCHOOL}_DD_PROD.DOCUMENT`, holds one dbt snapshot per in-scope
  **Deposit source table**: `select *`, no transformation.
- Why raw, not staging:
  - It follows dbt's guidance to snapshot sources.
  - Raw history can be re-processed when staging logic is fixed, whereas history
    captured through a buggy staging model would be permanently wrong.
  - Columns withheld by the redaction register already arrive as NULL (R4). A column
    withheld *later* stays in older versions either way, so the column purge (8)
    applies regardless.
- Snapshot settings: `strategy: check`, `hard_deletes: new_record`, `transient: false`.
- The per-table `unique_key` and change test are declared in the schema registry, and
  the snapshot config is generated from it, as staging is:
  - **Default:** `check_cols` is an explicit list of every data column. It
    **excludes the loader metadata columns** (`_LOADED_AT`, `_LOAD_ID`, `_SOURCE_FILE`,
    `_SOURCE_SYSTEM`), which change on every load and would otherwise version every
    row daily. This works for any source with a stable key.
  - **A single change marker** replaces the column list only after it is measured
    against two real extracts, and moves on every changed row.
  - **J1:** `unique_key: appid`, `check_cols: ['approwversion']`. Measured 2026-11-18:
    it advanced on 1,507 of 1,510 and 604 of 604 changed rows. `JOB_TIME` advanced on
    83–90% and is not used.
  - **Banner, Workday, Slate, PowerFAIDS:** the default until a marker is measured
    against real extracts. Banner's keys are composite.
- `known_from` is the extract's `_LOADED_AT`, not the snapshot run time and not a
  source timestamp.
- The project-level snapshot defaults in `dbt_project.yml` (`strategy: timestamp`,
  `updated_at: updated_at`, `unique_key: id`, `target_schema: deterge`) are replaced.
- `DOCUMENT` is the one schema that cannot be rebuilt from anything else. It is
  permanent, and readable only by the build service role and Serensoft administration.
  Consumers reach it through models.

**3. Scoped bitemporal models in `DETERGE`.**
- For each entity that a published figure depends on, an intermediate model joins
  `DOCUMENT` history (known from/to) with the source's effective dates (valid from/to).
- These models apply the same cleaning logic as staging, through shared macros, so a
  staging fix can be re-applied to all of history.
- The entities are: term calendar, section enrollment, registration, student term
  status, student type, program, applications, aid awards, GL, and the source decode
  tables for each.
- Intervals are half-open and built by one shared macro.
- These models are transient and rebuildable from `DOCUMENT`.
- Out of scope: tables no published figure reads, name and biography tables beyond
  their reporting columns, and the CX archive.

**4. Certified runs in `DECLARE`.**
- A new schema, `{SCHOOL}_DD_PROD.DECLARE`, holds every official or externally reported
  figure as a run:
  - the frozen output;
  - the valid-as-of date and the knowledge cutoff;
  - the extract it rests on;
  - the transformation git commit;
  - certification status and certifier;
  - the submission reference.
- `DECLARE` is written by a certification step outside dbt and is **never dbt-managed**,
  so no build or `--full-refresh` can recompute or overwrite it.
- Restating a figure is a new run, never an overwrite.

**5. Labels.** Every published figure is labelled *census (as captured)*, *corrected*,
or *current*.

**6. Snap marts.**
- The five `snap_*` marts in `DISTRIBUTE` become reporting views over the bitemporal
  models and `DECLARE`.
- Until then they are set `full_refresh = false` and `transient = false`, and their
  first-sight freeze is fixed before any J1 term flows through them.

**7. Production only; no clone-down of history.**
- `DOCUMENT` and `DECLARE` exist in PROD only (R5) and are **not cloned to DEV or
  TEST**, so real PII history lives in exactly one place.
- History logic is developed and tested against DEMEAU.

**8. Purge.**
- One written procedure purges by student key and by column across `DEPOSIT`, every
  Deposit clone in other environments, `DOCUMENT`, `DECLARE` (where outputs are
  per-student) and the `snap_*` tables.
- It executes within 5 business days of a register change or deletion request. Data is
  then gone from our access in about 2 weeks and from Snowflake in about 3 (Fail-safe
  is 7 days and cannot be shortened).
- `DOCUMENT` and `DECLARE` are added to `check_redaction.py`'s per-tenant sweep.

#### Relation to the medallion architecture

This adds **no new refinement layers**. The platform remains three layers plus two
stores:

| Medallion role | Schemas | Holds |
|---|---|---|
| Bronze | `DEPOSIT` + `DOCUMENT` | What arrived (latest extract), and how it changed (history) |
| Silver | `DETERGE` | Cleaned, conformed and bitemporal models |
| Gold | `DISTRIBUTE` | Analytics-ready marts and dimensions |
| Published record (outside medallion) | `DECLARE` | What we certified and reported, from which data and code |

**`DOCUMENT` restores a property medallion assigns to Bronze.**
- Bronze is meant to keep raw history, and Deposit's own design said it did
  (`models/deposit/_deposit_overview.md`, corrected 2026-11-19).
- In practice, `--truncate` loads and 1-day Time Travel made Deposit a mirror of the
  latest extract.
- `DOCUMENT` keeps raw change history: only the rows that changed, for in-scope tables
  only.
- That keeps it purgeable under the redaction register (R3). A copy of every extract
  forever would not be.
- It is a separate schema, not history inside Deposit. Deposit is cloned freely to DEV
  and TEST, while history must exist once, be permanent, and have its own grants and
  purge scope.

**`DECLARE` covers a need medallion does not address.** Medallion describes how data is
refined, not how published figures are recorded. Institutional research offices
traditionally keep frozen census files by hand; `DECLARE` keeps that record inside the
warehouse, reproducible and governed.

**Both are separate schemas, not folders in existing ones,** for three reasons:
- `clone_deposit.py` clones DEPOSIT down to DEV and TEST;
- dbt rebuilds DISTRIBUTE;
- grants and purge scope are set per schema.

**Not decided here:**
- the restatement policy (who certifies, which figure appears where);
- retention periods (G-09b);
- the scheduler technology;
- the operational definition of census. That is RDT's and each school's.

---

### Consequences

#### Pros

- Both kinds of question become answerable, at daily resolution, for every day after a
  school's capture begins: "as of" and "as known on day X".
- Prior-year pace curves, census-day figures and month-end close states all come from
  one history. There is no separate store per report.
- **A clear storage contract.** `DOCUMENT` must never be lost, `DECLARE` must never be
  recomputed, and everything else rebuilds from those two plus `DEPOSIT`.
- Storage stays small. Only changed rows are kept: about 1,500 of 34,000 sections
  changed over 20 days. Merrimack's whole Deposit is 0.11 GB compressed.
- Published numbers are reproducible and auditable. A certified figure cannot drift
  when data or logic changes, which the platform's logic history this autumn shows it
  would.
- Purge has one place per entity, and history has one copy.
- Full-refresh safe: dbt does not rebuild snapshots on `--full-refresh`, and `DECLARE`
  is outside dbt.

#### Cons

- **History starts on a school's first daily snapshot and never earlier.** CX-era
  figures remain as-of only. The first honest prior-year pace comparison comes a full
  cycle after capture starts.
- **A new operational dependency.** A daily scheduled extract, verification, PROD dbt
  run and alerting, none of which exist today. The loader's scheduled phase is unbuilt.
  Daily PROD compute becomes a standing cost, inside the resource monitors from O-58.
- **A new long-lived PII store.** Every change to an in-scope student record is
  retained in `DOCUMENT` until purged. That raises the stakes on the purge procedure,
  retention periods (still null, G-09b) and grants.
- **TEST cannot verify history logic against real data**, because history is not
  cloned down. TEST verifies grants and masking; logic is verified on DEMEAU.
- **"Known" means "captured by us"**, at extract resolution. It cannot reflect when the
  institution learned something, because J1's own timestamps miss changes.
- **Bitemporal joins are easy to get wrong.** This needs a shared macro and a
  backdated-correction test in DEMEAU before it is trusted.
- **The change test is per source, and only J1's is measured.** Full-row comparison,
  the default, costs more compute. It also records formatting-only changes as new
  versions: when `USER_NAME` was blanked at extract, 557,760 `NAMEMASTER` rows read as
  changed. Each new source needs a measurement before a cheaper marker is trusted.
- **Schema drift across years.** Source upgrades change columns, and the loader maps by
  position (R4), so history models must tolerate added and dropped columns.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| Status quo: `snap_*` incremental marts | No new work | Holds no as-known history; rewritten by `--full-refresh`; two marts freeze at first sight; labels reconstructions as census |
| History inside Deposit, loader-side (merge on a change marker at load time) | One schema fewer | Rebuilds dbt snapshots in loader Python, without tests or lineage; history would be cloned to DEV and TEST with Deposit unless Deposit is split, which is `DOCUMENT` by another name |
| Snapshot the staging models instead of Deposit | Withheld and unneeded columns filtered first | A staging bug corrupts history permanently; withheld columns are already NULL at extract; against dbt guidance |
| Append every load to Deposit | Complete raw history; simple loader change | All 124 J1 staging models must filter to the latest load, and a missed filter silently duplicates rows (O-46 precedent); keeps every column later withheld; largest purge surface |
| Dated zero-copy clones of Deposit on capture dates | No modelling needed; exact copy | At daily cadence, about 365 full copies a year, each to purge; invisible to dbt and the redaction sweep (O-49 precedent); knowledge only on capture dates |
| dbt snapshots, `timestamp` strategy | Standard pattern | `_LOADED_AT` changes on every row each truncate load; `JOB_TIME` misses 10–17% of changes; CX `upd_date` is a DATE |
| Streams into an append-only table | Native change tracking | A truncate reload emits every row as delete and insert; a pipeline outside dbt lineage and tests; goes stale if not consumed |
| Longer Time Travel as history | No new objects | Recovery only (90-day maximum); lengthens every purge by the retention period |
| Bitemporal modelling for every table | Uniform | Cost with no reporting need; enlarges the PII store |
| **Daily change capture into `DOCUMENT`, scoped bitemporal models, certified runs in `DECLARE` (chosen)** | Answers both questions daily; small storage; reproducible publications; one copy of history; one purge point per entity | Needs a daily scheduler, verification and PROD runs; new PII history store; history begins at first capture |

---

### References

- `ditteau_data_transform/docs/designs/point_in_time_assessment.md`: findings,
  measurements, scope (E.2), build order (E.3) and production requirements (F)
- `ditteau_data_transform/docs/forecasting/point_in_time_findings.md`: its §E is
  superseded by the assessment's §G
- [ADR-002: Delete-Where Flag for Deposit Loader](adr-002-delete-where-flag-deposit-loader.md)
- [ADR-004: Category-Based Data Retention Policy Schema](adr-004-retention-policy-category-based-schema.md)
- [ADR-009: dbt Build Provenance via on-run-end Hook](adr-009-dbt-build-provenance.md)
- [ADR-011: Time-Series Forecast Schema Design](adr-011-time-series-forecast-schema.md): its training series should read certified census runs from `DECLARE`
- [ADR-013: Point-in-Time Feature Table](adr-013-point-in-time-feature-table.md): "What this model does not promise"
- `ditteau_data_transform/docs/governance/data_access_policy.md`: L.3 retention, O-49, O-54, O-55
- `ditteau_schema_registry/redaction_register.yml`
- `ditteau_data_infra/deposit_loader/deposit_loader.py`, `README.md` (scheduled phase)
- `ditteau_data_transform/scripts/clone_deposit.py`
