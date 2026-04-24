---
name: navi-scan
description: >
  Scan control skill for Tenable navi CLI. Covers navi_scan subcommands: create
  (build a new scan), start (launch a saved scan), stop (halt a running scan),
  evaluate (scanner performance analysis). Trigger on: "create a scan", "launch
  a scan", "start scan X", "kick off a scan", "stop the scan", "how's the scan
  doing", "why is scan X so slow", "evaluate scan performance", "analyze scanner
  load", "build a verification scan", "remediation scan", "scan this asset",
  "is my scanner load balanced". Companion: navi_explore_info subcommand="scans"
  / "scanners" / "policies" / "credentials" for pre-scan ID lookups.
---

# Navi Scan — Scan Control & Performance

Two surfaces for working with Tenable scans:

- **`navi_scan(subcommand=..., ...)`** — create, start, stop, evaluate.
  Control and diagnostics.
- **`navi_explore_info(subcommand=...)`** — list scans, scanners, policies,
  credentials, currently-running scans. Always current, live from the
  Tenable API.

When running under navi-mcp, use tool-invocation form (shown first in each
example). Bash forms are standalone CLI equivalents. See navi-mcp for the
write-gate convention — `create`, `start`, and `stop` are write-gated.
`evaluate` is read-only and doesn't require `confirm=True`.

---

## Recurring scans — use the Tenable UI

`navi_scan` covers one-off scan control (create, start, stop, evaluate).
**Recurring scan schedules should be set up in the Tenable UI**, not
through navi. The UI has a purpose-built scheduler with the right
guardrails for recurring operations.

If you're automating scans from a script or wrapper, keep the following
in mind:

- **Reuse scan targets across runs.** Instead of creating a new scan each
  time, create once and re-`start` the same scan. Every
  `navi_scan(subcommand="create", ...)` call produces a new scan object;
  automated create loops fill up the scan inventory quickly.
- **Clean up old scans monthly.** Stale scan objects accumulate and slow
  the TVM UI down measurably. Past ~10,000 scans, dashboard load times
  and scan pickers become noticeably slower.
  `navi_action_delete(kind="scan", id=<id>, confirm=True)` is the tool
  for removing them; pair with `navi_explore_info(subcommand="scans")`
  to identify candidates for cleanup.
- **Prefer scheduled exports over scheduled scans for reporting.** If the
  output you want is a CSV, a cron-driven `navi_export` is lighter than
  a full scan.

---

## Prerequisite lookups — get IDs before creating or starting

Every non-trivial scan operation needs one or more IDs from Tenable. Look
them up with `navi_explore_info` before composing a scan call.

| Before running… | Look up via |
|---|---|
| `navi_scan(subcommand="create", scanner_id=...)` | `navi_explore_info(subcommand="scanners")` |
| `navi_scan(subcommand="create", policy_id=...)` | `navi_explore_info(subcommand="policies")` |
| `navi_scan(subcommand="create", credential_uuid=...)` | `navi_explore_info(subcommand="credentials")` |
| `navi_scan(subcommand="start", scan_id=...)` | `navi_explore_info(subcommand="scans")` |
| `navi_scan(subcommand="stop", scan_id=...)` | `navi_explore_info(subcommand="running")` |
| `navi_scan(subcommand="evaluate", scan_id=...)` | `navi_explore_info(subcommand="scans")` |

**Trigger phrases:** "I need a scanner ID", "what policies do I have",
"list saved credentials", "show me scan IDs"

---

## `navi_scan(subcommand="create", ...)` — build a new scan

Creates a new scan without launching it. Write-gated — narrate the
operation and confirm before invoking.

**Required:** `targets` (comma-separated IPs, hostnames, CIDR ranges, or
a mix).

**Optional but usually needed:**

- `scanner_id` — which Tenable scanner will run it (get from
  `navi_explore_info(subcommand="scanners")`)
- `policy_id` — which scan policy / template to use (get from
  `navi_explore_info(subcommand="policies")`)
- `credential_uuid` — credentialed scan, maps to a saved credential
  (get from `navi_explore_info(subcommand="credentials")`)
