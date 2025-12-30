# Contract Testing for Mobile APIs

**Audience:** Mobile engineers (Senior / Staff)  
**Goal:** Prevent production outages caused by backend changes by validating API contracts continuously.

---

## What Is Contract Testing?

**Contract testing** verifies that a client (mobile app) and a provider (backend service) agree on the *shape* and *semantics* of their API — without requiring full end‑to‑end environments.

A *contract* defines:

- Endpoint paths
- HTTP methods
- Required / optional fields
- Data types and constraints
- Error formats

Instead of testing the full system, each side validates its own promises.

---

## Why Mobile Needs It More Than Anyone

Mobile apps ship to devices you cannot hotfix:

| Reality | Risk |
|-------|------|
| App Store review delays | Backend change breaks production users |
| Background execution limits | Failures only happen at scale |
| Offline caching | Old schemas persist longer |
| Long upgrade cycles | 6‑month‑old apps still in wild |

Contract testing gives you **early warnings** before the release train leaves the station.

---

## Consumer‑Driven Contracts (CDC)

The mobile app defines what it *needs* — not what the backend happens to return today.

```json
{
  "request": { "method": "GET", "path": "/profile" },
  "response": {
    "status": 200,
    "body": {
      "id": "string",
      "displayName": "string",
      "avatarURL": "string?"
    }
  }
}
```

The backend validates it still satisfies this contract.

---

## Pattern — Schema‑First Development

Define API schemas using tools like OpenAPI (Open Application Programming Interface).

Benefits:

- Autogenerate Swift models
- Detect breaking changes
- Share one source of truth

---

## Pattern — Contract CI Gate

Your Continuous Integration (CI) pipeline should:

| Step | Purpose |
|-----|---------|
| Validate mobile contracts | Fail if backend breaks |
| Validate backend stubs | Fail if client assumptions drift |
| Run serialization tests | Catch decoding crashes |

---

## Pattern — Backward Compatibility Windows

Never remove fields immediately.

| Version | Behavior |
|--------|----------|
| N | Field added |
| N+1 | Clients migrate |
| N+2 | Field eligible for removal |

Mobile reality demands slow, deliberate evolution.

---

## Common Gotchas

| Symptom | Root Cause |
|--------|-----------|
| Crash on decode | Optional field became required |
| UI breaks silently | Type changed server‑side |
| QA can’t repro | Only older app versions affected |
| Emergency rollback | Missing contract enforcement |

---

## Key Follow‑Up Questions

**Q: Why not rely on integration tests?**  
A: They require full environments and rarely cover version skew in production.

**Q: How do you test contracts without real servers?**  
A: Generate mock providers from contract files.

**Q: How do you handle optional fields safely?**  
A: Default decoding strategies and schema constraints.

---

## One‑Page Summary

| Rule | Why |
|------|-----|
| Use consumer‑driven contracts | Mobile defines truth |
| Schema‑first APIs | Shared language |
| CI contract validation | Catch breakage early |
| Enforce compatibility windows | Protect shipped apps |
| Test decoding paths | Prevent runtime crashes |

---

## Two‑Sentence Summary

Contract testing protects mobile apps from backend drift long after they ship.  
It replaces fragile integration testing with durable, version‑aware guarantees.
