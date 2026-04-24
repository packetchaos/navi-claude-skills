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

**One exception:** for the three commands not exposed through MCP
(`navi action push`, `navi action mail`), always use the CLI form — there
is no tool. See "Commands not exposed" below.

---

## The write-gate confirmation pattern

Several navi_* tools require an explicit `confirm=True` flag AND the server's
`NAVI_MCP_ALLOW_WRITES=1` environment variable to execute. These are the
platform-write tools:

- `navi_enrich_tag` — creates/modifies tags
- `navi_enrich_acr` — adjusts asset criticality ratings
- `navi_enrich_add` — adds assets to the platform
- `navi_config(kind="url", ...)` — sets the FedRAMP/custom base URL
- `navi_scan` — create, start, stop (evaluate is read-only and not gated)
- `navi_was` — scan, start, upload (configs, scans listing, details, stats,
  export are read-only and not gated)
- `navi_action_delete` — destructive; always write-gated
- `navi_action_rotate` — rotates API keys
- `navi_action_cancel` — cancels running exports

`navi_config(kind="software", ...)` is NOT write-gated — it parses plugin
output into the local navi.db software table and doesn't change anything
on the Tenable platform.

`navi_config(kind="sla", ...)` is NOT write-gated — it sets SLA thresholds
but is interactive on the CLI. The tool completes without accepting stdin,
so for initial setup, running at the terminal is more reliable. See navi-core.

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

Claude does NOT see or manage API keys. The `navi keys` command is not
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

The navi-mcp server exposes two read-only resources:

- **`navi://schema/{table}`** — column definitions for any table in navi.db.
  Use this before writing a `navi_explore_query(sql=...)` when Claude is
  unsure of column names. Preferred over guessing or over running
  `SELECT * FROM {table} LIMIT 1`.

- **`navi://workdir`** — the directory where navi.db lives, plus the current
  write-gate status and the navi binary path. Useful when diagnosing
  "why is the data stale" or "why did my write get rejected."

Resources are cheap — read them when they'd be helpful, no need to ask
permission.

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

The navi CLI has commands that are intentionally NOT wrapped as MCP tools.
They fall into three categories, each with a different behavior from Claude.

### Hazardous to automate — kept as CLI, recommended only when needed

These commands exist in navi, remain useful, but are deliberately kept out
of the MCP tool surface so an LLM cannot drive them on the user's behalf.
Claude explains them as CLI steps when a workflow naturally includes them.
Claude does not try to invoke them through any tool.

- **`navi action push`** — remote command execution against Linux hosts.
  Used in the Route → Tag → Push → Verify remediation cycle.
- **`navi action mail`** — email reports and files. Used as a chained step
  after exports or evaluations.

See navi-action for full CLI reference.

**Pattern for MCP → CLI handoff:**

When the workflow crosses from MCP tools into a CLI-only command, Claude
completes the MCP portion first, tells the user exactly what to run, and
then resumes on the MCP side if there's a verification step afterward.

**Worked example — Route → Tag → Push → Verify:**

> Here's the full remediation cycle for Jenkins hosts. I'll drive the
> tagging and verification through MCP tools; the push step is CLI-only,
> so you'll run that part yourself.
>
> **Step 1 (MCP) — build the route tag.** I'll query for every asset with a
> vuln path containing "jenkins" and tag them with `Remediation:Jenkins-Q2`.
>
> Tool call: `navi_enrich_tag(category="Remediation", value="Jenkins-Q2",
> query="SELECT DISTINCT asset_uuid FROM vuln_paths WHERE path LIKE '%jenkins%';",
> confirm=True)`
>
> Confirm to apply.
>
> *[...after user confirms and tool runs...]*
>
> **Step 2 (CLI) — run the remediation.** `navi action push` is not an MCP
> tool. Run this at your terminal:
>
> ```bash
> navi action push --tag "Remediation:Jenkins-Q2" \
>   --command "sudo apt-get upgrade jenkins -y"
> ```
>
> Wait for push to complete, then launch the verification scan. After that,
> let me know and I'll kick off step 3 on the MCP side.
>
> **Step 3 (MCP) — verify and clean up.** Once the remediation scan is done
> and vulns are re-synced, I'll confirm the vulnerability is gone and
> delete the temporary tag. Tell me when you're ready for that.

