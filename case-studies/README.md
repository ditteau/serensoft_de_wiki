# Case Studies

Worked accounts of real defects on the Ditteau platform — what broke, how we found it,
what we decided, and what we would do differently. Written after the fact, from
measurements taken at the time.

These exist because the reasoning behind a fix is worth more than the fix. A commit
records what changed; an ADR records a decision we expect to stand. Neither captures the
judgement call in the middle — the moment where two defensible options were on the table
and we picked one. That is what a case study is for, and it is the part that transfers
to the next problem.

---

## Index

| Case study | What it teaches | Date |
|---|---|---|
| [The Year 8015](case-studies/year-8015-medallion-layers.md) | Where a correction belongs, and why Deposit is never the answer | 2026-09-07 |

---

## When to write one

Not every bug earns a case study. Write one when at least two of these hold:

- **A reasonable engineer would have chosen the wrong option.** If the right answer is
  obvious in hindsight *and* was obvious at the time, a commit message is enough.
- **The lesson generalises past the model it happened in.** "Snowflake's `concat_ws`
  propagates NULL" is a note for `CLAUDE.md`. "A green build is not proof the model
  works" is a case study.
- **We nearly did the wrong thing**, or did it and caught it. Near misses teach better
  than clean successes, and they are the ones nobody writes down.
- **The evidence was expensive to gather.** If it took a day of querying to establish
  what was actually true, that measurement should not have to be repeated.

If a case study produces a rule we intend to hold to, it belongs in an
[ADR](decisions/adr-template.md) as well — the case study is the story, the ADR is the
commitment.

---

## Structure

Follow the shape of the existing ones rather than a rigid template:

1. **How it surfaced** — the symptom, as first seen, with the real numbers
2. **What was actually wrong** — the diagnosis, and how it was established
3. **The fork** — the options that were genuinely on the table
4. **What we chose, and why** — including what we rejected and the argument against it
5. **What it cost, measured** — not asserted
6. **What we left behind** — the detector, the test, the note

> **Every figure carries its measurement.** A case study that quotes a number without
> saying where it came from is a story, not a record. Same standard as the gap register
> in `docs/governance/data_access_policy.md` §O.

---

## Presentation versions

Some case studies have a designed HTML companion for presenting to the team or to a
client. Where one exists it is linked from the case study page. The markdown page is the
canonical record — the HTML is a rendering of it, and the two can drift, so treat the
markdown as authoritative for figures.
