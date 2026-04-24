---
name: navi-action
description: >
  Automation and action skill for Tenable navi CLI. Covers navi action commands
  available through navi-mcp: delete (remove tags/users/scans/assets/agents/exclusions),
  rotate (API key rotation), cancel (stop exports), encrypt/decrypt (file security).
  Also covers navi action push (remote command execution on Linux targets) and
  navi action mail (email reports/files) — both CLI-only, not exposed as MCP tools.
  Trigger on: "delete this tag", "remove stale tags", "rotate API keys", "offboard
  user", "cancel export", "encrypt a file", "send report to", "push a command to
  assets", "remediate tagged assets", "run a command across a tag group", "clean
  up tags".
---

# Navi Action — Operations & Remediation

Five action commands are exposed through MCP as tools; two (`push` and `mail`)
are kept CLI-only because they are hazardous to automate. See navi-mcp for
the full stance on the kept-as-CLI convention and the MCP → CLI handoff
pattern.

**MCP-exposed tools:**

- `navi_action_delete` — write-gated, destructive
- `navi_action_rotate` — write-gated
- `navi_action_cancel` — write-gated
- `navi_action_encrypt` — not write-gated, local files only
- `navi_action_decrypt` — not write-gated, local files only

**CLI-only — you run at your terminal:**

- `navi action push` — remote command execution against Linux hosts
- `navi action mail` — email reports and files

When running under navi-mcp, use tool-invocation form (shown first in each
MCP example). Bash forms are standalone CLI equivalents. For push and mail,
only CLI examples appear — there are no tool forms.

---

## `navi_action_delete` — remove objects from Tenable VM

**IRREVERSIBLE for most kinds.** Always write-gated. Narrate the specific
object to be deleted and get explicit user confirmation before invoking.

> **Not for tag rotation.** If the goal is to refresh which assets carry
> a tag, use `navi_enrich_tag(..., remove=True)` instead — it reassigns
> asset membership while preserving the tag UUID. Deleting the tag
> generates a new UUID on recreation and breaks downstream references
> (access groups, dashboards, API integrations). See navi-enrich's "Tag
> UUID preservation" section.

| Kind | Required args | Reversibility |
|---|---|---|
| `tag` | `category`, `value` | Reversible — but re-creation generates new UUID |
| `user` | `username` (email) | User data gone; account can be recreated |
| `scan` | `id` | Irreversible — scan history lost |
| `asset` | `uuid` | Irreversible in navi.db; Tenable may re-discover on next scan |
| `agent` | `id` | Irreversible — agent needs re-linking |
| `exclusion` | `id` | Reversible — recreate the exclusion |

### Tag deletion — for permanent retirement only

Use when a tag's entire purpose has ended (e.g. retiring a specific
remediation campaign). Do not use as part of a monthly refresh cycle.

`navi_action_delete(kind="tag", category="Remediation", value="Jenkins-Q2", confirm=True)`

```bash
navi action delete tag --c "Remediation" --v "Jenkins-Q2"
```

**Trigger phrases:** "retire this tag", "the campaign is done, clean up
the tag", "delete the X:Y tag permanently", "remove this tag entirely"

### User deletion (offboarding)

Use for departing employees and revoked access. Note that
`navi_explore_info(subcommand="users")` can check if a user exists before
you delete — and if they're disabled rather than deleted, you may be able
to re-enable instead.

`navi_action_delete(kind="user", username="former.employee@company.com", confirm=True)`

```bash
navi action delete user --username "former.employee@company.com"
```

**Trigger phrases:** "offboard X", "remove user X", "delete account for X",
"X left the company"

### Scan deletion

Removes the scan and its history. Irreversible — if you need to keep the
historical findings, export them first via `navi_export(subcommand="vulns")`
or a custom `navi_export(subcommand="query", sql=...)` scoped to the scan.

`navi_action_delete(kind="scan", id="<SCAN_ID>", confirm=True)`

```bash
navi action delete scan --id <SCAN_ID>
```

**Trigger phrases:** "delete this scan", "remove scan ID X", "clean up
old scans"

### Asset deletion

Removes an asset from navi.db. Tenable may re-discover it on the next scan
that includes its IP, so this is not always a durable removal.

`navi_action_delete(kind="asset", uuid="<UUID>", confirm=True)`

```bash
navi action delete asset --uuid <UUID>
```

**Trigger phrases:** "remove this asset", "delete host X from navi",
"decommissioned host, remove it"

### Agent deletion

Removes a Nessus agent. The agent itself needs re-linking on the host if
you want it back.

