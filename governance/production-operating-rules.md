# Ditteau Data — Production Operating Rules

**Status:** Active
**Effective:** 2026-08-16
**Owner:** LVP (Data Engineering) · Governance sign-off: KKM · Security: WDT
**Applies to:** all `{SCHOOL}_DD_{DEV,TEST,PROD}` databases

This document states how the Ditteau platform runs in production: what each environment
is for, how data moves between them, who is permitted to write, and what must be
observed before a change is considered done.

It is a rules document, not a runbook. Procedures live in
[runbooks/row-access-policies.md](../runbooks/row-access-policies.md). The reasoning
behind several of these rules is in
[governance/governance-and-production-readiness-review.md](governance-and-production-readiness-review.md).

---

## 1. Environment roles

| Environment | Purpose | Consumers |
|---|---|---|
| **PROD** | Environment of record. The only environment anyone consumes from. | `{CODE}_REPORTING_PROD`, business roles, Streamlit apps, BI tools |
| **TEST** | CI and pre-release verification | Automated pipelines only |
| **DEV** | Engineering workbench | Engineers |

**Rule 1.1** — PROD is the only environment with a `REPORTING` role, an `ANALYTICS`
warehouse, and warehouse grants for business-function roles. No dashboard, report or
external consumer may point at DEV or TEST.

**Rule 1.2** — DEV and TEST carry no service-level expectation. They may be rebuilt,
truncated or cloned over at any time without notice.

---

## 2. Data flow

**Rule 2.1** — Institutional feeds land in **PROD only**. A school delivers one export;
that export goes to `{CODE}_DD_PROD.DEPOSIT` via `deposit_loader`.

**Rule 2.2** — DEV and TEST receive deposit data by **zero-copy clone from PROD**, using
`scripts/clone_deposit.py`. Never the reverse.

```bash
python scripts/clone_deposit.py --src DEMEAU_DD_PROD --dst DEMEAU_DD_TEST
```

**Rule 2.3** — Deterge and distribute are **never copied** between environments. Each
environment rebuilds them from its own deposit via dbt. Copying built layers would
detach them from the deposit that produced them.

**Rule 2.4** — `_LOAD_HISTORY` is never cloned. Its `environment` column records where a
load happened; copying it makes the target misreport its own provenance.

**Rule 2.5** — Cross-school cloning is prohibited. `clone_deposit.py` refuses it. In a
multi-tenant account a mistyped database name is a cross-institution disclosure.

> Shares and `DITTEAU_SHARED` are environment-independent by design. All three
> environments read the same `{CODE}_CX_ARCHIVE` share live. This is correct and is not
> an exception to 2.2.

---

## 3. Who may write

**Rule 3.1** — PROD and TEST dbt runs execute as the dedicated service user
`SVC_{CODE}_DBT_{ENV}`, never as a person with a role override.

**Rule 3.2** — Service users are `TYPE = SERVICE`, authenticate by key pair, hold
exactly one role, and carry `DEFAULT_SECONDARY_ROLES = ()`.

**Rule 3.3** — Setting a role at connect time does **not** scope a person's session.
Accounts with `DEFAULT_SECONDARY_ROLES = ["ALL"]` keep `ACCOUNTADMIN` active as a
secondary role regardless of the primary role requested, and object access resolves
against the union. Any session used to test or verify access must begin:

```sql
USE SECONDARY ROLES NONE;
SELECT CURRENT_ROLE(), CURRENT_SECONDARY_ROLES();  -- confirm before trusting results
```

**Rule 3.4** — Engineers hold `DITTEAU_ENGINEER`: full in DEV and TEST, **read-only in
PROD**. Routine PROD writes go through the service account. Administrative changes go
through a reviewed migration in `ditteau_data_infra/school_setup/migrations/`.

**Rule 3.5** — Service roles are not granted `CREATE SCHEMA`. Schemas are provisioned by
migration. A transform role that can mint schemas lets a mistyped config create one
silently instead of failing.

**Rule 3.6** — No school's dbt role may hold write access to `DITTEAU_SHARED`. Shared
reference data is read-only to tenants; one tenant's build must not be able to corrupt
data every other tenant reads.

---

## 4. What can and cannot be rebuilt

Three classes of object, requiring different handling.

| Class | Objects | `--full-refresh` |
|---|---|---|
| **Derivable** | staging views, dimensions, facts, marts | Safe |
| **Accumulating** | the five `snap_*` models | **Destroys history** that cannot be recreated |
| **Minted identity** | `int_ditteau_id_registry` | **Re-mints every ID**; breaks every external reference |

**Rule 4.1** — `int_ditteau_id_registry` is append-only and must never be
full-refreshed. The model enforces `full_refresh=false`; do not override it.

**Rule 4.2** — The `snap_*` models hold point-in-time history that depends on what
deposit looked like on past dates. Full-refresh recreates only what current deposit can
produce.

