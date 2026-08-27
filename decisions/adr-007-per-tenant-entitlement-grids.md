# ADR-007: Per-Tenant Entitlement Grids

**Status:** **Accepted — implemented 2026-08-26.** Both seed families exist, the hardcoded
grid is out of `generate_school.py`, and N-16 is deployed as three assertions, green on
the `demeau_dev`, `merrimack_dev` and `anselm_dev` targets. TEST and PROD need the seeds
built there before the assertions can run, so N-16's coverage today is DEV only.

> **Two things this ADR did not anticipate, recorded here because they changed the build.**
>
> **1. Every difference that existed was rollout lag, not tenant preference.** The ADR
> reasoned from divergence between `DEMEAU_DD_DEV` (36 rows / 9 roles) and
> `MERRIMACK_DD_DEV` (20 / 5) and treated it as the kind of variation overrides exist to
> record. Measurement showed otherwise: Anselm and Merrimack do not have the Admissions,
> Finance, Cabinet or Student Success **roles at all** — all seven persona roles exist
> only in DEMEAU. The divergence was a project plan, not a posture. So **all six override
> files ship empty**, and encoding the lag as `NARROW` rows would have been actively
> harmful: it would have frozen a mid-rollout state as deliberate policy and hidden the
> drift behind a label reading "intended", which is the exact failure this ADR set out to
> fix.
>
> **2. The divergence is per-(tenant, environment), not per-tenant.** `DEMEAU_DD_DEV`
> differs from `DEMEAU_DD_TEST` and `DEMEAU_DD_PROD`. The override seed is keyed by school
> only, so it cannot express an environment-specific posture — which is correct, because
> an environment-specific *entitlement* is drift by definition rather than policy. The
> one genuinely per-environment row, the `{CODE}_DBT_{ENV}` service identity, is
> deliberately kept out of the seeds entirely and emitted by `generate_school.py`, because
> it is mechanical rather than a policy decision.
>
> **N-16 justified itself on its first run** by finding the D-15 Financial Aid correction
> applied to DEMEAU alone, leaving four wrong cells across Anselm and Merrimack. Fixed by
> `reconcile_fa_grid_d15_anselm_merrimack.sql` across six databases — an `UPDATE`, not an
> `INSERT`, because the rows existed carrying the wrong value and the additive
> `WHERE NOT EXISTS` pattern would have run clean and changed nothing.

**Date:** 2026-08-26

**Author:** LVP

---

### Context

The access policy states one entitlement grid: eleven personas against six data domains,
platform-wide. The enforcement mechanism, however, has always been per-tenant —
`role_domain_access` lives in `{CODE}_DD_{ENV}.governance` and every role is
`{CODE}`-prefixed, so Anselm's Institutional Research office having a different tier from
Merrimack's requires no new machinery at all. Only different rows in different databases.

Three things stand between that capability and using it.

**The policy already grants the right to differ but provides nowhere to record it.**
Section B.1 says each tenant defines legitimate educational interest in its annual FERPA
notification and that *"where a tenant's definition is narrower, the tenant's definition
governs"*. Section I.1 calls the n ≥ 5 suppression threshold *"a floor rather than a
ceiling"* if a tenant's own reporting standards are stricter. Both sentences concede
tenant variation; neither says how it is expressed, who approves it, or how anyone would
know it had happened.

**The generator asserts uniformity.** `generate_school.py` builds
`role_domain_access` from a hardcoded `VALUES` list templated only on `{code}`, so every
school provisioned by it receives a byte-identical grid. A per-tenant difference today
means editing the generator or hand-editing the database afterwards, and the second is
how drift starts.

**Assertion N-16 cannot be built.** It is the drift check — *"entitlement tables contain
no row not authorised by G or H"* — and it is blocked because the grids are markdown
prose rather than data. The policy already records the intended fix: extract G and H to
seeds and treat the document's tables as generated from them.

