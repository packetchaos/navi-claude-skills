---
name: navi-export
description: >
  CSV export skill for Tenable navi CLI. Use for ANY request to export data to CSV
  from navi. Covers all navi export subcommands: assets, bytag (includes ACR + AES
  scores), network, licensed, vulns, failures (SLA breaches), route (vulns by route ID),
  compliance, agents, group (agents by group), users, policy, parsed, compare, and
  query (custom SQL). Critical distinction: bytag is the ONLY export that includes
  ACR and AES scores alongside asset data — essential for risk reporting by business
  segment. Trigger on: "export to CSV", "give me a spreadsheet", "download asset list",
  "export vulnerabilities", "give me a report", "export compliance", "export by tag",
  "export users", "export agents", "export policies for migration".
---

# Navi Export — CSV Export Reference

All export commands write CSV files from navi.db. Every export subcommand is
exposed through MCP as `navi_export(subcommand=..., ...)` — no write-gating,
no confirmation required. The tool returns the CSV path, file size, row
count, header, and a short preview (first 5 rows). **The preview is a
preview, not the full export** — Claude surfaces the file path to the user
and prefers `navi_explore_query` against navi.db for further analysis.

**Data freshness matters.** CSV exports are only as current as navi.db. For
targeted refreshes between full syncs, use `navi_config_update(kind=...)`
(vulns, assets, agents, compliance, certificates, route, paths, was). For
foundational syncs and first-run setup, `navi config update full` is the
CLI command you run at your terminal — it's intentionally not exposed as
an MCP tool because first-run syncs can pull hundreds of GB and take hours.
See navi-mcp for the full stance.

When running under navi-mcp, use tool-invocation form (shown first in each
example below). The bash forms are standalone CLI equivalents for readers
outside an MCP context.

---

## Asset Exports

Full asset dump — `ip_address, hostname, fqdn, uuid, os, network, first_found,
last_found, etc.`

`navi_export(subcommand="assets")`

```bash
navi export assets
```

Assets matching a tag — **the only export with ACR + AES scores**. Use for
risk reporting by business segment, exec exposure summaries. ACR/AES are
NOT available via `navi_export(subcommand="query", ...)` — bytag only.

`navi_export(subcommand="bytag", category="Environment", value="Production")`

```bash
navi export bytag --c "Environment" --v "Production"
```

Assets in a specific network. Get network names first via
`navi_explore_info(subcommand="networks")`.

`navi_export(subcommand="network", network="Corporate")`

```bash
navi export network --network "Corporate"
```

Licensed assets only. Use for license management and billing reconciliation.

`navi_export(subcommand="licensed")`

```bash
navi export licensed
```

---

## Vulnerability Exports

Full vulnerability dump — `asset_ip, plugin_id, plugin_name, severity, output,
first_found, last_found, etc.`

`navi_export(subcommand="vulns")`

```bash
navi export vulns
```

Vulnerabilities that have failed SLA thresholds. Requires SLA thresholds
to be configured first via `navi_config(kind="sla")` — see navi-core.
Use for SLA breach reporting, escalation lists, overdue remediation.

`navi_export(subcommand="failures")`

```bash
navi export failures
```

Vulnerabilities for a specific route ID. Get route IDs first via
`navi_explore_data(subcommand="route")`. Use for handing a remediator their
exact vuln list for their technology stack.

`navi_export(subcommand="route", route_id="<ROUTE_ID>")`

```bash
navi export route --route <ROUTE_ID>
```

Parsed plugin data (normalized output). Use for feeding structured findings
to ticketing systems or SIEMs.

`navi_export(subcommand="parsed")`

```bash
navi export parsed
```

CVE-focused cross-asset comparison. Pulls CVE data from each plugin into a
structured CSV. Use for cross-environment CVE comparison and CVE
remediation tracking.

`navi_export(subcommand="compare")`

```bash
navi export compare
```

---

## Compliance Export

Compliance check results — `asset_uuid, check_name, status
(passed/failed/warning), audit_file, plugin_id`. Requires the compliance
table to be populated first via `navi_config_update(kind="compliance")`.
Use for audit evidence, CIS/STIG compliance reports, failed control lists.

`navi_export(subcommand="compliance")`

```bash
navi export compliance
```

---

## Agent Exports

All agent data — `name, IP, UUID, status, group, platform, last_connect,
version, etc.`

`navi_export(subcommand="agents")`

```bash
navi export agents
```

Agents in a specific group. Get group names first via
`navi_explore_info(subcommand="agent_groups")`.

`navi_export(subcommand="group", group_name="Production Servers")`

```bash
navi export group --name "Production Servers"
```

---

## User & Policy Exports

All users with roles and permissions — `username, email, permission level,
role, enabled/disabled`. Use for access reviews and offboarding audits.

`navi_export(subcommand="users")`

```bash
navi export users
```

