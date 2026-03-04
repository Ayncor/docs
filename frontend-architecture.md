# Frontend Architecture — Production Standards

Architecture for **ayncor** frontend: Web, Desktop, and Mobile across two repos with strict production-grade standards.

---

## 1. Repo Split

| Repo | Contents | Rationale |
|------|----------|-----------|
| **ayncor-client** (monorepo) | Web app + Desktop app | Shared React code, shared UI, same auth flow. Electron/Tauri loads the same UI as web. |
| **ayncor-mobile** (separate repo) | iOS + Android | Different runtime (React Native/Expo), different release cadence, App Store/Play Store workflows. |

**Why split:** Mobile has different build pipelines (Xcode, Gradle), store submissions, and device-specific UX. Keeping it separate avoids monorepo complexity for native tooling.

---

## 2. Monorepo Structure (ayncor-client)

```
ayncor-client/
├── apps/
│   ├── web/          # Next.js (or Vite + React)
│   └── desktop/      # Electron or Tauri shell
├── packages/
│   ├── shared/       # Types, utils, constants, API client
│   ├── ui/           # Shared React components (design system)
│   └── config/       # Shared ESLint, TS, Prettier configs
├── package.json      # Workspace root
├── pnpm-workspace.yaml
├── turbo.json
└── .env.example
```

---

## 3. Framework Choices

### Web

| Option | Pros | Cons |
|--------|------|------|
| **Next.js 14+ (App Router)** | SSR, RSC, auth patterns, SEO, industry standard | Heavier, some magic |
| **Vite + React Router** | Fast, explicit, SPA-only | No SSR out of box |
| **Remix** | Explicit data loading, web standards | Smaller ecosystem |

**Recommendation: Next.js 14+ (App Router)** — Production standard, mature auth patterns (middleware, server components), good for dashboards and SEO. Use `@tanstack/react-query` for client state.

### Desktop

| Option | Pros | Cons |
|--------|------|------|
| **Electron** | Mature, large ecosystem, same codebase as web | Heavy (~150MB+), Chromium bundled |
| **Tauri 2** | Light (~10MB), native WebView, Rust security | Newer, smaller ecosystem |

**Recommendation: Electron** for production reliability and ecosystem. **Tauri** if bundle size and native feel are critical. Both can load the same web app (Next.js or static build).

### Mobile (ayncor-mobile)

| Option | Pros | Cons |
|--------|------|------|
| **Expo (React Native)** | OTA updates, managed workflow, simpler setup | Less control over native |
| **React Native (bare)** | Full control | More setup |

**Recommendation: Expo** — Production-ready, over-the-air updates, good for SaaS.

---

## 4. Shared Code Strategy

- **Shared:** Types, API client, auth logic, business rules, constants.
- **Shared UI:** `packages/ui` — components built with **shadcn/ui** (Radix primitives) or **Headless UI** for consistency and accessibility.
- **API client:** Generated from `@ayncor/contracts` OpenAPI specs (e.g. `openapi-typescript` + `openapi-fetch`).
- **Auth:** Shared token storage, refresh logic, org context. Web uses cookies/httpOnly; desktop uses secure storage; mobile uses Keychain/Keystore.

---

## 5. Production Standards

### 5.1 Package Manager & Monorepo

- **pnpm** — Strict, fast, disk-efficient. `pnpm-workspace.yaml` for workspace definition.
- **Turborepo** — Caching, parallel builds, task pipelines. `turbo.json` for build/test/lint.

### 5.2 TypeScript

- **strict: true** in all `tsconfig.json`.
- **noUncheckedIndexedAccess** — recommended.
- Shared `packages/config/typescript` for base config.

### 5.3 Linting & Formatting

- **ESLint** 9+ flat config — `@eslint/js`, `typescript-eslint`, `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`.
- **Prettier** — Single config, no conflicts with ESLint.
- **Pre-commit:** Husky + lint-staged — run lint + typecheck on staged files.

### 5.4 Testing

- **Unit:** Vitest (fast, Vite-native).
- **E2E:** Playwright for web; Spectron/Playwright for Electron.
- **Coverage:** Minimum thresholds enforced in CI.

### 5.5 CI/CD

- **Lint, typecheck, test** on every PR.
- **Build** all apps in monorepo.
- **Deploy:** Web → Vercel/Cloudflare; Desktop → GitHub Releases (auto-update via electron-updater).

### 5.6 Security

- **No secrets in code** — env vars only.
- **CSP headers** — Content-Security-Policy for web.
- **Token storage:** httpOnly cookies (web), secure storage (desktop), Keychain (mobile).

### 5.7 Accessibility

- **WCAG 2.1 AA** — Target for all UI.
- **axe-core** in CI for automated a11y checks.
- **Keyboard navigation** — Full support.

---

## 6. Environment Variables

| App | Prefix | Example |
|-----|--------|---------|
| Web | `NEXT_PUBLIC_` | `NEXT_PUBLIC_IDENTITY_URL` |
| Desktop | `VITE_` or `ELECTRON_` | `VITE_identity_url` |
| Shared | — | `IDENTITY_URL` in shared config |

**Local dev defaults:**
- `IDENTITY_URL=http://localhost:3001`
- `CORE_URL=http://localhost:3002`
- `REALTIME_WS_URL=ws://localhost:3010`

---

## 7. API Integration

- **Contracts:** `@ayncor/contracts` from GitHub Packages (or local path in monorepo).
- **Client generation:** `openapi-typescript` + `openapi-fetch` from OpenAPI specs.
- **Auth:** `Authorization: Bearer <access_token>`; refresh on 401.

---

## 8. Alternative: Tauri + Vite for Web + Desktop

If you prefer **Tauri** over Electron:

- **Web:** Vite + React + React Router (SPA). Deploy to Vercel/Cloudflare.
- **Desktop:** Tauri shell loads the same Vite build (or a Tauri-specific build).
- **Shared:** Same `packages/shared` and `packages/ui`.

**Trade-off:** Tauri is lighter; Electron has more mature tooling and auto-update patterns.

---

## 9. Implementation Order

1. **Scaffold monorepo** — pnpm, Turborepo, workspace structure.
2. **packages/shared** — Types, API client stub, env config.
3. **packages/ui** — Design system base (shadcn/ui).
4. **apps/web** — Next.js app, auth, signup/login, basic layout.
5. **apps/desktop** — Electron shell loading web (or Tauri).
6. **ayncor-mobile** — Separate repo, Expo scaffold, later.

---

## 10. Checklist

- [ ] pnpm workspace + Turborepo
- [ ] TypeScript strict
- [ ] ESLint + Prettier + Husky
- [ ] packages/shared, packages/ui
- [ ] apps/web (Next.js)
- [ ] apps/desktop (Electron)
- [ ] API client from contracts
- [ ] Auth flow (login, signup, refresh)
- [ ] Vitest + Playwright
- [ ] CI pipeline
