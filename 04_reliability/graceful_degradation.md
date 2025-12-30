# Graceful Degradation on Mobile

**Audience:** Senior / Staff mobile engineers  
**Goal:** Ensure apps fail *politely* instead of catastrophically when conditions degrade.

---

## What Graceful Degradation Actually Means

Graceful degradation is the discipline of *preserving user intent* even when technical constraints appear.  
It is not about hiding errors — it is about reshaping the experience so the user can still accomplish something meaningful.

| Degradation | What Happens |
|------------|---------------|
| Low bandwidth | Images downscale, videos paused |
| Auth expiry | Read-only guest mode |
| CDN outage | Placeholder images with lazy retry |
| Write failures | Offline queue + optimistic UI |

The user should always feel the app is *working with them*, not against them.

---

## Pattern — Define Experience Tiers

Every app has implicit modes. Make them explicit.

| Tier | Capabilities | Example |
|------|--------------|---------|
| Ideal | Full read/write, realtime refresh | On strong WiFi |
| Degraded | Reads + queued writes | Subway LTE |
| Survival | Cached read-only | Airplane mode |
| Dead | Friendly failure | Backend outage |

By defining tiers early, product and engineering agree on *acceptable loss*.

---

## Pattern — UX Over Accuracy

Users prefer slightly stale data over empty screens.

```swift
func loadFeed() async {
    do {
        let fresh = try await api.fetchFeed()
        cache.store(fresh)
        show(fresh)
    } catch {
        show(cache.lastSnapshot())
    }
}
```

This pattern ensures the user sees *something* even when freshness cannot be guaranteed.

---

## Pattern — Feature Flags for Failure

Instead of branching on error types, derive runtime feature availability.

```swift
struct FeatureAvailability {
    let canUploadMedia: Bool
    let canPost: Bool
    let canRefresh: Bool
}
```

This avoids deeply nested error handling scattered throughout the UI.

---

## Pattern — Progressive Disclosure of Errors

Not all failures deserve equal attention.

| Failure | UX |
|--------|----|
| Image fetch | Silent placeholder |
| Feed refresh | Subtle banner |
| Write failure | Inline retry with context |
| Auth expired | Blocking modal reauth |

Escalate only when the user’s intent is blocked.

---

## Pattern — Local-First Fallbacks

| Component | Fallback |
|----------|----------|
| Feed | Last snapshot |
| Search | Recent history |
| Profile | Cached identity |
| Settings | Local defaults |

Local data should be the *first resort*, not the last.

---

## Common Gotchas

| Symptom | Root Cause |
|--------|------------|
| App feels broken | Too many blocking spinners |
| Users distrust data | Empty states instead of cached |
| Rage taps | No feedback on queued actions |
| Support tickets | Errors surfaced too aggressively |

---

## Key Follow-Ups

**Q: How do you know when to degrade?**  
A: From capability flags driven by sync failures, auth state, and connectivity heuristics.

**Q: How do you avoid lying to users?**  
A: Subtle staleness indicators, never silence data loss.

**Q: How do you test degradation?**  
A: Chaos toggles, fault injection, and network conditioners in CI.

---

## One-Page Summary

| Rule | Why |
|------|-----|
| Preserve intent | Builds trust |
| Prefer stale over empty | Continuity |
| Capability-driven UI | Predictable |
| Tier error surfacing | Lower anxiety |
| Always show progress | Prevent rage taps |
| Chaos-test failure | Production realism |

> Graceful degradation is empathy encoded in architecture.
