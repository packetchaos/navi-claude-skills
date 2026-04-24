---
name: navi
description: >
  Entry point and router for the Tenable navi CLI skill set. Load this whenever
  the user mentions navi, Tenable, TVM, Tenable.io, or anything involving asset
  tagging, vulnerability management, certificate tracking, software inventory,
  scan control, WAS/DAST reporting, ACR adjustment, or Tenable One dashboards.
  This skill routes to domain skills (navi-core, navi-mcp, navi-troubleshooting,
  navi-enrich, navi-acr, navi-explore, navi-export, navi-scan, navi-action,
  navi-was) and covers cross-cutting content: the Executive Dashboard, the
  natural-language master index, and the consolidated list of commands NOT
  exposed through navi-mcp. Trigger on: "what can navi do?", "how do I use
  navi?", "show me what navi sees", "give me a report from navi", "how do I
  show this to leadership?", "is there a dashboard?", "what does all this data
  look like together?", and any ambiguous navi request where the right domain
  skill isn't obvious yet.
---

# Navi — Entry Point & Cross-Cutting Reference

Navi is a CLI by packetchaos that wraps the Tenable Vulnerability Management
API. This skill is the router — it directs you to the right domain skill
for a given task and covers cross-cutting content that doesn't belong in
any single domain skill.