- `plugin` — restrict the scan to a single plugin ID (useful for
  verification scans after remediation)
- `name` — custom scan name

### Examples

**Basic discovery scan against a subnet:**

`navi_scan(subcommand="create", targets="192.168.10.0/24", name="Subnet-10-discovery", confirm=True)`

```bash
navi scan create 192.168.10.0/24 --name "Subnet-10-discovery"
```

**Credentialed scan with a specific policy and scanner:**

`navi_scan(subcommand="create", targets="10.0.5.12,10.0.5.13,10.0.5.14", scanner_id="<SCANNER_ID>", policy_id="<POLICY_ID>", credential_uuid="<CRED_UUID>", name="DB tier patch verification", confirm=True)`

```bash
navi scan create 10.0.5.12,10.0.5.13,10.0.5.14 \
  --scanner <SCANNER_ID> --policy <POLICY_ID> --cred <CRED_UUID> \
  --name "DB tier patch verification"
```

**Single-plugin verification scan:**

Useful after remediation — scan just the specific plugin you're trying to
clear rather than running a full scan.

`navi_scan(subcommand="create", targets="<TARGETS>", plugin=10863, name="Cert expiry verification", confirm=True)`

```bash
navi scan create <TARGETS> --plugin 10863 --name "Cert expiry verification"
```

**Trigger phrases:** "create a scan for X", "build a scan against X",
"set up a verification scan", "scan these specific assets"

---

## `navi_scan(subcommand="start", ...)` — launch a saved scan

Starts an existing scan by ID. Write-gated.

`navi_scan(subcommand="start", scan_id="<SCAN_ID>", confirm=True)`

```bash
navi scan start <SCAN_ID>
```

Use `navi_explore_info(subcommand="scans")` to get the scan ID first. Use
`navi_explore_info(subcommand="running")` afterward to confirm the scan
actually started and is progressing.

**Trigger phrases:** "start scan X", "launch the X scan", "kick off X",
"run the saved scan"

---

## `navi_scan(subcommand="stop", ...)` — halt a running scan

Stops a currently-running scan. Write-gated.

`navi_scan(subcommand="stop", scan_id="<SCAN_ID>", confirm=True)`

```bash
navi scan stop <SCAN_ID>
```

Check `navi_explore_info(subcommand="running")` first to confirm the scan
is actually running and get its ID if you don't have it.

**Trigger phrases:** "stop scan X", "halt the scan", "the scan is stuck,
kill it", "abort scan X"

---

## `navi_scan(subcommand="evaluate", ...)` — scanner performance analysis

**Read-only.** No confirmation required. Reports scan-time averages across
three views — by policy, by scanner, and by scan name — using plugin 19506
data from the local navi.db.

`navi_scan(subcommand="evaluate", scan_id="<SCAN_ID>")`

```bash
navi scan evaluate --scanid <SCAN_ID>
```

**Refresh navi.db before running.** Evaluate reads the local database, not
the live Tenable API. Stale data means stale averages. Run
`navi_config_update(kind="vulns")` first for a targeted refresh, or
`navi config update full` at the CLI for a foundational sync.

### What evaluate returns

Three breakdowns of average scan time per asset, each counted against the
asset total:

- **Policies** — average scan time per asset, grouped by scan policy.
  Useful for comparing policy overhead: a "Basic Network Scan" vs. a
  "Credentialed Host Audit" will have very different per-asset times.
- **Scanners** — per-scanner averages. Uneven distribution across
  scanners is the signal here — one scanner averaging 12 min/asset while
  another averages 6 min/asset on the same policy usually means the slow
  scanner is either doing more work (co-hosted with other scans) or has
  a network-latency problem reaching its assigned targets.
- **Scan Name** — per-scan averages. Different scans running the same
  policy can still have different averages because of target composition.

### Example output

