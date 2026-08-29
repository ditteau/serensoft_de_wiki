# ADR-009: dbt Build Provenance via on-run-end Hook

**Status:** Accepted

**Date:** 2026-08-28

**Author:** LVP

---

### Context

Gap C-02 (also tracked as G-10) has sat in §7.5 of `data_governance_policy.md` as
"No dbt run audit table", LOW risk, P3, with the remediation "Persist run_results
via on-run-end hook". Nothing had been built: `dbt_project.yml` carried no
`on-run-end` entry at all.

ADR-008 established the habit of asking what the platform already records before
building anything, and that question has a real answer here. dbt already stamps
every query it issues, and Snowflake keeps it for 365 days:

```json
/* {"app": "dbt", "dbt_version": "1.11.2", "profile_name": "ditteau_data_transform",
    "target_name": "demeau_prod", "node_id": "test....c26ea5525e"} */
```

So per-node SQL, timing, rows and errors are already free. Unlike C-01, though,
that does not close the gap. Four things dbt knows that `QUERY_HISTORY`
structurally cannot carry:

- **No `invocation_id` in the comment**, so queries cannot be grouped back into a
  run — which is exactly what a *run* audit table is.
- **Test outcomes are invisible.** A failing dbt test's SQL succeeds; the failure
  is dbt's reading of the rows returned. `QUERY_HISTORY` records SUCCESS.
- **Skipped nodes issue no query at all**, so a model skipped because its parent
  failed leaves no trace anywhere.
- **`warn` vs `error` severity** is a dbt concept with no SQL equivalent, and it
  matters here because `assert_aggregate_models_cleared` warns by design.

`QUERY_HISTORY` answers *what SQL ran*; this answers *what dbt concluded*.

---

### Decision

**Scope is build provenance, not observability.** The table answers "what was
built, by whom, into which target, and did it succeed" — a compliance record that
pairs with C-01's access log. It is deliberately not a performance or flaky-test
store. `execution_time` is recorded because it is free, not because trend analysis
belongs here.

- `{DB}.DBT_AUDIT.DBT_RUN_AUDIT` in all nine databases, created by
  `add_dbt_run_audit_2026-08-28.sql`. One row per node per invocation, carrying
  `invocation_id`, run start, command, dbt version, school, target, the Snowflake
  user and role that ran it, and per node the id, name, resource type, status,
  execution time, rows affected, failure count and message.
- `macros/audit/log_dbt_results.sql`, wired as the project's only `on-run-end`
  entry and guarded by `var('enable_run_audit', true)`.
- Per-school rather than platform-level, unlike C-01. Each school's dbt role
  writes only its own database, so tenant isolation is preserved by construction.
- `tests/governance/assert_dbt_run_audit_written.sql` warns if the table has rows
  but the newest is over 30 days old.

**A failed audit write must never fail the build.** dbt fails a run when an
`on-run-end` hook errors, so every INSERT is wrapped in a Snowflake Scripting
block with `EXCEPTION WHEN OTHER`, which swallows the error server-side. Verified
by pointing the macro at a nonexistent table: the build reported
`Completed successfully` / `PASS=20 WARN=0 ERROR=0` and wrote nothing.

That verification is also the argument for the assertion. A green build and an
empty audit table look identical from the run log, so without
`assert_dbt_run_audit_written` the compliance record could go silently empty.

Writes are chunked at 200 rows. A clean full build is ~1,189 nodes, which puts a
single rendered `INSERT ... VALUES` in the same order of magnitude as Snowflake's
1MB statement cap once failure messages are included — a statement that works
until the day a build fails badly is worse than one that never approaches the
limit. The multi-chunk path was tested by temporarily setting the chunk size to 5.

**Approval:** the Asana task records C-02's gate as WDT, and the project requires
WDT approval for production merges. LVP took this decision on 2026-08-28 and it is
recorded as LVP-approved. Noted explicitly because the task metadata and the
approving party differ.

---

