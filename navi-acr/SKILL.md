---
name: navi-acr
description: >
  Asset Criticality Rating (ACR) adjustment skill for Tenable One. Use for any
  request involving calibrating ACR values, fixing inaccurate risk scores,
  setting business-tier-appropriate criticality, or explaining why Tenable
  One's default risk scores don't reflect organizational reality. Covers the
  navi_enrich_acr tool, the three mod operations (set/inc/dec), all four
  Change Reason flags (business/compliance/mitigation/development), and the
  suggested ACR tier mapping (10/9/8/6/3/2). Trigger on: "adjust ACR",
  "set ACR for production", "risk scores are wrong", "assets aren't
  prioritised correctly", "Tenable One isn't showing the right things",
  "how do I improve my AES scores", "set production assets as most critical",
  "calibrate criticality", "why is the test box showing critical exposure".
  Prerequisite: assets already tagged by business tier (see navi-enrich).
---

# Navi ACR — Asset Criticality Rating Calibration

ACR (Asset Criticality Rating) is a 1–10 score in Tenable One that directly
shapes everything leadership sees. It combines with vulnerability severity
to produce the **Asset Exposure Score (AES)** — the number that drives
dashboards, triage queues, and board-level risk reporting.

This skill covers `navi_enrich_acr` — adjusting ACR values across groups
of assets by tag. For the tagging that precedes ACR calibration, see
navi-enrich.

`navi_enrich_acr` is **write-gated.** Each call requires `confirm=True`
and requires the navi-mcp server to be running with
`NAVI_MCP_ALLOW_WRITES=1`. See navi-mcp for the write-gate convention —
Claude narrates the operation, states the exact tool call, and waits for
user confirmation before invoking.

---

## Why ACR matters — the default state problem

Every new asset in Tenable One gets a system-generated ACR based on
observable technical attributes: OS, open ports, plugin findings, scan
coverage. Tenable One has no idea which Linux server is your payment
processor vs. your developer's test machine.

Without business context from you, the risk scores treat those two assets
as roughly equivalent. Leadership ends up making decisions on numbers
that don't reflect what the organisation actually cares about:

- A dev workstation with a critical vuln shows up at the top of the
  exposure list alongside a production database with the same vuln.
- Triage queues surface noise because everything looks equally important.
- Board-level reporting can't distinguish regulated data from throwaway
  test infrastructure.

**The navi solution: Tag → ACR → recalibrated scores.** You use navi to
tag assets by business tier (production, dev, internet-facing, PII, etc.),
then adjust ACR per tag so Tenable One's prioritization reflects
organizational reality.

**Why this matters for Mythos-era defence**: When exploit windows collapse
to hours, you can't investigate everything. A well-calibrated AES score
tells you which hosts to act on first when a new CVE drops. Without
accurate ACR, that signal is noise. With it, prioritisation is automatic
and trustworthy.

**One-sentence version**: Tenable One's risk scores are only as accurate
as the ACR data behind them — and navi is the fastest way to make ACR
reflect what the business actually cares about.

---

## Tool signature

```
navi_enrich_acr(
    category,           # tag category (required)
    value,              # tag value (required)
    score,              # ACR value 1-10 (required)
    mod="set",          # "set" | "inc" | "dec" (default: "set")
    note=None,          # optional audit trail text
    business=False,     # Change Reason flag
    compliance=False,   # Change Reason flag
    mitigation=False,   # Change Reason flag
    development=False,  # Change Reason flag
    confirm=True,       # required for write gate
)
```

CLI equivalent (for standalone reference):

```bash
navi enrich acr --c <category> --v <value> --score <N> --mod set \
  -business   # or -compliance, -mitigation, -development
```

---

## Prerequisite — assets must be tagged first

ACR adjustment targets a tag category/value pair. Every asset carrying
that tag receives the adjustment. If the tag doesn't exist or doesn't
have the right assets on it, the ACR call has no effect (or the wrong
effect).

Before running `navi_enrich_acr`, verify the tag's membership:

`navi_explore_query(sql="SELECT count(*) FROM tags WHERE tag_key='<category>' AND tag_value='<value>';")`

