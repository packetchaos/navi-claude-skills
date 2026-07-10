# navi-mcp reference — commands not exposed through MCP

The full rationale for every navi CLI command intentionally kept off the MCP
tool surface, the one-time setup commands, and the worked MCP→CLI handoff
example. The lean list + handoff rule are in SKILL.md; load this when you need
the detail.

> **Now exposed (previously CLI-only):** `navi explore api` is now
> `navi_explore_api` (GET free; POST/PUT write-gated); `navi config
> certificates` is now `navi_config(kind="certificates")`; and `navi action
> mail` / `navi action push` are now `navi_action_mail` / `navi_action_push`,
> each double-gated behind its own capability env var (`NAVI_EMAIL` /
> `NAVI_REMOTE_CODE_EXECUTION`) on top of the write gate. They are no longer in
> the not-exposed set below — see the navi-mail and navi-remote-exec skills.

> **Targeted-sync 4-minute ceiling:** the host's ~4-min tool-call cap on long
> `navi_config_update(kind="vulns")` syncs (and the `days=N` / `--threads 1`
> fallback) is documented in **navi-core** — its canonical home.

## Commands not exposed through navi-mcp

The navi CLI has commands that are intentionally NOT wrapped as MCP tools.
They fall into three categories, each with a different behavior from Claude.

### Hazardous — exposed but double-gated

`navi action push` and `navi action mail` used to be kept off the tool surface
entirely. They are now exposed as `navi_action_push` / `navi_action_mail`, but
each requires a SECOND capability gate on top of the master write gate, plus
per-call `confirm=True`:

- **`navi_action_push`** — remote command execution against a single Linux host.
  Needs `NAVI_MCP_ALLOW_WRITES=1` **and** `NAVI_REMOTE_CODE_EXECUTION=1`. No
  `--tag` — one IP per call; loop per-IP for a group. Harness: navi-remote-exec.
- **`navi_action_mail`** — email reports and files. Needs
  `NAVI_MCP_ALLOW_WRITES=1` **and** `NAVI_EMAIL=1`. Harness: navi-mail.

The dedicated skills carry the full guidance; the rule here is only that both
are real tools now, gated more tightly than the ordinary platform-write tools.

**Worked example — Route → Tag → Push → Verify (all MCP now):**

> Here's the full remediation cycle for Jenkins hosts, driven entirely through
> MCP tools (push is a tool now, double-gated).
>
> **Step 1 — build the route tag.** Query every asset with a vuln path
> containing "jenkins" and tag them `Remediation:Jenkins-Q2`.
>
> `navi_enrich_tag(category="Remediation", value="Jenkins-Q2",
> query="SELECT DISTINCT asset_uuid FROM vuln_paths WHERE path LIKE '%jenkins%';",
> confirm=True)` — confirm to apply.
>
> *[...after the 30-min propagation window...]*
>
> **Step 2 — push the remediation.** push targets one IP, so read the tagged
> hosts' IPs, narrate the full set + command, then call the tool once per IP
> (each with confirm=True):
>
> `navi_explore_query(sql="SELECT a.ip FROM tags t JOIN assets a ON t.asset_uuid=a.uuid WHERE t.tag_key='Remediation' AND t.tag_value='Jenkins-Q2';")`
> then per IP:
> `navi_action_push(target="<ip>", command="sudo apt-get upgrade jenkins -y", confirm=True)`
>
> **Step 3 — verify and clean up.** After a re-scan and vuln re-sync, confirm
> the finding is gone (`navi_explore_query(...)`) and retire the tag with
> `navi_action_delete(kind="tag", category="Remediation", value="Jenkins-Q2", confirm=True)`.

Each write and each push still gets its own narration and confirmation — the
tools being available does not remove the per-call confirm step.

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
