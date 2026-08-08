# MCP transports

The Blindrelay MCP server is available over two transports with the same tool surface. The plugin ships the Streamable HTTP variant because it needs no local binary install. You can switch to stdio if you prefer a local process.

## Streamable HTTP (default, what this plugin ships)

```json
{
  "$schema": "https://agent-plugins.org/schemas/1.0.0/mcp.schema.json",
  "mcpServers": {
    "blindrelay": {
      "type": "streamable-http",
      "url": "https://api.blindrelay.app/mcp"
    }
  }
}
```

- No local binary. Works on any host that speaks MCP over HTTP.
- Auth: `Authorization: Bearer <api_key>` header, configured in your client's MCP/auth settings (the plugin deliberately omits it — secrets must not ship inside a plugin package).
- Endpoints by environment:

| Environment | URL |
|-------------|-----|
| Production | `https://api.blindrelay.app/mcp` |
| Staging | `https://api.stag.blindrelay.app/mcp` |
| Local backend | `http://127.0.0.1:$PORT/mcp` (same `PORT` as the backend) |

## stdio (optional, local binary)

For hosts that prefer a local process (Cursor, Claude Desktop). Binary: `blindrelay-mcp`.

Install:

```bash
# Linux (checksum-verified)
curl -fsSL https://blindrelay.app/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"

# macOS
brew tap blindrelay/tap https://blindrelay.app/tap.git
brew install blindrelay/tap/blindrelay-mcp

# Contributors (from this repo)
cargo install --path mcp
```

Auth: `BLINDRELAY_API_KEY` env var, or `blindrelay auth login` once (writes `~/.config/blindrelay/credentials.json`).

To use stdio instead of the shipped HTTP server, replace the `blindrelay` entry in your client's MCP config with:

```json
{
  "mcpServers": {
    "blindrelay": {
      "type": "stdio",
      "command": "blindrelay-mcp",
      "env": {
        "BLINDRELAY_API_KEY": "br_…"
      }
    }
  }
}
```

Prefer an **absolute path** for `command` when the host does not inherit your shell `PATH`:

| Install path | Absolute `command` |
|--------------|-------------------|
| Homebrew (Apple Silicon) | `/opt/homebrew/bin/blindrelay-mcp` |
| Homebrew (Intel) | `/usr/local/bin/blindrelay-mcp` |
| Linux curl installer | `$HOME/.local/bin/blindrelay-mcp` |

Logs go to **stderr** only (stdout is the MCP JSON-RPC stream).

## Self-hosted / on-prem

If you operate your own Blindrelay backend, point the `url` (Streamable HTTP) or `BLINDRELAY_BASE_URL` (stdio) at your instance's machine API host. The MCP gateway is served at `/mcp` on the same host as `/v1/*`. Caddy/Traefik must route `/mcp` to the backend on the `api.*` host.

## Choosing

- **Remote agents, no local install, cross-client portability** → Streamable HTTP (the shipped default).
- **Desktop host, you already `brew install` the toolbelt, you want env-based auth** → stdio.
- **Air-gapped / on-prem** → whichever transport reaches your backend; stdio with `BLINDRELAY_BASE_URL` is often simplest.
