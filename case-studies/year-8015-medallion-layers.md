# The Year 8015

**Where a correction belongs, and why Deposit is never the answer.**

A corrupt academic-year code in a Jenzabar CX share threw enrollment reporting off by
~1,900 students across three academic years. Fixing it was easy. Deciding *which layer*
to fix it in was the part worth writing down — and a second defect found on the way out
turned out to be the sharper lesson, because there the obvious fix was the wrong one.

<a href="case-studies/year-8015.html" target="_blank" rel="noopener"><strong>→ Open the presentation version</strong></a>
&nbsp;(designed single-page write-up, opens in a new tab)

| | |
|---|---|
| **Date** | 2026-09-07 |
| **Author** | LVP |
| **Models** | `stg_jcx__academic_calendar`, `dim_term`, `mart_enrollment_census` |
| **Schools** | DEMEAU, Anselm, Merrimack (all CX shares) |
| **Gap register** | O-45 (closed), O-46 (raised) |

---

## 1. How it surfaced

A dashboard, as usual. Enrollment for 2011–12 came back at **230 students** against a
baseline of roughly 2,100. Two other years came back at more than double it. Nothing
errored, no test failed, and the build was green end to end.

| Academic year | Headcount as published | Plausible? |
|---|---:|---|
| 2010–11 | 2,078 | ✓ |
| 2011–12 | **230** | — |
| 2012–13 | 2,122 | ✓ |
| 2016–17 | **4,516** | — |
| 2018–19 | **4,682** | — |

The shape is the diagnosis: students were not missing, they were filed under the wrong
year. 2011–12 lost about 1,900; 2016–17 and 2018–19 each gained roughly that many.

---

## 2. What was actually wrong

The academic calendar arrives as `acad_cal_rec` — 396 rows for DEMEAU, one per term.
Each carries a year (`YR`), a session (`SESS`), and an academic-year code (`ACYR`)
saying which academic year the term belongs to.

On four rows `ACYR` was impossible:

| Term | Actually runs | `ACYR` says | Should be |
|---|---|---|---|
| Spring 2011 | Jan–May 2011 | `1819` | `1011` |
| Fall 2011 | Aug–Dec 2011 | `1819` | `1112` |
| Spring 2012 | Jan–May 2012 | `1617` | `1112` |
| Spring 2013 | Jan–May 2013 | `1617` | `1213` |

**The row contradicts itself**, and that is the load-bearing detail. `YR` and `SESS`
already state when the term ran, so the correct `ACYR` is recoverable from data the
source itself sent. Hold onto that — it decides everything in §4.

The convention was confirmed before being trusted: the derivation agrees with `ACYR` on
**385 of 396 rows**, and summer trails its academic year here on **218 of 223** summer
rows.

---

## 3. The fork

Three places you could fix this. The arguments against two of them are the arguments for
the architecture.

### Not at the source

The CX archive is a Snowflake data share; we could not edit it if we wanted to. But we
would decline anyway. The source system is the institution's system of record. If the
registrar's office believes Spring 2011 belongs to 2018–19, that is a conversation to
have with them, not a value to quietly overwrite. Silently diverging from a client's
system of record is how a platform stops being reconcilable against the thing it
reports on.

### Not in Deposit

This is the one to be firm about, because it is technically trivial and genuinely
destructive. Deposit is our copy of what arrived, and its entire value is that it is
*faithful* — it is the only thing that answers "what did the share actually send us?"
Repair a value there and that question is unanswerable forever. You cannot audit a
record you have edited, and you cannot reproduce a build whose inputs you have changed.

> **Deposit answers what arrived. Deterge answers what we believe is true, and why.
> Distribute answers what we publish.** A correction that lands in the wrong layer does
> not just sit in the wrong folder — it destroys the answer that layer existed to give.

### In Deterge

Which leaves the middle layer, which is what it is for. Deterge is where we may assert
something the source did not — on the condition that we say so and carry the original
forward beside it.

| Layer | Owns the question | `ACYR` | `BEG_DATE` |
|---|---|---|---|
| **Deposit** (bronze) | What arrived | `1819` | `8015-12-21` |
| **Deterge** (silver) | What we believe, and why | `1112` *(+ `…_source` = `1819`)* | `null` *(+ `…_source` = `8015-12-21`)* |
| **Distribute** (gold) | What we publish | `1112` | `null` |

The `_source` companion column is the price of being allowed to correct anything at all.
It keeps the chain from *what arrived* to *what we published* unbroken:

```sql
-- stg_jcx__academic_calendar
case
    when trim(SESS) in ('FA', 'WI') then
        lpad(right(YR::int::varchar, 2), 2, '0')
        || lpad(right((YR + 1)::int::varchar, 2), 2, '0')
    when trim(SESS) in ('SP', 'SU') then
        lpad(right((YR - 1)::int::varchar, 2), 2, '0')
        || lpad(right(YR::int::varchar, 2), 2, '0')
    else trim(ACYR)
end                                     as academic_year_display,

ACYR                                    as academic_year_display_source,
```

---

## 4. Correcting and inventing are not the same thing

While fixing the calendar we found a second defect in the same table: one term carried a
start date in the year **8015**. It had been flowing into `dim_term` unnoticed and was
setting the dimension's published maximum term date.

