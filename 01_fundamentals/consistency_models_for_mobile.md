
# Consistency Models for Mobile Systems

Mobile systems do not live in a single consistency model.
They move between models over time depending on connectivity, power state, and user behavior.

> **Consistency on mobile is a spectrum, not a guarantee.**

---

## Why Classic Consistency Breaks on Mobile

Traditional backend systems often assume:

- Stable connectivity
- Short-lived requests
- Synchronous workflows

Mobile violates all three:

- Users go offline arbitrarily.
- Apps are suspended mid-flight.
- Writes may surface hours later.

This forces us to design for **temporal inconsistency**.

---

## The Mobile Consistency Spectrum

| Model | Description | Example |
|------|-------------|---------|
| Strong | Reads reflect latest writes immediately | In-memory cache after local write |
| Session | Client sees its own writes | Local persistence after enqueue |
| Eventual | System converges over time | Background sync |
| Stale | Reads may be arbitrarily old | Cold launch offline |
| Conflict | Divergent histories exist | Multi-device edits offline |

Your app transitions between these during a single user session.

---

## The Offline-First Principle

Offline-first does **not** mean offline-only.

It means:

- Every user action is recorded locally first.
- Sync is opportunistic.
- UI renders from local state, not network callbacks.

This guarantees liveness even under total network loss.

---

## Write-Behind vs Write-Through

| Strategy | Behavior | Tradeoff |
|----------|----------|----------|
| Write-through | Wait for server before committing locally | Strong consistency, terrible UX |
| Write-behind | Commit locally, sync later | Fast UX, conflict handling required |

Mobile systems should default to **write-behind** for user-generated data.

---

## Conflict Models

Conflicts happen when multiple devices write independently.

Common strategies:

| Strategy | Description |
|----------|-------------|
| Last-write-wins | Simple, lossy |
| Server-authoritative | Server rejects stale writes |
| Merge by field | Domain-aware merge |
| CRDTs | Automatic conflict resolution |

The correct strategy is domain-specific — choose explicitly.

---

## Read-Your-Writes Guarantee

Even in eventual systems, users must see their own changes.

Implementation:

- Apply local mutations immediately.
- Mark records as `pending_sync`.
- Reconcile on server ack.

---

## Sync Windows

Not all data deserves equal urgency.

Examples:

- Critical: bookings, payments → sync immediately
- Soft: analytics, logs → batch later
- Heavy: media uploads → Wi-Fi + charging

Design explicit sync classes.

---

## Consistency vs Battery

Every consistency upgrade costs power:

| Feature | Battery Cost |
|--------|--------------|
| Frequent polling | High |
| Push-based sync | Medium |
| Opportunistic batch | Low |

Your consistency model must be battery-aware.

---

## Key Concept

> “Mobile apps don’t live in one consistency model. They flow between strong, session, and eventual consistency depending on network and lifecycle. Our job is to choose where inconsistency is acceptable and design conflict strategies explicitly.”