`navi_action_delete(kind="agent", id="<AGENT_ID>", confirm=True)`

```bash
navi action delete agent --id <AGENT_ID>
```

**Trigger phrases:** "unlink this agent", "remove agent X", "agent is
broken, delete it"

### Exclusion deletion

Removes a scan exclusion window.

`navi_action_delete(kind="exclusion", id="<EXCLUSION_ID>", confirm=True)`

```bash
navi action delete exclusion --id <EXCLUSION_ID>
```

**Trigger phrases:** "remove scan exclusion", "delete exclusion window X",
"we can scan that asset now, remove the exclusion"

---

## `navi_action_rotate` — rotate user API keys

Used for offboarding, security incidents, and credential hygiene.

**Critical side effect:** the old keys stop working immediately. Anything
using them — automations, scripts, CI, other navi workloads — will fail
until they receive the new keys. Always warn the user about this before
invoking, and confirm they've identified all downstream consumers.

`navi_action_rotate(username="user@company.com", confirm=True)`

```bash
navi action rotate --username "user@company.com"
```

**Trigger phrases:** "rotate keys for X", "regenerate API credentials",
"user left, rotate their keys", "security incident, rotate X", "refresh
credentials"

---

## `navi_action_cancel` — stop a running export

Cancel a running Tenable export job. Useful when an export started with
the wrong filters or is taking too long.

Check what's running first:

`navi_explore_info(subcommand="exports")`

If nothing is running, the cancel is a no-op. Two kinds, matching the two
export types:

`navi_action_cancel(kind="assets", confirm=True)` — cancel running asset export
`navi_action_cancel(kind="vulns", confirm=True)` — cancel running vuln export

```bash
navi action cancel -a
navi action cancel -v
```

**Trigger phrases:** "cancel the export", "stop the running export",
"the export is stuck, kill it", "abort the asset/vuln export"

---

## `navi_action_encrypt` / `navi_action_decrypt` — local file security

Encrypt or decrypt a file locally. **Not write-gated** — these operate on
local files only and do not touch the Tenable platform or navi.db.

Use for: sanitizing exported CSVs containing sensitive data before sharing,
securing credential files at rest, encrypting reports before email
delivery via `navi action mail`.

Encrypt — produces `<file>.enc` alongside the original:

`navi_action_encrypt(file="sensitive_export.csv")`

```bash
navi action encrypt --file sensitive_export.csv
```

Decrypt — expects a `.enc` file:

`navi_action_decrypt(file="sensitive_export.csv.enc")`

```bash
navi action decrypt --file sensitive_export.csv.enc
```

Paths can be absolute or relative to the navi-mcp workdir. Check
`navi://workdir` if unsure where files land.

**Trigger phrases:** "encrypt this file", "secure the export", "lock down
this CSV", "decrypt the .enc file", "unlock this report"

---

## `navi action push` — remote command execution (CLI only)

**Not exposed through navi-mcp.** This command executes arbitrary shell
commands against Linux hosts grouped by tag. It is deliberately kept
CLI-only — letting an LLM drive remote shell execution is a risk surface
not worth opening, even with confirmation gates.

Claude explains this command as a CLI step when a workflow naturally
includes it (remediation, patching, inventory collection), and never tries
to invoke it through any tool.

### Common push patterns

Update a package across a tagged group:

```bash
navi action push --tag "Remediation:OpenSSL-Q2" \
  --command "sudo apt-get update && sudo apt-get upgrade openssl -y"
```

Restart a service:

```bash
navi action push --tag "Service:Nginx" --command "sudo systemctl restart nginx"
```

Collect inventory:

```bash
navi action push --tag "Environment:Production" --command "uname -a; uptime"
```

Apply a configuration change:

```bash
navi action push --tag "CIS:Hardening-Required" \
  --command "sudo sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config && sudo systemctl reload sshd"
```

### The Route → Tag → Push → Verify cycle

The canonical remediation pattern crosses between MCP tools and the CLI.
Claude drives the tagging and verification; the user runs `push` at their
terminal.

**Worked example — OpenSSL upgrade across affected hosts:**

