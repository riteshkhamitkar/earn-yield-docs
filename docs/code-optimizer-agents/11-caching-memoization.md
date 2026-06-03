# Agent 11: Caching & Memoization

Audit reference: `.agents/skills/code-optimizer/references/caching-memoization.md`

## Summary

Hardened TTL caches across the backend: bounded in-memory LRU, shared Redis-backed quote storage, per-id CoinGecko pricing with in-flight dedupe, fail-closed auth block checks, and a shared known-token registry cache for holdings enrichment.

## Changes

### `CacheService` (`src/common/cache/cache.service.ts`)

- **Memory backend**: Enforces `MEMORY_CACHE_MAX_ENTRIES` (5000) with LRU semantics — `get` promotes a key to most-recently-used; `set` re-inserts updated keys; overflow evicts oldest entries.
- **Redis backend**: Unchanged — JSON values, per-key TTL, errors treated as cache miss.

### `QuoteCacheStore` (`src/common/cache/quote-cache.store.ts`)

- New global helper used by **consolidation** and **YO quotes**.
- Keys: `quote:consolidation:{quoteId}`, `quote:yo:{quoteId}`.
- TTL: **15 minutes** (`QUOTE_CACHE_TTL_SECONDS = 900`), aligned with prior in-process `expiresAt`.
- Serializes `expiresAt` for Redis; revives `Date` on read and deletes stale entries.

### `ConsolidationService`

- Removed unbounded in-process `Map` for consolidation quotes.
- `cacheConsolidation`, `getCachedConsolidation`, `patchCachedConsolidationIntent`, and `linkSwapQuoteId` are **async** and persist via `QuoteCacheStore` (Redis when `REDIS_URL` is set).

### `YoQuoteHandler`

- Deposit/redemption quote payloads stored in `QuoteCacheStore` instead of a local `Map`.
- `getCachedQuote` / `deleteCachedQuote` are async; execution handler awaits them.
- Short-lived **idempotency** map (30s) remains in-process by design.

### `PricingProvider`

- **Per-id, per-currency** cache (`Map<currency, Map<id, entry>>`) with 60s TTL.
- **Partial hits**: Returns cached IDs immediately; fetches only missing IDs from CoinGecko.
- **In-flight dedupe**: Concurrent requests for the same currency+id set share one upstream call (Solana token prices included).

### `AuthGuard` (`blockCache`)

- **Expired entries** removed on read (`delete` when `expiresAt <= now`).
- **DB errors**: Fail closed — if lookup fails and cache does not show `blocked: true`, responds with `403` / `ACCOUNT_STATUS_UNAVAILABLE` instead of allowing the request.
- Still honors a recent cached `blocked: true` during a blip so already-blocked users stay blocked.

### `KnownTokensCacheService` + `HoldingAssetsHandler`

- Shared service caches `prisma.token.findMany` enrichment rows (key `balances:known-tokens:enrichment`, TTL 5m, inflight dedupe).
- `HoldingAssetsHandler` uses `getLookupMaps()` instead of inline cache logic.

## Callers updated for async consolidation cache

- `swaps.service.ts`, `swap-lifi-quote.handler.ts`, `swap-flashnet-quote.handler.ts`
- `transfer-relay-quote.handler.ts`, `transfers.service.ts`
- `yo-quote.handler.ts`, `yo-execution.handler.ts`

## Operations

- Set `REDIS_URL` in production so quote and known-token caches are shared across pods.
- Without Redis, `CacheService` uses bounded in-memory LRU (per process).

## Verification

```bash
npm run build
npm test -- --testPathPattern="consolidation.service|yo-quote"
```
