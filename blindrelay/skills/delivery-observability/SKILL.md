---
name: delivery-observability
description: Configure and inspect Blindrelay delivery observability — OTLP export endpoints, delivery webhooks, usage/quota, delivery stats, per-message status, recent messages, suppression list, and control-plane audit. Use when the user says "set up OTLP", "point delivery events to Grafana/Datadog/Honeycomb", "delivery webhook", "check usage", "how many delivered/bounced", "was my email delivered", "status of message <id>", "suppression list", "remove suppression", or "audit log".
---

# Delivery observability and tenant operations

Blindrelay pushes delivery events to **your** OTLP endpoint and/or delivery webhook — your collector is the system of record. These tools configure where events go and inspect counters. Required scopes: `read` for getters/listers; `write` for save/test/remove; `email_status` also accepts a `send`-only key. `read`/`write` need Mini+.

## OTLP export

### `otlp_get` — read OTLP config

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `domain_id` | no | Omit for workspace (tenant) config; pass a domain UUID for a per-domain override. |

Returns `{ endpoint, protocol, has_auth_header, auth_header_preview, scope, domain_id, otlp_override }`. The auth header is **masked** — you can confirm it is set but cannot read it back.

### `otlp_save` — save OTLP config

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `endpoint` | yes | Your OTLP collector URL, e.g. `https://otlp.example.com`. |
| `protocol` | yes | `http` or `grpc`. Default `http`. |
| `auth_header` | no | Omit to keep the stored secret; empty string `""` clears it. |
| `domain_id` | no | Omit for workspace config; pass a UUID to set a per-domain override. |
| `clear_otlp_override` | no | `true` to drop a per-domain override and fall back to workspace config. |

### `otlp_test` — probe collector connectivity

Args: `endpoint`, `protocol`, `auth_header`, `domain_id` (all optional — override stored config for the probe only). Returns `{ success, message }`. Use this before flipping a collector change to catch typos / firewalls.

## Delivery webhook

### `delivery_settings_get` — read webhook config

Args: `domain_id` (optional, same semantics as `otlp_get`). Returns `{ enforce_soft_bounce_suppression, soft_bounce_suppression_locked, delivery_retry_enabled, webhook_url, has_webhook_auth_header, webhook_auth_header_preview, scope, domain_id, webhook_override, diagnostics }`. Auth header is masked.

`diagnostics` answers "is my hook working?" without a per-message archive: `{ attempts_total, failures_total, counting_since, last_attempt_at, last_success_at, last_failure_at, last_http_status, last_outcome, last_failure_category }`. Each scope counts its own hook — the workspace row counts attempts to the workspace hook, a `domain_id` row counts attempts to that domain's override. Saving a different hook URL restarts the counters (`counting_since` moves to that save). `last_outcome` is `ok` or one failure category:

| Category | Meaning |
|----------|---------|
| `rejected_url` | URL refused before sending (private address, bad scheme) |
| `connect_failed` | DNS lookup or TCP connect failed |
| `timeout` | No answer within the timeout (5 s for delivery) |
| `tls` | TLS handshake or certificate failure |
| `auth_failed` | Endpoint answered 401 or 403 |
| `auth_not_enforced` | Test only: endpoint accepted a request without or with a wrong auth header |
| `invalid_auth_header` | Saved auth header is not a valid HTTP header |
| `http_3xx` / `http_4xx` / `http_5xx` | Other non-2xx answer (redirects are not followed) |
| `client_error` | Any other client-side failure |

No URL, payload, envelope id or response body is stored or returned. When OTLP is configured, each attempt also increments `email.webhook.attempt.total{outcome, http_status_class, scope, domain}`.

### `delivery_settings_save` — save webhook config

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `webhook_url` | no | Your webhook URL. |
| `webhook_auth_header` | no | Omit to keep stored secret; empty string clears. |
| `enforce_soft_bounce_suppression` | no | Workspace-only. Soft-bounce suppression is workspace-scoped; per-domain overrides cannot set it. |
| `delivery_retry_enabled` | no | Workspace-only; default on. `true`: mail that cannot be delivered immediately waits on disk for up to 24h, then is deleted. `false`: mail stays in memory and a temporary failure is final. Pass it alone to change only this. |
| `domain_id` | no | Omit for workspace; pass UUID for per-domain override. |
| `clear_webhook_override` | no | `true` to drop a per-domain override. |

