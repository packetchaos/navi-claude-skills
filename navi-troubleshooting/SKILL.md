---
name: navi-troubleshooting
description: >
  Troubleshooting skill for Tenable navi CLI. Use for ANY request involving navi
  errors, unexpected results, or "it's not working" symptoms. Covers: "zero
  chunks" on update commands, sqlite3 database locked errors, slow tagging
  performance, commands returning empty results, missing assets, database
  errors after upgrading navi, schema mismatches, MCP tool-call timeouts on
  long-running syncs. Also covers the SQL index pattern for accelerating
  repeat tagging workloads and the purpose-built workload alternative. Trigger
  on: "navi isn't working", "error", "zero chunks", "db locked", "database is
  locked", "sqlite3.OperationalError", "empty results", "no data returned",
  "missing assets", "tagging is slow", "after upgrade", "schema mismatch",
  "timeout", "tool call timed out", "why doesn't this work", "what went
  wrong", "fix my navi". Companion: navi-core (setup, schema — preventive
  context for most issues here).
---

# Navi Troubleshooting — Common Issues & Fixes

This skill is for reactive fix-it workflows — when something has already
gone wrong. For preventive / install-time context, see navi-core.

Every symptom below is presented the same way: how it looks, what's
actually causing it, and how to fix it.

---

## "Zero chunks" on update commands

```
Downloading chunks: 0 chunks
```

A "0 chunks" result from `navi config update` or an export means Tenable
returned no data for your query window. It is **not** the same as an error,
and it's **not always** an API key problem — though that's one possibility
among several. Diagnose in this order, cheapest check first.

### 1. No new data in the window (most common)

`navi config update assets` and `navi config update vulns` default to a
short lookback. If no scans have produced new findings in that window,
there's nothing to export. Re-run with a wider window:

```bash
navi config update assets --days 365
navi config update vulns --days 365
```

If the chunk count jumps from 0 to non-zero, confirmed: the previous
window was too short.

A wide window also acts as a **backfill mechanism** when scanners were
broken in the recent past. If `--days 365` returns data but `--days 7`
doesn't, you have a recent scanner outage — the older data was already
in Tenable, the new data isn't being collected. Fix scanner health
(item 2), then re-run with a wider window once to catch up.

### 2. Scanners are failing

Wide window still returns zero. Empty exports with healthy scanners are
rare; empty exports with broken scanners are common. Check from the
Tenable UI (Scans → Settings → Scanners) for offline scanners or
credential-failed scans, or query navi.db directly:

```sql
SELECT scanner_name, last_modified, status FROM scanners WHERE status != 'on';
```

Fix scanner credentials or connectivity in Tenable before re-running
the sync.

### 3. API key permissions

Wide window, healthy scanners, still zero. The API key used by navi may
lack access to the scope being queried.

- Verify the API key has access to the assets/vulns you expect.
- Keys scoped to a subset of assets will only download that subset —
  expected and useful for scoped workloads, but surprising when you
  expected full coverage.
- If you need full environment coverage, ensure the key has permissions
  on ALL assets in Tenable One.
- Check the key is not expired or revoked in Tenable settings.

Under navi-mcp, key permissions are set out-of-band by your operator.
Claude cannot check them directly; verify in the Tenable platform.

### 4. Tag-restricted access groups

Less common: the API key is technically valid but scoped to a tag-based
access group that excludes all current assets. Only check this once
items 1–3 are ruled out.

### How to tell which one you're hitting

- A **401** anywhere in this chain means item 3 specifically.
- A **0-chunks response with HTTP 200** means items 1, 2, or 4.

**Prevention**: see navi-core's "API key permissions matter" subsection
under standalone installation, and treat scanner-health monitoring as a
precondition to expecting data freshness. A key scoped too narrowly at
install time causes item 3 on every subsequent update; broken scanners
cause item 2 silently for weeks.

---

## DB locks / database locked error

```
sqlite3.OperationalError: database is locked
```

