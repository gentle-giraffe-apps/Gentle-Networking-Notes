
# Read & Write Hotspots in Mobile Systems
*Gentle-Networking-Notes · 03_scalability*

> Why mobile traffic is never evenly distributed — and how to design backend & client architecture that survives sudden read/write spikes without melting batteries or databases.

---

## The Hotspot Reality

Mobile usage is bursty:

| Trigger | Result |
|--------|-------|
| Push notification | 100k concurrent opens |
| App Store feature | 10× normal QPS |
| App launch after update | Cold-cache stampede |
| UI animation bug | Infinite refresh loop |

Hotspots are not accidents — they are **product features interacting with human behavior.**

---

## The Two Classes of Hotspots

### Read Hotspots
* Feeds
* Leaderboards
* Home screen dashboards

### Write Hotspots
* Reactions / likes
* Comments
* Telemetry batching
* Presence / typing indicators

---

## Mobile-Specific Amplifiers

| Amplifier | Impact |
|-----------|--------|
| App foregrounding | Synchronized refresh |
| Background fetch limits | Burst retries |
| OS kill/relaunch | Duplicate writes |
| Poor caching | Redownload storms |

---

## Client-Side Dampening

Your app must be the first circuit breaker.

| Technique | Effect |
|-----------|--------|
| In-memory coalescing | Collapses duplicate reads |
| Write batching | Turns 50 writes into 1 |
| Exponential backoff | Stops retry storms |
| UI debounce | Prevents accidental spam |

---

## Backend Load Shaping

Never let mobile clients talk to raw tables.

| Endpoint | Pattern |
|---------|--------|
| `/feed` | Precomputed snapshot |
| `/like` | Append-only queue |
| `/telemetry` | Firehose ingestion stream |

Reads hit caches. Writes hit logs.

---

## Hot Partition Detection

Symptoms:

* Single Redis key 10× hotter than rest
* One DB shard hitting IOPS cap
* p95 latency increasing only at peak hours

Solution: **Partition by time + user + feature**, not by ID alone.

---

## Idempotency Is Mandatory

Mobile retries are guaranteed.

```
POST /like
Idempotency-Key: user123:post456
```

The backend must treat duplicates as success.

---

## When Things Go Wrong

| Failure | User Impact |
|--------|------------|
| Retry storm | Phone heats up |
| Cache stampede | White screens |
| Write amplification | Battery drain |
| Hot shard | Global outage |

---

## Observability

Track:

| Metric | Why |
|-------|-----|
| Writes per session | Battery predictor |
| Read QPS by endpoint | UI risk surface |
| Retry rate by carrier | Network quality |
| Top 10 hottest keys | Partition alarms |

---

## Key Concepts

> “Every mobile hotspot is either a product success or a backend failure. Your job is to make sure it never becomes both.”

---

*Part of Gentle-Networking-Notes · 03_scalability*
