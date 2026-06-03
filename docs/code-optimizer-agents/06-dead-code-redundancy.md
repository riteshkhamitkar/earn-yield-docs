# Agent 06: Dead Code & Redundancy

Audit and cleanup report for `anzo-backend`. Patterns from `.agents/skills/code-optimizer/references/dead-code-redundancy.md`.

## Executive Summary

| Severity | Count | Action |
|----------|-------|--------|
| MEDIUM | 5 | Removed unused files, exports, and broken npm scripts |
| LOW | 2 | Removed unused barrel re-exports |

**Result:** 7 files deleted, 3 source files trimmed, 2 broken `package.json` scripts removed, format glob corrected.

---

## Findings & Fixes

### 1. Unused swap provider scaffold (MEDIUM) — **Deleted**

| Item | Evidence | Action |
|------|----------|--------|
| `src/modules/swaps/interfaces/swap-provider.interface.ts` | `ISwapProvider`, `SwapQuote`, `SwapQuoteRequest` only referenced in deleted README; swaps use handler-based architecture (`SwapQuoteDto`, relay/LiFi/Flashnet handlers) | Deleted |
| `src/modules/swaps/providers/README.md` | Sole consumer of the interface; `providers/` had no implementations | Deleted |

The swaps module never adopted the provider-strategy pattern this scaffold described.

---

### 2. Orphan CLI script (MEDIUM) — **Deleted**

| Item | Evidence | Action |
|------|----------|--------|
| `scripts/list-onebalance-assets.ts` | Not referenced in `package.json`, no imports anywhere in repo; noted for deletion in `ob-relay-docs/plan.md` | Deleted |

OneBalance asset listing is handled via runtime APIs and token registry scripts (`populate-tokens`, `sync-deposit-token-networks`).

---

### 3. Unused exports in `account.util.ts` (MEDIUM) — **Removed**

| Export | Used by | Action |
|--------|---------|--------|
| `determineSourceAccounts` | swaps, transfers, blend, yield handlers | **Kept** |
| `determineDestinationAccount` | blend-deposit, yo-quote handlers | **Kept** |
| `determineSourceAccount` | Nothing | Removed |
| `getAddressFromAccount` | Nothing (transfer handler has its own private method) | Removed |
| `extractChainFromAccount` | Nothing | Removed |

---

### 4. Unused export in `evm-tx-hash.util.ts` (LOW) — **Removed**

| Export | Used by | Action |
|--------|---------|--------|
| `normalizeEvmTxHash` | bridge payout, yield DTOs/handlers | **Kept** |
| `isValidEvmTxHash` | Nothing | Removed |

---

### 5. Unused bridge barrel files (LOW) — **Deleted**

| File | Evidence | Action |
|------|----------|--------|
| `src/modules/bridge/index.ts` | No imports from `modules/bridge` barrel | Deleted |
| `src/modules/bridge/utils/index.ts` | All consumers import `../utils/<file>.util` directly | Deleted |
| `src/modules/bridge/webhooks/index.ts` | Module imports controllers/types by direct path | Deleted |

**Kept** (actively used):

- `src/modules/bridge/dto/index.ts` — imported by `bridge.controller.ts`, `bridge.service.ts`
- `src/modules/bridge/handlers/index.ts` — imported by `bridge.module.ts`, `bridge.service.ts`

---

### 6. Broken `package.json` scripts (MEDIUM) — **Fixed**

| Script | Problem | Action |
|--------|---------|--------|
| `test:e2e` | Referenced `./test/jest-e2e.json`; no `test/` directory exists | Removed |
| `test:endpoints` | Referenced `test-all-endpoints.js`; file does not exist | Removed |
| `format` | Included `"test/**/*.ts"` glob; no matching files (specs live under `src/**/*.spec.ts`) | Updated to `src/**/*.ts`, `scripts/**/*.ts`, `prisma/**/*.ts` |

Unit tests remain via `npm test` (Jest, `rootDir: src`, `*.spec.ts`).

---

## Verification

After cleanup:

```bash
npm run build
npm test
```

Both should pass with no import errors from removed symbols or barrels.

---

## Remaining Opportunities (not in scope)

These were flagged by pattern scans but left unchanged — follow-up for future passes:

- **`package.json` lint glob** still includes `test/` and `apps/`, `libs/` paths that may not exist; align with actual layout if lint noise appears.
- **`ob-relay-docs/plan.md`** still references deleted `list-onebalance-assets.ts` (plan doc only).
- **Commented-out code blocks** — run a dedicated pass with `//\s*(function|const|import)` patterns if desired.
