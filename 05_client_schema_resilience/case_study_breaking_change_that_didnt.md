
# Case Study: The Breaking Change That Didn’t
**Chapter 05 – Client Schema Resilience**

> How a real production-breaking backend change became a non-event.

---

## The Incident

A backend team needed to replace:

```
price: Double
```

with:

```
pricing: { amount: Double, currency: String }
```

This was required for internationalization and dynamic discounts.

Historically this would have caused:
- Decode crashes
- Corrupted offline caches
- Emergency rollbacks

It didn’t.

---

## Why Nothing Broke

### 1. Enveloped Responses Were Already Live

All payloads were wrapped:

```json
{
  "schemaVersion": 3,
  "features": ["pricing_v2"],
  "data": { ... }
}
```

### 2. Header Negotiation Was Active

Clients sent:

```
X-Client-Schema-Version: 2
X-Client-Capabilities: pricing_v2
```

Only capable clients received the new structure.

---

## The Migration

### Phase 0 – Shadow Write

Backend dual-wrote both fields:

```
price
pricing
```

### Phase 1 – Canary

1% of traffic received `pricing` while `price` remained.

Observability showed:
- Enum fallback rate unchanged
- No structural decode failures

### Phase 2 – Client Backfill

iOS logic:

```swift
if pricing == nil {
    pricing = payload.extra?["pricing"]?.decode(Pricing.self)
}
```

Old cached payloads upgraded automatically.

---

## Observability Saved the Day

Telemetry dashboard showed:

| Metric | Before | After |
|-------|--------|-------|
| Structural failures | 0.00% | 0.00% |
| Enum fallbacks | 0.02% | 0.02% |
| Unknown fields | +pricing | Expected |

Canary was ramped safely to 100%.

---

## What Would Have Happened Without This

| Missing Practice | Result |
|------------------|--------|
| No envelope | Root decode crash |
| No extras capture | Data loss |
| No taxonomy | Canary ignored |
| No telemetry | Blind rollout |

---

## Lessons

- Schema evolution is operational discipline.
- Unknown fields are future-proofing, not clutter.
- Telemetry is how you earn the right to move fast.

---

## Final Thought

The best breaking change is the one your users never notice.
