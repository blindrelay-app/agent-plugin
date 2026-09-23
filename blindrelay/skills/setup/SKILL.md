---
name: setup
description: Onboard a tenant to the Blindrelay Agent Plugin — OAuth Connect or optional PAT on api-pat, then verify tools. Use when the user says "set up Blindrelay", "configure Blindrelay", "connect Blindrelay", "blindrelay isn't working", or the MCP server returns 401/403.
---

# Set up the Blindrelay Agent Plugin

Use this skill the first time the user wants to use Blindrelay through the agent, or whenever MCP auth fails (401/403).

## What this plugin provides

Two Streamable HTTP servers (enable **one**):

- `blindrelay` → `https://api.blindrelay.app/mcp` — MCP OAuth. No Authorization header. Signup or existing session.
- `blindrelay-pat` → `https://api-pat.blindrelay.app/mcp` — same `br_…` keys as the Keys UI.

Tools match the CLI: `send`, `domains_*`, `dedicated_ip_*`, `inbound_forward_*`, `usage`, `usage_history`, `delivery_stats`, `suppression_*`, `audit_list`, `otlp_*`, `delivery_settings_*`. Scopes: `send`, `read`, `write`.

Never enable both servers with an empty PAT. Unexpanded `Bearer ${BLINDRELAY_API_KEY}` is ignored. Do not put a PAT on `api.*`. See [dual-origin.md](../../references/dual-origin.md).

## Install

If Blindrelay is not installed yet, add <https://github.com/blindrelay-app/agent-plugin> from the plugin or connector UI (Claude, ChatGPT, Cursor, and other Agent Plugins 1.0 hosts). Cursor example: `/add-plugin` that URL, then Install in Customize.

## The one-time setup (auth)

1. **Prefer OAuth.** Enable only `blindrelay`. Complete Connect in the browser (same-tab email OTP if the user is new). No MCP tokens until the email is verified.
2. **PAT (optional).** Create a key at <https://blindrelay.app/keys>. Enable only `blindrelay-pat`. Set `Authorization: Bearer br_…` on the PAT URL. Cursor example: Plugins → Configure, paste `BLINDRELAY_API_KEY` (default empty).
3. **Never paste the API key into a file the agent writes to disk inside this repo.**

## Verify

- OAuth / `read`: call `usage`.
- `send`-only: do not send a real email just to test.
- `401` on PAT with empty key: switch to OAuth or set the key.
- `401` + `WWW-Authenticate` on `api.*` with a `br_…` key: you are on the OAuth host — use `api-pat` or Connect.

## After setup

- Send → [send-mail](../send-mail/SKILL.md)
- Domains → [manage-domains](../manage-domains/SKILL.md)
- Stats / suppression → [delivery-observability](../delivery-observability/SKILL.md)

## Reference

- [tools.md](../../references/tools.md)
- [transports.md](../../references/transports.md)
- [dual-origin.md](../../references/dual-origin.md)
- [no-content-on-disk.md](../../references/no-content-on-disk.md)
