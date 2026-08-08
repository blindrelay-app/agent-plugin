# Blindrelay Agent Plugin

> Send transactional email and manage sending domains, OTLP observability, and suppression through any Agent-Plugins-compatible client (Cursor, Claude Code, Codex, Copilot, VS Code, …) — without reinventing the setup.

This is a portable [Agent Plugins 1.0](https://agent-plugins.org) package. It bundles:

- **One MCP server** (`blindrelay`, Streamable HTTP → `https://api.blindrelay.app/mcp`) exposing the same tools as the Blindrelay CLI: `send`, `domains_*`, `usage`, `usage_history`, `delivery_stats`, `suppression_*`, `audit_list`, `otlp_*`, `delivery_settings_*`.
- **Four Agent Skills** that teach an agent *when* and *how* to use those tools, and the no-content-on-disk invariant that must be honored on every send.

Blindrelay is a **zero-content EU email relay**: message body, subject, and headers live in process RAM only until delivery; the recipient address is stored as a bcrypt hash. See [`references/no-content-on-disk.md`](references/no-content-on-disk.md).

## Layout

```
agent-plugin/
├── plugin.json                       # Agent Plugins 1.0 manifest (portable; Cursor/Code/Codex/Copilot)
├── mcp.json                          # blindrelay MCP server (Streamable HTTP, no secrets)
├── .cursor-plugin/                   # Cursor-optimized layer (optional, single-click auth)
│   ├── plugin.json                   #   Cursor manifest + variables (BLINDRELAY_API_KEY)
│   └── mcp.json                      #   MCP config using ${BLINDRELAY_API_KEY} / ${BLINDRELAY_MCP_URL}
├── skills/
│   ├── setup/SKILL.md                # onboarding + auth
│   ├── send-mail/SKILL.md            # send transactional email
│   ├── manage-domains/SKILL.md       # add/verify/rotate-dkim/remove domains
│   └── delivery-observability/SKILL.md  # OTLP, webhook, usage, stats, suppression, audit
├── references/
│   ├── tools.md                      # full tool reference + scopes + error codes
│   ├── no-content-on-disk.md         # the load-bearing invariant
│   └── transports.md                 # streamable-http vs stdio, staging, self-hosted
├── README.md
└── LICENSE
```

The root `plugin.json` + `mcp.json` are the **portable** Agent Plugins 1.0 entry (works in every compatible client). The `.cursor-plugin/` directory is a **Cursor-only** layer that lets Cursor users set the API key once via a dashboard variable instead of editing MCP headers by hand. Non-Cursor clients ignore `.cursor-plugin/` and use the portable manifest. Cursor users: if your client lists the plugin twice, prefer the `.cursor-plugin/` entry and ignore the root one.

## Install

Install through your client's plugin UI by pointing it at this directory (or the published registry entry, when listed). For example, in Cursor: Settings → Plugins → Add → select the `agent-plugin/` directory.

## Configure auth (one time)

Agent Plugins 1.0 does not let a plugin carry secrets, so the API key is client-managed. There are two paths:

### Cursor (single-click, recommended)

The `.cursor-plugin/` layer declares two dashboard variables. Set them once under **Plugins → Configure**:

| Variable | Required | Purpose |
|----------|----------|---------|
| `BLINDRELAY_API_KEY` | yes | Scoped API key from <https://blindrelay.app/keys> (one-time reveal). Stored only in the Cursor dashboard, never in the plugin repo. |
| `BLINDRELAY_MCP_URL` | no | MCP endpoint. Defaults to `https://api.blindrelay.app/mcp`. Use `https://api.stag.blindrelay.app/mcp` for staging, or `http://127.0.0.1:18082/mcp` for a local backend. |

Cursor substitutes `${BLINDRELAY_API_KEY}` into the `Authorization: Bearer …` header and `${BLINDRELAY_MCP_URL}` into the server URL automatically. No manual header editing.

### Other clients (portable)

1. Create a scoped API key at <https://blindrelay.app/keys> (reveal once, copy `br_…`). Scopes:
   - `send` → Free plan, can call `send`.
   - `read` + `write` → Starter+, unlocks domains / suppression / OTLP / delivery settings / audit / usage.
2. In your client's MCP settings for the `blindrelay` server, add the header:
   ```
   Authorization: Bearer br_…
   ```
3. Verify by calling `usage` (needs `read`) or `domains_list` (needs `read`). A 401 means the key/header is missing; a 403 `scope_missing` means the key lacks the scope.

Prefer a local binary + env? See [`references/transports.md`](references/transports.md) for the stdio variant using `blindrelay-mcp`.

## Use

Ask your agent in plain language:

- "Send a password reset email to the user from noreply@mydomain.com"
- "Add mail.example.com as a sending domain and show me the DNS records"
- "Verify the DKIM for my domain"
- "Point delivery events to my Grafana OTLP endpoint"
- "How many emails have I sent this month?"
- "Show me the suppression list"
- "What changed in my tenant in the last week?" (audit log)

The skills activate automatically based on intent. The agent will never log a recipient address or email body after a send.

## Environments

| Environment | MCP URL |
|-------------|---------|
| Production | `https://api.blindrelay.app/mcp` |
| Staging | `https://api.stag.blindrelay.app/mcp` |
| Local backend | `http://127.0.0.1:$PORT/mcp` |

To target staging or local, edit the `url` in `mcp.json` (or your client's override).

## Safety properties

- **No content on disk.** Body/subject/headers are RAM-only; recipient is a bcrypt hash. Enforced by `backend/tests/no_content_on_disk.rs` on every release. The skills reinforce this on the agent side.
- **No secrets in the plugin.** The API key never ships in `mcp.json` headers or stdio `env`; it lives in your client's secret store (Cursor dashboard variable, or your client's MCP auth settings).
- **Customer-owned observability.** Delivery events go to *your* OTLP endpoint and/or delivery webhook — your Grafana / Datadog / Honeycomb is the system of record.

## Validation

The package is checked on every MR by the `blindrelay_agent_plugin_lint_job` CI job (`scripts/validate-agent-plugin.py`): manifest closed-schema, MCP transport validity, no embedded secrets, Cursor `${VAR}` placeholders all declared in `variables`, skill frontmatter + name/dir match, and internal markdown links. Run it locally:

```bash
just agent-plugin-lint        # or: python3 scripts/validate-agent-plugin.py
```

## License

MIT — applies only to the contents of this `agent-plugin/` package (manifests,
skills, reference docs). The Blindrelay product itself is licensed separately
and remains proprietary; see the repository root `README.md`. See [`LICENSE`](LICENSE).
