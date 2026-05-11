# README

### Assumptions made

- All four teams use React + TypeScript as per constraints. No cross-framework support required.
- The dashboard is a SPA. SEO is not a priority so client-side composition is preferred over SSR.
- A private npm registry (e.g. GitHub Packages) is available for shared package publishing.
- Teams operate independently but share a monorepo. Git history, tooling, and CI pipelines are centralised.
- Auth is handled via an OAuth2/OIDC provider (e.g. Auth0, Cognito) with JWT access tokens.
- CDN (e.g. CloudFront or Cloudflare) is available with edge caching support.
- The monorepo toolchain is Nx for affected builds and task caching.

[View Assigment Answers](./AssignmentAnswers.md)

---

### Time spent

3 hours 30 minutes.

---

### Trade-offs debated

**1. Module Federation (chosen) vs server-side composition**

Module Federation gives seamless UX with shared singletons (one React, one auth context). The trade-off is that all MFEs must align on React version. A version mismatch crashes hooks at runtime. Server-side composition is more resilient and framework-agnostic but adds latency from stitching and makes local development significantly harder. Given all teams are on React and the product is a dashboard rather than a public-facing SEO site, Module Federation wins.

**2. Monorepo (chosen) vs polyrepo**

Monorepo with path-scoped CI pipelines gives teams full deployment autonomy while making shared package governance (design system, auth SDK) trivially enforceable via CODEOWNERS. The trade-off is that monorepos require more tooling discipline upfront with Nx, Changesets, and CI path filters. Given 4 teams sharing a design system, the governance benefits outweigh the setup cost.

---

### Given more time I would

- Define a formal MFE manifest schema (JSON) so the shell can dynamically discover and load remotes without hardcoded config
- Add E2E contract tests using Pact (consumer-driven contract testing) so each MFE can verify its integration with the shell in isolation
- Document the observability schema (trace IDs, span naming conventions) so cross-MFE request traces are joinable in the APM tool
- Explore Vite-based Module Federation as an alternative to Webpack. Vite's faster dev server and build times would significantly improve local development experience and CI build times at scale
- Investigate Next.js micro-frontend architecture and how React Server Components could be leveraged. Server components open up a compelling server-side composition model where each MFE renders its own RSC tree, potentially replacing the BFF aggregation layer for data-heavy pages
- Design a proper CI/CD pipeline per environment (dev, staging, production) with environment-specific `config.js` generation, automated smoke tests per environment, and promotion gates between environments
- Improve local development experience by making it trivial for a developer to point the shell at any combination of local and deployed MFEs. A simple CLI or `.env.local` convention so Team A can run only their MFE locally while consuming staging builds of all other remotes
- Evaluate monorepo tooling more thoroughly. Nx vs Turborepo vs Rspack each have different trade-offs around build speed, caching strategy, and plugin ecosystem. Rspack in particular is worth investigating as a Webpack-compatible drop-in that offers significantly faster build times, which matters at scale with four teams and frequent deploys

---

### Next steps

Each of the above areas will be validated through a dedicated proof of concept before any production decision is made. The intention is to test Vite-based Module Federation, Next.js RSC composition, per-environment CI/CD pipelines, local dev tooling, and monorepo toolchain options (Nx, Turborepo, Rspack) independently so trade-offs are grounded in real findings rather than assumption.
