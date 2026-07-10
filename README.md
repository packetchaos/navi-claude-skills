# navi-skills

A Claude skill set for the [Tenable **navi**](https://github.com/packetchaos/navi)
CLI and the `navi_*` MCP tools. A router skill plus focused sub-skills cover
setup/core mechanics, asset tagging, data exploration, CSV export, ACR
calibration, scan control, actions, email, remote execution, WAS/DAST, and
troubleshooting.

Skill folders live at the repo root (with `.claude-plugin/plugin.json`), matching
this repo's established layout.

## Skills (13)

| Skill | Purpose |
|---|---|
| `navi` | Router — directs to the right domain skill; cross-cutting content |
| `navi-core` | Install, API keys, DB sync, schema, core mechanics |
| `navi-mcp` | Conventions for driving navi through the navi-mcp server |
| `navi-troubleshooting` | Errors, empty results, slow tagging, post-upgrade recovery |
| `navi-enrich` | Asset tagging (all selectors), the ephemeral `-remove` pattern, UUID preservation |
| `navi-explore` | Query/inspect navi.db and live Tenable state |
| `navi-export` | CSV exports of any data |
| `navi-acr` | Asset Criticality Rating calibration |
| `navi-scan` | Create / start / stop / evaluate scans |
| `navi-action` | Delete, rotate keys, cancel exports, encrypt/decrypt |
| `navi-mail` | Email a report/file via `navi_action_mail` (double-gated) |
| `navi-remote-exec` | Run a command / push a file to a host via `navi_action_push` (double-gated) |
| `navi-was` | Web Application Scanning (WAS / DAST) |

Each skill is a folder with `SKILL.md` (and `references/` for deeper material).

## What's new in this update (v0.2.0)

- **Two new skills:** `navi-mail` (`navi_action_mail`) and `navi-remote-exec`
  (`navi_action_push`) — both double-gated tools exposed by the navi-mcp server
  (`NAVI_EMAIL=1` / `NAVI_REMOTE_CODE_EXECUTION=1`, each on top of
  `NAVI_MCP_ALLOW_WRITES=1`, plus per-call `confirm=True`).
- **`-remove` semantics fix (navi-enrich):** `-remove` is a CLEAR of a tag's
  current membership (it ignores selectors). The accurate ephemeral refresh is
  two steps — clear (`remove=True`, no selector) → wait ~30 min → re-apply the
  selector with no `remove`. Combining them in one call is allowed but only
  adds/updates against current membership; the clear step is proportional to
  churn (stable tags like `Hardware:iDRAC` rarely need it).
- **push/mail are tools now:** the `navi-action`, `navi-mcp`, `navi` (router),
  `navi-core`, and `navi-export` skills were updated so nothing still calls them
  "CLI-only."
- **Command-name cleanup:** bare pre-7.5.x forms (`navi mail`, `navi push`) were
  corrected to `navi action mail` / `navi action push`.

## Notes

- `navi-mail` and `navi-remote-exec` require the navi-mcp server started with the
  matching capability gates. See the navi-mcp repo.
- Set `author` in `.claude-plugin/plugin.json` to your identity before publishing.
