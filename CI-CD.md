# CI/CD: Branch strategy and setup

This doc describes the three-branch strategy and how to wire CI/CD across **identity-service**, **core-service**, **realtime-gateway**, and **ayncor-e2e**.

---

## Branch strategy

| Branch        | Purpose |
|---------------|--------|
| **development** | Day-to-day work. Contributors merge feature branches here. Sync point for local dev. |
| **staging**     | Pre-production. Changes promoted from `development`. Deploy to staging environment. |
| **main** (or **master**) | Production. Changes promoted from `staging`. Deploy to production. |

**Flow:** feature → `development` → `staging` → `main`

- Create **development** and **staging** in each repo (main/master already exist).
- Do not merge directly to `staging` or `main` from feature branches; always flow through the pipeline.

---

## What’s in place

1. **Branches**  
   You create `development` and `staging` in each repo.

2. **GitHub Actions workflows** (in each repo under `.github/workflows/`)  
   - **ci.yml** – runs on push and pull_request to `development`, `staging`, and `main`:
     - `npm ci`
     - `prisma generate` (identity-service, core-service only)
     - `npm run build`
     - `npm test`
   - Same job definition for all three branches; later you can add branch-specific steps (e.g. deploy only on `staging` / `main`).

3. **ayncor-e2e**  
   CI runs `npm ci` and `npm test`. E2E tests expect running services (identity, core, realtime-gateway). For now they either:
   - run in CI only when services are available (e.g. from staging URLs), or  
   - are skipped in CI and run manually / in a separate “full E2E” pipeline.  
   The workflow is set up so you can add `IDENTITY_URL`, `CORE_URL`, `REALTIME_WS_URL` (and secrets) when you have a staging environment.

---

## Next steps (your side)

### 1. Create branches (once per repo)

In **identity-service**, **core-service**, **realtime-gateway**, **ayncor-e2e**:

```bash
git checkout main   # or master
git pull
git checkout -b development
git push -u origin development

git checkout main   # or master
git checkout -b staging
git push -u origin staging
```

### 2. Branch protection (GitHub)

For each repo, in **Settings → Branches → Branch protection rules**:

- **main** (or **master**):
  - Require PR before merging.
  - Require status checks (e.g. “CI” or the name of your workflow job).
  - Restrict who can push (optional).
  - Only allow merge from `staging` (enforce via process or a second rule).

- **staging**:
  - Require PR before merging.
  - Require status checks.
  - Only allow merge from `development` (enforce via process).

- **development**:
  - Optional: require PR and status checks for feature merges.

### 3. Deployment

**identity-service** has deploy jobs in `.github/workflows/ci.yml` (Azure Web App). They run only on **push** to the branch (not on PRs).

- **deploy-development:** runs after build-and-test on push to `development`; deploys to the development Azure Web App.
- **deploy-staging:** runs after build-and-test on push to `staging`; deploys to the staging Azure Web App.
- **deploy-production:** runs after build-and-test on push to `main`; deploys to the production Azure Web App.

Deploy steps are set to `continue-on-error: true` until you add the secrets; then set them to `false`.

#### identity-service: Azure Web App setup

1. **Create three Azure App Service Web Apps** (development, staging, production), Node 20 runtime, same region.
2. **Download each app’s Publish profile:** Azure Portal → App Service → **Get publish profile** (or Overview → **Download publish profile**).
3. **In identity-service repo → Settings → Secrets and variables → Actions:**
   - **Secrets:**  
     - `AZURE_WEBAPP_PUBLISH_PROFILE_DEVELOPMENT` = development app’s publish profile (whole XML).  
     - `AZURE_WEBAPP_PUBLISH_PROFILE_STAGING` = staging app’s publish profile.  
     - `AZURE_WEBAPP_PUBLISH_PROFILE` = production app’s publish profile.
   - **Variables:**  
     - `AZURE_WEBAPP_NAME_DEVELOPMENT` (e.g. `ayncor-identity-dev`).  
     - `AZURE_WEBAPP_NAME_STAGING` (e.g. `ayncor-identity-staging`).  
     - `AZURE_WEBAPP_NAME` (e.g. `ayncor-identity`).
4. **App settings in Azure:** for each app, set `DATABASE_URL`, `JWT_ACCESS_SECRET`, `BOOTSTRAP_EMAIL`, `BOOTSTRAP_PASSWORD`, `BOOTSTRAP_ORG_SLUG` (and any others from `.env.example`) in **Configuration → Application settings**. Run Prisma migrations (e.g. once per app or from a release step).
5. When everything is configured, in the workflow set `continue-on-error: false` for the deploy steps.

**Other platforms (Render, Fly.io, etc.):** replace the `azure/webapps-deploy` step with that platform’s action or CLI; the rest of the job (build, prune, package) can stay.

### 4. E2E against staging

When staging is deployed:

- In **ayncor-e2e** add secrets (e.g. `STAGING_IDENTITY_URL`, `STAGING_CORE_URL`, `STAGING_REALTIME_WS_URL`, `E2E_EMAIL`, `E2E_PASSWORD`, `E2E_ORG_SLUG`).
- In the E2E workflow, run `npm test` with those env vars (and optionally only on `staging` or `main`).

### 5. Other hosts (Azure DevOps, GitLab, etc.)

The same branch strategy applies. Replace the GitHub Actions workflows with equivalent pipelines (e.g. Azure Pipelines, GitLab CI) that run the same steps on `development`, `staging`, and `main`.

---

## Repos and workflows

| Repo              | Workflow steps |
|-------------------|----------------|
| identity-service  | npm ci, prisma generate, build, test |
| core-service      | npm ci, prisma generate, build, test |
| realtime-gateway  | npm ci, build, test |
| ayncor-e2e        | npm ci, test (point at staging URLs when configured) |

### GitHub Packages (@ayncor/contracts)

identity-service, core-service, and realtime-gateway depend on **@ayncor/contracts** from a separate **contracts** repo (GitHub Packages). CI needs a PAT with `read:packages` to install it.

- In **identity-service**, **core-service**, and **realtime-gateway**: add a repo secret **NPM_TOKEN** with your PAT (same one you use to publish contracts).
- The workflow uses `NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}` so `npm ci` can authenticate to `npm.pkg.github.com`.

---

## Monorepo vs separate repos

- **Separate repos:** Each repo has its own `.github/workflows/ci.yml` (e.g. `identity-service/.github/workflows/ci.yml`). Push each repo to its own remote; CI runs per repo.
- **Single repo (monorepo):** GitHub only runs workflows under the repo root. Put one `.github/workflows/` at the **root** of the monorepo and use a matrix or multiple jobs that `cd` into `identity-service`, `core-service`, etc., then run `npm ci`, `npm run build`, `npm test`. The workflow files under each service folder then serve as reference; copy or merge them into root `.github/workflows/` (e.g. `ci-identity.yml`, `ci-core.yml`).

## Summary

1. Create **development** and **staging** in all four repos (or in the single repo).  
2. Push the added `.github/workflows/ci.yml` to each repo (or merge into root `.github/workflows/` if monorepo); CI will run on push/PR to `development`, `staging`, and `main`.  
3. Configure branch protection for `main` and `staging`.  
4. Add deployment jobs when you have staging/production targets.  
5. Optionally wire E2E to staging via repo vars/secrets (IDENTITY_URL, CORE_URL, REALTIME_WS_URL, E2E_EMAIL, E2E_PASSWORD).
