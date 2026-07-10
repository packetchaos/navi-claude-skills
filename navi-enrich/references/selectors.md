# navi-enrich reference — full `navi_enrich_tag` selector catalog

Every primary selector with tool + CLI forms. Pass exactly ONE primary
selector per call; `plugin_output`/`plugin_regexp` modify `plugin`; `xid`
requires `xrefs`; `histid` requires `scanid`. All tag calls are write-gated.
The lean selector map + use-case playbook are in SKILL.md.

### By vulnerability content

Plugin fired:

`navi_enrich_tag(category="Cat", value="Val", plugin=<ID>, confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --plugin <ID>
```

Plugin fired + text in output:

`navi_enrich_tag(category="Cat", value="Val", plugin=<ID>, plugin_output="text", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --plugin <ID> --output "text"
```

Plugin fired + regex in output:

`navi_enrich_tag(category="Cat", value="Val", plugin=<ID>, plugin_regexp="PATTERN", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --plugin <ID> -regexp "PATTERN"
```

Text in plugin name:

`navi_enrich_tag(category="Cat", value="Val", plugin_name="Apache", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --name "Apache"
```

By CVE ID:

`navi_enrich_tag(category="Cat", value="Val", cve="CVE-2021-44228", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --cve "CVE-2021-44228"
```

By CPE:

`navi_enrich_tag(category="Cat", value="Val", cpe="cpe:/a:apache:http_server", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --cpe "cpe:/a:apache:http_server"
```

CISA Known Exploited Vulnerabilities (KEV):

`navi_enrich_tag(category="CISA", value="KEV", xrefs="CISA", confirm=True)`

```bash
navi enrich tag --c "CISA" --v "KEV" --xrefs "CISA"
```

Cross-reference + specific ID:

`navi_enrich_tag(category="Intel", value="IAVA", xrefs="IAVA", xid="2024-001", confirm=True)`

```bash
navi enrich tag --c "Intel" --v "IAVA" --xrefs "IAVA" --xid "2024-001"
```

Vuln on a specific port:

`navi_enrich_tag(category="Cat", value="Val", port=3389, confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --port 3389
```

By route ID:

`navi_enrich_tag(category="Route", value="Jenkins", route_id="<ID>", confirm=True)`

```bash
navi enrich tag --c "Route" --v "Jenkins" --route_id <ID>
```

### By asset identity

CSV of IPs:

`navi_enrich_tag(category="Cat", value="Val", file="assets.csv", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --file assets.csv
```

Specific asset UUID:

`navi_enrich_tag(category="Cat", value="Val", manual="<UUID>", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --manual <UUID>
```

Agent group (requires `navi_config_update(kind="agents")`):

`navi_enrich_tag(category="Cat", value="Val", group="Prod", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --group "Prod"
```

AD group CSV:

`navi_enrich_tag(category="AD", value="Finance", byadgroup="ad.csv", confirm=True)`

```bash
navi enrich tag --c "AD" --v "Finance" --byadgroup ad.csv
```

Agents missing auth for N days:

`navi_enrich_tag(category="Health", value="Missed 7d", missed=7, confirm=True)`

```bash
navi enrich tag --c "Health" --v "Missed 7d" --missed 7
```

> **Agent tagging prereq**: `group`, `missed`, `byadgroup` all require
> agent data. Run `navi_config_update(kind="agents")` first — it is NOT
> included in `navi config update full`. Zero results from `group` is
> almost always a stale agents table.

### By scan data

All assets in a scan:

`navi_enrich_tag(category="Cat", value="Val", scanid="<SCAN_ID>", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --scanid <SCAN_ID>
```

Specific scan history run:

`navi_enrich_tag(category="Cat", value="Val", scanid="<ID>", histid="<HIST_ID>", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --scanid <ID> --histid <HIST_ID>
```

Assets with scan time > N minutes:

`navi_enrich_tag(category="Health", value="Slow", scantime=30, confirm=True)`

```bash
navi enrich tag --c "Health" --v "Slow" --scantime 30
```

### By custom SQL query

Any SQL that returns `asset_uuid` values. Most powerful option.

`navi_enrich_tag(category="Cat", value="Val", query="SELECT asset_uuid FROM <table> WHERE ...;", confirm=True)`

```bash
navi enrich tag --c "Cat" --v "Val" --query "SELECT asset_uuid FROM <table> WHERE ...;"
```

Examples:

`navi_enrich_tag(category="Route", value="Jenkins", query="SELECT DISTINCT asset_uuid FROM vuln_paths WHERE path LIKE '%jenkins%';", confirm=True)`

`navi_enrich_tag(category="CertExpiry", value="ExpiringSoon", query="SELECT asset_uuid FROM certs WHERE not_valid_after LIKE 'Apr%2026%';", remove=True, confirm=True)`

`navi_enrich_tag(category="WAS Risk", value="Critical", query="SELECT uuid FROM apps WHERE critical_count > 0;", remove=True, confirm=True)`

### Tag-based derivation

Derive from an existing tag — assets that already have `Environment:Production`
become `Priority:High`:

`navi_enrich_tag(category="Priority", value="High", by_tag="Environment:Production", confirm=True)`

```bash
navi enrich tag --c "Priority" --v "High" --by_tag "Environment:Production"
```

Match by tag value (fuzzy):

`navi_enrich_tag(category="Tier", value="Prod-Like", by_val="Prod", confirm=True)`

```bash
navi enrich tag --c "Tier" --v "Prod-Like" --by_val "Prod"
```

Match by tag category:

`navi_enrich_tag(category="Scope", value="Security", by_cat="CVE", confirm=True)`

```bash
navi enrich tag --c "Scope" --v "Security" --by_cat "CVE"
```

### Hierarchical tags

Parent-child relationship. `parent_category`/`parent_value` specify the
parent:

`navi_enrich_tag(category="Country", value="US", parent_category="Region", parent_value="Americas", confirm=True)`

```bash
navi enrich tag --c "Country" --v "US" --cc "Region" --cv "Americas"
```

AND logic with `require_both=True`:

`navi_enrich_tag(category="Child", value="Val", parent_category="Parent", parent_value="Val", require_both=True, confirm=True)`

```bash
navi enrich tag --c "Child" --v "Val" --cc "Parent" --cv "Val" -all
```

### Special modes

- `tone=True` — create a TONE tag (Tenable One Exposure) instead of a TVM tag
- `remove=True` — CLEAR the tag from all assets it currently carries (ignores
  any selector). For an accurate ephemeral refresh, run it alone, wait ~30 min,
  then re-apply the selector in a separate call with no remove. Combining
  remove with a selector in one call is allowed but only adds/updates against
  current membership (the tool flags it) — see the ephemeral pattern in SKILL.md.
- `description="text"` — add a description to the tag

---

