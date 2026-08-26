# ADR-005: Declare Domain and Grain at Build Time

**Status:** Accepted

**Date:** 2026-08-26

**Author:** LVP

---

### Context

ADR-003 established a domain-scoped row access architecture: entitlement is granted
over a subject area rather than over individual tables, so a new model inherits a
posture instead of requiring its own grant decision. That is the whole efficiency
argument for the design — five schools times three environments times a growing model
count makes per-table grant decisions unmanageable.

The architecture assumed a fact it never established. **A model can only inherit a
posture it has been assigned, and nothing assigned one.** Measured on 2026-08-25 across
the `ditteau_data_transform` project: of the 48 models in `deterge/intermediate/` and
`distribute/marts/`, zero declared a subject area, and nine intermediate models
(`int_students`, `int_applicants`, `int_course_registrations` and six others) had no
YAML entry of any kind. The access policy's merge gate asks, as its first question,
whether every changed model declares its domain, grain and grain entity. That question
was unanswerable for the entire project.

The consequence was not theoretical. Three person-grained marts were carrying student
billing content while governed by a `student_academic` row access policy, and nobody
had a way to enumerate which models were cross-domain other than reading all of them.
`mart_registration_holds` turned out to be 17,029 of 19,237 rows placed by the Business
Office — 88.5 per cent student-accounts content under an academic policy — and that was
found by querying the data, not by consulting metadata that should have recorded it.

A related problem shaped the mechanism rather than the decision. The project already
had a folder-level `meta.domain` key in `dbt_project.yml` carrying values like
`enrollment` and `registration`. Those are business areas, not policy domains, and
neither appears in the five-domain vocabulary the access policy defines. Anyone reading
`domain: enrollment` and treating it as an entitlement axis would be wrong, so the
governance keys needed to be distinguishable from it rather than layered on top.

---

### Decision

Every model in `intermediate/` and `marts/` declares three governance keys under
`config.meta` in its YAML: `domains` (a list, from the policy's controlled vocabulary),
`grain` (what one row represents), and `grain_entity` (the thing the grain counts).
Cross-domain models list every contributing domain. Models whose domain genuinely
cannot be derived carry `domains: []` plus `domains_unresolved: true` and a
`domains_question` explaining what is undecided — an explicit, flagged admission rather
than a silent default.

`tests/governance/assert_model_governance_metadata.sql` enforces it. The test reads the
**dbt graph** via `graph.nodes` at compile time rather than querying the warehouse,
because this is a property of the project rather than of what happens to be built. It
fails on four shapes: metadata absent; a domain outside the five-value vocabulary; a
domain the policy forbids outright (`IT`, `ADVANCEMENT`, `UNKNOWN`,
`INSTITUTIONAL_RESEARCH`); and an empty list with no `domains_unresolved` marker. It
deliberately does not fail on `domains_unresolved`, because six models legitimately
carry it pending a governance ruling and a test that fails from the day it lands gets
switched off.

The keys go under `config.meta`, not beside `description`. dbt 1.11 rejects a
model-level `meta` when `config.meta` also exists, and several models already had
`config.meta` blocks carrying `owner` and `ferpa_sensitive`. The governance keys merge
into those.

**What this does not cover.** The test asserts that a decision was recorded, never that
it was correct — nothing can check that mechanically, which is what the human merge-gate
questions and KKM sign-off are for. It also does not cover staging models, which inherit
their domain from their source table's registry `data_owner` and are governed by a
separate crosswalk.

---

### Consequences

#### Pros

- Makes ADR-003's central claim true rather than aspirational. Entitlement by domain is
  implementable only if every model has a domain; now every model does, and a new one
  cannot merge without one.
- Turns a twelve-question manual merge gate into one automated check plus eleven
  judgement calls. The manual gate was going to become a rubber stamp by the third week.
- Cross-domain models became enumerable for the first time. That immediately surfaced
  two gaps — three marts carrying billing content under an academic policy, and the
  registration-holds model being 88.5 per cent student-accounts — neither of which was
  visible before.
- Reading the dbt graph rather than the warehouse means the check runs in CI without a
  built database, and catches a bad model at test time rather than after deployment.
- The nine undocumented intermediate models now have YAML entries, so they are
  documentable and testable at all. That was a gap nobody had counted.
- `domains_unresolved` gives open questions a home. Six models carry it — the two
  identity-spine models, two federal-reference marts, `mart_executive_summary` and
  `mart_ipeds_reporting` — which is a shorter and more honest list than "we think most
  of them are fine".

#### Cons

- 48 models had to be classified by hand, and the classification is only as good as the
  judgement behind it. A wrong domain now looks settled, which is arguably worse than an
  absent one — the test cannot tell the difference.
- Adds a required step to every future model. A new mart cannot be merged without a
  governance decision about it, which is the point, but it is friction and it will be
  felt.
- The vocabulary is now load-bearing. A typo like `student_account` for
  `student_accounts` resolves to no tier and denies silently, which presents as a broken
  dashboard. The test catches this specific case, but only because the vocabulary is
  enumerated in the test as well as in the policy — two places to keep in step.
- The pre-existing folder-level `meta.domain` key still carries business-area values
  that are not policy domains. Both now coexist and the distinction is documented but
  not enforced. Renaming it to `business_area` is outstanding.
- Six models are classified as unresolved, so the coverage figure of 48 overstates how
  much is actually decided.

---

### Alternatives Considered

| Option | Pros | Cons |
|---|---|---|
| Status quo — no declared domain | No work; no new friction | ADR-003's inheritance model cannot function; the merge gate's first question is unanswerable; cross-domain models are undiscoverable except by reading all of them |
| Derive the domain from folder location | Zero per-model work; already implicit in `dbt_project.yml` | Folders are organised by business area (`enrollment`, `registration`), which does not map to the policy's five domains; cross-domain models have one location and several domains; a model moved between folders would silently change posture |
| Infer the domain from upstream `ref()` lineage | Automatic; no author decision; self-updating | Lineage gives contributing domains but not the **grain entity**, which is what F.1 says determines a model's domain. A student-grained fact carrying an aid amount is `student_academic` by grain and cross-domain by content, and lineage cannot distinguish those |
| Maintain the mapping in a seed table outside the models | Diffable in one file; no YAML churn across six files | Separates the declaration from the thing declared, so a new model can be merged with no entry and nothing notices until someone reads the seed |
| Declare in `config.meta` per model, enforced by a graph-reading test **(chosen)** | Declaration lives with the model; enforced at test time without a built database; supports explicit unresolved state | 48 models classified by hand; vocabulary duplicated between test and policy; a wrong domain looks settled |

---

### References

- [ADR-003: Row Access Policy Architecture](adr-003-row-access-policy-architecture.md) — the architecture this decision supplies the missing precondition for
- [Row Access Policies runbook](../runbooks/row-access-policies.md)
- `/Users/laurievanpelt/ditteau_data_transform/docs/governance/data_access_policy.md` — section F is the authoring contract; F.6 is the merge gate; N-2 is the assertion
- `/Users/laurievanpelt/ditteau_data_transform/tests/governance/assert_model_governance_metadata.sql`
- `/Users/laurievanpelt/ditteau_data_transform/models/deterge/intermediate/_int_models.yml` — includes the nine entries added for previously undocumented models
