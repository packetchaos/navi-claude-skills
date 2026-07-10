---
name: navi-remote-exec
description: >
  Remote command execution skill for the Tenable navi CLI, exposed through
  navi-mcp as the navi_action_push tool. Use for ANY request to run a shell
  command on — or copy a file to — a remote Linux host via navi: "push a command
  to this host", "run apt upgrade on the affected servers", "restart nginx on
  10.0.5.12", "patch the OpenSSL hosts", "scp this file to the target",
  "remediate the tagged group". Covers the highest gate in navi (double gate:
  NAVI_MCP_ALLOW_WRITES=1 + NAVI_REMOTE_CODE_EXECUTION=1), the confirm=True
  narration pattern, the single-host constraint (no --tag; loop per IP), the SSH
  prerequisite (navi config ssh), and the Route→Tag→Push→Verify remediation
  cycle. Companion: navi-enrich (tag the hosts), navi-scan (verify after).
---

# Navi Remote Exec — Push Commands to Hosts

`navi action push` is exposed through navi-mcp as **`navi_action_push`**. It
runs a shell command on, or copies a file to, a **single Linux host** over SSH,
using credentials configured out-of-band (`navi config ssh` — not exposed).

This is **remote code execution** — the highest-risk capability in navi. A bad
command can take a production host down. It is double-gated and requires
per-call confirmation with the literal command spelled out.

---

## Gating — what has to be true before a push

`navi_action_push` runs only when ALL of these hold:

1. **`NAVI_MCP_ALLOW_WRITES=1`** on the server — the master write gate.
2. **`NAVI_REMOTE_CODE_EXECUTION=1`** on the server — the RCE capability gate. A
   separate opt-in stacked on the write gate; enabling writes alone does NOT
   enable push.
3. **`confirm=True`** on the call — passed only AFTER you narrate the exact
   target IP and the exact command, and the user approves in chat.

If either server gate is closed, the tool returns a block message. Report it and
stop. To enable, the operator restarts navi-mcp with both env vars set (config
helper: `navi_mcp_config.py --allow-writes --allow-remote-code-execution`).

---

## The tool

`navi_action_push(target, command=None, file=None, confirm=False)`

| Arg | Required | Notes |
|---|---|---|
| `target` | yes | Host IP. push hits **one** host — there is **no `--tag`**. |
| `command` | one of | Shell command to run on the target. |
| `file` | one of | Local file to scp to the target. |

Provide **exactly one** of `command` or `file` — the tool rejects both-or-neither.

**Pushing a file** (scp) is how the navi project automates rollouts — e.g.
shipping a bash installer or an RPM to stand up and link a Tenable Nessus Agent
on a host:

> `navi_action_push(target="10.0.5.12", file="install_agent.sh", confirm=True)`

Copy the artifact with one call, then run it with a second (`command="sudo bash
install_agent.sh"`) — each its own confirmation.

---

## Confirmation pattern — spell out the literal command

Narrate the target and command verbatim → state the exact call → wait → call
with `confirm=True`. Never paraphrase the command; the user approves the
literal string that will run.

> I'll restart nginx on 10.0.5.12.
>
> Tool call: `navi_action_push(target="10.0.5.12",
> command="sudo systemctl restart nginx", confirm=True)`
>
> Confirm and I'll run it.

Each host gets its own confirmation. Do not batch multiple pushes behind a
single approval.

---

## Single host — running across a tagged group

push targets one IP. To act on a whole tag group, first read the tagged hosts'
IPs, then call the tool once per IP — narrating the full set before you start
and getting one approval for the batch, then confirming each call.

> These 6 hosts carry `Remediation:OpenSSL-Q2`: 10.0.5.12, 10.0.5.19, …
> I'll run `sudo apt-get update && sudo apt-get upgrade openssl libssl3 -y` on
> each. Confirm and I'll push them one at a time.
>
> `navi_action_push(target="10.0.5.12", command="sudo apt-get update && sudo apt-get upgrade openssl libssl3 -y", confirm=True)`
> `navi_action_push(target="10.0.5.19", command="…", confirm=True)`
> …

Pull the IPs with a read query, e.g.
`navi_explore_query(sql="SELECT a.ip FROM tags t JOIN assets a ON t.asset_uuid=a.uuid WHERE t.tag_key='Remediation' AND t.tag_value='OpenSSL-Q2';")`
(confirm column names against `navi://schema/tags` and `navi://schema/assets`).

---

## The Route → Tag → Push → Verify remediation cycle

The canonical remediation pattern, now fully MCP-driven (push is no longer a CLI
hand-off):

1. **Tag (MCP)** — `navi_enrich_tag(...)` marks the affected hosts (write-gated;
   narrate + confirm). Mind the 30-min tag propagation window.
2. **Push (MCP)** — pull the tagged IPs, then `navi_action_push(...)` per host
   to apply the fix (double-gated; narrate + confirm each).
3. **Verify (MCP)** — after a re-scan and vuln re-sync, confirm the finding is
   gone (`navi_explore_query(...)`), then retire the campaign tag with
   `navi_action_delete(kind="tag", ...)` if it's truly done.

---

## Failure modes

- **"remote code execution disabled" block** → `NAVI_REMOTE_CODE_EXECUTION`
  (and/or writes) not set. Report and stop.
- **SSH credential / connection error** → `navi config ssh` not set, target
  isn't Linux, or creds don't work. Surface navi's error; the operator fixes SSH
  at the terminal. Don't retry blindly.
- **Command failed on the host (non-zero exit)** → the tool raises with the
  host's stderr. Show it; don't assume the change applied. Re-verify before
  moving on.

---

## Remote-Exec → Natural Language

| User says | Tool call |
|---|---|
| "run <cmd> on <ip>" | `navi_action_push(target="<ip>", command="<cmd>", confirm=True)` |
| "copy this file to <ip>" | `navi_action_push(target="<ip>", file="<path>", confirm=True)` |
| "patch/remediate the tagged hosts" | read IPs → loop `navi_action_push(...)` per IP (Route→Tag→Push→Verify) |
| "restart <service> on <ip>" | `navi_action_push(target="<ip>", command="sudo systemctl restart <service>", confirm=True)` |

Cross-references: **navi-enrich** (tag the hosts), **navi-explore** (pull the
IPs), **navi-scan** (verification scan), **navi-mcp** (gate + confirm
conventions in full).
