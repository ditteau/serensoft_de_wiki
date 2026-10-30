# Jenzabar One — Extraction

How to get data out of a Jenzabar One (MSSQL) tenant and into the Deposit layer.

Derived from the Merrimack engagement, where direct access was first obtained
2026-10-29. Everything marked **measured** was observed on a live system; the rest is
design. Site-specific values — server address, login, keychain service — are
deliberately **not** in this file; see *Site specifics* at the end.

---

## Overview

Jenzabar CX arrives at every tenant through a Snowflake organization share. **Jenzabar
One has no share and no vendor-supplied pipeline.** Extraction is hand-run SQL against
the tenant's SQL Server, which makes the access path the first thing to establish and
usually the thing that decides the design.

Two extractors exist. They produce the same output contract:

| Tool | Runs on | Use when |
|---|---|---|
| `ditteau_data_infra/docs/j1_extraction/extract_j1.py` | Any machine with a route to port 1433 | **Preferred.** Python, tested, resumable |
| `ditteau_data_infra/docs/j1_extraction/export_j1.ps1` | A Windows host inside the tenant's network | The only option when SQL is reachable solely from a jump host |

---

## Step 1 — Establish the access path before anything else

Three topologies turn up, and they permit very different things. Ask which one you
have rather than assuming.

| Topology | File writes | Shell | Can reach Snowflake | Verdict |
|---|---|---|---|---|
| **Direct SQL** — a route to `:1433` from your own machine | local | yes | yes, but see Step 2 | ✅ Use `extract_j1.py` |
| **RemoteApp** — a published SSMS window, no desktop | to the remote host only | no | usually no | ⚠️ Clipboard is the only channel |
| **Full remote desktop** on a host in their network | to that host | yes | depends | `export_j1.ps1` on that host |

⚠️ **"They gave me SSMS" tells you almost nothing.** It proves a network path and
credentials exist, from *somewhere*. It does not tell you whether that somewhere is
your laptop or a hosted RemoteApp, whether files can leave it, or whether it can reach
the internet. **Measured at Merrimack:** SSMS arrived as a RemoteApp through an RD Web
portal — no desktop, no shell, no browser, no redirected drives in the save dialog.
Files written there could not leave the host.

⚠️ **The RemoteApp being a dead end does not mean direct SQL is.** At Merrimack, port
1433 answered from the laptop over the tenant VPN even though the RemoteApp was
sealed. Test the direct route before building anything around a jump host:

```bash
# get the address the server sees, from any SSMS session
#   SELECT CONNECTIONPROPERTY('local_net_address'), CONNECTIONPROPERTY('local_tcp_port');
nc -z -w 6 <server_ip> 1433 && echo reachable
```

Windows/AD logins work from macOS: FreeTDS negotiates NTLM with a `DOMAIN\user`
credential, so `pymssql` connects without any Kerberos setup. **Measured.**

---

## ⚠️ Step 2 — Two VPNs may be mutually exclusive

Where the tenant's database needs their VPN and Snowflake needs ours, **check whether
both can be up at once. Often they cannot** — starting the second tears down the
first's tunnel, or a full-tunnel client swallows the split-tunnel routes.

```bash
# the client's UI lies about this; only an interface with an inet address is up
for i in $(ifconfig -l | tr ' ' '\n' | grep utun); do
  a=$(ifconfig $i 2>/dev/null | awk '/inet /{print $2" --> "$4}')
  [ -n "$a" ] && echo "  $i  $a"
done
curl -s --max-time 8 https://api.ipify.org; echo
```

**Measured at Merrimack:** exclusive. Tenant VPN reaches J1 and Snowflake refuses the
egress IP; our VPN reaches Snowflake and kills the route to J1.

**Design consequence: extract everything, switch, load everything.** One switch per
batch, never per table. Do not design a pipeline that needs both at once — it cannot
run unattended until the network question is solved.

**Two durable fixes, both someone else's to make:**
○ Add the egress IP to the Snowflake network policy. ⚠️ Weak if that egress is a
  residential address — it will change. `ALTER NETWORK POLICY ... SET ALLOWED_IP_LIST`
  **replaces** the list, so existing entries must be carried over.
○ Ask whether the tenant VPN can be full-tunnel, which makes the egress a stable
  institutional address and is the better thing to allowlist.

---

## ⚠️ Step 3 — Verify permissions before believing any absence

**`INFORMATION_SCHEMA` and `sys.tables` are permission-filtered in SQL Server. They
return fewer rows for an unauthorised login and do not error.**

**Measured at Merrimack:** all three J1 databases reported exactly **6 tables** — the
PowerBuilder catalog tables plus a login table, the ones `public` holds a grant on. It
read as three empty databases. One of them held 2,739.

