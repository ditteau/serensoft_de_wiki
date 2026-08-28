# ADR-008: PII Access Logging on Built-In ACCESS_HISTORY

**Status:** Accepted

**Date:** 2026-08-27

**Author:** LVP

---

### Context

Gap C-01 ("No PII access logging") has sat in §7.5 of `data_governance_policy.md`
as a MEDIUM/P2 compliance gap, with the stated remediation "Enable Snowflake Query
History monitoring." Section K.1 of `data_access_policy.md` carries the matching
policy commitment and is marked `[PLANNED]`: query history retained per tenant,
every query against a person-grained model attributable to a named principal, and
access to that history restricted to `DITTEAU_ADMIN`.

The framing in both documents assumed capture had to be built. It did not.
`SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` has been recording column-level reads
since the account was created — 365,600 rows spanning 2026-02-24 to 2026-08-27 when
this was measured — with no configuration and no cost. C-01 was never a capture
problem. It was a reachability problem plus a retention problem.

Reachability had a specific cause. On 2026-08-24 ambient secondary roles were
removed from all person users, and with them the ambient `ACCOUNTADMIN` that had
been making `ACCOUNT_USAGE` readable from a `SYSADMIN` session. Afterwards every
role in the account except `ACCOUNTADMIN` received `Schema 'SNOWFLAKE.ACCOUNT_USAGE'
does not exist or not authorized`. Separately, the team had already declined to
grant `IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE` to the per-school dbt service
roles, because it would have given each school's ETL role visibility into every
other school's query history — so the obvious grant was closed off, and so was
implementing C-01 as a dbt model.

Three candidate definitions of "PII-classified table" existed and had to be
resolved before anything could be scoped. Two do not work. `contains_pii` and
`data_class` (190 and 276 occurrences) live in dbt YAML and describe only the
deposit and staging layers, which no persona holds a grant on, and dbt cannot read
`ACCOUNT_USAGE` in any case. Snowflake `DATA_CLASSIFICATION` tags exist, but all 38
tag assignments in the account are `DATABASE`-level and `MANUAL` — zero table-level,
zero column-level — so tagging a whole database `FERPA_PROTECTED` cannot
distinguish a PII column from any other.

---

### Decision

Build C-01 on the built-in views, scoped from live policy attachments, and accept
that it covers row access enforcement only.

One grant, one schema's worth of objects, deployed by
`school_setup/migrations/add_pii_access_log_2026-08-27.sql`:

- `GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE DITTEAU_ADMIN` — the
  granular governance-scoped role, not `IMPORTED PRIVILEGES`. It covers
  `ACCESS_HISTORY`, `POLICY_REFERENCES`, `TAG_REFERENCES` and the policy views, and
  nothing else.
- `DITTEAU_PLATFORM.GOVERNANCE.V_PROTECTED_COLUMNS` — the scope, derived from
  `ACCOUNT_USAGE.POLICY_REFERENCES` rather than declared. It is the live attachment
  inventory, already maintained by the `apply_masking()` and `apply_rap()`
  post-hooks in `dbt_project.yml`, so the protected set changes without anything
  here being edited.
- `DITTEAU_PLATFORM.GOVERNANCE.V_PII_ACCESS` — one row per (query, object, column)
  where the column carries a masking policy or its table carries a RAP, attributed
  to a named principal and flagged with whether a row access policy evaluated.
- `DITTEAU_PLATFORM.GOVERNANCE.PII_ACCESS_LOG` — append-only durable copy, because
  `ACCESS_HISTORY` retains 365 days and K.1 commits to per-tenant periods that will
  exceed that.
- `TASK_CAPTURE_PII_ACCESS` — daily at 06:00 UTC on `PLATFORM_WH`.

Objects are owned by `DITTEAU_ADMIN`, not `SYSADMIN`. These are owner's-rights
views over `ACCOUNT_USAGE`, so the owner needs `GOVERNANCE_VIEWER`; granting that
to `SYSADMIN` would put account-wide, cross-tenant access history behind the role
every developer session already lands in. K.1 already names `DITTEAU_ADMIN` as the
sole principal for cross-tenant query history.

