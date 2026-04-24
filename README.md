# navi-claude-skills

Claude skill set for the [navi](https://github.com/packetchaos/navi) CLI —
a Python wrapper around the Tenable Vulnerability Management API.

This repository contains 11 Claude skills that teach Claude how to drive navi
effectively, whether through the navi-mcp server (tool-based) or as CLI
reference material. Skills are structured so Claude loads only the relevant
ones for a given task, rather than consuming a single monolithic reference.

## What's inside

| Skill | Lines | Purpose |
|---|---:|---|
| [`navi`](./navi/SKILL.md) | 391 | Router — entry point, Executive Dashboard, natural-language master index |
| [`navi-mcp`](./navi-mcp/SKILL.md) | 432 | MCP conventions — write-gate pattern, freshness check, commands not exposed |
| [`navi-core`](./navi-core/SKILL.md) | 595 | Setup, navi.db schema, config commands, scale fork, multi-workload pattern |
| [`navi-troubleshooting`](./navi-troubleshooting/SKILL.md) | 272 | Fix-it workflows — zero chunks, db locks, slow tagging, post-upgrade recovery |
| [`navi-enrich`](./navi-enrich/SKILL.md) | 564 | Tagging — 20+ selectors, ephemeral `remove=True` pattern, tag UUID preservation |
| [`navi-acr`](./navi-acr/SKILL.md) | 308 | Asset Criticality Rating — Change Reasons, tier mapping, incident workflows |
| [`navi-explore`](./navi-explore/SKILL.md) | 459 | Query surfaces — `explore data`, `explore info`, raw SQL against navi.db |
| [`navi-export`](./navi-export/SKILL.md) | 302 | CSV exports — all 15 subcommands including bytag (the only ACR+AES export) |
| [`navi-scan`](./navi-scan/SKILL.md) | 392 | Scan control — create, start, stop, evaluate; scanner load diagnostics |
| [`navi-action`](./navi-action/SKILL.md) | 468 | Operations — delete, rotate, cancel, encrypt; push and mail (CLI-only) |
| [`navi-was`](./navi-was/SKILL.md) | 239 | Web Application Scanning — configs, findings, WAS tagging |

**Total: 4,422 lines across 11 skills.**

## Who this is for

- **Operators running navi-mcp** who want Claude to drive their Tenable VM
  tenant through MCP tools with the right conventions baked in.
- **Standalone navi users** who want reference material for the CLI — every
  skill includes CLI equivalents for users outside an MCP context.
- **Anyone integrating Claude with Tenable VM** who wants a worked example
  of a skill set designed around MCP tool conventions (write-gate ceremony,
  freshness checks, hazardous-command CLI handoff, etc.).

## How Claude uses these skills

Skills are loaded by Claude based on their `description` frontmatter — the
user's message triggers skills whose descriptions match the intent. For
navi-specific requests, Claude typically loads:

- **`navi`** (the router) — for ambiguous requests, the Executive Dashboard,
  or when the right domain skill isn't immediately obvious
- **`navi-mcp`** — whenever the `navi_*` MCP tools are available in the
  session
- **A domain skill** — based on the specific intent (tagging, exploration,
  export, etc.)

Skills cross-reference each other, so loading one often brings in
relevant content from others contextually.

## Using with navi-mcp

The [navi-mcp](https://github.com/packetchaos/navi-mcp) server exposes a
`navi_workflow` prompt that injects a SKILL.md file as Claude context.
Point it at the router:

```bash
export NAVI_SKILL_PATH=/path/to/navi-claude-skills/navi/SKILL.md
```

Claude loads the router skill, which routes to domain skills based on the
user's request. The domain skills need to be accessible to Claude's skill
loader — exact mechanism depends on your Claude deployment.

For Claude Desktop and Claude Code, skills are typically loaded from a
skills directory; copy or symlink this entire repo into that directory.

## Using standalone (without navi-mcp)

Every skill includes CLI equivalents alongside the MCP tool forms. If
you're using navi directly rather than through navi-mcp, the skills still
work as reference documentation — navigate to the skill matching your
task and follow the CLI examples.

The [`navi`](./navi/SKILL.md) router has a natural-language master index
that maps common user phrasings to the right domain skill and command.

## Design principles

Every skill in this set follows these conventions:

1. **Tool-invocation form first** — when an MCP tool exists, it's shown
   before the CLI equivalent. CLI is secondary reference.
2. **Write-gate ceremony** — write operations narrate in prose, state the
   exact tool call, and wait for user confirmation before invoking with
   `confirm=True`.
3. **`remove=True` for ephemeral tags** — the pattern preserves tag UUIDs
   across refreshes, which matters for access groups, dashboards, and API
   integrations that reference tags by UUID. Never delete-and-recreate.
4. **Stable tag values with rotating queries** — monthly cert rotations
   and other recurring ephemeral tags use a stable `category:value` pair
   and change only the query.
5. **Clean cuts, no footnotes** — commands not exposed through navi-mcp
   (push, mail, deploy, automate, plan, etc.) are either kept fully as
   CLI-only or cleanly removed, never left as stubs.
6. **Read-first before writes** — schema lookups via `navi://schema/{table}`,
   ID lookups via `navi_explore_info(...)`, and count/freshness checks via
   `navi_explore_query(...)` all happen before write operations.

See [`navi-mcp/SKILL.md`](./navi-mcp/SKILL.md) for the full conventions
reference.

## Repository structure

```
navi-claude-skills/
├── README.md                         ← you are here
├── LICENSE                           ← MIT
├── navi/
│   └── SKILL.md                      ← router + Executive Dashboard
├── navi-mcp/
│   └── SKILL.md                      ← MCP conventions
├── navi-core/
│   └── SKILL.md                      ← setup, schema, config
├── navi-troubleshooting/
│   └── SKILL.md                      ← fix-it workflows
├── navi-enrich/
│   └── SKILL.md                      ← tagging
├── navi-acr/
│   └── SKILL.md                      ← ACR calibration
├── navi-explore/
│   └── SKILL.md                      ← querying
├── navi-export/
│   └── SKILL.md                      ← CSV exports
├── navi-scan/
│   └── SKILL.md                      ← scan control
├── navi-action/
│   └── SKILL.md                      ← operations
└── navi-was/
    └── SKILL.md                      ← Web Application Scanning
```

Each skill is a standalone folder containing a single `SKILL.md` — the
Anthropic skill loader convention.

## Dependencies between skills

Most skills cross-reference each other. The router (`navi`) knows about
all of them; `navi-mcp` is referenced from every skill that involves tool
calls. Common compose patterns:

- **Tag + ACR + re-sync** → `navi-enrich` + `navi-acr` + `navi-core`
- **Route → Tag → Push → Verify remediation** → `navi-enrich` + `navi-action` (CLI push) + `navi-scan`
- **Executive Dashboard** → `navi` (router) + `navi-export` + `navi-explore` + `navi-scan`
- **Weekly operational hygiene** → `navi-action` (orchestration) + `navi-enrich` + `navi-export`

## Contributing

Issues and PRs welcome. A few notes for contributors:

- **Ground truth is the navi-mcp server code.** Tool signatures, write-gate
  behavior, and subcommand Literals should match what
  [navi-mcp](https://github.com/packetchaos/navi-mcp)'s `server.py` actually
  exposes. When the server changes, skills need to follow.
- **The design principles listed above are load-bearing.** New content
  should follow the tool-first, narrate-writes, `remove=True`-not-delete
  patterns.
- **When cutting content, cut cleanly.** Don't leave footnotes about
  commands that used to be here — the "Commands not exposed through
  navi-mcp" list in `navi-mcp/SKILL.md` is the single place those live.
- **Test cross-references before merging.** If you rename a section or
  move content between skills, grep the rest of the set for pointers
  that need updating.

## Related projects

- [navi](https://github.com/packetchaos/navi) — the underlying CLI
- [navi-mcp](https://github.com/packetchaos/navi-mcp) — MCP server that
  wraps navi and injects these skills

## License

MIT — see [LICENSE](./LICENSE).
