# Circuit Breakers on Mobile

**Audience:** Senior / Staff mobile engineers  
**Goal:** Prevent cascading failures when backend services degrade under poor mobile connectivity.

---

## Why Circuit Breakers Matter on Mobile

Without circuit breakers, mobile apps can unintentionally amplify outages:

- Aggressive retries on flaky LTE
- Background wakes causing thundering herds
- Battery drain from repeated failures
- Server rate limiting that looks like client bugs

---

## Pattern — The Mobile Circuit Breaker

```swift
enum CircuitState {
    case closed      // normal traffic
    case open        // fail fast
    case halfOpen    // limited probes
}
```

```
UI → Local Store → Sync Engine → Circuit Breaker → Network
```

---

## State Transitions

| From | To | Trigger |
|------|----|--------|
| closed | open | Error rate > threshold |
| open | halfOpen | Cool-down timer elapsed |
| halfOpen | closed | Probe request succeeds |
| halfOpen | open | Probe request fails |

---

## Failure Signals Worth Tracking

| Signal | Why |
|--------|-----|
| Timeout rate | Captive portals & radio sleep |
| 5xx rate | Backend degradation |
| TLS failures | MITM portals |
| DNS failures | OS resolver flakiness |

---

## Pattern — Sliding Failure Window

```swift
struct FailureSample {
    let timestamp: Date
    let success: Bool
}
```

Evaluate last N samples over T seconds.

---

## Pattern — Adaptive Thresholds

| Network | Open Threshold |
|---------|---------------|
| WiFi | 40% |
| LTE / 5G | 60% |
| Low Power Mode | 25% |

---

## Pattern — UI-Aware Short-Circuiting

When circuit is open:

- Skip network calls
- Surface cached content
- Show “We’ll retry when connection improves”

---

## Common Gotchas

| Symptom | Root Cause |
|--------|-----------|
| Circuit never closes | No half-open probes |
| Users stuck offline | Cool-down too long |
| Works on WiFi only | Thresholds not adaptive |
| Battery drain | Breaker not global per-host |

---

## Key Follow-Ups

**Q: Why not rely on URLSession retry policies?**  
A: They retry blindly and don’t model systemic failure.

**Q: How do you test breakers?**  
A: Inject failing transports and advance virtual clocks.

**Q: How do breakers interact with offline-first queues?**  
A: They gate flush attempts without discarding mutations.

---

## One-Page Summary

| Rule | Why |
|------|-----|
| Track failure windows | Detect degradation |
| Short-circuit fast | Protect battery & UX |
| Use half-open probes | Automatic recovery |
| Adaptive thresholds | Mobile ≠ server |
| UI fallback paths | Graceful failure |
| One breaker per host | Avoid cascade loops |

> Circuit breakers are not pessimism. They are compassion for failing systems.
