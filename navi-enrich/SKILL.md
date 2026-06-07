---
name: navi-enrich
description: >
  Asset tagging skill for Tenable navi CLI. Use for ANY request involving tagging
  assets in Tenable VM: tag by plugin, CVE, CPE, CISA KEV, port, route, file, manual
  UUID, scan ID, agent group, AD group, cross-references, custom SQL query, or
  tag-on-tag derivation. Also covers adding assets from external sources
  (navi_enrich_add). Critical patterns: ephemeral tagging with remove=True for
  health tags (cred failures, slow scans, CISA KEV), tag UUID preservation
  (do NOT delete-and-recreate), scale fork at 50K assets, 30-minute propagation
  window for tags in the Tenable UI/API. Trigger on: "tag all assets", "create
  a tag", "how do I tag", "enrich", "refresh a tag", "import assets". For
  Asset Criticality Rating (ACR) adjustment, see navi-acr.
---

# Navi Enrich — Tagging & Asset Enrichment

Two enrichment tools covered here: `navi_enrich_tag` (tagging) and
`navi_enrich_add` (importing assets from external sources). Both are
**write-gated** — each call requires `confirm=True` and requires the
navi-mcp server to be running with `NAVI_MCP_ALLOW_WRITES=1`.

**For ACR adjustment (`navi_enrich_acr`), see navi-acr.** ACR workflows
are conceptually distinct from tagging — tagging establishes which
assets are what; ACR assigns criticality to those tagged groups.

See navi-mcp for the full write-gate convention: Claude describes the
operation in prose, states the exact tool call, and waits for user
confirmation before invoking.

When running under navi-mcp, use tool-invocation form (shown first in each
example below). Bash forms are standalone CLI equivalents for readers
outside an MCP context.

---

## Prereqs

- **Data freshness**: tagging works off the local navi.db. If the data is
  stale, tags apply to a stale picture. Under navi-mcp, the freshness
  check runs automatically at the start of a data-dependent workflow —
  see navi-mcp. If navi.db is empty or very old, the foundational
  `navi config update full` needs to run at the user's terminal before
  tagging is reliable.
- **Propagation window**: after a tagging write, allow **up to 30 minutes**
  for tags to be fully visible in the Tenable UI/API. This is a
  platform-side delay, not a navi delay. `navi_explore_query` against
  navi.db reflects the write immediately; `navi_explore_info` against the
  Tenable API can lag during the propagation window. The same window
  applies to ACR changes — see navi-acr for the ACR-specific timing.
- **Scale fork**: check asset count first —
  `navi_explore_query(sql="SELECT count(uuid) FROM assets;")`. Under 50K
  uses plugin regex; over 50K uses dedicated `certs`/`software` tables
  via `query=...`.
- **Slow tagging?** First try `navi config optimize` at the CLI (navi
  8.5.31+) — it builds curated indexes that fix most slowness in seconds.
  If you're on older navi or need custom indexes, see navi-troubleshooting
  for the manual SQL index fallback. For repeated workloads against a
  specific subset, a purpose-built navi directory (see navi-core's
  multi-workload pattern) is structurally faster.

---

## `navi_enrich_tag` — selectors & use cases

**Required args:** `category`, `value`, and `confirm=True` at call time.
**Optional:** `description` for a human-readable tag description.

Server enforces: pass exactly ONE primary selector per call. `plugin_output`
and `plugin_regexp` are modifiers that require `plugin`. `xid` requires
`xrefs`. `histid` requires `scanid`. `require_both=True` requires both
`parent_category` and `parent_value`.

Every tag call is write-gated. Examples show the tool form with `confirm=True`;
in actual use, Claude narrates first and then asks for confirmation before
invoking.

### Selector quick map

One primary selector per call. Full per-selector examples (tool + CLI) are in
**`references/selectors.md`** (`navi://skill/enrich/selectors`).

