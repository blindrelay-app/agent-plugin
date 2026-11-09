---
name: manage-domains
description: Register, inspect, verify, rotate DKIM, and remove sending domains in Blindrelay via the domains_* MCP tools. Use when the user says "add a sending domain", "verify DKIM/SPF/DMARC", "DNS records", "DNS prompt", "rotate DKIM key", "remove a domain", or sending fails with domain_not_verified.
---

# Manage sending domains

Required scopes: `read` for `domains_list` / `domains_get` / `domains_dns_guide` / `domains_dmarc` / `domains_dmarc_sources`; `write` for `domains_add` / `domains_verify` / `domains_rotate_dkim` / `domains_remove`. `read`/`write` need Mini+.

## Lifecycle

```
domains_add  →  domains_dns_guide (or domains_get)  →  publish DNS  →  domains_verify  →  send
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
| `dedicated_ip_assignment_id` | no | Live dedicated IP assignment UUID to pin the domain to; omit to follow the workspace default. Get IDs from `dedicated_ip_get`. |

Returns a `DomainDetail` with the DNS records you must publish: SPF (TXT), DKIM (TXT, public key), DMARC (TXT), plus the recommended bounce CNAME (`bounce_record`: `bounce.<domain>` → platform bounce domain). The bounce record is optional and never blocks sending; once verified, bounces use the customer domain so SPF aligns with From. **Do not** publish these to a public chat channel verbatim if the user is on a shared screen — confirm the user wants them shown; they are not secrets, but they are customer infra.

### `domains_list` — list sending domains

No args. Returns `DomainSummary[]` with `id`, `domain`, `status`, `spf_verified`, `dkim_verified`, `dmarc_verified`, `dedicated_ip_assignment_id`, `dedicated_ip_use_shared`, `created_at` and `dns_checks` (per record `verified` plus an issue `code`/`message`, including the optional `bounce`). Use this to find the `id` you need for get/verify/rotate/remove.

### `domains_get` — fetch DNS/DKIM details

Args: `id` (the domain UUID from `domains_list`). Returns the full `DomainDetail` including the exact TXT record values to publish. Use this when the user wants structured JSON. For a copy-paste prompt a DNS agent can follow, use `domains_dns_guide` instead.

### `domains_dns_guide` — LLM DNS publish prompt

Args: `id`. Returns a plain-text prompt with the same records as `domains_get` (three required, one recommended bounce CNAME) plus merge-SPF / add-to-existing-DMARC-rua / split-DKIM / CNAME-proxy rules. Give this prompt to a DNS agent (Cloudflare MCP, registrar) as-is. Do not invent, omit, or paraphrase record values. After the records are saved, call `domains_verify` (or tell the user to click Verify now). This tool does not write customer DNS.

### `domains_dmarc` / `domains_dmarc_sources` — DMARC aggregate reports (Growth and up)

`domains_dmarc` (`id`) returns where reports go (`setup.rua_address`, `setup.points_at_us`), Blindrelay vs unknown traffic and `readiness`: status `insufficient_data`, `alignment_issues`, `review_sources` or `candidate` with a plain `recommendation`. Report it as evidence from received reports, never as a promise that `p=reject` is safe. `domains_dmarc_sources` (`id`, optional `class`, `days`, `limit`, `offset`) lists senders that are not Blindrelay by default. Below Growth both answer 403 `dmarc_plan_required`; `domains_get` still shows `dmarc_reporting`, and every plan gets the weekly digest by email.

### `domains_verify` — re-check SPF/DKIM/DMARC and the bounce CNAME

Args: `id`. Triggers a live DNS lookup and updates `*_verified` flags and `dns_checks`. `dns_checks.bounce` reports the optional bounce CNAME; it never changes `status`. Call this after the user has published the records from `domains_get`. If any flag is still `false`, show the user the expected record (from `domains_get`) and the value that DNS currently returns (the tool result explains the mismatch).

### `domains_rotate_dkim` — rotate the DKIM key

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `id` | yes | Domain UUID. |
| `selector` | no | New selector. Omit for the default rotation selector. |

Rotation generates a new key pair. After rotation the user must publish the new DKIM TXT record (returned in the response) and call `domains_verify`. Old selectors keep working until DNS propagates; do not remove the old record until the new one verifies.

### `domains_remove` — delete a sending domain

Args: `id`. Returns `{"ok":true}`. This disables sends from the domain immediately. Confirm with the user before calling — removal is reversible only by re-adding and re-verifying.

### `domains_set_dedicated_ip` — pin egress

Args:

| Field | Required | Notes |
|-------|----------|-------|
| `id` | yes | Domain UUID. |
| `assignment_id` | no | Live assignment UUID from `dedicated_ip_get`. |
| `use_shared` | no | `true` forces the shared pool. Cannot combine with `assignment_id`. |

Omit both `assignment_id` and `use_shared` to follow the workspace default. See [dedicated-ip](../dedicated-ip/SKILL.md).

## No-content-on-disk

Domain config (`local_part`, `destination`, DNS records) is operator/tenant data and is persisted by Blindrelay — this is expected and fine. The invariant only forbids **message content** (body/subject/headers) and **plaintext recipient addresses** on disk. None of the `domains_*` tools touch message content.

## Reference

- [tools.md](../../references/tools.md) — full tool reference.
- [setup](../setup/SKILL.md) — if you hit 401/403, or to see with `setup_status` which domain step (register, DNS, verify) is next and which DNS records are still pending.