**Scope boundary, and it is the important one: this log is evidence that row
access policies enforced. It is not evidence that masking enforced, and no view
built on `ACCESS_HISTORY` can be.** Measured 2026-08-27, `policies_referenced`
carries 1,397 `ROW_ACCESS_POLICY` firings across six users and **zero**
`MASKING_POLICY` entries in six months of history. This is not masking failing: a
direct probe the same day showed `DEMO_REGISTRAR` reading `Bryson, Ty Y` /
`1965-06-14` where `DEMO_ADVISOR`, `DEMO_FA` and `DEMO_IR` read `***` /
`1965-01-01`, so `mask_name` and `mask_dob` both demonstrably fire. Taking those
same `DEMO_ADVISOR` query IDs and reading their `policies_referenced` back returns
`RAP_STUDENT_ACADEMIC` and nothing else.

The consequence is that `ditteau_data_transform/scripts/capture_persona_results.py`
is the only artifact in the estate that demonstrates masking enforcement. It is
currently filed as demo tooling. It is a compliance control and must be scheduled
and retained as one.

Principal classification uses naming convention, not `ACCOUNT_USAGE.USERS.TYPE`.
The account holds 13 SERVICE, 3 PERSON and 1 NULL, and both obvious filters are
wrong: `TYPE = 'PERSON'` omits `LVANPELT`, which predates the column and reports
NULL, and excluding `TYPE = 'SERVICE'` omits all four `DEMO_*` personas, which are
service users and are the entire demonstration record.

Verified by `school_setup/platform/verify_pii_access_log.sql` — seven checks, all
PASS on deployment. It is deliberately platform-level rather than added to the
per-school `08_hardening.sql`: C-01 is one account-wide task writing one
account-wide table, and asserting it nine times would imply nine controls.

---

### Consequences

#### Pros

- Closes C-01 with four objects and two grants. No classification profile, no
  custom classifiers, no `apply_governance_tags()` post-hook, no platform tag set —
  all of which were scoped before measurement and are now unnecessary.
- Six months of history are already present. The log backfilled to 29,141 rows
  reaching back to 2026-07-12 on first run, so the control has retrospective
  evidence rather than starting empty.
- Produces enforcement evidence §N cannot. §N asserts policies are *attached*;
  `V_PII_ACCESS` shows they *executed*, per user, per query.
- The scope maintains itself. Because it derives from `POLICY_REFERENCES`, adding
  or removing an `apply_rap()` call changes what is logged with no edit here — the
  decoupling a tag-based scope was wanted for, without the tag substrate.
- `GOVERNANCE_VIEWER` is materially narrower than `IMPORTED PRIVILEGES`, which
  makes the cross-tenant seam K.1 permits much easier to defend at audit.
- Nothing writes classification, so C-01 stays outside the KKM approval gate on
  `data_class` / `is_pii`.

#### Cons

- **Masking enforcement is unevidenced by this log, permanently.** The audit story
  is split across two artifacts, and the second one is a Python script that runs on
  a workstation. Until `capture_persona_results.py` is scheduled and retained,
  masking enforcement has no durable record at all.
- **The log covers only what carries a policy.** At deployment that was 17
  attachments across two DEMEAU databases, reading 28 an hour later as `ACCOUNT_USAGE`
  latency caught up with the same-day D-28/D-29 migration — the scope is derived and
  lags the account by up to ~2h, which is a feature for accuracy and a trap for anyone
  quoting a fixed number. Nothing at Anselm or Merrimack at either reading. The
  two highest-traffic PII surfaces in the account are outside it: `DEMEAU_DD_DEV`
  (75,077 accesses, real Anselm identifiers per O-29, one masking attachment and no
  RAP) and `DEMEAU_CX_ARCHIVE` (23,929 accesses, raw deposit PII, neither). Reads
  there are in `ACCESS_HISTORY` but do not enter the log.
- Principal classification is pattern-based and will misfile a principal that
  breaks naming convention. It fails toward `PERSON`, which surfaces rather than
  hides, but it is a convention dependency rather than a guarantee.
