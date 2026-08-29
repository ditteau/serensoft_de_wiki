# Streamlit in Snowflake — Workspaces Runbook

Operational guide for deploying the DEMEAU dashboards into `DEMEAU_DD_PROD` as
Streamlit in Snowflake apps, backed by a Git repository and served through
Workspaces.

**Last updated:** 29 August 2026
**Maintained by:** LVP

---

## Contents

- [A. What exists today](#a-what-exists-today)
- [B. The tier problem, and why two of three apps are gated](#b-the-tier-problem-and-why-two-of-three-apps-are-gated)
- [C. Prerequisites — all applied 2026-08-29](#c-prerequisites--all-applied-2026-08-29)
- [D. Running the identity probe](#d-running-the-identity-probe)
- [E. Deploying an app from a Git-backed Workspace](#e-deploying-an-app-from-a-git-backed-workspace)
- [F. Verification](#f-verification)
- [G. Traps](#g-traps)

---

## A. What exists today

### A.1 Deployed apps, as of 2026-08-29

| App | Location | Owner | Note |
|---|---|---|---|
| Ditteau Data KPI Library | `DEMEAU_DD_DEV.DISTRIBUTE` | `SYSADMIN` | warehouse runtime |
| Ditteau Role Based Demo Dashboard | `DEMEAU_DD_DEV.DISTRIBUTE` | `SYSADMIN` | warehouse runtime |
| Ditteau Data Dashboard Template | `MERRIMACK_DD_DEV.DISTRIBUTE` | `SYSADMIN` | warehouse runtime |
| DITTEAU DATA GOVERNANCE DEMEAU | `USER$LVANPELT.PUBLIC` | `LVANPELT` | Workspaces |
| Preview of `Test_Gov` | `USER$LVANPELT.PUBLIC` | `LVANPELT` | Workspaces |

**Nothing is deployed in any PROD database.** The two Workspaces apps sit in a
personal database: no teammate can reach them, and their code has no lineage
back to a commit.

⚠️ Both DEV apps are owned by `SYSADMIN`, which is on the bypass list in every
RAP and every masking policy. They have never exercised the enforcement path, so
their working in DEV is not evidence that anything works in PROD.

### A.2 Target state

Three apps in `DEMEAU_DD_PROD.DISTRIBUTE`, owned by
`DEMEAU_STREAMLIT_OWNER_PROD`, deployed from a Workspace backed by
`DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM`:

| App | Source file | Status |
|---|---|---|
| Governance (recorded capture) | `streamlit/demeau_governance_sis.py` | ready to deploy |
| Enrollment v2 | `streamlit/demeau_enrollment_dashboard_v2.py` | gated — see B |
| Role-based demo | `streamlit/demeau_role_dashboard.py` | gated — see B |

Apps go in `DISTRIBUTE` rather than a dedicated schema because both existing DEV
apps do, and because F.5 of the access policy governs where *models* live and is
silent on application objects. The cost is that `DISTRIBUTE` now holds non-dbt
objects in a dbt-managed schema — dbt will not touch them, but a cleanup script
might.

---

## B. The tier problem, and why two of three apps are gated

### B.1 The mechanism

Streamlit in Snowflake runs with **owner's rights**. Every query executes as the
app's owner role, whoever is viewing.

`DEMEAU_STREAMLIT_OWNER_PROD` is **deliberately absent** from
`role_domain_access`, and `user_domain_access` is empty. Every RAP resolves

```
COALESCE( user_domain_access tier ,  role_domain_access tier )
```

which is `COALESCE(NULL, NULL)`, and falls through to the policy's trailing
`ELSE FALSE`.

That absence is a control, not an oversight. It is asserted by
`tests/governance/assert_no_app_owner_in_role_domain_access.sql`, whose comment
explains why an explicit `NONE` row is *not* an acceptable substitute: a `NONE`
invites a later well-meaning edit to `FULL`, which would grant every SiS viewer
full row access **and** unmasked PII at once, because masking resolves through
the same lookup.

### B.2 Measured, not inferred

Run as the app owner role against `DEMEAU_DD_PROD` on 2026-08-29:

| Object | Rows | RAP |
|---|---|---|
| `GOVERNANCE.DEMO_PERSONA_RESULTS` | 289 | none |
| `MART_ENROLLMENT_CENSUS` | 163 | none |
| `MART_SECTION_UTILIZATION` | 27,888 | none |
| `DIM_STUDENT` | **0** | `rap_student_academic` |
| `FACT_ENROLLMENT` | **0** | `rap_student_academic` |

Ten objects carry a RAP in PROD and all ten read zero for this role:
`dim_student`, `fact_student_term`, `fact_enrollment`, `mart_student_at_risk`,
`mart_academic_progress`, `mart_registration_holds`, `dim_applicant`,
`fact_application`, `fact_aid_award`, `mart_student_health_holds`.

Pre-aggregated marts carry no RAP by design — that is what makes the
`AGGREGATED` tier meaningful — so they are unaffected.

### B.3 What this means per app

- **Governance SiS** reads exactly one non-RAP table. Fully functional. Deploy it.
- **Enrollment v2** loses the tabs backed by `FACT_STUDENT_TERM`,
  `MART_ACADEMIC_PROGRESS` and `MART_REGISTRATION_HOLDS`. Census, demographics,
  admissions, aid, scorecard and both snapshots still work.
- **Role dashboard** loses `DIM_STUDENT`, `FACT_ENROLLMENT` and
  `FACT_STUDENT_TERM` — most of it.

### B.4 Why the probe gates the fix

The documented remedy is to entitle viewers through `user_domain_access`. That
table keys on `CURRENT_USER()`.

Snowflake documents `CURRENT_USER()` as returning the **viewer** under the
warehouse runtime, and the **owner's context** under the container runtime.
**Workspaces supports container runtimes only.** So the remedy may not work at
all on the runtime we are deploying to — and no one has measured which behaviour
this account exhibits.

Until that measurement exists, any `user_domain_access` row added for a demo
viewer is a guess that fails silently: the dashboard renders normally whichever
tier it applied.

⚠️ Do not resolve this by adding `DEMEAU_STREAMLIT_OWNER_PROD` to
`role_domain_access`. It breaks a CODEOWNERS-protected assertion and hands every
future SiS viewer the same tier indiscriminately.

⚠️ Do not resolve it by creating a differently-named owner role that *does* hold
tiers. `assert_no_app_owner_in_role_domain_access` matches on
`ilike '%STREAMLIT_OWNER%'`, so a role named around it passes the test while
defeating the control. That is worse than the honest change, because it also
hides.

---

## C. Prerequisites — all applied 2026-08-29

### C.1 Grants

`add_streamlit_owner_prod_app_grants_2026-08-29.sql`:

| Grant | Why |
|---|---|
| `USAGE ON SCHEMA DEMEAU_DD_PROD.GOVERNANCE` | revived a dead grant — see C.2 |
| `USAGE ON SCHEMA DEMEAU_DD_PROD.DISTRIBUTE` | restated |
| `CREATE STREAMLIT ON SCHEMA ...DISTRIBUTE` | create the app object |
| `CREATE STAGE ON SCHEMA ...DISTRIBUTE` | app files land in a stage |
| `USAGE ON COMPUTE POOL SYSTEM_COMPUTE_POOL_CPU` | **container runtime cannot start without it** |
| `ROLE ... TO USER LVANPELT` | a role granted to nobody is invisible in the picker |

Not granted, deliberately: `READ SESSION` (no PROD app calls `st.user`), and
`SELECT` on the entitlement grids (the capture snapshots them, so the viewer
needs exactly one object).

### C.2 The dead grant

`DEMEAU_STREAMLIT_OWNER_PROD` was granted `SELECT` on
`GOVERNANCE.DEMO_PERSONA_RESULTS` on 2026-08-18 but never `USAGE` on the
`GOVERNANCE` schema. Schema `USAGE` is required to reach a table inside it, so
that `SELECT` was inert for eleven days and the governance viewer would have
failed on deploy.

⚠️ `SHOW GRANTS TO ROLE` lists the `SELECT` and reads as provisioned. Only
`SHOW GRANTS ON SCHEMA` reveals the gap. **An object grant is not evidence a
principal can read the object** — check the schema too, or just query as the role.

### C.3 Git repository

`DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM`, reusing
`DITTEAU_GIT_INTEGRATION` (allowed prefix is the whole `ditteau` org) and
`DITTEAU_GITHUB_SECRET`.

```sql
ALTER GIT REPOSITORY DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM FETCH;
LS @DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM/branches/main/streamlit/;
```

### C.4 Warehouses

`DEMEAU_ANALYTICS_PROD` serves the apps. Its monitor was raised 25 → 50 credits
and its 100% trigger changed from `SUSPEND_IMMEDIATELY` to `NOTIFY` on
2026-08-29, so a monitor can no longer cut off a live client demo.
`DEMEAU_MONITOR_TRANSFORM_DEV` was raised 50 → 100 after August consumed 69.76.

---

## D. Running the identity probe

The probe lives in `streamlit/governance_identity_probe/` and has its own README
covering grants and interpretation. This section is the operational sequence.

### D.1 What it must answer

Under the **container runtime**, does `CURRENT_USER()` return the viewer or the
app owner? If the viewer, `user_domain_access` can carry per-viewer demo
entitlement and ADR-003 Phase 6 holds on Workspaces. If the owner, it cannot,
and the analytics dashboards need a different design.

### D.2 Prerequisites, already in place

- `DEMEAU_STREAMLIT_OWNER_DEV` holds `READ SESSION`, compute pool `USAGE`,
  `CREATE STREAMLIT`/`CREATE STAGE` on `DEMEAU_DD_DEV.GOVERNANCE`, and `SELECT`
  on both tier tables.
- The role is granted to `LVANPELT`, `WTRILLICH`, `KMARIE` — so all three see it
  in the **App executes as** picker.
- `DEMEAU_TRANSFORM_DEV` is running again (it was quota-suspended until
  2026-08-29, which blocked the probe entirely).

### D.3 Sequence

1. Snowsight » **Workspaces** » **+ Add new** » **Streamlit app**.
2. Replace the generated `streamlit_app.py` and `pyproject.toml` with the two in
   `streamlit/governance_identity_probe/`.
3. **Leave the generated `snowflake.yml` alone.** Edit it through the deploy
   dialog. It carries the account's current compute-pool and runtime keys; a
   hand-authored one drifts from them.
4. **App settings » Execution**:
   - App executes as — `DEMEAU_STREAMLIT_OWNER_DEV`, **not** `SYSADMIN`
   - Query warehouse — `DEMEAU_TRANSFORM_DEV`
   - Compute pool — `SYSTEM_COMPUTE_POOL_CPU`
   - Artifact repositories — empty
5. Deploy into `DEMEAU_DD_DEV.GOVERNANCE` and grant usage to the reviewer roles.
6. **Three different people open the same deployed app** and each copies the
   Section F findings block. One run proves nothing — the whole question is
   whether the value varies by viewer.

### D.4 Reading the result

- `CURRENT_USER()` **differs** across the three → the user leg tracks the viewer;
  Phase 6 holds on this runtime.
- `CURRENT_USER()` **identical** across the three → it returns the owner;
  `user_domain_access` cannot entitle SiS viewers, and B.4's remedy is dead.

⚠️ An empty or errored caller's-rights column in Section D means the D.4 caller
grants were skipped. That is evidence about the grants, not about the runtime.

⚠️ Running the probe as `SYSADMIN` invalidates it: `SYSADMIN` bypasses every RAP
and every masking policy, so it measures nothing about how real apps behave.

---

## E. Deploying an app from a Git-backed Workspace

### E.1 Push first

Snowflake serves the last `FETCH`, not GitHub's current state.

```bash
git push origin main
```
```sql
ALTER GIT REPOSITORY DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM FETCH;
```

⚠️ `FETCH` reports `is up to date. No change was fetched.` both when origin
genuinely has not moved **and** when you forgot to push. The two are
indistinguishable from the output. A successful fetch shows
`Branch | main | FAST_FORWARD`.

### E.2 Create the workspace

Snowsight » **Workspaces** » **+ Add new** » **From Git repository**, selecting
`DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM`, branch `main`.

### E.3 Deploy

**App settings » Execution**:

| Setting | Value |
|---|---|
| App executes as | `DEMEAU_STREAMLIT_OWNER_PROD` |
| Query warehouse | `DEMEAU_ANALYTICS_PROD` |
| Compute pool | `SYSTEM_COMPUTE_POOL_CPU` |
| Target | `DEMEAU_DD_PROD.DISTRIBUTE` |

⚠️ Do **not** select `SYSADMIN` as the execution role. It bypasses every RAP and
every masking policy, so the app would serve unmasked PII to every viewer, and
the deployment would prove nothing about the enforcement posture.

---

## F. Verification

### F.1 Before declaring an app deployed

```sql
SHOW STREAMLITS IN SCHEMA DEMEAU_DD_PROD.DISTRIBUTE;
```

Check `owner` is `DEMEAU_STREAMLIT_OWNER_PROD` and `owner_role_type` is `ROLE`.
An `owner_role_type` of `USER` means it is running as a person, which is the
posture the owner roles exist to avoid.

### F.2 Confirm the data path as the owner role

Do not infer it from grants — query as the role:

```bash
SNOWFLAKE_DEV_ROLE=DEMEAU_STREAMLIT_OWNER_PROD \
QUERY_WAREHOUSE=DEMEAU_ANALYTICS_PROD \
  python scripts/query_snowflake.py \
  "SELECT COUNT(*) FROM DEMEAU_DD_PROD.GOVERNANCE.DEMO_PERSONA_RESULTS"
```

`QUERY_TARGET` and `QUERY_WAREHOUSE` were added to `query_snowflake.py` on
2026-08-29; before that it was pinned to `demeau_dev`.

### F.3 After any masking, RAP or grid change

```bash
DRY_RUN=1 bash scripts/run_persona_capture.sh
tail -60 ~/Library/Logs/ditteau/persona-capture.log
```

Expected against PROD: Registrar `89,123 / 485,290 / 0` aid, Advisor `125`, FA
and IR full with `204` aid. Tiers must render as `[FULL]`/`[SCOPED]`/`[NONE]`.

⚠️ A wall of `[MISSING]` means the grid snapshot failed and the capture is
worthless. Before 2026-08-29 that condition still exited 0 and logged OK; it now
exits non-zero. `mart_student_health_holds` legitimately shows `[MISSING]` —
N-17 encodes that domain's denial as absence from the grid.

---

## G. Traps

**`_resolve_database()` follows the deployment, not the code.** Every DEMEAU
dashboard resolves its database from `session.get_current_database()`, falling
back to `DEMEAU_DD_PROD`. Deployed into `USER$LVANPELT.PUBLIC` — where both
current Workspaces apps live — that returns `USER$LVANPELT` and the fallback
never fires, so every query targets a schema that does not exist. Deploying into
`DEMEAU_DD_PROD.DISTRIBUTE` resolves correctly. Confirm on first launch rather
than assuming.

**`SYSTEM_COMPUTE_POOL_CPU` is shared account-wide.** Acceptable for demos, not
for a paying tenant: consumption is not attributable per role and a noisy
neighbour affects demo latency. A dedicated pool is an infrastructure scope
change through WDT.

**A migration is fleet-wide; a model build is per-target.** Grants applied by a
migration reach all nine databases at once. A dashboard fix reaches only the
target you actually build and deploy.

**Future grants cover rebuilds, but only in `DISTRIBUTE`.**
`DEMEAU_STREAMLIT_OWNER_PROD` has future `SELECT` on `TABLE` and `VIEW` in
`DEMEAU_DD_PROD.DISTRIBUTE`, so a full dbt rebuild re-grants automatically —
verified after the 2026-08-29 rebuild. The project has no `grants:` or
`copy_grants` config, so anything **outside** that schema would be lost on
replace.

**The DEV apps prove nothing about PROD.** They are `SYSADMIN`-owned in a
database with RLS off.
