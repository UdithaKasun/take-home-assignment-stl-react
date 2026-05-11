
# Take Home Assignment - Senior Tech Lead (Frontend) - Uditha Senavirathna

This document provides following sections related to the take home assignment provided.

- Architecture Overview
- Routing & Navigation
- Design System Sharing
- Authentication
- Independent Build, Test & Deploy
- Key Operational Details
- Risks & Mitigations
- Config Snippets



## 1. Architecture Overview

A single dashboard product owned by four independent frontend teams. Each team develops, tests, and deploys their feature area without coordinating with other teams. The architecture is built on Webpack Module Federation, a CDN-based deployment model, and a single BFF that centralises auth and routing to downstream microservices.

<img width="2184" height="1042" alt="image" src="https://github.com/user-attachments/assets/206d94d0-50ec-4cf6-9cfa-744604065b8a" />
<br/>
Google Drive Link - https://drive.google.com/file/d/1szkyvgYeJKrJQdJjsbYO5Y2vkzGWTikR/view?usp=sharing

<br/>
<br/>

**1.1 Shell - Module Federation host**

The shell is the entry point of the application. The shell has no knowledge of what any MFE does internally. It only knows where to load them from and how to mount them. Its responsibilities are:

- **Auth guard** (@company/auth) - runs before anything else. Validates the existing session, holds the JWT access token in memory, and handles silent token refresh against the BFF. No MFE is mounted until auth resolves.
- **Top-level routing** - owns the four top-level route prefixes `/analytics/*`, `/billing/*`, `/settings/*`, `/home/*` and maps each to its corresponding MFE slot.
- **Error isolation** - every MFE slot is wrapped in an ErrorBoundary and Suspense. If a remote fails to load or crashes at runtime, that slot shows a skeleton fallback. The shell and all other MFEs continue working normally.

**1.2 Routing**

Routing follows a two-tier hybrid strategy. The shell owns the top level and each MFE owns everything beneath it.

***Shell-level routing***

The shell uses React Router to define four top-level routes. Each route is a lazy-loaded MFE slot wrapped in an ErrorBoundary and Suspense:

- `/analytics/*` - mounts the Analytics MFE
- `/billing/*` - mounts the Billing MFE
- `/settings/*` - mounts the Settings MFE
- `/home/*` - mounts the Home MFE

The shell intercepts navigation events from the browser URL bar and decides which MFE to mount. It does not know anything about sub-routes inside each MFE.

***MFE-level routing***

Each MFE receives two props from the shell - `basePath` and `initialPath`. The MFE uses a `MemoryRouter` internally with `initialPath` as the starting entry. This means:

- Each MFE owns its own routing history stack completely independently
- Teams can add, remove, or restructure internal routes without touching the shell
- Deep links work because the shell strips the top-level prefix and passes the remainder to the MFE as `initialPath` on mount

***Cross-MFE navigation***

When an MFE needs to navigate to a route owned by a different MFE (for example a billing page linking to settings), it dispatches a custom event on `window.__shell_bus__`. The shell listens for this event and performs a top-level React Router push. This keeps MFEs decoupled - no MFE imports the shell router directly.

***URL ownership summary***

- Shell owns: `/analytics`, `/billing`, `/settings`, `/home`
- Analytics MFE owns: everything under `/analytics/*`
- Billing MFE owns: everything under `/billing/*`
- Settings MFE owns: everything under `/settings/*`
- Home MFE owns: everything under `/home/*`


**1.3 MFE remotes**

Each of the four teams owns one MFE:

- **Analytics MFE** - Team A
- **Billing MFE** - Team B
- **Settings MFE** - Team C
- **Home MFE** - Team D

Each MFE is an independently deployable unit. A team runs their own CI pipeline, builds their own `remoteEntry.js`, and publishes it to their own CDN path. No other team or the shell needs to redeploy when a team ships. Teams expose exactly one entry point to the shell - a single React component that accepts a `basePath` and `initialPath` prop. Everything else inside the MFE is private.


**1.4 UI library - @company/ui**

The shared design system is published as a versioned npm package to a private registry. Each MFE team pins their own version in their `package.json` and bundles it into their own `remoteEntry.js` at build time. This means:

- Team A can be on `v2.4.0` while Team C is still on `v2.2.0`
- No runtime version conflict is possible because each MFE carries its own isolated copy
- Teams upgrade the design system on their own schedule without blocking each other

React itself is still a singleton shared through Module Federation - only one React instance ever runs in the browser. `@company/ui` components from different versions coexist safely because they all consume that single React instance.

How Drift Prevention is handled ?

- `@deprecated` JSDoc on any prop scheduled for removal - shows as strikethrough in editors across all MFE codebases immediately
- Changeset required - CI blocks any PR touching `packages/ui` without a `.changeset/*.md` file
- Visual regression via Chromatic - pixel diffs across all Storybook stories on every PR
- Every major bump ships a jscodeshift codemod so teams get an auto-applied migration rather than manual work

