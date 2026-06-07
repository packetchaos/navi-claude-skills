# navi-explore reference — useful `navi_explore_query` SQL patterns

Raw-SQL recipes for `navi_explore_query(sql="…")` (direct sqlite — reads need no
confirmation). Check a table's columns with `navi://schema/{table}` before
composing. Table-by-table schema + populate commands: see navi-core.

```sql
-- Top exploitable assets
SELECT asset_ip, plugin_name FROM vulns
WHERE state='active'
ORDER BY severity DESC LIMIT 20;

-- Assets with criticals, sorted
SELECT asset_ip, count(*) AS crits FROM vulns
WHERE severity='critical'
GROUP BY asset_ip ORDER BY crits DESC;

-- "Where am I using <technology>?" — by plugin family
-- (plugin families group detections, e.g. 'Artificial Intelligence' ≈ 36 plugins)
SELECT DISTINCT asset_uuid FROM vulns
WHERE plugin_family='Artificial Intelligence';

-- Certs expiring soonest
SELECT common_name, not_valid_after FROM certs
ORDER BY not_valid_after LIMIT 20;

-- DISTINCT vuln paths — true remediator workload
SELECT DISTINCT path, asset_uuid FROM vuln_paths
WHERE path LIKE '%jenkins%';

-- Workload reduction: raw vs distinct
SELECT count(*) AS raw FROM vuln_paths;
SELECT count(DISTINCT path) AS distinct_locations FROM vuln_paths;

-- Software inventory ranked by asset count
-- (requires navi_config(kind='software'); plugins 20811 Win / 22869 Linux / 83991 Mac)
SELECT software_string, count(*) AS assets FROM software
GROUP BY software_string ORDER BY assets DESC LIMIT 20;

-- Find a specific package (e.g. risky tooling like tcpdump / wireshark)
SELECT DISTINCT asset_uuid, software_string FROM software
WHERE software_string LIKE '%wireshark%';

-- EPSS — prioritize by exploit probability, not just severity
-- (requires epss table — downloaded separately from the EPSS CSV)
SELECT e.epss_value, v.asset_ip, v.plugin_name, v.severity
FROM vulns v
INNER JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
WHERE e.epss_value > 0.5
ORDER BY e.epss_value DESC LIMIT 20;

-- Highest EPSS score per asset
SELECT v.asset_ip, MAX(e.epss_value) AS highest_epss
FROM vulns v JOIN epss e ON v.cves LIKE '%' || e.cve || '%'
GROUP BY v.asset_ip ORDER BY highest_epss DESC LIMIT 10;

-- Fixed vulns — verify remediation was confirmed
SELECT asset_ip, plugin_name, severity, last_found FROM fixed
ORDER BY last_found DESC LIMIT 20;

-- Remediation velocity per asset
SELECT asset_ip, count(*) AS fixed_count FROM fixed
GROUP BY asset_ip ORDER BY fixed_count DESC;

-- Plugin lookup — what does this plugin do, what CVEs, what xrefs
SELECT plugin_id, plugin_name, severity, cves FROM plugins
WHERE plugin_id='10863';

-- All plugins with CISA xrefs (what xrefs="CISA" uses under the hood)
SELECT plugin_id, plugin_name FROM plugins
WHERE xrefs LIKE '%CISA%' LIMIT 20;
```