The important pattern: Claude does NOT try to orchestrate the CLI step
silently, does NOT pretend to invoke `action push` through an MCP tool,
and does NOT continue past the CLI step without user confirmation that
the CLI portion completed.

### Too heavy for a tool call — kept as CLI, actively encouraged

**`navi config update full`** is the foundational data sync that populates
navi.db from the Tenable platform. Without it, navi.db is empty or stale
and every read tool silently returns partial or useless results.

It is NOT exposed as an MCP tool because:

- First-run syncs on large tenants can pull hundreds of GB — 400GB+ is not
  unusual.
- Runtime is typically hours, sometimes longer.
- An MCP tool call timing out, retrying, or being interrupted mid-sync
  causes real damage to the local database.

But unlike push and mail, Claude *actively recommends* this command. It is
the single most important prerequisite for the entire navi-mcp toolchain.
When navi.db is empty or stale, nothing else Claude does is reliable.

**When Claude surfaces the `config update full` recommendation:**

- **On apparent first-run** — if the data-freshness check (see next section)
  shows navi.db is empty or has never been synced, Claude surfaces it before
  attempting any data-dependent workflow.
- **When stale-data symptoms appear** — queries returning unexpectedly small
  counts, freshness check showing the newest `last_found` is weeks old, or
  the user reports "missing assets" / "missing vulns."
- **After ACR changes and tagging writes** — these operations only reach
  Tenable's dashboards after a re-sync. Claude reminds the user once per
  write sequence that a `navi config update full` (or a targeted
  `navi_config_update`) at their terminal will reflect the change in the
  platform. The 30-minute propagation window applies to the tag itself;
  the re-sync is what pulls the updated state back into navi.db.

**How Claude surfaces it:**

> Before we go further: your navi.db was last synced 23 days ago
> (newest vuln `last_found` is Mar 29). For reliable results on this
> workflow, you'll want to refresh the local data. `navi config update full`
> is the authoritative sync, but it's a CLI command — run it at your
> terminal:
>
> ```bash
> navi config update full
> ```
>
> On a large tenant, this can take hours and pull hundreds of GB the first
> time. It's kept out of the MCP tool surface for exactly that reason.
> For a lighter targeted refresh (vulns only, assets only, etc.), I can
> run `navi_config_update(kind="vulns")` directly — that finishes in
> minutes and is fine as a tool call. Which would you like?

Claude does NOT nag — the recommendation surfaces at the points above, not
on every session or every message.

### Out of scope for navi-mcp entirely

These commands exist in navi but are not part of any navi-mcp workflow.
Claude does not recommend them, compose around them, or teach them.

- `navi action deploy` (all containers)
- `navi action automate`
- `navi action plan`
- `navi enrich attribute`
- `navi enrich migrate`
- `navi enrich tagrule`

If a user asks for any of these by name, Claude can acknowledge the command
exists in navi and suggest the user consult the navi project directly —
but does not produce runnable examples or work them into automation
sequences.

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

- **navi-core** — setup, schema, the 30-minute propagation rule,
  50K-asset scale fork, multi-workload pattern
- **navi-troubleshooting** — fix-it workflows for errors, empty results,
  slow tagging, post-upgrade recovery
- **navi-enrich** — tagging, the `remove=True` ephemeral pattern, tag UUID
  preservation
- **navi-acr** — Asset Criticality Rating adjustment, Change Reasons,
  mod set/inc/dec, suggested tier mapping
- **navi-explore** — data and info subcommands, raw SQL patterns
- **navi-export** — CSV exports
- **navi-scan** — scan creation, control, evaluate
- **navi-action** — delete, rotate, cancel, encrypt/decrypt; also push / mail (CLI-only)
- **navi-was** — WAS configs, scans, findings, tagging
