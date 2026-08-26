# ADR-006: Mask Out-of-Domain Columns in Cross-Domain Models

**Status:** Accepted

**Date:** 2026-08-26

**Author:** LVP (ratified by WDT for the masking rule, by KKM for the health-domain bound)

---

### Context

ADR-003 records a constraint and stops there: Snowflake permits **one row access policy
per table**, so a conformed dimension has a single domain owner, and that ownership
determines what every role reads from it regardless of their tier in any other domain.
ADR-003 calls this *"not a limitation to work around ... a constraint to design the tier
grid against"*, which is correct for dimensions. It leaves open what happens to a model
whose *content* spans domains.

The access policy had meanwhile written an intersection rule for exactly that case: a
viewer must hold entitlement in every contributing domain to see any row. That rule was
written before anyone checked whether it could be built. It cannot. A person-grained
model spanning two domains cannot carry one policy per domain and have them AND
together, because there is only one policy slot. The rule was a statement of intent
described as a control.

Once ADR-005 made every model declare its contributing domains, the affected set became
countable rather than suspected. Five person-grained or aggregate models draw on more
than one domain, and three of them are person-grained and already carrying a
`student_academic` row access policy while holding student-accounts content:
`fact_student_term`, `mart_academic_progress` and `mart_student_at_risk`. A fourth,
`mart_registration_holds`, turned out to be the worst case — measured 2026-08-26 by
`hold_description`, 17,029 of its 19,237 rows were placed by the Business Office, which
is 88.5 per cent student-accounts content governed as though it were entirely academic.
A registrar holds `NONE` on student accounts in the entitlement grid and was reading all
of them.

A decision was needed because the models exist and are being read now, not because a
future model might have the problem.

---

### Decision

**A cross-domain model resolves rows under its grain domain's policy, and the columns
belonging to any other contributing domain are masked.** The viewer sees the row; they
do not see the values they are not entitled to. This composes with the existing
column-masking layer, needs no new policy machinery, and does not multiply policy bodies
by the number of domain pairs in use.

Ratified by WDT on 2026-08-26 with no per-model exceptions, including for
`mart_registration_holds`. The reasoning accepted was that a registration hold is the
registrar's business — it is what stops a student enrolling — while the amount owed is
not.

**The bound on this decision matters as much as the decision.** Masking asserts that the
sensitive thing is a column value. Where the sensitive thing is instead the **existence
of the row**, masking protects nothing and a row access policy is required. The
canonical case is the `student_health` domain, added the same day: a health-services hold
carries no amount to mask, and "this student has a Health Services hold" discloses a
health matter with every column masked. Anselm holds 313 such rows and Merrimack 2,360,
in databases holding real student records.

So the rule is:

- **Values sensitive, existence not** → mask the out-of-domain columns. This is the
  default and covers the billing content in all four affected models.
- **Existence sensitive** → the content leaves the model and gets its own row access
  policy. `student_health` works this way, and every persona holds `NONE` in it encoded
  as *absence* of a grid row rather than an explicit `NONE`, so entitling anyone requires
  an addition rather than an edit.

The two rulings are not in conflict once scoped: masking governs the billing content in
`mart_registration_holds`, and the health rows leave that model entirely for one governed
by `rap_student_health`.

**What this does not cover.** The masking is not yet applied — adopting a shape and
attaching the policies are different acts, and only the second is a control.
`rap_student_health` is deliberately not created either: the health rows still sit inside
`mart_registration_holds`, so a policy created now would attach to nothing, which is the
same trap already recorded for `rap_student_accounts`. The model split lands first.

A masking policy named after HIPAA was considered and rejected. HIPAA attaches only where
a health centre is a covered entity, which in practice means billing third-party payers
electronically, and FERPA education records are carved out of protected health
information even then. Whether a given institution's health centre bills insurance is a
per-tenant factual question for its agreement, not a platform decision, and naming a
policy after the statute would encode a legal conclusion the platform is not entitled to
reach.

---

### Consequences

#### Pros

- Resolves the constraint ADR-003 raised, using a mechanism that already exists. No new
  policy construct, no second lookup path, no parallel resolution to keep in step.
- Turns the intersection rule from an unimplementable statement of intent into something
  buildable. The policy previously described a control that could not exist.
- Scales with domain pairs rather than multiplying by them. A combined policy per domain
  pair would have needed one policy body per pair actually used.
- Gives a stated, reusable test for which mechanism to reach for — values versus
  existence — rather than deciding model by model. That test is what identified
  `student_health` as the exception before anything was built for it.
- Keeps the registrar's operational function intact. They still see that a hold exists
  and that it blocks registration, which is what they need to act.
- The health carve-out is fail-closed by construction: absence of a grid row denies, so
  the default for a new persona is no health access without anyone deciding it.

#### Cons

- **Masking leaks existence, and for the holds model that was accepted rather than
  solved.** A registrar learns which students carry a Business Office hold even with the
  amount masked. The entitlement grid gives them `NONE` on student accounts, so the
  posture and the mechanism disagree by one inference. This was a deliberate call with
  WDT, not an oversight, but it is a real residual.
- Two mechanisms now govern one conceptual problem, and the choice between them rests on
  a judgement about whether existence is sensitive. That judgement is not automatable and
  nothing asserts it was made correctly.
- `mart_registration_holds` needs splitting to honour the health carve-out, which is
  modelling work created by a governance decision — the shape of thing that gets deferred.
- Nothing is enforced yet. Four models still serve out-of-domain content unmasked, and
  the health rows are still readable by a registrar. The decision is recorded; the
  control is not built.
- A future model where existence is sensitive will take the default and be wrong, unless
  its author reads this ADR. The rule is written down but not asserted.

---

### Alternatives Considered

| Option | Pros | Cons |
|---|---|---|
| Status quo — leave the intersection rule as written | No work | It describes a control that cannot be built on Snowflake's one-policy-per-table limit. Four models keep serving out-of-domain content with nothing recorded about it |
| One combined policy per domain pair, ANDing both tiers | Correct; genuinely implements intersection; a single mechanism for every case | Multiplies policy bodies by the number of domain pairs in use, and each needs its own body, signature and attachment. Reintroduces the parallel-lookup problem ADR-003 removed |
| Split every cross-domain model so each domain owns its own table | Cleanest governance; existence and values both protected; no judgement call needed | The most modelling work by a wide margin; fragments models that exist because the joined view is what consumers want; five models become at least ten |
| Mask the out-of-domain columns **(chosen)** | Uses existing machinery; scales with pairs rather than multiplying; composes with the column layer | Leaks row existence, which is unacceptable where existence is the sensitive fact — hence the `student_health` exception |
| Mask everywhere including health | One rule, no exceptions, simplest to explain | Protects nothing on a health hold, which carries no amount. Would have left 313 Anselm and 2,360 Merrimack health rows readable by a registrar |

---

### References

- [ADR-003: Row Access Policy Architecture](adr-003-row-access-policy-architecture.md) — the one-policy-per-table constraint this decision resolves
- [ADR-005: Declare Domain and Grain at Build Time](adr-005-model-authoring-contract.md) — made the affected model set countable
- [Row Access Policies runbook](../runbooks/row-access-policies.md)
- `/Users/laurievanpelt/ditteau_data_transform/docs/governance/data_access_policy.md` — F.2 is the cross-domain rule; B.4 covers the health-record boundary; D-9, D-19 and D-20 are the decisions
- `/Users/laurievanpelt/ditteau_data_infra/school_setup/migrations/add_governance_meeting_decisions_2026-08-26.sql` — section 4 is the `student_health` stub and records why it is not yet executed
