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

## Recommended: **Railway**

**Best fit** for your current stage: multi-service Node stack, Postgres, Redis, WebSockets, three environments, and industry-aligned DX.

### Why Railway

- **One platform** – Deploy identity, core, realtime-gateway, and relay as services; add Postgres (x2) and Redis from the dashboard. No VMs or Kubernetes to manage.
- **Managed Postgres and Redis** – Provisioned in a few clicks; connection strings and env vars wired automatically.
- **WebSockets** – Fully supported; no extra config for realtime-gateway.
- **Environments** – Use Railway “environments” (e.g. development, staging, production) and attach services per environment. Map GitHub branches (e.g. `development` → dev, `staging` → staging, `main` → production) via GitHub integration or CI/CD.
- **GitHub integration** – Connect repo(s), optional auto-deploy from branch; or keep using your existing GitHub Actions and deploy via Railway CLI/API.
- **Pricing** – Usage-based; affordable for small teams; free trial to validate.
- **Industry practice** – PaaS for speed and reliability; managed DBs; clear path to scale or move to Azure/AWS later if needed.

### How it maps

- **identity-service** → Railway Web Service (Node, build: `npm ci && prisma generate && npm run build`, start: `npm start`).
- **core-service** → Railway Web Service (same pattern).
- **realtime-gateway** → Railway Web Service (Node, WebSockets supported).
- **relay** → Railway Background Worker or separate service (long-running process; same repo as core or separate).
- **PostgreSQL** → Two Railway Postgres plugins (one for identity, one for core), or one Postgres with two databases.
- **Redis** → Railway Redis plugin; same instance can serve realtime-gateway and relay.

Use **Railway environments** for development / staging / production and attach the same service layout to each.

---

## Alternative 1: **Azure** (enterprise / Microsoft stack)

Use this if you need enterprise compliance, existing Azure/Microsoft investment, or want to keep the Azure-focused GitHub Actions you already have.

- **Compute:** Azure App Service (Web App) per Node app (identity, core, realtime-gateway); relay as another App Service or Azure Function (timer).
- **Data:** Azure Database for PostgreSQL (Flexible Server) – one server, two databases, or two servers; Azure Cache for Redis.
- **Environments:** Separate App Service plans or resource groups for dev/staging/prod.
- **Pros:** Enterprise-ready, SLA, VNet, GitHub Actions `azure/webapps-deploy` already in place.
- **Cons:** More configuration and cost than Railway for a small team.

---

## Alternative 2: **Render**

Good middle ground: simple PaaS, managed Postgres and Redis, GitHub auto-deploy.

- **Compute:** Web Services for identity, core, realtime-gateway; Background Worker for relay.
- **Data:** Render Postgres (x2) and Redis.
- **Environments:** Separate services per environment or use Render “environments” where available.
- **Pros:** Simple, good free tier for trying, WebSockets on paid tiers.
- **Cons:** Cold starts on free tier; less flexible than Railway for fine-grained env mapping.

---

## Alternative 3: **AWS**

Use when you need maximum scale, existing AWS footprint, or specific AWS services.

- **Compute:** ECS Fargate (containers) or App Runner per service; relay as ECS task or Lambda (scheduled).
- **Data:** RDS PostgreSQL (one instance, two DBs, or Aurora); ElastiCache Redis.
- **Pros:** Industry standard, very scalable, rich ecosystem.
- **Cons:** Steep learning curve and more moving parts; overkill for early stage.

---

## Summary table

| Criteria           | Railway     | Azure        | Render    | AWS        |
|--------------------|------------|-------------|-----------|------------|
| Setup simplicity   | ★★★★★      | ★★★         | ★★★★      | ★★         |
| Postgres + Redis   | Built-in   | Built-in    | Built-in  | RDS + ElastiCache |
| WebSockets         | Yes        | Yes         | Yes (paid)| Yes        |
| 3 envs (dev/stg/pr)| Environments | Resource groups | Services | Accounts/VPCs |
| Cost (early stage) | Low        | Medium      | Low       | Medium     |
| Enterprise / compliance | Good   | Strong      | Good      | Strong     |
| GitHub / CI        | CLI + API  | Actions ✅  | Auto-deploy | Actions   |

---

## Recommendation

- **Start with Railway** for the whole stack: one place for all services, Postgres, and Redis; clear dev/staging/production; good fit for async-first Node + WebSockets + relay.
- **Keep or adjust CI/CD:** Either use Railway’s GitHub integration for deploys, or keep GitHub Actions and add a “deploy to Railway” step (Railway CLI or API) instead of (or in addition to) Azure deploy.
- **Move to Azure (or AWS) later** if you need enterprise compliance, stricter networking, or existing cloud contracts; the three-branch flow and app design stay the same.

If you tell me which platform you prefer (Railway, Azure, or Render), I can outline concrete steps and, for identity-service, suggest exact workflow changes (e.g. swap Azure deploy for Railway deploy).