### Consequences

#### Pros

- Records the one thing nothing else in the estate captures: dbt's own verdict.
  First run already caught `assert_aggregate_models_cleared` at `warn` with **21
  failures** — the O-7b backlog — an outcome `QUERY_HISTORY` reports as SUCCESS.
- Cannot break a build, verified rather than asserted.
- Per-school tables keep tenant isolation intact with no cross-database seam.
- Costs one extra query per chunk per invocation; nothing runs on a schedule.

#### Cons

- **The hook does not run if dbt fails to parse or compile.** The catastrophic
  failures are exactly the ones absent from the table. A gap in the record is not
  evidence of a clean period, and someone will eventually read it as one.
- **The never-fail guarantee is what makes silent emptiness possible.** The
  assertion mitigates but does not remove this; it fires after 30 days, not
  immediately.
- Residual failure mode: a syntax error in the generated block would still raise,
  because a block cannot catch its own compilation. That is a macro bug rather
  than an operational condition.
- Every dbt run here is manual — no CI, nothing scheduled — so this is a record of
  ad-hoc workstation builds. Useful, and not the pipeline audit the name suggests.
- Under `--select`, only selected nodes appear. Correct, but a partial run must
  not be read as a full build.
- Nine more tables to keep in step, and a tenth schema per database.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **on-run-end hook into a per-school table (chosen)** | Captures dbt's verdict; tenant-isolated; no new dependency | Silent on parse/compile failure; one more schema per database |
| Rely on `QUERY_HISTORY` alone | Free, already retained 365 days, zero build | Cannot group a run, cannot see test outcomes, cannot see skipped nodes |
| `dbt_artifacts` package (Brooklyn Data) | Maintained, models included, richer schema | Another package dependency and several new models in a project that keeps its surface deliberately small; more than a P3 compliance record needs |
| Reuse `{DB}.DBT_TEST_RESULTS` | Schema already provisioned and granted in all nine databases | Holds 751–976 tables and 562k–916k rows per database from `store_failures`; an audit table there is invisible. Also the schema O-13 is open against |
| Platform-level table like C-01's | One place to read across tenants | C-01 is cross-tenant by nature; this is not, and a shared table would create a seam principle 1 does not require |

---

### References

- Migration: `/Users/laurievanpelt/ditteau_data_infra/school_setup/migrations/add_dbt_run_audit_2026-08-28.sql`
- Macro: `/Users/laurievanpelt/ditteau_data_transform/macros/audit/log_dbt_results.sql`
- Assertion: `/Users/laurievanpelt/ditteau_data_transform/tests/governance/assert_dbt_run_audit_written.sql`
- Policy: `data_governance_policy.md` §7.5 (compliance gaps — C-02 / G-10)
- [ADR-008: PII Access Logging on Built-In ACCESS_HISTORY](adr-008-pii-access-logging-access-history.md) — the companion record, and the source of the "check what already exists first" habit

---

### Footnote: a grant trap found on the way

`GRANT SELECT ON FUTURE TABLES IN SCHEMA ...` is an **account-level** privilege
despite naming a schema, so it requires `MANAGE GRANTS`. `SYSADMIN` owns these
schemas and still cannot issue it.

This is the inverse of the `CREATE ROW ACCESS POLICY` trap already documented:
there, reaching for `SECURITYADMIN` is wrong because a policy is a schema object;
here, staying on `SYSADMIN` is wrong because a future grant outlives the objects
it describes. Ownership decides object DDL; `MANAGE GRANTS` decides grants.

⚠️ `add_dbt_test_results_schema.sql` (2026-08-15) carries the same six statements
under "Required role: SYSADMIN" and would fail today. It succeeded when it ran
because person users still held ambient `ACCOUNTADMIN` via
`DEFAULT_SECONDARY_ROLES = ["ALL"]`, removed 2026-08-24. Re-running it now hits
the error. This is O-15 reconciliation work, not C-02's, and was left alone.
