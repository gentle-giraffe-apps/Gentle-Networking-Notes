# Mocking vs Record & Replay for Mobile Networking Tests

**Audience:** Mobile engineers (Senior / Staff)  
**Goal:** Choose the right isolation strategy to test networking code realistically without brittle test suites.

---

## Background

Mobile networking tests must balance **speed, determinism, and realism**. Two dominant strategies exist:

- **Mocking** — replacing dependencies with hand-written fakes.
- **Record & Replay** — capturing real network traffic and replaying it offline.

Each solves different problems and each can cause subtle production blind spots.

---

## Mocking

Mocking replaces the networking layer with deterministic, developer-authored behavior.

### Strengths

- Extremely fast
- Fully deterministic
- Easy to trigger rare error paths

### Weaknesses

| Issue | Impact |
|------|--------|
| Drift from reality | Production-only failures |
| Overfitting tests | Tests encode implementation details |
| Serialization blind spots | Decoding crashes in prod |

---

## Record & Replay

Traffic is captured once from real servers and replayed locally.

### Strengths

- Real payload shapes
- Schema drift detection
- High confidence decoding safety

### Weaknesses

| Issue | Impact |
|------|--------|
| Stale fixtures | Tests mask breaking changes |
| Sensitive data | Requires sanitization |
| Slow evolution | Hard to update intentionally |

---

## Pattern — Layered Strategy

| Layer | Strategy |
|------|----------|
| Domain logic | Mocking |
| Serialization | Record & Replay |
| Integration harness | Hybrid |

Use mocking for logic, record & replay for safety nets.

---

## Pattern — Fixture Freshness Budget

Define expiration policies for recorded data.

| Fixture Age | Action |
|------------|--------|
| < 7 days | Safe |
| 7–30 days | Warning |
| > 30 days | Regenerate |

---

## Common Gotchas

| Symptom | Root Cause |
|--------|-----------|
| CI failures after API change | Stale recordings |
| Green tests, red prod | Over-mocking |
| Huge repos | Unsanitized recordings |

---

## Key Follow-Up Questions

**Q: When should I never mock?**  
A: Serialization boundaries and schema validation.

**Q: How do you regenerate safely?**  
A: Automated capture pipelines with redaction.

**Q: How do you avoid leaking secrets?**  
A: Redact tokens and personal data before commit.

---

## One-Page Summary

| Rule | Why |
|------|-----|
| Mock logic | Speed |
| Replay payloads | Safety |
| Expire fixtures | Freshness |
| Hybrid harness | Balance realism |
| Sanitize recordings | Compliance |

---

## Two-Sentence Summary

Mocking makes tests fast, record & replay makes them truthful.  
Production reliability requires both.