| Selector | Tags assets by | Tool param |
|---|---|---|
| plugin (+ output / regexp) | a plugin firing, optionally matching output text | `plugin=`, `plugin_output=` |
| `plugin_name` | text in the plugin NAME | `plugin_name=` |
| `cve` / `cpe` | a CVE / CPE identifier | `cve=` / `cpe=` |
| `xrefs` (+ `xid`) | a cross-reference type (e.g. CISA) | `xrefs=`, `xid=` |
| `port` | a vuln on a given port | `port=` |
| `route_id` | a route in `vuln_route` | `route_id=` |
| `file` / `manual` | IPs in a CSV / explicit UUIDs | `file=` / `manual=` |
| `group` / `byadgroup` | an agent group / AD groups in a CSV | `group=` / `byadgroup=` |
| `missed` | agents that missed auth in N days | `missed=` |
| `scanid` (+ `histid`) / `scantime` | a scan / assets scanning > N min | `scanid=` / `scantime=` |
| `query` | a custom SELECT returning `asset_uuid` | `query=` |
| `by_tag` / `by_cat` / `by_val` | derivation from existing tags | `by_tag=` etc. |
| hierarchical | child→parent tag relationships | `parent_category=`, `parent_value=`, `require_both=` |
| modes | ephemeral refresh / TONE tag | `remove=True` / `tone=True` |

### Tagging use-case playbook

Common real-world tagging goals → the selector to reach for:

- **IoT / device fingerprinting** (correct assets Nessus mislabels as "Linux"):
  SSL-cert appliances via `plugin=10863` + `plugin_output=`; local-network IoT
  (e.g. Chromecast) via `plugin=66717`; MAC OUI via `plugin=35716`.
- **Software inventory** ("where is `<package>`?"): `plugin_name=`, or `plugin=`
  + `plugin_output=` against software plugins (20811 Win / 22869 Linux /
  83991 Mac), or a `query=` on the `software` table. Classic case: find
  tcpdump / wireshark.
- **Plugin family** ("where am I using `<tech>`?", e.g. AI): `query=`
  `"SELECT DISTINCT asset_uuid FROM vulns WHERE plugin_family='Artificial Intelligence';"`
  — no native family selector, so use `query`.
- **User access / offboarding** (which assets a local user can reach): `query=`
  on the local-user enumeration plugins (95928 Linux / 71246 Windows), matching
  the username in plugin output.
- **CISA KEV** (actively-exploited): `xrefs="CISA"`, refreshed as an
  **ephemeral** tag (`remove=True`) on each KEV release.
- **Bulk by CVE from an external CSV** (e.g. MITRE ATT&CK→CVE): the `cve=`
  selector in a download→parse→loop — worked example in
  **`references/tag-by-cve-external-csv.md`** (`navi://skill/enrich/tag-by-cve-external-csv`).
- **Slow-to-scan assets**: `scantime=<minutes>` (e.g. "Long Scan Times").

## The `remove=True` ephemeral tagging pattern

**The problem**: without `remove=True`, tags accumulate. Fixed assets stay
tagged. A credential failure tag run twice = 12 tagged on Monday, 8 fixed
Tuesday, still 12 tagged.

**The solution**: `remove=True` clears all assets with this tag before
re-applying. Result: tag always reflects current reality.

### Tag UUID preservation — why `remove=True` matters beyond cleanup

`remove=True` keeps the existing tag UUID intact. It only reassigns which
assets carry the tag; the tag itself is the same object each run.

This matters because **access groups, API integrations, saved dashboards,
and external automation often reference tags by UUID**. Deleting a tag
and recreating it — even with the same category and value — generates a
NEW UUID and silently breaks every downstream reference.

The correct pattern for any "refresh this tag's membership" workflow is:

- Same `category`, same `value`
- Updated `query` (or whatever selector) to reflect current conditions
- `remove=True` to replace the old asset membership with the new

Do NOT use `navi_action_delete(kind="tag", ...)` followed by a fresh
`navi_enrich_tag(...)` to "rotate" a tag's contents. That's the
UUID-breaking pattern. Use `remove=True` alone.

### Always use `remove=True` for operational health tags

Cred failure tag:

`navi_enrich_tag(category="Scan Health", value="Cred Failure", plugin=104410, remove=True, confirm=True)`

Auth issue tag:

`navi_enrich_tag(category="Scan Health", value="Auth Issue", plugin=21745, remove=True, confirm=True)`

Slow scan tag:

`navi_enrich_tag(category="Scan Health", value="Slow Scan", scantime=30, remove=True, confirm=True)`

Reboot required:

