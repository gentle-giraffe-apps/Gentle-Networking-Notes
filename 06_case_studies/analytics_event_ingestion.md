# Case Study: Analytics Event Ingestion

**Audience:** Senior / Staff mobile engineers
**Goal:** Design a production-grade analytics pipeline optimized for mobile constraints — where battery efficiency, offline resilience, and cross-platform consistency matter more than guaranteed delivery.

---

## How Analytics Differs from Other Systems

Analytics has fundamentally different tradeoffs than transactional APIs:

| Concern | Transactional API | Analytics |
|---------|-------------------|-----------|
| Delivery guarantee | Exactly-once critical | At-least-once acceptable |
| Individual event value | High (money, state) | Low (signal in aggregate) |
| Latency requirement | Sub-second | Minutes to hours |
| Retry strategy | Aggressive | Best-effort with budget |
| Offline behavior | Queue and replay all | Queue with eviction |
| Batching | Usually not | Essential |

**The core insight:** Analytics events are cheap individually but expensive in aggregate (battery, bandwidth). Optimize for throughput and efficiency, not reliability.

---

## Cross-Platform Event Schema

### The Alignment Problem

A "button_clicked" event must mean the same thing on iOS, Android, and web. Without a shared spec:

| Platform | Implementation | Problem |
|----------|----------------|---------|
| iOS | `buttonTapped` | Different naming |
| Android | `button_click` | Different casing |
| Web | `click_button` | Different word order |

This creates analytics chaos — dashboards can't aggregate, A/B tests are invalid.

### Shared Event Specification

Define events in a platform-agnostic spec:

```yaml
# events/checkout_started.yaml
event_name: checkout_started
description: User began the checkout flow
category: conversion
platforms: [ios, android, web]

properties:
  cart_value:
    type: integer
    description: Cart total in minor currency units
    required: true
  item_count:
    type: integer
    description: Number of items in cart
    required: true
  currency:
    type: string
    description: ISO 4217 currency code
    required: true
  source_screen:
    type: string
    enum: [cart, product_detail, quick_buy]
    required: true
  experiment_group:
    type: string
    description: Active A/B test variant
    required: false

context:
  - session_id
  - user_id
  - device_id
  - app_version
  - platform
```

### Code Generation

Generate type-safe event classes from specs:

```swift
// Generated: CheckoutStartedEvent.swift
struct CheckoutStartedEvent: AnalyticsEvent {
    static let eventName = "checkout_started"
    static let category = EventCategory.conversion

    let cartValue: Int
    let itemCount: Int
    let currency: String
    let sourceScreen: SourceScreen

    enum SourceScreen: String, Codable {
        case cart
        case productDetail = "product_detail"
        case quickBuy = "quick_buy"
    }
}
```

```kotlin
// Generated: CheckoutStartedEvent.kt
data class CheckoutStartedEvent(
    val cartValue: Int,
    val itemCount: Int,
    val currency: String,
    val sourceScreen: SourceScreen
) : AnalyticsEvent {
    override val eventName = "checkout_started"

    enum class SourceScreen {
        @SerializedName("cart") CART,
        @SerializedName("product_detail") PRODUCT_DETAIL,
        @SerializedName("quick_buy") QUICK_BUY
    }
}
```

**Benefits:**
- Compile-time validation
- Consistent naming across platforms
- Auto-generated documentation
- Breaking changes caught in CI

---

## Event Envelope Structure

### Standard Envelope

Every event shares common context:

```json
{
  "event_name": "checkout_started",
  "event_id": "evt_a1b2c3",
  "timestamp": "2025-03-14T18:32:15.123Z",
  "client_timestamp": "2025-03-14T18:32:14.987Z",
  "context": {
    "session_id": "sess_xyz",
    "user_id": "usr_123",
    "anonymous_id": "anon_456",
    "device_id": "dev_789",
    "platform": "ios",
    "app_version": "2.5.1",
    "app_build": "1847",
    "os_version": "17.2",
    "device_model": "iPhone15,2",
    "locale": "en-US",
    "timezone": "America/Los_Angeles",
    "network_type": "wifi",
    "battery_level": 0.72,
    "battery_state": "unplugged",
    "screen_width": 393,
    "screen_height": 852
  },
  "properties": {
    "cart_value": 4299,
    "item_count": 3,
    "currency": "USD",
    "source_screen": "cart"
  }
}
```

**Design decisions:**

