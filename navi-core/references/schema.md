# navi-core reference — navi.db schema

Full table-by-table schema for navi.db. Loaded on demand from navi-core.
Schemas are also available live via the `navi://schema/{table}` resource —
prefer that when composing a query, as it can't go stale. Use this file for
the complete map (which tables exist, how they're populated, how they join).

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
Populate: `navi_config_update(kind="fixed")` — a dedicated sync (CLI: `navi
config update fixed`). Required before `navi_export(subcommand="failures")` and
SLA processing.

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
Populate: `navi_config(kind="certificates")` (CLI: `navi config certificates`).
Not `config update certificates` — that subcommand does not exist.

**epss** — Exploit Prediction Scoring System scores per CVE (downloaded from EPSS CSV)
`cve (PK), epss_value, percentile`
Provides probability scores (0.0–1.0) for CVE exploitation likelihood.
Use to prioritize remediation by actual exploit probability, not just CVSS severity.
Populate: `navi config epss` at the CLI (downloads and parses the EPSS CSV; not exposed as an MCP tool)

**Key EPSS query** (join vulns to EPSS scores):
```sql
SELECT e.epss_value, v.asset_ip, v.plugin_name, v.severity
FROM vulns v
INNER JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
WHERE e.epss_value > 0.5
ORDER BY e.epss_value DESC;
```

**zipper** — merge table joining EPSS scores with the plugin DB (so EPSS data
can be associated with plugins/vulns without a per-query join).
Populate: `navi config update zipper` at the CLI (not exposed as an MCP tool;
requires the `epss` and `plugins` tables to be present first).

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
