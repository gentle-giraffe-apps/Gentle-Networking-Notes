# Chaos Testing for Mobile Backends

**Audience:** Mobile engineers (Senior / Staff)  
**Goal:** Systematically surface failure modes that only appear under real-world conditions like flaky radios, background suspension, and backend degradation.

---

## What Is Chaos Testing?

**Chaos testing** is the practice of intentionally injecting failures into a system to validate that it behaves safely under stress.  
On mobile, this is not optional — it is the only way to simulate the environments your users actually live in.

Chaos testing answers:  
> “What happens when *everything goes wrong at the same time*?”

---

## Why Mobile Is a Chaos Magnet

| Mobile Reality | Backend Impact |
|---------------|----------------|
| Radio sleep cycles | TLS (Transport Layer Security) handshakes fail mid-flight |
| OS suspends apps | Requests are cancelled without callbacks |
| Captive portals | Network looks connected but is unusable |
| Version skew | Old clients hit new servers |
| Background retries | Retry storms during outages |

---

## Types of Chaos Experiments

| Category | Example |
|---------|--------|
| Network | 10% packet loss, 2s latency |
| Backend | Return 500 errors for one endpoint |
| Client | Kill app mid-write |
| Infrastructure | Expire authentication tokens early |
| Data | Corrupt JSON fields |

---

## Pattern — Failure Injection Layer

Introduce a fault-injectable transport.

```swift
protocol NetworkTransport {
    func send(_ request: URLRequest) async throws -> Data
}
```

Swap implementations to inject latency, corruption, or errors.

---

## Pattern — Scenario-Based Chaos

| Scenario | Expected Behavior |
|---------|------------------|
| Token expires mid-upload | Pause + reauth + resume |
| CDN (Content Delivery Network) fails | Images fallback to placeholders |
| Backend returns 503 | Circuit breaker opens |
| App backgrounded during sync | Queue persists safely |

---

## Pattern — Chaos Profiles

| Profile | Simulates |
|--------|-----------|
| Subway | 40% loss, 800ms latency |
| Hotel WiFi | Captive portal + DNS failures |
| Low Power Mode | Aggressive OS throttling |
| Outage | 80% 5xx backend errors |

---

## Pattern — Observability Hooks

Chaos without telemetry is theater.

Capture:

- Network path state
- Battery level
- Retry count
- Queue depth
- Circuit breaker state

---

## Common Gotchas

| Symptom | Root Cause |
|--------|-----------|
| Tests flaky in CI | Wall-clock timing |
| No actionable insight | Missing telemetry |
| False confidence | Only testing “offline mode” |
| Chaos ignored | No gating in CI |

---

## Key Follow-Up Questions

**Q: When should chaos tests run?**  
A: Nightly CI and before major backend releases.

**Q: How do you avoid wasting engineer time?**  
A: Encode chaos scenarios as deterministic profiles.

**Q: How do you prevent panic in QA?**  
A: Flag chaos builds clearly and isolate environments.

---

## One-Page Summary

| Rule | Why |
|------|-----|
| Inject failures deliberately | Reality check |
| Scenario-based chaos | Reproducibility |
| Capture telemetry | Learn from failure |
| Run in CI | Prevent regressions |
| Tie to product UX | Meaningful resilience |

---

## Two-Sentence Summary

Chaos testing is the only honest way to understand how mobile apps fail in the wild.  
If you are not injecting failure, production is doing it for you.
