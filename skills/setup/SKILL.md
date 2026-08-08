---
name: setup
description: Onboard a tenant to the Blindrelay Agent Plugin — create a scoped API key, configure Bearer auth for the blindrelay MCP server in the host client, and verify the connection. Use when the user says "set up Blindrelay", "configure Blindrelay", "connect Blindrelay", "blindrelay isn't working", or the MCP server returns 401/403.
---

# Set up the Blindrelay Agent Plugin

Use this skill the first time the user wants to use Blindrelay through the agent, or whenever the blindrelay MCP server is failing to authenticate (401/403).

## What this plugin provides

The plugin ships one MCP server (`blindrelay`, Streamable HTTP → `https://api.blindrelay.app/mcp`) exposing the same tools as the Blindrelay CLI: `send`, `domains_*`, `usage`, `usage_history`, `delivery_stats`, `suppression_*`, `audit_list`, `otlp_*`, `delivery_settings_*`. Tools are authorized by API-key scopes (`send`, `read`, `write`).

## The one-time setup (auth)

Agent Plugins 1.0 does **not** let a plugin carry secrets. The API key is client-managed. You only do this once.

1. **Create an API key** in the Blindrelay UI: <https://blindrelay.app/keys>. Reveal the secret once and copy it (`br_live_…` or `br_test_…`). Decide scopes:
   - `send` only → Free plan works, can call `send`.
   - `read` + `write` → needs Starter+; unlocks domains, suppression, OTLP, delivery settings, audit, usage.

2. **Tell the host client to send the key as a Bearer token** on every call to the `blindrelay` MCP server. The plugin's portable `mcp.json` deliberately omits the `Authorization` header because secrets must not ship inside a plugin package. Configure it in the client, not in the plugin:
   - **Cursor (single-click):** this plugin ships a `.cursor-plugin/` layer that declares a `BLINDRELAY_API_KEY` variable. Open **Plugins → Configure** and paste the key once; Cursor injects `Authorization: Bearer ${BLINDRELAY_API_KEY}` automatically. No header editing.
   - **Other HTTP hosts (Claude Desktop, VS Code, …):** add the header `Authorization: Bearer br_…` to the `blindrelay` server entry in the client's MCP/auth settings.
   - **Prefer a local binary?** Install `blindrelay-mcp` (see [transports.md](../../references/transports.md)) and replace the server entry with a stdio server that reads `BLINDRELAY_API_KEY` from the environment.

3. **Never paste the API key into a file the agent writes to disk inside this repo** (it would be committed). Keep it in the client's secret store or a gitignored env file.

## Verify the connection

Ask the user to confirm the key is configured, then call a cheap read tool to prove auth works:

- Free / `send`-only key: ask the user to confirm; `send` is the only callable tool. Do **not** send a real email just to test — use `usage` only if the key has `read`.
- `read` scope: call `usage` (current billing period). A successful JSON response means auth + scope are good.
- No scope yet: call `domains_list` only if `read` is granted.

If you get `401` / "unauthorized": the key is missing, wrong, or expired — repeat step 2. If you get `403` / "scope_missing": the key lacks the scope the tool needs — recreate the key with the right scopes in the UI.

## After setup

Hand off to the right skill:

- Send an email → [send-mail](../send-mail/SKILL.md)
- Add / verify / rotate a sending domain → [manage-domains](../manage-domains/SKILL.md)
- OTLP / delivery webhook / stats / suppression / audit → [delivery-observability](../delivery-observability/SKILL.md)

## Reference

- [tools.md](../../references/tools.md) — every tool, its args, and required scope.
- [transports.md](../../references/transports.md) — streamable-http vs stdio, self-hosted.
- [no-content-on-disk.md](../../references/no-content-on-disk.md) — the load-bearing invariant you must not violate.
