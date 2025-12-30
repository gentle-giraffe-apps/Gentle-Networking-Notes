# Partial Failure Handling on Mobile

**Audience:** Senior / Staff mobile engineers  
**Goal:** Design apps that remain usable when only *parts* of the system fail.

---

## Why Partial Failures Are the Norm

In distributed mobile systems, it is rare for everything to fail at once.

Common partial failures:

- Auth service down, content service up
- Read endpoints succeed, writes timeout
- Image CDN failing, JSON API healthy
- Background refresh works, foreground blocked

---

## Pattern — Decompose Dependencies

Avoid monolithic “load everything” calls.

```
Profile → Auth
Feed → Content
Images → CDN
`````

Each dependency fails independently.

---

## Pattern — Capability Matrix

| Capability | Online | Partial | Offline |
|-----------|--------|---------|---------|
| Browse cached feed | ✔️ | ✔️ | ✔️ |
| Like / Save | ✔️ | ✔️ | ✔️ (queued) |
| Post comment | ✔️ | ⚠️ | ❌ |
| Upload media | ✔️ | ❌ | ❌ |

---

## Pattern — Tiered Error Surfaces

| Failure | UX |
|--------|----|
| Image fetch | Placeholder |
| Feed refresh | Toast |
| Write failure | Inline retry |
| Auth expiry | Full-screen reauth |

---

## Pattern — Fallback Trees

```text
Load Feed
  ├─ CDN images → fallback placeholders
  ├─ Content API → cached snapshot
  └─ Auth expired → limited guest mode
```

---

## Pattern — Capability Flags

```swift
struct RuntimeCapabilities {
    let canRead: Bool
    let canWrite: Bool
    let canUpload: Bool
}
```

Drive UI visibility from live capability state.

---

## Common Gotchas

| Symptom | Cause |
|--------|------|
| Blank screens | Single fatal dependency |
| Endless spinners | Missing terminal state |
| Random crashes | Optional capability unwrapped |
| Confusing errors | Wrong error tier surfaced |

---

## Key Follow-Ups

**Q: How do you test partial failures?**  
A: Fault injection per-service, not global offline mode.

**Q: How do you keep UI predictable?**  
A: Capability-driven rendering, not error-driven rendering.

**Q: How do you avoid cascading errors?**  
A: Fail locally, never propagate upstream by default.

---

## One-Page Summary

| Rule | Why |
|------|-----|
| Dependencies fail independently | Reality |
| Build capability matrix | Product clarity |
| Tier error surfaces | Reduce noise |
| Use fallback trees | Graceful UX |
| Capability-driven UI | Predictability |
| Fault inject per-service | Real testing |

> Partial failure handling is the difference between resilience and chaos.
