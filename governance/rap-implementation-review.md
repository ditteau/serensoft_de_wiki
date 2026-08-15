# Row Access Policy Implementation — Review and Lessons

A retrospective on ADR-003, covering what we built, the eighteen defects we found along the way, and what we changed about how we work as a result.

**Period:** August 2026
**Author:** LVP
**Audience:** Ditteau DE team

---

## Contents

- [1. What we set out to do](#1-what-we-set-out-to-do)
- [2. What we built](#2-what-we-built)
- [3. The defects, grouped by what caused them](#3-the-defects-grouped-by-what-caused-them)
- [4. The lessons](#4-the-lessons)
- [5. What we changed](#5-what-we-changed)
- [6. What is still open](#6-what-is-still-open)

---

## 1. What we set out to do

ADR-003 specified domain-scoped row access: a registrar sees every student, an advisor sees only their advisees, an institutional researcher sees aggregates and no individual rows. Four access tiers, four data domains, applied per institution across nine Snowflake databases.

Phases 1–2 were recorded as complete before this work began. We picked up at Phase 3 and carried through Phase 6.

**The headline result is not the feature.** It is that a system recorded as 60% complete was, on inspection, largely unexecuted — and that most of what was wrong could not have been caught by reading the code.

---

## 2. What we built

| | |
|---|---|
| Business roles | 12 (registrar / advisor / financial aid / IR analyst × 3 schools) |
| Tier grid rows | 360 (10 roles × 4 domains × 9 databases) |
| Domain policies | 3 — `student_academic`, `financial_aid`, `admissions` |
| Policy attachments | 9, across 8 models |
| Masking policies | 7 (4 attached to real columns, 3 deliberately unattached) |
| Models inventoried | 44, classified for whether their grain reaches an individual |
| Automated governance tests | 3, each verified to fail when its invariant is broken |

Tier resolves `COALESCE(user_grid, role_grid)`; the scoped predicate resolves against `CURRENT_USER()`. Masking resolves against a **separate** grid keyed on PII class, not on access tier.

Enforcement remains off in every environment. That is deliberate and gated on institutional rollout.

---

## 3. The defects, grouped by what caused them

Eighteen in total. They are more useful grouped by root cause than listed chronologically.

### 3.1 Code behind a feature flag — five defects

Everything gated on `enable_row_level_security` or `enable_masking_policies` had never executed, because neither flag has ever been true in any environment.

| Defect | Consequence if undetected |
|---|---|
| `apply_rap()` resolved the policy path via `var('env')` — no such var exists in this project | Enforcement fails on the first PROD model with `Required var 'env' not found` |
| `enable_masking_policies` is read by no code at all | Three model docs described masking as active; nothing masked anything |
| Policy signature was `VARCHAR`; every target column is `NUMBER` or `TEXT` | Policy creates successfully, then fails at attachment (`error 003554`) |
| `fact_enrollment` carried only a surrogate key, no natural `student_id` | Attaches cleanly, matches nothing, returns zero rows for every scoped user |
| `GRANT APPLY … TO ROLE SYSADMIN` — SYSADMIN already owns the policies | Grant is a no-op; the dbt service role that runs the hook has no APPLY |

The pattern: **a feature flag guarantees its own code is untested.** Each of these was written, reviewed, wired and recorded as done. None could run.

### 3.2 Committed but never deployed — three defects

| Defect | How it presented |
|---|---|
| Phase 2's `add_business_roles.sql` had never been executed | The account contained 48 roles and not one business role, months after the migration was committed |
| `DITTEAU_SHARED.governance.role_definitions` was never created | ADR marked the item complete; the table does not exist |
| `refresh_advisor_student_map()` fails on first call | It targets a `deterge` layer that does not exist — **no PROD database has ever been built in any school** |

Executing SQL leaves no trace in git. A migration file in the repo is evidence of intent, not of deployment.

### 3.3 Silent failure — three defects

Access control fails quietly by design. A row access policy returns **empty results, not errors**.

- **`FA_ROLE` saw zero students.** It held `FULL` on financial aid and `SCOPED` on student academic, and `SCOPED` resolves through the *advisor* mapping table. Financial aid counselors are not advisors. Measured: 204 aid records, 0 students, 0 from the join. Every aid analysis touching demographics would have returned nothing, presenting as "the dashboard is broken" rather than "access control is misconfigured."
- **`dim_program` join fan-out.** Nine models joined a conformed dimension on `program_code` alone, but its grain is `(institution, program_type, program_code)`. Codes existing under multiple types silently multiplied rows — Anselm's census carried 22 duplicate rows and inflated FTE.
- **`snap_retention_term` duplicated its entire contents on every run.** Configured as a merge with a six-column key, one of which is always `NULL`. `NULL = NULL` is never true, so no row ever matched and every run inserted a full duplicate set.

### 3.4 The tooling itself was broken — one defect, and the most instructive

We attempted to validate attachment by running a build with the flag enabled. It returned **`PASS=1213 WARN=16 ERROR=0`** and attached nothing.

Every school run script is shaped:

```bash
dbt "$@" --target X --vars '{ ...school vars... }'
```

The script's `--vars` comes *after* `"$@"`, and dbt honours only the last `--vars`. Any variable passed on the command line was silently discarded. The flag never reached dbt, the post-hook no-opped, and the build reported success.

**We would have recorded "attachment validated" on the strength of an exit code.** The only reason we did not is that the plan required querying `information_schema.policy_references` to observe the attachment rather than infer it.

Worth noting the obvious fix was wrong. Moving `--vars` earlier would have inverted the bug into something worse: dbt *replaces* the vars dict rather than merging it, so a caller passing one key would have wiped `school_code` and every feature flag — silently, and still exiting 0.

### 3.5 Designed against assumed data — four defects

| Assumption | Reality |
|---|---|
| Masking policies protect SSN, phone, address in the analytics layer | **No such column exists there.** That PII sits in `deposit` (305 tables) and `deterge` (27), which no business role can reach |
| The six-policy set covers the layer's PII | It missed `STUDENT_FULL_NAME` — the most directly identifying column present |
| One policy signature fits all domains | Applicants are not students; an applicant who never enrolls has no `student_id` at all |
| `mart_admissions_funnel` is applicant-grain (per the modelling doc) | Deployed grain is program × entry term with count measures — the doc was stale |

### 3.6 Documentation asserting things that were not true — two defects

Three model docs stated masking was "applied in PROD via `enable_masking_policies`." It never was. The ADR marked four criteria complete that were not.

This is the category with the sharpest external consequence. A documented control that is not operating is a **worse** audit finding than an absent one, because the auditor tests the described control and finds it missing. We are mid-SOC 2 Type II.

---

## 4. The lessons

### 4.1 A criterion is not met until its effect is observed

Not "the macro is written." Not "the build passed." **The effect.**

- For a policy: query `policy_references` and see the attachment
- For a procedure: call it
- For a table: select from it in the target account
- For a test: verify it *fails* when its invariant is deliberately broken

We adopted this mid-work and it immediately caught things. All three governance tests were verified in both directions — each was deliberately violated and confirmed to fail before being trusted.

### 4.2 A feature flag guarantees its own code is untested

Anything gated on a flag that has never been true has never run. Treat "behind a flag" as equivalent to "unwritten" until exercised with the flag on in a safe environment.

Our answer: flip the flag in DEMEAU DEV, observe the effect, flip it back. DEMEAU's CX share is de-identified and authorised for demonstration use, which makes it the correct place to do this. That single practice would have caught five of the eighteen defects.

### 4.3 Exit code zero means "nothing raised," not "something happened"

The false validation pass is the clearest lesson in this whole body of work. A green build told us nothing about whether the thing we cared about occurred.

Ask: *what would I observe if this had actually worked?* Then observe that.

### 4.4 Access control fails silently, so it needs louder tests

A misconfigured RAP produces empty results, not errors. Users report missing data; nobody reports a security problem. This inverts the usual debugging instinct, where an error points at its cause.

Three tests now guard the invariants, and each is written to fail loudly rather than deny quietly:

| Test | Guards against |
|---|---|
| `assert_no_app_owner_in_role_domain_access` | A Streamlit owner role gaining a tier — which would apply that tier to *every* viewer |
| `assert_no_unimplemented_scoped_tiers` | A `SCOPED` row in a domain with no scoped branch — provisioned-looking, sees nothing |
| `assert_cross_domain_tier_coherence` | The `FA_ROLE` shape: non-zero in one domain, zero in a dimension it must join |

### 4.5 Read the entitlement grid across rows, not down columns

The 40-row tier grid was authored domain by domain. Every column was individually sensible. The `FA_ROLE` defect only appears when you read one role's row across all four domains.

This is structural, not incidental: Snowflake permits one row access policy per table, so a conformed dimension has a single domain owner, and that ownership fixes the tier every role gets on it **regardless of their tier elsewhere**. `dim_student` belongs to `student_academic`, so financial-aid access to student demographics is not expressible at dimension grain at all.

### 4.6 Design against the data you have

Four defects came from reasoning about a schema rather than querying it. Checking first would have cost minutes:

- The masking set targeted PII that is not in the layer it protects
- The policy signature assumed a type family that no target column uses
- A fact table was assumed to carry a natural key it does not have
- A mart's grain was taken from a doc that no longer matched the model

### 4.7 Separate decisions that sound similar

Row access and column masking were originally to resolve through the same tier lookup. Splitting them was the single most consequential design change in Phase 6.

The proof arrived immediately: we widened `FA_ROLE` to `FULL` for a legitimate row-access reason. Under the original design, that approval would have **silently unmasked names and dates of birth** — a privacy change nobody agreed to, arriving as a side effect of an operational one.

Masking now resolves against its own grid, keyed on PII class, defaulting to deny by absence. A registrar can be granted names without gaining dates of birth.

### 4.8 Absence is safer than an explicit permissive-adjacent value

`{CODE}_STREAMLIT_OWNER_{ENV}` is deliberately **absent** from the tier grid rather than present with tier `NONE`. Both deny today. But an explicit row invites a later well-meaning edit to `FULL`, which would grant every Streamlit viewer full access at once.

The same reasoning governs the unmask grids: sparse, with absence meaning masked.

---

## 5. What we changed

**Process**

- A criterion is complete when its effect is observed, not when the code is written
- Flag-gated code is exercised in DEMEAU DEV with the flag on, then reverted
- Deployment state is verified against Snowflake, never inferred from the repo
- Tests are verified to fail before being trusted to pass

**Code and tooling**

- All six run scripts rewritten to merge `EXTRA_VARS` and to **reject** a command-line `--vars` outright rather than discard it
- `run_migration.py` added — splits on `$$`-quoted procedure bodies, reports per-statement status, `--dry-run` first
- Governance tests live in `tests/governance/`, and that path is proposed for CODEOWNERS protection, because deleting a test is the quiet way to undo a control

**Documentation**

- Four ADR criteria re-marked with the correction visible rather than silently fixed
- Three model docs corrected where they asserted masking was active
- A runbook section recording the run-script trap, with the false-pass incident as the worked example
- Four DEV/TEST/PROD asymmetries recorded, each found by hitting it

---

## 6. What is still open

| Item | Owner |
|---|---|
| Eight models flagged for small-cell re-identification review — `mart_admissions_funnel` currently has a cell of one admitted applicant | KKM |
| Whether `FA_ROLE` needs unmasked names and dates of birth, or whether row access with masked PII covers their work | KKM |
| CODEOWNERS on governance paths, making KKM's documented approval an operating control rather than a convention | WDT |
| CI running the governance tests — nothing currently runs them automatically | WDT |
| No PROD database has been built in any school | — |
| Advisor roster is empty; no institutional staff hold Snowflake accounts | — |

---

## The one-line version

> Most of what was wrong could not be found by reading the code, because the code was never run. We had confused *written* with *working*, and the only reliable cure was to observe the effect.
