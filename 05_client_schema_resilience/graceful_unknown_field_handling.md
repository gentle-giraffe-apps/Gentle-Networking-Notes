
# Graceful Unknown Field Handling
**Chapter 05 – Client Schema Resilience**

> How to survive schema evolution without crashing your users.

---

## Why Unknown Fields Exist

Backend teams move faster than mobile release cycles.
By the time a new app ships, the server may have already introduced fields the client has never seen.

Your system must assume:

> Unknown fields are normal, not exceptional.

---

## UI Experience Principles

Unknown fields must **never**:
- Crash the app
- Blank entire screens
- Break navigation flows

Instead they should:
- Be ignored safely
- Render placeholders if necessary
- Fail locally, not globally

---

## iOS Client Best Practices

### Decoding Strategy

Use tolerant decoding:

```swift
struct Payload: Decodable {
    let known: String
    let extra: [String: AnyDecodable]?
}
```

When decoding API responses, clients should not silently discard fields they do not recognize. Instead, they should capture all unknown key–value pairs into a generic dictionary (for example, `extra: [String: AnyDecodable]`) so that forward-introduced fields are preserved even if the current app version does not yet model them. This prevents irreversible data loss when payloads are cached, persisted, or replayed later, and enables future app versions to migrate or reinterpret previously unknown data. Teams should also consider emitting the _names of unknown fields_ via telemetry (batched and privacy-safe) so backend and mobile engineers can observe schema drift in real time, detect accidental breaking changes early, and coordinate migrations based on evidence rather than production failures.

### Enum Safety

```swift
enum Feature: Decodable {
    case newCheckout
    case unknown(String)
}
```

This prevents fatal decode crashes when backend adds cases.

---

## Server Responsibilities

- Never assume clients ignore unknown fields.
- Add fields as optional first.
- Run shadow serialization to older schemas before rollout.

---

## Caching & Persistence

### Mobile Local DB

If you persist API payloads:

| Risk | Mitigation |
|-----|-----------|
| DB schema mismatch | Version your local DB |
| Unknown fields dropped | Persist raw JSON blobs |
| Decode crash on rehydrate | Store envelope metadata |

### Backend Caches

- Edge caches must vary on schemaVersion header.
- CDN cache keys must include client version or capability hash.

---

## Incremental Rollout Model

1. Add field behind feature flag
2. Enable for internal builds
3. Canary to 1% users
4. Ramp gradually

---

## Real World Gotchas

- Snapshot tests silently accept new fields.
- SQLite migrations are forgotten until first crash.
- Unknown fields inside nested arrays are hardest to detect.
- Background tasks rehydrate stale payloads days later.

---

## Database Schema Design

| Layer | Strategy |
|------|----------|
| API | Additive only |
| iOS Persistence | Raw JSON + typed projections |
| Analytics | Forward-compatible ingestion |

---

## Telemetry You Must Track

- Unknown field frequency by endpoint
- Enum fallback rate
- Decode error count per schemaVersion

---

## Key Concepts

- Unknown ≠ Error
- Additive first, destructive last
- Headers negotiate, envelopes explain
- Persist raw when possible

---

## Summary

Graceful handling of unknown fields is the foundation of long-lived mobile systems.
If unknown fields crash your app, your schema strategy is already broken.