Meanwhile divergence exists already, and by accident. `DEMEAU_DD_DEV` carries 36 rows
across 9 roles; `MERRIMACK_DD_DEV` carries 20 across 5. The Financial Aid drift
correction of 2026-08-26 was applied to DEMEAU only, deliberately, and nothing anywhere
distinguishes that deliberate scoping from unintended drift. The decision taken the same
day to defer office-by-office confirmation of grids G and H to a pre-go-live task
guarantees this problem arrives with the first tenant, because that task exists precisely
to produce per-tenant deltas.

---

### Decision

**The platform grid becomes a ceiling expressed as data, and each tenant carries a sparse
override recording only where it differs.**

Two seed families, following the pattern ADR-004 established for retention:

- `seeds/shared/seed_persona_domain_grid.csv` — `persona, domain, tier`. The platform
  ceiling, and the generated source of the tables printed in policy sections G and H.
- `seeds/{school}/seed_{school}_persona_grid_override.csv` — rows **only** where that
  tenant differs, carrying `persona, domain, tier, direction, legal_basis, approver,
  approved_date, review_on`.

`direction` is the load-bearing column, because narrowing and widening are different acts
with different authority:

| `direction` | Authority | Requires |
|---|---|---|
| `NARROW` | Tenant data owner, evidenced to KKM | Nothing further — B.1 already permits it |
| `WIDEN` | KKM, with the tenant's legal basis recorded | `legal_basis`, `approver`, `approved_date`, `review_on` all non-null |

**A widening does not raise the platform ceiling.** It is recorded against the one tenant
that needs it and leaves the shared grid untouched. This matters: if every tenant needing
something wider forced a platform-grid change, the ceiling would ratchet upward with each
onboarding and stop bounding anything.

N-16 then asserts three things, and becomes implementable:

1. Every row deployed in `{CODE}_DD_{ENV}.governance.role_domain_access` appears either in
   the platform grid or in that school's override.
2. Every `NARROW` row is genuinely narrower than the platform cell it replaces.
3. Every `WIDEN` row carries a legal basis, a named approver and a review date.

**Tier comparison is a partial order, not a total one.** `NONE < AGGREGATED < SCOPED-* <
FULL` holds, but the SCOPED variants are not nested — `SCOPED-ORG`, `SCOPED-SECTION` and
`SCOPED-CASELOAD` describe different populations rather than different sizes of the same
one. A change from one SCOPED kind to another is therefore neither a narrowing nor a
widening and must be classified explicitly rather than inferred. The test must refuse to
guess.

#### The four kinds of variation this does and does not cover

| Kind | Example | Handled by |
|---|---|---|
| Entitlement preference | Anselm's IR at `AGGREGATED` on financial aid where the platform says `FULL` | `NARROW` override row |
| Different legal regime | Whether `student_health` is HIPAA-covered depends on whether that institution's health centre bills third-party payers; Springfield is a public institution subject to Massachusetts records schedules that may compel disclosure FERPA permits withholding | `WIDEN` override row with legal basis |
| Domain unavailable at that tenant | Springfield on Banner may lack the billing tables that make `student_accounts` meaningful | **Not this ADR.** A separate implementability check, the shape of `assert_no_unimplemented_scoped_tiers` |
| Persona does not exist at that tenant | A small college with no separate Bursar — the function sits with the Registrar | **Not this ADR.** The role should not be created; an override row would wrongly imply the office exists |

**Scope boundary.** This ADR covers row-access grid G. Column grid H needs the same
treatment and the same seed shape, but H is `[PARTIALLY DEPLOYED]` and its per-tenant
variation has not been thought through — a tenant narrowing *column* access has different
mechanics, because masking is attached per column rather than resolved per row.

---

### Consequences

#### Pros

- Makes N-16 buildable, which closes the last structural gap in the drift story. Drift is
  currently detected by a human reading two documents side by side.
- Bounds the fleet with one review. No tenant can exceed the platform ceiling without an
  explicitly marked, legally justified `WIDEN` row, so reviewing the shared grid tells you
  the maximum any tenant can reach.
- Distinguishes deliberate scoping from drift, which is impossible today. The DEMEAU-only
  Financial Aid correction and the DEMEAU-only persona provisioning would both be
  recorded as intentional rather than looking like inconsistency.
- Sparse overrides keep the reviewable surface small. A full per-tenant grid is 5 schools
  × 11 personas × 6 domains = 330 cells; the override files show only the deltas, which
  should be a handful each.
