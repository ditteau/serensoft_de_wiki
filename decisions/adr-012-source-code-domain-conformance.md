# ADR-012: Publish the Full Code Domain, Registered Union Observed

**Status:** Accepted

**Date:** 2026-09-15

**Author:** LVP

---

### Context

Jenzabar CX enforces almost no referential integrity between a coded column and
its decode table. A column can carry a code the decode never defined, and CX will
neither reject the write nor report it. This is not an edge case at one tenant:
it is a property of the source, and it will keep producing new instances.

The platform found out the expensive way. O-48: CX moved DEMEAU's entering cohort
from class level `FR` to `FF` in AY2023-24 while four models hardcoded `'FR'` as
the test for a new student. Every one of them absorbed the unrecognised value in
an `else` branch that resolved to `'continuing'`. The result was **583 new
students republished as returning**, the AY2023-24 retention cohort collapsing
from **506 to 7**, and twelve months of green builds. Nothing was null, so N-22
(`assert_no_undeclared_null_columns`) structurally could not see it — a dead code
path that yields a plausible *number* leaves no nulls behind.

The same shape turned up three more times in one week, and the instances are not
the same problem wearing different clothes:

| Instance | What it actually is |
|---|---|
| `TR` in Merrimack's `cl`, `CS` in its `cl_end` | Real codes, never added to that tenant's decode |
| Merrimack's `FF`, retired 2002, still in `cl_end` | Registered and withdrawn; twenty years of correct history |
| `E` in `cw_rec.stat` | Valid — absent even from the vendor's own m4 macro list |
| `reg_stat = 'TR'` tested against a `char(1)` | Not a data problem. A code defect |
| 8 rows of a U+FFFD byte in Merrimack's `reg_stat` | Not a code at all |
| Lowercase `r`/`d`/`w` in `cw_rec.stat` | Case-significant: historical/superseded, not a variant |

A single rule — "reject unknown codes", or "map them to a default" — is wrong for
five of those six. What was missing was not a rule but a *measurement*: nothing in
the platform compared the codes present in the data against the codes the decode
defines, so every one of these was invisible until someone went looking.

Scale rules out the obvious response. The CX share holds **662 `*_TABLE` lookup
tables, 476 of them non-empty**, against **16 staged** today. A hand-written
assertion per coded column does not survive contact with that number;
`stg_jcx__student_terms` alone carries seventeen columns that plausibly decode
against something.

---

### Decision

**A decode is published as the union of what the tenant registered and what the
tenant's data actually uses**, in an intermediate model, with two booleans
distinguishing the quadrants:

| `is_registered_code` | `is_observed_in_data` | Meaning |
|---|---|---|
| true | true | normal |
| true | false | **dead vocabulary** — defined, never used |
| false | true | **residue** — the code CX let through |
| false | false | impossible by construction |

Reference implementation, built and verified at all three provisioned schools:

- **`models/deterge/intermediate/int_jcx__class_level_codes.sql`** — `table`,
  gated on `has_jenzabar_cx`. Registered side is `stg_jcx__cl_table`; observed
  side scans all three consuming columns (`stu_acad_rec.CL`,
  `stu_acad_rec.CL_END`, `prog_enr_rec.CL`) via `school_ref()`. Carries
  `observed_row_count`, `observed_in_columns`, and first/last observed academic
  year.
- **`seeds/shared/seed_jcx_code_rulings.csv`** — per-tenant human rulings, keyed
  `(school_code, code_domain, code_value)` with a `disposition` from a fixed
  vocabulary: `valid_unregistered`, `retired_historical`, `corruption`,
  `superseded`. Two rows at ratification — Merrimack's `CS` and `TR`, both
  `retired_historical` (LVP, 2026-09-15). `+column_types` is set in
  `dbt_project.yml` and must stay: the seed was header-only when created and will
  be again for any school with no residue.
- **`tests/assert_class_level_codes_decoded.sql`** — rewritten to read the model
  and honour the seed. Still `severity: warn`.

Four rules govern the pattern:

1. **The decode stage stays 1:1 with its source; the union lives downstream.**
   Putting the scan inside `stg_jcx__cl_table` works today, but the moment
   anything upstream of a mart wants a decode attribute you get
   `stg_cl_table → stg_student_terms → stg_cl_table` and dbt fails at parse.
   Staging models are also views, so a scan added to one never executes at build
   (O-45) and a fault surfaces in whichever mart queries it first.

2. **Every consuming column is enumerated, or the model reports confidently wrong
   answers in both directions.** Measured: scanning `stu_acad_rec` alone misses
   `CS` entirely (it exists only in `CL_END`) *and* invents `UN` as dead
   vocabulary when it has 9 live rows in `prog_enr_rec.CL`.

3. **Membership excludes null and blank, and nothing else.** A well-formedness
   filter is how the thing you most need to see disappears. Corruption becomes a
   member with `is_registered_code = false` and is ruled on as corruption. A
   missing value is not a bad code, so it is not a member.

