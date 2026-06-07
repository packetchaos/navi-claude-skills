# navi-mcp reference — commands not exposed through MCP

The full rationale for every navi CLI command intentionally kept off the MCP
tool surface, the one-time setup commands, and the worked MCP→CLI handoff
example. The lean list + handoff rule are in SKILL.md; load this when you need
the detail.

> **Now exposed (previously CLI-only):** `navi explore api` is now
> `navi_explore_api` (GET free; POST/PUT write-gated), and `navi config
> certificates` is now `navi_config(kind="certificates")`. They are no longer
> in the not-exposed set below.

> **Targeted-sync 4-minute ceiling:** the host's ~4-min tool-call cap on long
> `navi_config_update(kind="vulns")` syncs (and the `days=N` / `--threads 1`
> fallback) is documented in **navi-core** — its canonical home.

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
> tool, and it targets a single IP (`--target`, no `--tag`). Pull the tagged
> hosts' IPs and loop. Run this at your terminal:
>
> ```bash
> for ip in $(navi explore data query \
>   "SELECT a.asset_ip FROM tags t JOIN assets a ON t.asset_uuid=a.uuid \
>    WHERE t.tag_key='Remediation' AND t.tag_value='Jenkins-Q2';"); do
>   navi action push --target "$ip" --command "sudo apt-get upgrade jenkins -y"
> done
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

These commands exist in navi but are intentionally CLI-only because their
runtime, scope, or "this is foundational; do it once deliberately"
character makes them a poor fit for tool-call lifecycles. Unlike the
hazardous-to-automate commands above, Claude *actively recommends* these
when relevant.

#### Long-running foundational syncs

**`navi config update full`** — the foundational data sync that populates
navi.db from the Tenable platform. Without it, navi.db is empty or stale
and every read tool silently returns partial or useless results.

Not exposed because:

- First-run syncs on large tenants can pull hundreds of GB — 400GB+ is
  not unusual.
- Runtime is typically hours, sometimes longer.
- An MCP tool call timing out, retrying, or being interrupted mid-sync
  causes real damage to the local database.

It is the single most important prerequisite for the entire navi-mcp
toolchain. When navi.db is empty or stale, nothing else Claude does is
reliable.

**When Claude surfaces the `config update full` recommendation:**

- **On apparent first-run** — if the data-freshness check (see next
  section) shows navi.db is empty or has never been synced, Claude
  surfaces it before attempting any data-dependent workflow.
- **When stale-data symptoms appear** — queries returning unexpectedly
  small counts, freshness check showing the newest `last_found` is weeks
  old, or the user reports "missing assets" / "missing vulns."
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

Claude does NOT nag — the recommendation surfaces at the points above,
not on every session or every message.

#### Deliberate one-time setup commands

These commands are fast (seconds to minutes) but are deliberate setup
operations that the operator runs once at the CLI. Claude can recommend
them when their absence is causing a problem, but does not invoke them
through any tool.

**`navi config optimize`** (navi 8.5.31+) — builds curated indexes
against `navi.db` for fast tagging and querying. `navi config update full`
does NOT build these indexes itself. Without optimize, tagging operations
can take hours; with it, they take seconds.

When Claude surfaces this:

- After an apparent first-run + first sync ("now run optimize once and
  every workflow afterward will be much faster")
- When tagging is slow despite navi.db being fresh
- After post-upgrade recovery (indexes are dropped when navi.db is
  rebuilt)

```bash
navi config optimize
```

Targeted `navi_config_update(kind="vulns")` calls through MCP build
indexes automatically — so users running incremental refreshes through
the tool get this for free. Optimize is mainly relevant for users who
do `update full` and then want fast performance.

**`navi config epss`** — downloads the EPSS CSV and populates the `epss`
table. Required before any EPSS-based query or join works (joining
vulns to EPSS scores in navi-explore depends on this). Claude recommends
running it when a user asks about EPSS-driven prioritization and the
table is empty.

```bash
navi config epss
```

**`navi config smtp`** — set up SMTP credentials for `navi action mail`.
Without it, `navi action mail` can't actually send. Cross-referenced
from navi-action's mail section.

**`navi config ssh`** — enter SSH service account credentials for
`navi action push`. Without it, `navi action push` has no way to
authenticate to remote hosts. Cross-referenced from navi-action's push
section.

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
