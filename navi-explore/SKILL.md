---
name: navi-explore
description: >
  Data exploration skill for Tenable navi CLI. Use for ANY request to query, search,
  or display navi data. Covers all navi explore data subcommands: cve, exploit, name,
  output, xrefs, docker, webapp, creds, scantime, software, audits, plugin, port,
  route, paths, asset, db_info. Also covers all 26 navi explore info subcommands:
  users, scanners, scans, running, policies, credentials, agents, agent_groups,
  networks, tags, categories, assets, licensed, status, sla, logs, permissions, auth,
  exclusions, target_groups, templates, exports, tone, attributes, user_groups,
  version. Also covers raw SQL access via navi_explore_query. Trigger on: "show me",
  "find assets", "query navi", "which assets have", "list scanners/users/scans/policies",
  "show me what's running", "CVE lookup", "exploitable assets", "what's in my
  environment".
---

# Navi Explore — Data Exploration Reference

Three surfaces for getting data out of navi:

- **`navi_explore_data(subcommand=..., ...)`** — canned SQL against the local
  navi.db for the most common queries (find by CVE, by plugin, by xref, etc.).
  Fast, preformatted output.
- **`navi_explore_info(subcommand=...)`** — live reads against the Tenable API
  for platform state (users, scanners, scans, policies, current license
  status, etc.). Always current, doesn't depend on navi.db freshness.
- **`navi_explore_query(sql=..., limit=500)`** — raw SQL against navi.db.
  Use when a subcommand doesn't cover what you need, or when you want
  custom joins and aggregations. Supports both reads (SELECT / WITH, no
  confirmation) and writes (DDL, UPDATE, DELETE — require `confirm=True`).

When running under navi-mcp, use tool-invocation form (shown first in each
example below). Bash forms are standalone CLI equivalents for readers
outside an MCP context. See navi-mcp for the conventions around the
freshness check and write-gated operations.

---

## When to reach for which surface

| You want… | Use… |
|---|---|
| Platform state (users, scanners, scans, policies) | `navi_explore_info` |
| A common pattern navi has a subcommand for | `navi_explore_data` |
| Custom columns, joins, aggregations, or time windows | `navi_explore_query` |
| To see what tables exist / a table's schema | `navi://schema/{table}` resource |
| To verify a write just landed in navi.db | `navi_explore_query` (fast, reads local DB) |
| To verify a write is visible in Tenable platform | `navi_explore_info` (live, may lag the 30-min propagation window) |

---

## `navi_explore_data` — queries against the local database

### Core query and asset lookup

Custom SQL (also covered under `navi_explore_query` — use whichever reads
more naturally in context).

`navi_explore_query(sql="SELECT ...")`

```bash
navi explore data query "SELECT ..."
```

Pre-8.2 CLI equivalent (standalone only): `navi find query "SELECT ..."`

All data for one asset — pass IP or UUID via the `asset` parameter.

`navi_explore_data(subcommand="asset", asset="<IP_or_UUID>")`

```bash
navi explore asset <IP_or_UUID>
```

All output for one plugin across all assets.

`navi_explore_data(subcommand="plugin", plugin_id=<PLUGIN_ID>)`

```bash
navi explore plugin <PLUGIN_ID>
```

Inspect a table's schema. Prefer the `navi://schema/{table}` resource when
running under navi-mcp — it's cheaper and always available. The db_info
subcommand works too.

`navi_explore_data(subcommand="db_info", table="<table>")`

```bash
navi explore data db-info --table <table>
```

### CLI-only helpers — coming to MCP

Two `navi explore` commands are not yet exposed as MCP tools but are planned:

- **`navi explore uuid <IP_or_UUID>`** — single-asset detail lookup with
  flags for specific plugin output (patch info, processes, connections,
  services, software, exploits, etc.). Highly valuable for deep-diving a
  single host. Until `navi_explore_uuid` ships, use
  `navi_explore_data(subcommand="asset", asset="<target>")` for the basic
  case, or drop to the CLI for flag-driven views:
  ```bash
  navi explore uuid 192.168.1.50 -software
  navi explore uuid 192.168.1.50 -patches
  navi explore uuid 192.168.1.50 --plugin 19506
  ```
- **`navi explore api <URL>`** — passthrough GET/POST/PUT against the
  Tenable API. Useful for troubleshooting API behavior directly. Until
  `navi_explore_api` ships, drop to the CLI:
  ```bash
  navi explore api /scans
  navi explore api /scans/123 -post --payload '{"name":"updated"}'
  ```

Both tools are planned additions to the navi-mcp surface. When they ship,
the CLI handoff becomes a tool call.