**Root cause**: almost always **disk speed** — slow spinning disk or high
disk latency causes write operations to queue up and lock. Less commonly,
insufficient RAM (under 4GB) creates a processing backlog that also
causes locks.

There is also a less-obvious cause specific to navi-mcp: a hung subprocess
from a previously timed-out MCP tool call still holding navi.db open.
See "Long-running operations and MCP timeouts" below if you see this
error right after a tool call returned a 4-minute timeout.

**Resolution**: reduce the thread count on the full sync CLI command:

```bash
# Default is 10 threads. Drop to 1 to resolve disk-speed locks.
navi config update full --threads 1

# Thread range: 1 to 20. Scale up only on fast SSD storage.
navi config update full --threads 5   # moderate
navi config update full --threads 20  # max — SSD only
```

If running on a VM with slow disk I/O or a shared NAS, `--threads 1` is
the right default.

**Thread count on targeted updates**: `navi_config_update` tool calls use
the server's configured default; if locks appear during targeted
refreshes, your operator can lower the default in the navi-mcp server
config.

**Prevention**: on known-slow storage (shared NAS, VM with contention,
HDD), start with `--threads 1` before scaling up. Faster to reduce later
than to recover a locked DB mid-sync.

---

## Slow tagging (plugin-based tags taking a long time)

Almost always missing indexes on the `vulns` table.

**Primary fix — `navi config optimize`** (navi 8.5.31+). Builds a curated set of
indexes against `navi.db` in seconds — the fastest, most comprehensive fix. Run
at the terminal:

```bash
navi config optimize
```

Key behavior:

- **Run it AFTER a sync** — against an empty navi.db it prints "no such table:
  main.vulns" for each index and exits (benign; sync first, then re-run).
- **`navi config update full` does NOT build these indexes**; the targeted
  `navi_config_update(kind="vulns")` tool DOES — so incremental MCP refreshes
  keep them current.
- **Indexes are dropped when navi.db is rebuilt** — re-run `optimize` after any
  post-upgrade recovery.

**Older navi (< 8.5.31), custom indexes, or a repeated specialised workload?**
The manual `CREATE INDEX` fallback (Option B) and the purpose-built-workload
approach (Option C) — plus the full list of the seven indexes `optimize` builds
and what each accelerates — are in **`references/slow-tagging.md`**
(`navi://skill/troubleshooting/slow-tagging`).

---

## Commands returning no results / unexpected behaviour

### First check under navi-mcp

1. **Is navi.db populated?** Run:

   `navi_explore_query(sql="SELECT MAX(last_found) FROM vulns;")`

   NULL or very old means the foundational `navi config update full`
   hasn't been run at the terminal. See navi-mcp's "Too heavy for a tool
   call" section for why this command is CLI-only.

2. **Check `navi://workdir`** to see where navi.db is and whether
   write-gate is enabled.

3. **API keys may not be set, or may be scoped too narrowly.** Keys are
   set out-of-band by the operator; verify with them. Keys scoped to a
   subset of assets will silently return partial data — this is the most
   common cause of "missing data" questions.

### First check standalone

Have API keys been set?

```bash
navi config keys --a <ACCESS_KEY> --s <SECRET_KEY>
```

Navi will not error loudly if keys are missing — it may simply return
empty results. Always check this first before any other troubleshooting.

### Common root causes for empty results

| Symptom | Most likely cause |
|---|---|
| Every command returns empty | Keys not set, or navi.db empty |
| Some assets missing, others present | Key scope narrower than expected |
| Data older than you expect | No recent `config update full` |
| Tagging commands tag zero assets | Stale agents table (run `navi_config_update(kind="agents")`) |
| Agent-based tags return nothing | Agents kind NOT included in `config update full` — run separately |

---

## Database errors after upgrading navi

After any `pip install --upgrade navi-hostio` or new container build, the
existing navi.db will have a schema mismatch with the new navi version.