| Field | Rationale |
|-------|-----------|
| `event_id` | Deduplication on server |
| `timestamp` (server) | Assigned at ingestion; authoritative |
| `client_timestamp` | Preserved for latency analysis |
| `anonymous_id` | Pre-login identity for stitching |
| `network_type` | Correlate behavior with connectivity |
| `battery_*` | Understand power-constrained behavior |

### Timestamp Handling

| Timestamp | Source | Use |
|-----------|--------|-----|
| `client_timestamp` | Device clock | Event ordering within session |
| `timestamp` | Server receipt | Cross-device ordering, dashboards |
| `sent_at` | Batch envelope | Latency measurement |

**Gotcha:** Device clocks are unreliable. Users set wrong dates, timezones drift, NTP fails. Always use server timestamp for analytics queries; preserve client timestamp for debugging.

---

## Batching Strategy

### Why Batching Is Non-Negotiable

| Approach | Radio Wakeups | Battery Impact |
|----------|---------------|----------------|
| Send immediately | 1 per event | Catastrophic |
| Batch every 30 sec | ~2/min | High |
| Batch on thresholds | ~0.5/min | Acceptable |

### Multi-Trigger Batching

Flush the queue when ANY condition is met:

```swift
struct BatchConfig {
    let maxEvents: Int = 20           // Count threshold
    let maxBytes: Int = 50_000        // Size threshold (~50KB)
    let maxAgeSeconds: TimeInterval = 60  // Time threshold
    let flushOnBackground: Bool = true
    let flushOnSignificantEvent: Bool = true  // e.g., purchase
}
```

### Flush Triggers

| Trigger | When | Rationale |
|---------|------|-----------|
| Event count | ≥ 20 events | Prevent unbounded growth |
| Payload size | ≥ 50KB | HTTP efficiency |
| Time elapsed | ≥ 60 seconds | Freshness for dashboards |
| App backgrounding | Always | May not wake again |
| Significant event | Purchase, signup | Business-critical, send now |
| Network restored | After offline | Drain accumulated queue |

### Batch Request

```http
POST /v1/batch
Headers:
  Content-Type: application/json
  Content-Encoding: gzip
  X-Client-Version: 2.5.1
  X-Batch-Id: batch_d4e5f6
Body:
{
  "batch_id": "batch_d4e5f6",
  "sent_at": "2025-03-14T18:33:00Z",
  "events": [
    { "event_name": "screen_viewed", ... },
    { "event_name": "button_clicked", ... },
    { "event_name": "checkout_started", ... }
  ]
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| gzip compression | 70-90% size reduction typical |
| `batch_id` | Server-side deduplication of retried batches |
| `sent_at` | Measure client → server latency |

---

## Offline Queue Management

### Queue Architecture

```
┌─────────────────────────────────────────────────┐
│                   Event Queue                   │
│  ┌───────────────────────────────────────────┐  │
│  │  Ring Buffer (Memory)     [latest 100]    │  │
│  └───────────────────────────────────────────┘  │
│                      ↓ overflow                 │
│  ┌───────────────────────────────────────────┐  │
│  │  SQLite / File Queue      [max 10,000]    │  │
│  └───────────────────────────────────────────┘  │
│                      ↓ overflow                 │
│  ┌───────────────────────────────────────────┐  │
│  │  Eviction (oldest first, sample if full)  │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Eviction Policy

When queue exceeds limits:

| Strategy | Behavior | Use When |
|----------|----------|----------|
| Drop oldest | FIFO eviction | Default |
| Sample | Keep 1 in N | High-volume events |
| Priority-based | Keep conversions, drop views | Business-critical tracking |

```swift
struct QueueConfig {
    let memoryLimit: Int = 100
    let diskLimit: Int = 10_000
    let maxAgeHours: Int = 72  // Drop events older than 3 days
    let evictionStrategy: EvictionStrategy = .dropOldest
}

enum EvictionStrategy {
    case dropOldest
    case sampleAtRate(Double)  // e.g., 0.1 = keep 10%
    case priorityBased(keep: [EventCategory])
}
```

**Gotcha:** A 3-day-old event is probably worthless for real-time dashboards. Set aggressive TTLs; stale data creates confusion.

---

## Battery-Aware Transmission

### Adaptive Flush Strategy