### Vulnerability search subcommands

Find all assets affected by a specific CVE.

`navi_explore_data(subcommand="cve", cve="CVE-2021-44228")`

```bash
navi explore data cve --cve CVE-2021-44228
```

Find all assets with exploitable vulnerabilities.

`navi_explore_data(subcommand="exploit")`

```bash
navi explore data exploit
```

Find assets where the plugin NAME contains text.

`navi_explore_data(subcommand="name", name="Apache")`

```bash
navi explore data name --name "Apache"
```

Find assets where the plugin OUTPUT contains text.

`navi_explore_data(subcommand="output", output="Log4j")`

```bash
navi explore data output --output "Log4j"
```

Find by cross-reference type — CISA KEV, IAVA, Bugtraq, OSVDB, MSFT, MSKB, CVE.

`navi_explore_data(subcommand="xrefs", xref_type="CISA")`
`navi_explore_data(subcommand="xrefs", xref_type="iava", xref_id="2024-0001")`

```bash
navi explore data xrefs --type "CISA"
navi explore data xrefs --type "iava" --id "2024-0001"
```

Find assets with a vulnerability on a specific port.

`navi_explore_data(subcommand="port", port=3389)`

```bash
navi explore data port --port 3389
```

### Asset/service discovery subcommands

Find Docker hosts (plugin 93561 — requires credentialed scans).

`navi_explore_data(subcommand="docker")`

```bash
navi explore data docker
```

Find potential web applications (plugins 1442 + 22964).

`navi_explore_data(subcommand="webapp")`

```bash
navi explore data webapp
```

### Scan health & coverage subcommands

Find assets with credential failures (plugin 104410).

`navi_explore_data(subcommand="creds")`

```bash
navi explore data creds
```

Find assets where scan duration exceeded N minutes. Per-asset view from
navi.db. `navi_scan(subcommand="evaluate", ...)` gives a per-scan summary
instead — both are useful.

`navi_explore_data(subcommand="scantime", minutes=30)`

```bash
navi explore data scantime --minutes 30
```

### Software & compliance subcommands

Show software statistics across the environment. Requires
`navi_config(kind="software")` to have populated the software table.

`navi_explore_data(subcommand="software")`

```bash
navi explore data software
```

Show compliance audit results (CIS, STIG, etc.). Requires
`navi_config_update(kind="compliance")` to have populated the compliance table.

`navi_explore_data(subcommand="audits")`

```bash
navi explore data audits
```

### Routing & paths subcommands

All routes — `app_name, vuln_type, total_vulns`.

`navi_explore_data(subcommand="route")`

```bash
navi explore data route
```

All vuln paths — `path, plugin_id, asset_uuid`.

`navi_explore_data(subcommand="paths")`

```bash
navi explore data paths
```

### `explore data` → natural language

| User says | Tool call |
|---|---|
| "which assets have CVE-XXXX" | `navi_explore_data(subcommand="cve", cve="CVE-XXXX")` |
| "what can be exploited right now" | `navi_explore_data(subcommand="exploit")` |
| "find assets with Apache findings" | `navi_explore_data(subcommand="name", name="Apache")` |
| "find assets with Log4j in output" | `navi_explore_data(subcommand="output", output="Log4j")` |
| "find by IAVA / CISA / Bugtraq" | `navi_explore_data(subcommand="xrefs", xref_type=<type>)` |
| "show Docker hosts" | `navi_explore_data(subcommand="docker")` |
| "find web apps" | `navi_explore_data(subcommand="webapp")` |
| "which assets failed cred auth" | `navi_explore_data(subcommand="creds")` |
| "which assets are slow to scan" | `navi_explore_data(subcommand="scantime", minutes=30)` |
| "what software is in my environment" | `navi_explore_data(subcommand="software")` |
| "show compliance / audit results" | `navi_explore_data(subcommand="audits")` |
| "show routes" | `navi_explore_data(subcommand="route")` |
| "show vuln paths" | `navi_explore_data(subcommand="paths")` |
| "show everything about asset X" | `navi_explore_data(subcommand="asset", asset=<ip_or_uuid>)` |
| "show me plugin 12345 across the fleet" | `navi_explore_data(subcommand="plugin", plugin_id=12345)` |

---

## `navi_explore_query` — raw SQL against navi.db

Use when `navi_explore_data` doesn't have a subcommand for what you want,
or when you need custom columns, joins, or aggregations.

Before composing a query, check the relevant table's schema via
`navi://schema/{table}` rather than guessing column names or running
`SELECT * FROM {table} LIMIT 1`.

### Useful query patterns