`navi_enrich_tag(category="Remediation", value="Reboot Required", plugin=35453, remove=True, confirm=True)`

CISA KEV:

`navi_enrich_tag(category="CISA", value="KEV", xrefs="CISA", remove=True, confirm=True)`

Upcoming cert expiry (stable value, rotating query):

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Apr%2026%';", remove=True, confirm=True)`

### Monthly cert expiry rotation — the right way

Use a stable tag value and rotate only the query. Downstream references
to `CertExpiry:ExpiringSoon` keep working across rotations because the
UUID never changes.

This month:

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'May%2026%';", remove=True, confirm=True)`

Next month — same tag, new query:

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Jun%2026%';", remove=True, confirm=True)`

Month patterns: `Jan%2026%`, `Feb%2026%`, `Mar%2026%`, ..., `Dec%2026%`.

For a "certs expiring in the next N days" rotation that doesn't need
month boundaries:

`navi_enrich_tag(category="CertExpiry", value="Next60Days", query="SELECT asset_uuid FROM certs WHERE date(not_valid_after) <= date('now', '+60 days') AND date(not_valid_after) >= date('now');", remove=True, confirm=True)`

**Do NOT use `remove=True`** for stable classifications (Environment, OS,
Route) — these accumulate correctly and represent permanent context, not
point-in-time state.

**Trigger for `remove=True` suggestion**: user says "assets are still
tagged after we fixed them," "the tag is stale," "I deleted the tag and
now my access group is broken," or "how do I refresh a tag."

---

## Certificate expiry tagging (scale fork)

Check count first:

`navi_explore_query(sql="SELECT count(uuid) FROM assets;")`

**PATH A — under 50K (plugin regex):**

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", plugin=10863, plugin_regexp="Not After\s*:\s*Apr\s+\d{1,2}\s+[\d:]+\s+2026", remove=True, confirm=True)`

```bash
navi enrich tag --c "CertExpiry" --v "ExpiringSoon" \
  --plugin 10863 -regexp "Not After\s*:\s*Apr\s+\d{1,2}\s+[\d:]+\s+2026" -remove
```

**PATH B — over 50K (certs table — much faster):**

First populate the certs table: `navi_config_update(kind="certificates")`

Then:

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Apr%2026%';", remove=True, confirm=True)`

```bash
navi enrich tag --c "CertExpiry" --v "ExpiringSoon" \
  --query "SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Apr%2026%';" -remove
```

Both PATH A and PATH B use `remove=True` and a stable value
(`ExpiringSoon`) so the tag UUID is preserved across monthly rotations.
See "Monthly cert expiry rotation" above.

Month patterns: `Jan%2026%`, `Feb%2026%`, `Mar%2026%`, etc.


---

## Software tagging (scale fork)

**PATH A — under 50K:**

Linux:

`navi_enrich_tag(category="Software", value="Splunk", plugin=22869, plugin_output="splunk", confirm=True)`

Windows:

`navi_enrich_tag(category="Software", value="Splunk", plugin=20811, plugin_output="splunk", confirm=True)`

```bash
navi enrich tag --c "Software" --v "Splunk" --plugin 22869 --output "splunk"  # Linux
navi enrich tag --c "Software" --v "Splunk" --plugin 20811 --output "splunk"  # Windows
```

**PATH B — over 50K:**

First populate the software table: `navi_config(kind="software")`

Then:

`navi_enrich_tag(category="Software", value="Splunk", query="SELECT asset_uuid FROM software WHERE software_string LIKE '%splunk%';", confirm=True)`

```bash
navi enrich tag --c "Software" --v "Splunk" \
  --query "SELECT asset_uuid FROM software WHERE software_string LIKE '%splunk%';"
```

Requires credentialed scans. No data? Check for cred failures via
`navi_explore_data(subcommand="creds")`.

---

## ACR adjustment — see navi-acr

Adjusting Asset Criticality Ratings (`navi_enrich_acr`) has its own skill
file. ACR workflows are conceptually distinct from tagging — tagging
establishes which assets are what; ACR assigns criticality to those
tagged groups.

The typical sequence is:

1. **Tag by business tier** (this skill) — `navi_enrich_tag` with
   categories like `Environment:Production`, `Data Class:PII`, etc.
