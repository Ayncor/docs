# Ayncor — Future Services Roadmap

This document proposes **future backend services** Ayncor will likely need as the product matures beyond the current foundation:

- `identity-service` (auth + org + RBAC)
- `core-service` (channels/threads/messages/inbox + outbox events)
- `realtime-gateway` (stateless WebSocket fan-out)

The goal is to keep **clear service boundaries**, preserve **contracts-first** development, and only introduce new services when they reduce complexity, risk, or cost.

---

## 0. Principles for Adding Services

- **Prefer “modular within a service” first**: add a module/package, not a new service, until scaling, ownership, or security demands it.
- **One service = one authority**: each service should have a clear “system of record” for its data.
- **Async is normal**: any expensive/slow work should be done by workers off a durable queue.
- **Realtime is a hint**: continue to reconcile via HTTP APIs; realtime delivery should not become authoritative.
- **Contracts-first**: add OpenAPI + event schema(s) before implementing.

---

## 1. What Exists Today (for context)

### 1.1 Systems of record

- **Identity data**: Postgres owned by `identity-service`
- **Collaboration data** (threads/messages/inbox): Postgres owned by `core-service`
- **Realtime delivery**: `realtime-gateway` consumes Redis pub/sub; no DB

### 1.2 Event bus (current)

- `core-service` uses an **OutboxEvent** table and a **relay** process to publish envelopes to Redis pub/sub (`realtime:events`).
- This is correct for realtime fan-out, but **not** ideal for durable background jobs (retries, scheduling, backpressure).

---

## 2. “Next” Services (Most Likely, High ROI)

### 2.1 Notifications Service

**Why you’ll need it**
- Email + push + in-app notification routing gets complex fast: user preferences, quiet hours, batching/digests, retries.

**Owns**
- Notification preferences (per user/org)
- Delivery attempts + receipts (email provider / push provider IDs)
- Templates (or template references)

**Consumes**
- Core events (message created, thread state changed, needs_response toggles)
- Identity events (membership changed, invites)

**Produces**
- `Notification.Enqueued`
- `Notification.Delivered` / `Notification.Failed`

**Key APIs**
- `GET/PUT /me/notification-preferences`
- `POST /orgs/:orgId/notification-rules` (admin)

**Notes**
- Start with **email digests** and **urgent-only email**.
- Add push later when mobile is live.

---

### 2.2 Jobs / Queue + Workers (Infrastructure Service, not necessarily a “product” service)

**Why you’ll need it**
- Redis pub/sub is not durable. Background work needs retries, scheduling, rate limiting, and observability.

**Owns**
- Job definitions, retry policy, dead-lettering, scheduling

**Options**
- Managed queue: SQS, Cloud Tasks, Cloud Pub/Sub
- OSS: RabbitMQ, NATS JetStream, Redis Streams (durable)  

**Recommendation**
- Introduce a durable queue when you add Notifications, Search indexing, or file processing.

---

### 2.3 Search / Indexing Service

**Why you’ll need it**
- Postgres search works early, but permission-aware, ranked search over large message bodies benefits from a dedicated index.

**Owns**
- Search index (OpenSearch/Elasticsearch/Meilisearch)
- Index pipeline state (last indexed cursor, errors)

**Consumes**
- Core events: message/version created, thread created/state changed, participant changes

**Produces**
- `Search.Indexed` / `Search.IndexFailed` (optional)

**Key APIs**
- `GET /search?q=...&filters...` (usually via core-service façade or API gateway)

**Notes**
- Keep authorization strict: index must be **org-scoped** and permission-filtered.

---

### 2.4 Files / Attachments Service

**Why you’ll need it**
- Attachments demand different storage primitives: object store, antivirus scanning, thumbnails, signed URLs, retention.

**Owns**
- Attachment metadata (who uploaded, which thread/message references it)
- Storage references (bucket/key), size, mime, checksums
- Processing pipeline state (scan status, preview status)

**Uses**
- Object storage: S3/R2/GCS
- AV scanning (e.g., ClamAV)

**Key APIs**
- `POST /uploads/presign` (returns signed URL + upload token)
- `POST /attachments/commit` (finalize, bind to message/thread)
- `GET /attachments/:id` (signed download URL)

**Notes**
- Do not put binaries in Postgres.
- Keep this separate from `core-service` early to avoid mixing concerns.

---

## 3. “Soon” Services (As Product Grows)

### 3.1 Billing / Plans Service

**Why**
- Seat management, entitlements, usage metering, invoices, cancellations.