```sql
-- Top exploitable assets
SELECT asset_ip, plugin_name FROM vulns
WHERE state='active'
ORDER BY severity DESC LIMIT 20;

-- Assets with criticals, sorted
SELECT asset_ip, count(*) AS crits FROM vulns
WHERE severity='critical'
GROUP BY asset_ip ORDER BY crits DESC;

-- Certs expiring soonest
SELECT common_name, not_valid_after FROM certs
ORDER BY not_valid_after LIMIT 20;

-- DISTINCT vuln paths — true remediator workload
SELECT DISTINCT path, asset_uuid FROM vuln_paths
WHERE path LIKE '%jenkins%';

-- Workload reduction: raw vs distinct
SELECT count(*) AS raw FROM vuln_paths;
SELECT count(DISTINCT path) AS distinct_locations FROM vuln_paths;

-- Software inventory ranked by asset count
SELECT software_string, count(*) AS assets FROM software
GROUP BY software_string ORDER BY assets DESC LIMIT 20;

-- EPSS — prioritize by exploit probability, not just severity
-- (requires epss table — downloaded separately from the EPSS CSV)
SELECT e.epss_value, v.asset_ip, v.plugin_name, v.severity
FROM vulns v
INNER JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
WHERE e.epss_value > 0.5
ORDER BY e.epss_value DESC LIMIT 20;

-- Highest EPSS score per asset
SELECT v.asset_ip, MAX(e.epss_value) AS highest_epss
FROM vulns v JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
GROUP BY v.asset_ip ORDER BY highest_epss DESC LIMIT 10;

-- Fixed vulns — verify remediation was confirmed
SELECT asset_ip, plugin_name, severity, last_found FROM fixed
ORDER BY last_found DESC LIMIT 20;

-- Remediation velocity per asset
SELECT asset_ip, count(*) AS fixed_count FROM fixed
GROUP BY asset_ip ORDER BY fixed_count DESC;

-- Plugin lookup — what does this plugin do, what CVEs, what xrefs
SELECT plugin_id, plugin_name, severity, cves FROM plugins
WHERE plugin_id='10863';

-- All plugins with CISA xrefs (what xrefs="CISA" uses under the hood)
SELECT plugin_id, plugin_name FROM plugins
WHERE xrefs LIKE '%CISA%' LIMIT 20;
```

For schema references and table-by-table populate commands, see navi-core.

---

## `navi_explore_query` — reads and writes

**Reads** (statements starting with `SELECT` or `WITH`) are the default.
Call freely, no confirmation needed.

**Writes** (`CREATE INDEX`, `UPDATE`, `DELETE`, DDL) work too, but require
`confirm=True`. These modify navi.db, not the Tenable platform. Worst case
outcome is a stale local cache, recoverable via
`navi_config_update(kind=...)`. Because the effect is local and
recoverable, the platform-write environment gate
(`NAVI_MCP_ALLOW_WRITES=1`) does not apply — `confirm=True` alone is the
signal of write intent.

Claude narrates the effect in prose before running any non-SELECT statement:

> I'll add an index on `vulns.plugin_id` to speed up plugin-based tagging.
> This modifies navi.db (not the Tenable platform); indexes persist until
> navi.db is rebuilt.
>
> `navi_explore_query(sql="CREATE INDEX IF NOT EXISTS idx_vulns_plugin ON vulns(plugin_id);", confirm=True)`

Full write-gate ceremony — prose + confirmation + `NAVI_MCP_ALLOW_WRITES=1`
— still applies to the tools that change Tenable platform state:
`navi_enrich_tag`, `navi_enrich_acr`, `navi_enrich_add`, `navi_scan`
(create/start/stop), `navi_was` (scan/start/upload), `navi_action_delete`,
`navi_action_rotate`, `navi_action_cancel`, `navi_config(kind="url")`.

---

## `navi_explore_info` — live platform state

These read directly from the Tenable API — always current, not from
navi.db. Use when you need IDs, live status, or to verify current
platform state.

### Account & status

| Purpose | Tool call | CLI |
|---|---|---|
| Account status, license, asset counts | `navi_explore_info(subcommand="status")` | `navi explore info status` |
| Current SLA metrics | `navi_explore_info(subcommand="sla")` | `navi explore info sla` |
| Current navi version | `navi_explore_info(subcommand="version")` | `navi explore info version` |
| Activity log: actor + action | `navi_explore_info(subcommand="logs")` | `navi explore info logs` |
| Permission assignments | `navi_explore_info(subcommand="permissions")` | `navi explore info permissions` |
| Auth info for a specific user | `navi_explore_info(subcommand="auth")` | `navi explore info auth` |

### Users & access

