# Ayncor — System Architecture

Comprehensive architecture documentation for the Ayncor async-first team communication platform.

---

## 1. Product Philosophy

> **Async by default. Realtime as a hint.**

Ayncor is built on the principle that most team communication does not need to be instantaneous. Urgency is explicit and rare. Threads are first-class units of work; messages are immutable records. The system prioritizes correctness, determinism, and enterprise-grade security over real-time-first patterns.

---

## 2. High-Level Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              CLIENTS                                         │
│                                                                              │
│     ayncor-client (monorepo)            ayncor-mobile (separate repo)        │
│  ┌─────────────┐  ┌──────────────┐     ┌───────────────────────────┐         │
│  │   Web App   │  │  Desktop App │     │ iOS + Android (RN)        │         │
│  │ (Next.js)   │  │ (Electron /  │     │                           │         │
│  │             │  │   Tauri)     │     │                           │         │
│  └─────────────┘  └──────────────┘     └───────────────────────────┘         │
└──────────────────────────────────────────────────────────────────────────────┘
           │ HTTP                │ HTTP                   │ HTTP
           │ REST                │ REST                   │ REST
           │ Bearer JWT          │ Bearer JWT             │ Bearer JWT
           │ WebSocket           │ WebSocket              │ WebSocket
           ▼                     ▼                        ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           BACKEND SERVICES                                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ identity-service  :3001                                                │  │
│  │ Auth · Users · Orgs · Memberships · RBAC · Invites · Audit · Health    │  │
│  │ ── NestJS · TypeScript · Prisma · PostgreSQL ────────────────────────  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ core-service  :3002                                                    │  │
│  │ Channels · Threads · Messages · Versions · Reactions · Participants    │  │
│  │ Per-user thread state · Inbox · Outbox (event relay)                   │  │
│  │ ── NestJS · TypeScript · Prisma · PostgreSQL ────────────────────────  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ realtime-gateway  :3010                                                │  │
│  │ WebSocket fan-out · JWT auth · Org/topic subscriptions                 │  │
│  │ Stateless · Horizontally scalable                                      │  │
│  │ ── Express · ws · TypeScript ───────────────────────────────────────   │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │ Outbox Relay  (core-service subprocess)                                │  │
│  │ Polls OutboxEvent in Postgres → publishes to Redis pub/sub             │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
                              ┌──────────────────┐
                              │ Redis pub/sub    │
                              │ realtime:events  │
                              └──────────────────┘
