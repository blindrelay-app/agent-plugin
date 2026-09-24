# Dual origin (OAuth vs PAT)

Some MCP hosts cannot mix OAuth discovery and a static Bearer PAT on the **same URL**. Blindrelay therefore serves two hosts:

| Server entry | URL | Auth |
|--------------|-----|------|
| `blindrelay-with-oauth` | `https://api.blindrelay.app/mcp` | MCP OAuth (RFC 9728). No `Authorization` header in the plugin. |
| `blindrelay-with-api-key` | `https://api-pat.blindrelay.app/mcp` | Same scoped `br_…` API keys as CLI / Keys UI. Optional `Authorization: Bearer`. |

Staging: `https://api.stag.blindrelay.app/mcp` and `https://api-pat.stag.blindrelay.app/mcp`.

**Enable exactly one server.** If both are on and `BLINDRELAY_API_KEY` is empty or still `${BLINDRELAY_API_KEY}`, the PAT origin returns a closed `401` (no `/.well-known`). The OAuth origin ignores unexpanded `Bearer ${…}` and challenges with `WWW-Authenticate`.

Do not send a PAT to `api.*` after discovery is live — it still challenges.

Laptop loopback `/mcp` is PAT-only unless `MCP_PUBLIC_URL` is that same loopback.

See [transports.md](transports.md).