If zero: tag doesn't exist or isn't applied. See navi-enrich for tagging
workflows.

For the full Tag → ACR sequence, see "The full pattern" below.

---

## Change Reasons — required for audit compliance

Tenable One requires at least one Change Reason on every ACR adjustment
for audit compliance. Pass one or more — multiple reasons are supported
on a single change (e.g. `business=True, compliance=True` for production
systems handling PII).

| Tagging rationale | Reason flag |
|---|---|
| Production, Internet-facing, critical business function | `business=True` |
| PII, PCI, PHI, regulated data | `compliance=True` |
| Isolated / air-gapped / mitigated (controls reduce risk) | `mitigation=True` |
| Staging, Dev, non-prod lifecycle | `development=True` |

The `note` parameter is optional free-form text attached to the change
for audit trail purposes. Use it to explain *why* this adjustment was
made, especially for non-obvious cases (e.g. `note="Elevated ACR during
Q2 Log4j response; revert after Aug 1"` for temporary increments).

**A call with no Change Reason flags will fail Tenable One's audit
requirement.** Always pass at least one.

---

## `mod` parameter — set vs. inc vs. dec

- **`mod="set"`** (default) — absolute ACR value per tier. This is the
  common case for initial calibration and tier rebalancing. Existing ACR
  is overwritten with `score`.
- **`mod="inc"`** — increment current ACR by `score`. Use for temporary
  adjustments (e.g. bumping ACR on assets flagged during an active
  incident). The per-asset ACR goes up by `score`, capped at 10.
- **`mod="dec"`** — decrement current ACR by `score`. Use to reverse a
  temporary increment once the triggering condition clears. Capped at 1
  (the minimum ACR).

**When to use which:**

- Initial calibration of your environment → `mod="set"` for every tier
- Monthly rebalancing as business priorities shift → `mod="set"`
- "Bump these assets up while we investigate" → `mod="inc"` with `score=2`
  or so
- "OK, investigation is complete, return to baseline" → `mod="dec"` with
  the same `score` used for the earlier `inc`

Keep the `note` parameter consistent across paired inc/dec operations so
the audit trail is easy to follow.

---

## The full pattern

Step 1 — tag by business tier. Use any tagging method from navi-enrich.
Common approaches:

`navi_enrich_tag(category="Environment", value="Production", group="Production Servers", confirm=True)`

`navi_enrich_tag(category="Environment", value="Internet-Facing", port=443, confirm=True)`

`navi_enrich_tag(category="Data Class", value="PII", query="SELECT asset_uuid FROM assets WHERE hostname LIKE '%db-prod%';", confirm=True)`

`navi_enrich_tag(category="Environment", value="Development", group="Dev Workstations", confirm=True)`

Step 2 — push ACR values that reflect business reality. Each call is
write-gated; each needs at least one Change Reason:

**Production — critical business function:**

`navi_enrich_acr(category="Environment", value="Production", score=10, mod="set", business=True, confirm=True)`

**PII data — regulated:**

`navi_enrich_acr(category="Data Class", value="PII", score=10, mod="set", compliance=True, note="PII data — GDPR/CCPA scope", confirm=True)`

**Internet-facing — critical business function:**

`navi_enrich_acr(category="Environment", value="Internet-Facing", score=9, mod="set", business=True, confirm=True)`

**Staging — dev lifecycle:**

`navi_enrich_acr(category="Environment", value="Staging", score=6, mod="set", development=True, confirm=True)`

**Dev — dev lifecycle:**

`navi_enrich_acr(category="Environment", value="Development", score=3, mod="set", development=True, confirm=True)`

**Isolated / air-gapped — compensating controls:**

`navi_enrich_acr(category="Environment", value="Isolated", score=2, mod="set", mitigation=True, note="Air-gapped segment, no external exposure", confirm=True)`

Step 3 — re-sync so Tenable One recalculates AES across all dashboards:

`navi_config_update(kind="assets")`

---

## Propagation window — why AES doesn't update immediately