**1.5 CDN and runtime URL resolution**

Each team's CI pipeline uploads their built `remoteEntry.js` to a versioned CDN path. The shell does not have remote URLs hardcoded. Instead it reads a `config.js` file at boot which populates `window.__MFE_URLS__` with the current URL for each remote. This means:

- Updating which version of an MFE loads in production is a `config.js` change only - no shell rebuild, no shell redeploy
- Rolling back a bad MFE deploy means pointing `config.js` back to the previous version - takes effect within 60 seconds (the `config.js` CDN TTL)
- Each remote is loaded on demand only when the user navigates to that route, keeping initial load fast

**1.6 Auth Integration**

The shell owns the login flow. The JWT access token lives inside the `@company/auth` module's in-memory store, never in `localStorage`. All MFEs import `useAuth()` from the shared singleton and get the token without owning any refresh logic.

**Token flow**

    1. Shell mounts and calls `auth.init()` - attempts silent refresh via HttpOnly refresh token cookie
    2. If refresh fails, redirect to login
    3. Shell awaits `auth.ready()` before mounting any MFE slot - eliminates race conditions on page load
    4. Access tokens are short-lived (15 min). `getToken()` handles silent refresh transparently via the BFF
    5. On logout, `auth.logout()` clears the in-memory token and all MFE `useAuth()` hooks reactively return null

**Security**

- Refresh token cookie: `HttpOnly; Secure; SameSite=Strict; Domain=.company.com`
- API requests use `Authorization: Bearer` header only - no cookie sent to APIs
- All requests validated at the BFF auth middleware before reaching any microservice

**1.7 Key properties of the architecture**

- **No team blocks another** - each team deploys their MFE independently. A broken billing deploy does not affect analytics.
- **Auth is a chokepoint by design** - there is one place where JWT validation happens. No MFE has a path to a microservice that bypasses it.
- **Routing is a shared contract not a shared codebase** - the shell owns top-level prefixes and each MFE owns everything beneath. Neither side needs to know the other's internal structure.
- **The shell is stable and rarely changes** - feature work happens inside MFEs. The shell only changes when routing, auth behaviour, or the shared error handling strategy changes.
- **Rollback is fast** - because `remoteEntry.js` is versioned on the CDN and the shell resolves URLs from `config.js`, rolling back any MFE is a config change with no code deployment.
- **UI consistency is a contract not a runtime dependency** - the design system is versioned npm. Visual regression CI gates catch drift before it reaches production.

**1.8 Independent build, test and deploy**

Each MFE has its own CI pipeline triggered by path filters in the monorepo. A change in `mfes/billing/**` only triggers the billing pipeline. The shell deploys separately.

Each team's pipeline:

    1. Type check
    2. Unit tests with coverage
    3. Run Integration & E2E tests
    4. Build `remoteEntry.js` and hashed chunks
    5. Upload hashed chunks to CDN with `max-age=31536000,immutable`
    6. Upload `remoteEntry.js` with `max-age=60`
    7. Invalidate `remoteEntry.js` on CDN edge
    8. Update `config.js` to point to the new version

**1.9 Sample folder structure for mono repo**

