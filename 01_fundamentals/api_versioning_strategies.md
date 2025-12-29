
# API Versioning Strategies (Prefer *Not* Versioning)

Mobile version adoption is slow. Multiple client versions are always in the wild.
So “just bump the API version” often creates long-term complexity: duplicated code paths, fragmented telemetry, and hard-to-kill legacy behavior.

A good default is:

> **Design for backward/forward compatibility first. Version only when you must.**

This article focuses on **capability negotiation** and **schema evolution** practices that reduce or eliminate the need to version a BFF (Backend-for-Frontend).

---

## Mental Model: Tolerant Systems Beat Versioned Systems

Instead of thinking “v1 → v2 → v3”, think:

- **The server is stable and long-lived.**
- **Clients are diverse and lag behind.**
- **The contract is evolutionary, not replaced.**

Your goal is to make safe changes that old clients can ignore and new clients can adopt without breaking anyone.

---

## Prefer Capability Negotiation Over URL Versioning

### What capability negotiation looks like

Clients declare what they can do. The server chooses what to return.

Common approaches:

- **Feature headers**
  - `X-Client-Capabilities: supports_new_pricing, supports_v2_filters`
- **Semantic client metadata**
  - `X-Client-Version: 5.3.1`
  - `X-Platform: iOS`
- **Content negotiation**
  - `Accept: application/json`
  - `Accept: application/vnd.company.resource+json; profile="compact"`
- **Server-driven flags**
  - Server gates behavior based on account, cohort, region, or feature rollout.

### Why headers (or explicit capability fields) help

- Avoids endpoint sprawl (`/v1/...`, `/v2/...`) and duplicated routing.
- Enables progressive rollout and A/B testing.
- Allows the server to keep a single resource model with conditional fields.
- Supports “soft launches” and canary clients without hard branching.

### Best practice: Treat capabilities as a contract

- Capabilities should be **stable strings** (not build numbers).
- Capabilities should be **additive** (you can add a new capability without changing old ones).
- The server should log capabilities for debugging and gradual rollout.

---

## When to Version (The “Break Glass” List)

Versioning can be necessary when you cannot make a change compatible by design.

Typical triggers:

1. **Semantic breaking changes**
   - Existing fields change meaning (e.g., currency units, timezone rules).
2. **Non-additive shape changes**
   - A field that used to be scalar becomes an object and cannot be represented compatibly.
3. **Security or compliance requirements**
   - You must remove sensitive data exposure immediately and old clients require it.
4. **Performance constraints**
   - Old formats are too heavy and you need a new representation that old clients can’t handle.
5. **Contract errors that shipped**
   - A “bad field” is widely adopted and you need a clean replacement with clear migration.

If none of these apply, prefer negotiation + additive evolution.

---

## Server-Side Schema Practices That Avoid Versioning

### 1) Additive changes only
Safe changes:

- Add a new optional field
- Add a new enum case (with client fallback behavior)
- Add new endpoints without removing old ones

Avoid:

- Removing fields
- Renaming fields
- Changing field types
- Changing meaning without a new field

**Rule:** *New meaning → new field name.*

---

### 2) Default values and “missing means default”
Old clients won’t send new fields.

- Server should treat missing fields as defaults.
- Server should validate required invariants but tolerate absent optional data.

---

### 3) Tolerant readers and “unknown fields ignored”
Clients should ignore unknown fields.
Servers should ignore unknown fields if they accept request bodies from clients that may add fields in newer versions.

This is critical for forward-compatibility.

---

### 4) Never rely on field order
JSON has no stable ordering contract.
Be explicit and key-based always.

---

### 5) Prefer stable identifiers over computed labels
Examples:

- `status_code: "IN_FLIGHT"` (stable)
- `status_label: "In flight"` (localized/derived)

Clients should key on stable identifiers; labels should be presentation-only.

---

### 6) Make enums future-proof
Enums break clients when new values appear.

Options:

- Provide an `"UNKNOWN"` / `"other"` fallback on the client.
- Use string enums rather than integer enums.
- Consider using a structured object:
  - `{ "code": "IN_FLIGHT", "label": "In flight" }`

---

### 7) Use explicit “state machines” for lifecycle changes
Instead of overloading one field with meaning, model transitions:

- `state`
- `allowed_transitions`
- `updated_at`

This reduces semantic breakage as workflows evolve.

---

### 8) Use “expansions” instead of always embedding everything
Let clients opt in to heavy fields:

- `GET /resource?id=123&expand=details,owner`
- Or a header: `X-Response-Profile: compact|standard|full`

This avoids making “big response” changes that break slow clients or battery budgets.

---

## BFF-Specific Guidance (Avoid Versioning the BFF)

BFFs often become versioned because they embed product logic and UI assumptions.
To keep a BFF stable:

### 1) Keep BFF responses *semantic*
Return domain concepts, not UI-specific layout assumptions.

Bad (UI-coupled):
- `sectionTitle`, `carouselItems`, `tileStyle`

Better (semantic):
- `recommendations`, `alerts`, `actions`

If UI needs layout, drive it through *separate* config endpoints or feature flags.

---

### 2) Use “server-driven configuration” deliberately
If you must influence UI behavior across versions:

- Prefer a config endpoint with a well-defined schema
- Keep configs additive
- Add “min_client_version” gating for brand-new behaviors
- Avoid using config as a dumping ground for UI rendering logic

---

### 3) Separate *transport* from *domain*
Keep stable internal domain models, then map to representations.

This helps you:
- Add fields safely
- Support multiple representations without duplicating business logic

---

## Deprecation and Migration Playbook

Even if you avoid versioning, you still need planned evolution.

Recommended lifecycle:

1. **Introduce**
   - Add new optional fields/endpoints.
2. **Observe**
   - Measure adoption with analytics/telemetry.
3. **Migrate**
   - Flip feature gates by cohort and gradually ramp.
4. **Deprecate**
   - Mark fields as deprecated in docs and warnings.
5. **Sunset**
   - Stop sending fields first (tolerant readers).
   - Only later stop accepting old fields if applicable.
6. **Remove**
   - Remove code paths after confirmed adoption.

Best practice: publish explicit dates (e.g., `Sunset` header / documentation).

---

## Testing and Observability That Prevent “Accidental Versioning”

### Contract tests
- Validate responses against a schema (OpenAPI/JSON Schema).
- Ensure additive changes don’t break old contract expectations.

### Golden tests for payloads
- Snapshot “compact” vs “full” responses.
- Ensure stable fields remain stable.

### Compatibility test matrix
- Run CI against representative older client contracts if you can.
- At minimum, validate server behavior for “capabilities absent”.

### Telemetry
Log for each request:
- client version
- platform
- declared capabilities
- negotiated response profile

This makes production debugging possible without guessing.

---

## Key Concept

> “A default strategy is to avoid versioning by designing additive schemas and negotiating capabilities via headers. Only version when there's a true semantic break that can’t be represented compatibly.”

