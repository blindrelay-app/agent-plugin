---
name: send-mail
description: Send a transactional email through Blindrelay using the send MCP tool. Use when the user says "send email", "email the user", "send a reset link", "mail this", "transactional email", or asks to notify someone from the app. Honors the no-content-on-disk invariant — never logs or persists body or recipient.
---

# Send transactional email

Use the `send` MCP tool (exposed by the `blindrelay` MCP server). Required scope: `send` (Free plan OK).

## Preconditions

- Auth is configured (see [setup](../setup/SKILL.md) if you get 401/403).
- The `from` address must be on a **verified** sending domain registered in Blindrelay. If `domains_list` shows the domain as not verified, or the domain is missing, switch to [manage-domains](../manage-domains/SKILL.md) first.

## Tool: `send`

Args (all strings):

| Field | Required | Notes |
|-------|----------|-------|
| `from` | yes | Must be on a verified sending domain, e.g. `noreply@yourdomain.com`. |
| `to` | yes | One recipient. Plaintext only in the tool call — never log or echo it after success. |
| `subject` | yes | Email subject. |
| `html` | yes | HTML body. |
| `text` | no | Plaintext fallback. Include it for better deliverability. |

Returns `{ "id": "<uuid>", "status": "queued", "warnings": [] }` (202 Accepted). The envelope id is safe to log; the recipient address and body are **not**.

## How to call it

Call the MCP `send` tool directly with the five fields above. Do not shell out to `curl` or the CLI — the MCP server is already wired up and keeps the key out of process args / shell history.

## Idempotency and retries

- The HTTP API accepts an `Idempotency-Key` header for safe replay, but the MCP `send` tool does not expose it. For fire-and-forget transactional sends, one call is enough — Blindrelay gives each send **one** SMTP attempt and does not retain the body.
- If the tool returns a retryable error (`quota_exceeded`, `draining`, `queue_at_capacity`), wait and call `send` again with the same arguments.
- Non-retryable errors (`invalid_payload`, `recipient_suppressed`, `domain_not_verified`, `scope_missing`, `tenant_suspended`, `unauthorized`) require a fix, not a retry.

## No-content-on-disk rules (load-bearing)

These are product invariants, not preferences. Violating them is a release blocker for Blindrelay and a trust breaker for the user.

- **Never** echo the `to` address, `subject`, `html`, or `text` back in your response after a successful send. Report only the envelope `id` and a short confirmation.
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

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [no-content-on-disk.md](../../references/no-content-on-disk.md) — the invariant in detail.
