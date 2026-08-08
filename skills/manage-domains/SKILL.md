---
name: manage-domains
description: Register, inspect, verify, rotate DKIM, and remove sending domains in Blindrelay via the domains_* MCP tools. Use when the user says "add a sending domain", "verify DKIM/SPF/DMARC", "rotate DKIM key", "remove a domain", or sending fails with domain_not_verified.
---

# Manage sending domains

Required scopes: `read` for `domains_list` / `domains_get`; `write` for `domains_add` / `domains_verify` / `domains_rotate_dkim` / `domains_remove`. `read`/`write` need Starter+.

## Lifecycle

```
domains_add  →  domains_get (publish DNS records)  →  domains_verify  →  send
                domains_rotate_dkim (key rotation)        ↑ re-check
                domains_remove (decommission)
```

## Tools

### `domains_add` — register a sending domain

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `domain` | yes | Apex or subdomain you control DNS for, e.g. `mail.example.com`. |
| `selector` | no | DKIM selector. Omit to let Blindrelay pick the default. |

Returns a `DomainDetail` with the DNS records you must publish: SPF (TXT), DKIM (TXT, public key), DMARC (TXT). **Do not** publish these to a public chat channel verbatim if the user is on a shared screen — confirm the user wants them shown; they are not secrets, but they are customer infra.

### `domains_list` — list sending domains

No args. Returns `DomainSummary[]` with `id`, `domain`, `status`, `spf_verified`, `dkim_verified`, `dmarc_verified`, `created_at`. Use this to find the `id` you need for get/verify/rotate/remove.

### `domains_get` — fetch DNS/DKIM details

Args: `id` (the domain UUID from `domains_list`). Returns the full `DomainDetail` including the exact TXT record values to publish. This is the tool to call when the user says "show me the DNS records for X".

### `domains_verify` — re-check SPF/DKIM/DMARC

Args: `id`. Triggers a live DNS lookup and updates `*_verified` flags. Call this after the user has published the records from `domains_get`. If any flag is still `false`, show the user the expected record (from `domains_get`) and the value that DNS currently returns (the tool result explains the mismatch).

### `domains_rotate_dkim` — rotate the DKIM key

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `id` | yes | Domain UUID. |
| `selector` | no | New selector. Omit for the default rotation selector. |

Rotation generates a new key pair. After rotation the user must publish the new DKIM TXT record (returned in the response) and call `domains_verify`. Old selectors keep working until DNS propagates; do not remove the old record until the new one verifies.

### `domains_remove` — delete a sending domain

Args: `id`. Returns `{"ok":true}`. This disables sends from the domain immediately. Confirm with the user before calling — removal is reversible only by re-adding and re-verifying.

## No-content-on-disk

Domain config (`local_part`, `destination`, DNS records) is operator/tenant data and is persisted by Blindrelay — this is expected and fine. The invariant only forbids **message content** (body/subject/headers) and **plaintext recipient addresses** on disk. None of the `domains_*` tools touch message content.

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [setup](../setup/SKILL.md) — if you hit 401/403.