```

---

## 3. Repos and Their Roles

| Repo | Role | Port / Runtime |
|------|------|----------------|
| `identity-service` | Auth + identity authority | HTTP :3001 |
| `core-service` | Collaboration data + events | HTTP :3002 |
| `realtime-gateway` | WebSocket fan-out | WS/HTTP :3010 |
| `contracts` | Source of truth (OpenAPI + event schemas + realtime protocol) | published as `@ayncor/contracts` |
| `ayncor-client` | Web (Next.js) + Desktop (Electron/Tauri) monorepo | — |
| `ayncor-mobile` | iOS + Android (Expo/React Native) | — |
| `ayncor-e2e` | Cross-repo E2E tests | — |

---

## 4. Service Responsibilities

### 4.1 identity-service

The **sole authority** for anything identity-related.

**Owns:**
- Users (`User` — email, displayName, avatarUrl, status, emailVerifiedAt)
- Organizations (`Organization` — name, slug, allowedEmailDomains)
- Memberships (`Membership` — user+org join, role, status: ACTIVE/INVITED/SUSPENDED/LEFT)
- Roles (`Role` — orgId, permissions[], isSystem: ORG_ADMIN/ORG_MEMBER or custom)
- Refresh tokens (`RefreshToken` — tokenHash, device/IP metadata, rotation state)
- Org invites (`OrgInvite` — tokenHash, email, expiry, lifecycle: accepted/declined/revoked)
- Audit logs (`AuditLog` — every sensitive action with stable action name codes)

**Key behaviors:**
- Signup (`POST /orgs/signup`) — creates User + Org + admin Membership atomically; optional team invites
- Login (`POST /auth/login`) — org-scoped (must supply org_slug); returns access + refresh tokens
- Refresh (`POST /auth/refresh`) — token rotation; reuse detection revokes all tokens
- Logout / Logout-all — individual or bulk token revocation
- Invite flow — create invite → verify token (public) → accept → decline (all public except create/list/revoke)
- RBAC — system roles ORG_ADMIN/ORG_MEMBER; custom roles; permissions embedded in JWT claims

**JWT claims:**
```
{
  sub: user_id
  org_id: organization_id
  membership_id: membership_id
  role_id: role_id (optional)
  perms: ["org:read", "channels:manage", ...]
  jti: trace_id (optional)
}
```

**Does NOT own:** channels, threads, messages, inbox, realtime logic, notification delivery.

---

### 4.2 core-service

The **system of record for collaboration data**. Org and user context comes only from the JWT.

**Owns:**
- `Channel` — org-scoped, visibility (ORG/PRIVATE), slug
- `Thread` — first-class entity, belongs to channel, has state machine
- `Message` — immutable, belongs to thread, has kind (TEXT/SYSTEM), urgency, requires_response
- `MessageVersion` — append-only edit history; version increments from 1
- `Reaction` — toggle per (user, message, emoji); soft-delete via removedAt
- `ThreadParticipant` — per-thread membership with role (OWNER/PARTICIPANT/OBSERVER) and mute
- `ThreadUserState` — per-user per-thread workflow state (in_inbox, archived, snoozed + priority + last_read)
- `OutboxEvent` — transactional outbox for event relay to Redis

**Thread state machine:**
```
OPEN → BLOCKED → OPEN     (can recover)
OPEN → DECIDED            (closed outcome)
OPEN / BLOCKED / DECIDED → ARCHIVED   (terminal, irreversible)
```

**Message flow:**
1. `POST /messages` creates immutable Message + MessageVersion v1 in same transaction
2. `POST /messages/:id/versions` creates MessageVersion v2, v3, etc. (edits are append-only)
3. `GET /messages/thread/:threadId?limit=&cursor=` returns a page of messages (newest first) with latest version embedded; `next_cursor` loads older messages

**Inbox logic:**
- Only `ThreadUserState.status = IN_INBOX` threads surface in inbox
- Ordering: `priority_override` (HIGH > LOW > NONE) → `needs_response` → `last_activity_at desc`
- `unread_count` = messages after `last_read_message_id` when set; else total message count

**Does NOT own:** auth, users, orgs, realtime transport, notification delivery.

---

### 4.3 realtime-gateway

**Delivery only**. No business logic, no database, no event production.

**Responsibilities:**
- Accept WebSocket connections (`ws://host:3010`)
- Authenticate connection via JWT (query param `?access_token=<jwt>` or first message `{type:"auth"}`)
- Accept `subscribe`/`unsubscribe` frames for `topic: inbox | thread`
- Fan out events from Redis pub/sub to matching connected clients

**Trust model:** WebSocket events are **hints**, not authoritative. Clients must always reconcile via HTTP REST calls.

**Allowlisted event types forwarded (v1):**
- `Core.MessageCreated`
- `Core.MessageVersionCreated`
- `Core.ThreadCreated`
- `Core.ThreadStateChanged`
- `Core.ReactionAdded`
- `Identity.MembershipChanged` (optional access-change hint)

**Routing:**
- All events → route by `org_id`
- Topic `thread` → additionally route by `thread_id` from envelope payload/entity_ref

---

### 4.4 Outbox Relay (core-service subprocess)

Decouples database writes from Redis publishing for reliability.

```
1. core-service writes domain record + OutboxEvent in same Postgres transaction
2. Relay polls OutboxEvent where status=PENDING, ordered by createdAt ASC
3. Relay publishes envelope JSON to Redis channel (realtime:events)
4. Relay marks row PUBLISHED with publishedAt timestamp
5. realtime-gateway subscriber receives event → fans out to WebSocket clients
```

**Key properties:**
- Domain write and outbox append are **atomic** (same Prisma transaction)
- Relay is a separate process (`npm run relay` in core-service)
- Configurable: `RELAY_POLL_MS` (default 500ms), `RELAY_BATCH_SIZE` (default 100)
- Redis channel: `REDIS_CHANNEL` env (default `realtime:events`)

---

## 5. Data Models

### 5.1 identity-service database