This is the same class as gotcha 6 in `ditteau_data_transform/CLAUDE.md`, where
`INFORMATION_SCHEMA` under-reports for a Snowflake role. **An absence reads exactly
like a fact.**

```sql
USE <database>;
SELECT permission_name FROM fn_my_permissions(NULL, 'DATABASE') ORDER BY 1;
```

`CONNECT` plus `VIEW ANY COLUMN ENCRYPTION KEY DEFINITION` and `VIEW ANY COLUMN MASTER
KEY DEFINITION` means **default public access and no read rights** — those two come
free with any login.

⚠️ A direct read is a better probe than metadata. `SELECT COUNT(*) FROM dbo.<table>`
returning *"Msg 229: The SELECT permission was denied on the object '<name>'"* **names
the object**, which proves it exists. Metadata would just omit it.

### Ask for the right grant

`db_datareader` on the database. Nothing else, and nothing on individual tables.

⚠️ **Check what actually arrived.** At Merrimack the grant came back as `CONTROL` —
`db_owner`-equivalent, including INSERT, UPDATE, DELETE, ALTER and TAKE OWNERSHIP on a
live SIS. Ask for it to be reduced. The risk is not malice; it is that SSMS executes a
typo against production with no confirmation and no undo.

---

## Step 4 — Extract

```bash
cd ~/ditteau_data_infra/docs/j1_extraction

./extract_j1.py --tables-from <list>.txt --dry-run          # row counts only
./extract_j1.py --tables-from <list>.txt --max-rows 100000 --out <inbox>
./extract_j1.py --tables <table,table> --out <inbox>
```

○ Columns come from `ditteau_schema_registry` in **ordinal order**. That is the load
  contract — `deposit_loader` maps positionally as `$1..$N` and never reads headers, so
  a column out of place yields wrong values in correctly typed columns, with no error.
○ Writes `.csv.partial` and renames on completion, so an interrupted run leaves a
  partial rather than a short CSV. **A truncated CSV preflights clean and loads wrong** —
  this is the failure the rename exists to prevent.
○ Resumable: existing CSVs are skipped unless `--overwrite`.
○ Empty tables are skipped. ⚠️ **Expect most of them to be empty** — 127 of 223 at
  Merrimack, because the catalog covers modules the tenant never licensed.
○ `--max-rows` defers the heavy tables so a first pass delivers the reference data in
  minutes rather than hours.

**Throughput, measured:** 87 tables / 587k rows in ~3 min. 557,815 rows × 51 cols in
23s. 3,428,920 rows × 74 cols in 4m23s, producing a 1.04 GB CSV.

### When only the clipboard is available

A RemoteApp still gives a working clipboard, which is enough for config and lookup
tables.

○ **Right-click grid → Save Results As** produces RFC-4180-correct CSV — quotes
  embedded commas, doubles embedded quotes. **Measured.** *Results to File* produces
  space-padded text instead; avoid it.
○ ⚠️ Column headers are **off by default**: Tools → Options → Query Results → SQL
  Server → Results to Grid → "Include column headers", then open a **new** query window.
○ ⚠️ Nulls export as the literal word `NULL`. Convert to empty before loading. The
  Deposit file format's `NULL_IF` coerces `'NULL'` back to a null, which also silently
  eats a genuine text value of "NULL" in a free-text column.
○ `GO` separates batches so one bad statement does not kill the rest; `F5` runs the
  whole window unless text is selected.

---

## Step 5 — Load

```bash
cd ~/ditteau_data_infra/deposit_loader
python deposit_loader.py --school <CODE> --env DEV --warehouse <WAREHOUSE> \
    load-all jenzabar_one <inbox> --truncate
```

⚠️ **`--warehouse` may be required.** If the connection in `~/.snowflake/config.toml`
carries no warehouse, the load fails with *"No active warehouse selected"* **after**
writing a load-history row.

○ Filenames must match registry table keys, lowercase. Never rename an extract.
○ ⚠️ `COPY INTO` runs `ON_ERROR='CONTINUE'`, so a table can drop rows and still
  complete. Read the `load-all` summary's rejected-row counts; do not assume a clean
  exit means a clean load.
○ `--from-stage` exists for when the extract ran on a machine that is not the loader's:
  the producer `PUT`s to the internal stage and the loader COPYs from there. ⚠️ That
  path cannot run the ragged-row scan — prefer a local load when the file is local.

---

## ⚠️ Step 6 — Data gotchas

All measured at Merrimack 2026-10-29 and all silent when wrong.

