# MCP transports

Install the **Agent Plugin**. Hosted `/mcp` is infrastructure, not the customer install story.

## Dual Streamable HTTP (what this plugin ships)

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "blindrelay": {
      "type": "streamable-http",
      "url": "https://api.blindrelay.app/mcp"
    },
    "blindrelay-pat": {
      "type": "streamable-http",
      "url": "https://api-pat.blindrelay.app/mcp",
      "headers": {
        "Authorization": "Bearer ${BLINDRELAY_API_KEY}"
      }
    }
  }
}
```

Enable **one** entry. Default plan is OAuth Connect (`blindrelay`) after web signup. PAT is the same `br_…` key as the Keys UI, only on `api-pat.*`. Details: [dual-origin.md](dual-origin.md).

Do not add SSE on `GET /mcp` for Cursor idle resume. If a host suspends Streamable HTTP, set `"http.fetchAdditionalSupport": false` rather than inventing GET SSE.

Protocol version advertised on initialize: `2026-07-28`.

## Self-hosted

Point `url` at your instance. `Host` selects PAT vs OAuth (`MCP_PUBLIC_URL`, `MCP_PAT_HOST`). Traefik must route `/mcp` and `/.well-known` on `api.*`, and `/mcp` only (no well-known) on `api-pat.*`.
