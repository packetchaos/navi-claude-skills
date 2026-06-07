# navi-troubleshooting reference — slow tagging (full options)

The complete slow-tagging fixes. SKILL.md keeps the primary fix (`navi config
optimize`); this file has the seven indexes optimize builds, the manual-SQL
fallback (Option B), and the purpose-built-workload approach (Option C).

## Slow tagging (plugin-based tags taking a long time)

Two solutions — use whichever fits the situation.

### Option A — `navi config optimize` (navi 8.5.31+, fastest fix)

In navi 8.5.31 and later, the `navi config optimize` CLI command builds a
curated set of indexes against `navi.db` in seconds. This is the fastest
fix for slow tagging and slow queries, and it covers more columns than
the manual approach below. Run it at the user's terminal:

```bash
navi config optimize
```

The command builds these indexes against the `vulns` table:

| Index | Column / filter | Accelerates |
|---|---|---|
| `idx_plugin` | `plugin_id` | All plugin-based tagging and queries |
| `idx_plugin_2` | `plugin_id WHERE plugin_id='19506'` | Scan-time data lookups (used by `navi_scan(subcommand="evaluate")`) |
| `idx_plugin_3` | `plugin_id WHERE plugin_id='10863'` | Cert tagging and cert-related queries |
| `idx_OSes` | `OSes` | OS-based queries and tagging |
| `idx_severity` | `severity` | Severity filters across the board |
| `idx_cves` | `cves` | CVE-based queries (and EPSS joins via `LIKE`) |
| `idx_name` | `plugin_name` | Plugin name searches |

**Important behavioral notes:**

- **Run it AFTER `navi config update full`.** The `vulns` table needs to
  exist first. If you run `optimize` against an empty navi.db, navi will
  print "no such table: main.vulns" errors for each index and exit
  without creating anything. This is benign — just run it after at least
  one sync has completed.
- **`navi config update full` does NOT build these indexes.** They have
  to be created separately via `optimize`.
- **`navi_config_update(kind="vulns")` (the targeted MCP tool) DOES
  build them automatically.** If a user's regular workflow includes
  incremental vuln refreshes through navi-mcp, indexes stay current and
  optimize may not be needed.
- **Indexes are dropped when `navi.db` is rebuilt.** Any post-upgrade
  recovery (delete navi.db → re-keys → full sync) needs an `optimize`
  step at the end to restore index performance.

### Option B — Manual SQL indexes (fallback for navi < 8.5.31)

For users on navi versions older than 8.5.31, or for custom indexes navi
optimize doesn't create, manual `CREATE INDEX` statements still work.

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

These local DDL writes do NOT require `NAVI_MCP_ALLOW_WRITES=1` — the
platform-write gate doesn't apply to local-only navi.db changes.

### Option C — Purpose-built workload (best for repeated specialised use)

Create a separate navi directory containing only the data you need. See
navi-core's "navi.db — location, scope, and the multi-workload pattern"
section for the full pattern.

Under navi-mcp, this is operator-side: a second navi-mcp server pointed
at a dedicated workload directory. Best for long-running campaigns
(e.g. ongoing cert hygiene, a specific business unit, a WAS-only view)
where you'll reuse the scoped DB many times.

### Picking between A, B, and C

- On navi 8.5.31+, default to **Option A** — fastest, most comprehensive,
  zero ceremony.
- On older navi or for custom indexes navi optimize doesn't build →
  **Option B**.
- Repeating the same tagging workload regularly → **Option C** (structural
  fix; smaller DB, every operation faster).

