
# Idempotency and Retries in Mobile Systems

Retries are inevitable on mobile. Without idempotency, retries create data corruption.

> **Every mobile write request must be safe to replay.**

---

## Why Idempotency Is Non‑Negotiable

Mobile apps retry because:

- Radios drop mid-request
- Apps are backgrounded
- DNS fails
- TLS handshakes time out

The client often cannot know whether the server processed the request.

Without idempotency, retries become **duplicate side effects**:
- Double-charged payments
- Duplicate records
- Corrupted counters

---

## What Is Idempotency?

An operation is idempotent if executing it multiple times produces the same result as executing it once.

Examples:

| Operation | Idempotent? |
|----------|-------------|
| Create record without key | ❌ |
| Create with idempotency key | ✅ |
| Update by primary key | ✅ |
| Increment counter | ❌ |
| Set counter to value | ✅ |

---

## The Idempotency Key Pattern

### Client behavior

- Generate a UUID per user intent.
- Persist it locally *before* dispatch.
- Attach to every retry.

Header example:

```
Idempotency-Key: 3F1C1F76-2C3D-4F12-A5E7-1F9E4A4F93A2
```

### Server behavior

- Store `(key, request_hash, response)` with TTL.
- If the same key is seen again:
  - Return the original response.
  - Do **not** re-run side effects.

---

## What Must Be Hashed

To prevent misuse:

- HTTP method
- Path
- Canonicalized body
- Auth context

This prevents reusing keys across unrelated operations.

---

## Retry Classification

| Failure Type | Retry? |
|--------------|--------|
| Timeout | ✅ |
| DNS failure | ✅ |
| TLS error | ✅ |
| 5xx | ✅ |
| 4xx validation | ❌ |
| Auth expired | Refresh then retry |
| Conflict | Maybe (domain dependent) |

---

## Backoff Strategy

- Exponential backoff with jitter.
- Example: 1s, 2s, 4s, 8s… capped.
- Global retry budget to protect battery.

---

## Battery‑Aware Retry Budget

Retries cost:

- Radio power
- CPU wakeups
- Background execution time

Best practice:

- Define per‑operation retry limits.
- Pause retries when battery saver / low power mode is active.
- Defer non‑critical retries until charging or Wi‑Fi.

---

## Persistence Is Part of Idempotency

If the app crashes, the retry state must survive:

- Persist idempotency key
- Persist payload hash
- Persist retry count & last error

Treat retry queues as durable storage, not in‑memory helpers.

---

## Server-Side Pitfalls

- Keys with no TTL → memory leaks
- Keys scoped too broadly → false dedupe
- No hashing → replay attacks
- No telemetry → impossible debugging

---

## Key Concept

> “In mobile, retries are guaranteed. Without idempotency, retries become data corruption. Treat idempotency keys as first‑class domain primitives, not networking hacks.”
