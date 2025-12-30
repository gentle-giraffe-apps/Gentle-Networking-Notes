
# Rate Limiting & Throttling for Mobile Systems

## Why Mobile Clients Trigger Limits Faster Than You Expect

Mobile apps are noisy by nature:

- Automatic retries
- Background refresh
- Push-triggered sync
- Token refresh storms
- App resume bursts

Without defensive throttling, mobile clients can DDoS your own backend.

---

## Types of Limits

| Limit Type | Scope | Purpose |
|-----------|-------|---------|
| Per-IP | Network edge | Abuse prevention |
| Per-user | Auth layer | Protect accounts |
| Per-device | Mobile edge | Detect runaway clients |
| Global | Platform | System stability |
| Endpoint-specific | Hot paths | Protect critical flows |

---

## Server Signals

Servers must express limits clearly:

- `429 Too Many Requests`
- `Retry-After` header
- Optional error envelope fields:
  - `retryable`
  - `retry_after_seconds`
  - `limit_scope`

---

## Client Behavior Rules

### Honor Retry-After Always

Never retry before the server’s cooldown expires.

### Use Global Backoff

If one endpoint is throttled, pause *all* non-critical requests for that scope.

### Apply Jitter

Backoff schedules should be randomized to avoid thundering herds.

---

## Mobile Gotchas

- iOS background tasks may re-fire immediately after suspension.
- Foreground resume often triggers multiple requests simultaneously.
- Parallel token refresh + data refresh amplifies load.

Mitigation: funnel all requests through a centralized scheduler.

---

## Battery-Aware Throttling

Every network call costs power.

Best practices:

- Stop background sync in Low Power Mode.
- Defer non-critical retries until charging or Wi-Fi.
- Maintain per-session retry budgets.

---

## Graceful Degradation

When rate-limited:

- Keep cached data visible.
- Surface subtle UI (“Updating paused due to server load”).
- Offer user-initiated retry only after cooldown.

---

## Anti-Patterns

| Anti-Pattern | Impact |
|-------------|--------|
| Retrying immediately on 429 | Escalates outage |
| Per-endpoint backoff only | Limits ineffective |
| Hiding throttling from UX | Confusing failures |
| No telemetry on limits | Impossible debugging |

---

## Observability

Log:

- limit scope
- retry_after
- request category
- app state (foreground/background)
- battery state

---

## Additional Considerations

### How do you prevent refresh storms?
- Single-flight token refresh
- Queue dependent requests
- Fail fast if refresh is already running

### How do you cap runaway clients?
- Per-device rate keys
- Server-side anomaly detection
- Kill-switch config flags

---

## Key Concepts

> “Mobile clients amplify traffic through retries and background execution. We treat 429 as a global backoff signal, honor Retry-After, and throttle aggressively when in low power or background states to protect both backend and battery.”