- Reuses a pattern already in the repo rather than inventing one. Six `seeds/{school}/`
  folders already exist, including three for unprovisioned schools, so the plumbing and
  the `+enabled` guards are proven.
- Removes the hardcoded grid from `generate_school.py`, so provisioning a new school stops
  requiring a code change to give it a correct posture.
- Forces the legal basis into the record at the point of widening, rather than
  reconstructing it during an audit.

#### Cons

- Two sources for one answer. Reading a tenant's effective entitlement means composing the
  shared grid with an override file, and anyone who reads only the first will be wrong.
  Mitigated only by N-16 and by never printing the platform grid without noting overrides
  exist.
- The partial-order problem is a genuine sharp edge. A SCOPED-to-SCOPED change looks like
  a lateral move and is neither narrower nor wider; if the test's classification is ever
  loosened to "guess from an ordinal", it will silently permit real widenings.
- `WIDEN` is an escape hatch, and escape hatches get used. The controls are a required
  legal basis and a review date, both of which are only as good as the review actually
  happening on the K.2 cadence.
- Grid H is left out, so the platform will have machine-readable row entitlement and prose
  column entitlement for some period. That asymmetry invites the assumption that column
  access has no tenant variation, which is almost certainly false.
- Migrating the existing nine databases onto seed-derived grids is a reconciliation
  exercise against live tables, and one of them (`DEMEAU_DD_PROD`) has enforcement running.
  That is not a free refactor.
- The override files are per-school seeds, so a tenant's entitlement now lives in the
  transform repo. Whether a tenant should be able to see, review, or propose changes to
  its own file is an open question this ADR does not answer.

---

### Alternatives Considered

| Option | Pros | Cons |
|---|---|---|
| Status quo — one platform grid in markdown | No work; single source to read | N-16 unbuildable; the generator gives every school an identical grid; B.1's right to narrow has nowhere to be recorded; existing DEMEAU divergence is indistinguishable from drift |
| Fully independent per-tenant grids, no shared ceiling | Maximum flexibility; each tenant's file is complete and readable on its own | 330 cells to author and review with no shared baseline; a platform-wide review becomes five reviews; nothing bounds what a tenant can reach; the common case (identical to platform) is restated five times |
| Per-tenant overrides with no ceiling — first match wins | Sparse; simple test | Without a ceiling an override can widen silently, and there is no single artefact whose review bounds the fleet. Loses the property that makes the shared grid worth reviewing |
| Keep grids in the database only, no seeds | No migration; already per-tenant; nothing to keep in step | Entitlement stops being diffable or code-reviewed; N-16 has nothing authoritative to compare deployed state against, so it can only compare the database to itself |
| Shared ceiling plus sparse per-tenant overrides, direction-classified **(chosen)** | N-16 buildable; one review bounds the fleet; deliberate scoping distinguishable from drift; reuses ADR-004's proven pattern | Two sources to compose; SCOPED-to-SCOPED comparison must be explicit; grid H deferred; migrating nine live databases is real work |

---

### References

- [ADR-003: Row Access Policy Architecture](adr-003-row-access-policy-architecture.md) — supersedes the Phase 2 assumption that every school receives an identical grid
- [ADR-004: Category-Based Data Retention Policy Schema](adr-004-retention-policy-category-based-schema.md) — the per-school seed pattern this reuses, and the source of the Springfield public-institution example
- [ADR-005: Declare Domain and Grain at Build Time](adr-005-model-authoring-contract.md)
- [ADR-006: Mask Out-of-Domain Columns in Cross-Domain Models](adr-006-cross-domain-column-masking.md) — the `student_health` domain whose HIPAA status is per-tenant
- [Row Access Policies runbook](../runbooks/row-access-policies.md)
- `/Users/laurievanpelt/ditteau_data_transform/docs/governance/data_access_policy.md` — B.1 and I.1 concede tenant variation; G and H are the grids; N-16 is the assertion; K.2 sets the review cadence
- `/Users/laurievanpelt/ditteau_data_infra/school_setup/generate_school.py` — the hardcoded `role_domain_access` insert this decision removes
