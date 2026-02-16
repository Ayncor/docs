# Onboarding Extension Design

Extension to support **self-signup**, **org creation during onboarding**, **optional team invites**, and **org settings** (email domain restriction, team URL).

---

## 1. User Flow Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ NEW USER → Sign up → Create org → (Optional) Add team / Send invites         │
│            ↓                                                                │
│            Username, email, password                                         │
│            Team name, team URL (____.ayncor.com)                             │
│            (Optional) Invite team members now or skip for later              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. API Changes

### 2.1 Sign-up (new)

**`POST /orgs/signup`** — Public, rate-limited (5/60s)

Creates user + org in one transaction. Caller becomes ORG_ADMIN.

**Request:**
```json
{
  "email": "user@company.com",
  "password": "min 8 chars",
  "display_name": "John Doe",
  "org_name": "Acme Inc",
  "org_slug": "acme",
  "invites": [
    { "email": "teammate@company.com", "role_id": null }
  ]
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `email` | ✓ | Must be unique |
| `password` | ✓ | Min 8 chars |
| `display_name` | ✓ | Username / display name (1–120 chars) |
| `org_name` | ✓ | Team name (3–120 chars) |
| `org_slug` | ✓ | Team URL subdomain: `{org_slug}.ayncor.com`. Pattern: `^[a-z0-9-]+$`, 3–60 chars |
| `invites` | Optional | Array of `{ email, role_id? }`. Can be empty or omitted |

**Response:** Same as `POST /auth/login` — `AuthSession` (access_token, refresh_token, user, membership, org).

**Validation:**
- `org_slug` must be unique (no existing org)
- `email` must be unique (no existing user)
- Reserved slugs: `www`, `app`, `api`, `admin`, etc. (configurable blocklist)

---

### 2.2 Org Settings (new model + endpoints)

**New fields on `Organization` (or `OrgSettings` table):**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `allowed_email_domains` | `string[]` | `[]` | If non-empty: only emails with these domains can be invited/added. Empty = allow any. |
| `require_company_email` | `boolean` | `false` | Alias for "allowed_email_domains has entries" — UI checkbox |

**`GET /orgs/{orgId}/settings`** — Org admin only  
Returns org settings.

**`PATCH /orgs/{orgId}/settings`** — Org admin only  
Update `allowed_email_domains`, etc.

**Request (PATCH):**
```json
{
  "allowed_email_domains": ["company.com", "acme.co"],
  "require_company_email": true
}
```

When `allowed_email_domains` is set:
- `POST /orgs/{orgId}/invites` — Reject if invitee email domain not in list
- `POST /orgs/{orgId}/members` — Same
- Sign-up `invites` array — Same (during onboarding)

---

### 2.3 Slug availability check

**`GET /orgs/availability/slug?slug=acme`** — Public  
Returns `{ available: boolean }` for sign-up form validation.

---

## 3. Data Model Changes

### 3.1 Organization (extend)

```prisma
model Organization {
  // ... existing fields
  allowedEmailDomains String[] @default([])  // e.g. ["company.com"]
}
```

Migration: add column `allowedEmailDomains` (or `allowed_email_domains` in DB) as `TEXT[]` default `'{}'`.

### 3.2 User

- `display_name` already exists — use as "username" for now.
- Optional future: add `username` (unique handle like @johndoe) — out of scope for v1.

---

## 4. Frontend Onboarding Flow

### Step 1: Account
- Email
- Password
- Display name (username)

### Step 2: Team
- Team name
- Team URL: `____.ayncor.com` — input is the subdomain (e.g. `acme` → `acme.ayncor.com`)
- Optional: "Check availability" → `GET /orgs/slug-available?slug=acme`

### Step 3: Team members (optional)
- "Add team members now" or "I'll do this later"
- If now: list of emails, optional role per invite
- Submit → `POST /auth/signup` with `invites` array

### Step 4: Redirect
- On success: store tokens, redirect to dashboard
- Invite tokens returned once — frontend can show "Copy invite links" or "We've sent invites" (when email delivery is implemented)

---

## 5. Settings (post-onboarding)

**Org Settings → General**
- Team name (editable)
- Team URL / slug (editable with care — may break links)
- **Allow sign-up with company email only** (checkbox)
  - When checked: show domain input(s), e.g. `company.com`, `acme.co`
  - Stored as `allowed_email_domains`
  - When enabled: only those domains can be invited/added

**Org Settings → Members**
- Invite members (existing flow)
- Domain check applied if `allowed_email_domains` is set

---

## 6. Reserved / Blocked Slugs

Suggested blocklist for `org_slug`:
- `www`, `app`, `api`, `admin`, `mail`, `ftp`, `staging`, `dev`, `test`, `identity`, `core`, `realtime`

Validate in sign-up and `POST /orgs`.

---

## 7. Implementation Checklist

### Backend (identity-service)

- [x] Migration: add `allowedEmailDomains` to Organization
- [x] `POST /orgs/signup` — create user + org + optional invites, return session
- [x] `GET /orgs/availability/slug?slug=...` — public
- [x] `GET /orgs/{orgId}/settings` — org admin
- [x] `PATCH /orgs/{orgId}/settings` — org admin
- [x] Enforce `allowed_email_domains` in createInvite, createMember
- [x] Slug blocklist validation
- [ ] Update OpenAPI contract

### Frontend (separate repo / later)

- [ ] Sign-up page (multi-step or single form)
- [ ] Slug availability check on blur
- [ ] Optional invite step in onboarding
- [ ] Settings: team name, team URL, allowed domains checkbox
- [ ] Apply domain validation in invite UI

---

## 8. Open Questions

1. **Email delivery:** Invites return a token; delivery (email, Slack) is out-of-band. Integrate SendGrid/Resend later?
2. **Username vs display_name:** Use `display_name` for v1; add `username` (unique handle) later if needed.
3. **Slug edit:** Allowing slug change post-creation may break `acme.ayncor.com` links. Consider read-only or warning.
