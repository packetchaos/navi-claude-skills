---
name: navi-action
description: >
  Automation and action skill for Tenable navi CLI. Covers navi action commands
  exposed through navi-mcp: delete (remove tags/users/scans/assets/tgroups/
  usergroups/tone), rotate (API key rotation), cancel (stop exports),
  encrypt/decrypt (file security). The two hazardous action commands — push
  (remote command execution) and mail (email) — are now exposed as double-gated
  tools; their detailed harnesses live in the navi-remote-exec and navi-mail
  skills. Trigger on: "delete this tag", "remove stale tags", "rotate API keys",
  "offboard user", "cancel export", "encrypt a file", "send report to", "push a
  command to assets", "remediate tagged assets", "run a command across a tag
  group", "clean up tags".
---

# Navi Action — Operations & Remediation

Seven action commands are exposed through MCP as tools. Five general-purpose
ones are documented here; the two hazardous ones (`push`, `mail`) are exposed
but each sits behind its own capability gate and has a dedicated harness skill.

**Documented here:**

- `navi_action_delete` — write-gated, destructive
- `navi_action_rotate` — write-gated
- `navi_action_cancel` — write-gated
- `navi_action_encrypt` — not write-gated, local files only
- `navi_action_decrypt` — not write-gated, local files only

**Hazardous — double-gated, see the dedicated skill:**

- `navi_action_push` — remote command execution on a Linux host. Requires
  `NAVI_MCP_ALLOW_WRITES=1` + `NAVI_REMOTE_CODE_EXECUTION=1` + `confirm=True`.
  Full harness: **navi-remote-exec** (`navi://skill/remote-exec`).
- `navi_action_mail` — email a report/file. Requires `NAVI_MCP_ALLOW_WRITES=1` +
  `NAVI_EMAIL=1` + `confirm=True`. Full harness: **navi-mail**
  (`navi://skill/mail`).

When running under navi-mcp, use tool-invocation form (shown first in each
MCP example). Bash forms are standalone CLI equivalents.

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
| `bytag` | `tag_string` (`"category:value"`) | Deletes **assets** matching the tag — irreversible (Tenable may re-discover) |
| `asset` | `object_id` (Asset UUID) | Irreversible in navi.db; Tenable may re-discover on next scan |
| `scan` | `object_id` (Scan ID) | Irreversible — scan history lost |
| `user` | `user_id` (numeric User ID — not email/UUID) | User data gone; account can be recreated |
| `tgroup` | `object_id` (target-group ID) | Reversible — recreate the group |
| `usergroup` | `object_id` (user-group ID) | Reversible — recreate the group |
| `tone` | `category`, `value` (+ optional `remove`) | TONE tag; reversible. `remove=True` strips assets from the tag instead of deleting it |

> **No `agent` or `exclusion` delete exists in navi** — those commands aren't in
> the CLI. Remove agents/exclusions in the Tenable UI, or via the
> `navi_explore_api` passthrough against the relevant Tenable endpoint.

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

Use for departing employees and revoked access. `navi action delete user`
takes the **numeric User ID** (not the email) — get it from
`navi_explore_info(subcommand="users")`. If the user is disabled rather than
deleted, you may be able to re-enable instead.

`navi_action_delete(kind="user", user_id="<USER_ID>", confirm=True)`

```bash
navi action delete user <USER_ID>
```

**Trigger phrases:** "offboard X", "remove user X", "delete account for X",
"X left the company"

### Scan deletion

Removes the scan and its history. Irreversible — if you need to keep the
historical findings, export them first via `navi_export(subcommand="vulns")`
or a custom `navi_export(subcommand="query", sql=...)` scoped to the scan.

`navi_action_delete(kind="scan", object_id="<SCAN_ID>", confirm=True)`

```bash
navi action delete scan <SCAN_ID>
```

**Trigger phrases:** "delete this scan", "remove scan ID X", "clean up
old scans"

### Asset deletion

Removes an asset from navi.db. Tenable may re-discover it on the next scan
that includes its IP, so this is not always a durable removal.

`navi_action_delete(kind="asset", object_id="<UUID>", confirm=True)`

```bash
navi action delete asset <UUID>
```

**Trigger phrases:** "remove this asset", "delete host X from navi",
"decommissioned host, remove it"

### bytag deletion — deletes ASSETS, not the tag

**Very destructive.** `bytag` deletes every *asset* carrying a tag, not the
tag itself. Use only for bulk asset removal (e.g. decommissioning everything in
a retired segment). To remove a tag, use `kind="tag"`; to refresh tag
membership, use `navi_enrich_tag(..., remove=True)`.

`navi_action_delete(kind="bytag", tag_string="Decommissioned:Q1-Segment", confirm=True)`

```bash
navi action delete bytag "Decommissioned:Q1-Segment"
```

**Trigger phrases:** "delete all assets tagged X", "remove every host in the
decommissioned segment"

### tone deletion (TONE tags)

For Tenable One TONE tags (case-sensitive). `remove=True` strips assets from
the tag instead of deleting the tag.

