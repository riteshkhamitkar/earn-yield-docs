# Build & Compilation — Agent 12 Report

**Domain:** Build & Compilation  
**Reference:** `.agents/skills/code-optimizer/references/build-compilation.md`  
**Stack:** NestJS 11, TypeScript, Prisma, Node 20, Docker multi-stage, PM2

## Executive Summary

- **2** critical, **4** high, **2** medium issues found and fixed
- Top fixes:
  1. **Prisma client missing from CI build** — `build` now runs `prisma generate` before `nest build`
  2. **Postinstall patches silently failing** — patch scripts now `exit(1)` on error so `npm ci` fails fast
  3. **Production runtime mismatch** — Dockerfile and PM2 now set `NODE_OPTIONS=--conditions=import` to match `start:prod`

## Findings & Fixes

### `package.json`

| # | Severity | Pattern | Fix |
|---|----------|---------|-----|
| 1 | CRITICAL | `build` only ran `nest build`; Prisma client not guaranteed before compile | `"build": "prisma generate && nest build"` |
| 2 | HIGH | No `engines.node`; local/CI could use unsupported Node versions | Added `"engines": { "node": "20.x" }` |
| 3 | MEDIUM | `test:e2e` pointed at missing `test/jest-e2e.json` | Added minimal `test/jest-e2e.json` with `passWithNoTests: true` |

### `scripts/patch-blend-esm.js` & `scripts/patch-fastify-helmet.js`

| # | Severity | Pattern | Fix |
|---|----------|---------|-----|
| 1 | CRITICAL | `catch` block called `process.exit(0)` — patch failures did not fail install | Changed to `process.exit(1)` |

These patches are required for `@blend-money/*` (ESM exports) and `@fastify/helmet` under `--conditions=import`. A silent success on failure would produce a broken runtime.

### `Dockerfile`

| # | Severity | Pattern | Fix |
|---|----------|---------|-----|
| 1 | HIGH | Production stage used deprecated `--only=production` | `npm ci --omit=dev` |
| 2 | HIGH | `CMD` ran plain `node` without `NODE_OPTIONS`; differed from `start:prod` | `ENV NODE_OPTIONS=--conditions=import` + comment |
| 3 | MEDIUM | Production `npm ci` ran `postinstall` but patch scripts were not copied | Copy `scripts/patch-*.js` before `npm ci` |

Build stage still uses full `npm ci` (devDependencies needed for `nest build`). Production stage omits dev deps and applies patches via `postinstall`.

### `ecosystem.config.js`

| # | Severity | Pattern | Fix |
|---|----------|---------|-----|
| 1 | HIGH | `wait_ready: true` but `main.ts` never calls `process.send('ready')` | Set `wait_ready: false` |
| 2 | HIGH | PM2 env lacked `NODE_OPTIONS` used by `start:prod` | Added `NODE_OPTIONS: '--conditions=import'` with comment |

Alternative for PM2 cluster mode: keep `wait_ready: true` and emit ready from `main.ts` after `app.listen()`:

```typescript
if (typeof process.send === 'function') {
  process.send('ready');
}
```

### `tsconfig.build.json`

| # | Severity | Pattern | Fix |
|---|----------|---------|-----|
| 1 | MEDIUM | `src/modules/payment-links/scripts/integration-test.ts` compiled into `dist/` | Exclude `**/integration-test.ts` and `src/**/scripts/**` |

Dev/ops scripts under `src/` should not ship in the production bundle.

## Verification

```bash
npm run build          # prisma generate + nest build
npm run test:e2e       # passes with no e2e specs (passWithNoTests)
node -e "require('./package.json').engines"  # node: 20.x
```

Docker: production stage installs with `--omit=dev`, applies postinstall patches, and runs with `NODE_OPTIONS=--conditions=import`.

## Remaining Recommendations (not applied)

| Severity | Item | Notes |
|----------|------|-------|
| LOW | Pin `node:20-alpine` digest in Dockerfile | Reproducible builds |
| LOW | Add `.dockerignore` if missing | Smaller build context |
| LOW | Add first `.e2e-spec.ts` for critical routes | `rules.md` encourages E2E for API endpoints |
