
# Schema Migration Strategies
**Chapter 05 – Client Schema Resilience**

> A practical guide for evolving backend schemas without breaking iOS clients.

---

## Why This Matters

Mobile apps live in the wild for *years*. Old versions continue to call your APIs long after you ship a new backend.  
You cannot rely on “everyone updating” — you must design for permanent backward compatibility.

---

## Real‑World Change Categories

### 1. Minor Changes (Low Risk)
Examples:
- Adding new optional fields
- Expanding enum cases
- Adding metadata blocks

**Tactics**
- Additive only
- Fields must be nullable / optional
- iOS decoders must ignore unknown keys

**Gotcha**
- Adding a *required* field is **not minor**.

---

### 2. Medium Changes (Moderate Risk)
Examples:
- Renaming fields
- Changing types (String → Object)
- Splitting a field into multiple concepts

**Tactics**
- Dual‑write both schemas
- Maintain backward compatibility for 2–3 release cycles
- Add translation layers server‑side

**Gotcha**
- Renames without alias support break analytics and caching layers.

---

### 3. Large Changes (High Risk)
Examples:
- Entire object redesigns
- Breaking pagination semantics
- Replacing nested graphs with flattened representations

**Tactics**
- Introduce a new schema version
- Run old + new schemas in parallel
- Use capability negotiation

**Gotcha**
- Never force‑upgrade schema for iOS users already in production.

---

## Rollout Gameplan

### Step 1 – Make sure Everything is already Enveloped

Every API response should be wrapped:

```json
{
  "schemaVersion": 3,
  "features": ["flex_pricing"],
  "data": { ... }
}
```

---

### Step 2 – Header Negotiation

Client sends:

```
X-Client-Schema-Version: 2
X-Client-Capabilities: feature_flags,partial_checkout
```

Server responds with the *highest compatible* schema.

---

### Step 3 – Canary Rollout

- Enable new schema for 1% of traffic
- Monitor crash logs, decoding errors, latency
- Gradually ramp to 10%, 25%, 50%, 100%

**Telemetry to Track**
- Decode failures by schemaVersion
- Fallback usage rate
- Unknown enum frequency

---

## Incremental Migration Pattern

| Phase | Backend | iOS Client |
|------|---------|------------|
| Phase 0 | Old schema only | Old parsing |
| Phase 1 | Dual‑write | Add tolerant decoders |
| Phase 2 | New schema default | Shadow‑parse new |
| Phase 3 | Remove old paths | Remove dead code |

**Gotcha**
- Leaving dead code “just in case” becomes permanent tech debt.

---

## When to Bump `schemaVersion`

| Change Type | Bump? |
|------------|-------|
| Add optional field | No |
| Add enum case | No |
| Rename field | Yes |
| Change meaning of field | Yes |
| Remove field | Yes |
| Structural object change | Yes |

---

## Clients Without Best Practices

**Symptoms**
- Crashes on unknown enum values
- Hard failures on missing fields
- Manual JSON parsing

**Mitigation**
- Introduce *adapter APIs* for legacy clients server-side
- Freeze old server-side schema permanently
- Force them to migrate via app update, not backend change

---

## Clients With Best Practices

They support:
- Unknown enum handling
- Optional decoding
- Feature flag gating
- Schema negotiation

These clients:
- Ship faster
- Break less
- Enable backend velocity

---

## Fallback Strategy

If decode fails:
1. Log telemetry
2. Switch to last‑known‑good schema
3. Trigger kill‑switch remotely

---

## Gotchas from Real Teams

- “We’ll clean it later” never happens.
- Removing fields too early breaks BI pipelines.
- Enum expansion breaks Swift if not defensive.
- Canary without telemetry is theater.

---

## Key Concepts

- Schema envelopes are mandatory.
- Additive first, destructive last.
- Version headers beat URL versioning.
- Dual‑write beats big‑bang rewrites.
- Telemetry is your early‑warning system.

---

## Summary

Schema migration is not an event — it is a **continuous operational discipline**.  
Teams that treat schema evolution as a first‑class engineering system ship faster, break less, and sleep better.
