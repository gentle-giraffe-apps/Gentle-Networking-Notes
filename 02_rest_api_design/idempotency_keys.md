
# Idempotency Keys — Best Practices for REST APIs (with iOS Client Examples)

Idempotency keys protect users from **duplicate side effects** when network retries occur
(timeouts, flaky connectivity, app restarts). They are essential for POST endpoints that
create or finalize business actions.

---

## What is an Idempotency Key?

A client-generated unique identifier (usually UUID) sent with a mutating request:

```
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
```

If the same request is retried with the same key, the server returns the **original result**
instead of executing the action again. It is typically added to the header.

---

## When to Use

Use idempotency keys on endpoints that:

- Create money movement (payments, tips, refunds)
- Create bookings or orders
- Finalize workflows (checkout, submit application, accept ride, cancel with fee)
- Any action where duplication is harmful

Avoid for:
- GET
- PUT-style "set state to X" updates
- Telemetry / heartbeats

---

## Responsibilities

### Client (iOS)

- Generate **once per user intent**
- Reuse the same key for retries
- Persist key for long-running or critical flows
- Never regenerate on network retry

### Server

- Store `(user_id, endpoint, idempotency_key)` → response mapping
- TTL minutes–days depending on domain
- If key reused with different payload → return `409 Conflict`

---

## iOS Client Pattern

### Generate at User Intent Layer

```swift
struct PendingOperation: Codable {
    let idempotencyKey: String
    let createdAt: Date
    let payloadHash: String
}
```

```swift
func createRideRequest(parameters: RideRequest) async throws -> Ride {
    let key = UUID().uuidString
    let operation = PendingOperation(
        idempotencyKey: key,
        createdAt: Date(),
        payloadHash: parameters.hashValue.description
    )
    pendingStore.save(operation)

    return try await api.createRide(parameters, idempotencyKey: key)
}
```

### Networking Layer

```swift
func createRide(_ request: RideRequest, idempotencyKey: String) async throws -> Ride {
    var urlRequest = URLRequest(url: endpoint)
    urlRequest.httpMethod = "POST"
    urlRequest.addValue(idempotencyKey, forHTTPHeaderField: "Idempotency-Key")
    urlRequest.httpBody = try JSONEncoder().encode(request)

    let (data, _) = try await session.data(for: urlRequest)
    return try JSONDecoder().decode(Ride.self, from: data)
}
```

### Retry Flow

Retries must **reuse the same idempotency key**:

```swift
retry {
    try await createRide(parameters, idempotencyKey: operation.idempotencyKey)
}
```

---

## Server Behavior

| Scenario | Result |
|--------|--------|
| New key | Process request, persist response |
| Same key, same payload | Return original response |
| Same key, different payload | `409 Conflict` |

---

## TTL Guidelines

| Domain | TTL |
|-------|-----|
| Payments / bookings | 24–72 hours |
| Checkout / ride request | 1–6 hours |
| Generic mutations | 10–60 minutes |

---

## Interview Soundbite

> "For any POST that creates or finalizes state, the client generates an `Idempotency-Key`
once per user intent and reuses it across retries. The backend stores the result keyed by
that value with a TTL and safely returns the same response on replay."

---
