# Governance and Production Readiness — Review and Lessons

A retrospective covering ADR-003 (row access policies and column masking), the move to
service-account identity, and the first production build of the Ditteau platform.

**Period:** August 2026
**Author:** LVP
**Audience:** Ditteau DE team
**Companion document:** [Production Operating Rules](production-operating-rules.md) — the
rules this review produced

---

## Contents

- [1. What we set out to do](#1-what-we-set-out-to-do)
- [2. What we built](#2-what-we-built)
- [3. The defects, grouped by what caused them](#3-the-defects-grouped-by-what-caused-them)
- [4. The lessons](#4-the-lessons)
- [5. What we changed](#5-what-we-changed)
- [6. Where the platform stands](#6-where-the-platform-stands)
- [7. What is still open](#7-what-is-still-open)

---

## 1. What we set out to do

ADR-003 specified domain-scoped row access: a registrar sees every student, an advisor
sees only their advisees, an institutional researcher sees aggregates and no individual
rows. Four access tiers, four data domains, applied per institution across nine Snowflake
databases.

Phases 1–2 were recorded as complete before this work began. We picked up at Phase 3 and
carried through Phase 6. That was the first half.

The second half was not planned. Having built the access-control machinery, we tried to
run the platform the way it was designed to be run — as a dedicated service account
rather than as an engineer with administrator rights — and discovered that almost none of
it worked. That led to building PROD and TEST for the first time in any school.

**The headline result is not the feature.** It is that a system recorded as substantially
complete was, on inspection, largely unexecuted, and that the single change which
surfaced the most defects was not a code change at all. It was logging in as someone
else.

---

## 2. What we built

| | |
|---|---|
| Business roles | 12 (registrar / advisor / financial aid / IR analyst × 3 schools) |
| Tier grid rows | 360 (10 roles × 4 domains × 9 databases) |
| Domain policies | 3 — `student_academic`, `financial_aid`, `admissions` |
| Masking policies | 7 (DEMEAU DEV only — see §7) |
| Models inventoried | 44, classified for whether their grain reaches an individual |
| Automated governance tests | 3, each verified to fail when its invariant is broken |
| Service accounts commissioned | 9 (`SVC_{CODE}_DBT_{ENV}`, key-pair, `TYPE = SERVICE`) |
| Environments fully built | 3 (DEMEAU DEV, TEST, PROD — the first PROD in any school) |
| Migrations written and executed | 11 |

Tier resolves `COALESCE(user_grid, role_grid)`; the scoped predicate resolves against
`CURRENT_USER()`. Masking resolves against a **separate** grid keyed on PII class, not on
access tier.

Enforcement remains off in every environment. That is deliberate and gated on the
preconditions in the Operating Rules §5.3.

---

## 3. The defects, grouped by what caused them

Thirty-five in total — eighteen in the ADR-003 phase, seventeen in the production
readiness phase. They are more useful grouped by root cause than listed chronologically.

### 3.1 Code behind a feature flag — five defects

Everything gated on `enable_row_level_security` or `enable_masking_policies` had never
executed, because neither flag has ever been true in any environment.

| Defect | Consequence if undetected |
|---|---|
| `apply_rap()` resolved the policy path via `var('env')` — no such var exists | Enforcement fails on the first PROD model |
| `enable_masking_policies` is read by no code at all | Three model docs described masking as active; nothing masked anything |
| Policy signature was `VARCHAR`; every target column is `NUMBER` or `TEXT` | Policy creates, then fails at attachment (`003554`) |
| `fact_enrollment` carried only a surrogate key, no natural `student_id` | Attaches cleanly, matches nothing, returns zero rows for every scoped user |
| `GRANT APPLY … TO ROLE SYSADMIN` — SYSADMIN already owns the policies | No-op; the dbt role that runs the hook has no APPLY |

The pattern: **a feature flag guarantees its own code is untested.**

### 3.2 Privilege masked by ownership — six defects

This category did not exist until we tried to run as the designed identity. Every dbt run
in the platform's history had executed as `SYSADMIN`, which *owns* the shares and the
shared database and therefore reads them by ownership rather than by grant.

| Defect | Scope |
|---|---|
| No service role could read its school's CX data share | All 9 role/environment combinations, all 3 schools |
| No service role could read `DITTEAU_SHARED` (IPEDS, Scorecard) | Same |
| No `APPLY` on `rap_student_academic` for any dbt role | 9 of 9 — the Phase 6b fix reached the two policies created that day and never the Phase 4 one |
| `refresh_advisor_student_map()` existed in the three PROD databases only | The procedure driving the entire `SCOPED` tier had never existed anywhere it could be safely tested |
| `DBT_TEST_RESULTS` schema absent from all TEST and PROD databases | `+store_failures: true` is project-wide; the first PROD build would have written every table then failed in the test phase |
| `seed_cpi_index` materialised into `DITTEAU_SHARED` | **Every school's dbt run truncated and reloaded one physical table in a cross-tenant database** |

That last one is the most serious thing we found all month. Under `SYSADMIN` it worked
silently for months. Under a service role it failed immediately, which is how we learned
that one tenant's build could corrupt reference data every other tenant reads.

### 3.3 Committed but never deployed — three defects

| Defect | How it presented |
|---|---|
| Phase 2's `add_business_roles.sql` had never been executed | The account contained 48 roles and not one business role, months after the migration was committed |
| `DITTEAU_SHARED.governance.role_definitions` was never created | ADR marked the item complete; the table does not exist |
| Service accounts existed but could not authenticate | `SVC_*` users had no `RSA_PUBLIC_KEY` against a JWT-only profile, and were `TYPE = PERSON` rather than `SERVICE` |

Executing SQL leaves no trace in git. A migration file in the repo is evidence of intent,
not of deployment.

### 3.4 Silent failure — five defects

Access control and left joins both fail quietly. A row access policy returns **empty
results, not errors**; a left join that matches nothing returns **NULL, not an error**.

- **`FA_ROLE` saw zero students.** It held `FULL` on financial aid and `SCOPED` on student
  academic, and `SCOPED` resolves through the *advisor* mapping table. Financial aid
  counselors are not advisors. Measured: 204 aid records, 0 students, 0 from the join.
- **`fact_student_term.term_key` was NULL for all 89,122 rows**, in every environment,
  since the model was written. The join compared a session code (`'FA'`, 4 chars padded)
  to a composite term code (`'FA--2023UNDG'`, 12 chars). It could never match.
- **`dim_program` join fan-out.** Nine models joined a conformed dimension on
  `program_code` alone, but its grain is `(institution, program_type, program_code)`.
  Anselm's census carried 22 duplicate rows and inflated FTE.
- **`snap_retention_term` duplicated its entire contents on every run** — a merge key with
  an always-`NULL` column, so no row ever matched.
- **`snap_cohort_milestone` fanned out across summer subsessions.** Four terms share the
  start date 2010-05-24 (`--`, `D1`, `E1`, `S1`), and cohorts keyed on the date, so one
  intake became four cohorts.

### 3.5 The tooling itself was broken — four defects

We attempted to validate policy attachment by running a build with the flag enabled. It
returned **`PASS=1213 WARN=16 ERROR=0`** and attached nothing: the school run scripts put
their own `--vars` *after* `"$@"`, and dbt honours only the last one. The flag never
reached dbt.

Three more of the same shape followed:

- **A build reported "exit code 0" that was `tail`'s status, not dbt's**, because the
  command had been piped. The run had two errors.
- **The deposit clone reported reaching `3500/3537` against a target holding 1,093
  tables.** An auth token expired mid-run and the per-table `except` caught each
  subsequent failure as a table error, churning through ~2,445 doomed statements.
- **Test result tables outlived their tests.** Renaming a test gives it a new
  `store_failures` table and leaves the old one behind forever. Ten had accumulated,
  including one reporting 547 failures for a test deleted weeks earlier.

Worth noting the obvious fix to the first was wrong. Moving `--vars` earlier would have
inverted the bug into something worse: dbt *replaces* the vars dict rather than merging
it, so a caller passing one key would have wiped `school_code` and every feature flag —
silently, and still exiting 0.

### 3.6 Designed against assumed data — six defects

| Assumption | Reality |
|---|---|
| Masking policies protect SSN, phone, address in the analytics layer | **No such column exists there.** That PII sits in `deposit` and `deterge`, which no business role can reach |
| The six-policy set covers the layer's PII | It missed `STUDENT_FULL_NAME` — the most directly identifying column present |
| One policy signature fits all domains | Applicants are not students; an applicant who never enrols has no `student_id` |
| `mart_admissions_funnel` is applicant-grain | Deployed grain is program × entry term with count measures — the doc was stale |
| Advisor assignments come from the SIS of record | The procedure read `stg_jcx__students` for **every** school. Anselm runs Workday, Merrimack runs J1 |
| Advisor data exists somewhere | **It does not, in any school.** Anselm's Workday export has no advisor column at all; Merrimack's J1 advisor tables hold 0 rows; DEMEAU's is NULL in all 3,328 rows |

### 3.7 Fixes that reached one sibling and not the others — four defects

A pattern distinct enough to name. In each case the code looked correct exactly where you
would naturally read it.

| Fix | Reached | Missed |
|---|---|---|
| `GRANT APPLY` corrected off SYSADMIN | `rap_financial_aid`, `rap_admissions` | `rap_student_academic` |
| `refresh_advisor_student_map()` deployed | 3 PROD databases | 6 DEV and TEST databases |
| dim_term composite join with `TRIM` | `fact_enrollment` | `fact_student_term` |
| `advisor_id` precedence by `primary_sis` | `primary_program`, `is_active` | `advisor_id`, hardcoded JCX-first with Workday absent entirely |

### 3.8 Tests that measured their own assumptions — two defects

- **A relationship test flagged 3,436 orphans; 3,435 were fictional.** It resolved against
  the CX *share* while the model unions share **and** CSV deposit rows, so every
  deposit-sourced student was an orphan by construction. Split by ingest type: 35,398
  share rows produced 1 flag, 3,435 deposit rows produced 3,435.
- **Role-scoped access tests could not scope anything.** `DEFAULT_SECONDARY_ROLES = ["ALL"]`
  keeps `ACCOUNTADMIN` active as a secondary role no matter what primary role a
  connection requests, and object access resolves against the union. Three deliberate
  cross-tenant reads all *succeeded* before we found it.

### 3.9 Documentation asserting things that were not true — three defects

Three model docs stated masking was "applied in PROD". It never was. The ADR marked four
criteria complete that were not. `CLAUDE.md` described the `fact_student_term` → `dim_term`
join as working, with a note explaining the `TRIM()` fix — for a column that was 100% NULL.

This is the category with the sharpest external consequence. A documented control that is
not operating is a **worse** audit finding than an absent one, because the auditor tests
the described control and finds it missing. We are mid-SOC 2 Type II.

---

## 4. The lessons

### 4.1 A criterion is not met until its effect is observed

Not "the macro is written." Not "the build passed." **The effect.**

| Change | Required observation |
|---|---|
| Policy attached | a row in `information_schema.policy_references` |
| Procedure deployed | a successful call |
| Grant issued | a read performed **as the grantee** |
| Test added | a confirmed failure when its invariant is deliberately broken |

### 4.2 Run as the identity that will run it in production

This is the lesson of the second phase, and it produced more findings than any code
review could have. Six defects were invisible for months purely because `SYSADMIN` owns
the objects it was reading. Ownership is not a grant, and testing under ownership tests
nothing about grants.

The corollary is uncomfortable: **any environment that has only ever been driven by an
administrator has an unknown number of privilege defects**, and the only way to find them
is to stop being an administrator.

### 4.3 Setting a role does not scope a session

A person's session carries their secondary roles regardless of the primary role requested.
`USE SECONDARY ROLES NONE` is mandatory for any access test, and dedicated service users —
one role each, no secondary roles — are the durable answer.

Note the direction this failed in: it produced a **false alarm** (a denial test that
passed when it should have failed). The identical mechanism produces a silent **false
pass** whenever a test expects access to be allowed, which is the more common case.

### 4.4 Exit code zero means "nothing raised", not "something happened"

Three separate incidents. A green build that attached nothing. A pipe that replaced dbt's
exit code with `tail`'s. A progress counter that reached 3500/3537 over a target holding
1,093 tables.

Ask: *what would I observe if this had actually worked?* Then observe that.

### 4.5 An incremental model hides its own duplicates after the first run

First run does `CREATE TABLE AS` — no merge, no dedup. Every later run merges on the
unique key and silently collapses duplicates, keeping a non-deterministic winner.

So a fan-out defect is visible **only** on a first build. DEV had grown incrementally for
months and showed zero duplicates; PROD's first build surfaced six immediately. Every new
school's first build is the one chance to see this class of bug.

### 4.6 Access control fails silently, so it needs louder tests

A misconfigured RAP produces empty results, not errors. Users report missing data; nobody
reports a security problem. Three tests now guard the invariants, each written to fail
loudly rather than deny quietly.

### 4.7 When you fix a defect, look for its siblings

Four defects in §3.7 shared this shape. The habit that catches them is asking "what else
is shaped like this?" before closing the ticket — not "is this file correct now?"

### 4.8 Design against the data you have

Six defects came from reasoning about a schema rather than querying it. Checking first
would have cost minutes. The starkest: we built an entire access tier around advisor
caseloads, and no school has advisor data in any source system.

### 4.9 Separate decisions that sound similar

Row access and column masking were originally to resolve through the same tier lookup.
Splitting them was the single most consequential design change in Phase 6.

The proof arrived immediately: we widened `FA_ROLE` to `FULL` for a legitimate row-access
reason. Under the original design that approval would have **silently unmasked names and
dates of birth** — a privacy change nobody agreed to, arriving as a side effect of an
operational one.

### 4.10 Absence is safer than an explicit permissive-adjacent value

`{CODE}_STREAMLIT_OWNER_{ENV}` is deliberately **absent** from the tier grid rather than
present with tier `NONE`. Both deny today. But an explicit row invites a later
well-meaning edit to `FULL`, which would grant every Streamlit viewer full access at once.

### 4.11 A shared resource written by every tenant is not shared, it is contended

`seed_cpi_index` was configured into `DITTEAU_SHARED` because CPI data is national and
identical for every school. The reasoning was sound; the implementation gave nine
environments write access to one table. **"Same content" does not require "same physical
table"** — it is 11 rows.

---

## 5. What we changed

**Identity and access**

- Nine service accounts commissioned: key-pair auth, `TYPE = SERVICE`, one role each,
  `DEFAULT_SECONDARY_ROLES = ()`
- `demeau_test` and `demeau_prod` now run as service users, not as a person
- Six categories of missing grant repaired across all three schools by migration
- `DITTEAU_SHARED_READER` role added, so onboarding a school is one `GRANT ROLE` rather
  than ~144 statements

**Process**

- A criterion is complete when its effect is observed
- Flag-gated code is exercised in DEMEAU DEV with the flag on, then reverted
- Deployment state is verified against Snowflake, never inferred from the repo
- Tests are verified to fail before being trusted to pass

**Code and tooling**

- All six run scripts rewritten to merge `EXTRA_VARS` and **reject** a command-line
  `--vars`; DEMEAU's is now env-aware via `ENV=dev|test|prod`
- `clone_deposit.py` — zero-copy deposit promotion, refuses cross-school clones, survives
  token expiry, `--resume` for interrupted runs
- `run_migration.py` — splits `$$`-quoted procedure bodies, per-statement status,
  `--dry-run` first
- `find_stale_test_tables.py` — diffs `store_failures` tables against the manifest
- Governance tests live in `tests/governance/`, proposed for CODEOWNERS protection

**Documentation**

- [Production Operating Rules](production-operating-rules.md) — the rules this review produced
- Runbook §E.1.4 and §E.6 record the secondary-roles trap and the service-user gap
- Four ADR criteria re-marked with the correction visible rather than silently fixed

---

## 6. Where the platform stands

DEMEAU now has three fully built environments. TEST was built last, entirely from
corrected code by a scoped service identity, and came up clean on the first attempt:

```
PASS=1216  WARN=15  ERROR=0  SKIP=0  TOTAL=1231
```

| | DEV | TEST | PROD |
|---|---|---|---|
| `DETERGE` objects | 239 | 239 | 239 |
| `DISTRIBUTE` objects | 54 | 54 | 54 |
| `fact_student_term` null `term_key` | 0 | 0 | 0 |
| `snap_cohort_milestone` rows | 927 | 927 | 927 |

That TEST matched PROD exactly, with no errors and no manual intervention, is the
strongest evidence we have that the defects were fixed at the source rather than patched
in place.

The promotion path now runs the right direction: institutional feeds land in PROD, and
DEV and TEST receive deposit by zero-copy clone from it.

---

## 7. What is still open

| Item | Owner |
|---|---|
| Masking policies exist in DEMEAU DEV only — absent from 8 of 9 databases | DE |
| `user_domain_access` empty everywhere; **required** before any Streamlit RLS demo, since owner's-rights makes the role path deny | DE / KKM |
| Eight models flagged for small-cell re-identification review | KKM |
| Whether `FA_ROLE` needs unmasked names and dates of birth | KKM |
| DEMEAU's share appears derived from Anselm's — `ID_REC` matches at 356,950 rows. "Synthetic" may be the wrong word, with different obligations | KKM |
| Nine service-account private keys on an engineer workstation | WDT |
| CODEOWNERS on governance paths; CI running the governance tests | WDT |
| Advisor roster empty at every school — no source system has the field | Institutions |
| Four dashboards still hardcode `DEMEAU_DD_DEV` | DE |
| Merrimack and Anselm have DEV only; no TEST or PROD built | DE |

---

## The one-line version

> Most of what was wrong could not be found by reading the code, because the code was
> never run — and the rest could not be found by running it as ourselves, because we had
> privileges the system was never meant to rely on. We had confused *written* with
> *working*, and *works for me* with *works*.
