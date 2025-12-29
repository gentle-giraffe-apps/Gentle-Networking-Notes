
# Mobile Systems Design – Mental Models

## Mobile as an Unreliable Edge Compute Node

Mobile clients are not thin browsers. They are long‑lived, stateful, battery‑constrained edge compute nodes that must:

- Render complex UI
- Cache and transform data locally
- Synchronize with backend systems opportunistically
- Survive arbitrary termination and resume later

Unlike web clients, mobile apps cannot assume continuous connectivity, stable execution, or instant updates.

---

## Clients Do Not Update Instantly

In production, multiple app versions coexist:

- App Store adoption curves are slow.
- Enterprises may delay updates for months.
- Offline devices may reappear with year‑old schemas.

Your backend must support **multi‑version clients** and tolerate partial capability mismatches.

This implies:

- Backward‑compatible API evolution
- Feature flags & server‑side gating
- Defensive decoding on the client

---

## The Four Failures

Every mobile system should be designed assuming these failures are the default state.

### 1. Network Unavailable
- Radio disabled, airplane mode, captive portals.
- Retry logic must be idempotent.
- Local queues become part of your system design.

### 2. App Suspended
- iOS may freeze or kill your process at any moment.
- Requests may never complete.
- State must be persisted *before* suspension.

### 3. Token Expired
- Access tokens will expire mid‑session.
- Refresh flows must be atomic and thread‑safe.
- Avoid cascading failures from auth races.

### 4. Schema Drift
- Client and server models will diverge.
- Decoders must tolerate unknown fields.
- Breaking changes are outages in slow motion.

---

## The Mobile Triangle

You can only fully optimize **two** of the following:

| Constraint   | Description |
|-------------|-------------|
| Latency     | Time to visible UI |
| Battery     | Energy efficiency |
| Consistency | Accuracy & freshness of data |

Examples:

- Real‑time feeds → Latency + Consistency (battery suffers)
- Offline‑first → Battery + Consistency (latency on sync)
- Aggressive caching → Latency + Battery (eventual consistency)

Good systems design is choosing which corner to sacrifice *intentionally*.

---

## Interview Language

When discussing architecture, use this framing:

> “Our mobile app is an unreliable edge compute node. We design around the four failures and choose where to spend our mobile triangle budget.”

This instantly signals senior‑level systems thinking.