```
Evaluating multiple scans on 33 assets

Policies                                            AVG Minutes Per/Asset     Total Assets
Basic Network Scan                                                      7               33
---
Scanners                                            AVG Minutes Per/Asset     Total Assets
192.168.128.8                                                           6               29
192.168.113.150                                                        12                4
---
Scan Name                                           AVG Minutes Per/Asset     Total Assets
Daily Scan Production Network                                           6               29
Vulnerable Network Scan                                                12                4
```

Reading this example: 33 assets across one policy. Scanner 192.168.113.150
is averaging twice the per-asset time of 192.168.128.8 — but it's only
running against 4 assets via the "Vulnerable Network Scan." That could
mean: those 4 hosts are genuinely slow to scan (slow network, deep
credentialed work), or the scanner at .150 has other load during the scan
window. Investigating further means looking at the individual slow hosts
(via `navi_explore_data(subcommand="scantime", minutes=10)`) or checking
whether the scanner is running concurrent scans.

### In larger environments

The output expands as you have more policies, scanners, and scans. A
tenant running 20 active scans across 5 scanners with 3 policies produces
a much longer report — each section lists every unique policy/scanner/scan
with its own average. The interpretation logic is the same; the signals
just take more reading.

### When to run evaluate

- **After a scan that "took forever"** — figure out whether the slowness
  is a specific scanner, policy, or scan config.
- **Before changing scanner assignments** — establish a baseline so you
  can tell whether changes helped.
- **When SLA reporting looks wrong** — if
  `navi_export(subcommand="failures")` shows assets with surprising
  last-found dates, evaluate the scans that should have covered them.
- **After upgrading Nessus or a scanner version** — confirm averages
  haven't regressed.
- **Periodically as a scanner-load health check** — uneven per-scanner
  averages are an early warning that scan assignments need rebalancing.

### Companion diagnostics for slow scans

When evaluate shows a scanner or policy averaging high, these follow-up
queries localize the problem:

Assets with long individual scan times:

`navi_explore_data(subcommand="scantime", minutes=10)`

Assets with credential failures (often a cause of cascading timeouts):

`navi_explore_data(subcommand="creds")`

Tag slow assets for follow-up (ephemeral, auto-refreshes):

`navi_enrich_tag(category="Scan Health", value="Slow Scan", scantime=10, remove=True, confirm=True)`

Tag cred-failure assets for follow-up:

`navi_enrich_tag(category="Scan Health", value="Cred Failure", plugin=104410, remove=True, confirm=True)`

See navi-enrich for the full ephemeral tagging pattern.

**Trigger phrases:** "evaluate scan X", "why is scan X slow", "analyze
scanner performance", "is my scanner load balanced", "what's the
bottleneck on scan X"

---

## Workflow examples

### New scan setup — first time scanning a new subnet

> I'll set up a credentialed scan against the 10.0.5.0/24 subnet. First
> let me pull the IDs I'll need.
>
> `navi_explore_info(subcommand="scanners")` — shows which scanner can
> reach that subnet
> `navi_explore_info(subcommand="policies")` — shows which policy
> templates are available
> `navi_explore_info(subcommand="credentials")` — shows saved credentials
>
> *[reviews output, picks scanner/policy/cred with the user]*
>
> Now I'll create the scan.
>
> Tool call: `navi_scan(subcommand="create", targets="10.0.5.0/24",
> scanner_id="<SCANNER>", policy_id="<POLICY>", credential_uuid="<CRED>",
> name="DMZ-10-5 baseline", confirm=True)`
>
> Confirm to create.
>
> *[user confirms; scan is created but not launched]*
>
> Ready to launch? I'll use the returned scan_id:
>
> Tool call: `navi_scan(subcommand="start", scan_id="<NEW_SCAN_ID>", confirm=True)`

### Remediation verification loop

After pushing a fix, you want to confirm the vuln cleared. This pairs
with the Route → Tag → Push → Verify cycle in navi-action — specifically
the verify step.

