---
name: navi-was
description: >
  Web Application Scanning skill for Tenable navi CLI. Use for ANY request involving
  Tenable WAS (DAST / Dynamic Application Security Testing): downloading WAS data,
  listing WAS configs/scans/findings, launching WAS scans, WAS statistics, WAS exports,
  and WAS-driven tagging. Trigger on: "WAS", "web application scan", "DAST",
  "web app findings", "web app vulnerabilities", "OWASP findings", "scan a web app",
  "which web apps have criticals", "show WAS configs". Requires Tenable WAS license
  (separate from TVM). WAS data goes into navi.db apps and findings tables via
  navi_config_update(kind="was") under navi-mcp, or `navi config update was` standalone.
---

# Navi WAS — Web Application Scanning Integration

Tenable WAS = DAST (Dynamic Application Security Testing) for web applications.
**Requires a Tenable WAS license** separate from TVM.

When running under navi-mcp, use tool-invocation form (shown first in each
example). Bash forms are standalone CLI equivalents for readers outside
an MCP context. See navi-mcp for the conventions around write-gated calls
(`scan`, `start`, `upload`) and the MCP → CLI handoff pattern.

---

## Prerequisites — get WAS data into navi.db

Populates the `apps` table (scan summaries) and the `findings` table
(per-finding detail). Takes significant time depending on app count and
scan history depth; the tool call is fine for targeted refreshes, though
a first-run on a tenant with long WAS history is better handled by a full
CLI sync.

`navi_config_update(kind="was")`

```bash
navi config update was
```

Verify the sync succeeded:

`navi_explore_query(sql="SELECT count(*) FROM apps;")`

```bash
navi explore data query "SELECT count(*) FROM apps;"
```

---

## The 3-step WAS exploration workflow

Drill-down chain — each step's output feeds the next.

**Step 1: list all WAS scan configurations** — shows config name, config
UUID, target URL. Use the returned `config_id` in step 2.

`navi_was(subcommand="configs")`

```bash
navi was configs
```

**Step 2: list scan runs for a config** — shows scan IDs, completion time,
status for each run. Use the returned `scan_id` in step 3.

`navi_was(subcommand="scans", config_id="<config_id>")`

```bash
navi was scans --config <config_id>
```

**Step 3: show findings for a scan run** — shows each finding, severity,
OWASP category, solution. This is the list you hand to the dev team.

`navi_was(subcommand="details", scan_id="<scan_id>")`

```bash
navi was details --scan <scan_id>
```

**Use this sequence whenever someone asks:** "show me WAS findings," "what
did the WAS scan find?", "show web app vulnerabilities for [app]."

---

## Scan control

Ad-hoc scan against a URL. **Write-gated** — narrate the call and ask for
confirmation before launching. See navi-mcp for the convention.

`navi_was(subcommand="scan", target="https://app.company.com", confirm=True)`

```bash
navi was scan --target "https://app.company.com"
```

Start a saved WAS config. **Write-gated.** Get `config_id` first via
`navi_was(subcommand="configs")`.

`navi_was(subcommand="start", config_id="<config_id>", confirm=True)`

```bash
navi was start --config <config_id>
```

Statistics across all WAS scans (read-only).

`navi_was(subcommand="stats")`

```bash
navi was stats
```

Export WAS data to CSV (read-only).

`navi_was(subcommand="export")`

```bash
navi was export
```

Upload a completed scan file. **Write-gated.**

`navi_was(subcommand="upload", file="<path/to/scan/file>", confirm=True)`

```bash
navi was upload --file <path/to/scan/file>
```

---

## WAS database queries

Once `navi_config_update(kind="was")` has run, query the `apps` and
`findings` tables directly. Schemas are also available via
`navi://schema/apps` and `navi://schema/findings`.

**apps table** — one row per completed WAS scan config: `name, uuid, target,
scan_completed_time, pages_audited, pages_crawled, requests_made,
critical_count, high_count, medium_count, low_count, info_count, owasp,
tech_list, config_id`

**findings table** — per-finding detail: `uuid, config_id (FK→apps),
plugin_id, plugin_name, severity, output, solution, scan_completed_time`

Run queries via `navi_explore_query(sql=...)`:

