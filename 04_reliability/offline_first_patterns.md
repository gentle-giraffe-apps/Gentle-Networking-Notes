# Offline‑First Networking Patterns (Mobile)

**Audience:** Senior / Staff mobile engineers  
**Goal:** Build resilient mobile apps that treat connectivity as unreliable by default.

---

## Why Offline‑First Matters

Mobile networks are *hostile environments*:

- Radios sleep aggressively
- Captive portals lie about connectivity
- Background suspension cancels requests
- Processes are killed mid‑write
- Users move through dead zones constantly

**Offline‑first does not mean “no network required”.**  
It means the *core experience never blocks on the network*.

---

## Pattern 1 — Network Is a Sync Layer, Not Source of Truth

```
UI → Local Store → Render
             ↕
         Sync Engine → Network
```

The UI only reads local state. The server is a replica.

---

## Pattern 2 — Durable Command Queue

Every mutating action is persisted before transmission.

```swift
struct PendingMutation: Codable, Identifiable {
    let id: UUID
    let endpoint: String
    let payload: Data
    let createdAt: Date
    let retryCount: Int
}
```

**Benefits**

| Without Queue | With Queue |
|--------------|-----------|
| Data loss on airplane mode | Guaranteed durability |
| Crashes lose writes | Replayable mutations |
| No audit trail | Full traceability |

---

## Pattern 3 — Idempotent Writes

Every write must be safely retryable.

```
POST /notes
Idempotency-Key: <UUID>
```

Prevents duplicate objects when retries occur after app suspension.

---

## Pattern 4 — Connectivity Is a Hint

`NWPathMonitor` can report reachable while behind captive portals.

Treat errors as state:

```swift
enum SyncState { case idle, syncing, offline, backoff }
```

---

## Pattern 5 — Progressive Backoff with Jitter

```
delay = min(60, pow(2, attempt)) + random(0...3)
```

Avoids network thrashing and server overload.

---

## Pattern 6 — Partial Sync Windows

Never sync everything.

| Resource | Window |
|---------|--------|
| Feed | 7 days |
| Messages | last 200 |
| Profile | always |

---

## Pattern 7 — Conflict Resolution Is a Product Choice

| Strategy | UX |
|---------|----|
| Last‑write‑wins | Simple, risky |
| Server authoritative | Possible data loss |
| Client authoritative | Best for personal data |
| Merge UI | Rare but safest |

---

## Pattern 8 — Observability

Log *why* sync fails, not just that it failed.

```swift
struct SyncFailure {
    let mutationID: UUID
    let error: Error
    let networkPath: NWPath.Status
    let battery: UIDevice.BatteryState
}
```

---

## Common Gotchas

| Symptom | Cause |
|--------|------|
| Offline button useless | UI still blocks on network |
| Duplicate records | Missing idempotency |
| Missing user data | Queue eviction |
| Long cold start | No local snapshot |

---

## Key Follow‑Ups

**Q: How do you survive background suspension?**  
Persist every mutation before sending.

**Q: How do you test offline?**  
Disable radios, chaos‑test retries, inject fake monitors.

**Q: How do you avoid inconsistent UI?**  
UI reads only local store.

---

## One‑Page Summary

| Rule | Why |
|-----|-----|
| Network is replica | Prevents blocked UI |
| Queue every write | Zero data loss |
| Idempotent APIs | Safe retries |
| Backoff with jitter | Avoid meltdown |
| Partial sync windows | Performance |
| Connectivity lies | Captive portals |
| Conflict strategy is product | Predictability |
| Log context | Debug reality |

> Offline‑first is not a feature. It is an architectural posture.
