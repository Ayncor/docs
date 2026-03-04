# Railway setup for ayncor

Step-by-step guide to run **identity-service**, **core-service**, **realtime-gateway**, and **relay** on [Railway](https://railway.app) with **PostgreSQL** (x2) and **Redis**, across **development**, **staging**, and **production**.

**All four app components come from three GitHub repos.** identity-service, core-service, and realtime-gateway are one service each; relay is a second service from the **core-service** repo (same repo, different start command).

---

## 1. Repos and what deploys

| GitHub repo          | Railway service(s)     | Type              |
|----------------------|------------------------|-------------------|
| **identity-service** | identity-service       | Web Service       |
| **core-service**     | core-service, relay    | Web Service, Worker |
| **realtime-gateway** | realtime-gateway       | Web Service       |

**Data (per environment):** 2× Postgres (identity-db, core-db), 1× Redis. Create these first, then deploy the app services.

---

## 2. Prerequisites

- [Railway](https://railway.app) account (sign up with GitHub).
- GitHub repos: **identity-service**, **core-service**, **realtime-gateway** (each in its own repo or in a monorepo).
- **Relay** runs from the **core-service** repo (same codebase, different start command).

---

## 3. Create project and environments

1. Go to [railway.app](https://railway.app) → **New Project**.
2. Name the project (e.g. **ayncor**).
3. Create **three environments** (if not already present):
   - **development**
   - **staging**
   - **production**  
   (Project → **Environments** → **New**.)

You’ll add the same set of services and data to each environment (or share data and only deploy different branches).

---

## 4. Add data (Postgres + Redis)

In **each** environment (or once and reference from all):

1. **Add Postgres (identity):**  
   **+ New** → **Database** → **PostgreSQL**. Name it **identity-db**.  
   Copy the **DATABASE_URL** (or use variable reference in services).

2. **Add Postgres (core):**  
   **+ New** → **Database** → **PostgreSQL**. Name it **core-db**.  
   Copy **DATABASE_URL** for core-service and relay.

3. **Add Redis:**  
   **+ New** → **Database** → **Redis**. Name it **redis**.  
   Copy **REDIS_URL** (or use variable reference).  
   Used by **realtime-gateway** (subscribe) and **core-service** relay (publish).

If you use one environment for “dev” first, add these three there; repeat for staging and production when ready.

---

## 5. Private npm (@ayncor/contracts)

All Node services install `@ayncor/contracts` from GitHub Packages. Add **NPM_TOKEN** (GitHub PAT with `read:packages`) as a **secret** variable in each service. Use it in the build command:

```
echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && ...
```

---

## 6. Deploy identity-service

1. **+ New** → **GitHub Repo** → select **identity-service** repo.
2. **Environment:** choose **development** (or staging/production).
3. **Settings** for the new service:
   - **Name:** `identity-service` (or `identity-service-dev` for dev env).
   - **Root Directory:** leave empty if repo root is the app; otherwise set (e.g. `identity-service` in a monorepo).
   - **Build Command:**  
     `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
   - **Start Command:**  
     `npx prisma migrate deploy && npm start`
   - **Watch Paths:** leave default so pushes to the repo trigger deploys.

4. **Variables** (Settings → **Variables** → **Add** or use **Variables** from Postgres):
   - **NPM_TOKEN** → GitHub PAT with `read:packages` (secret).
   - **DATABASE_URL** → **Add Reference** → select **identity-db** → `DATABASE_URL`.
   - **PORT** → Railway sets this; optional override (e.g. `3001` for local parity).
   - **JWT_ACCESS_SECRET** → generate a long random value (e.g. 32+ bytes, base64); **same value** in identity, core, and realtime-gateway.
   - **BOOTSTRAP_EMAIL**, **BOOTSTRAP_PASSWORD**, **BOOTSTRAP_ORG_SLUG**, **BOOTSTRAP_ORG_NAME** → from your `.env.example` (e.g. `admin@ayncor.local`, secure password, `ayncor`, `AynCor`).
   - **JWT_ACCESS_TTL_SECONDS**, **JWT_REFRESH_TTL_MS** → optional (defaults are fine).

5. **Deploy:** Railway builds and runs. Migrations run automatically on start.

6. **Custom domain / URL:** Settings → **Networking** → **Generate Domain** (or add your own). Note the URL (e.g. `https://identity-service-xxx.up.railway.app`) for **core-service** and clients.

---

## 7. Deploy core-service

1. **+ New** → **GitHub Repo** → select **core-service** repo.
2. **Environment:** same as identity (e.g. **development**).
3. **Settings:**
   - **Name:** `core-service` (or `core-service-dev` for dev env).
   - **Build Command:**  
     `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
   - **Start Command:**  
     `npx prisma migrate deploy && npm start`

4. **Variables:**
   - **NPM_TOKEN** → GitHub PAT with `read:packages` (secret).
   - **DATABASE_URL** → reference **core-db** `DATABASE_URL`.
   - **REDIS_URL** → reference **redis** `REDIS_URL` (or `REDIS_PRIVATE_URL` if available).
   - **REDIS_CHANNEL** → `realtime:events`.
   - **JWT_ACCESS_SECRET** → **same** as identity-service (and realtime-gateway).
   - **PORT** → optional.

5. **Deploy.** Migrations run on start.

---

## 8. Deploy realtime-gateway

1. **+ New** → **GitHub Repo** → select **realtime-gateway** repo.
2. **Environment:** same as identity and core.
3. **Settings:**
   - **Name:** `realtime-gateway` (or `realtime-gateway-dev` for dev env).
   - **Build Command:**  
     `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npm run build`
   - **Start Command:**  
     `npm start`

4. **Variables:**
   - **NPM_TOKEN** → GitHub PAT with `read:packages` (secret).
   - **REDIS_URL** → reference **redis** `REDIS_URL`.
   - **REDIS_CHANNEL** → `realtime:events`.
   - **JWT_ACCESS_SECRET** → **same** as identity-service.
   - **PORT** → optional.

5. **Deploy.**  
   **Networking:** Generate domain; clients will use **wss://** that URL for WebSockets.

---

## 9. Deploy relay (from core-service repo)

The relay is a long-running process that polls core’s outbox and publishes to Redis. Two options:

**Option A – Same repo, separate Railway service**

1. **+ New** → **GitHub Repo** → select **core-service** again (same repo).
2. **Name:** `relay` (or `relay-dev` for dev env).
3. **Build Command:**  
   `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
4. **Start Command:**  
   `npx prisma migrate deploy && node dist/relay/main.js`
5. **Variables:** **NPM_TOKEN**, **DATABASE_URL** (core-db), **REDIS_URL**, **REDIS_CHANNEL**.
6. **Deploy.**

**Option B – Single “core” service that runs API + relay**

- Not recommended: relay is usually a separate process. Keep **core-service** (HTTP) and **relay** as two Railway services.

**Start Command for relay service:** `node dist/relay/main.js`  
After `npm run build`, Nest compiles `src/` including `src/relay/main.ts` → `dist/relay/main.js`. If your build doesn’t output it, add a script in package.json (e.g. `"relay": "node dist/relay/main.js"`) and use **Start Command:** `npm run relay`.

---

## 10. Run migrations (first time per environment)

Migrations run automatically in the **Start Command** (`npx prisma migrate deploy && ...`). If you need to run them manually (e.g. before first deploy):

- **Identity DB:** `cd identity-service && npx prisma migrate deploy` (with `DATABASE_URL` = Railway identity-db URL)
- **Core DB:** `cd core-service && npx prisma migrate deploy` (with `DATABASE_URL` = Railway core-db URL)

---

## 11. Wire branches to environments

- **development** branch → deploy to **development** environment.  
- **staging** branch → deploy to **staging** environment.  
- **main** branch → deploy to **production** environment.

For each service in Railway: **Settings** → **Source** → **Branch** → set to `development`, `staging`, or `main` for that environment. Railway will redeploy on push to that branch.

If you use **one** Railway project with three environments, create three “sets” of services (one set per environment) or use one set and switch branch per environment; the clean approach is one environment per branch and attach the same service types to each.

---

## 12. GitHub Actions (optional)

You can keep **CI only** in GitHub (build + test) and let **Railway** do the deploy (via “Deploy on push” above). No change to Actions needed.

If you want **Actions to trigger or confirm deploys**:

- Use Railway’s **Deploy** API or **CLI** in a workflow step.
- Store **RAILWAY_TOKEN** (project or service token) in repo secrets and run e.g. `railway up` or call the deploy API after `build-and-test`.

Your current identity-service workflow uses **Azure** deploy; you can remove those jobs and rely on Railway’s GitHub integration, or add a “deploy to Railway” step using the CLI/API.

---

## 13. Checklist per environment

| Step | identity-service | core-service | realtime-gateway | relay |
|------|-------------------|---------------|-------------------|-------|
| New service from GitHub | ✓ | ✓ | ✓ | ✓ (from core-service repo) |
| Build / Start commands | ✓ | ✓ | ✓ | ✓ |
| DATABASE_URL | identity-db | core-db | — | core-db |
| REDIS_URL | — | ✓ | ✓ | ✓ |
| JWT_ACCESS_SECRET | ✓ | ✓ | ✓ | — |
| BOOTSTRAP_* (identity only) | ✓ | — | — | — |
| NPM_TOKEN | ✓ | ✓ | ✓ | ✓ |
| Run migrations (in Start) | identity | core | — | core |
| Generate domain | ✓ | ✓ | ✓ | — |

---

## 14. URLs for ayncor-e2e / clients

After deploy, note:

- **Identity:** `https://identity-service-xxx.up.railway.app`
- **Core:** `https://core-service-xxx.up.railway.app`
- **Realtime (WS):** `wss://realtime-gateway-xxx.up.railway.app`

Use these in **ayncor-e2e** (vars/secrets: `IDENTITY_URL`, `CORE_URL`, `REALTIME_WS_URL`) and in frontend/env config. Ensure **JWT_ACCESS_SECRET** is identical in identity, core, and realtime-gateway so JWTs work across services.