Everyone's instinct is to write `2015-12-21` and move on. It is obviously a typo. That
instinct is exactly what the middle layer has to discipline, because the two cases are
not alike — and the difference is not how confident you feel.

**The test is: is the correct value recoverable from data the source sent?**

| | Corrupt `ACYR` | Year-8015 date |
|---|---|---|
| Recoverable from the source's own data? | **Yes** — `YR` + `SESS` state when the term ran | **No** — nothing in the row states the start date |
| Evidence | Convention verified on 385 of 396 rows | Suggestive only: ends `2016-01-18`, every other winter term starts in December |
| Action | **Derive** the value | **Suppress** to `null` |
| Why | Reading information already present, in another field | Writing `2015-12-21` publishes a date the source never sent, under our name |

*Suggestive is not the same as stated.* That is the whole line.

### Suppressing cost nothing — measured, not assumed

- Every affected term carries **zero enrollment**.
- `dim_date` already excluded them: it requires both dates present, and an inverted range
  matches no days regardless.
- In-term day coverage across the affected window is **26 of 29 days before and after** —
  identical.
- It fixed the one visible defect: `dim_term`'s latest term start read `8015-12-21` and
  now reads `2028-12-01`.

Being strict is cheap far more often than it feels. Price it before agonising over it.

> **If a term ever genuinely needs a repaired date**, the route is a correction seed
> recording it as a human decision — who decided, when — the way
> `seed_hold_domain_crosswalk` handles hold-code classification. What there is no route
> for is an invented value hardcoded into a model, indistinguishable from something the
> source sent.

---

## 5. Result

| | Before | After |
|---|---:|---:|
| 2011–12 | 230 | **2,105** |
| 2016–17 | 4,516 | **2,057** |
| 2018–19 | 4,682 | **2,153** |

All eighteen years now fall between 2,046 and 2,204.

| Count | |
|---:|---|
| **9** | rows corrected |
| **2** | rows protected from a wrong "fix" |
| **4** | bad dates suppressed |
| **0** | source values altered |

The two protected rows are where `YR`, not `ACYR`, turned out to be the corrupt field —
`2026 WI` begins `2025-12-01`, `2028 FA` begins `2027-09-01`. Deriving blindly from `YR`
would have replaced a correct value with a wrong one. The guard defers to the source
wherever a term's own dates contradict its year.

---

## 6. Two things that nearly got past us

### A green build is not proof the model works

Staging models are **views**. `dbt build` issues a `CREATE VIEW` and never executes the
query, so an expression that fails on real data reports **PASS** and then errors for
every reader.

```bash
# Not sufficient for a derived column — the view is created, never run:
bash scripts/run_merrimack_dev.sh build --select stg_jcx__academic_calendar   # PASS

# This is what actually verifies it:
python scripts/query_snowflake.py \
  "SELECT COUNT(academic_year_display) FROM MERRIMACK_DD_DEV.DETERGE.STG_JCX__ACADEMIC_CALENDAR"
```

The first version of this fix built green at Merrimack while the view was unreadable; it
only surfaced when `dim_term` — a table, so actually executed — selected from it.

> `COUNT(*)` is not enough either. Snowflake answers it from metadata without evaluating
> the column. Count the column itself.

### One school is not the fleet

The fix was written and checked against DEMEAU. It broke **Merrimack**, which carries a
blank filler row no other school has — a `CASE` with one `::integer` branch coerces its
`else` branch too, and `'    '::integer` fails.

DEMEAU and Anselm agreed perfectly, which was reassuring and meaningless: **DEMEAU *is*
Anselm, pseudonymised**. Merrimack was the only genuinely independent share, and it was
the one that broke. Check the source that can actually disagree with you.

---

## 7. What we left behind

The durable output was not the fix, it was the detector. Nothing validated term dates
anywhere in the project before this.

Two tests now sit on `dim_term` — `end_date_not_before_begin_date` and
`dates_within_plausible_years` — placed on the **dimension**, not the staging model,
because `dim_term` unions CX with Slate and a staging-side test would have missed the arm
holding most of the bad rows.

Both were verified against the bad shape: disabling the guard makes them fire.

> They are currently `warn`, not `error`, because of **O-46** — 78 of Merrimack's 616
> `dim_term` rows are Slate placeholder junk (`ba--bank`, `th--the`, `ya--yard`), 16 of
> them with inverted dates. Promote them to `error` when O-46 closes. A permanent warn is
> a warn people stop reading.

---

## Takeaways

1. **Fix it in the layer that owns the question.** Deposit owns "what arrived" — edit it
   and that question has no answer, forever.
2. **Publish the original next to the correction.** A `_source` column is what makes
   asserting a value legitimate rather than silent. If you would not be comfortable
   showing the pair to the client, do not make the change.
3. **Ask whether the right value is recoverable, not whether it is obvious.** Recoverable
   from the source's own data means derive. Merely obvious means suppress.
4. **Price the suppression before agonising over it.** Here it cost nothing measurable.
5. **Leave a detector, not just a patch.** Put it where the sources converge.

---

## References

- Gap register O-45 / O-46 — `ditteau_data_transform/docs/governance/data_access_policy.md` §O
- [dbt Conventions](dbt/conventions.md) · [DQ Framework](governance/dq-framework.md)
- Commits `22d612c`, `99eadf0` in `ditteau_data_transform`
