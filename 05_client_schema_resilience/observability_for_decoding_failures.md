
# Observability for Decoding Failures
**Chapter 05 – Client Schema Resilience**

> If decoding failures are invisible, they are guaranteed to repeat.

---

## Why Decoding Failures Are Special

Unlike crashes, decoding errors often:
- Happen in background tasks
- Affect only subsets of users
- Are silently swallowed by retry loops

Without observability, teams only notice after customers complain.

---

## Error Classification

| Severity | Example | Action |
|---------|--------|--------|
| Minor | Unknown optional field | Log locally |
| Medium | Enum fallback used | Aggregate & upload |
| Major | Root decode failure | Immediate alert |

---

## Logging Without Explosion

Never log raw payloads. Instead log:
- Endpoint name
- schemaVersion
- Unknown field names (hashed)
- Error category
- Client version

Batch logs locally and upload periodically.

---

## Local vs Remote Telemetry

| Location | Purpose |
|---------|---------|
| On-device ring buffer | Fast debug, offline support |
| Remote aggregation | Trend detection, alerting |

---

## Sync Engine Considerations

If your app has background sync:
- Classify failures as *transient* vs *fatal*
- Retry transient failures with backoff
- Persist fatal failures for upload

---

## Canary Rollout Monitoring

During schema rollout:
- Compare decode error rates between canary and control
- Block rollout if delta exceeds threshold

---

## Common Gotchas

- Logging full JSON causes PII leaks.
- Sampling too aggressively hides regressions.
- Not versioning logs makes trends meaningless.

---

## Key Concepts

- Classify before logging.
- Persist before uploading.
- Observe before blaming.

---

## Summary

Decoding failures are not bugs — they are signals.  
Your job is to ensure they are visible, classified, and actionable.
