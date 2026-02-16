# Hosting recommendation for ayncor

Recommendation for running **identity-service**, **core-service**, **realtime-gateway**, **relay**, **PostgreSQL** (x2), and **Redis** across development, staging, and production.

---

## What you need to host

| Component            | Type              | Notes                                      |
|----------------------|-------------------|--------------------------------------------|
| identity-service     | Node.js (NestJS)  | HTTP API, port 3001                        |
| core-service         | Node.js (NestJS)  | HTTP API, port 3002                        |
| realtime-gateway     | Node.js + WebSocket | Long-lived connections, port 3010       |
| relay                | Node.js (process) | Polls outbox, publishes to Redis          |
| PostgreSQL (identity) | Database         | identity-service only                      |
| PostgreSQL (core)    | Database          | core-service + outbox                      |
| Redis                | Cache / Pub-Sub   | realtime-gateway + relay (realtime:events) |

Three environments: **development**, **staging**, **production**.

---

## Recommended: **Render**

**Best fit** for your current stage: stable PaaS, multi-service Node stack, Postgres, Redis, WebSockets, and three environments. Mature platform with reliable UI and predictable behavior.

### Why Render

- **Stable and predictable** – No flaky "unknown error" UI; create services and databases without random failures. Mature product, good uptime.
- **One platform** – Web Services for identity, core, realtime-gateway; Background Worker for relay; managed Postgres (x2) and Redis from the dashboard.
- **Managed Postgres and Redis** – Provision from dashboard; connection strings and env vars (e.g. `DATABASE_URL`, `REDIS_URL`) available to link to services.
- **WebSockets** – Supported on paid Web Services; no extra config for realtime-gateway.
- **Environments** – Use separate Render services per environment (e.g. identity-service-dev, identity-service-staging, identity-service-prod). Map GitHub branch per service (e.g. branch `development` → dev service).
- **GitHub integration** – Connect repo(s), auto-deploy from branch; build and start commands in dashboard; env vars (including `NPM_TOKEN` for private npm) work as expected.
- **Pricing** – Free tier for trying; paid tiers straightforward. No surprise "apply changes" failures.

### How it maps

- **identity-service** → Render Web Service (Node; build/start: see [render-setup.md](render-setup.md); run migrations in start: `npx prisma migrate deploy && npm start`).
- **core-service** → Render Web Service (same pattern).
- **realtime-gateway** → Render Web Service (Node, WebSockets on paid).
- **relay** → Render Background Worker (same repo as core-service; start: `node dist/relay/main.js`).
- **PostgreSQL** → Two Render Postgres instances (identity-db, core-db); link via `DATABASE_URL`.
- **Redis** → One Render Redis; link via `REDIS_URL` to core-service, realtime-gateway, relay.

See **[render-setup.md](render-setup.md)** for step-by-step setup per environment.

---

## Alternative 1: **Railway**

Same topology (services + Postgres x2 + Redis, environments). Can work when the platform is stable; some users hit "unknown error" when creating services or adding envs. Prefer Render for reliability.

---

## Alternative 2: **Azure** (enterprise / Microsoft stack)

Use this if you need enterprise compliance, existing Azure/Microsoft investment, or want to keep the Azure-focused GitHub Actions you already have.

- **Compute:** Azure App Service (Web App) per Node app (identity, core, realtime-gateway); relay as another App Service or Azure Function (timer).
- **Data:** Azure Database for PostgreSQL (Flexible Server) – one server, two databases, or two servers; Azure Cache for Redis.
- **Environments:** Separate App Service plans or resource groups for dev/staging/prod.
- **Pros:** Enterprise-ready, SLA, VNet, GitHub Actions `azure/webapps-deploy` already in place.
- **Cons:** More configuration and cost than Render for a small team.

---

## Alternative 3: **AWS**

Use when you need maximum scale, existing AWS footprint, or specific AWS services.

- **Compute:** ECS Fargate (containers) or App Runner per service; relay as ECS task or Lambda (scheduled).
- **Data:** RDS PostgreSQL (one instance, two DBs, or Aurora); ElastiCache Redis.
- **Pros:** Industry standard, very scalable, rich ecosystem.
- **Cons:** Steep learning curve and more moving parts; overkill for early stage.

---

## Summary table

| Criteria           | Render     | Railway     | Azure        | AWS        |
|--------------------|------------|------------|-------------|------------|
| Setup simplicity   | ★★★★      | ★★★★★*     | ★★★         | ★★         |
| Reliability / UI   | Stable     | Variable   | Stable      | Stable     |
| Postgres + Redis   | Built-in   | Built-in   | Built-in    | RDS + ElastiCache |
| WebSockets         | Yes (paid) | Yes        | Yes         | Yes        |
| 3 envs (dev/stg/pr)| Services   | Environments | Resource groups | Accounts/VPCs |
| Cost (early stage) | Low        | Low        | Medium      | Medium     |
| GitHub / CI        | Auto-deploy| CLI + API  | Actions ✅  | Actions    |

*Railway can be simple when it works; some users hit creation/UI errors.

---

## Recommendation

- **Use Render** for the whole stack: stable UI, one place for all services, Postgres (x2), and Redis; clear path for dev/staging/production. Step-by-step: **[render-setup.md](render-setup.md)**.
- **Keep CI in GitHub Actions** (build + test); let Render auto-deploy from branch, or trigger via Render API/CLI after CI passes.
- **Move to Azure or AWS** later if you need enterprise compliance or existing cloud contracts; app design and three-branch flow stay the same.