**Standalone recovery:**

```bash
rm navi.db
navi config keys --a <ACCESS_KEY> --s <SECRET_KEY>
navi config update full
```

**Under navi-mcp recovery:** this is a multi-channel operation because
Claude can't do all of it:

1. **You (at the CLI):** delete the old database. Use `navi://workdir`
   to confirm the path.

   ```bash
   rm navi.db
   ```

2. **Your operator:** re-enter API keys into the navi-mcp server
   configuration and restart it. Keys are stored in navi.db and are lost
   when it's deleted.

3. **You (at the CLI):** re-sync foundational data.

   ```bash
   navi config update full
   ```

4. **Back to Claude:** once navi.db exists again and is populated,
   resume MCP workflows.

This is expected behaviour, not a bug. Store API keys securely outside
of navi.db (password manager, environment variables) so re-entering them
after upgrades is quick.

**Prevention**: always expect to re-sync after a navi upgrade. Treat
version upgrades as a deliberate maintenance window, not a hot-patch.

---

## `navi config optimize` errors with "no such table: main.vulns"

```
DB ERROR:
The SQL statement: CREATE INDEX idx_plugin ON vulns (plugin_id);
Created the following error: no such table: main.vulns
```

**Cause**: `navi config optimize` ran against an empty navi.db. The
command tries to build indexes against the `vulns` table, but `vulns`
doesn't exist until at least one sync has populated it.

**Resolution**: run a vulns sync first, then re-run optimize.

```bash
navi config update full     # or navi config update vulns
navi config optimize
```

**Why this happens**: optimize is independent of `navi config update`.
You can run it in any order, but it's only meaningful after data exists.
The error output looks alarming because navi prints a separate failure
line per attempted index, but no damage is done — it just doesn't create
anything. Running again after the sync completes works.

---

## Agent-based tagging returns zero matches

Agent tagging uses the `group`, `missed`, and `byadgroup` selectors in
`navi_enrich_tag`. If those tags come back with zero results:

**Cause**: the `agents` table is stale or empty. `agents` data is NOT
included in `navi config update full` — it must be refreshed explicitly.

**Fix**:

`navi_config_update(kind="agents")`

Then re-run the tagging command.

See navi-enrich for the full agent-tagging prerequisite note.

---

## Long-running operations and MCP timeouts

`navi_config_update` (and `navi_export`) shell out to Tenable export
endpoints. On a large tenant these can run for tens of minutes to hours —
well past the MCP host's tool-call cap (~4 minutes in Claude Desktop), which
is enforced by the host app, not by navi or navi-mcp, with no model-side flag
to extend it.

**With the corrected server (call-budget aware):** navi-mcp sets its subprocess
timeout *below* the host ceiling (~220s) and returns a clean error naming the
exact CLI command to run — e.g. *"exceeded the 220s MCP call budget … run it at
your terminal: navi config update vulns --threads 1"*. That's the expected
behavior now, and it kills its own subprocess, so the DB-lock chain below is far
less likely.

**On an older/uncorrected server**, you instead get the opaque host timeout
(below) while the Tenable export keeps running server-side and the local navi
subprocess stays attached to `navi.db`:

```
No result received from the Claude Desktop app after waiting 4 minutes.
The local MCP server providing this tool may be unresponsive...
```

A successful `assets` call immediately followed by a hanging `vulns` call —
same keys — is the classic signature: it's **not** auth, it's duration.

Two follow-on consequences (mainly on the uncorrected server) worth knowing
before you re-fire anything:

- A second MCP call hitting `navi.db` right after a timeout may return
  "database is locked" — almost always the prior subprocess still attached,
  not a real DB problem. (See "DB locks" above.)
- Re-running the same export back-to-back can collide with the export still
  in flight on Tenable's side.

### Triage

1. **Don't blindly re-fire the tool.** Mashing retry causes the DB-lock
   chain above and risks a second collision.
