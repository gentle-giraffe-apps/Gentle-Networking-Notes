
# Error Taxonomy for Mobile APIs (Defensive Design)

## Why Error Taxonomy Matters

On mobile, errors are not just HTTP status codes — they are **product states**.

A backend may treat an error as “a failed request,” but a mobile app must decide:

- Should the user retry?
- Should we retry automatically?
- Should we refresh auth and replay?
- Should we show a blocking screen?
- Should we degrade gracefully (cached/offline)?
- Should we queue the operation for later sync?

If you don’t define a clear error taxonomy, you end up with:

- Infinite retry loops
- Battery-draining storms
- Confusing UX (“Something went wrong”)
- Silent data corruption (especially with writes)
- Production debugging that becomes guesswork

This article proposes a **mobile-first error taxonomy** with gotchas, anti-patterns, and best practices.

---

## Core Principle: Separate *Transport*, *Auth*, *Domain*, and *System* Errors

A single HTTP response often bundles multiple concerns. Mobile clients do better when they split errors into layers:

1. **Transport layer** (can’t reach server / TLS / timeout)
2. **Protocol layer** (HTTP status + headers)
3. **Auth layer** (token expired, session revoked, MFA needed)
4. **Domain layer** (validation, business rule failures)
5. **System layer** (rate limits, server overload, dependency failures)
6. **Client state layer** (offline, backgrounded, stale cache, schema mismatch)

Then each layer has explicit handling rules.

---

## The Mobile Error Categories

### 1) Connectivity / Transport Errors
Examples:
- No internet / airplane mode
- DNS failure
- TCP reset
- TLS handshake failure
- Timeout

**Mobile handling**
- Treat as **recoverable** (usually).
- Switch UI to offline/cached mode if available.
- Retry with **exponential backoff + jitter** (bounded).
- For writes: enqueue durable operation and retry later (with idempotency).

**Gotchas**
- iOS may return errors that look fatal but are transient (e.g., “network connection was lost”).
- Captive portals can look like “connected” but still fail requests.
- Background execution limits can turn a “timeout” into “app suspended.”

---

### 2) HTTP Protocol Errors (Status + Headers)
HTTP status codes are not the taxonomy — they are a *signal*.

**Mobile handling**
- Use status code + error body to map into your domain categories.

**Gotchas**
- Treating all 4xx as “client bug” is wrong.
  - `401` might be refreshable.
  - `409` might be recoverable via reconciliation.
  - `429` is a backoff instruction.
- Treating all 5xx as retryable is also wrong if it’s systemic or long-lived.

---

### 3) Authentication / Session Errors
Examples:
- Access token expired
- Refresh token revoked
- User logged out elsewhere
- Account locked
- MFA required

**Mobile handling**
- Token expired → **refresh + replay** original requests.
- Refresh revoked → move to signed-out state (but preserve local drafts).
- MFA required → transition to auth UX flow without losing pending work.

**Gotchas**
- **Refresh storms:** multiple requests hit `401` simultaneously and all refresh.
  - Fix: coalesce refresh into one inflight task.
- Clearing all local state on auth failure can destroy user trust.
  - Preserve drafts and queued operations where safe.

---

### 4) Validation Errors (Client Input)
Examples:
- Missing required field
- Invalid format
- Out-of-range values

**Mobile handling**
- Don’t auto-retry.
- Surface actionable feedback (field-level errors when possible).
- Consider doing basic validation locally to reduce round trips.

**Gotchas**
- A validation error might be caused by **schema drift** (client is older than server expectations).
  - If you see a spike after a server release, treat as compatibility incident, not user error.

---

### 5) Domain / Business Rule Errors
Examples:
- “Insufficient funds”
- “Seat no longer available”
- “Cannot cancel after departure”

**Mobile handling**
- Don’t auto-retry.
- Present a clear, user-readable message with recovery options.
- Keep errors stable and code-based for i18n and analytics.

**Best practice: stable error codes**
```json
{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "That seat was just taken.",
    "details": { "flight_id": "f_123" }
  }
}
```

**Gotchas**
- Using free-form `message` as the contract is brittle.
- Domain errors often need a UX action (“choose another seat”) not a toast.

---

### 6) Concurrency / Conflict Errors
Examples:
- Optimistic locking failure
- Version mismatch
- Duplicate submission

**Mobile handling**
- `409 Conflict` often means: fetch latest + reconcile.
- For writes: enforce idempotency keys to prevent duplicates.
- Provide conflict UI only when needed; otherwise auto-merge.

**Gotchas**
- Conflicts increase with offline-first behavior and multi-device usage.
- “Last write wins” can be acceptable for low-value fields, disastrous for money.

---

### 7) Rate Limiting / Throttling
Examples:
- `429 Too Many Requests`
- Server quota exceeded

**Mobile handling**
- Honor `Retry-After`.
- Apply **global backoff**, not per-endpoint backoff.
- Avoid retrying in tight loops; mobile radios are expensive.

**Gotchas**
- Background sync + retries can amplify traffic and *cause* rate limiting.
- If you ignore `Retry-After`, you can lock a user out of core flows.

---

### 8) Server / Dependency Failures
Examples:
- `500` internal error
- `502/503/504` upstream or overload

