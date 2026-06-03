# Agent 09 — Data Structures & Serialization

Reference: [data-structures.md](../../.agents/skills/code-optimizer/references/data-structures.md)

## Scope

Replace linear lookups and JSON round-trip cloning with Map/Set and `structuredClone` where hot paths repeat work per request or per token.

**Out of scope (Agent 03):** `savings.service.ts` — not modified in this pass.

## Findings & fixes

| Location | Issue | Change |
|----------|--------|--------|
| `transfer-relay-quote.handler.ts` | `JSON.parse(JSON.stringify(relayQuote))` for quote metadata | `structuredClone(relayQuote)` — preserves types better, no stringify/parse |
| `transaction-history.handler.ts` | Double sort: `mergeTransactions` sorted, then caller sorted again; comparators allocated `Date` per comparison | `mergeTransactions` returns unsorted values; **one** `.sort()` after merging bridge events; `getTransactionSortTime()` uses numeric `Date.parse` / number / `Date` without `new Date()` in the comparator |
| `portfolio.handler.ts` | `allKnownTokens.find` per `ALWAYS_SHOW_ASSETS` entry | `tokenByAssetId` `Map` built alongside existing `tokenBySymbol` |
| `kyc.service.ts` | `verifications.find` per configured step (O(steps × verifications)) | `verificationsByStep` / `paytrieVerificationsByStep` `Map` keyed by `step` |
| `populate-tokens.ts` | `categoriesLower` as array with `.includes` in classification loops | `Set` for lowercase categories; `STOCK_CATEGORIES` membership via `.has`; substring/metal/meme checks iterate the set once |

## Patterns applied

1. **Map for keyed lookups** — asset ID → token, KYC step → verification.
2. **Set for membership** — CoinGecko category slugs (module-level category constants were already `Set`).
3. **Single sort pass** — merge deduplicates only; sort runs once at the API boundary with a shared numeric comparator helper.
4. **Clone without JSON** — `structuredClone` for relay quote metadata stored in Prisma JSON.

## Verification

```bash
npx tsc --noEmit
```

Manual smoke (optional):

- Portfolio: zero-balance major assets still appear when configured in `ALWAYS_SHOW_ASSETS`.
- Transaction history: ordering unchanged (newest first) for mixed DB / Zerion / savings / bridge activity.
- KYC status: step statuses still match DB verifications per partner.
- Relay transfer quote: metadata persisted with `requestId` and `relayQuote: true`.

## Residual / follow-up

- Other handlers still use `.find` on small in-memory lists (e.g. relay source accounts) — acceptable when list size is bounded.
- `populate-tokens.ts` `priorityChains.includes` / `bridgeAssets.includes` remain array membership on tiny static lists.