2. **Check whether the export already finished** — now available as tools (no
   longer CLI-only):

   ```
   navi_explore_info(subcommand="exports")
   navi_explore_api(url="/vulns/export/<EXPORT_UUID>/status")
   navi_explore_api(url="/assets/export/<EXPORT_UUID>/status")
   ```

   The `chunks_available` field tells you how many chunks are ready. If state
   is `FINISHED`, the data may already be local — confirm with
   `navi_explore_query(sql="SELECT MAX(last_found) FROM vulns;")` or a row
   count. If recent data is present, you don't need to re-run at all.
   (CLI equivalents: `navi explore info exports` / `navi explore api
   '/vulns/export/<EXPORT_UUID>/status'`.)

3. **Run the long sync on the CLI**, where it can take as long as it
   needs. Throttle threads if the disk is slow:

   ```bash
   navi config update vulns --threads 1
   navi config update assets
   ```

4. **If you must stay in MCP**, shrink the job under the timeout with a
   lookback window. This is the practical in-MCP lever that keeps a
   sync inside the timeout:

   ```
   navi_config_update(kind="vulns", days=7)
   ```

5. **If navi-mcp is unresponsive afterward**, restart the local MCP
   server before issuing further navi tool calls.

### Rule of thumb

Through MCP: incremental refreshes you're confident will finish in a few
minutes (small/scoped tenants, short `days=N` windows). On the CLI: any
first-time pull, full-history refresh, or anything you expect to run
long. This is the same reasoning that keeps `navi config update full`
off the MCP surface — see navi-mcp's "Too heavy for a tool call".

---

## Summary — Troubleshooting Quick Reference

A one-row-per-symptom index for fast lookup. Full detail in the sections
above.

| Symptom | Most likely cause | Fix |
|---|---|---|
| "Zero chunks" on update | Empty window, broken scanners, or key scope — in that order | Wider window first (`--days 365`), then check scanner health, then API key. See "Zero chunks" |
| DB locked error | Slow disk | `--threads 1` on full sync |
| DB locked + low RAM | Under 4GB RAM | `--threads 1` + close other apps |
| DB locked right after an MCP timeout | Prior subprocess still attached | See "Long-running operations and MCP timeouts" |
| Tagging very slow | Indexes missing on `vulns` | `navi config optimize` (8.5.31+) |
| Tagging slow on older navi | No indexes, navi < 8.5.31 | Manual SQL indexes via `navi_explore_query` |
| `optimize` errors with "no such table" | Empty navi.db | Run `navi config update full` first, then re-run optimize |
| No results from any command (MCP) | navi.db empty or keys out-of-band | Run `navi config update full` at CLI; verify with operator |
| No results from any command (standalone) | Keys not set | `navi config keys --a ... --s ...` |
| DB errors after upgrade | Schema mismatch | `rm navi.db` + re-keys + `update full` + `optimize` |
| Missing assets | Key scoped to subset | Check key permissions in Tenable One |
| Agent tags return zero | Stale agents table | `navi_config_update(kind="agents")` |
| `navi_config_update` exceeds the call budget (~220s) | Sync longer than the budget; corrected server returns a clean error naming the CLI command (older server: opaque ~4-min host timeout, export keeps running) | Run on CLI: `navi config update vulns --threads 1`; or shrink with `days=N`. See "Long-running operations and MCP timeouts" |

---

## Cross-references

- **navi-core** — setup, schema reference, API key permissions context
  (preventive guidance), the multi-workload pattern, "Long syncs must
  run on the CLI" warning
- **navi-mcp** — why `navi config update full` is CLI-only, write-gate
  conventions, data freshness check
- **navi-enrich** — agent-tagging prerequisites, tag UUID preservation
  (the other common "my tags are broken" source)
- **navi-export** — export workflows; export-status polling is now tool-driven
  via `navi_explore_api(url="/vulns/export/<UUID>/status")` (CLI fallback: `navi explore api '...'`)
