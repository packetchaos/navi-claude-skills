---
name: navi-core
description: >
  Core reference for Tenable navi CLI: installation (Python 3.12+, Docker build),
  API key setup, database sync commands, version detection, all navi.db table schemas,
  tagging timing, and the 50K asset scale fork. Use for navi setup and core mechanics:
  "how do I install navi?", "set up navi", "update my database", "what tables does
  navi have?", "what version am I running?". Also covers navi config commands, FedRAMP
  URL config, SLA setup, Docker setup, thread count, SQL indexes, multi-workload pattern,
  API key permissions, and prerequisite steps before other navi commands. For fix-it
  workflows ("why isn't navi working", "zero chunks", "db locked", "empty results",
  "missing assets", "after upgrade") use navi-troubleshooting instead. See the navi
  router skill for the full skill index.
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
- `navi-action` — delete, rotate, cancel, encrypt/decrypt; plus push and mail (CLI-only)
- `navi-was` — WAS integration

This skill has two setup paths. Read whichever matches your situation:

- **Running under navi-mcp** → jump to "Setup under navi-mcp" below. Keys and
  installation are handled by your operator; your job is the initial data sync
  and knowing when to re-sync.
- **Installing navi standalone** → read "Standalone installation" for the full
  install-from-zero walkthrough.

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

After the initial sync completes, you can use `navi_config_update(kind=...)`
for targeted incremental refreshes — those finish in minutes and are fine
as tool calls. See "Targeted database sync (MCP-exposed)" below.

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

4. **Back to Claude:** once navi.db exists again and is populated, resume
   your MCP workflows.

Store API keys securely outside of navi.db (password manager, environment
variables) so the out-of-band step in (2) is quick after an upgrade.

---

## Standalone installation

**This section is for operators installing navi directly, not for end-users
running under navi-mcp.** Under navi-mcp, these steps have already been
done for you.

### Python requirement

Navi requires **Python 3.12 or higher**. This is the most common first-run blocker.

```bash
# Check Python version first
python3 --version

# If below 3.12, install via pyenv (recommended — avoids system Python conflicts)
curl https://pyenv.run | bash
pyenv install 3.12
pyenv global 3.12

# Or via Homebrew on macOS
brew install python@3.12
echo 'export PATH="/usr/local/opt/python@3.12/bin:$PATH"' >> ~/.zshrc

# Or via apt on Ubuntu/Debian
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update && sudo apt install python3.12 python3.12-pip

# Then install navi
pip3 install navi-hostio
```

### Docker — build from source (recommended over the published image)

The published Docker image on Docker Hub is outdated. Build from the latest
GitHub source instead:

```bash
# Clone the latest navi source
git clone https://github.com/packetchaos/navi.git
cd navi

# Build a fresh image from the current Dockerfile
docker build -t navi:latest .

# Run navi in a container — mount a local directory to persist navi.db
docker run -it \
  -v $(pwd)/navi-data:/home/navi \
  navi:latest \
  navi keys --a <ACCESS_KEY> --s <SECRET_KEY>
```

Mounting a local volume (`-v`) is critical — otherwise the navi.db is lost
when the container stops.

### API keys — set these BEFORE running any other command

**The most common mistake**: running `navi config update full` or any other
command before setting API keys. Navi will appear to run but return no data.

```bash
navi keys --a <ACCESS_KEY> --s <SECRET_KEY>
```

API keys must be set first. Every other command depends on them.

### API key permissions matter

Navi can only see what your API key can see. If the API key is scoped to a
subset of assets (e.g. a specific network or business unit), `navi config
update full` will only download data for those assets. This is intentional
and useful for scoped workloads — but it is also the cause of "why am I
missing assets?" questions.

**"Zero chunks" error on update commands** = the API key has no permission
to the data being requested. Check:

- Does the key have access to the assets/vulnerabilities you expect?
- Is the key scoped to ALL assets, or a subset?
- If you need full environment coverage, the key must have permissions on
  ALL assets.

### Check version after install

```bash
navi explore info version
```

Under navi-mcp, the same check is available as
`navi_explore_info(subcommand="version")`.

- **8.2+**: use `navi enrich tag`, `navi explore data query`, `navi explore asset`
- **Pre-8.2**: use `navi tag`, `navi find query`

Default to 8.2+ syntax unless the user confirms an older version.

### After upgrading navi to a new version (standalone)

Navi version upgrades cause a database schema mismatch. When this happens:

```bash
# 1. Delete the old database
rm navi.db

# 2. Re-enter API keys (they are stored in navi.db)
navi keys --a <ACCESS_KEY> --s <SECRET_KEY>

# 3. Re-sync data
navi config update full
```

This is expected behaviour — not a bug. For MCP users, see the three-step
recovery under "Setup under navi-mcp" above.

---

## Targeted database sync (MCP-exposed)

