# Agent 03: Algorithmic Complexity

Audit and remediation for `anzo-backend`. Patterns from `.agents/skills/code-optimizer/references/algorithmic-complexity.md`.

## Executive Summary

| Severity | Count | Action |
|----------|-------|--------|
| MEDIUM | 5 | Replaced repeated `.find` in loops with `Map` lookups; deduplicated hot-path sorts and API calls |
| LOW | 2 | Shared sort helper; hoisted CoinGecko fetch in populate script |

**Result:** Savings home/activity paths, relay consolidation pickers, swap quote holdings, and token population script run with fewer O(n²) passes and redundant network calls.

---

## Findings & Fixes

### 1. Savings service — goal/vault lookups in loops (MEDIUM) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| `activeGoals.find` inside tx loops | `getHomeData` | O(goals × txs) per pass | `activeGoalsById`, `goalsById` Maps |
| `vaultsToFetch.find` in activity preview | `onChainActivityPreview` | O(vaults × txs) | `vaultByKey` keyed by `network:address` |
| `goals.find` in transfer metadata | `transferBetweenGoals` | O(goals) per name resolve | `goalsById` Map |

### 2. Savings service — duplicated internal transfer application (MEDIUM) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| Two nearly identical loops over `sortedInternalTxs` | `getHomeData` (EUR before Blend, USD after) | Duplicated logic; easy to drift | Shared `applyInternalTransferToGoal(tx, currency)` — **two passes kept** so USD internals still run after Blend deposits (ordering invariant) |

### 3. Relay consolidation — sort inside `while` balance picker (MEDIUM) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| `[...working].sort(...)` every `while` iteration | `SwapsService.tryCreateRelaySwapConsolidation`, `TransferRelayQuoteHandler.resolveAggregatedTransferSource` | O(chains × iterations) re-sorts | `sortPerChainBalancesByBalanceDesc` in `consolidation.service.ts`; sort once, re-sort only when on-chain verification corrects a row |

### 4. Swap quote — double `getHoldingAssets` (MEDIUM) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| `getHoldingAssets` for dynamic from-asset, then again for Solana max buffer | `SwapsService.requestQuote` | Two Zerion-backed holdings fetches per quote | `cachedHoldings` passed into `applySolanaMaxSpendBufferFromHoldings` |

### 5. Holding assets — redundant holdings fetch by price API id (LOW) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| DB token lookup then separate `getHoldingAssets`; not-in-DB path fetched holdings again in `Promise.all` | `getHoldingAssetByPriceApiId` | Extra sequential work | Single `Promise.all([token, getHoldingAssets])`; CoinGecko detail only when token missing from DB |

### 6. `populate-tokens.ts` — repeated CoinGecko + array chain checks (LOW) — **Fixed**

| Pattern | Location | Issue | Fix |
|---------|----------|-------|-----|
| `getTopTokens(100)` per bridge asset | `processBridgeAsset` | N API round-trips in bridge loop | Hoist `getTopTokens` once in `run()`, build `cgLogoBySymbol` Map |
| `priorityChains.includes(chainId)` in LI.FI scan | `LiFiClient.findToken` | O(priority × chains) | `PRIORITY_LIFI_CHAIN_IDS` as `Set` |

---

## Shared utility

```typescript
// src/modules/swaps/services/consolidation.service.ts
export function sortPerChainBalancesByBalanceDesc(rows: PerChainBalance[]): PerChainBalance[]
```

Used by swap and transfer relay consolidation source-chain pickers.

---

## Verification

```bash
npm run build
npm test -- --testPathPattern="swaps.service|holding-assets|consolidation.service" --passWithNoTests
```

Manual smoke: savings home screen activity preview, multi-chain USDC swap quote, transfer consolidation quote, `npm run populate-tokens -- --limit 50`.

---

## Out of scope (follow-up)

- Full chronological merge of on-chain + internal + Blend events in `getHomeData` (would change balance ordering semantics).
- Request-scoped cache for `getHoldingAssets` across unrelated handlers in the same HTTP request.