**Rule 4.3** — Identity is order-dependent. `ditteau_id` is deterministic given
`(institution, anchor_source_system, anchor_source_id)`, but *which* source becomes the
anchor depends on what data existed when the person first appeared. Two environments
populated in a different order can assign different `ditteau_id` values to the same
person. Do not join identity across environments.

---

## 5. Governance controls

**Rule 5.1** — `enable_row_level_security` and `enable_masking_policies` are **false in
all environments** until the preconditions in §5.3 are met for that environment.
Flipping either is a governance decision requiring KKM sign-off, not a deployment step.

**Rule 5.2** — Access tiers resolve `COALESCE(user_domain_access, role_domain_access)`.
The user-keyed path exists because Streamlit in Snowflake runs owner's-rights, so
`CURRENT_ROLE()` returns the app owner for every viewer.

**Rule 5.3 — Preconditions before enabling row-level security in an environment:**

1. The three domain policies exist in that database
2. `{CODE}_DBT_{ENV}` holds `APPLY` on each of them
3. `role_domain_access` is populated
4. `user_domain_access` is populated for every user who will access through a
   Streamlit app — **an empty grid denies everything under owner's-rights**
5. `advisor_student_map` is populated, or no role holds `SCOPED` in that environment
6. Attachment is confirmed by querying `information_schema.policy_references`

**Rule 5.4 — Preconditions before enabling masking:** the policy set exists in that
database, and `role_pii_unmask` / `user_pii_unmask` are populated. Both grids default to
deny by absence: enabling masking against empty grids masks the column for everyone,
including the registrar.

**Rule 5.5** — `{CODE}_STREAMLIT_OWNER_{ENV}` must **never** appear in
`role_domain_access`, including with tier `NONE`. Absence and `NONE` both deny today, but
an explicit row invites a later edit to `FULL`, which would grant every viewer of that
app full access simultaneously. Enforced by
`tests/governance/assert_no_app_owner_in_role_domain_access.sql`.

**Rule 5.6** — Row access and column masking resolve through **separate** grids. Widening
a role's row access must not silently unmask PII.

**Rule 5.7** — Read the entitlement grid across a role's row, not down a domain's column.
Snowflake permits one row access policy per table, so a conformed dimension has a single
domain owner, and that ownership fixes the tier every role gets on it regardless of their
tier elsewhere.

---

## 6. Verification standard

**Rule 6.1** — A change is complete when its **effect is observed**, not when the code is
written or the command exits zero.

| Change | Required observation |
|---|---|
| Policy attached | a row in `information_schema.policy_references` |
| Procedure deployed | a successful call |
| Table created | a `SELECT` in the target account |
| Grant issued | a read performed **as the grantee** |
| Test added | a confirmed failure when its invariant is deliberately broken |

**Rule 6.2** — Exit code zero means "nothing raised", not "something happened". Piping a
command through another process replaces its exit code. Confirm the effect independently.

**Rule 6.3** — Flag-gated code is untested code. Anything behind a flag that has never
been true has never run. Exercise it with the flag on in DEMEAU DEV, observe, revert.

**Rule 6.4** — Deployment state is verified against Snowflake, never inferred from the
repository. A migration file in git is evidence of intent, not of execution.

**Rule 6.5** — When fixing a defect, check whether its siblings share it. Repeated
finding: fixes reach the model, policy or environment in front of you and not the others
beside it.

---

## 7. Prohibited actions

| Never | Why |
|---|---|
| `CREATE OR REPLACE SCHEMA` on a deposit schema | Destroys `INGEST_STAGE` and the direct `WRITE` grant no future grant covers |
| dbt `+grants:` configuration | Revokes working future grants |
| Full-refresh of `int_ditteau_id_registry` | Re-mints every `ditteau_id` |
| Adding `STREAMLIT_OWNER` to `role_domain_access` | See 5.5 |
| Granting a school's dbt role write on `DITTEAU_SHARED` | See 3.6 |
| Testing access as `SYSADMIN` or `ACCOUNTADMIN` | Policies short-circuit to `TRUE` |
| Passing `--vars` or `--target` to a school run script | Replaces the var block; use `EXTRA_VARS` and `ENV` |

---

## 8. Current state — 2026-08-16

| | DEMEAU | Merrimack | Anselm |
|---|---|---|---|
| DEV / TEST / PROD built | ✅ all three | DEV only | DEV only |
| Service users in use | TEST, PROD | — | — |
| RLS enabled | ❌ | ❌ | ❌ |
| Masking enabled | ❌ | ❌ | ❌ |
| Masking policies deployed | DEV only | none | none |
| `user_domain_access` populated | ❌ | ❌ | ❌ |
| Advisor roster populated | ❌ | ❌ | ❌ |

Open dependencies outside engineering control:

- No school has advisor data in any source. Anselm's Workday export carries no advisor
  column; Merrimack's J1 advisor tables are empty. `SCOPED` cannot function until a
  school supplies the field.
- No institutional staff hold Snowflake accounts.
- Nine service-account private keys are held on an engineer workstation pending a
  secrets-manager decision.
