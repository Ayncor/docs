# Render setup for ayncor

Step-by-step guide to run **identity-service**, **core-service**, **realtime-gateway**, and **relay** on [Render](https://render.com) with **PostgreSQL** (x2) and **Redis**, across **development**, **staging**, and **production**.

**All four app components come from three GitHub repos.** identity-service, core-service, and realtime-gateway are one service each; relay is a second service from the **core-service** repo (same repo, different start command).

---

## 1. Repos and what deploys

| GitHub repo          | Render service(s)     | Type              |
|----------------------|------------------------|-------------------|
| **identity-service** | identity-service       | Web Service       |
| **core-service**     | core-service, relay    | Web Service, Background Worker |
| **realtime-gateway** | realtime-gateway       | Web Service       |

**Data (per environment):** 2× Postgres (identity-db, core-db), 1× Redis. Create these first, then create the app services and link them.

---

## 2. Prerequisites

- [Render](https://render.com) account (sign up with GitHub).
- Three separate GitHub repos with `development`, `staging`, `main` branches:
  - **identity-service**, **core-service**, **realtime-gateway**

---

## 3. Environments: one set of services per environment

Render does not have built-in environments. You get **one set of services per environment** by creating separate services with clear names:

- **Development:** identity-service-dev, core-service-dev, realtime-gateway-dev, relay-dev, identity-db-dev, core-db-dev, redis-dev
- **Staging:** identity-service-staging, core-service-staging, etc.
- **Production:** identity-service, core-service, etc. (or identity-service-prod if you prefer)

Create **development** first (one Postgres, one Redis, then add the second Postgres; then the four app services). Repeat the same structure for staging and production.

---

## 4. Create databases and Redis (per environment)

1. Dashboard → **New +** → **PostgreSQL**. Name: `identity-db-dev` (or `identity-db` in prod). Region: choose one. Create. Note the **Internal Database URL** (or **External** if apps are elsewhere).
2. **New +** → **PostgreSQL** again. Name: `core-db-dev`. Create. Note its URL.
3. **New +** → **Redis**. Name: `redis-dev`. Create. Note **Internal Redis URL** (or External).

Repeat for staging and production with names like `identity-db-staging`, `core-db-staging`, `redis-staging`, etc.

---

## 5. Private npm (@ayncor/contracts)

All Node services need to install `@ayncor/contracts` from GitHub Packages. On Render:

1. In each **Web Service** or **Background Worker**, add an **Environment Variable**: **Key** `NPM_TOKEN`, **Value** = your GitHub PAT with `read:packages` (same as CI). Mark **Secret**.
2. In the service **Build Command**, ensure the install step has auth. Use:
   - **Build Command:** `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`  
   (Omit `npx prisma generate` for **realtime-gateway**; omit both prisma and build for relay if it shares a build with core-service—see below.)

---

## 6. identity-service (Web Service)

1. **New +** → **Web Service**.
2. Connect the **identity-service** GitHub repo. Name: `identity-service-dev` (or `identity-service` for prod).
3. **Branch:** `development` (or `staging` / `main` for other envs).
4. **Runtime:** Node.
5. **Build Command:**  
   `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
6. **Start Command:**  
   `npx prisma migrate deploy && npm start`
7. **Environment Variables:** Add **NPM_TOKEN** (secret, GitHub PAT). Link **DATABASE_URL** from the **identity-db** Postgres you created (Render lets you “Link” an existing DB and injects DATABASE_URL). Add **JWT_ACCESS_SECRET**, **BOOTSTRAP_EMAIL**, **BOOTSTRAP_PASSWORD**, **BOOTSTRAP_ORG_SLUG**, **BOOTSTRAP_ORG_NAME** (and optional **PORT**, **JWT_ACCESS_TTL_SECONDS**, **JWT_REFRESH_TTL_MS**).
8. **Instance type:** Free for try; paid for always-on and WebSockets if needed later.
9. Create Web Service. After deploy, open the service URL (e.g. `https://identity-service-dev.onrender.com`).

Repeat for **staging** and **production** (new Web Service, same repo, different branch and env var values; link to that env’s identity-db).

---

## 7. core-service (Web Service)

1. **New +** → **Web Service**. Connect **core-service** repo. Name: `core-service-dev`.
2. **Branch:** `development`.
3. **Build Command:**  
   `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
4. **Start Command:**  
   `npx prisma migrate deploy && npm start`
5. **Environment Variables:** **NPM_TOKEN** (secret). Link **DATABASE_URL** from **core-db**, **REDIS_URL** from **redis**. Add **REDIS_CHANNEL** = `realtime:events`, **JWT_ACCESS_SECRET** (same as identity-service in this env).
6. Create Web Service.

Repeat for staging/production.

---

## 8. realtime-gateway (Web Service)

1. **New +** → **Web Service**. Connect **realtime-gateway** repo. Name: `realtime-gateway-dev`.
2. **Branch:** `development`.
3. **Build Command:**  
   `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npm run build`
4. **Start Command:**  
   `npm start`
5. **Environment Variables:** **NPM_TOKEN**. Link **REDIS_URL** from **redis**. Add **REDIS_CHANNEL** = `realtime:events`, **JWT_ACCESS_SECRET** (same as identity-service).
6. Create Web Service. WebSockets are supported on paid instances.

Repeat for staging/production.

---

## 9. relay (Background Worker, same repo as core-service)

1. **New +** → **Background Worker**. Connect the **core-service** repo again. Name: `relay-dev`.
2. **Branch:** `development`.
3. **Build Command:**  
   Same as core-service: `echo "//npm.pkg.github.com/:_authToken=$NPM_TOKEN" >> .npmrc && npm ci && npx prisma generate && npm run build`
4. **Start Command:**  
   `npx prisma migrate deploy && node dist/relay/main.js`
5. **Environment Variables:** **NPM_TOKEN**. Link **DATABASE_URL** (core-db), **REDIS_URL** (redis). Add **REDIS_CHANNEL** = `realtime:events`. No JWT needed.
6. Create Background Worker.

Repeat for staging/production.

---

## 10. Variables to change per environment

| Variable / setting   | Development     | Staging         | Production      |
|----------------------|------------------|-----------------|-----------------|
| Branch               | `development`   | `staging`      | `main`          |
| DATABASE_URL / REDIS_URL | Link identity-db-dev, core-db-dev, redis-dev | Link staging DBs/Redis | Link prod DBs/Redis |
| JWT_ACCESS_SECRET    | Dev secret       | Different       | Strong, unique  |
| BOOTSTRAP_* (identity) | Dev admin       | Staging admin   | Real admin      |

Use different **JWT_ACCESS_SECRET** per environment. **NPM_TOKEN** can be the same everywhere (build-time only).

---

## 11. Checklist per environment

| Step                    | identity-service | core-service | realtime-gateway | relay |
|-------------------------|------------------|--------------|-------------------|-------|
| Create service          | Web Service      | Web Service  | Web Service       | Background Worker |
| Connect repo            | identity-service | core-service | realtime-gateway  | core-service |
| Build / Start           | §6               | §7           | §8                | §9    |
| NPM_TOKEN               | ✓                | ✓            | ✓                 | ✓     |
| DATABASE_URL            | identity-db      | core-db      | —                 | core-db |
| REDIS_URL               | —                | ✓            | ✓                 | ✓     |
| JWT_ACCESS_SECRET       | ✓                | ✓            | ✓                 | —     |
| BOOTSTRAP_* (identity)  | ✓                | —            | —                 | —     |
| Migrations              | In Start Command | In Start Command | —              | In Start Command |

---

## 12. URLs for ayncor-e2e / clients

After deploy, note each service URL (e.g. `https://identity-service-dev.onrender.com`). Use them in **ayncor-e2e** and frontend env (IDENTITY_URL, CORE_URL, REALTIME_WS_URL). Keep **JWT_ACCESS_SECRET** identical across identity, core, and realtime-gateway in the same environment so JWTs work across services.
