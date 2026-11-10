# Blindrelay MCP tool reference

The Blindrelay MCP servers expose the same operations as the Blindrelay CLI over `/v1/*`. **Plugin first.** OAuth host: `https://api.blindrelay.app/mcp`. PAT host: `https://api-pat.blindrelay.app/mcp` with `Authorization: Bearer <api_key>`. Scopes: `send`, `read`, `write` (Free = `send` only).

## setup_status

Where the workspace's setup stands. Any scope. Call it right after connecting and again after each step; it resumes wherever the workspace is. Same steps as the dashboard checklist and `blindrelay setup`.

| Field | Meaning |
|-------|---------|
| `steps[]` | `connected`, `domain_registered`, `dns_published`, `domain_verified`, `test_send`, in order: `{ id, title, done, blocked_reason?, blocked_code?, missing_scopes[], next_action }` |
| `next_action` | `{ summary, ui_path, ui_url, cli[], mcp_tools[], required_scopes[] }`; `null` once done |
| `steps[].pending_records` | `dns_published` only: `{ record, type, host, value, code, message }` for records the last check did not find |
| `next_step` / `complete` | First step not done; `true` when every step is done |
| `domain` | `{ id, domain, status }` the domain steps refer to |
| `plan_restrictions[]` | `{ code, message, steps[], limit?, used? }`: `plan_restricted` (Free keys are `send`-only: do domain steps in the dashboard), `domain_limit_reached`, `quota_exceeded` |
| `missing_scopes` | Scopes this key lacks for the remaining steps |

`blocked_code`: `previous_step_pending`, `scope_missing`, or a plan restriction code. Follow `next_action.mcp_tools` (`domains_add`, `domains_dns_guide`, `domains_verify`, `send`). Never send the test email without the user's explicit consent and a recipient they typed.

## send

Queue a transactional email. Scope: `send`.

| Arg | Type | Required |
|-----|------|----------|
| `from` | string | yes |
| `to` | string | yes |
| `subject` | string | yes |
| `html` | string | yes |
| `text` | string | no |
| `headers` | object | no (e.g. `Reply-To`, `List-Unsubscribe`) |
| `cc` | string or string[] | no. Shown in `Cc`; with `to` and `bcc`, at most 50 addresses. Each address is one envelope and one quota unit |
| `bcc` | string or string[] | no. Never in any header of any copy; shares the 50 address limit |
| `tags` | object[] | no. `{ key, value }`, at most 5; key `[a-z0-9_]{1,32}`, value `[A-Za-z0-9_.-]{1,64}`, so a tag cannot hold an address. Returned on the message's webhook events; `400 invalid_tag` names the position, never the value |
| `attachments` | object[] | no. Paid plans. `{ filename, content_type, content_base64 }`, at most 10 files and 10,240,000 decoded bytes; each recipient costs one quota unit per started MiB. The MCP request is capped at 1 MiB, so larger files need the CLI or REST API |

Returns `{ id, status, warnings[] }`. See [send-mail](../skills/send-mail/SKILL.md).

## email_check

Check an email before sending it. Evaluated in the MCP process with the same rules as `blindrelay check`, `POST /v1/emails/check` and the dashboard Email check page. Any key; needs no verified domain and does not count against quota. Never fetches links; content is checked in memory, never stored or logged, and findings never quote it.

| Arg | Type | Required |
|-----|------|----------|
| `subject` | string | no |
| `html` | string | no |
| `text` | string | no |
| `from` | string | no (checked for a placeholder domain) |

Same names as `send`; an omitted field is not checked. Returns `{ ok, error_count, warning_count, truncated, findings[] }`, each finding `{ code, severity: error\|warning, field: subject\|from\|html\|text, message, location: { line, column } \| null }`; `ok` is `false` when any error exists. Codes: `unresolved_template_variable`, `link_empty`, `link_javascript`, `link_placeholder`, `link_relative`, `link_insecure` (warning), `unclosed_tag`, `mismatched_tag`, `stray_closing_tag`, `unclosed_comment`, `script_tag`, `iframe_tag`, `form_tag` (warning), `external_stylesheet` (warning), `img_missing_alt` (warning), `text_alternative_missing` (warning), `subject_empty` (warning), `html_too_large` (warning), `body_missing`, `from_placeholder`. A field over the `send` limits is a tool error `payload_too_large` (413).