```
User
├── id (uuid PK)
├── email (unique)
├── displayName
├── avatarUrl?
├── status (ACTIVE | SUSPENDED | DELETED)
├── emailVerifiedAt?
├── passwordHash + passwordSalt
└── createdAt / updatedAt / deletedAt

Organization
├── id (uuid PK)
├── name
├── slug (unique) ← Team URL: {slug}.ayncor.com
├── status (ACTIVE | SUSPENDED | DELETED)
├── allowedEmailDomains[] ← Domain restriction
└── createdAt / updatedAt / deletedAt

Role
├── id (uuid PK)
├── orgId (FK → Organization)
├── name (unique per org)
├── permissions[] ← e.g. ["org:read", "channels:manage"]
├── isSystem (true for ORG_ADMIN, ORG_MEMBER)
└── createdAt / updatedAt

Membership
├── id (uuid PK)
├── orgId (FK → Organization)
├── userId (FK → User)
├── roleId? (FK → Role)
├── status (ACTIVE | INVITED | SUSPENDED | LEFT)
├── joinedAt?
├── invitedByUserId?
└── createdAt / updatedAt

RefreshToken
├── id (uuid PK)
├── tokenHash (unique) ← SHA256 of actual token
├── userId / orgId / membershipId
├── expiresAt / revokedAt / replacedById
├── userAgent? / ipAtIssue?
├── lastUsedAt? / lastUsedFromIp?
└── createdAt

OrgInvite
├── id (uuid PK)
├── orgId / email / roleId? / invitedByUserId?
├── tokenHash (unique) ← token shown only once
├── expiresAt
└── acceptedAt? / declinedAt? / revokedAt?

AuditLog
├── id (uuid PK)
├── orgId (FK)
├── actorUserId?
├── action ← stable code (identity.auth.login, identity.auth.signup, …)
├── targetType / targetId?
├── metadata (JSON)
└── createdAt
```

### 5.2 core-service database

```
Channel
├── id (uuid PK)
├── orgId / name / slug (unique per org)
├── description? / visibility (ORG | PRIVATE)
├── createdByUserId?
└── createdAt / updatedAt / archivedAt?

Thread
├── id (uuid PK)
├── orgId / channelId (FK → Channel)
├── state (OPEN | BLOCKED | DECIDED | ARCHIVED)
├── title / purpose?
├── createdByUserId? / createdByMembershipId?
├── lastActivityAt ← updated on every message/reaction
└── createdAt / updatedAt / archivedAt?

Message
├── id (uuid PK)
├── orgId / threadId (FK → Thread)
├── authorUserId? / authorMembershipId?
├── kind (TEXT | SYSTEM)
├── urgency (NORMAL | URGENT)
├── requiresResponse (bool)
├── replyToMessageId? / metadataJson
└── createdAt / deletedAt?

MessageVersion
├── id (uuid PK)
├── orgId / messageId (FK → Message)
├── version (int, 1-based)
├── body / format (PLAIN | MARKDOWN)
├── editorUserId?
└── createdAt

Reaction
├── id (uuid PK)
├── orgId / messageId (FK → Message)
├── emoji
├── actorUserId? / actorMembershipId?
├── createdAt / removedAt? ← soft delete

ThreadParticipant
├── id (uuid PK)
├── orgId / threadId (FK → Thread) / userId
├── role (OWNER | PARTICIPANT | OBSERVER)
├── joinedAt / leftAt?
└── mutedUntil?

ThreadUserState
├── id (uuid PK)
├── orgId / threadId / userId
├── status (IN_INBOX | ARCHIVED | SNOOZED)
├── needsResponse (bool)
├── snoozedUntil?
├── lastReadMessageId?
├── lastReviewedAt?
├── priorityOverride (NONE | LOW | HIGH)
└── createdAt / updatedAt

OutboxEvent
├── id (uuid PK)
├── envelope (JSON) ← full EventEnvelopeV1
├── status (PENDING | PUBLISHED)
├── createdAt / publishedAt?
```

---

## 6. Event Architecture

Events use a **transactional outbox pattern**. Events are never lost due to process crashes between DB write and publish.

### 6.1 Event Envelope schema