```
dashboard-monorepo/
├── apps/
│   ├── shell/
│   │   ├── src/
│   │   │   ├── main.tsx
│   │   │   ├── routes.tsx
│   │   │   ├── AppShell.tsx
│   │   │   ├── components/
│   │   │   │   ├── MFEErrorBoundary.tsx
│   │   │   │   └── MFESkeleton.tsx
│   │   │   ├── lib/
│   │   │   │   └── loadRemote.ts
│   │   │   └── types/
│   │   │       └── remotes.d.ts
│   │   ├── public/
│   │   │   └── config.js
│   │   ├── webpack.config.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── analytics/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes.tsx
│   │   │   ├── pages/
│   │   │   └── components/
│   │   ├── webpack.config.ts
│   │   ├── jest.config.ts
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── Dockerfile
│   ├── billing/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes.tsx
│   │   │   ├── pages/
│   │   │   └── components/
│   │   ├── webpack.config.ts
│   │   ├── jest.config.ts
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── Dockerfile
│   ├── settings/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── routes.tsx
│   │   │   ├── pages/
│   │   │   └── components/
│   │   ├── webpack.config.ts
│   │   ├── jest.config.ts
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── Dockerfile
│   └── home/
│       ├── src/
│       │   ├── index.ts
│       │   ├── routes.tsx
│       │   ├── pages/
│       │   └── components/
│       ├── webpack.config.ts
│       ├── jest.config.ts
│       ├── package.json
│       ├── tsconfig.json
│       └── Dockerfile
│
├── packages/
│   ├── ui/
│   │   ├── src/
│   │   │   ├── Button/
│   │   │   ├── Input/
│   │   │   ├── Modal/
│   │   │   └── index.ts
│   │   ├── tokens/
│   │   │   ├── colors.ts
│   │   │   ├── spacing.ts
│   │   │   └── typography.ts
│   │   ├── codemods/
│   │   │   └── button-type-to-variant.ts
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── CHANGELOG.md
│   ├── auth/
│   │   ├── src/
│   │   │   ├── useAuth.ts
│   │   │   ├── getToken.ts
│   │   │   └── AuthProvider.tsx
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── analytics-sdk/
│   │   ├── src/
│   │   │   ├── track.ts
│   │   │   └── events.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── utils/
│   │   ├── src/
│   │   │   ├── formatters/
│   │   │   ├── hooks/
│   │   │   └── types/
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── eslint-config/
│   │   ├── index.js
│   │   └── package.json
│   └── tsconfig/
│       ├── base.json
│       └── package.json
│
├── infra/
│   └── .github/
│       └── workflows/
│           ├── shell.yml
│           ├── mfe-analytics.yml
│           ├── mfe-billing.yml
│           ├── mfe-settings.yml
│           ├── mfe-home.yml
│           └── packages.yml
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── OPERATIONAL.md
│   └── RISKS.md
│
├── nx.json
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.base.json
├── .eslintrc.base.js
├── .gitignore
├── CODEOWNERS
└── README.md
```

**1.10 BFF - Backend for Frontend**

A single Node.js BFF acts as the only backend the browser ever talks to. All requests from all four MFEs go through it. It has two layers:

**Auth middleware** sits at the front of every request. It validates the JWT before any routing decision is made. It also handles token refresh requests from the shell auth guard directly. No MFE can reach any downstream service without passing through this layer.

**Route handlers** sit behind the auth middleware:

- **Analytics routes** - proxies to the Analytics microservice
- **Billing routes** - proxies to the Billing and User microservices
- **Settings routes** - proxies to the User and Notification microservices
- **Home aggregator** - fans out to Analytics, Billing, and User microservices in parallel and aggregates the responses into a single payload for the home dashboard

## 2. Key Operational Details

**2.1 Shared dependencies and avoiding duplicate bundles**

React, React Router, and `@company/auth` are declared as `singleton: true` in Module Federation shared config. They are loaded once by the shell and resolved from the shared scope by all remotes - no duplication. A CI script asserts that singleton dep versions are identical across all `package.json` files in the monorepo. Any mismatch fails the build before anything ships.

`@company/ui` is intentionally not a singleton - each MFE bundles its own version to allow independent upgrade schedules.

**2.2 CSS isolation strategy**

- All MFE components use CSS Modules - class names are scoped to `[filename]__[class]__[hash]` at build time
- No global class names permitted in MFE code, enforced by ESLint rule
- Design tokens are CSS custom properties on `:root` set by the shell - MFEs consume `var(--color-primary)` without writing global CSS
- `@company/ui` components use CSS Modules internally so consuming MFEs cannot accidentally override their internals

**2.3 Integration and E2E testing**

- **Unit tests** - each MFE runs Jest and React Testing Library in isolation
- **Integration** - Playwright tests in `apps/shell/e2e/` spin up the full shell and all MFEs in local dev mode and test cross-MFE flows
- **Visual regression** - Chromatic on `packages/ui` catches design system regressions before they reach any MFE
- **Contract testing** - Pact consumer-driven contracts so each MFE verifies its integration with the shell event bus without needing a live shell

**2.4 Deployment model and CI/CD outline**

Each MFE is deployed independently. The shell never redeploys when an MFE ships. The deployment sequence for any MFE:

    1. CI builds `remoteEntry.js` and hashed JS chunks
    2. Hashed chunks uploaded to `cdn.company.com/mfe/{name}/{version}/` with immutable cache headers
    3. `remoteEntry.js` uploaded with 60-second TTL
    4. `config.js` updated to point the shell at the new version
    5. CDN invalidation fired for `remoteEntry.js` and `config.js`

Rollback means updating `config.js` to point back to the previous version. No code deployment. Takes effect within 60 seconds.

**2.5 Observability and monitoring**

- **Distributed tracing** - each MFE wraps fetch calls with `tracedFetch` from `@company/utils` which injects `X-Trace-Id` and `X-Span-Id` headers. Traces joinable in Datadog APM.
- **MFE crash tracking** - `MFEErrorBoundary` sends `mfe_crash` events with MFE name, error message, and component stack. A Datadog monitor alerts if any MFE crash rate exceeds 0.1% over 5 minutes.
- **Synthetic monitoring** - Playwright tests run on a 5-minute schedule against production covering critical paths across MFE boundaries.
- **CDN cache hit rate** - tracked per MFE path. A drop below 90% indicates a deployment or cache misconfiguration issue.
- **Bundle size** - `size-limit` runs in CI and posts a PR comment with the delta. A regression over 5% fails the build.