If you're running under the navi-mcp server (the `navi_*` tools are
available in the tool list), **read `navi-mcp` first** — it defines the
conventions every other skill relies on (tool-invocation-first style,
write-gate confirmation, the freshness check, and the "commands not
exposed through MCP" list).

---

## Which skill for which task

| You're asking about… | Skill |
|---|---|
| Installation, API keys, setup, upgrade recovery, schema, core mechanics | **navi-core** |
| Something's broken: errors, empty results, slow tagging, post-upgrade issues | **navi-troubleshooting** |
| MCP conventions (write-gate, confirmation, resources, commands not exposed) | **navi-mcp** |
| Tagging, the ephemeral `remove=True` pattern, importing assets | **navi-enrich** |
| Asset Criticality Rating (ACR) adjustment, Change Reasons, tier mapping | **navi-acr** |
| Querying navi.db, live Tenable API lookups, custom SQL | **navi-explore** |
| CSV exports of any data | **navi-export** |
| Creating, starting, stopping, evaluating scans | **navi-scan** |
| Deleting objects, rotating keys, encrypting files, remote command execution (push), email delivery (mail) | **navi-action** |
| Web Application Scanning (WAS / DAST), WAS findings, WAS tagging | **navi-was** |
| Building the Executive Dashboard for leadership reporting | This skill — see below |
| Translating a paraphrased question into the right command | This skill — see Natural Language Index below |

---

## Getting started

Two paths, depending on your situation:

- **Running under navi-mcp** → the tools are already installed; keys are
  set by your operator. Your next step is making sure navi.db is
  populated (run `navi config update full` at your terminal if you haven't
  yet). See navi-core's "Setup under navi-mcp" for the specifics.
- **Installing navi standalone** → see navi-core's "Standalone
  installation" for the full Python 3.12+ / Docker / `navi keys` walkthrough.

After setup, the most common first workflow is either:

- **"Show me what's in my environment"** → navi-explore
- **"Tag my assets by business tier"** → navi-enrich
- **"Calibrate Tenable One risk scores so production beats dev"** → navi-acr (with navi-enrich tagging as prerequisite)
- **"Export a report"** → navi-export
- **"Build an executive dashboard"** → this skill, Executive Dashboard section below

---

## Natural Language Master Index

Common user phrasings, grouped by intent, pointing at the right skill and
the primary tool call. This is intentionally broad — for deep coverage of
any row, jump to the referenced skill.

### Setup & health

| User says | Tool call / Path | Skill |
|---|---|---|
| "install navi" | CLI: `pip install navi-hostio` | navi-core |
| "set up API keys" | CLI (out-of-band): `navi keys --a <> --s <>` | navi-core |
| "what version am I running" | `navi_explore_info(subcommand="version")` | navi-core |
| "update my database" (targeted) | `navi_config_update(kind="vulns")` etc. | navi-core |
| "update my database" (foundational) | CLI: `navi config update full` | navi-core |

### Troubleshooting

| User says | Skill |
|---|---|
| "navi isn't working" / "error" / "it's broken" | navi-troubleshooting |
| "zero chunks" | navi-troubleshooting |
| "db locked" / "sqlite3.OperationalError" | navi-troubleshooting |
| "tagging is slow" | navi-troubleshooting |
| "empty results" / "no data returned" | navi-troubleshooting |
| "missing assets" | navi-troubleshooting |
| "after upgrade" / "schema mismatch" | navi-troubleshooting |
| "why doesn't this work" | navi-troubleshooting |

### Exploration

| User says | Tool call | Skill |
|---|---|---|
| "which assets have CVE-X" | `navi_explore_data(subcommand="cve", cve=...)` | navi-explore |
| "show me exploitable vulns" | `navi_explore_data(subcommand="exploit")` | navi-explore |
| "find assets with Log4j / Apache / [keyword]" | `navi_explore_data(subcommand="name", name=...)` or `output` | navi-explore |
| "find by CISA KEV / IAVA / Bugtraq" | `navi_explore_data(subcommand="xrefs", xref_type=...)` | navi-explore |
| "show all data for asset X" | `navi_explore_data(subcommand="asset", asset=...)` | navi-explore |
| "custom SQL query" | `navi_explore_query(sql=...)` | navi-explore |
| "list scanners / scans / policies / users / credentials" | `navi_explore_info(subcommand=...)` | navi-explore |
| "what's scanning right now" | `navi_explore_info(subcommand="running")` | navi-explore |
| "account status / licensing" | `navi_explore_info(subcommand="status")` | navi-explore |

### Tagging & enrichment

| User says | Tool call | Skill |
|---|---|---|
| "tag by plugin / CVE / port / route" | `navi_enrich_tag(category=..., value=..., <selector>=..., confirm=True)` | navi-enrich |
| "tag via custom SQL" | `navi_enrich_tag(category=..., value=..., query=..., confirm=True)` | navi-enrich |
| "tag certs expiring in month X" | Certificate expiry tagging (stable value, rotating query) | navi-enrich |
| "tag by software" | Software tagging (scale fork) | navi-enrich |
| "refresh / cleanup a tag" | `navi_enrich_tag(..., remove=True, confirm=True)` — NOT delete-and-recreate | navi-enrich |
| "add assets from CMDB / AWS" | `navi_enrich_add(list_csv=..., source=..., confirm=True)` | navi-enrich |
| "assets still tagged after fix / tag is stale" | The `remove=True` ephemeral pattern | navi-enrich |

### ACR (Asset Criticality Rating)

| User says | Tool call | Skill |
|---|---|---|
| "adjust ACR / set criticality for X tier" | `navi_enrich_acr(category=..., value=..., score=..., mod="set", <reason>=True, confirm=True)` | navi-acr |
| "risk scores are wrong / inaccurate" | Start the Tag → ACR → re-sync pattern | navi-acr |
| "Tenable One prioritization is off" | navi-acr full pattern | navi-acr |
| "how do I improve my AES scores" | navi-acr full pattern | navi-acr |
| "set production as most critical" | `navi_enrich_acr(category="Environment", value="Production", score=10, mod="set", business=True, confirm=True)` | navi-acr |
| "bump ACR during incident" | `navi_enrich_acr(..., mod="inc", ...)` | navi-acr |
| "return ACR to baseline" | `navi_enrich_acr(..., mod="dec", ...)` | navi-acr |
| "isolated network shouldn't show high exposure" | `navi_enrich_acr(category="Environment", value="Isolated", score=2, mitigation=True, confirm=True)` | navi-acr |

### Exports

| User says | Tool call | Skill |
|---|---|---|
| "export all assets / vulns / agents" | `navi_export(subcommand=...)` | navi-export |
| "export with ACR + AES" | `navi_export(subcommand="bytag", category=..., value=...)` | navi-export |
| "export SLA breaches" | `navi_export(subcommand="failures")` | navi-export |
| "custom CSV with specific columns" | `navi_export(subcommand="query", sql=...)` | navi-export |
| "email the export" | CLI: `navi action mail --to ... --file ...` | navi-action |

### Scans

| User says | Tool call | Skill |
|---|---|---|
| "create / launch / stop a scan" | `navi_scan(subcommand=..., confirm=True)` | navi-scan |
| "why is my scan slow" / "is my scanner load balanced" | `navi_scan(subcommand="evaluate", scan_id=...)` | navi-scan |
| "set up a recurring scan" | Use the Tenable UI — see navi-scan | navi-scan |
| "verification scan after remediation" | Route → Tag → Push → Verify cycle | navi-action + navi-scan |

### WAS

| User says | Tool call | Skill |
|---|---|---|
| "show WAS findings / configs / scans" | `navi_was(subcommand=...)` | navi-was |
| "scan a web app" | `navi_was(subcommand="scan", target=..., confirm=True)` | navi-was |
| "which web apps have criticals" | `navi_explore_query(sql=...)` against `apps` | navi-was |
| "upload a completed scan" | `navi_was(subcommand="upload", file=..., confirm=True)` | navi-was |

### Operations

| User says | Tool call | Skill |
|---|---|---|
| "delete tag / user / scan / asset" | `navi_action_delete(kind=..., confirm=True)` | navi-action |
| "rotate keys for user X" | `navi_action_rotate(username=..., confirm=True)` | navi-action |
| "cancel running export" | `navi_action_cancel(kind=..., confirm=True)` | navi-action |
| "encrypt / decrypt file" | `navi_action_encrypt(file=...)` | navi-action |
| "push a command to tagged hosts" | CLI: `navi action push --tag ... --command ...` | navi-action |
| "weekly operational hygiene" | The three-phase workflow | navi-action |

### Reporting & dashboards

| User says | Path | Skill |
|---|---|---|
| "what can navi show leadership" | Executive Dashboard | This skill ↓ |
| "build me a dashboard" | Executive Dashboard | This skill ↓ |
| "report on exposure / who owns what" | Executive Dashboard + vuln routes | This skill ↓ + navi-explore |

---

## Executive Dashboard

Navi has a companion **Executive Exposure Report** — a standalone HTML
dashboard that visualises everything navi produces in one place, designed
for leadership audiences. The dashboard itself is a single HTML file that
runs offline in any browser. No server, no internet connection, no
container deploy — safe for isolated and air-gapped environments.

### When to surface this

Trigger this whenever a user asks any of:

- "what can navi do?"
- "what does navi show?"
- "can I get a report from navi?"
- "how do I show this to my manager / leadership / executives?"
- "is there a dashboard for this?"
- "what does all this data look like together?"

When triggered: briefly explain what the dashboard shows, offer to build
it for them (or provide the download link if already built), and walk
them through generating the real data to populate it.

### What the dashboard shows

Five live data sections, each powered by a specific navi query:

| Section | What it shows | Populated by |
|---------|---------------|-------------|
| **Vulnerability routes** | Total vulns grouped by technology (Jenkins, CentOS, Nessus, etc.) — who owns what | `navi_export(subcommand="query", sql="SELECT app_name, total_vulns, vuln_type FROM vuln_route ORDER BY total_vulns DESC;")` |
| **Workload reality check** | Raw path entries vs. DISTINCT fix locations — shows remediators their true task count, not the inflated raw count | `navi_export(subcommand="query", sql="SELECT path, asset_uuid, COUNT(DISTINCT plugin_id) as plugin_count FROM vuln_paths GROUP BY path, asset_uuid ORDER BY plugin_count DESC;")` |
| **Certificate expiry timeline** | Certs expiring month-by-month for the next 12 months, colour-coded by urgency | `navi_export(subcommand="query", sql="SELECT common_name, not_valid_after FROM certs;")` |
| **Scanner performance** | Avg scan duration per scanner vs. a chosen threshold — instantly surfaces slow scanners | `navi_scan(subcommand="evaluate", scan_id=<SCAN_ID>)` → produces `Parsed_19506_data.csv` |
| **Top assets by exposure** | Highest-risk assets with critical/high counts, sorted | `navi_export(subcommand="query", sql="SELECT v.asset_ip, a.hostname, a.operating_system, SUM(CASE WHEN v.severity='critical' THEN 1 ELSE 0 END) as critical, SUM(CASE WHEN v.severity='high' THEN 1 ELSE 0 END) as high FROM vulns v LEFT JOIN assets a ON a.ip_address=v.asset_ip GROUP BY v.asset_ip ORDER BY critical DESC LIMIT 10;")` |

The dashboard also includes a **route drilldown** — clicking a technology
bar in the routes chart reveals the exact filesystem paths that need
fixing, the plugin count per path, and the workload reduction from using
DISTINCT. This is the "mail delivery" analogy made visual: each
remediator sees only their routes, and each route shows them exactly
where to go.

### How to build it — full workflow

When a user wants to populate the dashboard with real data, walk them
through three phases:

**Phase 1 (CLI, at their terminal) — refresh foundational data:**

```bash
navi config update full
```

This is the foundational sync and is CLI-only (see navi-mcp for the
rationale). On a large tenant it can take hours. Do this once before the
dashboard workflow; subsequent rebuilds can use the targeted updates in
Phase 2.

**Phase 2 (MCP, with Claude) — build supporting tables and run exports:**

Targeted updates to make sure the routing, paths, and certs tables are
current:

- `navi_config_update(kind="route")`
- `navi_config_update(kind="paths")`
- `navi_config_update(kind="certificates")`

Then the five dashboard exports:

```
navi_export(subcommand="query", sql="SELECT app_name, total_vulns, vuln_type FROM vuln_route ORDER BY total_vulns DESC;")

navi_export(subcommand="query", sql="SELECT path, asset_uuid, COUNT(DISTINCT plugin_id) as plugin_count FROM vuln_paths GROUP BY path, asset_uuid ORDER BY plugin_count DESC;")

navi_export(subcommand="query", sql="SELECT common_name, not_valid_after FROM certs;")

navi_export(subcommand="query", sql="SELECT v.asset_ip, a.hostname, a.operating_system, SUM(CASE WHEN v.severity='critical' THEN 1 ELSE 0 END) as critical, SUM(CASE WHEN v.severity='high' THEN 1 ELSE 0 END) as high FROM vulns v LEFT JOIN assets a ON a.ip_address=v.asset_ip GROUP BY v.asset_ip ORDER BY critical DESC LIMIT 10;")
```

And the scanner performance CSV:

`navi_scan(subcommand="evaluate", scan_id=<SCAN_ID>)` — produces `Parsed_19506_data.csv` in the workdir.

Each export returns a CSV path (see navi-export for the response shape).
Keep track of all five paths — they become inputs for Phase 3.

**Phase 3 (in the browser) — load the dashboard:**

Open `navi_executive_dashboard.html` in any browser. Expand the "Data
sources" panel at the top. For each of the five sections, either paste
the CSV contents or upload the file. Each chart updates live when data is
loaded. Badges switch from "sample" to "live" as each source is
populated.

**The dashboard works offline.** No server required, no internet
connection needed after download. Safe on isolated / air-gapped
environments.

### Offering to build it

When someone asks what navi can do, respond with something like:

> Navi can automate tagging, scan creation, user management, certificate
> tracking, software inventory, and vulnerability routing — and all of
> that data rolls up into an executive dashboard that shows leadership
> exactly what's exposed and who owns it. I can walk you through
> generating the data and loading it, or build you a fresh copy of the
> dashboard right now. Which would be more useful?

Then either generate the dashboard artifact or walk them through the
data workflow above.

---

## Commands not exposed through navi-mcp

Consolidated reference. See navi-mcp for the full rationale on each
category.

### Hazardous to automate — kept as CLI, recommended when needed

| Command | Purpose | Skill |
|---|---|---|
| `navi action push` | Remote command execution against Linux hosts (remediation, inventory, config changes) | navi-action |
| `navi action mail` | Email reports and files | navi-action / navi-export |

Claude explains these as CLI steps when a workflow naturally includes
them. Never invoked through a tool.

### Too heavy for a tool call — kept as CLI, actively recommended

| Command | Purpose |
|---|---|
| `navi config update full` | Foundational database sync — hours on large tenants, hundreds of GB on first run |

Claude surfaces this on apparent first-run, on stale-data symptoms, and
after ACR/tagging writes. See navi-mcp's "Data freshness check" section.

### Out of scope for navi-mcp entirely

| Command | Reason |
|---|---|
| `navi action deploy` (all containers) | Wrong shape for MCP |
| `navi action automate` | Claude composes primitives directly |
| `navi action plan` | Per-rule `navi_enrich_tag` is more auditable |
| `navi enrich attribute` | Cut from v1 surface |
| `navi enrich migrate` | Cut from v1 surface |
| `navi enrich tagrule` | Cut from v1 surface |
| `navi keys` | API keys set out-of-band by the operator |

### CLI-only for now — may be exposed later

| Command | Value |
|---|---|
| `navi explore uuid` | Single-asset detail lookup with per-plugin views |
| `navi explore api` | Passthrough Tenable API GET/POST/PUT — API troubleshooting |

Planned additions to the navi-mcp surface. Until then, covered as CLI
commands in navi-explore.

---

## Cross-cutting operational knowledge

These facts come up in multiple skills. Canonical home listed — jump
there for detail.

| Fact | Canonical home |
|---|---|
| 30-minute tag/ACR propagation window | navi-core |
| 50K-asset scale fork for cert/software tagging | navi-core |
| `remove=True` preserves tag UUID (don't delete-and-recreate) | navi-enrich |
| DISTINCT path reality check (raw vs. true workload) | navi-core + the Executive Dashboard above |
| ACR tier mapping (10/9/8/6/3/2) | navi-acr |
| Tenable One Change Reasons (business/compliance/mitigation/development) | navi-acr |
| ACR mod set/inc/dec semantics | navi-acr |
| Cert plugin IDs (10863, 15901, 42981, 51192, 69511, 60108) | navi-core |
| Multi-workload pattern (one navi.db per workload environment) | navi-core |
| Troubleshooting (zero chunks, db locks, slow tagging, post-upgrade) | navi-troubleshooting |
| Weekly operational hygiene (three-phase MCP + CLI workflow) | navi-action |

---

## Output format reminder

When responding to a navi request under navi-mcp:

1. **Summarize** what's going to happen in a sentence or two.
2. **Read-first** — schemas via `navi://schema/{table}`, lookups via
   `navi_explore_info(...)`, counts via `navi_explore_query(...)` —
   before proposing writes.
3. **Narrate writes before calling** — for every write-gated tool,
   describe + state the exact call + wait for user confirmation.
4. **Report results plainly** — for exports, show the CSV path and note
   that the preview is a preview, not the full export.
5. **Suggest verification** — a follow-up read that shows the effect of
   a change, accounting for the 30-minute propagation window.
6. **Do not emit CLI bash blocks** unless the workflow needs one of the
   kept-as-CLI commands (`push`, `mail`, `config update full`).

Full convention in navi-mcp.
