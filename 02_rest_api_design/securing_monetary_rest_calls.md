
# Securing Monetary REST API Calls (Mobile-First)

## Why Money Endpoints Are Different

Monetary operations are *irreversible side effects*.
A retry is not just a retry — it can be a double charge, a duplicated transfer,
or an audit nightmare.

Mobile makes this harder:
- Requests are retried automatically.
- Apps are killed mid-flight.
- Devices can be compromised.

This article outlines the **minimum security posture** for any money-moving endpoint.

---

## Do You Need Extra Encoding?

Short answer: **No — you need better controls, not custom crypto.**

- TLS already provides confidentiality and integrity.
- Base64, “scrambling”, or homegrown encryption add complexity, not safety.

Security comes from **authorization, integrity validation, idempotency, and replay protection**.

---

## Baseline Requirements

### 1) TLS Everywhere
- HTTPS only
- Modern ciphers
- HSTS on web surfaces
- Optional: certificate pinning on mobile if your threat model demands it

### 2) Authentication & Authorization
- Short‑lived access tokens
- Refresh tokens protected by platform keychain
- Server must verify: *this user can charge this quote*

---

## The Quote Pattern (Never Trust Client Amounts)

Clients should never submit raw charge amounts.

### Step 1 — Create Quote

```json
POST /quotes
→
{
  "quote_id": "q_8123",
  "total": { "amount_minor": 39748, "currency": "USD" },
  "expires_at": "2025-12-29T18:30:00Z"
}
```

### Step 2 — Charge Quote

```json
POST /charges
Headers:
  Idempotency-Key: 3F1C1F76...
Body:
{
  "quote_id": "q_8123",
  "payment_method_token": "pm_987"
}
```

Server recomputes all pricing rules and validates the quote is not expired.

---

## Idempotency & Replay Protection

- Every charge requires an `Idempotency-Key`.
- Server stores `(key, user_id, request_hash, response)` with TTL.
- Replays return the original response.

This prevents:
- OS-level retries
- User double-taps
- Network ambiguity

---

## Request Integrity

Server must recompute:
- Line items
- Taxes
- Discounts
- FX conversions
- Rounding

The client is **advisory only**.

---

## Rate Limiting & Abuse Detection

Money endpoints must have stricter limits:

| Signal | Purpose |
|-------|---------|
| Velocity checks | Prevent brute-force & abuse |
| Per-user rate limits | Protect accounts |
| Global anomaly detection | Catch fraud patterns |

---

## Auditing & Observability

Every monetary operation must log:

- user_id
- quote_id
- idempotency_key
- amount + currency
- timestamp
- correlation_id

These logs are your legal defense.

---

## Mobile-Specific Gotchas

| Gotcha | Impact |
|-------|--------|
| Silent retries | Duplicate charges |
| Quote expiry mid-flow | “Price changed” UX |
| Jailbroken devices | Tampered requests |
| Background suspension | Half-complete flows |

Design assuming the client can and will misbehave.

---

## Key Concepts

> “We treat money endpoints as high-risk workflows. Clients submit quote IDs, not amounts, every charge is idempotent, and the server recomputes all financial logic at the billing boundary.”
