# Ayncor — Message encryption, at-rest protection, and E2EE

This document is the **canonical guide** for how Ayncor thinks about protecting message content: **in transit**, **at rest**, **customer-managed keys (Slack-style enterprise)**, and optional **end-to-end encryption (E2EE)**. It explains trade-offs (especially **search**), compares **Slack vs WhatsApp**, and lists **what to implement when**.

**Related:** [future-services-roadmap.md](./future-services-roadmap.md) (Section 7 summary + cold tier Section 6).

---

## 1. Why this file is named `e2ee.md`

E2EE is the **strongest** privacy model but **not** the default for a Slack-like product. The filename keeps the topic easy to find; the doc covers **all** encryption tiers, not only E2EE.

---

## 2. Threat models (short)

| Concern                          | Baseline SaaS (Option 1)                     | App-layer + KMS (Option 2)                 | E2EE (Option 3)                                |
| -------------------------------- | -------------------------------------------- | ------------------------------------------ | ---------------------------------------------- |
| Network eavesdropper             | Mitigated by TLS                             | Mitigated by TLS                           | Mitigated by TLS + client crypto               |
| Stolen DB dump / backup          | Mitigated partly by provider disk encryption | Stronger: ciphertext in DB without key     | Strongest: server should not have keys to read |
| Malicious/compromised **server** | Server can read messages in memory/DB        | Same while serving traffic; DB dump harder | Designed to limit server readability           |
| Legitimate **admin** / support   | Can access per policy & audit                | Key policy + KMS audit                     | Very limited without client cooperation        |

Choose the model that matches **product promises** and **compliance** stories.

---

## 3. Slack vs WhatsApp (how they differ)

### WhatsApp (simplified)

- **E2EE by default** for the core chat product (Signal-influenced protocols).
- **Implication:** the service is not designed to rely on reading every message body for server-side features.

### Slack (Ayncor’s inspiration)

- **Not E2EE** for standard workspace messages in the sense of “server never can decrypt.”
- **Baseline:** TLS in transit + **encryption at rest** (provider / platform controls).
- **Enterprise:** **Customer-managed keys** (e.g. **Slack EKM** with **AWS KMS**) so customers control encryption keys, rotation, and revocation—while **preserving** Slack-style features (search, notifications, link unfurling, compliance exports) because Slack’s architecture still processes content when keys allow.

**Ayncor target (Slack-like):** Option 1 for everyone → Option 2 for enterprise → Option 3 only as an explicit product bet.

---

## 4. The four practical layers (what “encryption” means)

### 4.1 Encryption in transit (TLS / HTTPS / WSS)

- **What:** Clients use `https://` and `wss://` to services; no plaintext on the wire.
- **Ayncor today:** Enforced by **deployment** (load balancer / PaaS), not inside Nest/Express. Local dev may use `http://` / `ws://`.
- **When you set up staging/prod:** Terminate TLS at the edge; redirect HTTP→HTTPS; use WSS for realtime.
- **Checklist:**
  - [ ] TLS certificates (managed by host or ACME)
  - [ ] HSTS (web)
  - [ ] Secure WebSocket (`wss://`) in client config for prod

### 4.2 Encryption at rest — provider (disk / backups)

- **What:** Cloud Postgres encrypts storage and backups (vendor-dependent).
- **Ayncor today:** **Yes, if the provider enables it** (typical on Render/RDS/etc.). No extra app code.
- **Checklist:**
  - [ ] Confirm provider docs: disk encryption + backup encryption
  - [ ] Restrict network access to DB (VPC, IP allowlist, private link where available)
  - [ ] Least-privilege DB roles; no shared superuser in app

### 4.3 Application-layer encryption at rest (encrypt `MessageVersion.body` before DB write)

- **What:** `core-service` stores **ciphertext** (+ metadata: algorithm, nonce, key id); decrypt on read before API response.
- **Effect on search:**
  - **Postgres `ILIKE` / simple full-text on `body`:** **does not work** on ciphertext.
  - **Dedicated search service:** **can** work if you **index at write time** (server has plaintext in memory, pushes to index) or use advanced encrypted-search schemes (heavy R&D).
- **Slack analogy:** Closer to **enterprise hardening**; not the same as E2EE. Often combined with **KMS** and later **customer-managed keys**.

### 4.4 End-to-end encryption (E2EE)

- **What:** Clients encrypt; server stores/forwards ciphertext; only clients hold decryption keys (with careful key distribution, recovery, and device sync).
- **Effect:** **Server-side search over body**, **server-side AI on body**, and naive **compliance export** become **hard or impossible** without explicit product design.
- **When:** Only if Ayncor chooses a **WhatsApp-like** or **zero-knowledge** positioning.

### 4.5 Customer-managed keys (EKM-style, enterprise)

- **What:** Per-org (or per-tenant) keys in **KMS**; customer can rotate/revoke; audit who decrypted what.
- **Size of change:** **Large**—policy UI, billing tiers, KMS integration, key rotation jobs, failure modes when keys are unavailable.
- **When:** After **product is live**, **durable jobs** exist, and **search/attachments** direction is clear—typically **Phase D enterprise**, not MVP.

---

## 5. What Ayncor has today (honest inventory)

| Layer                                         | Status                                                                             |
| --------------------------------------------- | ---------------------------------------------------------------------------------- |
| TLS in prod                                   | **Your responsibility** when you configure staging/prod (not enforced in app code) |
| Provider DB at rest                           | **Yes if provider enables it**                                                     |
| App-layer encryption of `MessageVersion.body` | **No** (plaintext at application layer in Postgres)                                |
| Customer-managed keys / EKM                   | **No**                                                                             |
| E2EE                                          | **No**                                                                             |
| Other security (Slack-like adjacent)          | JWT auth, refresh rotation, rate limits, RBAC, audit logs (`identity-service`)     |

