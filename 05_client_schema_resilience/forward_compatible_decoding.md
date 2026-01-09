# Forward‑Compatible JSON Decoding on Mobile Clients (Swift)

> Goal: design a Swift decoding layer that **does not crash**, **does not silently corrupt meaning**, and **keeps working** as server payloads evolve over months/years—across phased rollouts, A/B tests, partial deploys, and buggy backends.

This guide focuses on **professional-grade policies** for decoding, schema drift hardening, graceful degradation, observability, and test strategies. Examples use `Codable`, but the principles apply to any decoder.

---

## Table of contents

1. [The mental model: decoding is a resilience boundary](#the-mental-model-decoding-is-a-resilience-boundary)
2. [Classes of schema drift you must assume will happen](#classes-of-schema-drift-you-must-assume-will-happen)
3. [Decoding policies: the rules your team agrees to](#decoding-policies-the-rules-your-team-agrees-to)
4. [Foundation patterns (with production-ready Swift examples)](#foundation-patterns-with-production-ready-swift-examples)
   - [Resilient enums with `unknown(String)`](#resilient-enums-with-unknownstring)
   - [Unknown fields: ignore vs capture](#unknown-fields-ignore-vs-capture)
   - [Safe defaults vs optionals](#safe-defaults-vs-optionals)
   - [Flexible date decoding](#flexible-date-decoding)
   - [Number/string polymorphism](#numberstring-polymorphism)
   - [Lossy arrays and partial failure](#lossy-arrays-and-partial-failure)
   - [Tolerant nested decoding](#tolerant-nested-decoding)
   - [Backwards compatibility for renamed fields](#backwards-compatibility-for-renamed-fields)
   - [Semantic versioning inside payloads](#semantic-versioning-inside-payloads)
5. [Graceful degradation strategies](#graceful-degradation-strategies)
6. [Observability: detect drift early and safely](#observability-detect-drift-early-and-safely)
7. [Unit tests for schema drift](#unit-tests-for-schema-drift)
8. [Clean architecture placement: where this code lives](#clean-architecture-placement-where-this-code-lives)
9. [Enforcing best practices (SwiftLint, CI, LLM checks)](#enforcing-best-practices-swiftlint-ci-llm-checks)
10. [Common gotchas and failure modes](#common-gotchas-and-failure-modes)
11. [Key concepts (cheat sheet)](#key-concepts-cheat-sheet)

---

## The mental model: decoding is a resilience boundary

On mobile, you do not control:

- server deploy timing,
- cache layers (CDNs, gateways),
- partial rollouts,
- experiments injecting new enum values or fields,
- backwards compatibility mistakes,
- “hotfix” responses under incident pressure.

**Therefore decoding is not parsing.**  
It is a **fault-tolerant boundary** between untrusted input and your app’s internal model.

Treat the server payload as an *external protocol* you must handle defensively—like you would handle file formats or network packets.

---

## Classes of schema drift you must assume will happen

You can’t “prevent” drift; you can only **survive** it. Most real incidents fall into one of these:

### Additive changes (most common)
- New fields added
- New enum cases introduced
- New object members inside nested structures

**Client risk:** crashes only if you decode strictly (e.g., throwing on unknown enum case).

### Subtractive changes
- Field removed
- Enum case removed (server stops sending it)

**Client risk:** logic expecting it might break; decoding often survives if optional/defaulted.

### Renames
- `first_name` → `given_name`
- `status` → `state`

**Client risk:** suddenly empty UI or behavior changes.

### Type changes (very common in incident hotfixes)
- `"42"` (String) becomes `42` (Int)
- `null` shows up where a string was expected
- `"true"` becomes `true`

**Client risk:** `DecodingError.typeMismatch` if not hardened.

### Semantics change (the most dangerous)
- Field stays same type but meaning changes  
  - `status="active"` used to mean “paid”, now means “trial”  
  - `price` changes from dollars to cents  

**Client risk:** no decoding error—only wrong behavior. Needs policy + monitoring + versioning.

### Partial failure / partial data
- List has some corrupt elements
- Optional subobject missing only sometimes
- Server returns mixed shapes due to a bad join or cache poisoning

**Client risk:** strict arrays fail and drop everything.

---

## Decoding policies: the rules your team agrees to

Write these down and enforce them. Here’s a strong baseline policy set:

1. **Unknown enum values must not throw.**  
   Use `unknown(String)` (or `unknown(Int)` when appropriate).
2. **Decoding should prefer partial success over total failure** for collections.  
   One bad element should not drop the entire feed.
3. **Defaults must be explicit and documented.**  
   If you default `isEnabled` to `false`, that’s a product decision.
4. **Optionals represent “absence is meaningful”.**  
   Use them sparingly; avoid “optional soup”.
5. **Decoding failures must be observable** (metrics + logs, sampled).  
   Silent failures accumulate into product bugs.
6. **Domain models must not depend on server field names.**  
   Map API DTOs → domain entities.
7. **Schema drift tests must exist** and include “future” payloads (unknown fields/cases).
8. **Don’t swallow errors globally** (e.g., `try? decoder.decode(...)` at boundary).  
   Capture and classify failures.

---

## Foundation patterns (with production-ready Swift examples)

### Resilient enums with `unknown(String)`

This is the single highest-leverage pattern.

```swift
public enum PaymentStatus: Equatable, Sendable, Decodable {
    case pending
    case completed
    case failed
    case unknown(String)

    public init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let raw = try container.decode(String.self)

        switch raw {
        case "pending":   self = .pending
        case "completed": self = .completed
        case "failed":    self = .failed
        default:          self = .unknown(raw)
        }
    }
}
```

**Why store the raw value?**
- Enables telemetry (“we are seeing `paused` from server”)
- Enables safe UI fallback (“Status: paused”)
- Helps debugging and migration decisions

**Gotcha:** When encoding back, decide if you want to preserve `unknown`.  
If the enum is request/response shared, implement `Encodable` carefully.

```swift
extension PaymentStatus: Encodable {
    public func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        switch self {
        case .pending: try container.encode("pending")
        case .completed: try container.encode("completed")
        case .failed: try container.encode("failed")
        case .unknown(let raw): try container.encode(raw) // or encode("unknown")
        }
    }
}
```

**Recommendation:** For request payloads, prefer *separate* request DTO enums without `unknown`  
(or validate unknowns before sending).

---

### Unknown fields: ignore vs capture

Swift `Decodable` ignores unknown keys by default. That’s good, but sometimes you want to **capture unknown keys** for debugging.

A pragmatic compromise:
- Ignore unknown keys in production decoding
- Log unknown keys in debug/staging builds or sampled in prod

You can capture keys via a custom decoder wrapper, but that’s heavier than many teams need. An easier strategy: log the raw response body when decoding fails (sampled), and use contract tests to detect unexpected additions that matter.

**Rule of thumb:** capture unknown keys only if:
- you frequently debug backend rollouts,
- you are shipping an SDK,
- or the domain is regulated/high-stakes.

---

### Safe defaults vs optionals

Avoid making everything optional. It makes downstream code fragile.

Prefer:
- decode optional **from server**
- map to **non-optional domain** with explicit default policy

Example:

```swift
struct UserDTO: Decodable {
    let id: String
    let displayName: String?
    let isVerified: Bool?
}

struct User: Equatable, Sendable {
    let id: String
    let displayName: String
    let isVerified: Bool
}

extension User {
    init(dto: UserDTO) {
        self.id = dto.id
        self.displayName = dto.displayName ?? "Unknown"
        self.isVerified = dto.isVerified ?? false // explicit product decision
    }
}
```

**Gotcha:** Defaults can hide server bugs. If `isVerified` suddenly disappears, you may silently mark everyone unverified.  
Mitigate with observability: count missing fields at mapping time (sampled).

---

### Flexible date decoding

Dates are a classic drift source: ISO 8601 variants, milliseconds vs seconds, timezone quirks.

Create a reusable `JSONDecoder` factory:

```swift
enum DecoderFactory {
    static func make() -> JSONDecoder {
        let decoder = JSONDecoder()

        // Key strategy is domain dependent:
        decoder.keyDecodingStrategy = .convertFromSnakeCase

        decoder.dateDecodingStrategy = .custom { decoder in
            let container = try decoder.singleValueContainer()

            // Try ISO-8601 string first
            if let str = try? container.decode(String.self) {
                // ISO8601DateFormatter is not thread-safe if mutated; create local
                let f = ISO8601DateFormatter()
                f.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
                if let d = f.date(from: str) { return d }

                // Retry without fractional seconds
                let f2 = ISO8601DateFormatter()
                f2.formatOptions = [.withInternetDateTime]
                if let d = f2.date(from: str) { return d }

                throw DecodingError.dataCorruptedError(
                    in: container,
                    debugDescription: "Invalid ISO8601 date: \(str)"
                )
            }

            // Try numeric timestamps (seconds or milliseconds)
            if let num = try? container.decode(Double.self) {
                // Heuristic: milliseconds are usually > 10^12 for modern dates
                if num > 1_000_000_000_000 {
                    return Date(timeIntervalSince1970: num / 1000.0)
                } else {
                    return Date(timeIntervalSince1970: num)
                }
            }

            throw DecodingError.dataCorruptedError(
                in: container,
                debugDescription: "Unsupported date format"
            )
        }

        return decoder
    }
}
```

**Policy tip:** If date parsing fails, decide whether that is:
- fatal for that object, or
- recoverable with `nil` + telemetry

For feeds, prefer recoverable.

---

### Number/string polymorphism

Backends often send `"1"` or `1` depending on code path. Harden at the DTO layer.

```swift
struct StringOrInt: Decodable, Equatable, Sendable {
    let value: Int

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        if let i = try? container.decode(Int.self) { self.value = i; return }
        if let s = try? container.decode(String.self),
           let i = Int(s.trimmingCharacters(in: .whitespacesAndNewlines)) {
            self.value = i; return
        }
        throw DecodingError.typeMismatch(
            Int.self,
            .init(codingPath: decoder.codingPath, debugDescription: "Expected Int or numeric String")
        )
    }
}
```

Use the wrapper in DTOs, not in domain.

---

### Lossy arrays and partial failure

One corrupt element should not destroy an entire page of results.

A reliable way to “advance” the unkeyed container on failure is to decode a `RawJSON` value and discard it.

```swift
enum RawJSON: Decodable {
    case object([String: RawJSON])
    case array([RawJSON])
    case string(String)
    case number(Double)
    case bool(Bool)
    case null

    init(from decoder: Decoder) throws {
        let c = try decoder.singleValueContainer()
        if c.decodeNil() { self = .null; return }
        if let b = try? c.decode(Bool.self) { self = .bool(b); return }
        if let n = try? c.decode(Double.self) { self = .number(n); return }
        if let s = try? c.decode(String.self) { self = .string(s); return }
        if let a = try? c.decode([RawJSON].self) { self = .array(a); return }
        if let o = try? c.decode([String: RawJSON].self) { self = .object(o); return }
        throw DecodingError.dataCorruptedError(in: c, debugDescription: "Unknown JSON")
    }
}

struct LossyArray<Element: Decodable>: Decodable {
    var elements: [Element]

    init(from decoder: Decoder) throws {
        var container = try decoder.unkeyedContainer()
        var result: [Element] = []

        while !container.isAtEnd {
            do {
                result.append(try container.decode(Element.self))
            } catch {
                _ = try? container.decode(RawJSON.self) // discard invalid element, advances
                // In production: record drop + sampled error/codingPath
            }
        }

        self.elements = result
    }
}
```

---

### Tolerant nested decoding

A common failure: nested object shape changes; you still want the parent object.

Use optional decode with error capture (but pair with telemetry):

```swift
extension KeyedDecodingContainer {
    func decodeSafely<T: Decodable>(_ type: T.Type, forKey key: Key) -> T? {
        do { return try decode(T.self, forKey: key) }
        catch { return nil } // record sampled error + codingPath
    }
}
```

Example DTO:

```swift
struct PaymentDetailsDTO: Decodable {
    let method: String
    let last4: String?
}

struct OrderDTO: Decodable {
    let id: String
    let status: PaymentStatus
    let paymentDetails: PaymentDetailsDTO?

    enum CodingKeys: String, CodingKey { case id, status, paymentDetails }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)
        self.id = try c.decode(String.self, forKey: .id)
        self.status = (try? c.decode(PaymentStatus.self, forKey: .status)) ?? .unknown("missing")
        self.paymentDetails = c.decodeSafely(PaymentDetailsDTO.self, forKey: .paymentDetails)
    }
}
```

**Policy warning:** `decodeSafely` should be used intentionally. If overused, you hide real breakages. Pair it with metrics.

---

### Backwards compatibility for renamed fields

When server renames fields, decode from multiple keys:

```swift
struct ProfileDTO: Decodable {
    let givenName: String?
    let familyName: String?

    enum CodingKeys: String, CodingKey {
        case givenName
        case familyName
        // legacy keys
        case firstName
        case lastName
    }

    init(from decoder: Decoder) throws {
        let c = try decoder.container(keyedBy: CodingKeys.self)

        self.givenName =
            (try? c.decode(String.self, forKey: .givenName)) ??
            (try? c.decode(String.self, forKey: .firstName))

        self.familyName =
            (try? c.decode(String.self, forKey: .familyName)) ??
            (try? c.decode(String.self, forKey: .lastName))
    }
}
```

**Policy:** Keep legacy decoding for *at least* one full minimum supported app version window.

---

### Semantic versioning inside payloads

If you can influence backend contracts, add:

- `schemaVersion` at top-level or per object family
- or `apiVersion` header
- or `"type": "v2"` discriminator

Example:

```swift
struct EnvelopeDTO<T: Decodable>: Decodable {
    let schemaVersion: Int
    let data: T
}
```

Then you can:
- branch decoding/mapping by version,
- emit telemetry when unexpected versions appear.

---

## Graceful degradation strategies

When decoding succeeds but you have unknowns:

### UI fallbacks
- If status is unknown: show “Updating…” or “Unavailable”
- For unknown enum case: show generic icon + label
- Hide feature toggles for unknown capability flags

### Feature safety
- If a capability flag is missing/unknown: default to **off**
- For monetary amounts: if parsing uncertain, do not display “$0”; show “—”

### Data preservation
When you receive unknown enum/raw JSON, consider storing it (safe, size-limited) for debugging, especially for:
- payments,
- booking,
- state machines,
- compliance flows.

---

## Observability: detect drift early and safely

Your goal: **know about drift before users tell you**.

### Metrics to capture (sampled)
- Unknown enum case counts by endpoint
- Missing critical fields (e.g., `id`, `status`, `amount`)
- Decoding failures by `DecodingError` type
- Lossy array drop rate (how many elements skipped)
- Per-version drift (app version, OS version)

### Where to hook instrumentation

At the boundary:

```swift
protocol DecodeTelemetry: Sendable {
    func recordUnknownEnum(type: String, raw: String, endpoint: String)
    func recordDecodingFailure(endpoint: String, error: Error)
    func recordLossyDrop(endpoint: String)
}
```

Inject telemetry into your API client / repository layer so DTO decoding can report without importing analytics frameworks in the DTO module.

### Never log PII
- Do not log raw bodies by default.
- If you must, do it only in debug or via secure redaction.

### Sampling strategy
- log first N events per session per endpoint,
- or 1% sampling in production.

---

## Unit tests for schema drift

Tests should prove that:

1. unknown enum values don’t crash
2. new fields don’t break decoding
3. arrays survive partial corruption
4. renamed fields remain compatible
5. date formats are robust
6. semantics changes are detectable (via mapping tests + invariants)

### Test helpers

```swift
import XCTest

final class DecodingTests: XCTestCase {
    func decode<T: Decodable>(_ type: T.Type, json: String) throws -> T {
        let data = Data(json.utf8)
        return try DecoderFactory.make().decode(T.self, from: data)
    }
}
```

### Example: unknown enum case

```swift
func test_paymentStatus_unknownValue_isCaptured() throws {
    struct DTO: Decodable { let status: PaymentStatus }
    let dto = try decode(DTO.self, json: #"{"status":"paused"}"#)

    XCTAssertEqual(dto.status, .unknown("paused"))
}
```

### Example: new field doesn’t break

```swift
func test_newServerField_doesNotBreak() throws {
    struct DTO: Decodable { let id: String }
    _ = try decode(DTO.self, json: #"{"id":"123","newField":{"x":1}}"#)
}
```

### Example: lossy array drops only bad elements

```swift
func test_lossyArray_skipsCorruptElements() throws {
    struct Item: Decodable { let id: String }
    struct DTO: Decodable { let items: LossyArray<Item> }

    let dto = try decode(DTO.self, json: #"{"items":[{"id":"a"},{"id": 7},{"id":"b"}]}"#)

    XCTAssertEqual(dto.items.elements.map(\.id), ["a","b"])
}
```

### Contract tests with golden payloads

Maintain “golden” JSON fixtures:
- `*_v1.json`
- `*_v2_with_new_enum.json`
- `*_with_nulls.json`
- `*_with_type_flip.json`

Run them in CI to prevent regressions.

**Pro tip:** Add a fixture named `future_payload.json` that contains:
- unknown enum values,
- extra nested fields,
- mixed date formats,
- number/string flips.

It forces resilience to remain intentional.

### Property-based drift tests (optional but powerful)
Generate random payload variations (nulls, extra keys, unknown enum strings) and assert:
- decoder never crashes unexpectedly,
- mapping invariants hold (e.g., `id` must be non-empty).

---

## Clean architecture placement: where this code lives

A clean separation (common in staff-level codebases):

### Modules / layers

- **Networking module**
  - HTTP client, interceptors, retries
- **API DTO module**
  - `Decodable` structs/enums matching server
  - tolerant decoding helpers
- **Mapping layer**
  - DTO → Domain conversion
  - defaults + invariants + semantic translation
- **Domain module**
  - non-optional entities, business logic
- **Presentation**
  - UI fallbacks for unknown states

### Where specific code belongs

| Concern | Where |
|--------|------|
| `JSONDecoder` configuration | Networking module (factory) |
| `unknown(String)` enums | DTO module |
| `LossyArray`, `RawJSON`, `StringOrInt` | DTO module (shared decoding utils) |
| Defaulting policies | Mapping layer |
| Telemetry protocol | Domain boundary or Networking (protocol) |
| Analytics implementation | App layer / Composition root |

**Key rule:** Domain models shouldn’t know about snake_case keys or raw enum strings.

---

## Enforcing best practices (SwiftLint, CI, LLM checks)

### SwiftLint ideas (practical rules)

1. **Ban `try? JSONDecoder().decode` in production paths**  
   Require explicit error handling + telemetry.
2. **Ban `enum Foo: String, Codable` for response enums** (unless it has `unknown`)  
   Use custom `init(from:)` for resilience.
3. **Ban force unwraps in decoding code** (`!`).
4. **Require DTO naming convention** (`*DTO`) and mapping to domain types.

SwiftLint can’t express all semantic rules natively, but you can use:
- `custom_rules` with regex for simple patterns,
- or a lightweight SwiftSyntax-based linter for stronger enforcement.

### CI guardrails

- Run unit tests with fixture corpus
- Run a “schema drift suite” target
- Track coverage for decoding utilities

### LLM-assisted checks (safe, practical use)

Use an LLM in CI for **review assistance**, not for pass/fail on its own.

Good uses:
- detect suspicious patterns: “optional soup”, missing unknown enum handling
- summarize what changed in DTOs for reviewers
- propose fixture updates when DTOs change

Bad uses:
- “approve PR” automatically
- block merges based solely on LLM output

**Recommended setup:**
- CI step generates a report:
  - new/changed DTOs,
  - added enums lacking unknown case,
  - added required fields without defaults
- LLM comments on PR with suggestions, but does not decide pass/fail.

---

## Common gotchas and failure modes

### 1) Unknown enums causing hard crashes
If you use:

```swift
enum Status: String, Decodable { case pending, completed }
```

Then `"paused"` throws and can crash flows if you don’t handle it.

**Fix:** `unknown(String)` custom decoding.

### 2) `decodeIfPresent` is not “safe”
`decodeIfPresent` still throws on type mismatch; it only handles missing/null.

If server sends `{"count":"7"}` but you do `decodeIfPresent(Int.self, ...)`, you still fail.

**Fix:** wrappers like `StringOrInt`, or `decodeSafely`.

### 3) `convertFromSnakeCase` surprises
It can mangle acronyms and edge cases (`userID` vs `userId`).

**Policy:** prefer explicit `CodingKeys` for critical DTOs.

### 4) Default values hiding backend regressions
Defaults reduce crashes but can create silent correctness issues.

**Mitigate:** metrics + alerts, mapping invariants, schema versioning.

### 5) Lossy decoding hides serious data issues
Lossy arrays are great for feeds, not for transactions.

**Policy:** apply lossy decoding only to non-critical collections and always record drop counts.

### 6) Storing raw unknown values can leak PII
If unknown enum includes user-provided string, be careful.

**Policy:** classify which unknowns are safe to log; redact others.

### 7) “Semantics drift” needs product + backend alignment
You cannot code your way out of meaning changes without:
- versioning,
- explicit migration,
- validation invariants.

---

## Key concepts (cheat sheet)

- **Decoding is a resilience boundary**: treat server JSON as untrusted input.
- **Unknown enums never throw**: use `unknown(String)` and preserve raw values.
- **Prefer partial success for feeds**: `LossyArray` + telemetry.
- **Separate DTOs from domain**: mapping layer owns defaults and invariants.
- **Explicit defaults are product decisions**: document them and observe missing fields.
- **Handle type flips**: wrappers like `StringOrInt`, flexible date decoding.
- **Be careful with optionals**: optional means absence is meaningful, not “I don’t want to decide.”
- **Detect drift early**: metrics for unknowns, decode failures, lossy drop rates.
- **Test for the future**: fixtures with unknown fields/cases, mixed types, nulls.
- **Enforce with tooling**: SwiftLint custom rules + CI drift suite; LLMs for review assistance, not gating.