```sql
-- Top apps by critical count
SELECT name, target, critical_count, high_count FROM apps
ORDER BY critical_count DESC LIMIT 10;

-- Apps with any criticals
SELECT name, target, critical_count FROM apps
WHERE critical_count > 0;

-- OWASP category breakdown
SELECT owasp, count(*) AS count FROM apps
WHERE owasp != '' GROUP BY owasp ORDER BY count DESC;

-- Technology stack per app
SELECT name, target, tech_list FROM apps
WHERE tech_list != '';

-- All findings for a specific app (join apps → findings)
SELECT f.plugin_name, f.severity, f.solution
FROM findings f JOIN apps a ON f.config_id = a.config_id
WHERE a.name = 'MyApp' ORDER BY f.severity;
```

**Export all WAS findings to CSV for the dev team:**

`navi_export(subcommand="query", sql="SELECT a.name, a.target, f.plugin_name, f.severity, f.solution FROM findings f JOIN apps a ON f.config_id = a.config_id ORDER BY a.name, f.severity;")`

---

## WAS tagging

Bridge WAS findings into TVM asset tags using `apps.uuid` in the `query`
argument. All tag writes are write-gated — narrate and confirm before
calling. Use `remove=True` on health/risk tags so they always reflect
current WAS scan state.

**Tag all assets with critical WAS findings:**

`navi_enrich_tag(category="WAS Risk", value="Critical", query="SELECT uuid FROM apps WHERE critical_count > 0;", remove=True, confirm=True)`

```bash
navi enrich tag --c "WAS Risk" --v "Critical" \
  --query "SELECT uuid FROM apps WHERE critical_count > 0;" -remove
```

**Tag by OWASP category:**

`navi_enrich_tag(category="WAS OWASP", value="Injection", query="SELECT uuid FROM apps WHERE owasp LIKE '%Injection%';", remove=True, confirm=True)`

```bash
navi enrich tag --c "WAS OWASP" --v "Injection" \
  --query "SELECT uuid FROM apps WHERE owasp LIKE '%Injection%';" -remove
```

**Tag by technology stack** (no `remove=True` — tech stack is stable
classification):

`navi_enrich_tag(category="WAS Tech", value="PHP", query="SELECT uuid FROM apps WHERE tech_list LIKE '%PHP%';", confirm=True)`

```bash
navi enrich tag --c "WAS Tech" --v "PHP" \
  --query "SELECT uuid FROM apps WHERE tech_list LIKE '%PHP%';"
```

**Tag apps with scan coverage gaps** (not scanned in 30 days):

`navi_enrich_tag(category="WAS Coverage", value="Scan Gap", query="SELECT uuid FROM apps WHERE scan_completed_time < date('now', '-30 days');", remove=True, confirm=True)`

```bash
navi enrich tag --c "WAS Coverage" --v "Scan Gap" \
  --query "SELECT uuid FROM apps WHERE scan_completed_time < date('now', '-30 days');" -remove
```

See navi-enrich for the full `remove=True` ephemeral pattern and when to use it.

---

## WAS → Natural Language

| User says | Tool call |
|---|---|
| "download WAS data / set up WAS" | `navi_config_update(kind="was")` |
| "show WAS configs / what apps are we scanning" | `navi_was(subcommand="configs")` |
| "show scans for a WAS config" | `navi_was(subcommand="scans", config_id=<id>)` |
| "show WAS findings for a scan" | `navi_was(subcommand="details", scan_id=<id>)` |
| "which web apps have the most criticals" | `navi_explore_query(sql="SELECT name, critical_count FROM apps ORDER BY critical_count DESC;")` |
| "OWASP breakdown" | `navi_explore_query(sql="SELECT owasp, count(*) FROM apps GROUP BY owasp;")` |
| "WAS stats / summary" | `navi_was(subcommand="stats")` |
| "scan this web app / URL" | `navi_was(subcommand="scan", target=<url>, confirm=True)` |
| "upload a completed WAS scan file" | `navi_was(subcommand="upload", file=<path>, confirm=True)` |
| "export WAS findings to CSV" | `navi_was(subcommand="export")` or `navi_export(subcommand="query", sql=...)` from `apps`/`findings` |
| "tag assets by WAS risk" | `navi_enrich_tag(category="WAS Risk", value="Critical", query="SELECT uuid FROM apps WHERE critical_count > 0;", remove=True, confirm=True)` |
