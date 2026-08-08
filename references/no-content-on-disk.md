# No-content-on-disk invariant

Blindrelay is a **zero-content email relay**. This is a load-bearing product property, not a preference. Any agent using this plugin must respect it.

## What is never persisted

- Email **body**, **subject**, and **headers** — kept in process RAM only for the duration of the SMTP transaction, then discarded. Never written to Postgres, Redis, logs, or any persistent store.
- **Plaintext recipient address** — replaced with a **bcrypt hash (cost 12)** before any durable write. The plaintext address never appears in stored records, audit logs, suppression lists, or operator tooling.

## What is persisted (and is fine)

- In-flight envelope metadata: `id`, `from`, recipient **bcrypt hash**, domain, status, timestamp. Terminal sends delete the row.
- Bounce correlation: envelope id, tenant, optional api_key_id, hash, domain, egress_key, expiry — no message content.
- Delivery counters: integer buckets per tenant/domain/egress — no per-message rows.
- Suppression list: bcrypt hash + reason (hard bounce, complaint, optional soft bounce). No plaintext address.
- Control-plane audit log: account/domain/API-key changes — no message content.
- Domain config: `local_part`, `destination`, DNS/DKIM records. (Inbound forward perk keeps bodies in RAM only during transit.)
- OTLP export + delivery webhook events are pushed to the customer's endpoint — the customer's collector is the system of record.

## What this means for the agent

1. **Never** echo the `to` address, `subject`, `html`, or `text` back after a successful `send`. Report only the envelope `id` and a short confirmation.
2. **Never** write the recipient address, subject, or body to a file, commit, test fixture, log, or chat transcript unless the user explicitly asked for it in the same turn (e.g. "show me the email I just drafted"). Even then, prefer to redact.
3. The `from` address and sending domain are customer-owned and safe to display.
4. The envelope `id` (UUID) is safe to log and reference.
5. Suppression lists return **bcrypt hashes only** — you cannot recover the address, and you should not attempt to. If the user asks "is X@Y.com suppressed?", you cannot answer from the list alone; tell them the list is hash-only and they should test by sending (or check their own records).
6. OTLP and delivery webhook auth headers are **masked** in `otlp_get` / `delivery_settings_get`. You can confirm a header is set but cannot read it back. Do not ask the user to paste it into a tracked file.

## Why it matters

Customers choose Blindrelay specifically because they do not want a vendor to be a system of record for their email content (GDPR, HIPAA, procurement). An agent that logs a recipient address or a body into a chat, a commit, or a test fixture breaks the same guarantee the product is sold on. Treat the no-content rule as a hard constraint on every send-path interaction.