| Purpose | Tool call | CLI |
|---|---|---|
| All users (check before adding — re-enables if disabled) | `navi_explore_info(subcommand="users")` | `navi explore info users` |
| User group list | `navi_explore_info(subcommand="user_groups")` | `navi explore info user-groups` |

### Scanners & agents

| Purpose | Tool call | CLI |
|---|---|---|
| All scanners + status + network assignment | `navi_explore_info(subcommand="scanners")` | `navi explore info scanners` |
| Networks with scanner counts | `navi_explore_info(subcommand="networks")` | `navi explore info networks` |
| All Nessus agents | `navi_explore_info(subcommand="agents")` | `navi explore info agents` |
| Agent groups + membership | `navi_explore_info(subcommand="agent_groups")` | `navi explore info agent-groups` |

### Scans & policies

| Purpose | Tool call | CLI |
|---|---|---|
| All scans + IDs (needed for `navi_scan` start/evaluate) | `navi_explore_info(subcommand="scans")` | `navi explore info scans` |
| Currently running scans | `navi_explore_info(subcommand="running")` | `navi explore info running` |
| Saved policies (needed for `navi_scan(policy_id=...)`) | `navi_explore_info(subcommand="policies")` | `navi explore info policies` |
| Scan policy templates | `navi_explore_info(subcommand="templates")` | `navi explore info templates` |
| Saved credentials + UUIDs (needed for `credential_uuid=...`) | `navi_explore_info(subcommand="credentials")` | `navi explore info credentials` |
| Target groups | `navi_explore_info(subcommand="target_groups")` | `navi explore info target-groups` |
| Scan exclusions | `navi_explore_info(subcommand="exclusions")` | `navi explore info exclusions` |

### Tags & assets

| Purpose | Tool call | CLI |
|---|---|---|
| Tag categories + UUIDs | `navi_explore_info(subcommand="categories")` | `navi explore info categories` |
| All tag category/value pairs + UUIDs | `navi_explore_info(subcommand="tags")` | `navi explore info tags` |
| All assets (last 30d, live from API) | `navi_explore_info(subcommand="assets")` | `navi explore info assets` |
| Licensed asset count | `navi_explore_info(subcommand="licensed")` | `navi explore info licensed` |
| Custom attribute definitions | `navi_explore_info(subcommand="attributes")` | `navi explore info attributes` |
| Export job status | `navi_explore_info(subcommand="exports")` | `navi explore info exports` |
| Tenable One Exposure (TONE) objects | `navi_explore_info(subcommand="tone")` | `navi explore info tone` |

**Note on underscored subcommands**: `agent_groups`, `target_groups`, and
`user_groups` use underscores in the tool's subcommand Literal, but the
underlying CLI command uses hyphens (`agent-groups`, etc.). The server
handles the conversion — use underscores in tool calls and hyphens in
standalone CLI use.

### `explore info` → natural language

| User says | Tool call |
|---|---|
| "account status / how many licensed assets" | `navi_explore_info(subcommand="status")` |
| "are we meeting SLA" | `navi_explore_info(subcommand="sla")` |
| "who made changes / audit log" | `navi_explore_info(subcommand="logs")` |
| "list all users / who has access" | `navi_explore_info(subcommand="users")` |
| "show scanners" | `navi_explore_info(subcommand="scanners")` |
| "list scans / I need a scan ID" | `navi_explore_info(subcommand="scans")` |
| "what's scanning right now" | `navi_explore_info(subcommand="running")` |
| "list policies / I need a policy ID" | `navi_explore_info(subcommand="policies")` |
| "list credentials / I need a cred UUID" | `navi_explore_info(subcommand="credentials")` |
| "show tags / tag categories" | `navi_explore_info(subcommand="tags")` |
| "show agents" | `navi_explore_info(subcommand="agents")` |
| "show exclusions" | `navi_explore_info(subcommand="exclusions")` |

### Prerequisite lookups (suggest proactively)

| Before running… | Run first… |
|---|---|
| `navi_scan(subcommand="create", scanner_id=<id>, ...)` | `navi_explore_info(subcommand="scanners")` |
| `navi_scan(subcommand="create", policy_id=<id>, ...)` | `navi_explore_info(subcommand="policies")` |
| `navi_scan(subcommand="create", credential_uuid=<uuid>, ...)` | `navi_explore_info(subcommand="credentials")` |
| `navi_scan(subcommand="start"/"stop"/"evaluate", ...)` | `navi_explore_info(subcommand="scans")` |
| Any tag UUID for API/access groups | `navi_explore_info(subcommand="categories")` |
| Before adding a user (check if already exists) | `navi_explore_info(subcommand="users")` |