**Owns**
- Plan selection, subscription state, payment provider IDs (Stripe)
- Entitlements: limits and feature flags per org

**Consumes**
- Identity events (membership created/removed)
- Core usage events (messages created, storage used) if metering is needed

**Key APIs**
- `GET /orgs/:orgId/billing`
- `POST /orgs/:orgId/billing/checkout`

---

### 3.2 Admin / Compliance Service (or Compliance Module that later becomes a service)

**Why**
- Enterprise asks: retention, legal hold, exports, audit dashboards, org-wide policies.

**Owns**
- Retention policies
- Export jobs + results
- Legal hold rules and verification

**Consumes**
- Many domain events across identity and core

**Notes**
- You already have Audit Logs in identity. Expect to expand this area.

---

### 3.3 Analytics / Telemetry Service

**Why**
- Product analytics, funnels, performance tracking, org-level adoption metrics.

**Owns**
- Event ingestion store (warehouse or analytics DB)
- Dashboards + aggregate tables

**Consumes**
- Sanitized product events (ensure strict PII rules)

**Notes**
- Decide early what is allowed to be tracked and how you anonymize.

---

## 4. “Optional / Later” Services (Strategic Add-ons)

### 4.1 AI Service

**Why**
- Summaries, action item extraction, semantic search, smart triage.

**Owns**
- Prompting + models integration
- Redaction policies
- Storage for derived artifacts (summaries) and citations

**Consumes**
- Message/thread data (ideally via jobs + controlled fetch)

**Notes**
- Treat AI outputs as derived, auditable, and revocable.
- Build after you have Notifications + Search + durable queue.

---

### 4.2 API Gateway / Edge Service

**Why**
- Centralized auth checks, rate limiting, request shaping, routing, consistent error responses.

**Notes**
- Not required for MVP; helpful when you add more services or external clients.

---

### 4.3 Presence / Collaboration Enhancements

**Why**
- Presence, typing indicators, online status, richer realtime UX.

**Notes**
- This conflicts with “async-first” if overused; keep it optional and separate.

---

## 5. Suggested Build Order (Phased Roadmap)

### Phase A — Frontend + Stability (now)

- Build web/desktop clients against current APIs
- Strengthen observability (metrics/tracing), tighten contracts, and harden auth flows
- Decide stance on **message encryption** (see Section 6)

### Phase B — Notifications + Durable Jobs (first “new service”)

- Add **durable queue + workers**
- Implement **Notifications service** (email digests + urgent escalation)
- Add notification preferences endpoints

### Phase C — Search + Files

- Add Search indexer + search API (permission-aware)
- Add Attachments service (upload + scan + preview pipeline)

### Phase D — Enterprise + Billing

- Add Billing/Plans (Stripe)
- Add Compliance exports, retention policies

### Phase E — AI

- Add AI worker pipeline for summaries and extraction
- Add UX surface (thread summaries, inbox digests)

---

## 6. Message Encryption Roadmap (Decision Area)

Today, message bodies are stored as plaintext in `core-service` Postgres (application-level). You have three meaningful directions:

### Option 1 — “Standard SaaS” (recommended for now)

- **TLS in transit** (HTTPS/WSS)
- **DB at-rest encryption** via managed Postgres provider (disk encryption)
- Optional: encrypt sensitive columns at the application layer only for specific fields (later)

**Pros**: simplest, fastest iteration, compatible with search and compliance tools.  
**Cons**: not end-to-end encrypted.

### Option 2 — Application-layer encryption at rest (per org or per tenant key)

- Encrypt message bodies before storing, decrypt on read
- Keys stored/managed in KMS; rotate keys; audit key usage

**Pros**: stronger at-rest posture, still server-side searchable only if you add special schemes.  
**Cons**: complicates search, debugging, migrations, and performance.

### Option 3 — End-to-End Encryption (E2EE)

- Server never sees plaintext; clients encrypt/decrypt
- Requires complex key management, multi-device sync, recovery, and limits on server-side search/AI

**Pros**: strongest privacy story.  
**Cons**: product trade-offs (search, AI, compliance), significantly more engineering.

**Recommendation**: Start with Option 1. Revisit after PMF and after you’ve built Search + Notifications (since E2EE strongly impacts both).

---

## 7. Contract Additions Checklist (When Introducing a New Service)

For each new service:

- Add `contracts/v1/openapi/<service>.openapi.yaml`
- Add events under `contracts/v1/events/` (schema + envelope mapping)
- Add a short `docs/<service>-overview.md` with:
  - System of record
  - Key endpoints
  - Key events produced/consumed
  - Failure modes + retries

