---
name: navi-mcp
description: >
  Conventions for using the navi CLI through the navi-mcp server. Load this
  whenever the navi_* MCP tools are available (navi_enrich_tag, navi_explore_query,
  navi_export, navi_scan, etc.). Covers: tool-invocation-first style, write-gate
  confirmation pattern, the out-of-band key assumption, how to interpret empty
  results, and which navi CLI commands are intentionally NOT exposed through
  MCP. Trigger: any time Claude would otherwise produce a `navi ...` bash
  block AND the navi_* tools are visible in the tool list. If the tools are
  not visible, ignore this skill — fall back to CLI-first guidance from the
  domain skills (navi-core, navi-enrich, navi-explore, navi-export, navi-scan,
  navi-action, navi-was).
---

# Navi MCP — Conventions

This skill applies whenever Claude is running under the navi-mcp server. It
tells Claude *how* to use the navi_* tools; the domain skills
(navi-enrich, navi-explore, etc.) tell Claude *what* each tool does.

If the navi_* tools are not available in this session, stop reading — the
domain skills work standalone and Claude should follow their CLI guidance.

---

## Tool-invocation-first style

When an MCP tool exists for an operation, call it directly. Do NOT produce a
`navi ...` bash block for the user to copy-paste.

**Do:**
> I'll tag production assets with ACR 10.
> [calls `navi_enrich_tag(category="Environment", value="Production", group="Production Servers", confirm=True)`]

**Don't:**
> Run this:
> ```bash
> navi enrich tag --c "Environment" --v "Production" --group "Production Servers"
> ```

The CLI equivalents in the domain skills are secondary reference material —
useful for standalone readers, but not what Claude emits when tools are available.

`navi action push` and `navi action mail` are now exposed as
`navi_action_push` / `navi_action_mail`, but each sits behind an extra
capability gate on top of the write gate (see "Extra capability gates" below).

---

## The write-gate confirmation pattern

Several navi_* tools require an explicit `confirm=True` flag AND the server's
`NAVI_MCP_ALLOW_WRITES=1` environment variable to execute. These are the
platform-write tools:

- `navi_enrich_tag` — creates/modifies tags
- `navi_enrich_acr` — adjusts asset criticality ratings
- `navi_enrich_add` — adds assets to the platform
- `navi_config(kind="url", ...)` — sets the FedRAMP/custom base URL
- `navi_explore_api(method="POST"|"PUT", ...)` — mutating API passthrough (GET is read, not gated)
- `navi_scan` — create, start, stop, pause, resume (status/details/history/hosts/latest/evaluate are read-only, not gated)
- `navi_was` — scan, start, upload (configs, scans, details, stats, export are read-only, not gated)
- `navi_action_delete` — destructive; always write-gated
- `navi_action_rotate` — rotates API keys
- `navi_action_cancel` — cancels a running export (requires the export `uuid`)
- `navi_action_mail` — sends email; **double-gated** (also needs `NAVI_EMAIL=1`)
- `navi_action_push` — remote command execution; **double-gated** (also needs
  `NAVI_REMOTE_CODE_EXECUTION=1`)

### Extra capability gates (mail & push)

Two tools require a second env gate ON TOP of `NAVI_MCP_ALLOW_WRITES=1` — the
master write gate alone does not enable them:

- `navi_action_mail` runs only when `NAVI_MCP_ALLOW_WRITES=1` **and**
  `NAVI_EMAIL=1`.
- `navi_action_push` runs only when `NAVI_MCP_ALLOW_WRITES=1` **and**
  `NAVI_REMOTE_CODE_EXECUTION=1`.

Both still require per-call `confirm=True` after Claude narrates intent. If the
capability gate is closed, the tool returns a block message — report it and
stop; the operator restarts the server with the env var set (config helper:
`--allow-email` / `--allow-remote-code-execution`, both alongside
`--allow-writes`). Full harnesses: `navi://skill/mail` and
`navi://skill/remote-exec`.

`navi_config(kind="software", ...)` and `navi_config(kind="certificates", ...)`
are NOT write-gated — they parse plugin output into local navi.db tables
(software; certs via plugin 10863) and don't change anything on the Tenable
platform.

