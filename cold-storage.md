# Ayncor — Cold storage / message tiering blueprint

This document describes how Ayncor can introduce **cold storage** for messages in the future to reduce Postgres cost and operational load while keeping the product UX intact.

**Status:** Not implemented.

**Related docs:**

- [future-services-roadmap.md](./future-services-roadmap.md) (roadmap placement)
- [system-architecture.md](./system-architecture.md) (current systems)
- [e2ee.md](./e2ee.md) (encryption models; interacts with archival + search)

---

## 1. Definitions (what “cold storage” means here)

### Hot storage

“Hot” means message data lives in the primary OLTP store (`core-service` Postgres) and is served with normal latency:

- Threads, messages, versions, reactions, per-user state, inbox queries.

### Cold storage

“Cold” means older, rarely accessed message payloads are moved out of Postgres into **object storage** (S3 / R2 / GCS), potentially using cheaper archive classes for multi-year retention.

**Important:** Cursor pagination reduces **bandwidth and viewport latency**, but does **not** reduce long-term DB size. Cold storage does.

---

## 2. When to adopt (decision triggers)

Adopt cold storage when one or more become true:

- Postgres **storage + backups** are dominated by historical message bodies/versions.
- Restore time / migration time / index size becomes painful due to “ancient” data.
- You need long retention (years) but don’t need millisecond OLTP reads for all history.
- Legal/compliance requires durable retention while keeping the OLTP “working set” small.

Until then, prefer:

- Cursor pagination (already implemented)
- Proper indexing
- (Later) Postgres partitioning on time ranges

---

## 3. Guiding principles (do’s and don’ts)

- **Do not** add speculative columns now (“cold_ref”, “tier”, etc.) without a real plan and migration window.
- **Do not** put binaries in Postgres (attachments should go to object storage earlier).
- **Do** keep APIs stable: tiering should be transparent to clients if possible.
- **Do** run archival via **durable jobs** (queue + workers) — retryable, idempotent, observable.
- **Do** keep one authority: `core-service` remains system-of-record for message metadata and permissions.

---

## 4. What exactly gets archived?

Start with the highest ROI payload:

- **`MessageVersion.body`** (the message text) and optionally `format`

Usually keep in Postgres:

- Message metadata and routing: ids, org_id, thread_id, created_at, author ids, urgency, reply_to, etc.
- Version numbers and timestamps

Avoid archiving early:

- Anything needed by inbox ordering or unread computations (should remain hot or cheaply queryable).

---

## 5. Architecture options

### Option A — Transparent cold reads (server fetches from object storage)

- API remains unchanged.
- When `GET /messages/thread/:threadId` encounters a cold version, `core-service` fetches its body from object storage and returns plaintext to authorized clients.

**Pros:** no client changes; easiest product story.  
**Cons:** old-history reads become slower; requires object-store access on read path; caching needed.

### Option B — Rehydrate-on-demand (server rehydrates cold to warm/hot)

- First access to old history triggers a job to rehydrate a chunk back into Postgres or a warm cache.

**Pros:** better repeated-read latency.  
**Cons:** more moving pieces; needs durable jobs; risk of “thundering herd” on popular old threads.

### Option C — Explicit “archive endpoint”

- Introduce dedicated endpoints like `GET /threads/:id/archive?...` for old data.

**Pros:** clear latency semantics.  
**Cons:** client complexity; not very Slack-like.

**Recommendation:** start with **Option A** once you adopt cold storage (transparent).

---

## 6. Data format in cold storage

### Chunking strategy

Pick one and standardize:

- **Per-message-version object**: simplest addressing; more objects.
- **Per-thread time-chunk objects**: fewer objects; needs chunk index.

### Suggested blob format (v1)

- JSON is simplest. If volume gets high, migrate to Parquet later.

Example JSON blob:

```json
{
  "schema_version": 1,
  "org_id": "uuid",
  "thread_id": "uuid",
  "message_id": "uuid",
  "message_version_id": "uuid",
  "version": 3,
  "created_at": "2026-01-26T12:00:00.000Z",
  "format": "MARKDOWN",
  "body": "plaintext-or-encrypted-body"
}
```

**Encryption note:** whether `body` is plaintext or encrypted depends on your encryption model (see [e2ee.md](./e2ee.md)).

---

## 7. End-to-end flow (copy → verify → truncate)

Cold tiering must be safe and reversible.

### 7.1 Worker selection query

Examples:

- “Archive all message versions older than N days in orgs on plan X”
- “Archive messages in threads that are ARCHIVED and older than N days”

### 7.2 Write to object storage

- Compute object key (include org/thread/message/version identifiers).
- Write blob.
- Capture: `etag`/version id, size, checksum (sha256), written_at.

### 7.3 Verify before mutating Postgres

- Read back (or verify `etag` + checksum) to ensure write succeeded.

### 7.4 Mutate Postgres

Two common models:

1. **Truncate body in place**
   - Set `MessageVersion.body = NULL` (or empty)
   - Set `storage_tier = COLD`
   - Set `cold_ref = <bucket/key/version>`
   - Keep version row for ordering/history

2. **Move to archive table**
   - Copy to `MessageVersionArchive` (or similar)
   - Remove body from primary row

**Recommendation:** model (1) is simpler if your ORM + queries can handle nullable bodies cleanly.

---

## 8. Read path changes (what breaks if you do nothing)

Once bodies can be absent in Postgres, you must update:

- `GET /messages/thread/:threadId` (latest_version.body may need fetching)
- `GET /messages/:messageId/versions`
- Inbox preview (if you keep it): today it uses substring of body

If you keep previews, you have two options:

- **Decrypt/fetch** preview on demand (slow, but acceptable for old threads)
- Keep a minimal **non-sensitive preview** hot (but that is still data leakage)

---

## 9. Search + cold storage (interactions)

- If you plan to build **Search service**, you should decide whether:
  - you index message bodies **before** archiving, and keep search results independent of tier, or
  - you allow search only over hot window (simpler, but weaker UX)

Cold storage doesn’t automatically solve search. In fact, it often pushes you toward a real search index.

---

## 10. Where code changes will be required (when implementing)

### `core-service`

- Prisma schema changes (tier + pointer + null body policy)
- Message list + versions endpoints
- Inbox preview logic
- Operational tooling (backfill, verify, repair)

### Jobs / queue + workers (new infra)

- Durable queue
- Worker deployable (could live inside `core-service` repo but deployed separately)
- Retry policies + DLQ

### Infra

- Object storage bucket + IAM
- Lifecycle policies (hot → cool → archive)
- Metrics and alarms

### `contracts`

- Public APIs can remain unchanged if reads are transparent.
- Optional new events: `Core.MessageVersionArchived`, `Core.MessageRehydrated` (only if useful).

---

## 11. What we should do now (provisioning without overbuilding)

Do now:

- Keep message model immutable + versioned (already true).
- Keep list/read logic centralized in services (already true).
- Ensure cursor pagination exists (done).
- Capture this blueprint so the migration isn’t a “hack later.”

Do not do now:

- Do not add columns or worker code without triggers + queue + search plan.

---

## 12. Summary recommendation

- **Defer** cold storage implementation until **durable jobs** exist and metrics justify it.
- When you do implement, prefer **transparent reads** with a safe **copy → verify → truncate** worker.
- Decide search strategy up front so you don’t paint yourself into a corner.