**Mobile handling**
- Retry with jitter (bounded).
- If this is a critical workflow (checkout), switch to “Try again” with clear state.
- Consider degrade paths: cached reads, limited mode, or queue writes.

**Gotchas**
- “Retry always” can turn an outage into a DDoS from your own clients.
  - Use retry budgets and circuit breaker logic on the client.

---

### 9) Schema Drift / Compatibility Errors
Examples:
- Client can’t decode new enum value
- Server rejects older payload fields
- Client assumes field exists but server stopped sending

**Mobile handling**
- Use tolerant decoders (ignore unknown fields).
- Model enums with `unknown` fallback.
- Use additive schema evolution on server to avoid breaking older clients.

**Gotchas**
- A decode crash is an outage on mobile.
- Treat compatibility issues as incidents; add telemetry around decode failures.

---

## A Practical Decision Table (Mobile Handling)

| Category | Auto Retry? | User Action? | Queue for Later? | Notes |
|---|---:|---:|---:|---|
| Transport (offline/timeout) | ✅ (bounded) | Sometimes | ✅ (writes) | Backoff + jitter |
| 401 token expired | ✅ (after refresh) | No | N/A | Coalesce refresh |
| 403 forbidden | ❌ | Yes | ❌ | Usually policy/permission |
| 422 validation | ❌ | Yes | ❌ | Show field-level errors |
| 409 conflict | Sometimes | Sometimes | Maybe | Reconcile or merge |
| 429 rate limit | ✅ (after delay) | No | Maybe | Honor Retry-After |
| 5xx server | ✅ (bounded) | Sometimes | Maybe | Circuit breaker |
| decode/schema | ❌ | Sometimes | ❌ | Compatibility incident |

---

## Best Practices for Error Payloads (Server → Mobile)

Mobile apps benefit from a consistent error envelope:

```json
{
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "Session expired.",
    "retryable": true,
    "retry_after_seconds": 10,
    "correlation_id": "c_8c12...",
    "details": { }
  }
}
```

Recommended fields:
- `code`: stable machine-readable identifier
- `message`: user-facing (optional) or developer-facing
- `retryable`: explicit hint (still client decides)
- `retry_after_seconds` or rely on HTTP `Retry-After`
- `correlation_id`: critical for support/debugging
- `details`: structured metadata (safe, non-sensitive)

**Gotchas**
- Don’t leak secrets or internal stack traces in `details`.
- Don’t change `code` semantics casually — it’s a contract.

---

## Mobile UX Patterns for Errors

### Separate “blocking” vs “non-blocking” errors
- Blocking: checkout/charge failed, auth revoked
- Non-blocking: feed refresh failed, analytics send failed

### Use “stale-but-usable” modes
If cached data exists:
- keep showing cached content
- show a subtle offline banner
- allow user-triggered refresh

### Prefer explicit retry over infinite spinners
Infinite spinners feel like “the app is broken.” Give:
- a clear state label (“Can’t connect”)
- a retry button
- an offline mode affordance

---

## Anti-Patterns (Very Common)

### 1) “Something went wrong” everywhere
Users can’t recover. Support can’t debug.

### 2) Retrying writes without idempotency
Creates duplicates and corrupted state.

### 3) Retrying immediately on 429/503
Creates a client-side thundering herd. Always backoff.

### 4) Treating all 4xx as fatal
401/409/429 are often recoverable.

### 5) Clearing local state on auth issues
Destroys drafts and queued work. Preserve user intent when possible.

---

## Observability: What to Log (Client + Server)

To debug production issues, log:

- `correlation_id` (from server) for every failure
- client version, platform, device OS
- endpoint + operation type (read vs write)
- error category (your taxonomy)
- retry count and final outcome
- decode failures and unknown enum occurrences

**Mobile-specific tip:** avoid logging PII and secrets; log codes and ids only.

---

## Testing Strategies

### Client-side tests
- Simulate offline, captive portal, and DNS failure
- Background the app mid-request and resume
- Force token expiry with concurrent requests (refresh storm test)
- Snapshot/contract tests for error envelopes

### Server-side tests
- Validate stable error codes and envelopes
- Ensure `Retry-After` semantics for 429
- Chaos testing for upstream 503/504 scenarios

---

## Additional Considerations

### How do you classify errors reliably on iOS?
- Map `URLError` codes to transport categories.
- Use HTTP status + error envelope to map domain/system categories.
- Treat decode failures as compatibility category with dedicated telemetry.

### How do you prevent retry storms?
- Exponential backoff + jitter
- Global retry budget (per session / per app run)
- Circuit breaker behavior for repeated 5xx or network failures
- Coalesced token refresh

### How do you handle partial failures?
- Separate read failures (degrade to cache) from write failures (queue + idempotency).
- Use per-operation criticality (checkout vs feed refresh).

### How do you design error codes?
- Stable, documented, non-leaky codes
- Grouped by domain (e.g., `AUTH_*`, `PAYMENT_*`, `BOOKING_*`)
- Backward compatible: add codes, don’t change meanings

---

## Key Concepts

> “On mobile, errors are product states. We classify failures into transport, auth, domain, conflict, throttling, server, and compatibility categories, each with explicit retry, UX, and telemetry rules. This prevents retry storms, preserves user intent offline, and makes production debugging tractable.”