```json
{
  "event_id": "uuid",
  "event_type": "Core.MessageCreated",
  "schema_version": 1,
  "occurred_at": "ISO-8601",
  "org_id": "uuid",
  "actor_user_id": "uuid | null",
  "trace_id": "uuid | null",
  "ordering_key": "thread_id | message_id | ...",
  "entity_ref": { "type": "message", "id": "uuid" },
  "payload": { ... event-specific ... }
}
```

### 6.2 Emitted event types (v1)

| Event | Emitted on | Payload highlights |
|-------|-----------|-------------------|
| `Core.MessageCreated` | `POST /messages` | message_id, thread_id, author_id, urgency, requires_response |
| `Core.MessageVersionCreated` | `POST /messages/:id/versions` | message_id, version, body, format |
| `Core.ThreadCreated` | `POST /threads` | thread_id, channel_id |
| `Core.ThreadStateChanged` | `POST /threads/:id/state` | thread_id, from_state, to_state |
| `Core.ReactionAdded` | `POST /reactions` | reaction_id, message_id, emoji, user_id |
| `Core.ThreadParticipantAdded` | `POST /threads/:id/participants` | thread_id, user_id, role |
| `Identity.MembershipChanged` | org membership changes | user_id, org_id, status |

---

## 7. Authentication & Authorization Flow

### 7.1 Signup (new user + new org)

```
Client
  │ POST /orgs/signup { email, password, display_name, org_name, org_slug, invites? }
  ▼
identity-service
  ├── Validate: unique email, unique slug, reserved slug check, domain rules
  ├── Create User + Organization + ORG_ADMIN Membership (atomic)
  ├── Optionally: create OrgInvites for teammates (tokens returned once)
  ├── Issue access token + refresh token
  └── Emit audit: identity.auth.signup
  ▼
Client receives: access_token, refresh_token, user, membership, org, [invites_created]
```

### 7.2 Login

```
Client
  │ POST /auth/login { email, password, org_slug }
  ▼
identity-service
  ├── Validate credentials
  ├── Look up Membership (user + org scoped)
  ├── Issue org-scoped JWT (claims: sub, org_id, membership_id, role_id, perms[])
  ├── Store RefreshToken (tokenHash, device/IP metadata)
  └── Emit audit: identity.auth.login
  ▼
Client: store access_token + refresh_token
```

### 7.3 Token refresh (rotation)

```
Client
  │ POST /auth/refresh { refresh_token }
  ▼
identity-service
  ├── Hash token → look up RefreshToken
  ├── Check expiry + revocation
  ├── Detect reuse: if already replaced → REVOKE ALL for user/org (reuse attack)
  ├── Issue new access + refresh token pair
  ├── Update: previous token's lastUsedAt, lastUsedFromIp, replacedById
  └── Store new RefreshToken
```

### 7.4 JWT validation in core-service / realtime-gateway

- Both services validate JWT using the **same `JWT_ACCESS_SECRET`** — no identity-service call per request
- Claims used: `sub` (user_id), `org_id`, `membership_id`, (optionally `perms`)
- Services never issue, store, or manage tokens

---

## 8. Realtime Flow

```
1. User sends a message via core-service (POST /messages)
   ├── core-service writes Message + MessageVersion to Postgres
   ├── core-service writes OutboxEvent { event_type: "Core.MessageCreated", ... } to Postgres
   └── Both in the same Prisma $transaction

2. Outbox Relay (separate process in core-service)
   ├── Polls OutboxEvent WHERE status=PENDING ORDER BY createdAt ASC
   ├── For each row: JSON.stringify(envelope) → redis.publish("realtime:events", payload)
   └── Marks row: status=PUBLISHED, publishedAt=now()

3. realtime-gateway
   ├── Subscribed to Redis channel "realtime:events"
   ├── Receives envelope JSON
   ├── Validates event_type is in ALLOWED_EVENT_TYPES (allowlist filter)
   ├── Looks up connections subscribed to (org_id + matching topic/thread_id)
   └── Sends { type: "event", payload: { envelope } } to each matching WebSocket

4. Client receives WebSocket "event" frame
   ├── Treats it as a HINT only (not authoritative)
   └── Schedules debounced HTTP refresh:
       - inbox topic → GET /inbox
       - thread topic → GET /messages/thread/:threadId (paginate with `cursor` / `next_cursor`)
```

---

