
# Resource Modeling for Mobile APIs

## Introduction

Resource modeling is the quiet foundation of every successful mobile backend. Most production outages blamed on
“networking bugs” are in reality **resource modeling failures** — APIs shaped around database tables, internal
microservice boundaries, or short‑term UI needs instead of long‑lived mobile behavior.

Mobile clients are not browsers. They are:

- Offline‑capable edge compute nodes
- Version‑skewed for months or years
- Battery and bandwidth constrained
- Stateful across launches, crashes, and OS kills

Your resource model must survive those realities.

---

## The Mobile Contract Mindset

Treat your API as a **long‑term contract**, not a transport layer.

### Backend‑centric thinking (dangerous)

- “Expose what the database stores.”
- “Rename this field, the app will update.”
- “We’ll version the endpoint later.”

### Mobile‑centric thinking (durable)

- “What meaning must survive for 3 years?”
- “What happens when a 9‑month‑old client comes back online?”
- “How does this model behave when half the data is missing?”

Once you adopt the second mindset, many design decisions become obvious.

---

## Stable Identity Over Presentation

### Bad pattern

```json
{
  "status": "Boarding",
  "color": "#22C55E"
}
```

This binds business meaning to presentation.

### Durable pattern

```json
{
  "status_code": "BOARDING",
  "status_label": "Boarding"
}
```

Clients key off `status_code`, not `status_label`.

### Why this matters

- Labels change.
- Localization changes.
- Styling changes.

Identifiers must not.

---

## Avoid Boolean Explosion

Booleans rot quickly.

```json
{
  "is_active": true,
  "is_cancelled": false,
  "is_locked": true
}
```

Six months later nobody knows which combinations are legal.

### Replace with explicit state machines

```json
{
  "state": "LOCKED",
  "allowed_transitions": ["CANCELLED"]
}
```

Now your domain is inspectable and evolvable.

---

## Additive Evolution as a Rule

Mobile APIs must be **additive by default**.

### Safe changes

- Add optional fields
- Add enum cases
- Add new resources

### Breaking changes

- Rename fields
- Change types
- Change semantics

**Rule:** If meaning changes, create a new field.

---

## Modeling Partial State

Offline systems produce **incomplete objects**.

Design your schema assuming:

- Required fields may be absent
- Related resources may not exist yet
- Writes may be queued for hours

### Example: Draft objects

```json
{
  "id": "temp-123",
  "title": "Flight note",
  "sync_state": "PENDING"
}
```

This allows UI to proceed without backend confirmation.

---

## Resource Granularity

### Over‑fragmentation

- `/flight`
- `/flight-status`
- `/flight-metadata`
- `/flight-owner`

Each network hop drains battery.

### Over‑aggregation

One mega‑endpoint returning everything destroys caching and versioning.

### Balance

Group by **cohesion of change**, not storage location.

---

## Expansion Instead of Embedding

Allow clients to opt into heavy fields.

```
GET /flight?id=123&expand=crew,history
```

This avoids versioning every time payload weight changes.

---

## Defensive Field Design

| Field Type | Guideline |
|-----------|------------|
| Enums | Always support unknown |
| Dates | ISO‑8601, never locale |
| IDs | String, never integer |
| Flags | Prefer states |
| Money | Always currency + minor units |

---

## BFF Anti‑Patterns

Backend‑for‑Frontend layers rot when they encode UI layout.

### Bad

```json
{
  "carouselTitle": "...",
  "tiles": [...]
}
```

### Good

```json
{
  "recommendations": [...],
  "alerts": [...]
}
```

Layout belongs in the app.

---

## Migration Without Versioning

1. Add new fields.
2. Ship client support.
3. Observe telemetry.
4. Deprecate old fields.
5. Remove only after months.

Never remove first.

---

## Key Concept

> “Our resource model is additive, identity‑driven, and tolerant of partial state. We treat breaking changes as outages, not refactors.”