| | |
|---|---|
| **MSSQL `timestamp` is ROWVERSION** | Eight bytes of binary, not a datetime. `approwversion` is one on nearly every table — ~1,853 columns in the Merrimack catalog — and the registry maps it to `VARCHAR`. Naive stringification yields `b'\x00…'` in Python or `System.Byte[]` in PowerShell. **Both load without complaint and are worthless.** Hex-encode it |
| **`char(n)` padding is data** | Values arrive space-padded. Joins need `TRIM()`. The Deposit file format sets `TRIM_SPACE=TRUE`, so it is stripped on arrival but present in the CSV |
| **Preflight row counts over-report** | `compare_headers_to_registry` counts lines, not CSV records; a quoted field containing a newline inflates it. 3,428,920 records across 3,429,332 lines. Harmless — do not reconcile against it |
| **Audit columns carry impossible dates** | A `job_time` three years in the future was observed. Never use `MAX(job_time)` as an incremental watermark; `approwversion` is maintained by the engine and is the reliable marker |
| **The source is live** | ~1,100 rows/day on the transaction table. A row count taken minutes before an extract will not match what the extract writes. Expected, not an error |

---

## ⚠️ Step 6b — A one-hop FK closure lands the header and misses the fact

`generate_extract_sql.py --owner <OWNER> --fk-closure` follows foreign keys
**one hop** from the owner-tagged seed set. That is one hop short of a
`header → detail → amount` chain, which is the normal shape of a budgeting or
document module.

**Measured at Merrimack 2026-10-30**, after a 223-table pull had already been
loaded, staged and reviewed:

| Table | `data_owner` | Distance | Pulled? |
|---|---|---|---|
| `masterbudget` | `FINANCE` | seed | ✅ |
| `lineitem` | `UNKNOWN` | 1 hop | ✅ |
| `lineitemamount` | `UNKNOWN` | **2 hops** | ❌ |

`lineitem` arrived with **76,760 rows and no amount column.** The amounts, the
periodised amounts, the line-item code decode and the GL-account crosswalk were
all two hops out and silently absent.

⚠️ **The pull looked complete: 223 tables, 90 non-empty, zero failures.** The
single most-requested report could not be built from any of it, and nothing in
the extract, the load or the staging build said so. It was found by reading a
column list.

**Why it is structural rather than a site quirk:** only the header carries the
owner tag. The detail and amount tables read `data_owner: UNKNOWN` in every
tenant, because the registry's owner field is pattern-derived (see *Registry
staleness* below). One hop reaches the detail and stops.

**Two ways to catch it:**

○ **Run the closure twice** — feed pass one's table list back as the seed:
  `--tables-from pass1_tables.txt --fk-closure --list-only`.
○ **Test for orphaned headers before loading.** A table that arrives with rows,
  carries `*id` columns pointing at type or code tables, and has **no numeric
  column** is a header whose detail is missing. That test catches it on sight.

⚠️ **Do not respond by pulling everything.** At Merrimack the one-hop frontier
beyond pass one was **497 registry tables**. Widening without a reason trades a
known gap for an unreviewed 2,591-table pull.

⚠️ Separately: a table that *was* requested and did not arrive was **empty**,
not missed — the extractor skips empty tables. Three Merrimack budget tables
fell in that category, which is itself a finding about what the site uses.

---

## Step 7 — Registry staleness

The registry is generated from a catalog export, which is a point-in-time snapshot of a
working database.

⚠️ **Expect drift, and expect it to be invisible.** At Merrimack the export was four
months old: 2,739 live tables against 2,679 in the registry. The extra 60 were mostly
ticket-numbered backups and vendor help-desk working tables, **but not all** — a
site-built charge table with real data in it had no registry entry, so no extract SQL
could be generated for it.

Drift in that direction is safe: a table absent from the registry is simply never
pulled. The failure is omission, not corruption.

⚠️ **Site-built tables are not a UDF pattern.** The genuine user-defined-field
convention is `*_udf` — 83 such tables at Merrimack, named columns joining 1:1 to a base
table. A site prefix (`mcm_*` there) is something else: mostly standalone applications
and integrations, occasionally a **structurally identical fork** of a vendor table.
A fork is not an extension — joining it to its namesake pairs two competing
definitions rather than adding fields. Confirm which the tenant's reports actually read
before modelling either.

---

## Site specifics

Server address, database name, login and keychain service are **not** recorded here.
For Merrimack they live in `~/.claude/commands/merrimack-j1.md` on LVP's machine,
alongside the measured row counts and the current open questions.

Engagement context: `ditteau_data_transform/docs/engagement/merrimack_finance/`
Platform plan: `ditteau_data_infra/docs/j1_extraction/README.md`