### `delivery_settings_test` — send a test webhook

Args: `webhook_url`, `webhook_auth_header`, `domain_id` (all optional overrides). Returns `{ success, message, category, http_status }` — the same categories as `diagnostics`; `http_status` is `null` when no HTTP answer arrived. A failed probe is `success: false`, not a tool error. Tests are not counted in `diagnostics`. The test request is signed when the webhook has a secret.

### Signing and retries

The save that first configures a webhook returns `webhook_signing_secret` **once**: tell the user to store it in their receiver now, do not write it to a tracked file. Each request carries `Blindrelay-Signature: t=<unix>,v1=<hex>`, the HMAC-SHA256 of `<t>.<delivery id>.<body>`, and `Blindrelay-Delivery-Id`, the same for every attempt of one event, so receivers can drop duplicates.

`has_webhook_signing_secret: false` on a saved webhook means it was set up before signing existed: it keeps working unsigned until `delivery_settings_rotate_secret` generates a secret.

### `delivery_settings_rotate_secret` — generate or rotate the signing secret

Args: `domain_id` (optional; omit for the workspace hook). Returns `{ signing_secret, previous_secret_expires_at, secret_rotated_at, scope, domain_id }`; the secret is shown once. For 24 hours from a rotation each request also carries a `v1` made with the replaced secret. `400 hook_not_configured` means that scope has no webhook. Requires `write`.

Failed deliveries are retried by a durable queue: a network error, 5xx or 429 gets up to 12 attempts over about 24 hours with exponential backoff; any other 4xx stops it, and removing the webhook drops what is queued. Every attempt counts in `diagnostics`.

## Usage and stats

### `usage` — current billing period

No args. Returns `{ period, total_count, inbound_count, quota, plan, by_domain[], spend_cap }`. `total_count` counts outbound sends and forwarded inbound mail; `inbound_count` is the inbound share (outbound = `total_count - inbound_count`), and each `by_domain[]` row has `count` and `inbound_count`. Use this to answer "how much have I sent this month" and "am I near quota". `spend_cap` is `{ monthly_message_cap, minimum, maximum }`, or `null` on Free and unlimited plans.

### `spend_cap_set` — limit overage spend

Args: `monthly_message_cap` (optional, from `spend_cap.minimum` to `spend_cap.maximum`; omit or null removes the cap). Sends past the cap return `429 spending_cap_reached`. Requires `write`. Change it only when the user asks: it stops their mail.

### `usage_history` — per-period, per-domain history

No args. Returns `UsageHistoryRow[]` with `period`, `domain`, `message_count` (outbound + forwarded inbound), `inbound_count`, `quota`.

### `delivery_stats` — delivery counters

Args: `period_hours` (optional, hours, default = envelope retention), `domain` (optional sending domain), `egress_key` (optional: `shared`, or a `key` from `egress_keys`). Returns `{ pending, relay_accepted, delivered, failed, failed_breakdown, retention_hours, period_hours, domain?, egress_key?, series_resolution, series, egress_keys }`. `series` is zero-filled, oldest first, one bucket per UTC hour up to 72h (`series_resolution: hour`) and per UTC day beyond (`day`); each bucket has `bucket_start`, `relay_accepted`, `delivered`, `failed`, `failed_breakdown` and the same filters as the totals. `egress_keys` lists `{ key, label, kind, ipv4? }` for shared and each dedicated IP the workspace sent through or held in the period — use a `key` to filter, show the `label` to people. Pending is only populated when unfiltered. Content-free counters — no per-message rows.

### `email_status` — status of one message

