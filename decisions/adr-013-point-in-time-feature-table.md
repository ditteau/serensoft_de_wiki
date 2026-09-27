# ADR-013: Point-in-Time Feature Table for Persistence Modelling

**Status:** Accepted

**Date:** 2026-09-27

**Author:** LVP

**Serves:** Foresight Library FL-PRD-01 — *"Which currently enrolled students are
at elevated risk of not re-enrolling next term?"*

---

### Context

FL-PRD-01 asks for a model-scored persistence probability to replace the
rule-based `RISK_TIER`. Its Foresight Library entry names the missing input
directly: `mart_student_term_features` (proposed, ADR-013), at **student × term**
grain, against the platform's current **student × program**.

The gap is not just grain. A persistence model needs a *label* — did this student
come back — and features measured **as of the end of the term being scored**,
never as of today. Every existing mart is a current-state view: `is_current = true`
on the academic side, `is_active` on holds. Training on those leaks the future
into the features and produces a model that looks excellent in backtest and fails
in deployment.

The Point-in-Time Reconstruction investigation (RDT, v0.1) established that the
underlying history *is* recoverable: `fact_student_term` derives an SCD2 spine
from source term rows, `hold_rec` retains both window ends, and `dim_student`
carries 74,864 non-current versions at Merrimack. What did not exist was a table
that actually used any of it.

> **Note on the ADR number.** The Foresight Library v0.3 cover note reserves
> ADR-012 for this work. ADR-012 shipped 2026-09-15 as *Publish the Full Code
> Domain, Registered Union Observed*. This is **ADR-013**, and the library has
> been corrected.

---

### Decision

Build `mart_student_term_features` as a Distribute mart at one row per
`(student_id, term_key)` over main terms only, carrying one label column and a
set of as-of-term features.

Four decisions inside that are load-bearing, and three of them were forced by
measurement rather than chosen on principle.

#### 1. "Main term" is resolved by enrollment volume, not by a subsession code

`snap_retention_term` defines the FA → SP → FA chain as `term_subsess = '--'`
(ruled by LVP with RDT, 2026-09-12). That is correct at DEMEAU and **wrong at
Merrimack**, which changed its session encoding around 2017:

| Term code | Subsess | Dates | Students |
|---|---|---|---|
| `FAF32023UNDG` | `F3` | Aug 28 – Dec 15 | **4,940** |
| `FAF12023GRAD` | `F1` | Aug 28 – Oct 20 | 632 |
| `FAF22023GRAD` | `F2` | Oct 23 – Dec 15 | 722 |
| `FA--2023UNDG` | `--` | Aug 28 – Dec 15 | **1** |
| `WI--2023UNDG` | `--` | Jan 02 – Jan 15 | 586 |

At Merrimack `F3`/`R3` *is* the full semester; `--` is the January intersession
plus a vestigial pre-2017 encoding. Duration does not discriminate either —
`SP--2024UNDG` spans 114 days, one **more** than `SPR32024UNDG`'s 113, with zero
students.

The rule adopted is: within each `(term_year, term_sess, term_program_level)`,
the main term is the one the most distinct students registered in, ties broken by
duration then `term_code`. It is self-configuring per tenant *and per era*, and
it hands over cleanly across the switchover:

```
2016 FA  FA--2016UNDG  --   4,314     2017 FA  FAF32017UNDG  F3  4,298
2017 SP  SP--2017UNDG  --   4,013     2018 SP  SPR32018UNDG  R3  4,257
```

**Consequence for an existing model.** `snap_retention_term` still uses the
`'--'` rule and therefore holds **55 rows at Merrimack**, against 3,900–4,200
students per term that this model finds. Its published Merrimack retention rate
is built on almost nothing. Recorded as a follow-up; not fixed here, because
changing a published rate is a separate decision with its own consumers.

#### 2. The academic record is bridged on the semantic term, not on `term_key`

`fact_student_term` and `fact_enrollment` key **the same semester to different
`term_key`s** at Merrimack. The academic record stayed on `'--'` across its whole
range (2002–2025); registrations moved to `F3`/`R3` in 2017.

```
Fall 2023 academic record   FA--2023UNDG   4,524 students
Fall 2023 registrations     FAF32023UNDG   4,940 students
```

So the obvious join — on `term_key` — silently drops every post-2017 Merrimack
row. The first build of this model did exactly that and produced **NULL
`cumulative_gpa` on 50.85% of 149,553 rows**, on a table that still built and
still passed its uniqueness test.

