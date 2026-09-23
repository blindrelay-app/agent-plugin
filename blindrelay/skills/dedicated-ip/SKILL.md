---
name: dedicated-ip
description: Request, inspect, cancel, and route Blindrelay dedicated IPs; manage inbound forward aliases (active-IP perk). Use when the user says "dedicated IP", "pin domain to IP", "inbound forward", "inbound alias", or send/domain quota errors mention dedicated_ip.
---

# Dedicated IP and inbound forward

Required scopes: `read` for `dedicated_ip_get` / `inbound_forward_get`; `write` for request/cancel/pin/alias mutations. `read`/`write` need Starter+; dedicated IP itself requires Growth, Pro, Scale, or Enterprise.

## Dedicated IP lifecycle

```
dedicated_ip_get  →  dedicated_ip_request  →  (open checkout_url in a browser if present)
                  →  dedicated_ip_resume_checkout (if they left Checkout)
                  →  dedicated_ip_set_default
                  →  domains_set_dedicated_ip / domains_add with dedicated_ip_assignment_id
                  →  dedicated_ip_cancel
```

1. Call `dedicated_ip_get` first. Use `assignments[].id` for pin/cancel/default. `eligible` false means the plan cannot buy the add-on.
2. `dedicated_ip_request` creates an assignment. If the JSON includes `checkout_url`, tell the user to open it in a browser — do not pretend payment completed in-chat.
3. `dedicated_ip_resume_checkout` with `id` if they abandoned Checkout while status is `pending_payment`.
4. `dedicated_ip_set_default` with a live (`warming`/`active`) assignment id.
5. `dedicated_ip_dismiss_closure` with `assignment_id` when `recent_closure` is present and the user has read the reason.
6. `dedicated_ip_cancel` with `id`. Confirm first. Cancel can fail with `domain_quota_blocks_cancel` if too many domains would fall back to shared egress — re-pin or remove extras, then retry.

## Domain pin

- `domains_set_dedicated_ip`: `assignment_id` pins; `use_shared=true` forces shared; omit both to follow workspace default.
- `domains_add` accepts `dedicated_ip_assignment_id` when registering past the shared-domain cap.

## Inbound forward (active IP only)

Gate: at least one assignment with status `active`. `inbound_forward_get` returns `eligible`, `mx_host` (publish MX), and aliases.

- `inbound_forward_add`: `domain_id` (verified sending domain), `local_part`, `destination` (forward-to mailbox).
- `inbound_forward_update`: `destination` and/or `enabled`.
- `inbound_forward_remove`: `id`.

Do not log destination addresses. Alias config is tenant data (persisted); message bodies are never stored.

## Reference

- [tools.md](../../references/tools.md)
- [manage-domains](../manage-domains/SKILL.md)