`navi_action_delete(kind="tone", category="<cat>", value="<val>", confirm=True)`

```bash
navi action delete tone --c "<cat>" --v "<val>"        # delete the TONE tag
navi action delete tone --c "<cat>" --v "<val>" -remove # remove assets from it
```

### Target groups / user groups

`navi_action_delete(kind="tgroup", object_id="<TGROUP_ID>", confirm=True)` and
`navi_action_delete(kind="usergroup", object_id="<USERGROUP_ID>", confirm=True)`
delete a target group / user group by ID (both reversible — recreate as needed).

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

Cancel a running Tenable export job by its **export UUID** (required). Useful
when an export started with the wrong filters or is taking too long.

Find the running export's UUID first:

`navi_explore_info(subcommand="exports")` (or `navi_explore_api(url="/vulns/export/<UUID>/status")`)

Then cancel it — `kind` picks the export type (`assets`→`-a`, `vulns`→`-v`):

`navi_action_cancel(kind="assets", uuid="<EXPORT_UUID>", confirm=True)` — cancel an asset export
`navi_action_cancel(kind="vulns", uuid="<EXPORT_UUID>", confirm=True)` — cancel a vuln export

```bash
navi action cancel <EXPORT_UUID> -a
navi action cancel <EXPORT_UUID> -v
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

## `navi_action_push` — remote command execution (double-gated)

Exposed as a tool, but the highest-risk one in navi: it runs shell commands on a
**single Linux host** over SSH. Requires `NAVI_MCP_ALLOW_WRITES=1` +
`NAVI_REMOTE_CODE_EXECUTION=1` on the server, plus `confirm=True` after you
narrate the literal target and command.

`navi_action_push(target="10.0.5.12", command="sudo systemctl restart nginx", confirm=True)`

Key constraints: exactly one of `command`/`file`; **no `--tag`** — push hits one
IP, so loop per-IP for a tagged group. The full harness — the Route→Tag→Push→
Verify remediation cycle, per-IP looping, SSH prerequisite, and failure modes —
lives in the **navi-remote-exec** skill (`navi://skill/remote-exec`). Load it
before driving any remediation push.

---

## `navi_action_mail` — email reports (double-gated)

Exposed as a tool. Sends email via the out-of-band SMTP config. Requires
`NAVI_MCP_ALLOW_WRITES=1` + `NAVI_EMAIL=1` on the server, plus `confirm=True`
after you narrate the recipient, subject, and attachment.

`navi_action_mail(to="ciso@company.com", subject="Production export", file="bytag_export.csv", confirm=True)`

Compose with `navi_export` (produce the CSV) or `navi_action_encrypt` (secure a
sensitive attachment first, mail the `.enc`). The full harness — export→mail and
encrypt→mail patterns, gating detail, and failure modes — lives in the
**navi-mail** skill (`navi://skill/mail`).

---

## Operational hygiene workflow

A realistic recurring pattern that chains the navi_* tools with the CLI handoff
(push/mail) — and the rationale for splitting it into three phases — lives in
**`references/operational-hygiene.md`** (`navi://skill/action/operational-hygiene`).

---

## Action → Natural Language

| User says | Tool call / CLI |
|---|---|
| "delete tag X:Y permanently" | `navi_action_delete(kind="tag", category="X", value="Y", confirm=True)` |
| "refresh the tag's asset list" | Use `navi_enrich_tag(..., remove=True, ...)` — not delete (see navi-enrich) |
| "offboard user / remove user account" | `navi_action_delete(kind="user", user_id="<numeric ID>", confirm=True)` (ID from `navi_explore_info(subcommand="users")`) |
| "delete this scan" | `navi_action_delete(kind="scan", object_id="...", confirm=True)` |
| "remove this asset from navi" | `navi_action_delete(kind="asset", object_id="<UUID>", confirm=True)` |
| "delete all assets tagged X:Y" | `navi_action_delete(kind="bytag", tag_string="X:Y", confirm=True)` (deletes assets!) |
| "delete a TONE tag" | `navi_action_delete(kind="tone", category="X", value="Y", confirm=True)` |
| "rotate API keys for user" | `navi_action_rotate(username="...", confirm=True)` |
| "cancel the running asset export" | `navi_action_cancel(kind="assets", uuid="<EXPORT_UUID>", confirm=True)` |
| "cancel the running vuln export" | `navi_action_cancel(kind="vulns", uuid="<EXPORT_UUID>", confirm=True)` |
| "encrypt this file" | `navi_action_encrypt(file="...")` |
| "decrypt this file" | `navi_action_decrypt(file="...")` |
| "run a command against tagged hosts" | Read the tag's IPs → loop `navi_action_push(target="<IP>", command="...", confirm=True)` (see navi-remote-exec) |
| "remediate all OpenSSL / Log4j / \<package\> hosts" | Route → Tag → Push → Verify cycle (all MCP; see navi-remote-exec) |
| "email this report" | `navi_action_mail(to="...", file="...", confirm=True)` (see navi-mail) |
| "run the operational hygiene routine" | Three-phase workflow above |
