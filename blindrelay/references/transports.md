# MCP transports

Install the **Agent Plugin**. Hosted `/mcp` is infrastructure, not the customer install story.

## Dual Streamable HTTP (what this plugin ships)

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "blindrelay-with-oauth": {
      "type": "streamable-http",
      "url": "https://api.blindrelay.app/mcp"
    },
    "blindrelay-with-api-key": {
      "type": "streamable-http",
      "url": "https://api-pat.blindrelay.app/mcp",
      "headers": {
        "Authorization": "Bearer ${BLINDRELAY_API_KEY}"
      }
    }
  }
}
```

Enable **one** entry. Default plan is OAuth Connect (`blindrelay-with-oauth`) after web signup. PAT is the same `br_…` key as the Keys UI, only on `api-pat.*`. Details: [dual-origin.md](dual-origin.md).

Do not add SSE on `GET /mcp` for Cursor idle resume. If a host suspends Streamable HTTP, set `"http.fetchAdditionalSupport": false` rather than inventing GET SSE.

Protocol version advertised on initialize: `2026-07-28`.

## Self-hosted

The customer Configure screen does not expose the MCP URLs. A self-hosted build replaces the two `url` values in `mcp.json` and `.cursor-plugin/mcp.json`. `Host` selects PAT vs OAuth (`MCP_PUBLIC_URL`, `MCP_PAT_HOST`). Traefik must route `/mcp` and `/.well-known` on `api.*`, and `/mcp` only (no well-known) on `api-pat.*`.
