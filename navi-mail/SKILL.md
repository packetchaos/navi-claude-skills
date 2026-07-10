---
name: navi-mail
description: >
  Email delivery skill for the Tenable navi CLI, exposed through navi-mcp as the
  navi_action_mail tool. Use for ANY request to email a navi report, export, or
  file to someone: "email this report", "send the SLA breach CSV to the CISO",
  "mail the export to the security team", "send me a copy of this". Covers the
  double gate (NAVI_MCP_ALLOW_WRITES=1 + NAVI_EMAIL=1), the confirm=True
  narration pattern, the export→mail and encrypt→mail composition patterns, and
  the SMTP prerequisite (navi config smtp, set out-of-band). Companion:
  navi-export (produce the CSV), navi-action (encrypt an attachment first).
---

# Navi Mail — Emailing Reports & Files

`navi action mail` is exposed through navi-mcp as **`navi_action_mail`**. It
sends an email using the SMTP settings configured out-of-band (`navi config
smtp` — not exposed as a tool), optionally attaching a file.

This tool sends email on the user's behalf, so it is **double-gated** and
requires explicit per-call confirmation. Treat every send as a deliberate,
user-approved action.

---

## Gating — what has to be true before a send

`navi_action_mail` runs only when ALL of these hold:

1. **`NAVI_MCP_ALLOW_WRITES=1`** on the server — the master write gate.
2. **`NAVI_EMAIL=1`** on the server — the email capability gate. This is a
   separate opt-in stacked on top of the write gate; enabling writes alone does
   NOT enable email.
3. **`confirm=True`** on the call — passed only AFTER you narrate the recipient,
   subject, and attachment to the user and they approve in chat.

If either server gate is closed, the tool returns a block message. Report it and
stop — you cannot change server env from the tool surface. To enable, the
operator restarts navi-mcp with both env vars set (e.g. via the config helper:
`navi_mcp_config.py --allow-writes --allow-email`).

---

## The tool

`navi_action_mail(to, subject="navi report", file=None, message="", confirm=False)`

| Arg | Required | Notes |
|---|---|---|
| `to` | yes | Recipient email address. Always passed — navi prompts interactively when `--to` is empty, which deadlocks under MCP, so the tool never leaves it blank. |
| `subject` | no | navi appends `" - Emailed by navi"`. Defaults to `"navi report"`. |
| `file` | no | Path to attach (absolute or relative to the navi-mcp workdir). |
| `message` | no | Body text. |

---

## Confirmation pattern

Narrate → state the exact call → wait for approval → call with `confirm=True`.

> I'll email the SLA breach report to your security team.
>
> Tool call: `navi_action_mail(to="security-team@company.com",
> subject="Weekly SLA Breach Report", file="failures_export.csv", confirm=True)`
>
> Confirm and I'll send it.

Never batch a send behind another operation's confirmation — the mail step gets
its own explicit approval.

---

## Composition patterns

**Export → mail.** Produce the CSV with a navi-export tool, surface the path it
returns, then mail that path.

> Tool call: `navi_export(subcommand="bytag", category="Environment", value="Production")`
> *[returns: "Wrote bytag_export.csv — 1,247 rows"]*
>
> I'll email that file to your CISO:
> `navi_action_mail(to="ciso@company.com", subject="Production ACR/AES export",
> file="bytag_export.csv", confirm=True)` — confirm to send.

**Encrypt → mail** (for sensitive attachments). Encrypt locally first, then
attach the `.enc`.

> `navi_action_encrypt(file="pii_export.csv")`  *(not gated — local only)*
> then
> `navi_action_mail(to="auditor@company.com", subject="Q2 audit data",
> file="pii_export.csv.enc", confirm=True)`

Prefer encrypt→mail whenever the attachment contains PII, credentials, or raw
vulnerability detail leaving the organization.

**Tag → export → email owners.** A staple navi pattern: tag assets by owner or
plugin data, export that slice to CSV, then mail it to the responsible team.

> `navi_enrich_tag(...)` → `navi_export(subcommand="bytag", ...)` →
> `navi_action_mail(to="app-team@company.com", subject="Assets you own — action needed", file="bytag_export.csv", confirm=True)`

**Message-only alert.** `navi_action_mail` can carry just a `message` (no
attachment) — useful as a completion/alert ping after an automated run.

> `navi_action_mail(to="secops@company.com", subject="Nightly hygiene run complete", message="Tagging + export finished; CSV attached to the next mail.", confirm=True)`

---

## Failure modes

- **"email disabled" block** → `NAVI_EMAIL` (and/or writes) not set. Report and
  stop; point the operator at the config helper.
- **SMTP error / "Your Email information may be incorrect"** → SMTP isn't
  configured. The operator runs `navi config smtp` at the terminal (interactive,
  not exposed). Surface this and pause; don't retry blindly.
- **Attachment not found** → the `file` path is wrong. Check the navi-mcp
  workdir (`navi://workdir`) for where exports land, and confirm the export
  actually wrote a file before mailing.

---

## Mail → Natural Language

| User says | Tool call |
|---|---|
| "email this report to X" | `navi_action_mail(to="X", subject="...", file="...", confirm=True)` |
| "send the export to the team" | export first → `navi_action_mail(to="...", file="<csv>", confirm=True)` |
| "mail me the sensitive CSV" | `navi_action_encrypt(...)` → `navi_action_mail(..., file="<file>.enc", confirm=True)` |
| "just email a note, no file" | `navi_action_mail(to="...", subject="...", message="...", confirm=True)` |

Cross-references: **navi-export** (build the CSV), **navi-action**
(encrypt/decrypt, other action tools), **navi-mcp** (the gate + confirm
conventions in full).