**2.6 Security considerations**

- Access tokens stored in-memory only inside the `@company/auth` module closure. Never written to `localStorage` or `sessionStorage`.
- Refresh tokens in `HttpOnly; Secure` cookies scoped to the auth subdomain only - not readable by MFE JavaScript.
- APIs use strict CORS with an explicit allowlist of `company.com` origins.
- Shell sets a `Content-Security-Policy` header. `script-src` includes only the CDN origin and `strict-dynamic`. MFEs loaded from outside this origin are blocked.
- Dependabot or Snyk runs on the monorepo. High-severity CVEs in shared packages block all MFE deployments until patched.

## 3. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| MFE crash brings down the dashboard | High | `MFEErrorBoundary` per slot - failed MFE renders a fallback, rest of dashboard unaffected |
| Singleton version mismatch (React loaded twice, hooks break) | High | CI version-check script blocks any PR with mismatched singleton deps across the monorepo |
| Design system breaking change breaks MFEs | Medium | `@deprecated` JSDoc notice period + codemod shipped with every major bump |
| Performance regression at first load | Medium | `remoteEntry.js` preloaded via `<link rel="modulepreload">` hints, hashed chunks edge-cached with 1-year TTL |
| CSS bleed between MFEs | Medium | CSS Modules enforced by ESLint, design tokens via CSS custom properties, no global class names |
| Auth token race condition on page load | Medium | Shell awaits `auth.ready()` before rendering any MFE slot |

## 4. Deliverable Extras

**4.1 Bootstrap snippet - how MFEs are discovered and loaded at runtime**

```js
// apps/shell/public/config.js - served by CDN, TTL 60s
window.__MFE_URLS__ = {
  analytics: 'https://cdn.company.com/mfe/analytics/v2.4.1/remoteEntry.js',
  billing:   'https://cdn.company.com/mfe/billing/v1.9.0/remoteEntry.js',
  settings:  'https://cdn.company.com/mfe/settings/v3.1.2/remoteEntry.js',
  home:      'https://cdn.company.com/mfe/home/v2.0.5/remoteEntry.js',
}

// apps/shell/src/lib/loadRemote.ts
export function loadRemote(scope, url) {
  return new Promise((resolve) => {
    const script = document.createElement('script')
    script.src = url
    script.onload = () => {
      const container = window[scope]
      container.init(window.__webpack_share_scopes__?.default ?? {})
      resolve(container)
    }
    script.onerror = () => {
      // resolve with empty fallback - shell renders error boundary
      resolve({ get: () => () => ({ default: () => null }), init: () => {} })
    }
    document.head.appendChild(script)
  })
}
```

**4.2 Shared component API versioning example**

```tsx
// packages/ui/src/Button/Button.tsx - v2.3.0 to v2.4.0 (minor bump)

export interface ButtonProps {
  /** @deprecated Use `variant` instead. Removed in v3. */
  type?: 'primary' | 'secondary'

  /** Replaces `type`. Added in v2.4. */
  variant?: 'primary' | 'secondary' | 'ghost'

  children: React.ReactNode
}

// CHANGELOG.md - v3.0.0 breaking change
// Button: `type` prop removed. Use `variant` instead.
// Migration: npx @company/ui-codemod button-type-to-variant ./src
```

**4.3 Module Federation config snippet**

```ts
// apps/shell/webpack.config.ts
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    analytics: dynamicRemote('analytics'),
    billing:   dynamicRemote('billing'),
    settings:  dynamicRemote('settings'),
    home:      dynamicRemote('home'),
  },
  shared: {
    react:              { singleton: true, eager: true },
    'react-dom':        { singleton: true, eager: true },
    'react-router-dom': { singleton: true },
    '@company/auth':    { singleton: true },
    // @company/ui intentionally omitted - each MFE bundles its own version
  },
})

// mfes/analytics/webpack.config.ts
new ModuleFederationPlugin({
  name: 'analytics',
  filename: 'remoteEntry.js',
  exposes: { './App': './src/index.ts' },
  shared: {
    react:              { singleton: true },
    'react-dom':        { singleton: true },
    'react-router-dom': { singleton: true },
    '@company/auth':    { singleton: true },
  },
})
```

**4.4 CDN cache headers**

```yaml
# Hashed JS chunks - content-addressed, never change
Cache-Control: public, max-age=31536000, immutable

# remoteEntry.js - picked up within 60s of a new deploy
Cache-Control: public, max-age=60

# config.js - shell reads this at boot to resolve all MFE URLs
Cache-Control: public, max-age=60
```