## 9. API Surface Summary

### identity-service (`:3001`)

| Group | Endpoints |
|-------|-----------|
| **Health** | `GET /health`, `GET /health/ready` |
| **Auth** | `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `POST /auth/logout-all`, `POST /auth/password-reset/request`, `POST /auth/password-reset/confirm` |
| **Signup** | `POST /orgs/signup` (public), `GET /orgs/availability/slug` (public) |
| **Me** | `GET /me`, `GET /me/sessions`, `DELETE /me/sessions/:sessionId` |
| **Organizations** | `POST /orgs`, `GET /orgs/:orgId`, `GET/PATCH /orgs/:orgId/settings` |
| **Roles** | `GET/POST /orgs/:orgId/roles`, `PATCH /orgs/:orgId/roles/:roleId` |
| **Members** | `POST /orgs/:orgId/members`, `PATCH /orgs/:orgId/members/:memberId` |
| **Invites** | `POST/GET /orgs/:orgId/invites`, `POST /orgs/:orgId/invites/revoke`, `POST /orgs/:orgId/invites/verify` (public), `POST /orgs/:orgId/invites/accept` (public), `POST /orgs/:orgId/invites/decline` (public) |
| **Audit** | `GET /orgs/:orgId/audit` |

### core-service (`:3002`)

| Group | Endpoints |
|-------|-----------|
| **Health** | `GET /health`, `GET /health/ready` |
| **Channels** | `POST /channels`, `GET /channels` |
| **Threads** | `POST /threads`, `GET /threads/channel/:channelId`, `POST /threads/:threadId/state` |
| **Participants** | `POST/GET /threads/:threadId/participants`, `PATCH/DELETE /threads/:threadId/participants/:userId` |
| **User State** | `GET/PATCH /threads/:threadId/user-state` |
| **Inbox** | `GET /inbox` |
| **Messages** | `POST /messages`, `GET /messages/thread/:threadId` (cursor pagination, newest first), `POST/GET /messages/:messageId/versions` |
| **Reactions** | `POST /reactions`, `GET /reactions/message/:messageId` |

### realtime-gateway (`:3010`)

| Group | Endpoints |
|-------|-----------|
| **Health** | `GET /health`, `GET /health/ready` |
| **WebSocket** | `ws://:3010?access_token=<jwt>` |
| **WS Messages** | `{type:"auth"}`, `{type:"subscribe", payload:{org_id,topic,thread_id?}}`, `{type:"unsubscribe"}` |
| **WS Replies** | `subscribed`, `unsubscribed`, `event`, `error` |

---

## 10. Rate Limits (identity-service)

| Endpoint | Limit |
|----------|-------|
| `POST /auth/login` | 5 req / 60s |
| `POST /auth/refresh` | 30 req / 60s |
| `POST /orgs/signup` | 5 req / 60s |
| `POST /auth/password-reset/request` | 5 req / 60s |
| `POST /orgs/:orgId/invites/accept` | 5 req / 60s |

---

## 11. Frontend Architecture

### 11.1 Repo Split

| Repo | Contents |
|------|----------|
| `ayncor-client` | Web (Next.js) + Desktop (Electron/Tauri) monorepo |
| `ayncor-mobile` | iOS + Android (Expo/React Native) — separate repo, separate release cadence |

### 11.2 Client Monorepo Structure (`ayncor-client`)

```
ayncor-client/
├── apps/
│   ├── web/            ← Next.js 14+ App Router
│   └── desktop/        ← Electron or Tauri shell (loads web app)
├── packages/
│   ├── shared/         ← Types, API client, auth logic, constants
│   ├── ui/             ← Shared React components (shadcn/ui + Radix)
│   └── config/         ← Shared ESLint, TS, Prettier configs
├── turbo.json          ← Turborepo task pipeline
└── pnpm-workspace.yaml
```

### 11.3 API Client Generation

- Types + client generated from `@ayncor/contracts` OpenAPI specs
- Tooling: `openapi-typescript` → types; `openapi-fetch` → type-safe fetch client
- Auth: `Authorization: Bearer <access_token>` on every request; refresh on 401 using refresh token

### 11.4 Token Storage Strategy

