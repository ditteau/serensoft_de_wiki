# Monthly Organize Sweep

How to run the monthly staleness, duplication and task sweep across the `ditteau_*` repos and
this wiki. First run 2026-11-05; that run is the worked example throughout.

The sweep **finds and proposes**. Nothing is moved, deleted, archived or published until LVP
has seen the list and approved it. That rule has no exceptions, including for files that look
obviously dead.

---

## Contents

- [A. What the sweep is for, and what it is not](#a-what-the-sweep-is-for-and-what-it-is-not)
- [B. Never touch](#b-never-touch)
- [C. Where things live](#c-where-things-live)
- [D. The monthly run, step by step](#d-the-monthly-run-step-by-step)
- [E. Decision rules](#e-decision-rules)
- [F. Traps found on the first run](#f-traps-found-on-the-first-run)
- [G. The report script](#g-the-report-script)

---

## A. What the sweep is for, and what it is not

**For:** stale files, things duplicated across repos, dead branches, tracked junk, a working
task list drifting from reality, and Claude memory that has gone out of date.

**Not for:**
- **Archiving repositories.** Decided 2026-11-05: no repo is archived. "Archive" means files
  *inside* a repo. `ditteau_data_debut`, `ditteau_data_v1` and `ditteau_data_validate` are
  *reference repos*, kept as they are and not worked in.
- **Snowflake objects.** Stale tables, schemas, compute pools and monitors are a separate
  task. Items that need Snowflake to check are tracked in Asana with a `[Snowflake]` prefix.

---

## B. Never touch

| Item | Why |
|---|---|
| `~/ditteau_data_inbox/` | Real client data. The retention policy governs it, not tidiness. |
| `~/ditteau_data_quarantine/` | Evidence for O-46. |
| `ditteau_schema_registry/redaction_register.yml` | A control. Removing an entry re-exposes data; that is a KKM decision. |
| Governance seeds (`seed_hold_domain_crosswalk`, `seed_persona_domain_grid`, `seed_streamlit_owner_exceptions`, `seed_aggregate_register`) | Governance artifacts, not lookup tables. |
| O-register entries | Close them with evidence. Never delete one. |
| `school_setup/migrations/` | The record of SQL that was actually run. |
| Client source material (`schools/<school>/`, `source_material/`) | Provenance. |

---

## C. Where things live

| Kind | Home | Notes |
|---|---|---|
| Actionable work | Asana, *Ditteau Data Engineering* | Private, Serensoft team. **Assign only to LVP.** |
| Governance gaps | O-register, `ditteau_data_transform/docs/governance/data_access_policy.md` §O | Not copied into Asana. One description, one status. |
| Findings and research | `docs/` in the owning repo | e.g. `docs/designs/` |
| Point-in-time records | `docs/_dated/<original subfolder>/` | Accurate on their date; not maintained. `docs/checkins/` is also dated by design. |
| Decisions to keep a file | `ditteau_data_transform/scripts/organize_sweep_keep.txt` | So the report does not raise it again. |
| Working notes | `ditteau_data_transform/TODO.md` | **Git-ignored and not backed up.** Holds only pointers and the Watch list. Never tasks. |

### How Asana and the repos relate

The repos do not know about Asana, and nothing syncs. Asana holds **to-do state**; the repos
hold **outcomes**. Two conventions keep them traceable:

1. A commit made for a task carries `Asana: <task id>` in its message.
2. Closing a task: paste the commit hash or link into its final comment.

A task can carry out part of an O-entry's fix, but **closing the task does not close the
O-entry.** Resolve that in the register, with evidence and the databases it was measured in.

Name prefixes stand in for sections, because the Asana connector cannot create sections:
`[Blocked]`, `[Snowflake]`, `[Ask RDT]`, `[Ask WDT]`, `[Watch]`. A `[Watch]` task with a due
date is how to set a reminder; the Google Calendar connector has no write access.

---

## D. The monthly run, step by step

Each step produces a list, LVP approves it, then the change is made and committed. Commit and
push per repo; the `hooks/pre-push` redaction check runs on transform, infra and the registry.

### Step 0 — Safety net

1. Commit or stash in-progress work in every repo. **Ask before committing someone's
   uncommitted work.** It may be mid-edit.
2. Tag every git repo locally: `git tag -a sweep-YYYY-MM -m "Pre-sweep baseline"`. Do not push
   the tags. The report counts commits since the newest `sweep-*` tag.

### Step 1 — Run the report

```bash
/Users/laurievanpelt/testenv/bin/python3 \
  ~/ditteau_data_transform/scripts/organize_sweep.py --out ~/sweep_YYYY-MM.md
```

About ten seconds. Exit 1 means the positive control failed: **section C is then unreliable
and must not be acted on.** See [section G](#g-the-report-script).

### Step 2 — Branches and working trees (report sections A, B)

- **Merged branches:** delete locally and on GitHub.
- **Unmerged branches:** check whether another branch contains them before calling them
  unmerged work. On 2026-11-05 all seven foresight branches were one linear stack, and the
  fix was a fast-forward of `main`.
- **Tracked junk** (`.venv/`, `.DS_Store`, `*.backup`, empty files): `git rm --cached`, add
  to `.gitignore`, and check that scripts don't depend on it being on disk. Two registry
  scripts activate `.venv` when present.
- **Uncommitted work:** ask.

### Step 3 — Stale files (report sections C, D)

For each candidate, **read the file** before proposing anything. Then propose one of:

| Outcome | When |
|---|---|
| **Delete** | Superseded: a newer version exists, or it was replaced by a tool. Git history is the archive; list deleted files in the commit message. |
| **Move to `docs/_dated/`** | A point-in-time record worth keeping. Fix every inbound link in the same commit, and check outbound relative links in the moved file. |
| **Link it** | Current but orphaned. Add it to the README, the sidebar or an index. |
| **Keep** | Used by convention, or reviewed and still needed. Add it to the keep-list with the reason and date. |

⚠️ A file with zero references can still be dangerous. On 2026-11-05 that described
`create_and_load_test_data.sh`, which could still load PowerFAIDS and Slate Faker data into
Merrimack. So read the file as well as counting references.

### Step 4 — Duplicates (report section E, plus a by-hand look)

The report catches only byte-identical copies. Near-duplicates need a look. Places they
turned up on the first run:

- Docs copied between repos with small edits: transform `docs/forecasting/` vs foresight
  `library/`. The foresight copies were the verified ones; their `staging_manifest.md`
  records provenance.
- Generated or exported pages left behind in the wiki: `kpi_library.html`, three months
  older than the copy in `ditteau_kpi_library`.
- Overviews of one repo kept in several places. CLAUDE.md is the maintained overview.
- `.docx` exports older than their `.md` source. Regenerate them with
  `tools/md_to_docx.py`; never edit a `.docx` by hand.

For each pair, decide which copy is the one to keep from **what cites it**, not from which is
newer.

### Step 5 — Tasks

1. **Asana, completed since the last sweep:** each task should have a final comment with a
   commit or other evidence. Ask LVP about any that do not.
2. **Asana, open and untouched for 60 days:** list them for LVP: still wanted, blocked, or
   drop.
3. **`TODO.md`:** report section F counts open `- [ ]` items. The answer should be 0; move any
   to Asana or the O-register.
4. **The Watch list:** remove items whose date has passed, once confirmed.

### Step 6 — Claude-side context (report section F)

- **Memory not touched in 90 days:** re-verify each claim against the repo, then update or
  remove the memory. On the first run, 8 of 9 "missing" `dim_student` columns turned out to
  exist.
- **Memory that repeats CLAUDE.md:** remove it. CLAUDE.md is the one copy.
- **Dangling `[[links]]`:** repoint them to where the fact now lives.
- **Claude project folders for renamed repos:** empty ones can go.

### Step 7 — Context pack

Report section G gives the age of `~/ditteau_context_regen/`, the claude.ai project context
pack, for information only. **It is regenerated only when LVP initiates it** (decided
2026-11-05), never as part of the sweep. It is not in git.

### Step 8 — Record

Update the `monthly-organize-sweep` Claude memory with the run's date, commits and anything
left open. Add to the keep-list every file reviewed and kept.

---

## E. Decision rules

- **Never act on age alone.** Old files are often the most-cited ones.
- **One owner per fact.** When two copies exist, keep the one that is cited and turn the other
  into a pointer, or delete it. Never maintain both.
- **Governance gaps go only to the O-register.** A second description of a gap drifts from
  the first, and this project has been bitten by that more than once.
- **A tag is not a backup of uncommitted work.** It covers commits only.
- **Outward-facing actions need a fresh yes**: pushing, deleting remote branches, creating
  Asana tasks, anything in a shared workspace. Earlier approval doesn't carry over.

---

## F. Traps found on the first run

| Trap | What happened | Guard |
|---|---|---|
| A check that cannot match looks like a check that found nothing | The first scanner missed `.css`, `.ps1`, `.out` and extensionless files, and reported them as unreferenced | Token matching plus a positive control (section G) |
| A filename-shaped variable | A grep for `standard_columns` "found" the file `jcx_standard` in use. It was a variable name. | Match whole tokens only |
| zsh globs | `ls ~/.claude/projects/*ditteau*(N)` and folder names starting with `-` both broke shell loops | `ls -- "$p"`; check that the loop ran as many times as expected (CLAUDE.md gotcha 9) |
| The tool's own blind spot | `find_ssn_columns.py` silently runs without `jcxssn.txt`, which does not exist | Recorded under O-54 |
| A commit makes an idle repo look active | Committing leftover work to `v1` put it in the "active" set | `REFERENCE_REPOS` in the script |

---

## G. The report script

`ditteau_data_transform/scripts/organize_sweep.py` is read-only. Its header documents each
section.

- **Matching (section C).** A file is referenced when its basename appears as a whole token
  in any text file across the repos, `ditteau_context_regen`, the Claude memory and commands,
  and `~/Library/LaunchAgents`. Names containing spaces get a substring check.
- **Positive control.** Before trusting any zero, the script confirms it finds references to
  `data_access_policy.md`, `query_snowflake.py` and `redaction_register.yml`. On 2026-11-05
  it found 57, 35 and 22.
- **Excluded by convention:** hooks, `.claude/`, Streamlit config, dotfiles, READMEs, dbt
  project files, migrations, client source material, and anything already in `_dated/` or
  `checkins/`. A `.docx` beside a `.md` of the same name counts as an export.
- **Duplicated by design** (section E): per-school seeds, per-app Streamlit config (a
  `streamlit/CLAUDE.md` rule) and `LICENSE`.
- **Not covered:** Asana (use the connector, Step 5), near-duplicates (Step 4) and Snowflake
  (a separate task).
