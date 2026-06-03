# Agent 07 — I/O & Network

Remediation for sequential HTTP/DB I/O and missing response compression, per [io-network.md](../../.agents/skills/code-optimizer/references/io-network.md).

## Summary

| Area | Change | Effect |
|------|--------|--------|
| `src/main.ts` | `@fastify/compress` (gzip, deflate, br) | Smaller JSON payloads on the wire |
| `persona-kyc.handler.ts` | `Promise.all` for front/back ID images | ~2× faster Persona → Bridge KYC image attach |
| `sumsub.provider.ts` | `mapWithConcurrency(..., 3)` for inspection images | Bounded parallel Sumsub downloads |
| `swaps.service.ts` | Single `token.findMany` for from/to symbols | One DB round-trip per swap quote |
| `swap-flashnet-quote.handler.ts` | Same batched token lookup | Matches swaps service |
| `direct-btc-transfer.handler.ts` | `Promise.all` UTXOs + fee recommendations | One less sequential mempool API wait |
| `dashboard.handler.ts` | Shared FX promise + parallel CoinGecko/FX on cold cache | Non-USD dashboard avoids duplicate pricing calls |

## Details

### Response compression (`main.ts`)

Registered before Helmet so compressed bodies are served for eligible responses:

```typescript
await app.register(compress, {
  encodings: ['gzip', 'deflate', 'br'],
});
```

Dependency: `@fastify/compress` (already in `package.json`).

### KYC image downloads

**Persona** — government ID front/back URLs download concurrently (max 2 in flight).

**Sumsub** — government ID resources from inspection metadata download with concurrency **3** via `src/common/utils/concurrency.util.ts` (`mapWithConcurrency`). Failed resources are logged and skipped (same behavior as before).

### Swap token lookup

Replaced paired `findUnique` calls with:

```typescript
const swapTokens = await this.prisma.token.findMany({
  where: { symbol: { in: swapSymbols, mode: 'insensitive' } },
});
```

Lookup map keyed by `symbol.toUpperCase()` for stable matching.

### Direct BTC transfer

When `slippageTolerance` is not used as a fee override, UTXO fetch and `getFeeRecommendations()` run in parallel.

### Dashboard pricing (cache cold)

For non-USD currency:

1. Start one `pricingProvider.getPrices(['usd-coin'], currency)` promise at the top of `getDashboard`.
2. Pass that promise into `getTrendingTokens` so earn conversion and trending FX share the same in-flight request.
3. On trending cache miss, CoinGecko `/simple/price` and the FX promise run via `Promise.all`.

Trending cache hits still skip CoinGecko; FX is only awaited when trending needs a live fetch.

## Verification

```bash
npm run build
```

Spot-check (manual):

- `GET /api/v1/dashboard` with `currency=eur` — response includes converted earn/trending prices.
- Swap quote `fromAsset`/`toAsset` — still resolves tokens when symbols differ only by case.
- Bridge KYC `inquiry.approved` webhook path — identifying_information images still attached when URLs exist.

## Out of scope (follow-ups)

- LI.FI quote handler still uses paired `findUnique` (same pattern as swaps; can batch in a later pass).
- Direct BTC transfer still fetches BTC USD price after PSBT build (could overlap with UTXO fetch).
- Request deduplication / retry backoff not addressed in this agent.