## domains

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `domains_list` | `read` | — | `DomainSummary[]` |
| `domains_get` | `read` | `id` | `DomainDetail` (incl. DNS records) |
| `domains_dns_guide` | `read` | `id` | plain-text LLM prompt (`{ prompt }` on REST) |
| `domains_dmarc` | `read` | `id` | `DmarcSummary` (setup, traffic, `readiness`); 403 `dmarc_plan_required` below Growth |
| `domains_dmarc_sources` | `read` | `id`, `class?` (`unknown` default, `blindrelay`, `all`), `days?`, `limit?`, `offset?` | `DmarcSources`; 403 `dmarc_plan_required` below Growth |
| `domains_add` | `write` | `domain`, `selector?`, `dedicated_ip_assignment_id?` | `DomainDetail`; 403 `domain_limit_reached` once the workspace holds its plan's total domain count (Free 1; paid plans unlimited, fair use), shared and dedicated-IP domains alike |
| `domains_verify` | `write` | `id` | `DomainSummary` |
| `domains_rotate_dkim` | `write` | `id`, `selector?` | rotation result + new DKIM record |
| `domains_remove` | `write` | `id` | `{"ok":true}` |
| `domains_set_dedicated_ip` | `write` | `id`, `assignment_id?`, `use_shared?` | `DomainSummary` with pin fields |

`domains_list` / `domains_get` include `dedicated_ip_assignment_id` and `dedicated_ip_use_shared`.

See [manage-domains](../skills/manage-domains/SKILL.md).

## Dedicated IP

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `dedicated_ip_get` | `read` | — | eligibility, `assignments[]`, `default_assignment_id`, optional `recent_closure`. `assignments[].inventory.blocklist` is null until the first check, else `{ checked_at, listed_zones, context_listed_zones, errored_zones }`: only the lists Blindrelay checks; `context_listed_zones` are informational; a zone in `errored_zones` was not checked, so it is not clean. `assignments[].warmup` is null until live, else `{ warm, daily_limit, sent_today, overflowed_today, exempt }`; the limit follows the IP's own history and is null when warm or exempt |
| `dedicated_ip_request` | `write` | — | assignment; `checkout_url` when self-serve |
| `dedicated_ip_cancel` | `write` | `id` | cancelled assignment |
| `dedicated_ip_resume_checkout` | `write` | `id` | `{ checkout_url }` |
| `dedicated_ip_set_default` | `write` | `id` | `{"ok":true}` |
| `dedicated_ip_dismiss_closure` | `write` | `assignment_id` | `{"ok":true}` |

Self-serve checkout URLs must be opened in a browser. Paid plans are for business use under the Terms (confirmed at sign-up). See [dedicated-ip](../skills/dedicated-ip/SKILL.md).

## Inbound forward

Requires an **active** dedicated IP.

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `inbound_forward_get` | `read` | — | `{ eligible, mx_host, max_aliases, aliases[], max_endpoints, endpoints[] }` |
| `inbound_forward_add` | `write` | `domain_id`, `local_part`, `destination` (mailbox) **or** `endpoint_id` | alias (`destination_kind`, `destination`, `endpoint_id`) |
| `inbound_forward_update` | `write` | `id`, `destination?` or `endpoint_id?`, `enabled?` | alias |
| `inbound_forward_remove` | `write` | `id` | `{"ok":true}` |
| `inbound_endpoint_list` | `read` | — | `{ max_endpoints, endpoints[] }` (no secrets; `diagnostics` per endpoint) |
| `inbound_endpoint_add` | `write` | `name`, `url` (https) | `{ endpoint, signing_secret }` — secret shown **once** |
| `inbound_endpoint_update` | `write` | `id`, `name?`, `url?` | endpoint |
| `inbound_endpoint_remove` | `write` | `id` | `{"ok":true}`; `409 endpoint_in_use` while aliases use it |
| `inbound_endpoint_rotate_secret` | `write` | `id` | `{ endpoint, signing_secret }` — new secret once; the old one signs for 24 h |
| `inbound_endpoint_test` | `write` | `id` | `{ success, message, category, http_status }` (not counted) |

Endpoint deliveries are a signed `POST` of the raw message (`message/rfc822`, `Blindrelay-Signature: t=…,v1=…`). Endpoint 2xx delivers; 408/425/429/5xx/timeouts defer in the mail server's queue; 413 and other 4xx bounce.

