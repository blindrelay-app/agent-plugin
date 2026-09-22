# Blindrelay Agent Plugin

> Send transactional email and manage sending domains, OTLP observability, and suppression through any Agent Plugins 1.0 client (Claude, ChatGPT, Cursor, and others) — without reinventing the setup.

This is a portable [Agent Plugins 1.0](https://agent-plugins.org) package. It bundles:

- **Two MCP servers** (`blindrelay` OAuth on `api.*`, `blindrelay-pat` on `api-pat.*`) plus five Agent Skills. Enable **one** server.
- **Five Agent Skills** that teach an agent *when* and *how* to use those tools, and the no-content-on-disk invariant that must be honored on every send.

Blindrelay is a **zero-content EU email relay**: message body, subject, and headers live in process RAM only until delivery; the recipient address is stored as a bcrypt hash. See [`references/no-content-on-disk.md`](references/no-content-on-disk.md).

## Layout

```
agent-plugin/
├── plugin.json                       # Agent Plugins 1.0 manifest (portable; Cursor/Code/Codex/Copilot)
├── mcp.json                          # blindrelay MCP server (Streamable HTTP, no secrets)
├── .cursor-plugin/                   # Cursor-optimized layer (optional, single-click auth)
│   ├── plugin.json                   #   Cursor manifest + variables (BLINDRELAY_API_KEY)
│   └── mcp.json                      #   Fixed OAuth + PAT URLs; PAT header uses ${BLINDRELAY_API_KEY}
├── skills/
│   ├── setup/SKILL.md                # onboarding + auth
│   ├── send-mail/SKILL.md            # send transactional email
│   ├── manage-domains/SKILL.md       # add/verify/rotate-dkim/remove domains
│   ├── dedicated-ip/SKILL.md         # dedicated IP + inbound forward
│   └── delivery-observability/SKILL.md  # OTLP, webhook, usage, stats, suppression, audit
├── references/
│   ├── tools.md                      # full tool reference + scopes + error codes
│   ├── dual-origin.md                # OAuth vs PAT hosts
│   ├── no-content-on-disk.md         # the load-bearing invariant
│   └── transports.md                 # streamable-http, dual origin, self-hosted
├── README.md
└── LICENSE
```

The root `plugin.json` + `mcp.json` are the **portable** Agent Plugins 1.0 entry (works in every compatible client). The `.cursor-plugin/` directory is a **Cursor-only** layer that lets Cursor users set the API key once via a dashboard variable instead of editing MCP headers by hand. Non-Cursor clients ignore `.cursor-plugin/` and use the portable manifest. Cursor users: if your client lists the plugin twice, prefer the `.cursor-plugin/` entry and ignore the root one.

## Install

Public repository: <https://github.com/blindrelay-app/agent-plugin>

Install that repository from the plugin or connector UI. The same package works in Claude, ChatGPT, Cursor, and other Agent Plugins 1.0 hosts. Enable **one** server.

Example (Cursor): in chat run `/add-plugin https://github.com/blindrelay-app/agent-plugin`, then Install **blindrelay** in Customize. It does not appear in Cursor Marketplace search until it is listed there.

## Configure auth (one time)

Agent Plugins 1.0 does not let a plugin carry secrets, so the API key is client-managed. Enable **one** server in every client: OAuth `blindrelay` (Connect, no header) or PAT `blindrelay-pat` (same `br_…` key as the Keys UI).

### Example: Cursor

The `.cursor-plugin/` layer declares one dashboard variable. Set it once under **Plugins → Configure** when using the PAT server:

| Variable | Required | Purpose |
|----------|----------|---------|
| `BLINDRELAY_API_KEY` | no | Optional PAT for `blindrelay-pat` only. Leave empty if using OAuth Connect. |

The two endpoint URLs are fixed in the plugin (`https://api.blindrelay.app/mcp` and `https://api-pat.blindrelay.app/mcp`). They are not settings. Cursor substitutes `${BLINDRELAY_API_KEY}` into the PAT server's `Authorization: Bearer …` header. No manual header editing.

### Claude, ChatGPT, and other clients

1. Enable `blindrelay` and Connect (no API key), or enable `blindrelay-pat` and set `Authorization: Bearer br_…` on `https://api-pat.blindrelay.app/mcp`.
2. A PAT is a scoped key from <https://blindrelay.app/keys> (reveal once). `send` is enough on Free; `read` and `write` need Starter or higher.
3. Verify by calling `usage` (needs `read`) or `domains_list` (needs `read`). A 401 means the key/header is missing; a 403 `scope_missing` means the key lacks the scope.

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

## Safety properties

- **No content on disk.** Body/subject/headers are RAM-only; recipient is a bcrypt hash. Enforced by `backend/tests/no_content_on_disk.rs` on every release. The skills reinforce this on the agent side.
- **No secrets in the plugin.** The API key never ships in `mcp.json` headers; it lives in your client's secret store (Cursor dashboard variable, or your client's MCP auth settings).
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