`navi_config(kind="sla", ...)` is NOT write-gated — it runs `config sla
calculate` (computes SLA times locally). Setting/overwriting thresholds is
`config sla reset`, which is interactive and stays on the CLI. See navi-core.

`navi_action_encrypt` and `navi_action_decrypt` are NOT write-gated —
they operate on local files only, no platform interaction.

**For every write-gated call, Claude:**

1. **Describes the operation in prose** — what it will do, on what scope, what
   the effect is. Don't skip this even for small operations.
2. **States the exact tool call** it's about to make, including the argument
   values.
3. **Asks the user to confirm** before invoking.
4. **Only calls with `confirm=True` after the user confirms** in the chat.

**Example — tagging:**

> I'll refresh the `CertExpiry:ExpiringSoon` tag to cover assets with a
> cert expiring in April 2026. I'm using `remove=True` so the tag
> reflects current state — the tag UUID stays the same, only the assets
> it applies to change.
>
> Tool call: `navi_enrich_tag(category="CertExpiry", value="ExpiringSoon",
> query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Apr%2026%';",
> remove=True, confirm=True)`
>
> Confirm and I'll apply it.

**Example — deletion:**

> This will permanently delete the `Remediation:Jenkins-Q2` tag from every
> asset it's currently applied to. This cannot be undone.
>
> Tool call: `navi_action_delete(kind="tag", category="Remediation",
> value="Jenkins-Q2", confirm=True)`
>
> Confirm to proceed.

### When writes are blocked server-side

If the server rejects a write-gated call because `NAVI_MCP_ALLOW_WRITES`
is not set, Claude reports the block and stops. Do not retry.

When explaining this to the user, make the implications clear:

> This navi-mcp server is running in read-only mode. Enabling writes requires
> setting `NAVI_MCP_ALLOW_WRITES=1` on the server — typically in the
> environment or config file used to launch navi-mcp. That change:
>
> - affects every future session against this server, not just this one;
> - is a security-sensitive decision — writes include destructive operations
>   (tag deletion, API key rotation, scan cancellation) and changes that
>   ripple into your Tenable platform;
> - takes effect only after the navi-mcp server is restarted with the new
>   environment.
>
> Claude cannot change this from inside the tool surface. If you want to
> enable writes, update the server configuration yourself and restart it;
> I'll continue with the read-only parts of your request in the meantime.

**For read operations** (`navi_explore_data`, `navi_explore_info`,
`navi_explore_query` with SELECT/WITH statements, `navi_export`,
`navi_action_encrypt`, `navi_action_decrypt`), no confirmation is required.
Call them freely.

---

## API keys are set out-of-band

Claude does NOT see or manage API keys. The `navi config keys` command is not
exposed as a tool — it is the operator's responsibility to set keys before
navi-mcp is started.

**If a query returns empty or a command fails with no data**, do not assume
keys are missing and do not offer to set them. Instead:

1. Check `navi://workdir` to see where navi.db lives and whether write-gate
   is enabled.
2. Check `navi_explore_info(subcommand="version")` to verify the connection
   to Tenable is alive.