Args: `id` (the `id` returned by `send`, one per recipient; the `envelope_id` in delivery webhooks). Returns `{ id, status, summary, final, accepted_at, updated_at, lookup_window_hours }`. `status` uses the delivery webhook names: `queued` (every in-flight step), `relay_accepted`, `delivered`, `soft_bounce`, `hard_bounce`, `complaint`, `suppressed`, `failed`. `final: false` (`queued`, `relay_accepted`, `soft_bounce`) means a later event can still change it — check again later rather than reporting it as the outcome. `accepted_at` is `null` once the message has left the queue. `404 email_not_found` means the id is unknown in this workspace or older than `lookup_window_hours` (the two cannot be told apart). Lookup is by id only — there is no search by recipient. Rate limited per workspace (`429 rate_limited`). For "why didn't my email to X arrive?" without an id, use `suppression_lookup`.

### `email_list` — recent messages

Args: `status`, `domain`, `page_size` (default 50, at most 100), `cursor` (all optional). Returns `{ emails: [{ id, status, summary, final, domain, api_key_id, accepted_at, updated_at }], next_cursor, lookup_window_hours }`, newest activity first, for the delivery data window (`lookup_window_hours`). Use it to answer "did it go out?" without logs. Pass `next_cursor` back as `cursor` to page; `null` means the last page. There is **no recipient, subject or body** and no search by recipient. Required scope: `send` or `read`.

## Suppression

### `suppression_list` — list suppressed recipients (hashes only)

Args: `page`, `page_size` (both optional). Returns `{ summary, entries[], total, page, page_size }`. Each entry has `id` (UUID), `reason`, `created_at` — **no plaintext address**, only the bcrypt hash is stored. You cannot recover the address from this list.

### `suppression_export` — export suppression list

Args: `format` (optional, default `json`; also `csv`). Returns entries with `to_hash`, `reason`, `rejection_category`, `smtp_code`, `enhanced_status`, `created_at`. Again, hashes and codes only.

### `suppression_lookup` — find every entry for an address

Args: `address` (plaintext once; matched against the stored bcrypt hashes in process; never stored or logged). Returns `{ suppressed, entries, checked, complete }`: every matching entry (`{ id, reason, rejection_category, smtp_code, enhanced_status, created_at }`, newest first — one address can have several), how many entries were checked, and `complete: false` when older entries were not searched. Use it to answer "why didn't my email to X arrive?" and to find the right id before `suppression_remove`. Rate limited per workspace (`429 rate_limited`). Do not echo the address.

`rejection_category` explains a bounce: `invalid_recipient` (permanent hard bounce), `policy_or_authentication` (the receiver rejected the sender — fix SPF/DKIM/DMARC; the recipient is never suppressed for it), `mailbox_full`, `transient`, `unknown` (soft bounces). `null` on a bounce means it was recorded before causes were stored.

### `suppression_remove` — remove a suppression entry

Args: `id` (from `suppression_lookup` or `suppression_list`), `confirm` (required for `hard_bounce` and `complaint`), `confirm_complaint` (also required for `complaint`). Returns `{"ok":true}`. Any reason can be removed; the removal is audited as `suppression.removed`. Before removing a hard bounce, make sure the address works now; never set `confirm_complaint` without the user's explicit consent — the recipient reported the mail as spam. Missing flags answer `400 confirmation_required` / `complaint_confirmation_required`.

### `suppression_add` — hash-only manual add

Args: `address` (plaintext once; hashed bcrypt cost 12 in process; never log) **or** `to_hash` (precomputed bcrypt). Returns `{ id, reason=manual_suppression, created_at }`. Do not echo the address.

## Audit

### `audit_list` — control-plane audit events

Args: `page`, `page_size` (optional). Returns `{ entries[], total, page, page_size, retention_days }`. Entries are account/domain/API-key change events — **no message content**. Use this to answer "who changed what, when" for tenant config.

## No-content-on-disk

All tools in this skill return only metadata, counters, hashes, and config previews. None return email bodies, subjects, headers, or plaintext recipient addresses. The auth-header previews are masked. This is by design — keep it that way; never reconstruct or log a recipient address from a hash.

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [setup](../setup/SKILL.md) — if you hit 401/403.

**Product feedback:** after Blindrelay work, if something concrete worked surprisingly well or badly, draft ticket-ready feedback (area, sentiment, repro steps, desired vs observed, the user's underlying goal; no email addresses or message content), show it to the user and call `send_feedback` only after they approve; otherwise continue.
