---
name: send-mail
description: Send a transactional email through Blindrelay using the send MCP tool. Use when the user says "send email", "email the user", "send a reset link", "mail this", "transactional email", or asks to notify someone from the app. Honors the no-content-on-disk invariant — never logs or persists body or recipient.
---

# Send transactional email

Use the `send` MCP tool (exposed by the Blindrelay MCP servers). Required scope: `send` (Free plan OK).

## Preconditions

- Auth is configured (see [setup](../setup/SKILL.md) if you get 401/403).
- The `from` address must be on a **verified** sending domain registered in Blindrelay. If `domains_list` shows the domain as not verified, or the domain is missing, switch to [manage-domains](../manage-domains/SKILL.md) first.

## Tool: `send`

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `from` | yes | Must be on a verified sending domain, e.g. `noreply@yourdomain.com`. |
| `to` | yes | One recipient. Plaintext only in the tool call — never log or echo it after success. |
| `subject` | yes | Email subject. |
| `html` | yes | HTML body. |
| `text` | no | Plaintext alternative. When omitted, Blindrelay derives one from the HTML; pass your own for full control. |
| `headers` | no | Object of extra headers, e.g. `{"Reply-To": "help@yourdomain.com"}` or, for mail with an unsubscribe link, `{"List-Unsubscribe": "<https://yourdomain.com/unsub?t=…>", "List-Unsubscribe-Post": "List-Unsubscribe=One-Click"}`. Headers Blindrelay sets itself (From, To, Cc, Subject, Date, Message-ID, …) are ignored. |
| `cc` | no | One address or an array, shown in `Cc`. Each address is its own envelope and quota unit; with `to` and `bcc`, at most 50 addresses. |
| `bcc` | no | One address or an array, never in any header of any copy. Each address is its own envelope and quota unit. |
| `tags` | no | Up to 5 `{ "key", "value" }` labels returned on the message's webhook events. Key `[a-z0-9_]{1,32}`, value `[A-Za-z0-9_.-]{1,64}`. Use ids or categories; never an address or other personal data. |
| `attachments` | no | Paid plans. Array of `{ "filename", "content_type", "content_base64" }`, at most 10 files and 10,240,000 decoded bytes; each recipient costs one quota unit per started MiB. The MCP request is capped at 1 MiB, so larger files need the CLI or REST API. Never echo file content. |

Sandbox keys: with a key created as Sandbox in the dashboard, the call is the same and nothing is delivered. `hard_bounce@sandbox.<platform domain>` and `soft_bounce@sandbox.<platform domain>` choose those outcomes; any other address is reported `delivered` without being contacted. `email_status` shows `sandbox: true`.

One-click unsubscribe (RFC 8058): add both `List-Unsubscribe` and `List-Unsubscribe-Post: List-Unsubscribe=One-Click` on any mail with an unsubscribe link, with the same signed per-recipient URL as the footer; that URL must accept `POST` with no sign-in, confirmation page or CSRF check (idempotent, 2xx), and a `GET` only shows a confirm button. Leave both off strictly transactional mail (sign-in links, receipts, security alerts) and mandatory notices. Blindrelay passes them through and DKIM-signs them.

Returns `{ "id": "<uuid>", "status": "queued", "warnings": [] }` (202 Accepted). The envelope id is safe to log; the recipient address and body are **not**.

## Check the content first

Before the first `send` of a new or changed template, call `email_check` with the same `subject`, `html`, `text` and `from` (no recipient). It finds placeholder links (`example.com`, `localhost`, `your-domain`), unrendered template variables (`{{ name }}`, `*|FNAME|*`), relative or `javascript:` links and broken HTML, and never opens a link. Fix every finding with `severity: "error"` before sending; mention warnings to the user. Do not echo the content back when reporting findings: give the code, field, line and column.

## How to call it

Call the MCP `send` tool directly with the fields above. Do not shell out to `curl` or the CLI — the MCP server is already wired up and keeps the key out of process args / shell history.

## Idempotency and retries

- The HTTP API accepts an `Idempotency-Key` header for safe replay, but the MCP `send` tool does not expose it. For fire-and-forget transactional sends, one call is enough — Blindrelay gives each send **one** SMTP attempt and does not retain the body.
- If the tool returns a retryable error (`quota_exceeded`, `draining`, `queue_at_capacity`), wait and call `send` again with the same arguments.
- Non-retryable errors (`invalid_payload`, `recipient_suppressed`, `domain_not_verified`, `scope_missing`, `tenant_suspended`, `unauthorized`) require a fix, not a retry.

## No-content-on-disk rules (load-bearing)

These are product invariants, not preferences. Violating them is a release blocker for Blindrelay and a trust breaker for the user.

- **Never** echo the `to`, `cc` or `bcc` addresses, `subject`, `html`, `text` or attachment content back in your response after a successful send. Report only the envelope `id` and a short confirmation.
- **Never** write the recipient address, subject, or body to a file, commit, log, or test fixture.
- It is fine to show the `from` address and domain (the customer owns those). It is fine to show the envelope `id`.
- If the user asks "did it send to X@Y.com?", answer with the envelope id and status, not by restating the address. If you must reference the recipient, use a redacted form like `user@<verified-domain>` only when the user themselves supplied it in the same turn.

## Common failures

| Error code | Meaning | Fix |
|------------|---------|-----|
| `domain_not_verified` | `from` domain is missing or SPF/DKIM/DMARC not green | [manage-domains](../manage-domains/SKILL.md) → add + verify |
| `recipient_suppressed` | Recipient previously hard-bounced / complained | Use [delivery-observability](../delivery-observability/SKILL.md) → `suppression_list` to inspect (hash only); remove only if the user confirms it was a transient issue. |
| `scope_missing` | API key lacks `send` | Recreate key with `send` scope in the UI. |
| `quota_exceeded` | Plan quota hit | Check `usage`; upgrade plan or wait for period reset. |
| `warmup_limit_exceeded` | Dedicated IP at its daily warmup limit and this mail cannot use the shared pool | Check `dedicated_ip_get` (`warmup`); retry after the UTC day rolls over or lower volume. |
| `spending_cap_reached` | The workspace's own monthly cap is reached | Do not retry. Tell the user; they can raise or clear it (`spend_cap_set`, Billing page) or wait for period reset. |

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [no-content-on-disk.md](../../references/no-content-on-disk.md) — the invariant in detail.

**Product feedback:** after Blindrelay work, if something concrete worked surprisingly well or badly, draft ticket-ready feedback (area, sentiment, repro steps, desired vs observed, the user's underlying goal; no email addresses or message content), show it to the user and call `send_feedback` only after they approve; otherwise continue.