> I'll walk through the full OpenSSL remediation cycle. Tagging and
> verification happen through MCP tools; the push step is CLI-only, so
> you'll run that at your terminal.
>
> **Step 1 (MCP) — tag the affected hosts.** Find every asset with an
> OpenSSL-related finding and tag them `Remediation:OpenSSL-Q2`.
>
> Tool call: `navi_enrich_tag(category="Remediation", value="OpenSSL-Q2",
> query="SELECT DISTINCT asset_uuid FROM vulns WHERE plugin_name LIKE '%OpenSSL%' AND state='active';",
> confirm=True)`
>
> Confirm to apply.
>
> *[user confirms; tool runs; 30-min propagation window begins]*
>
> **Step 2 (CLI) — push the upgrade.** Run this at your terminal:
>
> ```bash
> navi action push --tag "Remediation:OpenSSL-Q2" \
>   --command "sudo apt-get update && sudo apt-get upgrade openssl libssl1.1 libssl3 -y"
> ```
>
> Wait for push to complete, then launch your verification scan. Let me
> know when the scan is done and vulns have re-synced.
>
> **Step 3 (MCP) — verify and clean up.** Once you've re-synced, I'll
> check that the OpenSSL vulns are gone and retire the remediation tag.
> Tool calls:
>
> - `navi_explore_query(sql="SELECT count(*) FROM vulns WHERE plugin_name LIKE '%OpenSSL%' AND state='active';")`
> - If zero: `navi_action_delete(kind="tag", category="Remediation", value="OpenSSL-Q2", confirm=True)`
>
> (Deleting the campaign tag is appropriate here — it's truly retiring,
> not rotating. See the delete section.)

Claude does NOT try to orchestrate the CLI step silently, does NOT pretend
to invoke `action push` through an MCP tool, and does NOT continue past
the CLI step without user confirmation that the push completed.

---

## `navi action mail` — email reports (CLI only)

**Not exposed through navi-mcp.** Email delivery is deliberately kept CLI
so that an LLM does not fire off email on a user's behalf. When a workflow
ends in "send X to someone," Claude completes the MCP portion, surfaces
the file path, and hands off to the CLI for mail.

### Common mail patterns

Mail a CSV to a single recipient:

```bash
navi action mail --to "ciso@company.com" --file "bytag_export.csv"
```

Mail with a custom subject:

```bash
navi action mail --to "security-team@company.com" \
  --subject "Weekly SLA Breach Report" --file "failures_export.csv"
```

Mail an encrypted file (compose with `navi_action_encrypt`):

Step 1 (MCP) — encrypt the file:
`navi_action_encrypt(file="pii_export.csv")`

Step 2 (CLI) — mail the encrypted version:
```bash
navi action mail --to "auditor@company.com" --file "pii_export.csv.enc"
```

### The export → mail handoff pattern

> I'll export production assets with ACR and AES scores.
>
> Tool call: `navi_export(subcommand="bytag", category="Environment", value="Production")`
>
> *[tool returns: "Wrote bytag_export.csv — 1,247 rows"]*
>
> CSV is at `bytag_export.csv`. To mail it to your CISO, run this at your
> terminal:
>
> ```bash
> navi action mail --to "ciso@company.com" --file "bytag_export.csv"
> ```

Claude does not try to invoke mail through any tool and does not batch the
mail step silently into a sequence.

---

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

---

## Action → Natural Language

| User says | Tool call / CLI |
|---|---|
| "delete tag X:Y permanently" | `navi_action_delete(kind="tag", category="X", value="Y", confirm=True)` |
| "refresh the tag's asset list" | Use `navi_enrich_tag(..., remove=True, ...)` — not delete (see navi-enrich) |
| "offboard user / remove user account" | `navi_action_delete(kind="user", username="...", confirm=True)` |
| "delete this scan" | `navi_action_delete(kind="scan", id="...", confirm=True)` |
| "remove this asset from navi" | `navi_action_delete(kind="asset", uuid="...", confirm=True)` |
| "unlink this agent" | `navi_action_delete(kind="agent", id="...", confirm=True)` |
| "remove scan exclusion" | `navi_action_delete(kind="exclusion", id="...", confirm=True)` |
| "rotate API keys for user" | `navi_action_rotate(username="...", confirm=True)` |
| "cancel the running asset export" | `navi_action_cancel(kind="assets", confirm=True)` |
| "cancel the running vuln export" | `navi_action_cancel(kind="vulns", confirm=True)` |
| "encrypt this file" | `navi_action_encrypt(file="...")` |
| "decrypt this file" | `navi_action_decrypt(file="...")` |
| "run a command against tagged hosts" | **CLI:** `navi action push --tag ... --command ...` |
| "remediate all OpenSSL / Log4j / \<package\> hosts" | Route → Tag (MCP) → Push (CLI) → Verify (MCP) cycle above |
| "email this report" | **CLI:** `navi action mail --to ... --file ...` |
| "run the operational hygiene routine" | Three-phase workflow above |