Once navi.db exists and has had a `navi config update full` run at least
once, targeted refreshes are exposed through MCP as `navi_config_update(kind=...)`.
Each finishes in minutes rather than hours and is safe to call as a tool.

`navi_config_update(kind="assets")` — assets only
`navi_config_update(kind="vulns")` — vulns only
`navi_config_update(kind="compliance")` — compliance checks (required before
`navi_export(subcommand="compliance")`)
`navi_config_update(kind="agents")` — agent data (required before agent-based
tagging: `group`, `missed`, `byadgroup` selectors in navi-enrich)
`navi_config_update(kind="certificates")` — SSL/TLS cert table (required for
large-scale cert tagging — see Scale Fork below)
`navi_config_update(kind="route")` — vuln_route table (technology-level routing)
`navi_config_update(kind="paths")` — vuln_paths table (filesystem/URL paths
per vuln)
`navi_config_update(kind="was")` — WAS apps + findings tables

**Lookback window:** `navi_config_update(kind="assets", days=N)` and
`navi_config_update(kind="vulns", days=N)` accept a `days` parameter to
change the lookback window. **`days` is ONLY valid for `kind="assets"` or
`kind="vulns"`** — passing it with any other kind raises an error.

**Standalone CLI equivalents:**

```bash
navi config update assets
navi config update vulns
navi config update compliance
navi config update agents
navi config update certificates
navi config update route
navi config update paths
navi config update was
```

**Agents note:** `navi_config_update(kind="agents")` is NOT included in the
foundational `navi config update full` CLI command — it must be run
explicitly whenever agent data is needed.

---

## Other configuration (MCP-exposed)

### SLA thresholds

Set custom SLA thresholds per severity. Required before
`navi_export(subcommand="failures")` returns meaningful data.

`navi_config(kind="sla")` — **not write-gated, no confirm required.**

```bash
navi config sla
```

Be aware that SLA setup is interactive on the CLI (it prompts for
threshold values). When invoked through MCP, the tool receives no stdin,
so the command will either use existing defaults or complete without
accepting interactive input. For **initial** SLA configuration, running
the command at the terminal is the more reliable path. For re-runs after
defaults are already set, the MCP tool works fine.

### Software table build (not write-gated)

Parses software inventory plugins (22869, 20811, 83991) into the `software`
table. Local DB operation, doesn't touch the Tenable platform.

`navi_config(kind="software")`

```bash
navi config software
```

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
navi keys --a <KEY> --s <KEY>
navi config update full          # everything — large, comprehensive

# Purpose-built workload: only assets with a specific plugin
mkdir ~/navi-jenkins && cd ~/navi-jenkins
navi keys --a <KEY> --s <KEY>
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

Schemas are also accessible live via the `navi://schema/{table}` resource
under navi-mcp — prefer that when composing a query and unsure of column
names, rather than relying on this static reference going out of date.

### Core tables — populated by `navi config update full`

**vulns** — active vulnerability findings (split exports to save space)
`navi_id (PK), asset_ip, asset_uuid, asset_hostname, plugin_id, plugin_name,
output, severity, cves, port, protocol, plugin_family, scan_uuid, scan_completed,
scan_started, schedule_id, first_found, last_found, state`
Note: `cves` is a comma-separated string of CVE IDs — used to JOIN with `epss` table.

**assets** — asset inventory (split exports to save space)
`ip_address, hostname, fqdn, uuid (PK), operating_system, mac_address, agent_uuid,
first_found, last_found, network, last_licensed_scan_date`

**plugins** — plugin metadata (descriptions, solutions, CVE/xref mappings)
`plugin_id (PK), plugin_name, plugin_family, description, solution, cves, xrefs,
severity, risk_factor`
This is separate from per-asset `vulns`. Use it to look up what a plugin does,
what CVEs it maps to, and what cross-references (CISA, IAVA, BID) it contains.
This is what enables `navi_explore_data(subcommand="cve")`,
`navi_explore_data(subcommand="xrefs")`, and `tone=True` tagging.

**fixed** — remediated vulnerabilities (from fixed vuln endpoint)
`asset_uuid, asset_ip, plugin_id, plugin_name, severity, output, last_found, fixed_date`
Tracks vulns that have transitioned to "fixed" state. Use to verify remediation
is confirmed and to track closure rates over time.
Populate: part of `navi_config_update(kind="vulns")`.

**agents** — Nessus agent inventory (Agent API endpoint)
`uuid (PK), name, ip_address, status, platform, version, group, last_connect,
agent_uuid, linked_on`
Populate: `navi_config_update(kind="agents")` — NOT included in the
foundational `navi config update full`; must be run explicitly.

**tags** — tag assignments
`tag_id (PK), asset_uuid, asset_ip, tag_key, tag_uuid, tag_value, tag_added_date`

---

### Enrichment tables — separate targeted updates

**compliance** — compliance check results
`asset_uuid, check_name, actual_value, expected_value, status, audit_file,
plugin_id, first_seen, last_seen, solution`
Populate: `navi_config_update(kind="compliance")`

