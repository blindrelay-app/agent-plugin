# Blindrelay MCP tool reference

The `blindrelay` MCP server exposes the same operations as the Blindrelay CLI over `/v1/*`. **Plugin first.** OAuth host: `https://api.blindrelay.app/mcp`. PAT host: `https://api-pat.blindrelay.app/mcp` with `Authorization: Bearer <api_key>`. Scopes: `send`, `read`, `write` (Free = `send` only).

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
| `domains_dns_guide` | `read` | `id` | plain-text LLM prompt (`{ prompt }` on REST) |
| `domains_add` | `write` | `domain`, `selector?`, `dedicated_ip_assignment_id?` | `DomainDetail` |
| `domains_verify` | `write` | `id` | `DomainSummary` |
| `domains_rotate_dkim` | `write` | `id`, `selector?` | rotation result + new DKIM record |
| `domains_remove` | `write` | `id` | `{"ok":true}` |
| `domains_set_dedicated_ip` | `write` | `id`, `assignment_id?`, `use_shared?` | `DomainSummary` with pin fields |

`domains_list` / `domains_get` include `dedicated_ip_assignment_id` and `dedicated_ip_use_shared`.

See [manage-domains](../skills/manage-domains/SKILL.md).

## Dedicated IP

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `dedicated_ip_get` | `read` | — | eligibility, `assignments[]`, `default_assignment_id`, optional `recent_closure` |
| `dedicated_ip_request` | `write` | — | assignment; `checkout_url` when self-serve |
| `dedicated_ip_cancel` | `write` | `id` | cancelled assignment |
| `dedicated_ip_resume_checkout` | `write` | `id` | `{ checkout_url }` |
| `dedicated_ip_set_default` | `write` | `id` | `{"ok":true}` |
| `dedicated_ip_dismiss_closure` | `write` | `assignment_id` | `{"ok":true}` |

Self-serve checkout URLs must be opened in a browser. See [dedicated-ip](../skills/dedicated-ip/SKILL.md).

## Inbound forward

Requires an **active** dedicated IP.

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `inbound_forward_get` | `read` | — | `{ eligible, mx_host, max_aliases, aliases[] }` |
| `inbound_forward_add` | `write` | `domain_id`, `local_part`, `destination` | alias |
| `inbound_forward_update` | `write` | `id`, `destination?`, `enabled?` | alias |
| `inbound_forward_remove` | `write` | `id` | `{"ok":true}` |

## usage / stats

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `usage` | `read` | — | `{ period, total_count, quota, plan, by_domain[] }` |
| `usage_history` | `read` | — | `UsageHistoryRow[]` |
| `delivery_stats` | `read` | `period_hours?`, `domain?`, `egress_key?` | `{ pending, relay_accepted, delivered, failed, failed_breakdown, retention_hours, period_hours, domain?, egress_key? }` |

## suppression

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `suppression_list` | `read` | `page?`, `page_size?` | `{ summary, entries[], total, page, page_size }` (hashes only) |
| `suppression_export` | `read` | `format?` (`json` default, `csv`) | entries with `to_hash`, `reason`, `created_at` |
| `suppression_remove` | `write` | `id` | `{"ok":true}` |
| `suppression_add` | `write` | `address?` or `to_hash?` | `{ id, reason, created_at }` — plaintext hashed in process, never logged |

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