4. **Unclassifiable values route to a named bucket, never a real category.** This
   is the O-48 lesson and it is the half that actually prevents recurrence. The
   union makes the discrepancy visible; the named bucket is what stops it being
   absorbed while you are not looking.

**Scope boundary.** This ADR governs how a code domain is *published and
measured*. It does not decide what any individual code means — that is a human
ruling recorded in the seed. It also does not mandate staging all 476 populated
lookup tables: stage a decode when it backs a column consumed by a published
dimension, fact or mart.

---

### Consequences

#### Pros

- **The residue becomes queryable data, not just a red build.** Measured on first
  build: DEMEAU 24 registered / 4 dead / 0 residue; Anselm identical; Merrimack
  10 registered / 0 dead / 2 residue (`CS` 11 rows from 2002, `TR` 1 row from
  2003). That table did not exist in any form yesterday. Both Merrimack codes
  were ruled `retired_historical` the same day and the assertion went green —
  cleared by a recorded decision, not by weakening the predicate, which is the
  mechanism this ADR is really buying.
- **`observed_row_count` makes proportionate response possible.** Merrimack's
  `TR` is one row from 2003. Without materiality every residue code is an
  incident, which is the fastest route to an assertion being switched off.
- **Coverage improved immediately.** The rewritten assertion scans three columns
  where the old one scanned one, and `CS` had been invisible to it since it was
  written.
- **Dead vocabulary is a free by-product** and is genuinely informative — it
  names what a tenant has abandoned, which is exactly the signal that precedes a
  recoding like `FR` → `FF`.
- **Joins stop silently dropping rows.** A consumer joining the union gets a
  member for every code in its data.
- **The ruling seed converges.** A ruled code stops warning without anyone
  weakening a predicate to silence it — the failure mode that turns assertions
  into decoration.

#### Cons

- **Residue rows are attribute-free by construction**, and NULL decode attributes
  walk straight into `not is_degree_seeking` being FALSE for a NULL. Downstream
  must test positively. This is the same trap Merrimack's blank `prog` on `FF`
  already set, and the pattern multiplies the surface for it.
- **One model per decode does not scale by hand past a handful.** If this reaches
  twenty decodes the union models should be generated from a registry by a macro,
  which is work this ADR defers rather than solves.
- **The observed scan costs a full pass over every consuming column** on each
  build of the union model. Cheap at current volumes (835k rows at Merrimack,
  ~3s), not free, and it grows with the fact tables rather than with the decode.
- **Enumerating consuming columns is manual and silently decays.** A fourth
  consumer of class levels added next year under-reports until someone remembers
  to add it. Nothing detects that omission.
- **The seed is a governance artifact and will be misread as an allowlist.** Same
  standing risk as `seed_hold_domain_crosswalk` and `seed_null_column_baseline`,
  and the mitigation is the same: it is documented in three places and it will
  still happen.
- **`prog_enr_rec` carries no academic year**, so the observed-year range covers
  the term-record columns only. A code seen only there has a row count with NULL
  years — a real state that reads like a gap.

---

### Alternatives Considered

| Option | Pros | Cons |
|--------|------|------|
| **Registered ∪ observed, two flags (chosen)** | Discrepancy becomes data; joins never drop; materiality carried; dead vocabulary free | One model per decode; manual column enumeration; attribute-free residue rows |
| Status quo — hardcode code literals in models | No new objects | This *is* O-48. 583 students misclassified, a cohort collapsed 506 → 7, twelve months green |
| Per-column `is_registered_code` on the entity staging models | Flag sits next to the value that needs it | `stg_jcx__student_terms` has seventeen candidate columns; seventeen joins and seventeen columns on one view, times 31 entity models. Fan-out risk on every one |
| Union inside the decode stage itself | Simplest; one model instead of two | Latent parse-time cycle the moment an upstream model wants a decode attribute; views never execute the scan at build (O-45); re-scans on every downstream query |
| Single long-format coverage model across all decodes | One object; scales to 476 | No complete decode to join to, so joins still drop rows; loses the decode attributes entirely. Better as a later roll-up *of* these models than instead of them |
| Fail the build on any unregistered code | Impossible to ignore | Fails the nightly run at whichever tenant recodes first, which is the fastest way to get the assertion deleted. The silence was the danger, not the build status |
| Auto-insert observed codes into the decode table | Self-healing; test always green | An ignore-list with extra steps. Fabricates a registration no one at the institution made, and destroys the only signal that a recoding happened |

---

### References

- [ADR-005: Declare Domain and Grain at Build Time](adr-005-model-authoring-contract.md) — N-2 metadata contract the new model satisfies
- `ditteau_data_transform/models/deterge/intermediate/int_jcx__class_level_codes.sql`
- `ditteau_data_transform/seeds/shared/seed_jcx_code_rulings.csv`
- `ditteau_data_transform/tests/assert_class_level_codes_decoded.sql`
- `ditteau_data_transform/docs/kpi_provenance/enrollment_new_returning.md` — the O-48 case file
- `ditteau_data_transform/docs/governance/data_access_policy.md` §O — gap register, O-45 (suppress, do not correct) and O-48