---

## 6. Search and app-layer encryption (why we defer Option 2 until later)

**Decision (current):** Do **not** implement application-layer encryption of message bodies **until** the product is further along and **search architecture** is chosen.

**Reason:** App-layer encryption **breaks naive server-side DB search** on message text. Slack-style products expect **search** to work; Slack achieves enterprise controls with **EKM**, not by hiding all plaintext from every server subsystem forever.

**When returning to Option 2, plan in this order:**

1. **Choose search approach** (e.g. OpenSearch / Meilisearch / vendor) and whether indexing is **at ingest** (recommended with encrypted DB bodies).
2. Introduce **`MessageCrypto` abstraction** in `core-service` (single place for encrypt/decrypt; swap KMS later).
3. **Schema:** ciphertext + nonce + `key_id` / algorithm on `MessageVersion` (or parallel columns); migration for existing rows.
4. Update **all read paths:** `MessagesService`, **InboxService** previews (`latest_message_preview` uses body substring → decrypt first).
5. **Background jobs:** any indexer worker must decrypt or receive plaintext via a controlled pipeline—**never** log bodies.
6. **Rotation:** document re-encrypt or multi-key read path before enterprise EKM.

---

## 7. Staged roadmap (what to do when)

### Phase A — Now (ship product, Slack-like baseline)

- [ ] **TLS** for staging/prod (HTTPS/WSS)
- [ ] Confirm **Postgres at-rest encryption** + locked-down connectivity
- [ ] Operational hygiene: secrets in env/KMS, no prod credentials in repos, audit admin DB access
- [ ] **Do not** add app-layer message encryption yet (preserves flexibility for search)

### Phase B — After notifications + durable queue

- [ ] Revisit Option 2 **only if** security roadmap or customer demand requires it **before** search
- [ ] Prefer implementing **search indexer** first if search is on the near-term roadmap

### Phase C — Search + files

- [ ] Search service + ingest pipeline (see [future-services-roadmap.md](./future-services-roadmap.md) §2.3)
- [ ] Attachments in object storage (separate encryption story: SSE-KMS, customer keys on bucket)

### Phase D — Enterprise

- [ ] **Option 2** with **KMS**; then **customer-managed keys (EKM-style)** per org/plan
- [ ] Admin UX: key state, rotation, revocation impact (“org unreadable until key restored”)

### Phase E — Optional E2EE mode (only if product strategy requires)

- [ ] Threat model + UX: recovery, multi-device, org offboarding, legal hold
- [ ] New client protocols; likely **separate** product tier or workspace mode
- [ ] Contracts and support playbooks

---

## 8. Implementation checklist — application-layer encryption (when you start)

**`core-service`**

- [ ] `MessageCrypto` interface + AES-256-GCM (or libsodium) implementation
- [ ] Env: `MESSAGE_ENCRYPTION_KEY` or KMS key id; support **key version** in stored metadata
- [ ] `MessagesService.createMessage` / `createVersion`: encrypt before `prisma.messageVersion.create`
- [ ] `MessagesService.listMessagesForThread` / `listVersions`: decrypt before mapping to DTO
- [ ] `InboxService`: decrypt for preview snippet (or remove preview until decrypt path exists)
- [ ] Metrics: encrypt/decrypt errors (no body in logs—use [logger](../.cursor/rules/ayncor-logging.mdc) rules)

**Schema (illustrative)**

- [ ] Store `body_ciphertext`, `body_nonce`, `body_key_id`, `body_alg` **or** replace `body` with opaque blob + metadata (pick one migration strategy)

**Contracts**

- [ ] Usually **no** public API shape change (responses stay plaintext JSON for authorized clients)
- [ ] Document security posture in service README + trust center material

**Search**

- [ ] Indexer consumes events or polls with **plaintext at index time**; secure index with org-scoped ACLs

---

## 9. Implementation checklist — EKM / customer-managed keys (later)

- [ ] Per-org KMS key ARN (or equivalent) in org settings / billing entitlements
- [ ] `MessageCrypto` resolves key by `org_id`
- [ ] Key rotation: dual-read old+new; background re-encrypt
- [ ] Revocation behavior defined (fail closed vs grace period)
- [ ] Audit: CloudTrail / provider equivalent; customer-facing reports

---

## 10. Implementation checklist — E2EE (optional, largest bet)

- [ ] Key lifecycle: device keys, org keys, rotation, recovery (recovery codes, escrow policy—explicitly chosen)
- [ ] Client: encrypt on send, decrypt on display; handle attachments consistently
- [ ] Server: store ciphertext + minimal metadata; **no** server-side body search unless client-side search or encrypted-search R&D
- [ ] Compliance and export: define legal hold without server plaintext
- [ ] New docs: protocol, key backup, device revocation

---

## 11. Summary

| Approach                 | Do now?                     | Search impact                               | Slack-like?          |
| ------------------------ | --------------------------- | ------------------------------------------- | -------------------- |
| TLS + provider at rest   | **Yes** (with prod setup)   | None                                        | Baseline             |
| App-layer encrypt bodies | **Defer** until search plan | Breaks DB text search; needs index pipeline | Enterprise hardening |
| Customer-managed keys    | **Later** (Phase D)         | Same as app-layer + ops complexity          | Yes (EKM-style)      |
| E2EE                     | **Only if strategic**       | Severe unless client-side search            | No (WhatsApp-like)   |

**Bottom line:** For a **Slack-like** Ayncor, **ship with TLS + provider at rest + strong access controls**; **defer app-layer message encryption** until after **search (and jobs) are designed**; treat **EKM** as enterprise; treat **E2EE** as a separate product decision.