**software** — installed software inventory (from plugins 22869/20811/83991)
`asset_uuid, software_string`
Populate: `navi_config(kind="software")` | Requires credentialed scans

**certs** — SSL/TLS certificate data (parsed from plugin 10863)
`asset_uuid, common_name, issuer_name, not_valid_before, not_valid_after,
algorithm, key_length, signature_algorithm, subject_name, serial_number,
country, state_province, organization_unit, signature_length`
Dates in OpenSSL format: `Sep 04 20:36:29 2024 GMT`
Populate: `navi_config_update(kind="certificates")`

**epss** — Exploit Prediction Scoring System scores per CVE (downloaded from EPSS CSV)
`cve (PK), epss_value, percentile`
Provides probability scores (0.0–1.0) for CVE exploitation likelihood.
Use to prioritize remediation by actual exploit probability, not just CVSS severity.
Populate: downloaded and parsed from EPSS CSV (separate process, not an MCP tool)

**Key EPSS query** (join vulns to EPSS scores):
```sql
SELECT e.epss_value, v.asset_ip, v.plugin_name, v.severity
FROM vulns v
INNER JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
WHERE e.epss_value > 0.5
ORDER BY e.epss_value DESC;
```

**zipper** — internal aggregation/join table that combines data from multiple sources
Used internally by navi to assemble cross-source views. Not typically queried directly.

---

### Routing tables — `navi_config_update(kind="route")` + `kind="paths"`

**vuln_route** — technology-level vuln routing
`route_id (PK), app_name, plugin_list, total_vulns, vuln_type (Application | Operating System)`
Populate: `navi_config_update(kind="route")` | View: `navi_explore_data(subcommand="route")`

**vuln_paths** — vulnerable filesystem/URL paths
`path_id (PK), plugin_id, path, asset_uuid`
Populate: `navi_config_update(kind="paths")` | View: `navi_explore_data(subcommand="paths")`

---

### WAS tables — `navi_config_update(kind="was")`

**apps** — WAS scan summaries (one row per completed WAS scan config)
`name, uuid (PK), target, scan_completed_time, pages_audited, pages_crawled,
requests_made, critical_count, high_count, medium_count, low_count,
info_count, owasp, tech_list, config_id`

**findings** — WAS finding details per scan (note: NOT "plugins" — "findings" is correct)
`uuid (PK), config_id (FK→apps), plugin_id, plugin_name, severity, output,
solution, scan_completed_time`

---

### Planned / not yet implemented

**mitre** — MITRE ATT&CK technique data (downloaded and parsed from MITRE CSV)
Status: planned, not implemented. Will map plugin_ids to ATT&CK techniques.
Currently MITRE cross-reference tagging relies on xref data in the `plugins`
table.

**cisa_kev** — CISA Known Exploited Vulnerabilities (downloaded and parsed)
Status: planned, not implemented as a standalone table.
Currently covered via `navi_enrich_tag(xrefs="CISA", ...)` which reads from `plugins.xrefs`.

**ownership** — asset ownership assignments
Status: potential/in development. Will link assets to owners/teams.

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
| Set API keys | CLI only, out-of-band: `navi keys --a ... --s ...` |
| Full foundational sync | CLI only: `navi config update full` |
| Sync assets | `navi_config_update(kind="assets")` |
| Sync vulns | `navi_config_update(kind="vulns")` |
| Sync agents | `navi_config_update(kind="agents")` |
| Build routing + paths tables | `navi_config_update(kind="route")` then `navi_config_update(kind="paths")` |
| Build cert table | `navi_config_update(kind="certificates")` |
| Build software table | `navi_config(kind="software")` |
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
| "Zero chunks" on update | API key permissions | Check key scope in Tenable |
| DB locked error | Slow disk | `--threads 1` on full sync |
| DB locked + low RAM | Under 4GB RAM | `--threads 1` + close other apps |
| Tagging very slow | Large DB, no index | SQL index or purpose-built workload |
| No results from any command (MCP) | navi.db empty or keys out-of-band | Run `navi config update full` at CLI; verify with operator |
| No results from any command (standalone) | Keys not set | `navi keys --a ... --s ...` |
| DB errors after upgrade | Schema mismatch | `rm navi.db` + re-keys + `update full` |
| Missing assets | Key scoped to subset | Check key permissions in Tenable One |
| Agent tags return zero | Stale agents table | `navi_config_update(kind="agents")` |

For full context on each symptom — root cause, resolution steps, MCP vs.
standalone variants — see **navi-troubleshooting**.

**Preventive context that stays in this skill**: the "API key permissions
matter" subsection under Standalone installation explains why scoped keys
cause Zero Chunks later. The multi-workload pattern explains how
purpose-built navi directories reduce tagging slowness structurally.
Both are install-time concerns; navi-troubleshooting covers the reactive
fixes.