3. Report the empty result to the user and suggest they verify key scope
   externally (in Tenable's settings). Keys scoped to a subset of assets
   will silently return partial data — this is the most common cause of
   "missing data" questions.

The full guidance for setting keys and their permission implications lives
in navi-core. Claude can cite it when useful, but cannot execute the setup
itself.

---

## Resources

The navi-mcp server exposes these read-only resources:

- **`navi://schema/{table}`** — column definitions for any table in navi.db.
  Use this before writing a `navi_explore_query(sql=...)` when Claude is
  unsure of column names. Preferred over guessing or `SELECT * ... LIMIT 1`.

- **`navi://workdir`** — where navi.db lives, the write-gate status, the navi
  binary path, the MCP call budget, AND navi.db freshness (newest vuln
  `last_found` / scan date). One read covers "where's my data," "why was my
  write rejected," and the staleness check below.

- **`navi://skill/{name}`** — load a domain skill on demand (`name` ∈ router,
  mcp, core, troubleshooting, enrich, acr, explore, export, scan, action, was).
  The router is injected by the `navi_workflow` prompt; pull others when the
  task matches their scope. If a skill has bundled references, they're listed
  at the end of its output.

- **`navi://skill/{name}/{ref}`** — load a skill's bundled reference file
  (`ref` = filename without `.md`), e.g. `navi://skill/core/schema`. Skills use
  progressive disclosure: `SKILL.md` is the lean index; deep material
  (full schemas, exhaustive tables, long examples) lives in references loaded
  only when needed.

The server also exposes the **`navi_workflow` prompt** — in Claude Desktop it's
the `/navi_workflow` slash command, which injects the router skill and frames
the user's task.

Resources are cheap — read them when they'd help, no need to ask permission.

### `navi_explore_query` supports both reads and writes

- **Reads** (statements starting with `SELECT` or `WITH`) — the default.
  No confirmation needed. Call freely.
- **Writes** (`CREATE INDEX`, `UPDATE`, `DELETE`, DDL) — require
  `confirm=True`. These modify navi.db only, not the Tenable platform.

Unlike the platform-write tools, local writes do NOT require
`NAVI_MCP_ALLOW_WRITES=1`. The platform-write gate protects things that
ripple into Tenable; local-only navi.db changes are recoverable via
`navi_config_update(kind=...)`, so `confirm=True` alone is the signal of
write intent.

For any non-SELECT statement, Claude narrates the effect in prose before
calling:

> I'll add an index on `vulns.plugin_id` to speed up plugin-based tagging.
> This modifies navi.db (not the Tenable platform); indexes persist until
> navi.db is rebuilt.
>
> `navi_explore_query(sql="CREATE INDEX IF NOT EXISTS idx_vulns_plugin ON vulns(plugin_id);", confirm=True)`

No separate write tool — the same `navi_explore_query` handles both. The
`confirm=True` flag is the signal of write intent.

---


## Commands not exposed through navi-mcp

Some navi CLI commands are intentionally not wrapped as tools. Three categories:

| Category | Commands | Claude's behavior |
|---|---|---|
| **Too heavy / foundational** (CLI; actively recommend) | `navi config update full`; one-time setup: `navi config optimize` / `epss` / `smtp` / `ssh` | Recommend at the right moment; run at the terminal |
| **Out of scope** (don't teach) | `navi action deploy` / `automate` / `plan`, `navi enrich attribute` / `migrate` / `tagrule`, `navi config keys` | Acknowledge they exist; no runnable examples |

**Now exposed (previously CLI-only):** `navi explore api` → `navi_explore_api`
(GET free, POST/PUT write-gated); `navi config certificates` →
`navi_config(kind="certificates")`; `navi action mail` → `navi_action_mail`
(double-gated, `NAVI_EMAIL`); `navi action push` → `navi_action_push`
(double-gated, `NAVI_REMOTE_CODE_EXECUTION`). See `navi://skill/mail` and
`navi://skill/remote-exec`.

**MCP → CLI handoff rule.** When a workflow crosses into a CLI-only command
(`update full`, `config smtp` / `config ssh` setup), Claude finishes the MCP
portion, tells the user exactly what to run, and waits for confirmation that it
completed before resuming on the MCP side. Claude never pretends to invoke a
CLI-only command through a tool, and never continues silently past the CLI step.
(`push` and `mail` are no longer part of this hand-off — they are tools now.)

**The ~4-minute call ceiling** on long targeted syncs
(`navi_config_update(kind="vulns")` on a large tenant) is covered in **navi-core**
— its canonical home — including the `days=N` and `--threads 1` CLI fallback.

Full per-command rationale, the one-time-setup command details, and the worked
Route → Tag → Push → Verify remediation example live in
**`references/commands-not-exposed.md`** (`navi://skill/mcp/commands-not-exposed`).

---

## Data freshness check

At the start of any data-dependent workflow, Claude checks how fresh navi.db
is and surfaces a heads-up if the data looks stale.

**A workflow is data-dependent when** it queries, exports, or tags based on
data that lives in navi.db — i.e. it uses any of: `navi_explore_query`,
`navi_explore_data`, `navi_export`, `navi_enrich_tag`, `navi_enrich_acr`,
`navi_enrich_add`. A workflow is NOT data-dependent (and skips the check)
when it only touches live Tenable state via `navi_explore_info` subcommands,
or when it's a pure management operation (`navi_action_rotate`,
`navi_action_cancel`, etc.).

**The check itself:** one query via `navi_explore_query`, cheap, no user
confirmation needed (reads don't require confirm).

```
navi_explore_query(sql="SELECT MAX(last_found) AS newest_vuln,
MAX(last_licensed_scan_date) AS newest_scan FROM vulns;")
```

(Or read **`navi://workdir`**, which now reports the same freshness figures —
one resource read covers the check.)

**Interpreting the result:**

- **Empty or NULL** → navi.db is empty. Treat as apparent first-run.
  Surface the `config update full` recommendation before doing anything
  else data-dependent.
- **Newer than ~7 days** → data is fresh. Proceed silently, no message to
  the user about freshness.
- **7–30 days old** → soft heads-up. Mention the age in one sentence
  before proceeding, so the user can choose to refresh or proceed on
  slightly stale data: *"navi.db's newest vuln last_found is 12 days old —
  proceeding on current data; let me know if you'd like to refresh first."*
- **Older than 30 days** → firm recommendation. Surface the full
  `config update full` recommendation (as shown in the previous section)
  and offer the targeted `navi_config_update(kind="vulns")` alternative.

**Ephemeral consideration:** on an empty or very stale DB, some targeted
updates (`navi_config_update(kind="vulns")`, `...(kind="assets")`) may be
sufficient for the immediate workflow. Claude prefers the targeted update
as the first suggestion when it will cover the scope of what the user
actually asked for, reserving the full-sync recommendation for truly
missing foundational data.

**One check per conversation.** Once Claude has run the freshness check
in a session, it does not re-check on every subsequent workflow unless
the user explicitly asks about data currency or a write operation has
just occurred (where a post-write reminder about re-syncing is in play).

---

## Output format

When Claude is fulfilling a navi request under navi-mcp:

1. **Summarize** what's going to happen in a sentence or two.
2. **Read-first**: gather any context needed (schemas via `navi://schema/{table}`,
   lookups via `navi_explore_info(...)`, counts via `navi_explore_query(...)`)
   before proposing writes.
3. **Narrate writes before calling**: for every write-gated tool, describe +
   state the exact call + wait for confirmation. Never batch multiple
   write-gated calls behind a single confirmation.
4. **Report results plainly** after the call returns. For exports, show the
   returned CSV path and note that the preview is a preview, not the full
   export — the user should open the CSV or use `navi_explore_query` for
   further analysis.
5. **Suggest verification** when meaningful — a follow-up
   `navi_explore_query(...)` or `navi_explore_info(...)` that lets the user
   see the effect of the change. Remember the 30-minute propagation window
   for tags: verification via Tenable's UI/API right after a write will
   often show stale data; `navi_explore_query` against navi.db reflects
   what navi just did.
6. **Do not include CLI bash blocks** unless the workflow needs one of the
   kept-as-CLI commands above.

---

## Cross-references

- **navi-core** — setup, schema, the 30-minute propagation rule, 50K-asset
  scale fork, multi-workload pattern, **and the ~4-min targeted-sync ceiling**
- **navi-troubleshooting** — fix-it workflows for errors, empty results,
  slow tagging, post-upgrade recovery
- **navi-enrich** — tagging, the `remove=True` ephemeral pattern, tag UUID
  preservation
- **navi-acr** — Asset Criticality Rating adjustment, Change Reasons,
  mod set/inc/dec, suggested tier mapping
- **navi-explore** — data and info subcommands, the `explore_api` passthrough, raw SQL patterns
- **navi-export** — CSV exports
- **navi-scan** — scan creation, control, read views, evaluate
- **navi-action** — delete, rotate, cancel, encrypt/decrypt (+ pointers to mail/push)
- **navi-mail** — `navi_action_mail`; email harness, double gate (`NAVI_EMAIL`)
- **navi-remote-exec** — `navi_action_push`; RCE harness, double gate
  (`NAVI_REMOTE_CODE_EXECUTION`)
- **navi-was** — WAS configs, scans, findings, tagging

**Bundled reference:** `references/commands-not-exposed.md` — full not-exposed
rationale, one-time-setup commands, and the Route→Tag→Push→Verify worked example.
