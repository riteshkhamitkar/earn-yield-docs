# Memory & Resources — Agent 02 Report

Audit reference: `.agents/skills/code-optimizer/references/memory-resources.md`

## Summary

Bounded in-process `Map` caches with TTL pruning or FIFO caps, replaced the transfer validate-address CoinGecko full-list load with per-`priceApiId` lazy fetches, aligned wallet Prisma reads with Agent 01 (`take: 1`), and used `structuredClone` / parallel Sumsub downloads where safe. No new heavy modules—shared helpers live in `src/common/utils/map-cache.util.ts`.

## Shared utility

### `src/common/utils/map-cache.util.ts`

| Helper | Purpose |
|--------|---------|
| `pruneExpiredEntries` | Drop `expiresAt`-stale entries |
| `pruneCachedAtEntries` | Drop `cachedAt` + TTL stale entries |
| `evictMapFifo` | Cap map size by insertion order |
| `boundExpiresAtMap` | Expired prune + FIFO cap |
| `pruneTimestampMap` | Cooldown maps (timestamp values) |

## Changes by file

### `src/common/cache/cache.service.ts`

| | Before | After |
|---|--------|-------|
| `MemoryBackend` | Unbounded growth possible | Max 5000 entries, LRU touch on hit, expired purge on `set` |

### `src/modules/yield/services/yo-api.service.ts`

| | Before | After |
|---|--------|-------|
| `historyCache` | Unbounded | `boundExpiresAtMap`, max 500 entries, 2 min TTL |

### `src/modules/yield/services/yield-positions.service.ts`

| | Before | After |
|---|--------|-------|
| `vaultCache` | Unbounded | `boundExpiresAtMap`, max 64 entries, 5 min TTL |

### `src/modules/balances/handlers/holding-assets.handler.ts`

| | Before | After |
|---|--------|-------|
| `marketCache` | Unbounded CoinGecko markets cache | TTL prune + FIFO cap 2000 entries |

### `src/common/guards/auth.guard.ts`

| | Before | After |
|---|--------|-------|
| `blockCache` | Grew with distinct userIds | `boundExpiresAtMap`, max 10_000, 5 s TTL |

### `src/modules/blend/handlers/blend-withdraw.handler.ts`

| | Before | After |
|---|--------|-------|
| `pendingGoalIds` | Unbounded intent map | `boundExpiresAtMap`, max 5000, 20 min TTL; prune on prepare/execute |

### `src/modules/kyc/kyc.service.ts`

| | Before | After |
|---|--------|-------|
| `lastPaytrieStatusReconcileAttempt` | Unbounded cooldown map | `maybePrunePaytrieStatusReconcileAttempts` (same pattern as status auto-submit): 10 min TTL, max 10_000 |

### `src/modules/transfers/transfers.service.ts`

| | Before | After |
|---|--------|-------|
| CoinGecko platforms | Single `/coins/list` (~15k rows) cached 1 h | Per-id `/coins/{id}` lazy cache, 30 min TTL, max 2000 entries; holdings fetched first, then platforms for `priceApiId` set only |
| `validateAddress` | Parallel full-list + holdings | Holdings → bounded platform map + yield CAIP set |

### `src/modules/transfers/handlers/transfer-relay-quote.handler.ts`

| | Before | After |
|---|--------|-------|
| Quote metadata clone | `JSON.parse(JSON.stringify(relayQuote))` | `structuredClone(relayQuote)` |

### `src/modules/savings/savings.service.ts`

| | Before | After |
|---|--------|-------|
| Wallet load (`getGoal`, `getHomeData`) | `include: { Wallet: true }` | `select: { Wallet: { select: { ethereumAddress: true }, take: 1 } }` (Agent 01 alignment) |

### `src/modules/swaps/swaps.service.ts`

| | Before | After |
|---|--------|-------|
| `requestQuote` wallet | `include: { Wallet: true }` | `take: 1` with `ethereumAddress`, `solanaAddress`, `bitcoinAddress` only |

### `src/modules/kyc/providers/sumsub.provider.ts`

| | Before | After |
|---|--------|-------|
| Document image download | Sequential `for` loop | `mapWithConcurrency(..., 3)` parallel downloads (bounded concurrency) |

## Out of scope / already present

- `transaction-history.handler.ts` — existing `MAX_CACHE_SIZE` FIFO (unchanged)
- `kyc.service.ts` `lastStatusAutoSubmitAttempt` — existing prune (unchanged)
- Redis-backed `CacheService` — no in-process map growth when `REDIS_URL` is set

## Verification

```bash
npm test -- --testPathPattern="transfers.service|swaps.service|map-cache"
```
