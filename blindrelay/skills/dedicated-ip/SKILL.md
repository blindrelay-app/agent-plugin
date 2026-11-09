---
name: dedicated-ip
description: Request, inspect, cancel, and route Blindrelay dedicated IPs; manage inbound forward aliases and the HTTPS endpoints they deliver to (active-IP perk). Use when the user says "dedicated IP", "pin domain to IP", "inbound forward", "inbound alias", "inbound webhook", "receive email in my app", or send/domain quota errors mention dedicated_ip.
---

# Dedicated IP and inbound forward

Required scopes: `read` for `dedicated_ip_get` / `inbound_forward_get` / `inbound_endpoint_list`; `write` for request/cancel/pin/alias/endpoint mutations. `read`/`write` need Mini+; dedicated IP itself requires Growth, Pro, Scale, or Enterprise.

## Dedicated IP lifecycle

```
dedicated_ip_get  →  dedicated_ip_request  →  (open checkout_url in a browser if present)
                  →  dedicated_ip_resume_checkout (if they left Checkout)
                  →  dedicated_ip_set_default
                  →  domains_set_dedicated_ip / domains_add with dedicated_ip_assignment_id
                  →  dedicated_ip_cancel
```

1. Call `dedicated_ip_get` first. Use `assignments[].id` for pin/cancel/default. `eligible` false means the plan cannot buy the add-on.
2. `dedicated_ip_request` creates an assignment. If the JSON includes `checkout_url`, tell the user to open it in a browser — do not pretend payment completed in-chat. Paid plans are for business use under the Terms (confirmed at sign-up). The owner gave that confirmation when creating the workspace, so ask nothing extra.
3. `dedicated_ip_resume_checkout` with `id` if they abandoned Checkout while status is `pending_payment`.
4. `dedicated_ip_set_default` with a live (`warming`/`active`) assignment id.
5. `dedicated_ip_dismiss_closure` with `assignment_id` when `recent_closure` is present and the user has read the reason.
6. `dedicated_ip_cancel` with `id`. Confirm first. Domains pinned to it fall back to the workspace default (or the shared pool); the domain count never blocks a cancel.

During the first 21 days from activation an IP has a daily limit (`assignments[].warmup`). Mail above it uses the shared pool and counts in `overflowed_today`; a workspace over the normal abuse limits gets `warmup_limit_exceeded` instead.

## Domain pin

- `domains_set_dedicated_ip`: `assignment_id` pins; `use_shared=true` forces shared; omit both to follow workspace default.
- `domains_add` accepts `dedicated_ip_assignment_id` to pin a new domain to a live assignment of this workspace. Any number of domains may share one dedicated IP; only the plan's total domain count applies (`domain_limit_reached`).

## Inbound forward (active IP only)

Gate: at least one assignment with status `active`. `inbound_forward_get` returns `eligible`, `mx_host` (publish MX), and aliases.

- `inbound_forward_add`: `domain_id` (verified sending domain), `local_part`, and exactly one of `destination` (forward-to mailbox) or `endpoint_id` (HTTPS endpoint).
- `inbound_forward_update`: `destination` (switch to a mailbox) or `endpoint_id` (switch to an endpoint), and/or `enabled`.
- `inbound_forward_remove`: `id`.

Do not log destination addresses. Alias config is tenant data (persisted); message bodies are never stored.

## Inbound HTTPS endpoints (active IP only)

Deliver an alias's mail to the user's own app instead of a mailbox. Blindrelay POSTs the raw message (`message/rfc822`) signed with `Blindrelay-Signature: t=<unix>,v1=<hex HMAC-SHA256(secret, "<t>.<Blindrelay-Delivery-Id>.<Blindrelay-Alias>.<body>")>`; the receiver accepts when any `v1` matches and `t` is within 5 minutes.

```
inbound_endpoint_add  →  (user stores signing_secret)  →  inbound_endpoint_test
                      →  inbound_forward_add / inbound_forward_update with endpoint_id
```

- `inbound_endpoint_add`: `name`, `url` (https, public address). The response holds `signing_secret` **once**: give it to the user straight away; it cannot be read again.
- `inbound_endpoint_test`: posts a fixed sample message; report `category` and `http_status`.
- `inbound_endpoint_rotate_secret`: ask first. Returns a new secret once; for 24 hours from this rotation each delivery also carries a signature made with the secret it replaced, so the receiver can be updated without dropped mail. A second rotation inside the window replaces that one (the older secret stops signing).
- `inbound_endpoint_update`: `name` and/or `url` (a new URL resets the diagnostics).
- `inbound_endpoint_remove`: refused with `endpoint_in_use` while aliases deliver to it — move them with `inbound_forward_update` first.
- `inbound_endpoint_list`: per-endpoint `diagnostics.last_outcome` / `last_http_status` explain failures. Endpoint 2xx delivers; 408/425/429/5xx/timeouts defer (the mail server retries); 413 and other 4xx bounce to the sender.

## Reference

- [tools.md](../../references/tools.md)
- [manage-domains](../manage-domains/SKILL.md)
