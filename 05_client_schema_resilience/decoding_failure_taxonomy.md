
# Decoding Failure Taxonomy
**Chapter 05 – Client Schema Resilience**

> Not all decoding failures are equal. Treating them as such is how systems rot.

---

## Why a Taxonomy Matters

When every failure is logged as "decode_error", teams lose the ability to:
- Prioritize fixes
- Correlate failures to rollouts
- Distinguish schema drift from true bugs

---

## Failure Classes

### 1. Structural Mismatch (Fatal)

| Example | Cause |
|-------|-------|
| Missing required root object | Breaking backend change |
| Type changed Int → Object | Schema contract violated |

**Response**
- Fail fast
- Block canary rollout
- Immediate backend rollback

---

### 2. Semantic Drift (High)

| Example | Cause |
|--------|------|
| Field meaning changed | Silent contract change |
| Unit change (ms → s) | Backend refactor leak |

**Response**
- Log remotely with priority
- Patch via migration layer

---

### 3. Enum Expansion (Medium)

| Example | Cause |
|--------|------|
| New enum case added | Backend feature rollout |

**Response**
- Fallback to `.unknown`
- Increment schema telemetry

---

### 4. Optional Field Addition (Low)

| Example | Cause |
|--------|------|
| New optional metadata | Additive change |

**Response**
- Capture in extras
- Observe frequency

---

### 5. Data Corruption (Rare but Severe)

| Example | Cause |
|--------|------|
| Truncated JSON | Proxy / CDN issue |
| Invalid UTF-8 | Encoding bug |

**Response**
- Retry once
- Quarantine payload
- Alert ops

---

## Logging Matrix

| Class | Persist Locally | Upload Remotely | Alert |
|------|-----------------|-----------------|-------|
| Structural | Yes | Yes | Yes |
| Semantic | Yes | Yes | No |
| Enum | No | Yes | No |
| Optional | No | Sampled | No |
| Corruption | Yes | Yes | Yes |

---

## Gotchas

- Lumping semantic drift into enum expansion hides business breakage.
- Retrying structural failures wastes battery and bandwidth.
- Treating corruption as schema drift delays infra response.

---

## Summary

A decoding failure taxonomy transforms crashes into signals.  
Without it, observability is noise, not intelligence.