- `V_PROTECTED_COLUMNS` going empty is a silent failure mode. `CREATE OR REPLACE
  TABLE` drops policy attachments, so a mis-sequenced rebuild empties the scope and
  the log goes quiet — and quiet reads as safe. Check 3 of the verification script
  exists for exactly this.
- The capture re-reads a 12-hour window every run to survive `ACCESS_HISTORY`'s ~3h
  latency, and depends entirely on the `NOT EXISTS` anti-join to stay idempotent.
  Remove either and the log silently loses rows or silently double-counts.
- One more account-wide object outside the per-school pattern, in a database
  (`DITTEAU_PLATFORM`) that was empty until now and has no other tenant.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Built-in `ACCESS_HISTORY`, scoped from `POLICY_REFERENCES` (chosen)** | Nothing to enable; six months of history already present; scope self-maintaining; narrow grant | Cannot evidence masking; covers only policy-protected objects |
| Status quo — no PII access log | No work | C-01 stays open; K.1 claims a control that does not exist; J.3 asserts auditing is the control on administrative access with nothing behind it |
| `QUERY_HISTORY` instead of `ACCESS_HISTORY` | What the Asana task literally asked for | No column-level detail. Identifying PII access would mean regex over SQL text, which is not auditable |
| dbt model over `ACCOUNT_USAGE` | Fits the existing governance-test pattern; assertions in `tests/governance/` | Requires `IMPORTED PRIVILEGES` on per-school dbt service roles — already declined, as it cross-wires tenant query history |
| Scope from Snowflake `DATA_CLASSIFICATION` tags | Snowflake-native; survives model rename; decoupled from enforcement | All 38 assignments are DATABASE-level and MANUAL; cannot distinguish a PII column. Would require standing up auto-classification first, and pulls C-01 under the KKM gate |
| Scope from dbt `contains_pii` / `data_class` | Broadest coverage of genuinely sensitive raw data | Describes deposit/staging only; dbt cannot read `ACCOUNT_USAGE`, so the halves live in different repos |
| Snowflake auto-classification + `CUSTOM_CLASSIFIER` | Would build the tag substrate automatically and reapply after rebuilds; `SNOWFLAKE.TAGS.SENSITIVITY` already carries the same four values as the hand-rolled tag, with lineage propagation implementing §F.4.4 for free | Real work, credits on a schedule across nine databases, and machine-assigned classification engages the KKM gate. Deferred, not rejected — see below |
| Extend `DQ_PLATFORM.LOG` | Reuses a reviewed logging framework with an existing task runtime | Puts governance evidence under `DQ_ADMIN` ownership, muddying who owns the audit trail |

---

### References

- Migration: `/Users/laurievanpelt/ditteau_data_infra/school_setup/migrations/add_pii_access_log_2026-08-27.sql`
- Verification gate: `/Users/laurievanpelt/ditteau_data_infra/school_setup/platform/verify_pii_access_log.sql`
- Masking evidence (the only source): `/Users/laurievanpelt/ditteau_data_transform/scripts/capture_persona_results.py`
- Policy: `data_access_policy.md` §K.1 (logging), §J.3 (administrative access), §N (deployment assertions)
- Policy: `data_governance_policy.md` §7.5 (compliance gaps — C-01)
- [ADR-003: Row Access Policy Architecture](adr-003-row-access-policy-architecture.md) — the policies whose firings this log records
- [ADR-006: Mask Out-of-Domain Columns in Cross-Domain Models](adr-006-cross-domain-column-masking.md) — the masking family this log cannot evidence
- [Row Access Policies runbook](../runbooks/row-access-policies.md)

---

### Follow-ups

1. Schedule and retain `capture_persona_results.py`. Until then masking
   enforcement has no durable evidence. This is the highest-value item here.
2. Decide whether unprotected-but-real-PII surfaces (`DEMEAU_DD_DEV`,
   `DEMEAU_CX_ARCHIVE`) need a companion view scoped by database rather than by
   policy. This is the tag question again, in a smaller form.
3. Agree per-tenant retention periods, which K.1 requires before go-live and which
   `PII_ACCESS_LOG` exists to satisfy.
4. Revisit auto-classification once the KKM gate is scheduled — it remains the
   right long-term substrate for classification, just not a prerequisite for C-01.
