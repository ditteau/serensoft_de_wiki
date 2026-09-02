# Streamlit in Snowflake — Workspaces Runbook

Operational guide for deploying the DEMEAU dashboards into `DEMEAU_DD_PROD` as
Streamlit in Snowflake apps, backed by a Git repository and served through
Workspaces.

**Last updated:** 2 September 2026
**Maintained by:** LVP

> **State as of this revision: ten applications deployed in `DEMEAU_DD_PROD.DISTRIBUTE`,
> all suspended, account pool spend zero.** See [A.1](#a1-deployed-apps--measured-2026-09-02).
> Sections **B** and **A.2** describe the superseded pre-D-1 world and are retained for the
> reasoning only — both are marked where they stop being true.
>
> ⚠️ **Two things a redeploy destroys, silently and without an error:** every **grant** on
> the app ([H.5a](#h5a--a-redeploy-destroys-every-grant-on-the-app-silently)) and the
> **idle-shutdown setting** ([H.6b](#h6b--a-redeploy-discards-the-idle-shutdown-setting-and-describe-cannot-see-it)).
> Both are now deploy steps rather than one-time fixes.

---

## Contents

- [A. What exists today](#a-what-exists-today)
- [B. The tier problem, and why two of three apps are gated](#b-the-tier-problem-and-why-two-of-three-apps-are-gated) — ⚠️ superseded by D-1; kept for the reasoning
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

### A.1 Deployed apps — measured 2026-09-02

**Ten applications, all in `DEMEAU_DD_PROD.DISTRIBUTE`**, all on
`DEMEAU_POOL_APPS_PROD`, all `owner_role_type = ROLE`, all
`query_warehouse = DEMEAU_ANALYTICS_PROD`. The target state in A.2 is **met and
exceeded** — this is no longer a plan.

| App | Owner role | Kind |
|---|---|---|
| `DEMEAU_REGISTRAR` | `DEMEAU_STREAMLIT_OWNER_REGISTRAR_PROD_ROLE` | persona (D-1) |
| `DEMEAU_FA` | `DEMEAU_STREAMLIT_OWNER_FA_PROD_ROLE` | persona (D-1) |
| `DEMEAU_IR` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | persona (D-1) |
| `DEMEAU_FINANCE` | `DEMEAU_STREAMLIT_OWNER_FINANCE_PROD_ROLE` | persona (D-1) |
| `DEMEAU_ADMISSIONS` | `DEMEAU_STREAMLIT_OWNER_ADMISSIONS_PROD_ROLE` | persona (D-1) |
| `DEMEAU_ENROLLMENT` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | analytics |
| `DEMEAU_BOARD_DATA_BOOK` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | analytics |
| `DEMEAU_COURSE_DEMAND` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | analytics |
| `DEMEAU_KPI_LIBRARY` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | analytics |
| `GOVERNANCE_SIS` | `DEMEAU_STREAMLIT_OWNER_PROD` | governance viewer |

> ⚠️ **`DEMEAU_STREAMLIT_OWNER_PROD` — the role on the last row — is the original
> unentitled one and holds no tier.** It is not a sixth persona. `GOVERNANCE_SIS` reads
> exactly one non-RAP table, so it works anyway; the role still reads **0 rows** from all
> ten RAP-protected objects, which is the control working (§B). Do not entitle it.

> ⚠️ **The four analytics dashboards are owned by the IR persona role**, which is what
> makes them safe to share: every row, zero identity (D-4). That is a deliberate
> assignment, not a leftover — an analytics dashboard owned by the Registrar role would
> render unmasked names to every viewer.

**Still in DEV, unchanged:** the three `SYSADMIN`-owned warehouse-runtime apps
(`DEMEAU_DD_DEV` ×2, `MERRIMACK_DD_DEV` ×1). ⚠️ `SYSADMIN` is on the bypass list in every
RAP and every masking policy, so these have **never exercised the enforcement path** and
their working in DEV is not evidence about PROD.

**Personal-workspace apps: gone.** The `USER$LVANPELT.PUBLIC` apps and the nine previews
created on 08-30 were dropped 2026-09-01/02. Account pool spend is currently **zero**.

> Reconstruct this table at any time with `SHOW SERVICES IN ACCOUNT` (each service's owner
> role) and `SHOW STREAMLITS IN DATABASE DEMEAU_DD_PROD`. Do not trust this list after a
> deploy session — see H.5's redeploy trap.

### A.2 Target state — met 2026-08-30, kept for the reasoning

> ⚠️ **A.2 describes the pre-D-1 plan. It is superseded and is kept only for why the
> shape changed.** The single-owner-role, three-app target below was replaced on
> 2026-08-29 when WDT ratified **D-1**: one application *per persona*, each owned by its
> own role holding that persona's tier. Section B's "two of three apps are gated" is
> likewise the old framing — the gate was the owner role holding no tier, which is exactly
> what D-1 changes. **Current state is A.1; the sequence that produced it is section H.**

Three apps in `DEMEAU_DD_PROD.DISTRIBUTE`, owned by
`DEMEAU_STREAMLIT_OWNER_PROD`, deployed from a Workspace backed by
`DITTEAU_PLATFORM.ADMIN.DITTEAU_DATA_TRANSFORM`:

| App | Source file | Status |
|---|---|---|
| Governance (recorded capture) | `streamlit/dashboards/governance_sis/demeau_governance_sis.py` | ✅ **deployed 2026-08-29** as `DEMEAU_DD_PROD.DISTRIBUTE.GOVERNANCE_SIS` |
| Enrollment v2 | `streamlit/dashboards/enrollment/demeau_enrollment_dashboard_v2.py` | ✅ deployed 2026-08-30 as `DEMEAU_ENROLLMENT`, owned by the **IR** role |
| Role-based demo | ~~`streamlit/demeau_role_dashboard.py`~~ | ⚠️ **superseded** — its content was reworked into the five per-persona apps (2026-08-30). Do not deploy it; it renders `numpy`-generated values beside real query results (O-43) |

`GOVERNANCE_SIS` is owned by `DEMEAU_STREAMLIT_OWNER_PROD` with
`owner_role_type = ROLE`, runs `SpcsOnly` / `execute_as: OWNER`, and carries
`SNOWFLAKE.SNOWPARK.PYPI_SHARED_REPOSITORY`.

> ⚠️ **Its definition is no longer at `streamlit/snowflake.yml`, and there must not be a
> file at that path.** A single shared yml is overwritten by every *Convert to Streamlit
> app*, silently repointing whichever app you deploy next — three deploys were lost to
> this on 2026-08-30. Every deployable app now carries its own committed yml in its own
> folder: `streamlit/dashboards/<app>/snowflake.yml` and
> `streamlit/persona_apps/<persona>/snowflake.yml`. See H.1a and `streamlit/README.md`.

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
all on the runtime we are deploying to.

> ✅ **Measured 2026-08-29, and the remedy is dead.** `CURRENT_USER()` returns
> `STPLATSTREAMLIT<n>` — the app's own service identity, not the viewer and not an account
> user (§J). **`user_domain_access` cannot entitle a SiS viewer at all**, so a row added
> there for a demo viewer does nothing whatsoever. The answer is D-1: one app per persona,
> owned by a role that holds that persona's tier. See section H.
>
> This paragraph is kept because the reasoning still matters — it is why the topology
> looks the way it does.

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

> ⚠️ **The central question is already answered, and the probe is now confirmatory
> rather than gating** (measured 2026-08-29, O-12 — see §J). `CURRENT_USER()` under the
> container runtime returns **`STPLATSTREAMLIT<n>`**, the app's own SPCS service identity
> — per-app, identical for every viewer, and **not an account user at all**. The
> `user_domain_access` leg of every RAP is therefore **inert** on Workspaces, and D-1's
> per-persona topology is the answer. Corroborated 2026-08-30 across three applications in
> one schema returning three different identities to one viewer.
>
> **What the probe would still close is the per-*viewer* half**, which currently rests on
> one viewer: nobody has watched three different people open the same app. That is a
> formality — the identity is demonstrably not the viewer's, is bound to the service
> object, and is not a user — but it is the only remaining way to prove invariance
> directly. Run it if you want the record; do not block anything on it.

### D.1 What it was written to answer

Under the **container runtime**, does `CURRENT_USER()` return the viewer or the
app owner? If the viewer, `user_domain_access` can carry per-viewer demo
entitlement and ADR-003 Phase 6 holds on Workspaces. If the owner, it cannot,
and the analytics dashboards need a different design.

✅ **Answered: neither.** It returns an application-scoped service identity, which is a
third outcome D.4 below does not contemplate. Read D.4 with that in mind — the "identical
across all three" branch is the one that fired, and it fired for a stronger reason than
that branch assumed.

### D.2 Prerequisites

- `DEMEAU_STREAMLIT_OWNER_DEV` holds `READ SESSION`, compute pool `USAGE`,
  `CREATE STREAMLIT`/`CREATE STAGE` on `DEMEAU_DD_DEV.GOVERNANCE`, and `SELECT`
  on both tier tables.
- The role is granted to `LVANPELT`, `WTRILLICH`, `KMARIE` — so all three see it
  in the **App executes as** picker.

> ⚠️ **`DEMEAU_TRANSFORM_DEV` is quota-suspended again and stays that way until
> 2026-09-10.** `DEMEAU_MONITOR_TRANSFORM_DEV` is at **100.52 / 100** credits with
> `suspend_immediately_at 100%`, and its `MONTHLY` cycle resets on the **10th** — the day
> of its own `start_time`, not the 1st (O-45). Measured 2026-08-31. This blocked the probe
> once before. Either wait, raise the quota, or point the probe at another warehouse.

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
| Named exception (N-13) | ✅ 28 rows in `seed_streamlit_owner_exceptions` — **25 `ROW_ACCESS` + 3 `PII_UNMASK`**; suite green |
| Compute pool | ✅ `DEMEAU_POOL_APPS_PROD`, **`MAX_NODES` 12** (raised 4 → 10 → 12; ADR-010 CP-5) |
| App files | ✅ Five, pushed to GitHub and fetched into the git repository |
| Roles granted to you | ✅ All five granted to `LVANPELT` |
| **The five apps themselves** | ✅ **Deployed 2026-08-30** — see A.1 |

> ⚠️ **All five persona apps are deployed, so H.1–H.4 below is now a *redeploy* procedure,
> not a first-run one.** That changes two things and both have bitten: a redeploy
> **destroys every grant on the app** (H.5), and it **discards the idle-shutdown setting**
> (H.6). Neither raises an error.

**Nothing below changes entitlement.** Every step is deployment. The act that creates
exposure is granting a *viewer* usage on an app, which is section H.5 and is deliberately
separate.

### H.1 Pull, as SYSADMIN

Already fetched on 2026-08-30 (`FAST_FORWARD`, not "up to date" — E.3). In the workspace,
role selector → `SYSADMIN` → **Pull**.

You should see five new folders under `streamlit/persona_apps/`:

```
registrar/  fa/  ir_analyst/  admissions/  finance/
```

Each holds `demeau_persona_app.py`, its own `snowflake.yml`, `pyproject.toml` and
`.streamlit/config.toml`.

⚠️ Pulling under an app-owner role fails with *"Secret 'secret from configuration' does
not exist or not authorized"*, which does not obviously mean "wrong role" (E.1).

### H.1a ⚠️ Deploy reads `snowflake.yml`, NOT the file open in the editor

**The single most misleading thing in this flow. It cost three deploys on 2026-08-30.**

The Deploy dialog offers App title, App ID, location, compute pool, query warehouse and
artifact repositories — and **no way to choose the main file**. That comes from
`snowflake.yml`. So opening `demeau_persona_app_admissions.py` and clicking Deploy shipped
`demeau_governance_sis.py`, because that is what the shared `streamlit/snowflake.yml` still
named. Twice, with a correct-looking dialog each time: right owner role, right database,
right pool, right artifact repository.

⚠️ **The failure was disguised by a correct-looking error.** The deployed app complained it
could not read `GOVERNANCE.DEMO_PERSONA_RESULTS` — which is *true and proper*, because the
Admissions owner role holds no `USAGE` on the `GOVERNANCE` schema. A wrong app producing a
right refusal reads as a permissions problem in the right app.

⚠️ **`Convert to Streamlit app` does not fix this and makes it worse at scale.** Convert
writes `snowflake.yml` *beside* the file, and there is only one `streamlit/` directory — so
each Convert overwrites the previous app's definition, including the committed
`GOVERNANCE_SIS` one. That is what a modified `snowflake.yml` in the file tree means.

**Resolved by structure, not by discipline.** Each persona app is now a self-contained
folder with its own committed `snowflake.yml`:

```
streamlit/persona_apps/<persona>/
    demeau_persona_app.py      main_file — identical in all five folders
    snowflake.yml              identifier + main_file for THIS persona
    pyproject.toml             copy — a subfolder is a different directory (E.4)
    .streamlit/config.toml     copy — the Ditteau theme
```

So there is **no Convert step** in H.2 below, nothing to hand-edit, and the deployable
definition lives in git rather than being generated inside a pull-only workspace and
mirrored back by hand (E.2). Regenerate with `scripts/generate_persona_apps.py`; verify
with `--check`, which compares byte-for-byte against `streamlit/demeau_persona_app.py`.

### H.1b Previews cannot be prevented, only cleaned up

Opening a file or its Settings starts a preview service, and there is no setting to stop
that. Previews **cannot run these apps at all** (E.7) so they produce nothing useful, and
each leaves a durable object behind: the service stops, the Streamlit object does not, and
the object regenerates under the same deterministic name next time.

Six were created and dropped over one afternoon. Treat it as cleanup-after, not
avoid-in-advance:

```sql
-- Everything running right now. Previews live in USER$<you>.PUBLIC.
SHOW SERVICES IN ACCOUNT;
SHOW STREAMLITS IN SCHEMA "USER$LVANPELT".PUBLIC;
DROP STREAMLIT "USER$LVANPELT".PUBLIC.<name>;
```

⚠️ **`DROP STREAMLIT` fails as `ACCOUNTADMIN`** with *"must have OWNERSHIP granted on
STREAMLIT …"*. It needs the app's owner role.

⚠️ **For a DEPLOYED app, suspend rather than drop.** Corrected 2026-08-30: ADR-010 said
no SQL lever exists, which was wrong — `ALTER SERVICE … SUSPEND` works once `OPERATE` is
granted, and `add_service_operate_grants_2026-08-30.sql` grants it to `DITTEAU_ADMIN` on
all and future services in `DEMEAU_DD_PROD.DISTRIBUTE`.

```sql
SHOW SERVICES IN COMPUTE POOL DEMEAU_POOL_APPS_PROD;   -- name + "Managed by" app
ALTER SERVICE DEMEAU_DD_PROD.DISTRIBUTE.<service> SUSPEND;
```

The UI equivalent is **Manage » Compute » Compute pools » `<pool>` » Services**, which
has a per-service **Suspend** / **Drop** menu. **Suspend** keeps the application and
releases its node; **Drop** destroys the service. Suspend a deployed app; drop a stray
preview.

⚠️ **Watch `active_nodes`, not the service state.** A `SUSPENDED` service whose node is
still active is still billing until the pool scales down.

### H.2 The five passes

One pass per row. **The role must change between passes** — this is the step that is easy
to forget and expensive to get wrong.

| # | File | Switch role to | App ID | App title |
|---|---|---|---|---|
| 1 | `persona_apps/registrar/` | `DEMEAU_STREAMLIT_OWNER_REGISTRAR_PROD_ROLE` | `DEMEAU_REGISTRAR` | Registrar |
| 2 | `persona_apps/fa/` | `DEMEAU_STREAMLIT_OWNER_FA_PROD_ROLE` | `DEMEAU_FA` | Financial Aid |
| 3 | `persona_apps/ir_analyst/` | `DEMEAU_STREAMLIT_OWNER_IR_ANALYST_PROD_ROLE` | `DEMEAU_IR` | Institutional Research |
| 4 | `persona_apps/admissions/` | `DEMEAU_STREAMLIT_OWNER_ADMISSIONS_PROD_ROLE` | `DEMEAU_ADMISSIONS` | Admissions |
| 5 | `persona_apps/finance/` | `DEMEAU_STREAMLIT_OWNER_FINANCE_PROD_ROLE` | `DEMEAU_FINANCE` | Finance |

Each file names its own intended role and App ID in a header banner, so you can check the
pair without coming back here.

For each pass:

1. **Switch role** to the owner role for that row.
2. Open that folder's `demeau_persona_app.py`. **Do not Convert** — the folder already
   carries its own `snowflake.yml` (H.1a). If Convert offers itself, decline it; it would
   overwrite the definition with a fresh one.
3. **Deploy**, with:

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

4. **Before clicking Deploy, read the dialog header.** It must say
   *App will execute with rights of `DEMEAU_STREAMLIT_OWNER_<PERSONA>_PROD_ROLE`*.
5. Nothing to mirror back — the `snowflake.yml` that was deployed is already in git.
6. Drop any preview objects the pass created (H.1b).

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

- Granting someone usage on `DEMEAU_REGISTRAR` shows them **unmasked student names**.
- Granting usage on `DEMEAU_FA` shows them **unmasked aid amounts**.
- `DEMEAU_IR` and `DEMEAU_FINANCE` carry every row and **no identity** — that is
  D-4 and D-16, and it is why those two are the safest to share widely.

```sql
USE ROLE SECURITYADMIN;
GRANT USAGE ON DATABASE  DEMEAU_DD_PROD            TO ROLE <viewer_role>;
GRANT USAGE ON SCHEMA    DEMEAU_DD_PROD.DISTRIBUTE TO ROLE <viewer_role>;
GRANT USAGE ON WAREHOUSE DEMEAU_ANALYTICS_PROD     TO ROLE <viewer_role>;
GRANT USAGE ON STREAMLIT DEMEAU_DD_PROD.DISTRIBUTE.DEMEAU_IR TO ROLE <viewer_role>;
```

⚠️ **The app object names carry no `_APP` suffix** — they are `DEMEAU_IR`, `DEMEAU_FA`,
`DEMEAU_REGISTRAR`, `DEMEAU_FINANCE`, `DEMEAU_ADMISSIONS`, `DEMEAU_ENROLLMENT`,
`DEMEAU_BOARD_DATA_BOOK`, `DEMEAU_COURSE_DEMAND`, `DEMEAU_KPI_LIBRARY`, `GOVERNANCE_SIS`
(A.1). An earlier version of this snippet said `DEMEAU_IR_APP`, which fails with
`002003 does not exist or not authorized` — the account's standing failure mode, and here
it really *is* a missing object rather than a missing privilege.

⚠️ **A viewer also needs `USAGE` on the database, the schema and the warehouse** to open
an app at all — the streamlit grant alone is not sufficient, and the failure looks like a
broken app rather than a missing privilege. That is why all four lines are above.

⚠️ **Person users hold `DEFAULT_ROLE = PUBLIC` and no secondary roles** (2026-08-24), so
a viewer must be granted a role that carries these usages, and must select it. `PUBLIC`
holds no tier in either grid, which is the fail-closed default and is asserted by N-20a/b.

#### H.5a ⚠️ A redeploy destroys every grant on the app, silently

**Measured 2026-09-02.** `DEMEAU_ENROLLMENT` was redeployed at 17:08:41 and a viewer's
usage grant vanished with the old object. **A deploy `CREATE OR REPLACE`s the Streamlit,
and grants do not survive the replace.** There is no error, no warning, and nothing in the
deploy output mentions grants — the first symptom is a person opening a link that worked
yesterday and getting *"does not exist or not authorized"*.

> ⚠️ **`SHOW GRANTS TO USER <name>` cannot detect this.** It lists what exists, not what is
> missing, so a partially-destroyed grant set looks exactly like a correct smaller one. The
> only reliable check is to assert the expected grant **per app** and compare against the
> list you intended.

**Re-grant after every deploy.** Treat it as part of the deploy, not as remediation:

```sql
-- After ANY redeploy, re-run the grants for every principal who had access.
SHOW GRANTS ON STREAMLIT DEMEAU_DD_PROD.DISTRIBUTE.<app>;   -- per app, not per user
```

#### H.5b Direct-to-`USER` grants sit outside every role-based control

The 2026-09-01 demo access for `RTHARP` was granted **to the user**, not to a role
(`grant_rtharp_demo_viewer_2026-09-01.sql`, all ten apps). That was the right call under
time pressure — it applies whichever role the session lands in, including `PUBLIC`, which
removes the "switch your role" failure mode entirely.

⚠️ **But N-13, the persona grid, `seed_streamlit_owner_exceptions` and
`assert_no_app_owner_in_role_domain_access` all reason about ROLES.** A direct-to-user
grant is invisible to every one of them. A `DEMEAU_DEMO_VIEWER_PROD_ROLE` is the
structural fix and is **recorded as deferred, not done**.

⚠️ **A gate written as a comment is not a gate.** That migration's header read
*"uncomment only if approved"* for the FA and Registrar apps while the `GRANT` statements
shipped **uncommented** under a heading reading *"Included for demo exploration"*. Both
executed. They turned out to be authorised — `RTHARP` has held `SYSADMIN` and
`DITTEAU_ADMIN` since 2026-02-26, so nothing was disclosed beyond his existing authority —
but the file gated them and then didn't. If a grant genuinely needs approval, **leave the
statement commented out.**

### H.6 Cost, before you leave ten apps running

> **Current state, 2026-09-02: all ten apps suspended, pool `auto_suspend` back to 300s
> after a temporary 3600s raise for a live demo, idle shutdown set to the 86,400s floor on
> all ten, `MAX_NODES` 12, both personal-workspace strays dropped. Account pool spend is
> zero.** Full measurement and reasoning:
> `ditteau_data_transform/docs/governance/compute_spend_findings_2026-08-31.md`.

Each running app pins a service, a pool cannot auto-suspend while any service runs, and
**each running service takes its own node** — measured 2026-08-30: `num_services 3` /
`active_nodes 3` on `DEMEAU_POOL_APPS_PROD`.

⚠️ **An earlier version of this section said services PACK onto a node. That was wrong**
(ADR-010 CP-5, corrected). One node is ~0.11 credits/hour ≈ **80 credits/month**, so the
cost is **per app left Active**: ten apps is roughly **800 credits/month** against a
warehouse consuming ~11. `MAX_NODES` is **12** — it bounds how many apps can run at once,
not spend. Below the app count you get *"The selected compute pool is unable to start your
app… the pool is full"*.

⚠️ **`num_services` is NOT the number of running services, and thresholding on it is
wrong.** Measured 2026-08-31: ten services against **two** active nodes, because eight
were suspended. The column counts service *objects* regardless of state, so
`num_services 10` on a pool sized 10 reads alarming and is not. **Read `ACTIVE NODES`.**

#### H.6a The standing posture is "none Active at rest" — and warming is a procedure

At 0.11 credits/node-hour: **all ten Active ≈ 800 credits/month** (80% of the 1,000
account quota, for zero use) · **`GOVERNANCE_SIS` only ≈ 80** · **none at rest ≈ 7**,
assuming ten two-hour demo sessions a month across three apps.

✅ **Every service carries `auto_resume = true`, so a suspended app is slow, not broken.**
Observed: `DEMEAU_ENROLLMENT` suspended 16:33:47, resumed **18:00:04** on someone simply
opening it, with no intervention. ⚠️ **ADR-010's alternatives table understates this
option** because it was written before suspension was known to work — that entry describes
suspending the *pool*, which does break an unannounced open. Suspending a *service* costs
latency, not availability.

✅ **Cold start measured 2026-09-02: ~70 seconds, and variable.** ⚠️ **The variance matters
more than the average**, which is why the mitigation is a procedure rather than a number:
**warm the apps you intend to show before the audience is in the room, and never open a
cold app live.** That holds whether a given start takes 40 seconds or three minutes; a
single average invites someone to budget 70 seconds of stage time and be wrong.

#### H.6b ⚠️ A redeploy discards the idle-shutdown setting, and `DESCRIBE` cannot see it

All ten services read `auto_suspend_secs = 259,200` (**72 hours**) when measured
2026-08-31, despite 86,400 having been applied to `GOVERNANCE_SIS` on 08-29.

**Resolved 2026-09-02:** `SHOW STREAMLITS` exposes `idle_auto_shutdown_time_seconds` —
**which `DESCRIBE STREAMLIT` does not** — and it read **unset on all ten**, including the
one set on 08-29. So a redeploy **replaces the Streamlit object and discards the
setting**, and 259,200 is the fallback default.

> ⚠️ **This recurs on every deploy. Re-applying it is a runbook step, not a one-time fix**
> — the same shape as the grant destruction in H.5a, from the same cause. Verify with
> `SHOW STREAMLITS`, never `DESCRIBE STREAMLIT`.

The cost of missing it: an abandoned app bills **72h × 0.11 = 7.92 credits** before
shutting itself down, against **2.64** at the floor. Across ten apps, ~79 credits of pure
idle tail versus ~26.

#### H.6c ⚠️ "Last query" is a trap; filter to `SELECT`

If you build any idle sweep, do not threshold on the newest query per service. Measured
over six hours on `DEMEAU_ENROLLMENT`: **3,085 `DESCRIBE` + 3,085 `LIST_FILES` against 25
`SELECT`s** — a metronomic **1,440 queries/hour**, one pair every 2.5 seconds, identical
across both running services to the second. That is the container runtime watching its own
stage, not a person.

**`MAX(start_time)` therefore reads "seconds ago" for anything running, forever, and a
sweep built on it would never fire once** — while looking like a working control. Filter
to `query_type = 'SELECT'`; the discrimination is roughly 200:1.

⚠️ **Even filtered, `idle_minutes` is a lower bound on inattention, never proof of
absence.** Streamlit queries on interaction and cache miss, not on reading — a viewer
studying a rendered page for an hour issues zero SELECTs. Any sweep needs a threshold
longer than a plausible reading session, which is why the CP-1 monitor **reports and does
not act**.

⚠️ **Per-application cost attribution is impossible, permanently.**
`SNOWPARK_CONTAINER_SERVICES_HISTORY` carries no service name, so pool credits can never be
split across the apps sharing a pool (O-47). Per-app cost is an inference from node-hours ×
running time, not a measurement.

⚠️ **Corrected 2026-08-30: there IS a scriptable way.** `ALTER SERVICE … SUSPEND`
works once `OPERATE` is granted — it failed as `ACCOUNTADMIN` only because nobody held
that privilege, not because the services are unstoppable. `ALTER STREAMLIT … SUSPEND`
still does not exist and `IDLE_AUTO_SHUTDOWN_TIME_SECONDS` still floors at 24 hours, so
the *idle timer* remains useless as a between-sessions control — but a scheduled sweep is
now buildable. See H.1b and ADR-010.

> The 24-hour floor is why H.6a's answer is "suspend deliberately" and H.6b's is "set the
> floor anyway." They are not in tension: the timer is a **backstop against total
> abandonment** (72h → 24h of idle tail), never a between-sessions control. ⚠️ **A
> scheduled sweep is buildable but is not built** — the CP-1 monitor is written,
> validated read-only, and **unexecuted**, awaiting the CP-4 decision. Report-only by
> design: suspending an app a client is mid-demo on is worse than the spend it saves.

⚠️ **Suspend as `DITTEAU_ADMIN` or `ACCOUNTADMIN`. As `SYSADMIN` it fails, and the
message names the wrong problem.** `OPERATE` was granted to `DITTEAU_ADMIN` only
(`add_service_operate_grants_2026-08-30.sql`); `ACCOUNTADMIN` inherits it via
`SECURITYADMIN`, to which `DITTEAU_ADMIN` is granted. `SYSADMIN` is in neither path and
returns:

```
002003 (02000): SQL compilation error:
Service 'DEMEAU_DD_PROD.DISTRIBUTE.STPLATSTREAMLIT392910720' does not exist or not authorized.
```

Measured 2026-08-30 on the same service, same statement, in the same minute:
`DITTEAU_ADMIN` ✅ · `ACCOUNTADMIN` ✅ · `SYSADMIN` ❌. **Nothing was wrong with the
service.** This is the account's standing failure mode — a missing privilege wearing a
message about existence, the same shape as `SYSADMIN` reading `DEMEAU_DD_PROD`'s
modelled layer as empty. Read *"does not exist or not authorized"* as **"try another
role"** before concluding anything about the object.

⚠️ **This bites hardest through the UI**, because the ⋮ **Suspend** on the compute-pool
page runs under the session's *current* role, which is not always the one you last
selected — and during a deploy session you are switching roles constantly (E.1). The
role badge in the corner is not proof of what a given page context is using. If a
suspend is refused, switch role explicitly and retry before investigating the service.

```bash
SNOWFLAKE_DEV_ROLE=DITTEAU_ADMIN QUERY_WAREHOUSE=DEMEAU_ANALYTICS_PROD \
  PATH="/Users/laurievanpelt/testenv/bin:$PATH" python scripts/query_snowflake.py \
  "ALTER SERVICE DEMEAU_DD_PROD.DISTRIBUTE.<service> SUSPEND"
```

✅ **The `ON FUTURE SERVICES` grant is confirmed working against a real redeploy**, not
just argued from the docs. `DEMEAU_ENROLLMENT`'s service was created at `00:33:44.620`
and `SHOW GRANTS ON SERVICE` records `OPERATE … granted_by SYSTEM$MANAGED` at
`00:33:44.715` — attached automatically, 0.7s later, to a service that did not exist
when the grant was written. Since every deploy mints a new `STPLATSTREAMLIT<n>` name,
this is the half that keeps any sweep alive; re-check it after a redeploy, because when
it lapses the sweep keeps running and silently stops suspending anything.

**The fastest check is the UI, and it is live.** Snowsight → **Manage » Compute »
Compute pools**. It lists every pool with `NUMBER OF JOBS`, `ACTIVE NODES`, `IDLE
NODES`, `AUTO SUSPEND`, `AUTO RESUME` and `RESUMED ON`.

⚠️ **`ACTIVE NODES` is the number to read**, and it is the one that disproved the
packing claim: three running applications showed **3 active nodes / 0 idle**, matching
`SHOW COMPUTE POOLS` exactly (`active_nodes 3, idle_nodes 0, target_nodes 3`). If that
number is not roughly equal to the number of apps you expect to be running, something is
running that you did not intend.

⚠️ **`IDLE NODES` above zero means paying for nothing** — nodes provisioned with no
service on them, waiting out `AUTO SUSPEND`.

⚠️ **Suspending every service does not stop the meter.** Measured 2026-08-30 with all
four services suspended: the pool reported `state IDLE`, `active_nodes 0` — and
`idle_nodes 2, target_nodes 2`. Those two nodes bill until `AUTO_SUSPEND_SECS` (300)
elapses. A screen full of `Suspended` badges is not evidence that a demo cost nothing;
re-read the pool a few minutes later.

⚠️ **The UI shows state, not spend.** It cannot tell you what a pool has cost; for that
you need the metering query below, which lags up to ~2h. Use the UI to answer *"what is
running right now"* and the query to answer *"what has this cost"*. Neither substitutes
for the other, and the six-week burn ADR-010 records was invisible to both because
nobody looked at either.

The equivalent in SQL, which also works from a script:

```sql
SHOW COMPUTE POOLS;   -- active_nodes, idle_nodes, target_nodes, num_services
SHOW SERVICES IN ACCOUNT;   -- which app each running service belongs to
```

```sql
-- What the pool has actually cost. Watch implied node-hours, not credits.
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
