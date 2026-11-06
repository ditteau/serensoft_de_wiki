# Jenzabar Source Feature Flags

How the Jenzabar CX and Jenzabar One feature flags gate models in `ditteau_data_transform`, school by school. The short rules are in [dbt Conventions](conventions.md#var-guards-for-optional-sources). This page explains how the flags actually flow through the project.

> **As of 2026-11-05.** The model counts and per-school values below were measured from the repo on that date. The run scripts are the source of truth; re-check them before relying on this page.

---

## Naming note

There are no vars called `has_j1` or `has_cx`. The real flags are `has_jenzabar_one` and `has_jenzabar_cx`, plus two narrower ones, `has_j1_finance` and `has_jcx_deposit`. A misspelled var name doesn't raise an error: `var('has_j1', false)` just returns `false`, and every model it guards drops out of the build.

---

## The flags

All of them default to `false` in `dbt_project.yml`. Each school's run script sets the real values.

| Var | Meaning | What it gates |
|---|---|---|
| `has_jenzabar_cx` | School has a Jenzabar CX archive, read through a Snowflake data share | All 59 `staging/jenzabar_cx/` models, the CX arms in intermediate routers, and CX blocks in several marts |
| `has_jenzabar_one` | School has a Jenzabar One (J1) deposit, loaded from CSV | All 117 `staging/jenzabar_one/` models and the J1 intermediate models |
| `has_j1_finance` | J1 general-ledger lineage (Merrimack only) | The GL chain. Only takes effect together with `has_jenzabar_one` |
| `has_jcx_deposit` | Extra CX rows in the DEPOSIT schema alongside the share (DEMEAU only) | The `jenzabar_cx_deposit` source and the UNION inside `jcx_base()` |
| `primary_sis` | Not a flag, but works with them: which SIS wins | Identity anchor in `int_ditteau_id_registry`, and precedence in `dim_integration_ids`, `int_students`, `int_student_race` and `int_program_crosswalk` |

> **`has_jenzabar_cx: true` does not mean CX is the primary SIS.** Most schools keep a CX archive alongside the system they actually run on. Precedence comes from `primary_sis`, never from which flags are on.

---

## Values by school

These come from `scripts/run_<school>*.sh`. The comment block in `dbt_project.yml` is a reading aid and can fall behind the scripts.

| School | `primary_sis` | `has_jenzabar_cx` | `has_jenzabar_one` | `has_j1_finance` | `has_jcx_deposit` |
|---|---|---|---|---|---|
| Merrimack | `JENZABAR_ONE` | true | true | true | false |
| Saint Anselm | `WORKDAY` | true | false | false | false |
| DEMEAU (demo, generated data) | `JENZABAR_ONE` | true | true | false | true |

Colby, Endicott and Springfield are not provisioned. Their planned profiles are in the `dbt_project.yml` comment block.

---

## The four gating patterns

### A. Folder-level `enabled` for staging

`dbt_project.yml` turns whole staging folders on or off:

```yaml
staging:
  jenzabar_cx:
    +enabled: "{{ var('has_jenzabar_cx', false) }}"
  jenzabar_one:
    +enabled: "{{ var('has_jenzabar_one', false) }}"
```

Every J1 staging model also repeats `enabled = var('has_jenzabar_one', false)` in its own `config()`. When the flag is false, these models are **removed from the dbt graph** entirely, not just skipped.

### B. `enabled=` on source-specific models

Leaf models that belong to one source carry their own guard:

| Model | Guard |
|---|---|
| `int_jcx__student_race`, `int_jcx__applicants` | `var('has_jenzabar_cx', false)` |
| `int_j1__student_race`, `int_j1_student_versions` | `var('has_jenzabar_one', false)` |
| `int_j1__gl_accounts`, `dim_gl_account`, `fact_gl_actual`, `fact_gl_actual_review`, `fact_gl_budget`, `fact_gl_period_balance`, `mart_finance_budget_variance_j1` | `var('has_jenzabar_one', false) and var('has_j1_finance', false)` |

### C. Routers with Jinja `{% if %}` arms

Cross-source models always stay enabled. The flags decide which arms of a UNION ALL get rendered:

```sql
WITH all_sources AS (
  {% if var('has_jenzabar_cx', false) %}
  SELECT * FROM {{ ref('int_jcx__student_race') }}
  {% endif %}

  {% if var('has_jenzabar_one', false) %}
  UNION ALL
  SELECT * FROM {{ ref('int_j1__student_race') }}
  {% endif %}

  {% if not (var('has_jenzabar_cx', false) or var('has_jenzabar_one', false)) %}
  -- No source enabled: empty set matching the contract
  SELECT NULL::VARCHAR AS student_id
  {% endif %}
)
```

After the union, the router ranks rows by `primary_sis`, so the primary system wins and the other sources fill gaps. Routers built this way: `int_student_race`, `int_students`, `int_ditteau_id_registry`, `dim_integration_ids`, `int_program_crosswalk`, `int_applicants`.

### D. Empty stubs via `{% if %}…{% else %}`

Some models stay in the graph for every school and output a typed empty result when the flag is off, so anything that refs them still compiles. Examples: `int_jcx__class_level_codes`, `int_jcx__program_catalog`, `int_jcx__source_column_coverage`.

```sql
{% if var('has_jenzabar_cx', false) %}
  -- real logic
{% else %}
SELECT
  NULL::VARCHAR AS class_level_code,
  NULL::BOOLEAN AS is_registered_code
WHERE 1 = 0
{% endif %}
```

---

## Rules

1. **Never put `enabled=false` on a model that another model `ref()`s.** A disabled model leaves the graph, and any `ref()` to it fails at runtime. That includes a `ref()` inside an `{% if %}` block that never executes, and compile does not catch it. Router arms stay enabled and use an empty stub or a `WHERE 1=0` body. See [Silent Failure Modes](silent-failure-modes.md).
2. **Router-arm guards go in the router's Jinja, not in the arm's config.** A config-level `enabled=var('has_x')` is fine only on leaf staging models that are not router arms.
3. **A flag is a claim, not a measurement.** Check that the deposit tables exist, and run the check as `DITTEAU_ADMIN`, because `SYSADMIN` silently sees fewer objects:

   ```sql
   SELECT table_name
   FROM {DB}.INFORMATION_SCHEMA.TABLES
   WHERE table_schema = 'DEPOSIT';
   ```

4. **Pass extra vars with `EXTRA_VARS`, never `--vars`.** dbt *replaces* the var dict with the last `--vars` it receives, so passing a single var on the command line wipes out every flag the script set. The run scripts reject `--vars` and merge `EXTRA_VARS` on top of their base block:

   ```bash
   EXTRA_VARS='{"enable_row_level_security": true}' \
     bash scripts/run_merrimack_dev.sh build --select dim_student
   ```

5. **`has_j1_finance` fails silently.** Any new `run_merrimack_*` script (test or prod) must set it to true. If it doesn't, the GL models leave the graph, and a missing model looks exactly like a model with nothing to build.
6. **Slate and PowerFAIDS are DEMEAU-only.** Do not turn on `has_slate` or `has_powerfaids` for Anselm or Merrimack (O-46/O-47 in the governance register).

---

## Related macros

| Macro | How it relates to the flags |
|---|---|
| `jcx_base(table)` | Selects from the CX share, and adds the deposit rows with `UNION ALL` when `has_jcx_deposit` is true. ⚠️ **No model calls it.** All 70 CX source reads use `source('jenzabar_cx_archive', ...)` directly, so DEMEAU's CX deposit rows have never been unioned in. |
| `safe_col(relation, col, alias, default, expression)` | Handles differences *within* an enabled source: when a column is missing from one school's copy, it outputs the default instead of failing. |
| `school_ref(model)` | `ref()` that resolves to a `_school_overrides/` model for the active `school_code`. The override list is hard-coded in the macro (Anselm has 6 CX overrides, Merrimack has 1). Override folders are enabled by `school_code`, not by any `has_*` flag. |

See also the [Macros Reference](macros.md).

---

## Source locations

| Source | Database | Schema |
|---|---|---|
| CX share | `var('jcx_share_database')`, e.g. `MERRIMACK_CX_ARCHIVE` | `var('jcx_share_schema')`, which is `DITTEAU_ARCHIVE` for every school |
| CX deposit | `var('jcx_deposit_database')` | `DEPOSIT` (source enabled by `has_jcx_deposit`) |
| J1 | `var('j1_source_database')` | `var('j1_source_schema')` |

Merrimack's J1 source is pinned to `MERRIMACK_DD_DEV.DEPOSIT`. DEMEAU's run script derives its database from `ENV` (`DEMEAU_DD_{DEV,TEST,PROD}`).

---

## Known issue: three routers fail without CX

Every current school has `has_jenzabar_cx: true`, so none of these has fired yet. A school without a CX archive would hit all three on its first build:

- **`int_student_versions` and `int_aid_awards`** make unguarded `ref()`s into CX models that read disabled `stg_jcx__*` models. They fail before any union is rendered. This is rule 1 above, broken.
- **`int_student_race`** starts its non-CX arms with `UNION ALL`. When CX is off, the first rendered arm begins with a bare `UNION ALL` and fails to compile.

The authority is `data_access_policy.md` §O, **O-64**, in `ditteau_data_transform/docs/governance/`.

---

## Related

- [dbt Conventions](conventions.md): var guards, naming, metadata columns
- [Silent Failure Modes](silent-failure-modes.md): why disabled models fail at runtime
- [Snowflake New School Setup](../runbooks/snowflake-new-school.md): where a new school's flags get set
