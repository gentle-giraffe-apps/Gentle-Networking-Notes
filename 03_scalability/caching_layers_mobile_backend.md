
# Caching Layers for a Mobile‑First Backend
*Gentle‑Networking‑Notes · 03_scalability*

> A practical guide to designing **multi‑layer caching** that respects mobile constraints: battery, bandwidth, latency, offline‑first UX, and Swift Concurrency correctness.

---

## Why Caching Is a Mobile Scalability Problem

On mobile, backend scalability is not just QPS — it is:

| Constraint | Impact |
|-----------|--------|
| Radio wakeups | Battery drain |
| Cold starts | User‑perceived slowness |
| Flaky connectivity | Offline correctness |
| OS memory pressure | Cache eviction storms |
| Background limits | Prefetch fragility |

A single server‑side cache is not enough. You need **layered caches that cooperate across client + edge + backend**.

---

## The 6‑Layer Cache Pyramid

```
┌───────────────────────────────┐
│ 6. Analytics / Event Cache    │ (asynchronous write‑behind)
├───────────────────────────────┤
│ 5. CDN / Edge Cache           │ (Cloudflare / Fastly / Akamai)
├───────────────────────────────┤
│ 4. Backend In‑Memory Cache    │ (Redis / Memcached)
├───────────────────────────────┤
│ 3. Backend Persistent Cache   │ (Postgres / Dynamo / Firestore)
├───────────────────────────────┤
│ 2. Client Disk Cache          │ (URLCache / SQLite / Filesystem)
├───────────────────────────────┤
│ 1. Client Memory Cache        │ (Actor‑protected LRU / NSCache)
└───────────────────────────────┘
```

Each layer trades **latency ↔ durability ↔ correctness**.

---

## Layer 1 — Client Memory Cache (Swift Concurrency‑Safe)

**Goal:** Zero‑latency UI rendering without touching disk or radio.

Design pattern:

```swift
actor MemoryCache<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]

    func value(for key: Key) -> Value? { storage[key] }
    func insert(_ value: Value, for key: Key) { storage[key] = value }
    func removeAll() { storage.removeAll() }
}
```

Rules:

* Never block the main actor.
* Purge aggressively on memory warnings.
* Never treat as source of truth.

---

## Layer 2 — Client Disk Cache (Offline First)

**Goal:** App launch should work in airplane mode.

Options:

| Use case | Tool |
|---------|------|
| HTTP responses | `URLCache` |
| Structured domain models | SQLite / GRDB |
| Media | Filesystem + hash key |

Key policy:

* All disk caches must carry **schema version + TTL**.
* Disk corruption must be survivable — fail open, never crash.

---

## Layer 3 — Backend Persistent Cache

This is your **materialized view layer**.

Example:

| Endpoint | Cache table |
|---------|-------------|
| `/feed` | `feed_snapshots(user_id, payload, updated_at)` |
| `/profile` | `user_profile_cache(user_id, payload, etag)` |

Invalidation is write‑time, never read‑time.

---

## Layer 4 — Backend In‑Memory Cache

Use Redis/Memcached only for **hot aggregates**:

* leaderboard pages
* feature flag payloads
* pricing tables

Never store user‑specific private data unless encrypted at rest.

---

## Layer 5 — CDN / Edge Cache

**Goal:** Kill cold‑start latency globally.

Mobile‑specific headers:

```
Cache-Control: public, max-age=60, stale-while-revalidate=300
ETag: "profile-v42"
```

This allows:

* instant load from edge
* silent refresh behind the scenes

---

## Layer 6 — Analytics Write‑Behind Cache

Do not block the UI on telemetry.

Pattern:

* Buffer events locally
* Flush opportunistically on WiFi + power

---

## Cache Read Strategy

```
Memory → Disk → CDN → Redis → DB → Source of Truth
```

Return the **first hit**, refresh all lower layers asynchronously.

---

## Cache Invalidation Is a Product Problem

Mobile bugs happen when backend engineers forget:

* User logs out
* User switches accounts
* Device time is wrong
* App upgrades schema

**Every cache entry must contain:**

```
(userID, appVersion, schemaVersion, ttl)
```

---

## Battery Budgeting

Each radio wakeup ≈ **~100–300ms of high‑power drain**.

Rule: Never perform more than **1 network call per screen load**.

---

## Observability

Track per‑layer hit rates:

| Layer | Target |
|-------|--------|
| Memory | >70% |
| Disk | >20% |
| CDN | >85% |
| Redis | <10% |
| DB | <2% |

If DB >2%, you are scaling wrong.

---

## Swift‑First Failure Modes

| Bug | Symptom |
|-----|--------|
| Non‑actor cache | Data races |
| Blocking disk IO | UI hitches |
| No TTL | Zombie UI states |
| ETag mismatch | Infinite refresh loops |

---

## Key Concepts

> “If your app works perfectly in airplane mode for 30 minutes, your backend is probably scalable.”

---

*Part of Gentle‑Networking‑Notes · 03_scalability*
