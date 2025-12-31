# Case Study: Realtime Tracking System (Rideshare)

**Audience:** Senior / Staff mobile engineers
**Goal:** Design a production-grade realtime tracking API for a rideshare system — balancing location accuracy, battery life, state synchronization, and offline resilience.

---

## System Overview

A rideshare tracking system connects two mobile clients (rider and driver) through a backend that must:

| Requirement | Challenge |
|-------------|-----------|
| Sub-second location updates | Battery drain, network bandwidth |
| Accurate ETAs | Traffic, GPS drift, prediction models |
| State synchronization | Both parties see consistent trip state |
| Offline resilience | Tunnels, dead zones, poor signal |
| Fraud prevention | Spoofed locations, fare manipulation |

This is one of the most demanding mobile architectures — every decision trades battery against accuracy against latency.

---

## Domain Resources

### Core Entities

```
Rider          → User requesting a ride
Driver         → User providing the ride
Vehicle        → Driver's car (license, capacity, type)
RideRequest    → Rider's intent before matching
Trip           → Matched ride in progress
Location       → Point-in-time position sample
Route          → Planned path with waypoints
ETA            → Estimated arrival (volatile, server-computed)
Fare           → Price calculation (quote → final)
```

### Resource Relationships

```
Rider
 └─ RideRequests[]
     └─ Trip (when matched)
          ├─ Driver
          ├─ Vehicle
          ├─ Route
          ├─ LocationStream (driver)
          ├─ ETAs (pickup, dropoff)
          └─ Fare

Driver
 └─ Trips[]
 └─ CurrentLocation (continuously updated)
 └─ OnlineStatus
```

---

## The Realtime Challenge

### Why HTTP Polling Fails

| Polling Interval | Battery Impact | Latency |
|------------------|----------------|---------|
| 1 second | Catastrophic | Acceptable |
| 5 seconds | High | Noticeable lag |
| 30 seconds | Moderate | Unusable for tracking |

Polling also creates thundering herds — 10,000 active trips polling every second = 10,000 QPS just for location.

### The Solution: Hybrid Protocol

```
┌─────────────┐      WebSocket       ┌─────────────┐
│   Driver    │◄────────────────────►│   Backend   │
│    App      │   (location push)    │             │
└─────────────┘                      │             │
                                     │             │
┌─────────────┐      WebSocket       │             │
│   Rider     │◄────────────────────►│             │
│    App      │   (location receive) │             │
└─────────────┘                      └─────────────┘
        │                                   │
        │         REST (fallback)           │
        └───────────────────────────────────┘
```

| Channel | Use Case |
|---------|----------|
| WebSocket | Location streams, state changes, ETAs |
| REST | Initial load, history, actions (cancel, rate) |
| Push Notification | App backgrounded, critical state changes |

---

## API Shape

### Ride Request Creation

```http
POST /ride-requests
Headers:
  Idempotency-Key: r1d3-...
Body:
{
  "pickup": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Market St, San Francisco",
    "place_id": "pl_abc123"
  },
  "dropoff": {
    "latitude": 37.8044,
    "longitude": -122.2712,
    "address": "456 Broadway, Oakland",
    "place_id": "pl_def456"
  },
  "ride_type": "STANDARD",
  "passenger_count": 2,
  "payment_method_id": "pm_xyz"
}
```

Response:

```json
{
  "request_id": "req_8a7b",
  "state": "SEARCHING",
  "allowed_transitions": ["CANCELLED"],
  "fare_estimate": {
    "min_amount_minor": 2400,
    "max_amount_minor": 3200,
    "currency": "USD",
    "surge_multiplier": 1.2
  },
  "estimated_pickup_seconds": 180,
  "search_radius_meters": 2000,
  "created_at": "2025-03-14T18:30:00Z",
  "expires_at": "2025-03-14T18:35:00Z"
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `place_id` alongside lat/lng | Stable reference for address; lat/lng for routing |
| `fare_estimate` as range | Honest about variability; prevents disputes |
| `surge_multiplier` | Transparent pricing; client can show warning |
| `expires_at` | Request times out if no match; prevents zombie requests |
| `search_radius_meters` | Client can show expanding search animation |

---

### WebSocket Connection

```
wss://api.example.com/realtime?token={access_token}
```

#### Connection Lifecycle

```json
// Client → Server: Subscribe to trip
{
  "type": "subscribe",
  "channel": "trip:tr_4e5f",
  "last_event_id": "evt_123"
}

