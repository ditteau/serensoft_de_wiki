# ADR-004: Category-Based Data Retention Policy Schema

**Status:** Accepted

**Date:** 2026-08-10

**Author:** LVP

---

### Context

GDPR, FERPA, and Title IV audits all impose data retention obligations on the
Ditteau platform. Before this work, no retention metadata existed anywhere in the
project — source tables had no indication of how long their data should be kept,
and there was no mechanism to query or enforce retention by school or table.

The straightforward approach — tag each source table with a `retention_years`
integer and populate it — runs into a structural problem: the Ditteau platform
serves five institutions across three SIS vendors (Jenzabar CX, Jenzabar One,
Ellucian Banner) with more source systems possible in future. Legal retention
obligations attach to **what a record is** — a student academic record, a
financial aid disbursement, a prospect — not which system originated it. Keying
retention by source system would mean re-tagging all tables every time a school
migrates SIS, and would prevent expressing the policy in a vendor-neutral way.

A secondary problem: retention periods differ by institution. Springfield College
is a public institution subject to Massachusetts state records schedules that may
differ from the FERPA minimums applied to private colleges like Merrimack or
Anselm. Any schema design must support per-school variation without requiring a
separate override for every table at every school.

Governance gap G-09 was split into two sub-items:
- **G-09a** — Define the schema and tag all source tables (this ADR)
- **G-09b** — Populate actual retention values via client engagement (blocked;
  not covered here)

---

### Decision

Retention policy is tracked at the source table level in dbt source YAML meta
blocks, keyed by **data category** — a classification of what kind of record
a table contains — rather than by source system or table name alone.

#### Data categories

Every source table is tagged with exactly one `data_category`. Seven categories
are defined, assigned by primary data purpose:

| Category | Scope |
|---|---|
| `STUDENT_RECORDS` | Identity, demographics, contact information |
| `ACADEMIC_RECORDS` | Enrollment, grades, programs, degrees |
| `FINANCIAL_AID` | Aid awards, disbursements, Title IV compliance |
| `ADMISSIONS` | Applications, decisions, prospect data |
| `FINANCIAL` | Billing, payments, general ledger |
| `REFERENCE_DATA` | Lookup and validation tables, term codes, seeds |
| `PUBLIC_DATA` | Federal public datasets (IPEDS, College Scorecard) |

Categories are assigned by what the record **is**, not where it came from. A
student identity record in Jenzabar CX (`id_rec`) and a student identity record
in Banner (`spriden`) both receive `STUDENT_RECORDS`. This means the category
taxonomy survives SIS migrations.

#### Source meta fields

Two fields are added to every table-level meta block:

```yaml
tables:
  - name: id_rec
    config:
      meta:
        data_category: STUDENT_RECORDS
        retention_years: null          # null until G-09b client engagement
```

`retention_years: null` is intentional during G-09a — it is a declared
placeholder, not an omission. Null-safe defaults are provided by the
`metadata_defaults` macro so existing `add_source_metadata` calls do not break.

#### Per-school retention seed

`seeds/shared/seed_school_retention_policy.csv` stores the agreed retention
periods at the grain of `(school_code, data_category)` — one row per combination.
All five active schools × seven categories = 35 rows, all `retention_years` null
pending G-09b.

| Column | Purpose |
|---|---|
| `school_code` | Institution identifier |
| `data_category` | One of the 7 categories |
| `retention_years` | Integer years; null until client engagement |
| `retention_source` | `REGULATORY_FLOOR` or `INSTITUTIONAL_POLICY` |
| `notes` | Regulatory citations, agreed exceptions |

Per-school grain is necessary because Springfield (public institution) is subject
to Massachusetts state records schedules that may differ from the FERPA minimums
governing private institutions.

#### Three-tier resolution macro

`macros/metadata/get_retention_years(school_code, source_name, table_name)`
implements a priority lookup chain:

1. **Per-table override** — if `retention_years` in the table's source meta is
   non-null, return it. Handles tables that deviate from their category default
   (e.g., a school that retains transcripts permanently).
2. **Category-level seed** — query `seed_school_retention_policy` keyed on
   `(school_code, data_category)`.
3. **null** — not yet defined; G-09b will populate the seed.

The macro is callable from governance audit models and `on-run-end` hooks.

#### Field name normalisations

The two commits also standardised inconsistent meta field names across scorecard,
IPEDS, and JCX sources:
- `data_classification` → `data_class` (consistent with metadata_defaults)
- `is_pii` → `contains_pii` (consistent with deposit-layer convention)