2. **Set ACR per tag** (navi-acr) — `navi_enrich_acr` with business-
   appropriate scores and required Change Reasons.
3. **Re-sync** — `navi_config_update(kind="assets")` so Tenable One
   recalculates AES across dashboards.

See navi-acr for the full `navi_enrich_acr` tool signature, Change Reason
mapping, mod set/inc/dec semantics, suggested tier mapping (10/9/8/6/3/2),
and temporary ACR adjustments for incident workflows.

**Trigger phrases that should route to navi-acr instead of here**: "adjust
ACR", "risk scores are wrong", "Tenable One isn't showing the right
things", "set production assets as most critical", "calibrate criticality",
"how do I improve my AES scores."

---

## `navi_enrich_add` — add assets from external sources

Each call is write-gated. Pass either `ip` (single) or `list_csv` (bulk),
not both.

Single IP:

`navi_enrich_add(ip="192.168.1.100", confirm=True)`

```bash
navi enrich add --ip 192.168.1.100
```

IP with optional identity fields — `hostname`, `fqdn`, `mac`, `netbios`
(`mac` matters for OT/IoT, where MAC OUI is the reliable fingerprint):

`navi_enrich_add(ip="192.168.1.100", hostname="web-prod-01", fqdn="web-prod-01.corp.com", mac="00:11:22:33:44:55", confirm=True)`

```bash
navi enrich add --ip 192.168.1.100 --hostname "web-prod-01" --fqdn "web-prod-01.corp.com" --mac "00:11:22:33:44:55"
```

CSV import — the tool param is `list_csv`, but the CLI flag is **`--file`** (not
`--list`). CSV column order is IP, MAC, FQDN, Hostname.

`navi_enrich_add(list_csv="assets.csv", source="CMDB", confirm=True)`

```bash
navi enrich add --file assets.csv --source "CMDB"
```

AWS inventory:

`navi_enrich_add(list_csv="aws_inventory.csv", source="AWS", confirm=True)`

```bash
navi enrich add --file aws_inventory.csv --source "AWS"
```

Use for: CMDB imports, OT/IoT devices that can't be actively scanned,
newly provisioned hosts between scan cycles.

---

## Quick reference table

All `navi_enrich_tag` args:

| Arg | Does | Example |
|--------|------|---------|
| `plugin` | Plugin fired (int) | `plugin=104410` |
| `plugin_output` | Text in output (requires `plugin`) | `plugin_output="splunk"` |
| `plugin_regexp` | Regex in output (requires `plugin`) | `plugin_regexp="jenkins 2\.\d+"` |
| `plugin_name` | Text in plugin name | `plugin_name="Apache"` |
| `cve` | CVE ID | `cve="CVE-2021-44228"` |
| `cpe` | CPE string | `cpe="cpe:/a:apache"` |
| `xrefs` | Cross-ref type | `xrefs="CISA"` |
| `xid` | Cross-ref ID (requires `xrefs`) | `xid="2024-001"` |
| `port` | Vuln on port (int) | `port=3389` |
| `route_id` | Route ID | `route_id="<id>"` |
| `file` | CSV of IPs | `file="assets.csv"` |
| `manual` | Asset UUID | `manual="<uuid>"` |
| `group` | Agent group | `group="Prod"` |
| `missed` | Missed auth N days (int) | `missed=7` |
| `byadgroup` | AD group CSV | `byadgroup="ad.csv"` |
| `scanid` | Assets in scan | `scanid="<id>"` |
| `histid` | Scan history ID (requires `scanid`) | `histid="<id>"` |
| `scantime` | Scan > N min (int) | `scantime=30` |
| `query` | Custom SQL | `query="SELECT ..."` |
| `by_tag` | Has this tag | `by_tag="Env:Prod"` |
| `by_val` | Match tag value | `by_val="Prod"` |
| `by_cat` | Match tag category | `by_cat="CVE"` |
| `parent_category` | Parent category (hierarchical) | `parent_category="Region"` |
| `parent_value` | Parent value (hierarchical) | `parent_value="Americas"` |
| `require_both` | AND logic with parent | `require_both=True` |
| `remove` | Ephemeral refresh | `remove=True` |
| `tone` | TONE tag instead of TVM | `tone=True` |
| `description` | Tag description | `description="auto-tagged by navi"` |
