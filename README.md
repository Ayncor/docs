# ayncor docs

Short reference for the ayncor platform.

| Doc | Description |
|-----|-------------|
| [contracts/](../contracts/README.md) | **Single source of truth** for APIs. OpenAPI specs (identity-service, core-service), realtime protocol. Postman collections in each service. |
| [CI-CD.md](./CI-CD.md) | Branch strategy (development → staging → main), GitHub Actions, deployment (identity-service Azure), E2E, NPM_TOKEN. |
| [hosting-recommendation.md](./hosting-recommendation.md) | Hosting options: Render (recommended), Azure, AWS; what to run where. |
| [render-setup.md](./render-setup.md) | Step-by-step Render setup: Postgres (x2), Redis, deploy identity-service, core-service, realtime-gateway, relay per environment. |
| [railway-setup.md](./railway-setup.md) | Step-by-step Railway setup: Postgres (x2), Redis, deploy identity-service, core-service, realtime-gateway, relay across 3 environments. |
| [onboarding-extension.md](./onboarding-extension.md) | Self-signup, org creation during onboarding, optional team invites, org settings (allowed email domains). |
| [frontend-architecture.md](./frontend-architecture.md) | Web + Desktop monorepo (ayncor-client), Mobile separate repo; framework choices, production standards, shared code. |
| [async_comm_architecture.mermaid](./async_comm_architecture.mermaid) | Architecture diagram (identity, core, realtime-gateway, relay, Redis, future services). |
