# No-content-on-disk invariant

Blindrelay is a **zero-content email relay**. This is a load-bearing product property, not a preference. Any agent using this plugin must respect it.

## What is never persisted

- Email **body**, **subject**, and **headers** — kept in process RAM only for the duration of the SMTP transaction, then discarded. Never written to Postgres, Redis, logs, or any persistent store. One narrow exception: a message the recipient's server defers waits with its recipient address in the relay's disk queue for up to 24 hours, then is deleted, unless the workspace set `delivery_retry_enabled` to false.
- **Plaintext recipient address** — replaced with a **bcrypt hash (cost 12)** before any durable write; a keyed HMAC lookup hash may be stored beside it for the send-time suppression check (its key is never in the database). The plaintext address never appears in stored records, audit logs, suppression lists, or operator tooling.

## What is persisted (and is fine)

- In-flight envelope metadata: `id`, `from`, recipient **bcrypt hash** and keyed lookup hash, domain, status, timestamp. Terminal sends delete the row.
- Bounce correlation: envelope id, tenant, optional api_key_id, hash, domain, egress_key, optional opaque `tags`, expiry — no message content.
- Delivery counters: integer buckets per tenant/domain/egress — no per-message rows.
- Suppression list: bcrypt hash and keyed lookup hash + reason (hard bounce, complaint, optional soft bounce). No plaintext address.
- Control-plane audit log: account/domain/API-key changes — no message content.
- Domain config: `local_part`, `destination`, DNS/DKIM records. (Inbound forward perk: bodies stay in RAM during transit, except that a forward to a mailbox the destination defers can wait in the retry spool, as outbound mail does.)
- OTLP export + delivery webhook events are pushed to the customer's endpoint — the customer's collector is the system of record.

`tags` on `send` are the one caller-chosen value stored with a message. The server refuses `@` and spaces, so never try to put an address or other personal data in one.

## What this means for the agent

1. **Never** echo the `to` address, `subject`, `html`, or `text` back after a successful `send`. Report only the envelope `id` and a short confirmation.
2. **Never** write the recipient address, subject, or body to a file, commit, test fixture, log, or chat transcript unless the user explicitly asked for it in the same turn (e.g. "show me the email I just drafted"). Even then, prefer to redact.
3. The `from` address and sending domain are customer-owned and safe to display.
4. The envelope `id` (UUID) is safe to log and reference.
5. Suppression lists return **hashes only** — you cannot recover the address, and you should not attempt to. If the user asks "is X@Y.com suppressed?", use `suppression_lookup` (the address is matched in process, never stored or logged) and do not repeat the address in your answer.
6. OTLP and delivery webhook auth headers are **masked** in `otlp_get` / `delivery_settings_get`. You can confirm a header is set but cannot read it back. Do not ask the user to paste it into a tracked file. A webhook signing secret is returned once and stored encrypted; the same applies to it.

## Why it matters

Customers choose Blindrelay specifically because they do not want a vendor to be a system of record for their email content (GDPR, HIPAA, procurement). An agent that logs a recipient address or a body into a chat, a commit, or a test fixture breaks the same guarantee the product is sold on. Treat the no-content rule as a hard constraint on every send-path interaction.