## usage / stats

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `usage` | `read` | — | `{ period, total_count, quota, plan, by_domain[], spend_cap }`; `spend_cap` is `{ monthly_message_cap, minimum, maximum }`, `null` on Free and unlimited plans |
| `usage_history` | `read` | — | `UsageHistoryRow[]` |
| `spend_cap_set` | `write` | `monthly_message_cap?` (from `spend_cap.minimum` to `spend_cap.maximum`; omit or null removes it) | `{ monthly_message_cap, minimum, maximum }`; `plan_restricted` on Free and unlimited plans; `admin_required` when the key's creator is not an owner or admin |
| `delivery_stats` | `read` | `period_hours?`, `domain?`, `egress_key?` | `{ pending, relay_accepted, delivered, failed, failed_breakdown, retention_hours, period_hours, domain?, egress_key?, series_resolution, series[], egress_keys[] }` |
| `email_status` | `send` or `read` | `id` (from `send`) | `{ id, status, summary, final, accepted_at, updated_at, lookup_window_hours, sandbox }` — `sandbox` true for a sandbox API key, which delivers nothing; status names match the delivery webhook events; lookup by id only, no recipient or content; `404 email_not_found` when unknown or older than the window; rate limited per workspace |
| `email_list` | `send` or `read` | `status?`, `domain?`, `page_size?` (max 100), `cursor?` | `{ emails: [{ id, status, summary, final, domain, api_key_id, accepted_at, updated_at, sandbox }], next_cursor, lookup_window_hours }` — recent messages for the delivery data window, newest activity first; never recipient or content, and no search by recipient |

## suppression

| Tool | Scope | Args | Returns |
|------|-------|------|---------|
| `suppression_list` | `read` | `page?`, `page_size?` | `{ summary, entries[], total, page, page_size }` (hashes only) |
| `suppression_export` | `read` | `format?` (`json` default, `csv`) | entries with `to_hash`, `reason`, rejection codes, `created_at` |
| `suppression_remove` | `write` | `id`, `confirm?` (hard bounce, complaint), `confirm_complaint?` (complaint) | `{"ok":true}` — audited as `suppression.removed` |
| `suppression_lookup` | `read` | `address` | `{ suppressed, entries[], checked, complete }` with rejection codes — matched in process, never stored or logged; rate limited per workspace |
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
| `delivery_settings_get` | `read` | `domain_id?` | `DeliverySettings` (auth header masked) with `diagnostics { attempts_total, failures_total, counting_since, last_attempt_at, last_success_at, last_failure_at, last_http_status, last_outcome, last_failure_category }` for that scope's hook; content-free, reset when the hook URL changes |
| `delivery_settings_save` | `write` | `webhook_url?`, `webhook_auth_header?`, `enforce_soft_bounce_suppression?`, `delivery_retry_enabled?`, `domain_id?`, `clear_webhook_override?` | `DeliverySettings`; the save that first configures a webhook also returns `webhook_signing_secret`, once |
| `delivery_settings_test` | `write` | `webhook_url?`, `webhook_auth_header?`, `domain_id?` | `{ success, message, category, http_status }` — same categories as `diagnostics`; a failed probe is `success: false`; not counted as an attempt |
| `delivery_settings_rotate_secret` | `write` | `domain_id?` | `{ signing_secret, previous_secret_expires_at, secret_rotated_at, scope, domain_id }` — `signing_secret` is shown once; generates the first secret of an unsigned webhook, or rotates with a 24-hour overlap |

## Error codes

An API refusal comes back as a tool error whose text is the REST envelope plus the HTTP status: `{"error": {"code", "message", "status"}}`. The codes are the same as REST and the CLI.

| Code | Retryable | Meaning |
|------|-----------|---------|
| `quota_exceeded` | yes | Plan quota hit; wait or upgrade |
| `warmup_limit_exceeded` | yes | Dedicated IP at its daily warmup limit and this mail cannot use the shared pool (workspace over the normal abuse limits, or no shared route); retry after the UTC day rolls over |
| `spending_cap_reached` | no | The workspace's own monthly cap is reached; raise or clear it with `spend_cap_set`, or wait for the next period |
| `draining` | yes | Stack is draining; retry shortly |
| `queue_at_capacity` | yes | Transient backpressure; retry |
| `invalid_payload` | no | Bad request body, path id or query (validation or parse failure) |
| `domain_not_found` | no | Domain id unknown in this workspace (every tool that takes a domain id) |
| `plan_restricted` | no | The plan does not allow this (e.g. Free keys are `send`-only) |
| `rate_limited` | yes | Request throttle; wait `retry_after_secs` |
| `recipient_suppressed` | no | Recipient is on suppression list |
| `domain_not_verified` | no | `from` domain missing or unverified |
| `scope_missing` | no | API key lacks the required scope |
| `email_not_found` | no | Message id unknown in this workspace or older than `lookup_window_hours` |
| `tenant_suspended` | no | Tenant is suspended |
| `unauthorized` | no | Missing/invalid API key |
