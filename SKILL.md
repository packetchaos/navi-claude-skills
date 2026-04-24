---
name: navi-troubleshooting
description: >
  Troubleshooting skill for Tenable navi CLI. Use for ANY request involving navi
  errors, unexpected results, or "it's not working" symptoms. Covers: "zero
  chunks" on update commands, sqlite3 database locked errors, slow tagging
  performance, commands returning empty results, missing assets, database
  errors after upgrading navi, schema mismatches. Also covers the SQL index
  pattern for accelerating repeat tagging workloads and the purpose-built
  workload alternative. Trigger on: "navi isn't working", "error", "zero
  chunks", "db locked", "database is locked", "sqlite3.OperationalError",
  "empty results", "no data returned", "missing assets", "tagging is slow",
  "after upgrade", "schema mismatch", "why doesn't this work", "what went
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

**Cause**: API key has no permission to the requested data.

**Resolution**:

- Verify the API key has access to the assets/vulns you expect.
- Keys scoped to a subset of assets will only download that subset — this
  is expected and useful for scoped workloads, but surprising when you
  were expecting full coverage.
- If you need full environment coverage, ensure the key has permissions
  on ALL assets in Tenable One.
- Check the key is not expired or revoked in Tenable settings.

Under navi-mcp, key permissions are set out-of-band by your operator.
Claude cannot check them directly; verify in the Tenable platform.

**Prevention**: see navi-core's "API key permissions matter" subsection
under standalone installation. A key scoped too narrowly at install time
causes this symptom on every subsequent update.

---

## DB locks / database locked error

```
sqlite3.OperationalError: database is locked
```

**Root cause**: almost always **disk speed** — slow spinning disk or high
disk latency causes write operations to queue up and lock. Less commonly,
insufficient RAM (under 4GB) creates a processing backlog that also
causes locks.

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

Two solutions — use whichever fits the situation.

### Option A — SQL index (fastest for in-place fix)

Adding indexes is a write operation against navi.db. Under navi-mcp, run
each via `navi_explore_query(sql=..., confirm=True)` — DDL statements
require `confirm=True`, and Claude will narrate each one before executing:

```sql
CREATE INDEX IF NOT EXISTS idx_vulns_plugin ON vulns(plugin_id);
CREATE INDEX IF NOT EXISTS idx_vulns_asset ON vulns(asset_uuid);
CREATE INDEX IF NOT EXISTS idx_vulns_output ON vulns(output);
```

Standalone CLI equivalent:

```bash
navi explore data query "CREATE INDEX IF NOT EXISTS idx_vulns_plugin ON vulns(plugin_id);"
# ...etc
```

Indexes persist in navi.db and speed up all subsequent tagging and
queries against the vulns table. **Indexes must be recreated if you
delete and rebuild navi.db.** These local DDL writes do NOT require
`NAVI_MCP_ALLOW_WRITES=1` — the platform-write gate doesn't apply to
local-only navi.db changes.

### Option B — Purpose-built workload (best for repeated use)

Create a separate navi directory containing only the data you need. See
navi-core's "navi.db — location, scope, and the multi-workload pattern"
section for the full pattern.

Under navi-mcp, this is operator-side: a second navi-mcp server pointed
at a dedicated workload directory. Best for long-running campaigns
(e.g. ongoing cert hygiene, a specific business unit, a WAS-only view)
where you'll reuse the scoped DB many times.

### Picking between A and B

- Hitting one slow plugin once → **Option A** (fast to apply, zero
  operator overhead)
- Repeating the same tagging workload regularly → **Option B** (structural
  fix, much smaller DB, every operation is faster)

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
navi keys --a <ACCESS_KEY> --s <SECRET_KEY>
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
navi keys --a <ACCESS_KEY> --s <SECRET_KEY>
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

## Summary — Troubleshooting Quick Reference

A one-row-per-symptom index for fast lookup. Full detail in the sections
above.

| Symptom | Most likely cause | Fix |
|---|---|---|
| "Zero chunks" on update | API key permissions | Check key scope in Tenable |
| DB locked error | Slow disk | `--threads 1` on full sync |
| DB locked + low RAM | Under 4GB RAM | `--threads 1` + close other apps |
| Tagging very slow | Large DB, no index | SQL index or purpose-built workload |
| No results from any command (MCP) | navi.db empty or keys out-of-band | Run `navi config update full` at CLI; verify with operator |
| No results from any command (standalone) | Keys not set | `navi keys --a ... --s ...` |
| DB errors after upgrade | Schema mismatch | `rm navi.db` + re-keys + `update full` |
| Missing assets | Key scoped to subset | Check key permissions in Tenable One |
| Agent tags return zero | Stale agents table | `navi_config_update(kind="agents")` |

---

## Cross-references

- **navi-core** — setup, schema reference, API key permissions context
  (preventive guidance), the multi-workload pattern
- **navi-mcp** — why `navi config update full` is CLI-only, write-gate
  conventions, data freshness check
- **navi-enrich** — agent-tagging prerequisites, tag UUID preservation
  (the other common "my tags are broken" source)