```swift
func shouldFlushNow(queue: EventQueue, context: DeviceContext) -> FlushDecision {
    // Always flush significant events
    if queue.containsSignificantEvent {
        return .flushNow
    }

    // Low power mode: batch aggressively
    if context.isLowPowerMode {
        return queue.count >= 50 ? .flushNow : .defer
    }

    // On WiFi + charging: flush freely
    if context.isOnWifi && context.isCharging {
        return queue.count >= 10 ? .flushNow : .defer
    }

    // On cellular: batch more
    if context.isOnCellular {
        return queue.count >= 30 ? .flushNow : .defer
    }

    // Offline: don't attempt
    if !context.hasConnectivity {
        return .defer
    }

    // Default thresholds
    return queue.count >= 20 ? .flushNow : .defer
}
```

### Context-Aware Batching

| Condition | Max Events | Max Age |
|-----------|------------|---------|
| WiFi + Charging | 10 | 30 sec |
| WiFi + Battery | 20 | 60 sec |
| Cellular | 30 | 120 sec |
| Low Power Mode | 50 | 300 sec |
| Background | Flush all | Immediate |

---

## Sampling Strategies

### When to Sample

High-frequency events can overwhelm:

| Event | Frequency | Sampling Strategy |
|-------|-----------|-------------------|
| `screen_viewed` | Every navigation | 100% (low volume) |
| `scroll_position` | Every 100ms | 1% or disable |
| `network_request` | Every API call | 10% |
| `render_performance` | Every frame | 0.1% |
| `purchase_completed` | Rare | 100% (never sample) |

### Client-Side Sampling

```swift
struct SamplingConfig {
    let defaultRate: Double = 1.0  // 100%
    let overrides: [String: Double] = [
        "scroll_position": 0.01,     // 1%
        "network_request": 0.10,     // 10%
        "render_performance": 0.001  // 0.1%
    ]
}

func shouldSample(eventName: String) -> Bool {
    let rate = config.overrides[eventName] ?? config.defaultRate
    return Double.random(in: 0...1) < rate
}
```

### Server-Side Adjustment

Include sampling rate in events for proper aggregation:

```json
{
  "event_name": "scroll_position",
  "sampling_rate": 0.01,
  "properties": { ... }
}
```

Dashboard multiplies counts by `1/sampling_rate` for estimates.

---

## Retry Semantics

### Analytics-Appropriate Retry

Unlike transactional systems, analytics should **not** retry aggressively:

| Failure | Transactional | Analytics |
|---------|---------------|-----------|
| Timeout | Retry immediately | Retry once, then queue |
| 5xx | Retry with backoff | Retry once, then defer |
| 4xx | Surface error | Log and drop |
| Network loss | Queue indefinitely | Queue with eviction |

### Retry Budget

```swift
struct RetryConfig {
    let maxRetries: Int = 2
    let retryDelaySeconds: [Int] = [5, 30]  // Backoff schedule
    let maxQueuedBatches: Int = 50
    let dropAfterHours: Int = 72
}

func handleFailure(batch: Batch, error: Error, attempt: Int) {
    if attempt >= config.maxRetries {
        // Move to cold storage or drop
        if batch.age < config.dropAfterHours.hours {
            persistToDisk(batch)
        } else {
            drop(batch, reason: .maxRetriesExceeded)
        }
        return
    }

    // Schedule retry
    scheduleRetry(batch, delay: config.retryDelaySeconds[attempt])
}
```

### Deduplication

Server must handle duplicate batches from retries:

```sql
-- Idempotent insert
INSERT INTO events (event_id, event_name, timestamp, properties)
VALUES (?, ?, ?, ?)
ON CONFLICT (event_id) DO NOTHING;
```

---

## Session Management

### Session Definition

```swift
struct SessionConfig {
    let timeoutMinutes: Int = 30
    let maxDurationHours: Int = 24
    let extendOnActivity: Bool = true
}
```

### Session Lifecycle

```
App Launch
    │
    ▼
┌─────────────────┐
│ Check Last Activity │
└────────┬────────┘
         │
    ┌────┴────┐
    │ > 30min │──────► New Session
    │  ago?   │
    └────┬────┘
         │ No
         ▼
   Continue Session
         │
         ▼
┌─────────────────┐
│ Session > 24h?  │──────► End Session, Start New
└────────┬────────┘
         │ No
         ▼
   Continue Session
```

### Session Events

```json
// Auto-generated on session start
{
  "event_name": "session_started",
  "properties": {
    "session_id": "sess_new",
    "previous_session_id": "sess_old",
    "time_since_last_session_seconds": 3847,
    "is_first_session": false,
    "entry_point": "push_notification"
  }
}

// Auto-generated on session end (best effort)
{
  "event_name": "session_ended",
  "properties": {
    "session_id": "sess_xyz",
    "duration_seconds": 847,
    "event_count": 23,
    "screens_viewed": 7
  }
}
```