The bridge is `(term_year, term_sess, term_program_level)` — the semester as a
human means it, independent of which session code a subsystem chose. DEMEAU
cannot catch this, because both facts use `'--'` there; the guard is therefore a
measured threshold (`assert_feature_academics_resolve`, 5%) rather than a code
review.

#### 3. The label is nullable, and the null is the point

`re_enrolled_next_term` is NULL when the outcome is not observable — the next
main term does not exist, or exists with no enrollments.

Labelling those FALSE would teach a model that the newest cohort all dropped out.
Measured at DEMEAU: the naive version scores `SP--2024UNDG` at **0 re-enrolled of
1,987**, a clean and entirely artificial 100% attrition signal sitting at the end
of the data — exactly the slice a deployed model scores.

`is_label_observable` marks it. `assert_feature_label_is_censored_not_false`
guards both directions at ERROR severity.

#### 4. Graduates are flagged, not filtered

Spring → Fall runs ~71% at Merrimack against Fall → Spring's ~90%, and most of
that gap is seniors completing. Excluding them:

| Transition | All students | Excluding completers |
|---|---|---|
| Fall → Spring | 90.4% | **91.8%** |
| Spring → Fall | 71.0% | **83.2%** |

3,942 of the 2018–2024 Merrimack spring cohort — about 15% — graduated and read
as "did not re-enrol". Training on them teaches the model that being a successful
senior predicts attrition.

`graduated_before_next_term` is provided and is **outcome-side**: it reads the
degree date from any `dim_student` version, including ones written after the
term. Exclude on it; never train on it. The model does not filter them out,
because "did they come back" and "should we have intervened" are different
questions and the caller owns that choice.

---

### What this model does *not* promise

**As-of-term, not as-was-known-at-the-time.** Features come from the row version
whose validity window covers term end, and holds are resolved by date range
rather than the build-time `is_active` flag. But the warehouse keeps one value
per version, not an edit history of that version. `stu_acad_rec.reg_upd_date`
shows **3.7%–4.7% of academic records per year edited after the year closed**,
some AY2019 rows carrying edit dates in late 2024.

So a GPA here is the *current best record of* that term, which is not always what
the registrar could see during it. A model trained on restated grades is mildly
optimistic about what was knowable at intervention time. This is a property of
the source, not a defect in the model, and closing it would require snapshotting
forward from now.

---

### Consequences

**Available.** 149,553 rows / 30,577 students / 92 terms / 2002–2025 at
Merrimack; 67,397 rows / 60 terms / 2007–2024 at DEMEAU. Both far exceed
FL-PRD-01's "3+ years of term transitions". Merrimack remains the build target:
DEMEAU has no `is_first_generation` at all, so only Merrimack supports the
two-axis fairness audit FL-PRD-01 requires.

**Fairness axes are carried, and fenced.** `is_urm`, `is_first_generation`,
`student_gender`, `student_ethnicity` exist for the audit and are marked in the
yml as not-features.

**FERPA.** `student_id` and `student_key` at row grain, RESTRICTED, FERPA-HIGH.
KKM sign-off required before any client-facing use, per FL-PRD-01.

**Two features carry corrections made the same week.** `on_track_gpa_flag` is now
nullable, where NULL means *no graded coursework yet* rather than *failing*;
coalescing it to false in a feature matrix reintroduces the defect.
`transcript_blocking_holds_at_term_end` depends on a hold decode corrected on
2026-09-27 — values computed before that date are low at Merrimack.

**Follow-ups recorded, not done here.**

1. `snap_retention_term` holds 55 rows at Merrimack under the `'--'` rule.
2. `fact_student_term` and `fact_enrollment` disagreeing on `term_key` for one
   semester is a platform-level trap, not a quirk of this model. Any other model
   joining those two facts on `term_key` has the same silent 50% loss.

---

### Alternatives considered

**Extend `snap_retention_term` to row grain.** Rejected: it is an incremental
aggregate with a merge key and weekly in-progress snapshots. A training table
wants a deterministic full rebuild, and mixing the two would make the label
depend on when the snapshot ran.

**Hardcode the Merrimack session codes.** Rejected — that is the defect ADR-012
records, and it would break again at the next tenant or the next encoding change.

**Filter graduates out of the model.** Rejected: it discards the only clean
positive control available. A senior who completes is the one case where "did not
re-enrol" is definitely not attrition, which makes them useful for validating
that the label means what it claims.