Scan policies. Use for policy migration between Tenable instances and
policy backup.

`navi_export(subcommand="policy")`

```bash
navi export policy
```

---

## Custom Export (SQL)

Full control over columns, joins, and filters. Does NOT include ACR/AES —
use `navi_export(subcommand="bytag", ...)` for those.

When composing a custom query, check the relevant schema first via the
`navi://schema/{table}` resource rather than guessing column names.

`navi_export(subcommand="query", sql="SELECT ...")`

```bash
navi export query "SELECT ..."
```

**Example queries:**

```sql
-- Critical vulns by asset
SELECT asset_ip, plugin_name, severity FROM vulns
WHERE severity='critical' ORDER BY asset_ip;

-- Asset + vuln join filtered by network
SELECT a.hostname, v.plugin_name FROM vulns v
JOIN assets a ON a.ip_address = v.asset_ip
WHERE a.network = 'DMZ';

-- Certs expiring in a given year
SELECT common_name, not_valid_after FROM certs
WHERE not_valid_after LIKE '%2026%';

-- Workload by route
SELECT app_name, total_vulns FROM vuln_route
ORDER BY total_vulns DESC;

-- Distinct fix locations by path
SELECT path, COUNT(DISTINCT plugin_id) AS vulns FROM vuln_paths
GROUP BY path ORDER BY vulns DESC;

-- WAS findings for dev team
SELECT a.name, a.target, f.plugin_name, f.severity, f.solution
FROM findings f JOIN apps a ON f.config_id = a.config_id
ORDER BY a.name, f.severity;
```

---

## Decision Guide

| Want | Tool call |
|------|---------|
| Full asset dump | `navi_export(subcommand="assets")` |
| **Assets with ACR + AES** | `navi_export(subcommand="bytag", category=<cat>, value=<val>)` |
| Assets in a network | `navi_export(subcommand="network", network=<n>)` |
| Licensed assets | `navi_export(subcommand="licensed")` |
| Full vuln dump | `navi_export(subcommand="vulns")` |
| **Overdue / SLA breaches** | `navi_export(subcommand="failures")` (needs SLA configured) |
| Vulns for one technology route | `navi_export(subcommand="route", route_id=<id>)` |
| Compliance / audit evidence | `navi_export(subcommand="compliance")` (needs `navi_config_update(kind="compliance")`) |
| Agent inventory | `navi_export(subcommand="agents")` |
| Agents in a group | `navi_export(subcommand="group", group_name=<group>)` |
| User list + roles | `navi_export(subcommand="users")` |
| Policies for migration | `navi_export(subcommand="policy")` |
| Parsed plugin data | `navi_export(subcommand="parsed")` |
| Cross-asset CVE comparison | `navi_export(subcommand="compare")` |
| Custom filter / joins | `navi_export(subcommand="query", sql="SELECT ...")` |

---

## Delivering a CSV to someone — `navi action mail`

`navi action mail` is CLI-only — it is not exposed as an MCP tool (see
navi-mcp's "Commands not exposed" section for the rationale). When a
workflow ends with "send this to X," Claude completes the export through
the MCP tool, reports the returned CSV path to the user, then hands off
to the CLI for mailing.

**Pattern:**

> I'll export production assets with ACR and AES scores.
>
> `navi_export(subcommand="bytag", category="Environment", value="Production")`
>
> *[tool returns: "Wrote bytag_export.csv — 1,247 rows. Columns: ip_address,
> hostname, uuid, os, network, acr, aes, ..."]*
>
> CSV is at `bytag_export.csv`. To mail it to your CISO, run this at your
> terminal:
>
> ```bash
> navi action mail --to "ciso@company.com" --file "bytag_export.csv"
> ```

Claude does not try to invoke `action mail` through any tool and does not
batch the mail step silently into an automation sequence.

---

## Natural Language → Export

| User says | Tool call |
|---|---|
| "export all assets" | `navi_export(subcommand="assets")` |
| "export production assets with risk scores" | `navi_export(subcommand="bytag", category="Environment", value="Production")` |
| "export assets in DMZ" | `navi_export(subcommand="network", network="DMZ")` |
| "export licensed assets" | `navi_export(subcommand="licensed")` |
| "export all vulns" | `navi_export(subcommand="vulns")` |
| "export overdue / failed vulns / SLA" | `navi_export(subcommand="failures")` |
| "export vulns for Jenkins route" | `navi_export(subcommand="route", route_id=<id>)` |
| "export compliance results" | `navi_export(subcommand="compliance")` |
| "export agents" | `navi_export(subcommand="agents")` |
| "export agents in a group" | `navi_export(subcommand="group", group_name=<group>)` |
| "export users / access report" | `navi_export(subcommand="users")` |
| "export policies for migration" | `navi_export(subcommand="policy")` |
| "custom export / specific columns" | `navi_export(subcommand="query", sql="SELECT ...")` |