**Gotcha:** `session_ended` is unreliable on mobile. Apps are killed without warning. Calculate session duration server-side from last event timestamp.

---

## Identity Management

### The Identity Problem

```
Anonymous User → Signs Up → Logs In → Logs Out → Different Device
     │              │           │         │            │
  anon_123      anon_123    usr_456   anon_789     usr_456
```

### Identity Stitching

```json
// Before login
{
  "context": {
    "anonymous_id": "anon_123",
    "user_id": null
  }
}

// Identify call on login
{
  "event_name": "$identify",
  "properties": {
    "anonymous_id": "anon_123",
    "user_id": "usr_456"
  }
}

// After login
{
  "context": {
    "anonymous_id": "anon_123",
    "user_id": "usr_456"
  }
}
```

### Alias for Account Merging

When anonymous activity should be attributed to known user:

```json
{
  "event_name": "$alias",
  "properties": {
    "previous_id": "anon_123",
    "new_id": "usr_456"
  }
}
```

Server merges event history from `anon_123` into `usr_456`.

---

## Privacy and Compliance

### Data Minimization

| Data | Collect? | Rationale |
|------|----------|-----------|
| Screen name | Yes | Essential for funnel analysis |
| Button label | Yes | UX optimization |
| Exact GPS | No | City-level sufficient |
| IP address | Hash only | Geo lookup, then discard |
| User input text | No | PII risk |
| Device ID | Resettable only | Privacy regulations |

### Consent Tracking

```swift
struct ConsentState {
    let analyticsConsent: Bool
    let advertisingConsent: Bool
    let consentTimestamp: Date
    let consentVersion: String  // Track which policy version
}

func track(event: AnalyticsEvent) {
    guard consentState.analyticsConsent else {
        return  // Don't collect without consent
    }

    // Include consent context
    var enrichedEvent = event
    enrichedEvent.context.consentVersion = consentState.consentVersion
    queue.enqueue(enrichedEvent)
}
```

### Right to Deletion

Support user data deletion requests:

```http
DELETE /v1/users/{user_id}/data
Headers:
  Authorization: Bearer {admin_token}
Body:
{
  "deletion_type": "FULL",
  "reason": "GDPR_REQUEST",
  "request_id": "del_789"
}
```

---

## Validation and Quality

### Client-Side Validation

```swift
func validate(event: AnalyticsEvent) -> ValidationResult {
    // Required properties
    for required in event.schema.requiredProperties {
        guard event.properties[required] != nil else {
            return .invalid(reason: "Missing required: \(required)")
        }
    }

    // Type checking
    for (key, value) in event.properties {
        guard let expectedType = event.schema.propertyTypes[key] else {
            return .invalid(reason: "Unknown property: \(key)")
        }
        guard typeMatches(value, expected: expectedType) else {
            return .invalid(reason: "Type mismatch: \(key)")
        }
    }

    // Enum validation
    for (key, value) in event.properties {
        if let allowedValues = event.schema.enumValues[key] {
            guard allowedValues.contains(value as? String ?? "") else {
                return .invalid(reason: "Invalid enum: \(key)=\(value)")
            }
        }
    }

    return .valid
}
```

### Schema Registry

```http
GET /v1/schemas/{event_name}?version=latest
```

```json
{
  "event_name": "checkout_started",
  "version": "2.1.0",
  "properties": {
    "cart_value": { "type": "integer", "required": true },
    "item_count": { "type": "integer", "required": true },
    "currency": { "type": "string", "required": true },
    "source_screen": {
      "type": "string",
      "enum": ["cart", "product_detail", "quick_buy"],
      "required": true
    }
  },
  "deprecated_properties": ["cart_total"],
  "added_in_version": "2.0.0"
}
```

### Quality Metrics

| Metric | Target | Alert |
|--------|--------|-------|
| Schema validation pass rate | > 99% | < 95% |
| Events with user_id | > 80% (post-login) | < 60% |
| Duplicate event rate | < 1% | > 5% |
| Avg batch latency | < 5 min | > 30 min |
| Queue overflow rate | < 0.1% | > 1% |

---

## SDK Architecture

### Layered Design

