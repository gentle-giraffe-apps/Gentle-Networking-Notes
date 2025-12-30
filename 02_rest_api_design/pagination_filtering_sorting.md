
# Pagination, Filtering, and Sorting for Mobile APIs (Expanded)

## Why This Is Hard on Mobile

Pagination feels simple until you combine:

- Offline usage and partial sync
- App suspension mid-scroll
- Mutating datasets (inserts, deletes, edits)
- Clock skew and timezone drift
- Retries, backoff, and ambiguous request completion

On mobile, naive pagination causes **silent data loss**, **duplicate rows**, and **cache corruption** — often without obvious crashes.

This article covers **cursor types**, **snapshot pagination**, and the mobile-specific gotchas that drive defensive design.

---

## Offset Pagination Is a Trap

```
GET /messages?offset=40&limit=20
```

Offset assumes “row 40” is stable. In real systems, it is not:

- New items inserted at the top shift offsets.
- Deletions collapse the list.
- Sorting keys change (e.g., edits affecting rank).

### Symptoms
- Duplicate items appearing during scroll
- Items “missing” that users later see on refresh
- Infinite scroll loops where the same page repeats

Offset pagination can be acceptable only for truly static datasets (rare) or admin tools (not mobile end-user UX).

---

## Cursor Pagination: The Default for Mobile

Cursor-based pagination uses a token that represents a **position in an ordered set**.

Typical response:

```json
{
  "items": [ ... ],
  "next_cursor": "eyJrZXkiOiJzb3J0X2tleSIsInZhbCI6MTIzfQ==",
  "has_more": true
}
```

The cursor must be built from **stable, deterministic sort keys**.

---

## Stable Sorting: Non-Negotiable

A cursor is meaningless without a deterministic order.

### Recommended pattern
Sort by:

1. A server-generated monotonic timestamp (or sequence)
2. A unique tie-breaker id

Example:

- `ORDER BY created_at DESC, id DESC`

### Why not client clocks?
Clients can be:
- Wrong
- Tampered with
- In different timezones
- Offline for long periods

Server time is authoritative for ordering.

---

## Cursor Types

Different datasets require different cursor strategies. Here are the common cursor types and their tradeoffs.

### 1) Keyset cursor (a.k.a. “seek method”)
Cursor encodes last item’s sort keys.

Example cursor payload:

```json
{ "created_at": "2025-12-29T18:20:00Z", "id": "msg_9812" }
```

Server query becomes:

- For DESC order:
  - `WHERE (created_at, id) < (:created_at, :id)`

**Pros**
- Fast at scale (uses indexes well)
- Stable under inserts and deletes

**Cons**
- Harder to jump to arbitrary pages
- Requires deterministic sort keys

This is the best default for “infinite scroll.”

---

### 2) Opaque cursor
Cursor is a server-issued token that hides internal structure (often signed/encoded).

**Pros**
- Server can evolve cursor internals
- Reduces client coupling
- Can embed snapshot/version info safely

**Cons**
- Harder to debug without tooling
- Requires server-side decode/validation

Best practice: treat cursors as **opaque** even if they’re base64 JSON. Clients should not parse them.

---

### 3) Time-window cursor (“since/until”)
Cursor is a time bound: `since=timestamp` or `until=timestamp`.

**Pros**
- Simple for event streams
- Good for incremental sync

**Cons**
- Collisions when many items share a timestamp
- Requires a tie-breaker id anyway

If you use timestamps, always include `id` as a secondary key.

---

### 4) Offset-with-snapshot (hybrid)
You keep offsets, but lock the dataset to a snapshot id.

**Pros**
- Enables “page numbers” in UX
- More stable than raw offset

**Cons**
- Requires snapshot storage and expiry strategy
- Higher server complexity

If you need page numbers, consider snapshot pagination.

---

## Snapshot Pagination (Paging a Stable View)

Snapshot pagination means: “Page through a consistent view of the dataset as it existed at a point in time.”
This prevents mutation during browsing from causing duplicates/misses.

### How it works
1. Client requests first page with filters/sort.
2. Server responds with:
   - items
   - `snapshot_id`
   - `next_cursor` (within that snapshot)
3. Client includes `snapshot_id` on subsequent pages.

Example:

```http
GET /messages?limit=20&sort=-created_at
```

Response:

```json
{
  "items": [ ... ],
  "snapshot_id": "snap_2f3a9c",
  "next_cursor": "cursor_abc"
}
```

Next request:

```http
GET /messages?limit=20&cursor=cursor_abc&snapshot_id=snap_2f3a9c
```

### What is a snapshot_id?
Common implementations:

- A database transaction snapshot (rare for long-lived browsing)
- A materialized query result keyed by id (expensive but stable)
- A logical version marker (e.g., “as_of” timestamp or index version)
- A search engine scroll context (e.g., Elasticsearch scroll / PIT)