> **Step 1 (MCP) — create a verification scan.** Targeted to just the
> plugin you're verifying, against the tagged remediation group.
>
> First get asset IPs from the tag:
> `navi_explore_query(sql="SELECT asset_ip FROM tags t JOIN assets a ON t.asset_uuid = a.uuid WHERE t.tag_key = 'Remediation' AND t.tag_value = 'OpenSSL-Q2';")`
>
> Build the comma-separated target list from the results, then:
>
> `navi_scan(subcommand="create", targets="<IPs>", plugin=10863, name="OpenSSL verify", confirm=True)`
>
> **Step 2 (MCP) — start it.**
>
> `navi_scan(subcommand="start", scan_id="<ID>", confirm=True)`
>
> **Step 3 (MCP) — wait for completion, then re-sync vulns.**
>
> Check status: `navi_explore_info(subcommand="running")` — if the scan
> is still listed, it's not done. When it's not listed, run:
>
> `navi_config_update(kind="vulns")`
>
> **Step 4 (MCP) — verify.**
>
> `navi_explore_query(sql="SELECT count(*) FROM vulns WHERE plugin_id=10863 AND state='active' AND asset_uuid IN (SELECT asset_uuid FROM tags WHERE tag_key='Remediation' AND tag_value='OpenSSL-Q2');")`
>
> If zero: remediation confirmed. Retire the tag via
> `navi_action_delete(kind="tag", category="Remediation", value="OpenSSL-Q2", confirm=True)`.

### Investigating a slow scan

> The X scan took 4 hours this week (normally 45 min). Let me evaluate.
>
> First, refresh vulns data so the averages are current:
> `navi_config_update(kind="vulns")`
>
> Then:
> `navi_scan(subcommand="evaluate", scan_id="<ID>")`
>
> *[reviews output — say one policy averaging 18 min/asset against a
> normal baseline of 7, one scanner averaging 22 min/asset vs. 6 on the
> other two scanners, a specific scan name accounting for most of the
> divergence]*
>
> Three things happening:
>
> 1. Scanner 192.168.X.X averaging 22 min/asset is the main outlier —
>    either it's overloaded with concurrent scans or has a network path
>    problem to its assigned targets. Check running scans: `navi_explore_info(subcommand="running")`.
> 2. The specific scan is heavier per-asset than peers — target
>    composition may include slow-responding hosts. Dig in:
>    `navi_explore_data(subcommand="scantime", minutes=10)`.
> 3. If cred failures show up downstream in the individual host checks,
>    those cascade into per-host timeouts. Tag them for follow-up:
>    `navi_enrich_tag(category="Scan Health", value="Cred Failure",
>    plugin=104410, remove=True, confirm=True)`.
>
> Want me to dig into any of those?

---

## Scan → Natural Language

| User says | Tool call |
|---|---|
| "create a scan for X" | `navi_scan(subcommand="create", targets=<targets>, ..., confirm=True)` |
| "scan these specific IPs" | `navi_scan(subcommand="create", targets="<ips>", confirm=True)` |
| "build a verification scan for plugin X" | `navi_scan(subcommand="create", targets=<targets>, plugin=<id>, confirm=True)` |
| "start scan X" / "launch X" | `navi_scan(subcommand="start", scan_id="<id>", confirm=True)` |
| "stop scan X" / "kill the scan" | `navi_scan(subcommand="stop", scan_id="<id>", confirm=True)` |
| "why is scan X slow" | `navi_scan(subcommand="evaluate", scan_id="<id>")` |
| "evaluate scanner performance" | `navi_scan(subcommand="evaluate", scan_id="<id>")` |
| "is my scanner load balanced" | `navi_scan(subcommand="evaluate", scan_id="<id>")` |
| "set up a recurring scan" | Use Tenable UI; see "Recurring scans" above |
| "what scans exist" / "list my scans" | `navi_explore_info(subcommand="scans")` |
| "what's scanning right now" | `navi_explore_info(subcommand="running")` |
| "show me scanners" | `navi_explore_info(subcommand="scanners")` |
| "show me policies" / "scan templates" | `navi_explore_info(subcommand="policies")` |
| "show me saved credentials" | `navi_explore_info(subcommand="credentials")` |
| "clean up old scans" | `navi_explore_info(subcommand="scans")` + `navi_action_delete(kind="scan", id=<id>, confirm=True)` per stale scan |