```
┌─────────────────────────────────────────────────┐
│              Public API Layer                   │
│  Analytics.track(event), Analytics.identify()   │
└─────────────────────────────────────────────────┘
                      │
┌─────────────────────────────────────────────────┐
│            Enrichment Layer                     │
│  Add context, validate schema, apply sampling   │
└─────────────────────────────────────────────────┘
                      │
┌─────────────────────────────────────────────────┐
│              Queue Layer                        │
│  Memory buffer → Disk persistence → Batching    │
└─────────────────────────────────────────────────┘
                      │
┌─────────────────────────────────────────────────┐
│            Transport Layer                      │
│  HTTP client, retry logic, compression          │
└─────────────────────────────────────────────────┘
```

### Thread Safety

```swift
actor AnalyticsQueue {
    private var memoryBuffer: [AnalyticsEvent] = []
    private let diskQueue: DiskQueue

    func enqueue(_ event: AnalyticsEvent) {
        memoryBuffer.append(event)
        if memoryBuffer.count >= config.memoryLimit {
            await flushToDisk()
        }
    }

    func flush() async -> [AnalyticsEvent] {
        let events = memoryBuffer + (await diskQueue.readAll())
        memoryBuffer.removeAll()
        await diskQueue.clear()
        return events
    }
}
```

---

## Testing Strategy

### Unit Tests

```swift
func testSamplingRespected() {
    let config = SamplingConfig(overrides: ["scroll": 0.0])
    let analytics = Analytics(config: config)

    analytics.track(ScrollEvent(position: 100))

    XCTAssertEqual(analytics.queue.count, 0)  // Dropped
}

func testBatchFlushOnThreshold() {
    let config = BatchConfig(maxEvents: 5)
    let analytics = Analytics(config: config)

    for i in 1...5 {
        analytics.track(TestEvent(index: i))
    }

    XCTAssertTrue(mockTransport.sendCalled)
    XCTAssertEqual(mockTransport.lastBatch?.events.count, 5)
}
```

### Integration Tests

| Scenario | Validation |
|----------|------------|
| App backgrounded with queued events | Events persisted to disk |
| Network restored after offline | Queued batches sent |
| Schema validation failure | Event logged, not sent |
| Clock skew simulation | Server timestamp used in queries |

### End-to-End Validation

```yaml
# CI pipeline validates event flow
- name: Track test event
  run: |
    curl -X POST $ANALYTICS_URL/v1/batch \
      -d '{"events": [{"event_name": "ci_test", "event_id": "test_123"}]}'

- name: Verify in data warehouse
  run: |
    result=$(query "SELECT * FROM events WHERE event_id = 'test_123'")
    assert_not_empty "$result"
```

---

## Common Gotchas Summary

| Gotcha | Mitigation |
|--------|------------|
| Device clock wrong | Use server timestamp for queries |
| Session end never fires | Calculate duration from last event |
| High-volume events drain battery | Sampling + aggressive batching |
| Schema drift across platforms | Code-generate from shared spec |
| Retry storm on backend outage | Limited retry budget, exponential backoff |
| Queue grows unbounded offline | Eviction policy with TTL |
| Duplicate events from retries | `event_id` deduplication server-side |
| PII in event properties | Validation layer blocks sensitive fields |
| Anonymous → logged-in gap | Identity stitching with `$alias` |
| Consent withdrawn mid-session | Check consent before every enqueue |

---

## Analytics vs Other Systems

| Aspect | Analytics Approach | Why Different |
|--------|-------------------|---------------|
| Delivery | At-least-once, lossy OK | Individual events low value |
| Latency | Minutes acceptable | Dashboards tolerate delay |
| Retry | 1-2 attempts max | Battery > completeness |
| Queue | Bounded with eviction | Prevent unbounded growth |
| Offline | Hours of storage, then drop | Stale data confuses analysis |
| Batching | Always, 20-50 events | Radio wakeup is expensive |
| Compression | Always gzip | 70-90% savings typical |
| Sampling | Per-event-type rates | High-frequency events sampled |

---

## Key Concepts

> "Analytics is a high-volume, low-value-per-event pipeline where battery and bandwidth efficiency matter more than guaranteed delivery. We batch aggressively (20+ events), flush on multiple triggers (count, size, time, backgrounding), sample high-frequency events, and evict stale data rather than retry indefinitely. Cross-platform consistency comes from a shared event specification with code-generated type-safe wrappers. Device timestamps are preserved but never trusted — server receipt time is authoritative. The goal is not to capture every event perfectly; it's to capture enough signal to make decisions while respecting the user's battery and the constraints of mobile execution."
