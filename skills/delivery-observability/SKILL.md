---
name: delivery-observability
description: Configure and inspect Blindrelay delivery observability — OTLP export endpoints, delivery webhooks, usage/quota, delivery stats, suppression list, and control-plane audit. Use when the user says "set up OTLP", "point delivery events to Grafana/Datadog/Honeycomb", "delivery webhook", "check usage", "how many delivered/bounced", "suppression list", "remove suppression", or "audit log".
---

# Delivery observability and tenant operations

Blindrelay pushes delivery events to **your** OTLP endpoint and/or delivery webhook — your collector is the system of record. These tools configure where events go and inspect counters. Required scopes: `read` for getters/listers; `write` for save/test/remove. `read`/`write` need Starter+.

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

Args: `domain_id` (optional, same semantics as `otlp_get`). Returns `{ enforce_soft_bounce_suppression, soft_bounce_suppression_locked, webhook_url, has_webhook_auth_header, webhook_auth_header_preview, scope, domain_id, webhook_override }`. Auth header is masked.

### `delivery_settings_save` — save webhook config

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `webhook_url` | no | Your webhook URL. |
| `webhook_auth_header` | no | Omit to keep stored secret; empty string clears. |
| `enforce_soft_bounce_suppression` | no | Workspace-only. Soft-bounce suppression is workspace-scoped; per-domain overrides cannot set it. |
| `domain_id` | no | Omit for workspace; pass UUID for per-domain override. |
| `clear_webhook_override` | no | `true` to drop a per-domain override. |

### `delivery_settings_test` — send a test webhook

Args: `webhook_url`, `webhook_auth_header`, `domain_id` (all optional overrides). Returns `{ success, message }`.

## Usage and stats

### `usage` — current billing period

No args. Returns `{ period, total_count, quota, plan, by_domain[] }`. Use this to answer "how much have I sent this month" and "am I near quota".

### `usage_history` — per-period, per-domain history

No args. Returns `UsageHistoryRow[]` with `period`, `domain`, `message_count`, `quota`.

### `delivery_stats` — delivery counters

No args. Returns `{ pending, relay_accepted, delivered, failed, failed_breakdown, retention_hours, smtp_uses_relay }`. Use this to answer "did my emails deliver / bounce". Counters are integer buckets, not per-message rows — no content, no plaintext addresses.

## Suppression

### `suppression_list` — list suppressed recipients (hashes only)

Args: `page`, `page_size` (both optional). Returns `{ summary, entries[], total, page, page_size }`. Each entry has `id` (UUID), `reason`, `created_at` — **no plaintext address**, only the bcrypt hash is stored. You cannot recover the address from this list.

### `suppression_export` — export suppression list

Args: `format` (optional, default `json`; also `csv`). Returns entries with `to_hash`, `reason`, `created_at`. Again, hashes only.

### `suppression_remove` — remove a suppression entry

Args: `id` (the entry UUID from `suppression_list`). Returns `{"ok":true}`. **Confirm with the user first.** Removing a hard-bounce suppression lets future sends to that recipient attempt delivery again — only do this when the user states the original failure was transient (e.g. a mailbox-full that has since cleared) or the recipient explicitly asked to be re-added. Never bulk-remove on the agent's own initiative.

## Audit

### `audit_list` — control-plane audit events

Args: `page`, `page_size` (optional). Returns `{ entries[], total, page, page_size, retention_days }`. Entries are account/domain/API-key change events — **no message content**. Use this to answer "who changed what, when" for tenant config.

## No-content-on-disk

All tools in this skill return only metadata, counters, hashes, and config previews. None return email bodies, subjects, headers, or plaintext recipient addresses. The auth-header previews are masked. This is by design — keep it that way; never reconstruct or log a recipient address from a hash.

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [setup](../setup/SKILL.md) — if you hit 401/403.