---

### Implementation Checklist

**G-09a (complete)**
- [x] Define 7 data categories; document in `docs/governance/retention_policy.md`
- [x] Add `data_category` and `retention_years: null` to all 226+ source table
      meta blocks across 8 source YAML files (banner, scorecard, ipeds, jcx, j1,
      powerfaids, slate, workday)
- [x] Add `data_category: none` and `retention_years: none` defaults to
      `macros/metadata/metadata_defaults.sql`
- [x] Create `seeds/shared/seed_school_retention_policy.csv` (35 rows, all null)
- [x] Add seed YAML documentation to `seeds/shared/_shared_seeds.yml`
- [x] Write `macros/metadata/get_retention_years.sql` (3-tier lookup)
- [x] Rewrite `docs/governance/retention_policy.md` with category definitions,
      resolution chain, seed schema, and G-09b population instructions
- [x] Normalise `data_classification → data_class` and `is_pii → contains_pii`
      across scorecard, ipeds, jcx sources
- [x] Mark G-09a complete in `data_governance_policy.md`

**G-09b (blocked — separate engagement)**
- [ ] For each school, agree retention periods with client Registrar / Records
      Officer
- [ ] Populate `retention_years` in `seed_school_retention_policy.csv`; note
      any per-table overrides directly in source meta
- [ ] Run `dbt seed --select seed_school_retention_policy` per school
- [ ] Mark G-09b complete in `data_processing_register.csv`

---

### Consequences

#### Pros

- **SIS-migration-proof:** retention obligations are expressed in terms of data
  category, not source system; a school migrating from Jenzabar CX to Banner
  requires no re-tagging of retention policy
- **Per-school variation supported:** the seed grain `(school_code, data_category)`
  means Springfield's state-mandated schedule can differ from Merrimack's FERPA
  minimum without any schema change
- **Graceful two-phase rollout:** null placeholders are explicit and documented;
  existing models do not break because `metadata_defaults` provides null-safe
  fallbacks throughout
- **Auditable:** a governance audit model can query `seed_school_retention_policy`
  and call `get_retention_years` to produce a complete school × table coverage
  report at any time
- **Per-table override escape hatch:** the three-tier resolution lets any
  individual table deviate from its category default with a one-line meta change,
  no seed modification needed

#### Cons

- **G-09b is the critical path to enforcement:** the schema is complete but no
  retention values exist; the policy cannot be acted on until client engagement
  is done for each school
- **Category assignment is manual and judgement-based:** with 226+ tables across
  8 source systems, misclassification is possible; future source tables require
  deliberate assignment at staging time
- **Macro requires run context:** `get_retention_years` uses `run_query()` and
  cannot be called during compilation or in source definitions — only in model
  or `on-run-end` contexts
- **Seed is shared across schools:** `seed_school_retention_policy.csv` is a
  single file; when one school's values are agreed and populated, the file must
  be committed without accidentally setting values for schools not yet engaged

---

### Alternatives Considered

| Option | Pros | Cons |
|---|---|---|
| Source-system-keyed retention | Familiar; maps 1:1 to existing source YAML structure | Breaks on SIS migrations; doesn't match legal obligation model; duplicates values across schools using the same system |
| Per-table explicit values only (no category abstraction) | Maximum precision; no abstraction layer | 226+ values to maintain across 8 source files; no way to express cross-school variation without per-school source YAML forks |
| Flat global retention table (no per-school grain) | Simpler seed; one value per category | Cannot accommodate Springfield's public-institution state schedule or any other school-specific regulatory exception |
| Category-based with per-school seed **(chosen)** | SIS-migration-proof; per-school variation; auditable; graceful null rollout | Manual category assignment; G-09b still required for enforcement |

---

### References

- `docs/governance/retention_policy.md` — category definitions, resolution chain,
  G-09b population instructions
- `seeds/shared/seed_school_retention_policy.csv` — per-school retention seed
- `macros/metadata/get_retention_years.sql` — resolution macro
- `macros/metadata/metadata_defaults.sql` — null-safe defaults
- `docs/governance/data_governance_policy.md` — G-09a/G-09b gap register entries
- Governance items: G-09a (complete) · G-09b (blocked, pending client engagement)
- Commits: `5e1c423` (schema phase) · `8f42fc4` (category tagging + seed + macro) ·
  `44b0fa9` (gap register update)
