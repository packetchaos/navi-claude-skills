# navi-action reference — operational hygiene workflow

A realistic recurring end-to-end pattern that exercises the navi_* tools and
the CLI handoff (push/mail) together, plus the rationale for the three-phase
split. Loaded on demand; the per-command detail is in SKILL.md.

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

**Phase 3 (CLI, at your terminal) — deliver the report:**

```bash
navi action mail --to "security-team@company.com" \
  --subject "Weekly SLA Breach Report" --file "failures_export.csv"
```

Or encrypt first if the report contains sensitive data (compose with
`navi_action_encrypt` in Phase 2, then mail the `.enc` file in Phase 3).

### Why the three-phase split

Each phase is deliberately in a different place:

- **Phase 1** is on your terminal because `config update full` is too
  heavy and long-running for a tool call — and because fresh foundational
  data should be a deliberate human action before automated workflows
  run against it.
- **Phase 2** is through MCP because tagging and export benefit from
  Claude's narration, write-gate confirmation, and the ability to adjust
  in-session if something looks off.
- **Phase 3** is on your terminal because email delivery should be a
  deliberate human-initiated action, not something an LLM fires off.

Schedule Phase 1 and Phase 3 via cron or your scheduler of choice. Phase
2 can be kicked off when you're next at the chat on your chosen cadence.

