# Blindrelay MCP tool reference

The `blindrelay` MCP server (Streamable HTTP → `https://api.blindrelay.app/mcp`) exposes the same operations as the Blindrelay CLI over the machine API (`/v1/*`). Auth: `Authorization: Bearer <api_key>`. Scopes: `send`, `read`, `write` (Free = `send` only; `read`/`write` need Starter+).

## send

Queue a transactional email. Scope: `send`.

| Arg | Type | Required |
|-----|------|----------|
| `from` | string | yes |
| `to` | string | yes |
| `subject` | string | yes |
| `html` | string | yes |
| `text` | string | no |

Returns `{ id, status, warnings[] }`. See [send-mail](../skills/send-mail/SKILL.md).

## domains

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `domains_list` | `read` | — | `DomainSummary[]` |
| `domains_get` | `read` | `id` | `DomainDetail` (incl. DNS records) |
| `domains_add` | `write` | `domain`, `selector?` | `DomainDetail` |
| `domains_verify` | `write` | `id` | `DomainSummary` |
| `domains_rotate_dkim` | `write` | `id`, `selector?` | rotation result + new DKIM record |
| `domains_remove` | `write` | `id` | `{"ok":true}` |

See [manage-domains](../skills/manage-domains/SKILL.md).

## usage / stats

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `usage` | `read` | — | `{ period, total_count, quota, plan, by_domain[] }` |
| `usage_history` | `read` | — | `UsageHistoryRow[]` |
| `delivery_stats` | `read` | — | `{ pending, relay_accepted, delivered, failed, failed_breakdown, retention_hours, smtp_uses_relay }` |

## suppression

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `suppression_list` | `read` | `page?`, `page_size?` | `{ summary, entries[], total, page, page_size }` (hashes only) |
| `suppression_export` | `read` | `format?` (`json` default, `csv`) | entries with `to_hash`, `reason`, `created_at` |
| `suppression_remove` | `write` | `id` | `{"ok":true}` |

## audit

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `audit_list` | `read` | `page?`, `page_size?` | `{ entries[], total, page, page_size, retention_days? }` (no message content) |

## OTLP export

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `otlp_get` | `read` | `domain_id?` | `OtlpConfig` (auth header masked) |
| `otlp_save` | `write` | `endpoint`, `protocol`, `auth_header?`, `domain_id?`, `clear_otlp_override?` | `OtlpConfig` |
| `otlp_test` | `write` | `endpoint?`, `protocol?`, `auth_header?`, `domain_id?` | `{ success, message }` |

## Delivery webhook

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `delivery_settings_get` | `read` | `domain_id?` | `DeliverySettings` (auth header masked) |
| `delivery_settings_save` | `write` | `webhook_url?`, `webhook_auth_header?`, `enforce_soft_bounce_suppression?`, `domain_id?`, `clear_webhook_override?` | `DeliverySettings` |
| `delivery_settings_test` | `write` | `webhook_url?`, `webhook_auth_header?`, `domain_id?` | `{ success, message }` |

## Error codes

| Code | Retryable | Meaning |
|------|-----------|---------|
| `quota_exceeded` | yes | Plan quota hit; wait or upgrade |
| `draining` | yes | Stack is draining; retry shortly |
| `queue_at_capacity` | yes | Transient backpressure; retry |
| `invalid_payload` | no | Bad request body |
| `recipient_suppressed` | no | Recipient is on suppression list |
| `domain_not_verified` | no | `from` domain missing or unverified |
| `scope_missing` | no | API key lacks the required scope |
| `tenant_suspended` | no | Tenant is suspended |
| `unauthorized` | no | Missing/invalid API key |
