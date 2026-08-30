# Streamlit in Snowflake — Workspaces Runbook

Operational guide for deploying the DEMEAU dashboards into `DEMEAU_DD_PROD` as
Streamlit in Snowflake apps, backed by a Git repository and served through
Workspaces.

**Last updated:** 30 August 2026
**Maintained by:** LVP

---

## Contents

- [A. What exists today](#a-what-exists-today)
- [B. The tier problem, and why two of three apps are gated](#b-the-tier-problem-and-why-two-of-three-apps-are-gated)
- [C. Prerequisites — all applied 2026-08-29](#c-prerequisites--all-applied-2026-08-29)
- [D. Running the identity probe](#d-running-the-identity-probe)
- [E. Deploying an app from a Git-backed Workspace](#e-deploying-an-app-from-a-git-backed-workspace)
- [F. Verification](#f-verification)
- [H. Deploying the five per-persona applications (D-1)](#h-deploying-the-five-per-persona-applications-d-1)
- [I. Dependencies: declaring one means declaring all](#i-dependencies-declaring-one-means-declaring-all)
- [J. What `CURRENT_USER()` actually returns under the container runtime](#j-what-current_user-actually-returns-under-the-container-runtime)
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

> ⚠️ **A.1 and A.2 describe the pre-D-1 world and are kept for the reasoning, not as
> current state.** "Nothing is deployed in any PROD database" ceased to be true on
> 2026-08-29 (`GOVERNANCE_SIS`), and the single-owner-role target below was superseded on
> 2026-08-29 when WDT ratified **D-1**: the topology is now one application *per persona*,
> each owned by its own role holding that persona's tier. Section B's "two of three apps
> are gated" is likewise the old framing — the gate was the owner role holding no tier,
> which is exactly what D-1 changes. **See section H.**

Three apps in `DEMEAU_DD_PROD.DISTRIBUTE`, owned by
`DEMEAU_STREAMLIT_OWNER_PROD`, deployed from a Workspace backed by
`DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM`:

| App | Source file | Status |
|---|---|---|
| Governance (recorded capture) | `streamlit/demeau_governance_sis.py` | ✅ **deployed 2026-08-29** as `DEMEAU_DD_PROD.DISTRIBUTE.GOVERNANCE_SIS` |
| Enrollment v2 | `streamlit/demeau_enrollment_dashboard_v2.py` | gated — see B |
| Role-based demo | `streamlit/demeau_role_dashboard.py` | gated — see B |

`GOVERNANCE_SIS` is owned by `DEMEAU_STREAMLIT_OWNER_PROD` with
`owner_role_type = ROLE`, runs `SpcsOnly` / `execute_as: OWNER`, and carries
`SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY`. Its definition is committed at
`streamlit/snowflake.yml`.

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

Walked end to end on 2026-08-29 deploying `GOVERNANCE_SIS`. This section is what
actually happened, not what the probe README predicted — several steps differ.

### E.1 Two roles, two jobs

The single most confusing thing about this flow. **One role cannot do both
halves**, and switching is easy to forget:

| Job | Role | Why |
|---|---|---|
| Pull from git | `SYSADMIN` | only role with `READ` on `DITTEAU_GITHUB_SECRET` |
| Convert / preview / deploy | `DEMEAU_STREAMLIT_OWNER_PROD` | the app must *execute* as this |

Pulling under the app-owner role fails with
`SQL compilation error: Secret 'secret from configuration' does not exist or not
authorized` — which does not obviously mean "wrong role".

⚠️ Do **not** fix this by granting the app-owner role `READ` on the secret. It
is a GitHub PAT scoped to the whole `ditteau` org; the app needs no git access
at runtime, only the developer does at pull time. The owner role exists to hold
as little as possible.

⚠️ Forgetting to switch **back** is the expensive direction. Deploying as
`SYSADMIN` produces an app that bypasses every RAP and every masking policy —
it would look better than working, showing unmasked PII to every viewer, while
proving nothing about enforcement. Confirm the dialog header reads
`App will execute with rights of DEMEAU_STREAMLIT_OWNER_PROD` before deploying.

### E.2 The workspace is pull-only, by design

`USER$LVANPELT.PUBLIC.ditteau_data_transform` is already git-backed. **Push is
deliberately not permitted** — the credential has no write scope, and `Push`
fails with `Operation push is not permitted by server for origin`.

No development happens inside the Snowflake workspace; git is the only write
path. The cost is that anything Workspaces generates — `snowflake.yml` in
particular — has to be copied back into the repo **by hand** or it exists
nowhere but that workspace.

### E.3 Push, then fetch, then pull

Three separate hops, and each can silently be a no-op:

```bash
git push origin main
```
```sql
ALTER GIT REPOSITORY DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM FETCH;
```
then **Pull** in the workspace, as `SYSADMIN`.

⚠️ `FETCH` reports `is up to date. No change was fetched.` both when origin
genuinely has not moved **and** when you forgot to push. A real fetch prints
`Branch | main | FAST_FORWARD`.

### E.4 Convert, don't create

Right-click the app file » **Convert to Streamlit app**. Do not use the
**Streamlit App** button on the Welcome screen — that generates a fresh
`streamlit_app.py` scaffold you would then have to delete.

Convert leaves the file where it is and writes `snowflake.yml` beside it. It
does **not** move the app into its own folder, which is why one shared
`streamlit/pyproject.toml` serves every dashboard in that directory.

### E.5 Attach the artifact repository before running

**App settings » Execution » Artifact repositories** →
`SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY`.

Not in the Convert dialog — only on the app afterwards. See section I for why
this is required and what it looks like when missing.

### E.6 Deploy

| Field | Value |
|---|---|
| App title | human-readable; shows top-left and in the dashboard |
| App ID | becomes the object name and the URL — `GOVERNANCE_SIS` |
| Owner-matches-preview checkbox | **leave ticked** |
| App location | `DEMEAU_DD_PROD` / `DISTRIBUTE` |
| Compute pool | `SYSTEM_COMPUTE_POOL_CPU` |
| Query warehouse | `DEMEAU_ANALYTICS_PROD` |
| Artifact repositories | `SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY` |
| Network tab | empty — the account has no EAIs and must not gain one here |
| Sharing tab | owner only to start |

Then mirror the generated `snowflake.yml` into the repo by hand (E.2).

### E.7 Preview cannot run these apps at all

**Every dashboard here opens with `get_active_session()`, and preview has no
Snowpark session.** Preview runs your file in a container with an identity —
it reports *"Running as DEMEAU_STREAMLIT_OWNER_PROD"* — but that is not the
injected session a deployed SiS app receives.

`demeau_governance_sis.py` catches the failure and renders *"No active Snowflake
session … use `streamlit/demeau_governance_dashboard.py` instead"*, which reads
as "this app does not belong in SiS" and is the opposite of true.

⚠️ **Deploy is the only real test.** Do not spend rounds debugging a preview
failure that deployment resolves.

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

## H. Deploying the five per-persona applications (D-1)

Written 2026-08-30. Section E is the general flow for **one** app; this is the specific
five-pass sequence, and the differences are not cosmetic. Read E first — everything there
still applies, especially E.1 (two roles) and E.7 (preview cannot run these).

### H.0 What is already done, so you do not redo it

| Thing | State |
|---|---|
| Five owner roles | ✅ Created, each holding exactly one persona's tier |
| Grid rows | ✅ 25 in `DEMEAU_DD_PROD.governance.role_domain_access` |
| PII unmask | ✅ Registrar `NAME`+`DOB`, FA `FINANCIAL_AMOUNT`; others none, by design |
| Named exception (N-13) | ✅ 28 rows in `seed_streamlit_owner_exceptions`, suite green |
| Compute pool | ✅ `DEMEAU_POOL_APPS_PROD`, `MAX_NODES` 4 (ADR-010 CP-5) |
| App files | ✅ Five, pushed to GitHub and fetched into the git repository |
| Roles granted to you | ✅ All five granted to `LVANPELT` |

**Nothing below changes entitlement.** Every step is deployment. The act that creates
exposure is granting a *viewer* usage on an app, which is section H.5 and is deliberately
separate.

### H.1 Pull, as SYSADMIN

Already fetched on 2026-08-30 (`FAST_FORWARD`, not "up to date" — E.3). In the workspace,
role selector → `SYSADMIN` → **Pull**.

You should see five new files in `streamlit/`:

```
demeau_persona_app_registrar.py
demeau_persona_app_fa.py
demeau_persona_app_ir_analyst.py
demeau_persona_app_admissions.py
demeau_persona_app_finance.py
```

⚠️ Pulling under an app-owner role fails with *"Secret 'secret from configuration' does
not exist or not authorized"*, which does not obviously mean "wrong role" (E.1).

### H.2 The five passes

One pass per row. **The role must change between passes** — this is the step that is easy
to forget and expensive to get wrong.

| # | File | Switch role to | App ID | App title |
|---|---|---|---|---|
| 1 | `demeau_persona_app_registrar.py` | `DEMEAU_STREAMLIT_OWNER_REGISTRAR_PROD_ROLE` | `DEMEAU_REGISTRAR_APP` | Registrar |
| 2 | `demeau_persona_app_fa.py` | `DEMEAU_STREAMLIT_OWNER_FA_PROD_ROLE` | `DEMEAU_FA_APP` | Financial Aid |
| 3 | `demeau_persona_app_ir_analyst.py` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | `DEMEAU_IR_APP` | Institutional Research |
| 4 | `demeau_persona_app_admissions.py` | `DEMEAU_STREAMLIT_OWNER_ADMISSIONS_PROD_ROLE` | `DEMEAU_ADMISSIONS_APP` | Admissions |
| 5 | `demeau_persona_app_finance.py` | `DEMEAU_STREAMLIT_OWNER_FINANCE_PROD_ROLE` | `DEMEAU_FINANCE_APP` | Finance |

Each file names its own intended role and App ID in a header banner, so you can check the
pair without coming back here.

For each pass:

1. **Switch role** to the owner role for that row.
2. Right-click the file → **Convert to Streamlit app** (E.4 — not the Welcome-screen
   button).
3. **App settings » Execution » Artifact repositories** →
   `SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY` (E.5). Not offered in the Convert dialog.
4. **Deploy**, with:

   | Field | Value |
   |---|---|
   | App ID | from the table above |
   | App title | from the table above |
   | App location | `DEMEAU_DD_PROD` / `DISTRIBUTE` |
   | Compute pool | `DEMEAU_POOL_APPS_PROD` |
   | Query warehouse | `DEMEAU_ANALYTICS_PROD` |
   | Owner-matches-preview checkbox | leave ticked |
   | Network tab | empty — the account has no EAIs and must not gain one |
   | Sharing tab | owner only |

5. **Before clicking Deploy, read the dialog header.** It must say
   *App will execute with rights of `DEMEAU_STREAMLIT_OWNER_<PERSONA>_PROD_ROLE`*.
6. Mirror the generated `snowflake.yml` back into the repo by hand (E.2), under a name
   that identifies which app it belongs to.

> ⚠️ **The compute pool is `DEMEAU_POOL_APPS_PROD`, not `SYSTEM_COMPUTE_POOL_CPU`.** E.6's
> table still says the system pool because that is what `GOVERNANCE_SIS` was deployed on
> before ADR-010 CP-2 created the per-school pool. Using the system pool here loses the
> per-tenant billing attribution that pool exists for, and pool spend is invisible to
> resource monitors (CP-1, still open).

> ⚠️ **Deploying as `SYSADMIN` is the expensive mistake.** The app would bypass every RAP
> and every masking policy, show unmasked PII to every viewer, and *look better than
> working*. If a deployed app shows a real student name where you expected `***`, suspect
> the owner before suspecting the policy.

> ⚠️ **Preview will fail on all five and that proves nothing.** They open with
> `get_active_session()` and preview has no injected session (E.7). Deploy is the only
> real test.

### H.3 Verify each one — as the role, not by counting grants

```sql
SHOW STREAMLITS IN SCHEMA DEMEAU_DD_PROD.DISTRIBUTE;
```

Five rows plus `GOVERNANCE_SIS`. Each `owner` must be its own owner role and
`owner_role_type` must be `ROLE`; `USER` means it is running as a person (F.1).

Then confirm the five genuinely differ. This is the whole point of D-1, and it is the
check that would have caught a role mix-up:

```bash
for R in REGISTRAR FA IR_ANALYST ADMISSIONS FINANCE; do
  printf "%-11s " $R
  SNOWFLAKE_DEV_ROLE=DEMEAU_STREAMLIT_OWNER_${R}_PROD_ROLE \
  QUERY_WAREHOUSE=DEMEAU_ANALYTICS_PROD \
    python scripts/query_snowflake.py \
    "SELECT (SELECT COUNT(*) FROM DEMEAU_DD_PROD.DISTRIBUTE.DIM_STUDENT)      AS students,
            (SELECT COUNT(*) FROM DEMEAU_DD_PROD.DISTRIBUTE.FACT_AID_AWARD)   AS aid,
            (SELECT COUNT(*) FROM DEMEAU_DD_PROD.DISTRIBUTE.FACT_APPLICATION) AS apps"
done
```

Measured 2026-08-30 — anything else is a finding:

| Owner role | students | aid | applications |
|---|---|---|---|
| Registrar | 89,123 | **0** — aid `NONE` | 279,102 |
| FA | 89,123 | 204 | 279,102 |
| IR | 89,123 | 204 | 279,102 |
| Admissions | 89,123 | **0** — `AGGREGATED` is not row access | 279,102 |
| Finance | 89,123 | 204 | **0** — admissions `AGGREGATED` |

And identity, on one student so the comparison is real (`ORDER BY student_key`):

| App | `student_full_name` | `student_dob` |
|---|---|---|
| Registrar | `Barefoot, Alden N` | `1999-08-14` |
| everyone else | `***` | `1999-01-01` |

⚠️ **All five showing the same numbers means something is wrong**, most likely that more
than one app got the same owner. A uniform answer is the failure signature of this
topology, exactly as it is for the persona capture (C-01 check 10).

### H.4 Then, and only then, stop

Five deployed apps, owner-only sharing, nobody else granted usage. That is a complete and
safe end state. Every viewer entitlement from here is a separate decision.

### H.5 Granting viewers — the act that creates exposure

Under D-1 the usage grant **is** the control surface. There is no per-viewer
differentiation inside an application (§J), so:

- Granting someone usage on `DEMEAU_REGISTRAR_APP` shows them **unmasked student names**.
- Granting usage on `DEMEAU_FA_APP` shows them **unmasked aid amounts**.
- `DEMEAU_IR_APP` and `DEMEAU_FINANCE_APP` carry every row and **no identity** — that is
  D-4 and D-16, and it is why those two are the safest to share widely.

```sql
USE ROLE SECURITYADMIN;
GRANT USAGE ON STREAMLIT DEMEAU_DD_PROD.DISTRIBUTE.DEMEAU_IR_APP TO ROLE <viewer_role>;
```

⚠️ **A viewer also needs `USAGE` on the database, the schema and the warehouse** to open
an app at all — the grant above alone is not sufficient, and the failure looks like a
broken app rather than a missing privilege.

⚠️ **Person users hold `DEFAULT_ROLE = PUBLIC` and no secondary roles** (2026-08-24), so
a viewer must be granted a role that carries these usages, and must select it. `PUBLIC`
holds no tier in either grid, which is the fail-closed default and is asserted by N-20a/b.

### H.6 Cost, before you leave five apps running

Each running app pins a service, and a pool cannot auto-suspend while any service runs.
Services **pack** — five apps do not mean five nodes; a flat 2.637 credits/day is one
`CPU_X64_S` node, measured while two to three services ran (ADR-010 CP-5). But one pinned
node is roughly **80 credits/month**, an order of magnitude above the warehouse behind it.

There is **no scriptable way to stop a running Streamlit service**: `ALTER SERVICE …
SUSPEND` fails even as `ACCOUNTADMIN`, `ALTER STREAMLIT … SUSPEND` does not exist, and
`IDLE_AUTO_SHUTDOWN_TIME_SECONDS` floors at 24 hours. The only levers are the UI **Active**
toggle and `DROP`.

```sql
-- What the pool is actually doing. Watch implied node-hours, not credits.
SELECT compute_pool_name,
       DATE_TRUNC('day', start_time)      AS day,
       ROUND(SUM(credits_used), 3)        AS credits,
       ROUND(SUM(credits_used) / 0.11, 1) AS implied_node_hours
FROM SNOWFLAKE.ACCOUNT_USAGE.SNOWPARK_CONTAINER_SERVICES_HISTORY
WHERE start_time > DATEADD('day', -14, CURRENT_TIMESTAMP())
GROUP BY 1, 2 ORDER BY 2 DESC, 1;
```

A move from ~24 to ~48 node-hours/day means a second node became durably active.

---


## I. Dependencies: declaring one means declaring all

The container runtime resolves packages from `streamlit/pyproject.toml`. The
rule that is not written down anywhere:

> **Declaring any dependency replaces the default environment. It does not add
> to it.**

So a list naming only `plotly` yields a container with plotly and no
`streamlit`, no `snowpark`, no `pandas`. The working list is:

```toml
dependencies = [
    "streamlit>=1.48.0",
    "snowflake-snowpark-python",
    "pandas",
    "plotly",
]
```

Getting there took four deploys, each with a different error and none of them
naming the real cause:

| # | State | Error |
|---|---|---|
| 1 | no `pyproject.toml` | `plotly` missing at import |
| 2 | `plotly` only | `Failed to fetch https://pypi.org/simple/plotly/ … dns error` |
| 3 | + artifact repository | `Failed to get the version of the Streamlit library … ">=1.48.0"` |
| 4 | + `streamlit`, `pandas` | app renders *"No active Snowflake session"* |
| 5 | + `snowflake-snowpark-python` | works |

⚠️ **Error 2 is not an EAI problem**, though it asks you outright *"Have you
enabled External Access Integration (EAI)?"* and Snowsight offers **Fix with
CoCo**, which proposes exactly that. The account has zero EAIs and
`governance_identity_probe/README.md` §F forbids adding one — an EAI grants
general outbound egress to an app that reads governance data. Attach the
Snowflake-hosted mirror instead.

⚠️ **Error 4 is the dangerous one.** The app's own `except Exception` around
`get_active_session()` converts a missing package into a well-written sentence
recommending workstation tooling. A packaging fault presenting as an
architectural statement.

⚠️ `governance_identity_probe/README.md` §F says listing `pandas` "forced a
network fetch for a package already on disk." That is correct **for the probe**,
which declares nothing and keeps the default environment intact. It stops being
true the moment a file declares anything. Do not carry that advice across.

---

## J. What `CURRENT_USER()` actually returns under the container runtime

Measured 2026-08-29 from the deployed `GOVERNANCE_SIS`, whose "This app's own
session" panel runs the context functions live:

| | |
|---|---|
| `CURRENT_USER()` | **`STPLATSTREAMLIT392910632`** |
| `CURRENT_ROLE()` | `DEMEAU_STREAMLIT_OWNER_PROD` |
| `CURRENT_WAREHOUSE()` | `DEMEAU_ANALYTICS_PROD` |
| `CURRENT_DATABASE()` | `DEMEAU_DD_PROD` |

`CURRENT_USER()` is neither the viewer nor the owner. It is the app's own SPCS
service identity — `SHOW USERS LIKE 'STPLATSTREAMLIT%'` returns **zero rows**,
so it is not an account user at all, and the numbering matches the service
objects behind the other workspace apps (`…620`, `…624`). It is **per-app**.

**Consequence for ADR-003 Phase 6.** Every RAP resolves

```sql
COALESCE(user_domain_access WHERE snowflake_username = CURRENT_USER(),
         role_domain_access WHERE role_name        = CURRENT_ROLE())
```

Under this runtime the user leg can never match a person's username, so **it is
inert**. Per-viewer entitlement inside a single SiS app is not achievable on
Workspaces with the current design. Both legs resolve from the app's identity.

This **validates D-1** — the per-persona application topology is not a
convenience, it is the only workable shape. D-1 is the one decision still
pending ratification and this is evidence for it.

⚠️ One viewer cannot prove the value never varies. The formal three-person probe
(section D) still closes the finding properly. But the identity is demonstrably
not the viewer's, is not a user, and is bound to the service object.

Also settled: `CURRENT_DATABASE()` returns the deployment database, so
`_resolve_database()` behaves correctly — previously unverified.

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
