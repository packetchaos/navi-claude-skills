# navi-action reference — operational hygiene workflow

A realistic recurring end-to-end pattern that exercises the navi_* tools —
including the double-gated `navi_action_mail` — plus the rationale for the
phased split. Loaded on demand; the per-command detail is in SKILL.md.

## Operational hygiene workflow

A realistic end-to-end pattern that exercises MCP tools and CLI handoff
together. Designed to run on a recurring cadence as ongoing operational
hygiene.

**On cadence:** weekly is a reasonable default. Adjust to match your
tenant's scan frequency and SLA tiers. Some teams run this daily during
active incidents; others run monthly in low-change environments.

**Phase 1 (CLI, at your terminal) — refresh foundational data:**

```bash
navi config update full
```

This can take hours on a large tenant. Run it overnight or in the
background before starting the MCP portion. `navi config update full` is
kept CLI-only per navi-mcp's "too heavy for a tool call" category.

**Phase 2 (MCP, with Claude) — targeted refreshes + health tagging + export:**

Claude narrates and invokes each of these with confirmation where
write-gated:

Targeted refreshes:
- `navi_config_update(kind="route")`
- `navi_config_update(kind="paths")`
- `navi_config_update(kind="certificates")`

Ephemeral health tagging — each refreshed with `remove=True` so the tag
always reflects current state AND preserves its UUID for any downstream
access groups or dashboards:

- `navi_enrich_tag(category="Scan Health", value="Cred Failure", plugin=104410, remove=True, confirm=True)`
- `navi_enrich_tag(category="Scan Health", value="Slow Scan", scantime=30, remove=True, confirm=True)`
- `navi_enrich_tag(category="CISA", value="KEV", xrefs="CISA", remove=True, confirm=True)`

Upcoming cert expiry tag — stable value, rotating query per the current
month:

- `navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE '<current_month>%<year>%';", remove=True, confirm=True)`

SLA breach export or targeted report:

- `navi_export(subcommand="failures")`

Or a more targeted export:

- `navi_export(subcommand="query", sql="SELECT asset_ip, plugin_name, severity, last_found FROM vulns WHERE severity='critical' AND last_found < date('now', '-30 days');")`

**Phase 3 (MCP, with Claude) — deliver the report:**

Email is a tool now, but double-gated — it runs only if the server has
`NAVI_MCP_ALLOW_WRITES=1` + `NAVI_EMAIL=1`, and Claude narrates + gets
`confirm=True` first:

- `navi_action_mail(to="security-team@company.com", subject="Weekly SLA Breach Report", file="failures_export.csv", confirm=True)`

For sensitive reports, encrypt in Phase 2 (`navi_action_encrypt`) and mail the
`.enc` file here. If `NAVI_EMAIL` isn't enabled on the server, the tool blocks —
fall back to `navi action mail ...` at the terminal.

### Why the three-phase split

Each phase is deliberately in a different place:

- **Phase 1** is on your terminal because `config update full` is too
  heavy and long-running for a tool call — and because fresh foundational
  data should be a deliberate human action before automated workflows
  run against it.
- **Phase 2** is through MCP because tagging and export benefit from
  Claude's narration, write-gate confirmation, and the ability to adjust
  in-session if something looks off.
- **Phase 3** runs through MCP now, but email delivery is deliberately
  double-gated (`NAVI_EMAIL` on top of writes) and confirmed per-send, so it
  stays an explicit, human-approved action rather than something fired off
  silently.

Schedule Phase 1 via cron or your scheduler of choice. Phases 2 and 3 can be
kicked off when you're next at the chat on your chosen cadence.

