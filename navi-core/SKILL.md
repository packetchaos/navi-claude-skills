---
name: navi-core
description: >
  Core reference for Tenable navi CLI: installation (Python 3.12+, Docker build),
  API key setup, database sync commands, version detection, all navi.db table schemas,
  tagging timing, and the 50K asset scale fork. Use for navi setup and core mechanics:
  "how do I install navi?", "set up navi", "update my database", "what tables does
  navi have?", "what version am I running?". Also covers navi config commands, FedRAMP
  URL config, SLA setup, Docker setup, thread count, SQL indexes, multi-workload pattern,
  API key permissions, and prerequisite steps before other navi commands. Also flags
  when long-running syncs must run on the CLI to avoid the MCP tool-call timeout.
  For fix-it workflows ("why isn't navi working", "zero chunks", "db locked", "empty
  results", "missing assets", "after upgrade", "tool call timed out") use
  navi-troubleshooting instead. See the navi router skill for the full skill index.
---

# Navi Core — Setup, Config & Schema

Navi is a CLI tool by packetchaos that wraps the Tenable VM API. All data is stored
in a local SQLite file (`navi.db`). Five command categories: Configuration, Enrichment,
Exploration, Action, Exportation.

**Companion skills — use these for deeper coverage:**
- `navi-mcp` — conventions for using navi through the navi-mcp server
- `navi-troubleshooting` — fix-it workflows for errors, empty results, slow tagging, post-upgrade issues
- `navi-enrich` — all tagging options, `remove=True` ephemeral pattern, tag UUID preservation
- `navi-acr` — Asset Criticality Rating adjustment, Change Reasons, tier mapping
- `navi-explore` — `explore data` and `explore info` full reference
- `navi-export` — all CSV export commands
- `navi-scan` — scan creation, control, evaluate
- `navi-action` — delete, rotate, cancel, encrypt/decrypt
- `navi-mail` — `navi_action_mail`, email harness (double-gated: `NAVI_EMAIL`)
- `navi-remote-exec` — `navi_action_push`, remote-exec harness (double-gated: `NAVI_REMOTE_CODE_EXECUTION`)
- `navi-was` — WAS integration

This skill has two setup paths. Read whichever matches your situation:

- **Running under navi-mcp** → jump to "Setup under navi-mcp" below. Keys and
  installation are handled by your operator; your job is the initial data sync
  and knowing when to re-sync.
- **Installing navi standalone** → see **`references/installation.md`**
  (`navi://skill/core/installation`) for the full install-from-zero walkthrough.

---

## Setup under navi-mcp

If Claude has `navi_*` tools available, navi is already installed and the
navi-mcp server is already running. Three things still apply:

### 1. API keys are set out-of-band

Your operator sets API keys when starting the navi-mcp server. Claude does
not see or manage them. If a query returns empty or commands fail with no
data, the most likely cause is either (a) keys scoped to a subset of
assets, or (b) keys not set at all. Check `navi://workdir` for write-gate
status; for key scope, verify in the Tenable platform directly. See
navi-mcp for the full stance.

### 2. Run `navi config update full` at least once before using the tools

This is the foundational data sync that populates navi.db from the Tenable
platform. **Without it, navi.db is empty and every query returns nothing.**

`navi config update full` is NOT exposed as an MCP tool — first-run syncs
can pull hundreds of GB and run for hours, which is not well-handled by a
tool-call lifecycle. Run it at your terminal:

```bash
navi config update full
```

Scope a full sync to a single tag with `--c`/`--v` (handy for a per-tag /
per-business-unit navi instance): `navi config update full --c "<category>" --v "<value>"`.

After the initial sync completes, you can use `navi_config_update(kind=...)`
for targeted incremental refreshes — those finish in minutes and are fine
as tool calls. See "Targeted database sync (MCP-exposed)" below.

#### Recommended: run `navi config optimize` after the first sync

In navi 8.5.31+, run `navi config optimize` once after the first
`navi config update full` completes:

```bash
navi config optimize
```

This builds a curated set of indexes against the `vulns` table and makes
tagging, querying, and exporting noticeably faster — often the difference
between a tag operation taking hours vs. seconds.

`navi config update full` does NOT build these indexes itself, so without
optimize you'll see slow performance until you run it. Note that targeted
`navi_config_update(kind="vulns")` calls through MCP DO build indexes
automatically — so users running incremental refreshes through the tool
get this for free, but users who only ever do `update full` need to run
optimize separately.

Optimize takes seconds against a populated database. If you run it
against an empty navi.db (no syncs have happened yet) it will print
"no such table: main.vulns" errors per index — that's benign, just
re-run after a sync. See navi-troubleshooting for the specific error.

Indexes survive `navi_config_update(kind=...)` calls. They do NOT
survive deleting navi.db. After any post-upgrade recovery (delete +
re-keys + re-sync), re-run optimize.

### 3. After upgrading navi, you need to recover navi.db

Navi version upgrades cause a schema mismatch with any existing navi.db.
Recovery under navi-mcp is a multi-channel operation — neither Claude nor
the MCP server can do all of it:

1. **You (at the CLI):** delete the old database.
   ```bash
   rm navi.db
   ```
   Use `navi://workdir` to confirm where it lives.

2. **Your operator:** re-enter API keys into the navi-mcp server
   configuration and restart it. Keys are stored in navi.db and are lost
   when it's deleted.

3. **You (at the CLI):** re-sync.
   ```bash
   navi config update full
   ```

4. **You (at the CLI, navi 8.5.31+):** rebuild indexes — `navi config update full`
   does not create them, and they were dropped when navi.db was deleted.
   Without this step, tagging and queries will be much slower than they
   were before the upgrade.
   ```bash
   navi config optimize
   ```

5. **Back to Claude:** once navi.db exists again, is populated, and is
   indexed, resume your MCP workflows.

Store API keys securely outside of navi.db (password manager, environment
variables) so the out-of-band step in (2) is quick after an upgrade.

---

## Standalone installation

Installing navi directly — Python 3.12+, Docker build, API key entry,
key-permission scoping, version detection, and standalone post-upgrade
recovery — is operator-only and already done for you under navi-mcp. Full
walkthrough lives in **`references/installation.md`** (via the resource:
`navi://skill/core/installation`).

> Key-scope reminder (the one install fact that bites later): navi only sees
> what the API key can see. A key scoped to a subset of assets silently
> returns partial data — the usual cause of "missing assets" / "zero chunks."
> See navi-troubleshooting for the diagnostic ladder.

---

## Targeted database sync (MCP-exposed)

Once navi.db exists and has had a `navi config update full` run at least
once, targeted refreshes are exposed through MCP as `navi_config_update(kind=...)`.
Each finishes in minutes rather than hours and is safe to call as a tool.

> **⚠️ Long syncs must run on the CLI, not through MCP.** The "minutes
> rather than hours" expectation above holds for most tenants — but on a
> large tenant a `vulns` (or full-size `assets`) refresh can run for tens
> of minutes to **hours**. MCP tool calls are synchronous and the client
> enforces a hard tool-call timeout (currently around 4 minutes in Claude
> Desktop) that **cannot be raised from Claude's side** — it's set by the
> MCP host, not by navi or navi-mcp, and there is no tool parameter to
> extend it. When a sync exceeds that window the tool call fails with a
> timeout even though the export usually keeps running on the Tenable
> side, leaving you with no progress visibility and the local subprocess
> still holding navi.db open.
>
> **Rule of thumb:**
> - **Through MCP** — incremental refreshes you're confident will finish
>   in a few minutes (small/scoped tenants, short `days=N` lookback
>   windows). `navi_config_update(kind="vulns", days=7)` for a weekly
>   catch-up is the canonical example.
> - **On the CLI** — any first-time pull, full-size refresh, or anything
>   on a large tenant. Run it at a terminal where it can take as long as
>   it needs, throttling threads if the disk is slow:
>
>     ```bash
>     navi config update vulns --threads 1
>     navi config update assets
>     ```
>
> Narrowing the window with `days=N` is the one practical lever that
> keeps a vulns refresh inside the MCP timeout. A full-history pull
> should always be CLI. For recovery steps when a sync does time out
> (including what to do about the stuck subprocess and the DB lock that
> often follows), see navi-troubleshooting's "Long-running operations
> and MCP timeouts". This is the same reasoning that keeps
> `navi config update full` off the MCP surface entirely.

`navi_config_update(kind="assets")` — assets only
`navi_config_update(kind="vulns")` — vulns only
`navi_config_update(kind="compliance")` — compliance checks (required before
`navi_export(subcommand="compliance")`)
`navi_config_update(kind="agents")` — agent data (required before agent-based
tagging: `group`, `missed`, `byadgroup` selectors in navi-enrich)
`navi_config_update(kind="route")` — vuln_route table (technology-level routing)
`navi_config_update(kind="paths")` — vuln_paths table (filesystem/URL paths
per vuln)
`navi_config_update(kind="was")` — WAS apps + findings tables
`navi_config_update(kind="fixed")` — fixed-vuln table (required before
`navi_export(subcommand="failures")` / SLA processing)
`navi_config_update(kind="plugins", size=N)` — full Tenable plugin DB.
**`size` (1000–10000) is REQUIRED for `kind="plugins"`** (it's the API page
size). Can be large/slow on the first run — mind the call budget.

> **Certificates are NOT a `config update` kind.** The SSL/TLS cert table is
> populated by **`navi_config(kind="certificates")`** (CLI: `navi config
> certificates`), which parses plugin 10863 into the `certs` table. There is no
> `navi config update certificates`. See "Other configuration" below.

**Lookback window:** `navi_config_update(kind=..., days=N)` accepts a `days`
parameter to change the lookback window. **`days` is ONLY valid for
`kind="assets"`, `kind="vulns"`, or `kind="fixed"`** — passing it with any
other kind raises an error.

**Standalone CLI equivalents:**

```bash
navi config update assets
navi config update vulns
navi config update compliance
navi config update agents
navi config update route
navi config update paths
navi config update was
navi config update fixed
navi config update plugins --size 10000
navi config certificates          # cert table — NOT `config update certificates`
```

**Agents note:** `navi_config_update(kind="agents")` is NOT included in the
foundational `navi config update full` CLI command — it must be run
explicitly whenever agent data is needed.

---

## Other configuration (MCP-exposed)

### SLA thresholds

`navi config sla` is a **group**, not a single command — it has two subcommands:
`calculate` (compute SLA times against your `fixed` data) and `reset`
(set/overwrite SLA threshold values). A bare `navi config sla` does nothing.

`navi_config(kind="sla")` runs **`config sla calculate`** — **not write-gated,
no confirm required.** It computes SLA times, so populate the `fixed` table
first with `navi_config_update(kind="fixed")`. Meaningful
`navi_export(subcommand="failures")` output depends on this.

```bash
navi config sla calculate          # what the MCP tool runs
```

**Setting/overwriting the thresholds themselves** is `navi config sla reset`,
which is **interactive** (it prompts for per-severity values). Run it at the
terminal — the MCP tool does not drive it:

```bash
navi config sla reset
```

### Software table build (not write-gated)

Parses software inventory plugins (22869, 20811, 83991) into the `software`
table. Local DB operation, doesn't touch the Tenable platform.

`navi_config(kind="software")`

```bash
navi config software
```

### Certificate table build (not write-gated)

Parses plugin 10863 (SSL Certificate Information) into the `certs` table. Local
DB operation, doesn't touch the Tenable platform. Required before large-scale
cert tagging (see Scale Fork below).

`navi_config(kind="certificates")`

```bash
navi config certificates
```

This is the correct cert-table command. It is **not** `navi config update
certificates` — that subcommand does not exist. (Plugin 10863 doubles as an
IoT/appliance fingerprint — dumped certs identify devices — not just an
expiry source; see navi-enrich's device-fingerprinting playbook.)

### FedRAMP / custom base URL (write-gated)

Change the Tenable base URL (FedRAMP tenants, test environments).

`navi_config(kind="url", url="https://fedcloud.tenable.com", confirm=True)`

```bash
navi config url "https://fedcloud.tenable.com"
```

---

## Tagging timing

After a tagging write (`navi_enrich_tag`, `navi_enrich_acr`), allow **up to
30 minutes** for tags to be fully visible in the Tenable UI/API before
re-syncing. This is a platform-side propagation delay, not a navi delay.

Two implications for MCP workflows:

1. **Verification via `navi_explore_query` reflects reality fast** — it
   reads navi.db, which navi just wrote to. Use this for fast verification
   that the tag was applied.
2. **Verification via Tenable UI or `navi_explore_info` can lag** — these
   read from the platform, which is still propagating. If the user looks
   at the UI immediately after a tag write and doesn't see it, that's
   expected, not broken.

To surface the new tags back into navi.db after the propagation window,
run `navi_config_update(kind="assets")` or a targeted refresh. For a full
data refresh after significant tag changes, `navi config update full` at
the CLI is the authoritative sync.

---

## navi.db — location, scope, and the multi-workload pattern

`navi.db` is created in **whichever directory you run navi commands from**.
This is a deliberate design feature, not a limitation.

Under navi-mcp, the workdir is fixed by the server config. Check
`navi://workdir` to see where it is.

**Why this matters**: you can maintain multiple, purpose-built navi
databases by running navi from different directories. Each database
contains only the data you synced into it — making queries faster and
tagging operations significantly quicker when working on a specific subset.

```bash
# Full environment database
mkdir ~/navi-full && cd ~/navi-full
navi config keys --a <KEY> --s <KEY>
navi config update full          # everything — large, comprehensive

# Purpose-built workload: only assets with a specific plugin
mkdir ~/navi-jenkins && cd ~/navi-jenkins
navi config keys --a <KEY> --s <KEY>
navi config update vulns         # then filter to only pull plugin 12345
# Results: smaller DB, faster tagging, faster queries
```

**The Exposure Management Environment pattern**: treat each navi directory
as a scoped workload environment — a compliance audit, a specific
vulnerability campaign, a WAS-only view, a single business unit. Navi runs
independently in each directory. The smaller the database, the faster
every operation against it.

**If tagging against a specific plugin is slow**: create a purpose-built
workload that only contains assets where that plugin fired. Smaller
dataset = faster tagging. The alternative is adding a SQL index (see
Troubleshooting below).

**Under navi-mcp, the multi-workload pattern is operator-side.** Each
workload environment would need its own navi-mcp server instance pointed
at its own directory. Claude operates against whichever single workdir
the current server is configured for.

---

## Scale Fork — 50K asset threshold

Two workflows exist for certificate and software tagging depending on
environment size. Always check first:

`navi_explore_query(sql="SELECT count(uuid) FROM assets;")`

```bash
navi explore data query "SELECT count(uuid) FROM assets;"
```

- **Under 50K**: use `plugin` + `plugin_output` or `plugin_regexp` (simpler, no
  extra download)
- **Over 50K**: use dedicated tables (`certs`, `software`) via `query=...`
  (much faster at scale)

See navi-enrich for full PATH A / PATH B workflows on both certs and
software tagging.

---

## Key SSL/TLS Certificate Plugins

| Plugin | Description |
|--------|-------------|
| `10863` | SSL Certificate Information (expiry dates) |
| `15901` | Weak Hashing Algorithm (SHA-1) |
| `42981` | Cannot Be Trusted (chain) |
| `51192` | Untrusted / expired / self-signed |
| `69511` | RSA key < 2048 bits |
| `60108` | Weak RSA key in chain |

---

## Database Schema Reference

The full table-by-table schema for navi.db — every table, its columns, how it's
populated, and how tables join — lives in **`references/schema.md`** (via the
resource: `navi://skill/core/schema`). Load it when composing non-trivial
queries. For a single table's columns, prefer the live `navi://schema/{table}`
resource, which can't go stale.

Quick orientation: `vulns`/`assets`/`plugins`/`tags`/`fixed`/`agents` come from
core syncs; `certs`/`software`/`compliance`/`epss`/`zipper` are targeted builds;
`vuln_route`/`vuln_paths` are routing tables; `apps`/`findings` are WAS.

---


## DISTINCT — the workload reality check

The `vuln_paths` table shows each path × plugin combination. A single path
can have 10+ plugins firing against it — but it's still **one fix location**.

```sql
-- Raw count (inflated — same path appears many times)
SELECT count(*) FROM vuln_paths;

-- TRUE workload — distinct locations a remediator actually needs to visit
SELECT count(DISTINCT path) FROM vuln_paths;
```

Real example: 148 raw entries → 28 distinct fix locations = **81% workload
reduction**. Always use DISTINCT when communicating workload to remediators.

---

## Quick Command Map

| Need | MCP tool call / CLI |
|------|---------|
| Set API keys | CLI only, out-of-band: `navi config keys --a ... --s ...` |
| Full foundational sync | CLI only: `navi config update full` |
| Sync assets | `navi_config_update(kind="assets")` |
| Sync vulns | `navi_config_update(kind="vulns")` |
| Sync agents | `navi_config_update(kind="agents")` |
| Build routing + paths tables | `navi_config_update(kind="route")` then `navi_config_update(kind="paths")` |
| Build cert table | `navi_config(kind="certificates")` (CLI: `navi config certificates`) |
| Build fixed table (SLA) | `navi_config_update(kind="fixed")` |
| Build full plugin DB | `navi_config_update(kind="plugins", size=10000)` (size required) |
| Build software table | `navi_config(kind="software")` |
| Build EPSS table | CLI only: `navi config epss` (downloads EPSS CSV and populates the `epss` table) |
| Build indexes for fast tagging/querying (8.5.31+) | CLI only: `navi config optimize` |
| Check version | `navi_explore_info(subcommand="version")` |
| Inspect table schema | `navi://schema/{table}` resource, or `navi_explore_data(subcommand="db_info", table=...)` |
| Spot-check an asset | `navi_explore_data(subcommand="asset", asset=<IP_or_UUID>)` |
| See workdir + write-gate status | `navi://workdir` resource |
| Reset after upgrade | See "After upgrading navi" above |

---

## Troubleshooting — see navi-troubleshooting

Full per-symptom fix guidance lives in the `navi-troubleshooting` skill.
The most frequent issues and their fixes:

| Symptom | Most likely cause | Fix |
|---|---|---|
| "Zero chunks" on update | Empty window, broken scanners, or key scope | Wider window first (`--days 365`), then scanner health, then key scope. See navi-troubleshooting |
| DB locked error | Slow disk | `--threads 1` on full sync |
| DB locked + low RAM | Under 4GB RAM | `--threads 1` + close other apps |
| Tagging very slow | Large DB, no index | SQL index or purpose-built workload |
| No results from any command (MCP) | navi.db empty or keys out-of-band | Run `navi config update full` at CLI; verify with operator |
| No results from any command (standalone) | Keys not set | `navi config keys --a ... --s ...` |
| DB errors after upgrade | Schema mismatch | `rm navi.db` + re-keys + `update full` |
| Missing assets | Key scoped to subset | Check key permissions in Tenable One |
| Agent tags return zero | Stale agents table | `navi_config_update(kind="agents")` |
| `navi_config_update` times out after ~4 min | Sync longer than MCP client timeout (big tenant, usually vulns) | Run on CLI: `navi config update vulns --threads 1`; or shrink with `days=N` |

For full context on each symptom — root cause, resolution steps, MCP vs.
standalone variants — see **navi-troubleshooting**.

**Preventive context**: the key-scope reminder in "Standalone installation"
above (and the full "API key permissions matter" detail in
`references/installation.md`) explains why scoped keys cause Zero Chunks later.
The multi-workload pattern explains how purpose-built navi directories reduce
tagging slowness structurally. Both are install-time concerns;
navi-troubleshooting covers the reactive fixes.
