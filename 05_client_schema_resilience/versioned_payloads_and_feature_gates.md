# Versioned Payloads and Feature Gates: Shipping Without Breaking Clients

Modern mobile systems fail less often due to outages and more often due to **incompatible evolution**.
This article explains how to design **versioned payloads + feature gating** so iOS, Android, Web, and backend services
can evolve independently without crash spikes, silent corruption, or emergency hotfixes.

---

## Why this matters

Backend changes almost never land simultaneously across:

- iOS App Store review cycles
- Android staged rollouts
- Web instant deploys
- Feature-flagged backend services
- CDN-cached API responses

If your system is not **version-aware and gateable from day zero**, your only safety mechanism becomes fear.

---

## The Core Contract

> The backend must never assume the client is current.  
> The client must never assume the backend is consistent.

---

## Versioning Strategies

| Strategy | When to use |
|---------|-------------|
| `schemaVersion` in JSON body | DTO shape changes |
| HTTP `X-API-Version` header | Routing & gateway behavior |
| Feature flags in payload | UI / business logic control |
| Capability negotiation | Cross-platform feature alignment |

---

## Versioned Envelope Pattern

### Server payload

```json
{
  "schemaVersion": 2,
  "features": ["new_checkout", "flex_pricing"],
  "data": {
    "id": "order_123",
    "status": "completed"
  }
}
```

### Swift DTO

```swift
struct EnvelopeDTO<T: Decodable>: Decodable {
    let schemaVersion: Int
    let features: [String]
    let data: T
}
```

---

## Feature Gating on the Client

```swift
enum Feature: Hashable, Sendable {
    case newCheckout
    case flexPricing
    case unknown(String)

    init(raw: String) {
        switch raw {
        case "new_checkout": self = .newCheckout
        case "flex_pricing": self = .flexPricing
        default: self = .unknown(raw)
        }
    }
}

struct FeatureSet: Sendable {
    let enabled: Set<Feature>

    init(raw: [String]) {
        // MUST be map, not compactMap — preserve unknowns
        self.enabled = Set(raw.map { Feature(raw: $0) })
    }

    func isEnabled(_ feature: Feature) -> Bool {
        enabled.contains(feature)
    }

    var unknownFeatures: [String] {
        enabled.compactMap {
            if case .unknown(let raw) = $0 { return raw }
            return nil
        }
    }
}
```

### Usage

```swift
if features.isEnabled(.newCheckout) {
    showNewCheckout()
} else {
    showLegacyCheckout()
}
```

---

## Coordinating iOS, Android, and Web

All clients must:

- Ship with **forward-compatible decoding**
- Respect feature flags as *source of truth*
- Never hard-code server behavior assumptions

Create a shared contract document:

```
/contracts/features/new_checkout.md
```

Describing:
- Feature intent
- Fallback behavior
- Kill-switch semantics

---

## Backend Best Practices

### Drop features only after telemetry proves safety

1. Track `X-Client-Version` header.
2. Measure active client versions.
3. Only remove deprecated fields when <1% of traffic depends on them.
4. Add server-side alerts when legacy clients spike unexpectedly.

---

## iOS Version Reporting

```swift
var headers: [String: String] {
    [
        "X-Client-Version": Bundle.main.shortVersionString,
        "X-Platform": "iOS"
    ]
}
```

---

## Android & Web Parity

Android and Web must send identical headers and feature semantics.
Never allow platform divergence in feature meaning.

---

## When Things Go Wrong

If telemetry reports unknown enum values or schemaVersion > current support:

- Log it
- Disable gated feature
- Fall back to stable UI
- Alert backend automatically

---

## Designing for This as Early as Possible

Every API should:

- Be wrapped in a versioned envelope
- Contain feature flags
- Support unknown enum handling
- Have deprecation telemetry baked in

---

## Designing Versioned Payloads and Feature Gates Early — and Retrofitting Them Safely

Designing versioned envelopes and feature gates from day one is **strongly preferred**, but many real systems
inherit APIs that shipped without these protections. This guide explains both:

- how to design them early, and  
- how to introduce them safely into existing backend + mobile ecosystems.

---

## Why “Day One” Design Is Ideal (But Rare)

Early design advantages:

- No breaking changes when adding new fields
- Predictable deprecation workflows
- Built-in telemetry for schema drift
- Cross-platform coordination baked into contracts

But reality:

- Many APIs predate mobile
- Versioning was added ad‑hoc or not at all
- Feature flags live only in backend or web
- Mobile apps crash when enums change

The solution is **incremental hardening**, not rewriting history.

---

## Step 1 — Introduce a Non-Breaking Envelope

### Backend

Wrap responses without removing existing fields:

```json
{
  "schemaVersion": 1,
  "features": [],
  "data": {
    "id": "123",
    "status": "pending"
  }
}
```

Keep the legacy flat payload available during transition.

### iOS Client

```swift
struct EnvelopeDTO<T: Decodable>: Decodable {
    let schemaVersion: Int
    let features: [String]
    let data: T
}
```

Deploy decoding support *before* backend switches fully.

---

## Step 2 — Dual-Read Period

For one full release cycle:

- Backend sends **both** flat and enveloped formats
- Client supports both shapes:

```swift
enum OrderPayload: Decodable {
    case legacy(OrderDTO)
    case enveloped(EnvelopeDTO<OrderDTO>)

    init(from decoder: Decoder) throws {
        if let env = try? EnvelopeDTO<OrderDTO>(from: decoder) {
            self = .enveloped(env)
        } else {
            self = .legacy(try OrderDTO(from: decoder))
        }
    }
}
```

This prevents crash spikes during rollout.

---

## Step 3 — Feature Flag Backfill

Introduce feature flags for *existing* behaviors:

- `legacy_checkout`
- `new_checkout`
- `beta_pricing`

Even if nothing is gated yet, shipping the pipeline early avoids
future rewrites.

---

## Step 4 — Telemetry First, Removal Last

Before removing any field or enum case:

1. Add logging for unknown / deprecated usage
2. Track client versions still sending / reading old fields
3. Wait until traffic <1%
4. Then remove — behind a kill switch

---

## Step 5 — Cross‑Platform Coordination

Create shared contract docs:

```
/contracts/schema/v2_checkout.md
```

Document:

- feature intent
- rollout strategy
- fallback semantics
- deprecation window

All platforms implement from this source of truth.

---

## Common Pitfalls When Retrofitting

| Mistake | Outcome |
|-------|---------|
| Removing fields before telemetry | Silent crashes |
| Feature flags only in backend | Clients hard‑code behavior |
| No dual‑read window | Emergency hotfix releases |
| Platform divergence | iOS / Android behavior drift |

---

## Key Concepts

- **Retrofitting must be incremental and dual‑read**
- **Versioned envelopes prevent catastrophic drift**
- **Feature flags are contracts, not toggles**
- **Telemetry is required before deletion**
- **Cross‑platform contracts are as important as API docs**
- **All platforms must evolve together**
- **Versioning is a system, not a field**
- **Never trust rollout timing**