// Server → Client: Subscription confirmed
{
  "type": "subscribed",
  "channel": "trip:tr_4e5f",
  "replay_from": "evt_120"
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `last_event_id` | Resume after reconnect without missing events |
| Token in query param | WebSocket headers unreliable across platforms |
| Channel-based subscription | Client subscribes only to relevant trips |

---

### Location Update (Driver → Server)

```json
// Driver app sends every 1-4 seconds
{
  "type": "location",
  "payload": {
    "latitude": 37.7751,
    "longitude": -122.4183,
    "accuracy_meters": 5.2,
    "heading": 45.0,
    "speed_mps": 12.5,
    "altitude_meters": 15.0,
    "timestamp": "2025-03-14T18:32:15.123Z",
    "battery_percent": 72,
    "is_charging": false
  },
  "sequence": 1847
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `accuracy_meters` | Server can filter low-quality samples |
| `heading` + `speed_mps` | Enables smooth client-side interpolation |
| `timestamp` (client) | Detect clock skew, order out-of-sequence |
| `battery_percent` | Server can adjust update frequency |
| `sequence` | Detect packet loss, replay attacks |

**Gotcha:** Never trust client timestamps for ordering. Use sequence numbers and server receipt time as authoritative.

---

### Location Broadcast (Server → Rider)

```json
{
  "type": "driver_location",
  "event_id": "evt_456",
  "payload": {
    "latitude": 37.7751,
    "longitude": -122.4183,
    "heading": 45.0,
    "speed_mps": 12.5,
    "interpolation_hint": {
      "duration_ms": 2000,
      "easing": "linear"
    },
    "server_timestamp": "2025-03-14T18:32:15.200Z"
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| Reduced payload | Rider doesn't need accuracy/altitude/battery |
| `interpolation_hint` | Client animates smoothly between updates |
| `server_timestamp` | Client knows data freshness |
| `event_id` | Resumable streams after reconnect |

**Gotcha:** Raw location updates create jerky map movement. The `interpolation_hint` tells the client to animate over 2 seconds, smoothing the visual experience.

---

### ETA Updates

```json
{
  "type": "eta_update",
  "event_id": "evt_457",
  "payload": {
    "pickup_eta": {
      "seconds": 145,
      "confidence": "HIGH",
      "computed_at": "2025-03-14T18:32:15Z"
    },
    "dropoff_eta": {
      "seconds": 1280,
      "confidence": "MEDIUM",
      "computed_at": "2025-03-14T18:32:15Z"
    },
    "route_polyline": "encoded_polyline_string...",
    "traffic_conditions": "MODERATE"
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| ETA in seconds (not timestamp) | Avoids timezone/clock issues |
| `confidence` level | UI can show "~3 min" vs "3 min" |
| `computed_at` | Client knows staleness |
| `route_polyline` | Draw updated route on map |

**Gotcha:** ETAs should only update when meaningfully changed (>30 sec delta). Constant micro-updates create anxiety and UI flicker.

---

### Trip State Machine

```
┌──────────────┐
│  REQUESTED   │ (rider waiting for match)
└──────┬───────┘
       │ driver accepts
       ▼
┌──────────────┐
│   ACCEPTED   │ (driver assigned, en route to pickup)
└──────┬───────┘
       │ driver arrives
       ▼
┌──────────────┐
│   ARRIVED    │ (driver waiting at pickup)
└──────┬───────┘
       │ rider enters vehicle
       ▼
┌──────────────┐
│  IN_PROGRESS │ (trip underway)
└──────┬───────┘
       │ arrive at dropoff
       ▼
┌──────────────┐
│  COMPLETED   │ (fare finalized)
└──────────────┘

Any state can transition to:
┌──────────────┐
│  CANCELLED   │ (by rider, driver, or system)
└──────────────┘
```

#### State Change Event

```json
{
  "type": "trip_state_changed",
  "event_id": "evt_460",
  "payload": {
    "trip_id": "tr_4e5f",
    "previous_state": "ACCEPTED",
    "state": "ARRIVED",
    "allowed_transitions": ["IN_PROGRESS", "CANCELLED"],
    "triggered_by": "DRIVER",
    "timestamp": "2025-03-14T18:35:00Z",
    "context": {
      "arrival_location": {
        "latitude": 37.7749,
        "longitude": -122.4194
      }
    }
  }
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `previous_state` | Client can validate transition was expected |
| `allowed_transitions` | UI shows valid actions only |
| `triggered_by` | Audit trail; explains who caused change |
| `context` | State-specific metadata |

---

### Trip Details (REST Fallback)

```http
GET /trips/{trip_id}
Headers:
  If-None-Match: "trip-v12"
```

Response:

```json
{
  "trip_id": "tr_4e5f",
  "state": "IN_PROGRESS",
  "allowed_transitions": ["COMPLETED", "CANCELLED"],
  "rider": {
    "id": "usr_rider",
    "first_name": "Jane",
    "rating": 4.92
  },
  "driver": {
    "id": "usr_driver",
    "first_name": "Alex",
    "rating": 4.87,
    "photo_url": "https://cdn.example.com/...",
    "vehicle": {
      "make": "Toyota",
      "model": "Camry",
      "color": "Silver",
      "license_plate": "ABC123"
    }
  },
  "pickup": {...},
  "dropoff": {...},
  "route": {
    "polyline": "encoded...",
    "distance_meters": 12400,
    "duration_seconds": 1280
  },
  "fare": {
    "estimate_min_minor": 2400,
    "estimate_max_minor": 3200,
    "currency": "USD"
  },
  "created_at": "2025-03-14T18:30:00Z",
  "etag": "trip-v13"
}
```

**When to use REST vs WebSocket:**

| Scenario | Channel |
|----------|---------|
| App launch / cold start | REST (full state) |
| Reconnect after network loss | WebSocket (resume from event_id) |
| Background → foreground | REST (validate state) then WebSocket |
| Action (cancel, rate) | REST |

---

## Battery-Aware Location Strategy

### Driver App (High Frequency)

| State | GPS Interval | Accuracy | Battery Impact |
|-------|--------------|----------|----------------|
| Online, no trip | 30 sec | City block | Low |
| En route to pickup | 2 sec | High | High |
| Trip in progress | 4 sec | High | Medium-High |
| Idle > 5 min | 60 sec | Low | Minimal |

```swift
struct LocationConfig {
    let desiredAccuracy: CLLocationAccuracy
    let distanceFilter: CLLocationDistance
    let updateInterval: TimeInterval
    let allowsBackgroundUpdates: Bool
}

func configForTripState(_ state: TripState) -> LocationConfig {
    switch state {
    case .enRouteToPickup:
        return LocationConfig(
            desiredAccuracy: kCLLocationAccuracyBest,
            distanceFilter: 10,
            updateInterval: 2,
            allowsBackgroundUpdates: true
        )
    case .inProgress:
        return LocationConfig(
            desiredAccuracy: kCLLocationAccuracyNearestTenMeters,
            distanceFilter: 20,
            updateInterval: 4,
            allowsBackgroundUpdates: true
        )
    // ...
    }
}
```

### Rider App (Low Frequency)

Riders only need location for:
- Setting pickup point
- "Find me" accuracy improvement

| State | GPS Usage |
|-------|-----------|
| Browsing | None |
| Setting pickup | On-demand, single shot |
| Tracking driver | None (server provides driver location) |

**Gotcha:** Never continuously track rider location during a trip. They're watching the driver, not reporting their own position.

---

## Offline Resilience

### Connection State Machine

```
┌──────────────┐
│  CONNECTED   │◄──────────────────┐
└──────┬───────┘                   │
       │ connection lost           │ reconnected
       ▼                           │
┌──────────────┐                   │
│ DISCONNECTED │───────────────────┘
└──────┬───────┘
       │ backoff timer
       ▼
┌──────────────┐
│  RECONNECTING│───► (retry with exponential backoff)
└──────────────┘
```

### Offline Capabilities

| Feature | Offline Behavior |
|---------|------------------|
| View trip details | Cached state |
| See driver location | Last known + staleness indicator |
| Cancel trip | Queued mutation |
| Contact driver | Phone call (native) |
| View driver info | Cached |

### Reconnection Protocol

```json
// On reconnect, client sends:
{
  "type": "reconnect",
  "last_event_id": "evt_456",
  "trip_id": "tr_4e5f",
  "client_state": "IN_PROGRESS"
}

// Server responds with:
{
  "type": "state_sync",
  "current_state": "IN_PROGRESS",
  "missed_events": [
    {"event_id": "evt_457", "type": "driver_location", ...},
    {"event_id": "evt_458", "type": "eta_update", ...}
  ],
  "latest_driver_location": {...}
}
```

**Design decisions:**

| Choice | Rationale |
|--------|-----------|
| `client_state` sent | Server detects stale client, forces full refresh if needed |
| `missed_events` replay | Client catches up without polling |
| `latest_driver_location` | Always include current position regardless of event history |

---

## Fraud Prevention

### Location Spoofing Detection

| Signal | Detection |
|--------|-----------|
| Impossible speed | > 200 km/h between samples |
| Teleportation | > 1 km jump in < 5 seconds |
| Mock location flag | Android `isFromMockProvider()` |
| Jailbreak detection | iOS integrity checks |
| GPS accuracy anomalies | Consistent 0.0m accuracy |

```json
// Server flags suspicious location
{
  "type": "location_warning",
  "payload": {
    "warning_code": "ACCURACY_ANOMALY",
    "action": "NONE",
    "message": "Location quality degraded"
  }
}
```

### Fare Manipulation Prevention

| Attack | Mitigation |
|--------|------------|
| Driver takes long route | Server computes optimal route; flag deviations |
| GPS spoofing for distance | Cross-reference with expected route |
| Rider disputes completed trip | Full location history audit trail |

---

## Error Taxonomy (Tracking-Specific)

### WebSocket Errors

| Error | Recovery |
|-------|----------|
| Auth token expired | Close socket, refresh token, reconnect |
| Rate limited | Back off per server instruction |
| Invalid subscription | Likely stale trip_id; fetch via REST |
| Server closing | Reconnect with backoff |

### Domain Errors

| Code | Meaning | Recovery |
|------|---------|----------|
| `TRIP_NOT_FOUND` | Invalid or expired trip | Clear local state |
| `TRIP_ALREADY_CANCELLED` | Race condition | Refresh state |
| `DRIVER_UNAVAILABLE` | Match failed | Show "no drivers" UX |
| `LOCATION_REQUIRED` | Can't verify pickup | Request location permission |
| `SURGE_CHANGED` | Price changed during request | Show new price, reconfirm |

---

## Caching Strategy

### What to Cache

| Data | Storage | TTL |
|------|---------|-----|
| Trip details | Disk | Until trip ends + 24h |
| Driver/vehicle info | Disk | Trip duration |
| Route polyline | Memory | Until route changes |
| Location history | Memory (ring buffer) | Last 100 points |
| ETA | Memory | Until next update |

### What Never to Cache

| Data | Reason |
|------|--------|
| Driver's current location | Must be realtime |
| Fare calculations | Must be server-authoritative |
| Trip state | Source of truth is server |

---

## Push Notification Strategy

### When to Push (App Backgrounded)

| Event | Push? | Content |
|-------|-------|---------|
| Driver accepted | Yes | "Alex is on the way in a silver Camry" |
| Driver arrived | Yes (high priority) | "Your driver has arrived" |
| Trip completed | Yes | "Trip complete. Rate your ride?" |
| Driver cancelled | Yes | "Your driver cancelled. Finding another..." |
| ETA changed significantly | Maybe | Only if > 5 min change |

### Silent Push for State Sync

```json
{
  "aps": {
    "content-available": 1
  },
  "trip_id": "tr_4e5f",
  "event_type": "state_changed",
  "new_state": "ARRIVED"
}
```

App wakes briefly to update local state, ensuring accuracy when user opens app.

---

## Rate Limiting

### Driver Location Uploads

| Condition | Max Rate |
|-----------|----------|
| Trip in progress | 1/sec |
| En route to pickup | 1/sec |
| Online, no trip | 1/30sec |
| Burst allowance | 5 rapid updates then throttle |

### Rider Requests

| Endpoint | Limit |
|----------|-------|
| Create ride request | 3/min |
| Cancel trip | 5/min |
| Get trip details | 30/min |
| WebSocket messages | 10/sec |

---

## Testing Strategy

### Chaos Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| WebSocket dies mid-trip | Auto-reconnect, replay missed events |
| Driver enters tunnel | Rider sees "Last updated X ago" |
| Rider backgrounds app | Silent push updates state |
| Server returns stale ETA | Client shows last computed_at |
| GPS gives bad accuracy | Server filters, client interpolates |

### Simulation Harness

```swift
// Inject fake location stream for testing
protocol LocationProvider {
    var locations: AsyncStream<CLLocation> { get }
}

class SimulatedLocationProvider: LocationProvider {
    func simulateDriveAlongRoute(_ route: Route, speed: Double) -> AsyncStream<CLLocation>
}
```

### Latency Injection

| Test | Simulated Condition |
|------|---------------------|
| High latency network | 2-5 second delays on WebSocket |
| Packet loss | Drop 20% of location updates |
| Reconnection storm | Kill and reconnect every 30 sec |

---

## Observability

### Client Metrics

```swift
struct RealtimeMetrics {
    let connectionUptime: TimeInterval
    let reconnectCount: Int
    let messagesReceived: Int
    let messagesDropped: Int
    let averageLatencyMs: Double
    let locationUpdatesRendered: Int
    let interpolationGlitches: Int  // visual jumps
}
```

### Key Dashboards

| Metric | Alert Threshold |
|--------|-----------------|
| WebSocket connection success rate | < 95% |
| Message delivery latency p95 | > 500ms |
| Reconnection rate | > 10% of sessions |
| Location staleness (rider view) | > 10 sec average |
| GPS accuracy (driver uploads) | > 50m average |

---

## Common Gotchas Summary

| Gotcha | Mitigation |
|--------|------------|
| Jerky map animation | Interpolation hints between updates |
| Battery drain from GPS | Adaptive accuracy per trip state |
| Stale UI after reconnect | Event replay from last_event_id |
| ETA anxiety from micro-changes | Only update on meaningful delta |
| Spoofed driver locations | Server-side anomaly detection |
| WebSocket auth expiry | Token refresh before expiry, reconnect |
| Rider tracking their own location | Don't—they're watching the driver |
| Thundering herd on reconnect | Jittered reconnection backoff |
| Lost state on app kill | Persist trip_id, restore via REST |

---

## Key Concepts

> "Realtime tracking is a battery budget negotiation. The driver trades power for accuracy; the rider trades freshness for smoothness. WebSockets eliminate polling overhead but require robust reconnection with event replay. Every location update must flow through fraud detection before reaching the rider's map. The system must degrade gracefully — a rider in a tunnel should see 'last updated 30 seconds ago' with the driver's interpolated position, not a spinner. When in doubt, prioritize the moment the rider looks at their phone: they should instantly see where their driver is, even if that answer is slightly stale."