| Platform | Access Token | Refresh Token |
|----------|-------------|---------------|
| Web | Memory (React state) | httpOnly cookie |
| Desktop | Secure in-memory | Electron secure storage |
| Mobile | Secure in-memory | iOS Keychain / Android Keystore |

---

## 12. Contracts — Source of Truth

```
contracts/v1/
├── openapi/
│   ├── identity-service.openapi.yaml   ← All identity HTTP API shapes
│   └── core-service.openapi.yaml       ← All core HTTP API shapes
├── events/
│   ├── envelope.schema.json            ← EventEnvelopeV1 shape
│   ├── core.message-created.schema.json
│   ├── core.message-version-created.schema.json
│   ├── core.thread-created.schema.json
│   ├── core.thread-state-changed.schema.json
│   ├── core.reaction-added.schema.json
│   ├── core.thread-participant-added.schema.json
│   ├── core.thread-user-state-changed.schema.json
│   ├── identity.user-created.schema.json
│   ├── identity.membership-changed.schema.json
│   └── identity.role-changed.schema.json
└── realtime/
    └── protocol.md                     ← WebSocket message protocol
```

**Rule:** Contracts are the single source of truth. Backend services implement them. Frontend generates clients from them. No silent changes.

---

## 13. Testing Architecture

### 13.1 Unit tests (per service)

- `identity-service`: Jest
- `core-service`: Jest
- `realtime-gateway`: Jest (unit tests for auth, subscription, message parsing)

### 13.2 E2E tests (`ayncor-e2e`)

Tests run against **live services**. Test flow covers:

1. Login (`identity-service`)
2. Channel + Thread + Message creation (`core-service`)
3. WebSocket connect + subscribe (`realtime-gateway`)
4. Message → relay → WebSocket event delivery (requires relay running)

**CI behavior:** Suite skips if no target URLs configured (so CI passes without staging).

---

## 14. Deployment Architecture

### 14.1 Current environments

| Environment | Purpose |
|------------|---------|
| `development` | Local; services run on ports 3001/3002/3010 |
| `staging` | Pre-prod on Render/Railway; mirrors prod config |
| `production` | Render/Railway with Postgres, Redis |

### 14.2 Services per environment

- **identity-service** → Node.js web service + Postgres
- **core-service** → Node.js web service + Postgres
- **core-service relay** → Node.js worker (polls outbox + publishes to Redis)
- **realtime-gateway** → Node.js web service + Redis (pub/sub only)
- **Redis** → shared by relay (producer) + realtime-gateway (subscriber)

### 14.3 Environment variables matrix

| Variable | identity | core | relay | realtime |
|----------|----------|------|-------|----------|
| `DATABASE_URL` | ✓ | ✓ | ✓ | — |
| `JWT_ACCESS_SECRET` | ✓ | ✓ | — | ✓ |
| `REDIS_URL` | — | — | ✓ | ✓ |
| `REDIS_CHANNEL` | — | — | ✓ | ✓ |
| `PORT` | ✓ (3001) | ✓ (3002) | — | ✓ (3010) |

---

## 15. Future Architecture (Planned)

```
Currently built:                   Next up:                    Further:
─────────────────                  ────────────────────        ─────────────────────
identity-service ✓                 Frontend (web/desktop) ◁   Notification service
core-service ✓                     Frontend (mobile)          AI service (summaries)
realtime-gateway ✓                 Email delivery             Search service
outbox relay ✓                     MFA / SSO                  Analytics service
contracts ✓                        API Gateway / LB            Object storage (S3)
ayncor-e2e ✓                       Monitoring stack            Redis cache layer
.github org meta ✓                                             Message queue (scale)
```

---

## 16. Design Principles (Non-negotiable)

1. **Async by default** — No presence, no typing indicators, no implicit urgency.
2. **Contracts first** — OpenAPI specs and protocol docs define behavior; services implement them.
3. **Strict service boundaries** — identity owns identity; core owns collaboration; no cross-DB queries.
4. **Events are hints** — Realtime delivery is best-effort; HTTP REST is always authoritative.
5. **Security over convenience** — Rate limiting, token rotation, reuse detection, no enum leakage.
6. **Immutable records** — Messages don't change; edits append versions; threads have state, not delete.
7. **Deterministic inbox** — Ordering is server-defined, stable, and explainable (not real-time-ranked).
