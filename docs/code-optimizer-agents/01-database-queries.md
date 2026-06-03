# Database & Queries — Agent 01 Report

Audit reference: `.agents/skills/code-optimizer/references/database-queries.md`

## Summary

Applied bounded queries, batch lookups, caching, and one composite index across the handlers called out in the audit. API response shapes are unchanged; limits only cap how much historical data is loaded server-side.

## Changes by file

### `src/modules/balances/handlers/holding-assets.handler.ts`

| | Before | After |
|---|--------|-------|
| Token enrichment | `token.findMany()` on every `getHoldingAssets` call | Same query cached 300s via global `CacheService` (`balances:known-tokens:enrichment`) |

### `src/modules/savings/savings.service.ts`

| | Before | After |
|---|--------|-------|
| Savings ledger `transaction.findMany` | Unbounded | `take: 5000` (`SAVINGS_TX_QUERY_LIMIT`) + `orderBy: { createdAt: 'asc' }` on blend/local/internal/recent-goal queries |
| Map / balance logic | — | Unchanged (agent 3 scope) |

### `src/modules/swaps/swaps.service.ts`

| | Before | After |
|---|--------|-------|
| `requestQuote` token lookup | Two `findUnique` by symbol | One `findMany` with `symbol: { in, mode: 'insensitive' }` |
| `getSwapHistory` | Unbounded `swap.findMany` | `take: 100`, `orderBy: { createdAt: 'desc' }` |

### `prisma/schema.prisma` + migration

| | Before | After |
|---|--------|-------|
| `Swap` indexes | `userId`, `status`, `quoteId` | Added `@@index([userId, createdAt(sort: Desc)])` |
| Migration | — | `prisma/migrations/20260603120000_swap_user_created_at_index/migration.sql` |

### `src/modules/user/handlers/username.handler.ts`

| | Before | After |
|---|--------|-------|
| Suggestions | N × `user.findUnique` in a loop | One `findMany` with `username: { in: candidates }` |

### `src/modules/payment-links/providers/token-decimals.provider.ts`

| | Before | After |
|---|--------|-------|
| DB lookup | `findFirst({ assetId: { contains: key } })` | `findMany` on exact CAIP-19 `eip155:{chain}/erc20:{address}` + `lifiAssetId` match |
| Batch API | — | `getDecimalsBatch(addresses)` for multi-token paths |

### `src/modules/payment-links/handlers/manage-link.handler.ts`

| | Before | After |
|---|--------|-------|
| `getUserLinks` decimals | N parallel `getDecimals` | `getDecimalsBatch(uniqueTokens)` |

### `src/modules/balances/handlers/transaction-history.handler.ts`

| | Before | After |
|---|--------|-------|
| Contract → token | `assetId: { contains: addr }` per address | `assetId: { endsWith: ':{addr}' }` (suffix match on CAIP-19) |
| `assetId: { in: assetIds }` | Exact | Unchanged |

### `src/modules/flashnet/lightning-pay-links/flashnet-lightning-pay-links.handler.ts`

| | Before | After |
|---|--------|-------|
| `listPayLinks` | Unbounded `findMany` | `take: 100` |

### `src/modules/auth/handlers/session-sync.handler.ts`

| | Before | After |
|---|--------|-------|
| `generateUniqueUsername` | Up to 5 sequential `findUnique` | One `findMany` over candidate set |

### `src/modules/bridge/handlers/bridge-kyc.handler.ts`

| | Before | After |
|---|--------|-------|
| `handleKycApproval` VA reads | Up to 3 identical `bridgeVirtualAccount.findMany` | One initial fetch; refresh once after creates |

### `src/modules/database/prisma.service.ts`

| | Before | After |
|---|--------|-------|
| Pool sizing | Undocumented | Class-level comment: set `connection_limit` on `DATABASE_URL` per pod |

## Remaining gaps

1. **Savings history cap** — Users with more than 5,000 savings-related rows may see balance/activity drift until pagination or incremental ledger aggregation is added.
2. **`transaction-history` contract OR clauses** — Still one `endsWith` OR per distinct contract address (better than `contains`, but large Zerion feeds could use a precomputed `lifiAssetId` map).
3. **`holding-assets` token cache** — Full curated token list is cached; very large token tables would benefit from selective columns or a materialized enrichment view.
4. **`Swap` deprecation** — Table is marked deprecated in schema; long-term history should move fully to `Transaction` with the same index pattern.
5. **Migration deploy** — Run `npx prisma migrate deploy` in each environment so the new `Swap` index exists before relying on it in production.

## Verification

```bash
npm test -- --passWithNoTests src/modules/swaps/swaps.service.spec.ts
```

Add other touched `*.spec.ts` paths as they are introduced.
