# Case Study: Flight Booking System API

**Audience:** Senior / Staff mobile engineers
**Goal:** Design a production-grade flight booking API that survives mobile reality — applying the patterns from fundamentals through testing.

---

## System Overview

A flight booking system involves high-stakes, money-moving workflows where failures are visible and costly:

| Flow | Criticality | Failure Cost |
|------|-------------|--------------|
| Search | Medium | User frustration |
| Seat selection | High | Lost sale |
| Payment | Critical | Double charge / lost revenue |
| Check-in | High | Missed flight |

This case study walks through API design decisions, applying mobile-first patterns at each layer.

---

## Domain Resources

### Core Entities

```
Flight         → Scheduled service (immutable reference data)
FlightInstance → Specific departure (seat availability changes)
Booking        → User's reservation (stateful, versioned)
Quote          → Price snapshot (expires, never trusted from client)
BoardingPass   → Check-in artifact (offline-capable)
```

### Resource Relationships

```
User
 └─ Bookings[]
     ├─ FlightInstance
     ├─ Passengers[]
     ├─ Seats[]
     ├─ Quote (at time of purchase)
     └─ BoardingPasses[]
```

---

## API Shape

### Flight Search

```http
GET /flights/search?origin=SFO&destination=JFK&date=2025-03-15&passengers=2
```

Response:

```json
{
  "results": [
    {
      "flight_id": "fl_8a7c",
      "flight_number": "UA123",
      "departure": {
        "airport_code": "SFO",
        "scheduled_at": "2025-03-15T08:30:00Z"
      },
      "arrival": {
        "airport_code": "JFK",
        "scheduled_at": "2025-03-15T17:15:00Z"
      },
      "status_code": "SCHEDULED",
      "status_label": "On time",
      "reference_price": {
        "amount_minor": 42900,
        "currency": "USD"
      },
      "seats_available": true,
      "cabin_classes": ["ECONOMY", "BUSINESS"]
    }
  ],
  "search_id": "srch_f912",
  "expires_at": "2025-03-14T19:30:00Z",
  "next_cursor": null,
  "consistency": {
    "as_of": "2025-03-14T18:30:00Z",
    "ttl_seconds": 300
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `status_code` + `status_label` | Stable identifier for logic, label for display (resource modeling) |
| `reference_price` in minor units | ISO 4217 compliance, no floating point |
| `search_id` + `expires_at` | Search results are a snapshot; prices change |
| `consistency.as_of` | Client knows data freshness for cache decisions |
| `seats_available` boolean | Sufficient for search; detail comes later |

**Gotcha:** Never show exact seat counts in search results. This creates write hotspots as users race to book.

---

### Flight Details & Seat Map

```http
GET /flights/{flight_id}/seats?cabin=ECONOMY
```

Response:

```json
{
  "flight_id": "fl_8a7c",
  "cabin_class": "ECONOMY",
  "seats": [
    {
      "seat_id": "seat_12a",
      "row": 12,
      "position": "A",
      "available": true,
      "features": ["WINDOW", "EXIT_ROW"],
      "price_delta": {
        "amount_minor": 2500,
        "currency": "USD"
      }
    }
  ],
  "hold_ttl_seconds": 300,
  "etag": "seats-v47"
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `seat_id` as opaque string | Future-proof; don't encode row+position in ID |
| `features` array | Additive; new features don't break old clients |
| `price_delta` | Upsell pricing separate from base fare |
| `hold_ttl_seconds` | Communicates how long a selected seat can be held |
| `etag` | Cache invalidation for polling during selection |

---

### Seat Hold (Optimistic Lock)

Before checkout, the client requests a temporary hold:

```http
POST /flights/{flight_id}/seats/{seat_id}/hold
Headers:
  Idempotency-Key: 9f2e3d4c-...
Body:
{
  "passenger_index": 0
}
```

Response (success):

```json
{
  "hold_id": "hold_7a8b",
  "seat_id": "seat_12a",
  "expires_at": "2025-03-14T18:35:00Z",
  "state": "HELD"
}
```

Response (conflict):

```json
{
  "error": {
    "code": "SEAT_UNAVAILABLE",
    "message": "This seat was just taken.",
    "retryable": false,
    "details": {
      "seat_id": "seat_12a",
      "suggested_alternatives": ["seat_12b", "seat_12c"]
    }
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| Explicit hold endpoint | Separates browsing from intent; prevents phantom inventory |
| `Idempotency-Key` | User may tap twice; retry must not create duplicate holds |
| `expires_at` | Client shows countdown; graceful expiry UX |
| `suggested_alternatives` | Domain-aware error recovery |

**Gotcha:** Holds are write hotspots. Rate limit aggressively per-user to prevent inventory gaming.

---

### Quote Creation

Never let the client submit prices. Create a server-side quote:

```http
POST /quotes
Headers:
  Idempotency-Key: a1b2c3d4-...
Body:
{
  "flight_id": "fl_8a7c",
  "passengers": [
    { "type": "ADULT", "seat_id": "seat_12a" },
    { "type": "ADULT", "seat_id": "seat_12b" }
  ],
  "ancillaries": ["PRIORITY_BOARDING"]
}
```

Response:

```json
{
  "quote_id": "qt_9c8d",
  "line_items": [
    { "description": "Base fare (x2)", "amount_minor": 85800, "currency": "USD" },
    { "description": "Seat 12A (Exit row)", "amount_minor": 2500, "currency": "USD" },
    { "description": "Seat 12B (Exit row)", "amount_minor": 2500, "currency": "USD" },
    { "description": "Priority boarding", "amount_minor": 1500, "currency": "USD" }
  ],
  "subtotal": { "amount_minor": 92300, "currency": "USD" },
  "taxes": [
    { "code": "US_EXCISE", "amount_minor": 6930, "currency": "USD" }
  ],
  "total": { "amount_minor": 99230, "currency": "USD" },
  "expires_at": "2025-03-14T18:40:00Z",
  "quote_version": 1
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| Server computes all pricing | Client cannot be trusted; taxes/FX change |
| `quote_id` | Charge references this, not raw amounts |
| `line_items` array | Transparent breakdown; additive for new item types |
| `taxes` separate | Jurisdiction-specific; may need itemization |
| `expires_at` | Prevents stale quote attacks |
| `quote_version` | Optimistic locking if quote needs refresh |

**Gotcha:** Quote expiry should be shorter than seat hold expiry. Otherwise users see "seat lost" after committing to price.

---

### Payment / Booking Creation

```http
POST /bookings
Headers:
  Idempotency-Key: x7y8z9-...
Body:
{
  "quote_id": "qt_9c8d",
  "payment_method_token": "pm_stripe_abc",
  "passengers": [
    {
      "first_name": "Jane",
      "last_name": "Doe",
      "date_of_birth": "1990-05-15",
      "document_type": "PASSPORT",
      "document_number": "X12345678"
    }
  ],
  "contact": {
    "email": "jane@example.com",
    "phone": "+14155551234"
  }
}
```

Response (success):

```json
{
  "booking_id": "bk_4e5f",
  "confirmation_code": "ABC123",
  "state": "CONFIRMED",
  "allowed_transitions": ["CANCELLED"],
  "flights": [...],
  "passengers": [...],
  "total_charged": { "amount_minor": 99230, "currency": "USD" },
  "created_at": "2025-03-14T18:38:00Z"
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `Idempotency-Key` required | Double-tap protection for payment |
| `quote_id` reference | Server re-validates quote; rejects if expired |
| `payment_method_token` | PCI compliance; client never handles card numbers |
| `state` + `allowed_transitions` | State machine, not boolean explosion |
| `confirmation_code` | Human-readable for support calls |

**Gotcha:** If quote expired mid-checkout, return:

```json
{
  "error": {
    "code": "QUOTE_EXPIRED",
    "message": "Prices have changed. Please review the updated quote.",
    "retryable": false,
    "details": {
      "new_quote_id": "qt_9c8e",
      "price_delta": { "amount_minor": 1200, "currency": "USD" }
    }
  }
}
```

Never silently charge a different amount.

---

### Booking Retrieval

```http
GET /bookings/{booking_id}
Headers:
  If-None-Match: "bk-v3"
```

Response:

```json
{
  "booking_id": "bk_4e5f",
  "confirmation_code": "ABC123",
  "state": "CONFIRMED",
  "allowed_transitions": ["CANCELLED", "CHECKED_IN"],
  "flights": [
    {
      "flight_id": "fl_8a7c",
      "status_code": "ON_TIME",
      "departure": {...},
      "gate": "B42",
      "gate_updated_at": "2025-03-15T06:00:00Z"
    }
  ],
  "passengers": [...],
  "check_in_opens_at": "2025-03-14T08:30:00Z",
  "etag": "bk-v4"
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `If-None-Match` support | Efficient polling without full payload |
| `gate_updated_at` | Client knows freshness of volatile data |
| `check_in_opens_at` | Enables local countdown without polling |
| `allowed_transitions` | UI can show/hide actions based on state |

---

### Check-In

```http
POST /bookings/{booking_id}/check-in
Headers:
  Idempotency-Key: c1d2e3-...
Body:
{
  "passenger_ids": ["pax_1", "pax_2"]
}
```

Response:

```json
{
  "booking_id": "bk_4e5f",
  "state": "CHECKED_IN",
  "boarding_passes": [
    {
      "pass_id": "bp_7a8b",
      "passenger_id": "pax_1",
      "barcode_data": "M1DOE/JANE...",
      "barcode_format": "AZTEC",
      "valid_from": "2025-03-15T05:30:00Z",
      "valid_until": "2025-03-15T09:00:00Z",
      "offline_token": "eyJhbGciOi..."
    }
  ]
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `barcode_data` + `barcode_format` | Client renders; format may vary by airline |
| `offline_token` | Signed JWT for offline boarding pass verification |
| Validity window | Prevents screenshot sharing abuse |

**Critical:** Boarding passes must work offline. The `offline_token` is a self-contained signed payload that airport scanners can verify without network.

---

## Offline-First Architecture

### Local State Model

```swift
struct LocalBooking: Codable {
    let bookingId: String
    let state: BookingState
    let flights: [LocalFlight]
    let boardingPasses: [LocalBoardingPass]
    let lastSyncedAt: Date
    let pendingMutations: [PendingMutation]
}

struct PendingMutation: Codable {
    let id: UUID
    let type: MutationType  // .checkIn, .seatChange, .cancel
    let payload: Data
    let createdAt: Date
    let retryCount: Int
    let idempotencyKey: String
}
```

### Sync Strategy

| Data Type | Sync Window | Priority |
|-----------|-------------|----------|
| Upcoming bookings | Always fresh | Critical |
| Past bookings | 30 days | Low |
| Boarding passes | Until flight departure + 24h | Critical |
| Flight status | 5 min TTL when within 24h of departure | High |

### Mutation Queue

```
User Action → Local Mutation → Queue → Network
                    ↓
              Optimistic UI
```

Check-in should work even if queued:

1. User taps "Check In"
2. App writes `PendingMutation` to disk
3. UI shows "Checked In (syncing...)"
4. Background sync attempts with idempotency key
5. On success: update `lastSyncedAt`, remove mutation
6. On conflict: surface "Check-in failed" with reason

---

## Error Taxonomy (Flight-Specific)

### Transport Errors

| Error | Retry? | UX |
|-------|--------|-----|
| Timeout | Yes (backoff) | "Connecting..." |
| DNS failure | Yes | "No connection" |
| TLS error | Yes | "Secure connection failed" |

### Domain Errors

| Code | Meaning | Retry? | UX |
|------|---------|--------|-----|
| `SEAT_UNAVAILABLE` | Race condition | No | Show alternatives |
| `QUOTE_EXPIRED` | Price changed | No | Refresh quote |
| `CHECK_IN_NOT_OPEN` | Too early | No | Show countdown |
| `BOOKING_CANCELLED` | State conflict | No | Refresh booking |
| `PASSENGER_DOCUMENT_INVALID` | Validation | No | Edit form |
| `PAYMENT_DECLINED` | Card issue | No | Try another method |

### System Errors

| Code | Meaning | Retry? | UX |
|------|---------|--------|-----|
| `SERVICE_UNAVAILABLE` | Backend down | Yes (backoff) | "Try again shortly" |
| `RATE_LIMITED` | Throttled | Yes (honor Retry-After) | Silent backoff |
| `MAINTENANCE_WINDOW` | Planned | No | Show message + ETA |

---

## Caching Strategy

### Layer Assignment

| Data | Layer | TTL |
|------|-------|-----|
| Airport/airline reference | CDN | 24h |
| Flight search results | Client disk | 5 min |
| Seat map | Client memory | 30 sec |
| User bookings | Client disk | Until sync |
| Boarding passes | Client disk | Until expiry |

### Cache Invalidation Triggers

| Event | Invalidate |
|-------|------------|
| Booking created | User bookings list |
| Check-in complete | Booking detail, boarding passes |
| Flight status change | Booking detail (push-triggered) |
| App foreground | Bookings within 24h of departure |

---

## Rate Limiting

| Endpoint | Limit | Scope |
|----------|-------|-------|
| `/flights/search` | 30/min | Per-user |
| `/seats/{id}/hold` | 10/min | Per-user |
| `/bookings` (POST) | 5/min | Per-user |
| `/check-in` | 3/min | Per-booking |

**Gotcha:** Seat holds are abuse vectors. Malicious users can hold inventory without purchasing. Implement:

- Short hold TTL (5 min)
- Maximum concurrent holds per user (3)
- Progressive backoff on repeated hold failures

---

## Circuit Breaker Configuration

| Service | Open Threshold | Half-Open Probes |
|---------|---------------|------------------|
| Search API | 50% failures over 30s | 1 per 10s |
| Booking API | 30% failures over 30s | 1 per 15s |
| Payment Gateway | 20% failures over 60s | 1 per 30s |

When payment circuit opens:

- Block new booking attempts
- Surface: "Checkout temporarily unavailable"
- Preserve cart state for retry

---

## Testing Strategy

### Contract Tests

```yaml
# booking_contract.yaml
- request:
    method: GET
    path: /bookings/{booking_id}
  response:
    status: 200
    body:
      booking_id: string
      state: enum[CONFIRMED, CHECKED_IN, CANCELLED, COMPLETED]
      allowed_transitions: array[string]
```

Validate on every backend deploy.

### Chaos Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| Payment timeout after charge | Idempotent retry succeeds |
| Seat hold expires mid-checkout | Quote rejected, user re-selects |
| Check-in API down | Queued mutation, offline pass unavailable |
| CDN fails | Placeholder images, cached reference data |

### Version Skew Matrix

| Client | Server | Must Work |
|--------|--------|-----------|
| N-2 | Current | Search, booking, check-in |
| N-1 | Current | All features |
| N | Current | All features + new capabilities |

---

## Observability

### Client Telemetry

```swift
struct BookingFlowEvent {
    let step: FlowStep  // .search, .selectSeat, .quote, .pay, .confirm
    let durationMs: Int
    let outcome: Outcome  // .success, .error, .abandoned
    let errorCode: String?
    let networkPath: NetworkPath
    let batteryState: BatteryState
    let retryCount: Int
}
```

### Key Metrics

| Metric | Alert Threshold |
|--------|-----------------|
| Booking completion rate | < 85% |
| Check-in success rate | < 95% |
| Payment retry rate | > 5% |
| Offline boarding pass usage | Track only |
| Search p95 latency | > 2s |

---

## Common Gotchas Summary

| Gotcha | Mitigation |
|--------|------------|
| Double payment on retry | Idempotency keys on all charges |
| Stale prices at checkout | Short-lived quotes with expiry |
| Seat races | Hold pattern with TTL |
| Boarding pass without network | Signed offline tokens |
| Quote expires before seat hold | Quote TTL < Hold TTL |
| Inventory gaming via holds | Rate limit + max concurrent holds |
| Flight status stale | Push notifications + short TTL polling |
| Check-in fails offline | Queue mutations, block boarding pass generation |

---

## Key Concepts

> "A flight booking system is a distributed transaction across inventory, payments, and fulfillment. Mobile adds offline requirements, retry storms, and UI responsiveness constraints. We protect against double-charges with idempotency, against stale prices with expiring quotes, against inventory races with hold patterns, and against network loss with offline-capable boarding passes. Every flow must degrade gracefully — a user stuck at the airport gate with no signal must still board their flight."