After ACR writes, the **30-minute propagation window applies** — just
like tagging. The immediate `navi_config_update(kind="assets")` may not
reflect the new AES scores because Tenable hasn't finished propagating
the ACR changes internally yet.

Two implications for verification:

1. `navi_explore_query` against navi.db reflects the ACR write immediately
   after you re-sync (navi.db is the source of truth for your local view).
2. Tenable UI / `navi_explore_info(...)` may lag during the propagation
   window. If the user checks the dashboard right after an ACR write and
   the AES score hasn't changed, that's expected.

For the final authoritative refresh after the propagation window, run
`navi config update full` at the CLI. See navi-mcp's "Too heavy for a
tool call" section.

After propagation: the payment processor shows highest exposure. The test
workstation stops topping triage queues. Leadership gets accurate risk
communication. Remediation teams focus on the right assets.

---

## Suggested tier mapping

| Business tier | ACR |
|---|---|
| Production + PII / PCI / PHI data | 10 |
| Internet-facing / DMZ | 9 |
| Production (standard) | 8 |
| Staging / Pre-prod | 6 |
| Development / Test | 3 |
| Isolated / Air-gapped | 2 |

This mapping is a starting point. Adjust based on your organization's
actual exposure and regulatory profile. The tier structure matters more
than the exact numbers — ensure there's clear separation between
production and dev, and between regulated and unregulated data classes.

**ACR scale**: 1 = low criticality, 10 = mission-critical.

---

## Temporary ACR adjustments — incident-driven workflows

During active incidents, you often want to temporarily elevate the ACR
of affected assets so they surface higher in triage queues. Use
`mod="inc"` with a clear note, then reverse with `mod="dec"` when the
incident closes.

**During incident:**

`navi_enrich_acr(category="Incident", value="Log4j-Active", score=3, mod="inc", business=True, note="Active Log4j response; temporary bump", confirm=True)`

Assets carrying the `Incident:Log4j-Active` tag get their ACR bumped by
3 (capped at 10).

**After incident closes:**

`navi_enrich_acr(category="Incident", value="Log4j-Active", score=3, mod="dec", business=True, note="Log4j response complete; revert temporary bump", confirm=True)`

Same assets get their ACR reduced by 3, returning to baseline (capped at
1).

**Pair incident-driven tagging with `remove=True`** so the tag's
membership stays accurate as assets are confirmed clean:

`navi_enrich_tag(category="Incident", value="Log4j-Active", query="SELECT DISTINCT asset_uuid FROM vulns WHERE plugin_name LIKE '%Log4j%' AND state='active';", remove=True, confirm=True)`

See navi-enrich for the full `remove=True` ephemeral pattern.

---

## ACR → Natural Language

| User says | Tool call |
|---|---|
| "set production ACR to 10" | `navi_enrich_acr(category="Environment", value="Production", score=10, mod="set", business=True, confirm=True)` |
| "calibrate criticality for dev assets" | `navi_enrich_acr(category="Environment", value="Development", score=3, mod="set", development=True, confirm=True)` |
| "bump ACR during active incident" | `navi_enrich_acr(category="Incident", value="<name>", score=<N>, mod="inc", business=True, confirm=True)` |
| "return ACR to baseline after incident" | `navi_enrich_acr(category="Incident", value="<name>", score=<N>, mod="dec", business=True, confirm=True)` |
| "risk scores are inaccurate / Tenable One shows wrong priorities" | Start the Tag → ACR → re-sync pattern above |
| "how do I improve AES accuracy" | Same — tag by business tier, set ACR per tier |
| "isolated network shouldn't show high exposure" | `navi_enrich_acr(category="Environment", value="Isolated", score=2, mod="set", mitigation=True, confirm=True)` |

---

## Cross-references

- **navi-enrich** — tagging assets by business tier (prerequisite for ACR)
- **navi-mcp** — write-gate convention and confirmation pattern
- **navi-core** — 30-minute propagation window, `navi config update full`
- **navi-export** — `navi_export(subcommand="bytag")` is the only export
  that includes ACR + AES scores (useful for verification and exec reporting)
- **navi** (router) — Executive Dashboard integrates ACR-calibrated AES
  into the Top Assets by Exposure section
