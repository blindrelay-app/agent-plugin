---
name: dedicated-ip
description: Request, inspect, cancel, and route Blindrelay dedicated IPs; manage inbound forward aliases and the HTTPS endpoints they deliver to (paid plans; a mailbox destination needs an active IP). Use when the user says "dedicated IP", "pin domain to IP", "inbound forward", "inbound alias", "inbound webhook", "receive email in my app", or send/domain quota errors mention dedicated_ip.
---

# Dedicated IP and inbound forward

Required scopes: `read` for `dedicated_ip_get` / `inbound_forward_get` / `inbound_endpoint_list`; `write` for request/cancel/pin/alias/endpoint mutations. `read`/`write` need Mini+; the dedicated IP and inbound forward need a paid plan too (Mini, Starter, Growth, Pro, Scale or Enterprise; not Free).

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

An IP has a daily limit until its own send history shows it is warm (`assignments[].warmup`). Mail above it uses the shared pool and counts in `overflowed_today`; a workspace over the normal abuse limits, or a spike past the shared IP's own limit, gets `warmup_limit_exceeded` instead (retry after midnight UTC).

## Domain pin

- `domains_set_dedicated_ip`: `assignment_id` pins; `use_shared=true` forces shared; omit both to follow workspace default.
- `domains_add` accepts `dedicated_ip_assignment_id` to pin a new domain to a live assignment of this workspace. Any number of domains may share one dedicated IP; only the plan's total domain count applies (`domain_limit_reached`).

## Inbound forward (paid plans)

Gate: a paid plan (`403 paid_plan_required` on Free). A mailbox `destination` also needs at least one assignment with status `active` (`403 inbound_forward_dedicated_ip_required`); an `endpoint_id` destination needs no dedicated IP. `inbound_forward_get` returns `eligible` (paid plan), `mailbox_eligible` (also an active IP), `mx_host` (publish MX), and aliases.

Each forwarded message counts as one message in the workspace's monthly usage (`usage.inbound_count`, included in `total_count`). At the quota hard stop or the spending cap, inbound mail is not forwarded and waits in the mail server's queue until the limit lifts.

- `inbound_forward_add`: `domain_id` (verified sending domain), `local_part`, and exactly one of `destination` (forward-to mailbox) or `endpoint_id` (HTTPS endpoint).
- `inbound_forward_update`: `destination` (switch to a mailbox) or `endpoint_id` (switch to an endpoint), and/or `enabled`.
- `inbound_forward_remove`: `id`.

Do not log destination addresses. Alias config is tenant data (persisted); message bodies are never stored.

## Inbound HTTPS endpoints (paid plans, no dedicated IP needed)

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

**Product feedback:** after Blindrelay work, if something concrete worked surprisingly well or badly, draft ticket-ready feedback (area, sentiment, repro steps, desired vs observed, the user's underlying goal; no email addresses or message content), show it to the user and call `send_feedback` only after they approve; otherwise continue.