### Expressing snapshot semantics
Include metadata so clients can reason about it:

```json
{
  "snapshot_id": "snap_2f3a9c",
  "snapshot_as_of": "2025-12-29T18:22:10Z",
  "snapshot_ttl_seconds": 300
}
```

This allows the client to:
- Warn users if the snapshot expired
- Decide whether to refresh the list

---

## Indicating “Content Changed While Browsing”

Even without snapshot paging, users need a sane experience if content changes mid-scroll.

### Best practice: include a collection version or watermark
Examples:
- `collection_etag`
- `dataset_version`
- `watermark` (monotonic sequence or timestamp)

First page:

```json
{
  "items": [ ... ],
  "watermark": "wm_192881",
  "next_cursor": "cursor_abc"
}
```

If subsequent page detects mismatch:

```json
{
  "items": [ ... ],
  "watermark": "wm_193102",
  "next_cursor": "cursor_def",
  "notice": {
    "type": "CONTENT_CHANGED",
    "message": "New items available. Refresh to see latest."
  }
}
```

### UX patterns that work
- “New items” pill/banner at top
- “Refresh” affordance that restarts from page 1
- Keep user’s scroll position stable while offering refresh

### Gotcha: don’t auto-insert at top during deep scroll
Auto-inserting items while users scroll down causes:
- Jumping content
- Missed taps
- A feeling of instability

Prefer a user-driven refresh action.

---

## Filtering and Sorting Gotchas

### Filters must be part of the cursor contract
If the client changes filters, **invalidate cursors**.
Cursors are only valid for the exact query shape that created them.

### Sorting must be explicit
Do not rely on “default sort.” Make it part of the API contract.

### Avoid ambiguous sorts
If sort is by a mutable field (like “rank” or “score”), pagination can become unstable.
Consider snapshot paging or using a stable tie-breaker set.

---

## Deletions, Edits, and Tombstones

### Deletions
If items can be deleted while browsing:
- You may see “gaps” or fewer items per page.
- That is okay — do not “compensate” by shifting offsets.

If you need stable list positions, use snapshots.

### Edits that change sort order
Example: “last_updated” sort when editing an item moves it to top.
This can cause duplicates across pages.

Mitigations:
- Use created_at for browsing order
- Put “updated” items in a separate feed
- Use snapshot paging for “ranked” lists

---

## Client Caching: Should Mobile Cache Paginated Results?

Usually **yes**, but with guardrails.

### What to cache
- Items (normalized by id)
- Cursor state per query
- Snapshot_id / watermark metadata
- Last successful fetch time

### Why cache helps
- Fast back navigation
- Offline browsing
- Reduced server load
- Better battery behavior

### What not to cache blindly
- Entire unbounded feeds without eviction
- Raw pages without normalization
- Cursors across query changes

### Recommended cache approach
- Store items in a normalized local store (by id)
- Maintain per-query “index lists” of ids
- Use TTL/eviction strategies:
  - size-based eviction (keep last N items)
  - time-based eviction (e.g., 7 days)
  - context-based eviction (clear when user logs out)

### Cache invalidation strategy
- Use watermarks/etags to detect staleness
- On refresh:
  - fetch page 1
  - reconcile ids
  - optionally prune missing items if the feed is authoritative

---

## Defensive API Response Shape

A “good” paginated response often includes:

```json
{
  "items": [ ... ],
  "next_cursor": "cursor_abc",
  "has_more": true,
  "query": {
    "sort": "-created_at",
    "filters": { "status": ["ACTIVE"] }
  },
  "consistency": {
    "snapshot_id": "snap_2f3a9c",
    "as_of": "2025-12-29T18:22:10Z",
    "ttl_seconds": 300,
    "watermark": "wm_192881"
  }
}
```

Clients can store `consistency` metadata and behave predictably.

---

## Testing and Observability

### Server tests
- Insert/delete between page requests; ensure no duplicates or missing beyond expected semantics.
- Validate cursor decoding and query binding.
- Ensure snapshot expiry behavior is deterministic.

### Client tests
- Simulate backgrounding mid-scroll.
- Resume with cached cursors.
- Validate refresh flows when watermark changes.

### Telemetry to log
- client version
- query hash (filters + sort)
- cursor presence
- snapshot_id presence
- watermark changes

---

## Key Concepts

> “Offset pagination breaks under mutation. We use keyset cursor pagination with stable sort keys, and for ranked or highly mutable datasets we page over a server-issued snapshot (PIT) with an explicit TTL. We surface ‘content changed’ signals via watermarks/etags and let users refresh intentionally. On mobile, we cache normalized results with bounded eviction and query-scoped cursor state.”
