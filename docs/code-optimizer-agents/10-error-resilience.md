# Error & Resilience (Agent 10)

Audit reference: `.agents/skills/code-optimizer/references/error-resilience.md`

This document records fixes applied to harden external I/O, polling loops, health checks, and misleading empty fallbacks.

## Summary

| Area | Change |
|------|--------|
| Zerion positions | Timed axios client (30s); stale position cache on API failure; `ServiceUnavailableException` when no cache |
| CoinGecko pricing | Timed axios client (30s); stale price cache on failure; `ServiceUnavailableException` instead of empty `Map` |
| Solana direct transfer | `AbortSignal.timeout(30s)` on all Turnkey `fetch` calls |
| BTC confirmation poll | `AbortSignal.timeout(15s)` per mempool.space request |
| Prisma bootstrap | Throw on `$connect` failure when `NODE_ENV=production` |
| Health check | `queryRaw` raced with 5s timeout (matches paymaster RPC timeout) |
| Relay status poll | Stop after 5 consecutive errors → `FAILED` |
| Flashnet status poll | Stop after 5 consecutive errors (leave swap `EXECUTING`) |
| Blend withdraw poll | Log errors on failed DB status updates (no empty `catch`) |
| Paytrie webhook | Documented fire-and-forget 200-first pattern (aligned with Flashnet) |

## Zerion positions (`zerion-positions.provider.ts`)

- `axios.create({ timeout: 30_000 })` for all Zerion wallet position requests.
- Successful responses update an in-memory `positionsCache` keyed by `address:evm|sol`.
- On transient API errors: return stale cache if present.
- On 401 (invalid key): still return `[]` (provider effectively disabled).
- With no cache: throw `ServiceUnavailableException` so holdings/portfolio do not show a false empty wallet.

## Pricing (`pricing.provider.ts`)

- Shared `axios` instance with 30s timeout for CoinGecko Pro API.
- `getPrices`: on failure, `getStalePrices()` returns any cached IDs for that currency (TTL ignored); otherwise `ServiceUnavailableException`.
- `getSolanaTokenPrices`: throws `ServiceUnavailableException` on failure (no partial-address cache today).

Callers that previously received `{}` on outage should handle 503 or catch at the handler layer (e.g. `holding-assets` exchange-rate block already catches pricing errors).

## Solana direct transfer (`solana-direct-transfer.service.ts`)

Turnkey HTTP calls use `AbortSignal.timeout(30_000)`:

- `POST .../submit/sol_send_transaction` (paymaster + sponsored flows)
- `POST .../query/get_send_transaction_status` (poll loop)

Prevents hung background polls when Turnkey is slow or unreachable.

## Transfer execution — BTC poll (`transfer-execution.handler.ts`)

`pollBtcConfirmation` mempool fetches use `AbortSignal.timeout(15_000)` so each 30s poll tick cannot block indefinitely.

## Database (`prisma.service.ts`)

- **Production**: failed `$connect` on module init propagates → process fails fast; health/orchestrator can mark instance unhealthy.
- **Non-production**: logs error and continues (local dev without DB still possible).

## Health (`health.handler.ts`)

`checkDatabase()` wraps `$queryRaw\`SELECT 1\`` in `Promise.race` with a 5s reject. Slow or wedged DB connections surface as `disconnected` instead of blocking the health endpoint.

## Relay status polling (`relay-status.service.ts`)

`pollUntilResolved` tracks consecutive poll exceptions. After **5** failures in a row, returns `{ status: 'FAILED', failReason: 'Stopped polling after 5 consecutive errors: ...' }` instead of spinning until the 5-minute wall timeout.

## Flashnet status polling (`swap-execution.handler.ts`)

`pollFlashnetStatusInBackground` uses the same **5** consecutive error cap. On cap: stop polling, leave transaction `EXECUTING` (webhooks + client status endpoint remain source of truth; funds may still be in-flight).

## Blend withdraw (`blend-withdraw.handler.ts`)

- Terminal status DB `update` failures: `logger.error` with message + stack (was warn-only / silent).
- Poll timeout `FAILED` update: logs error instead of empty `catch`.

## Paytrie webhook (`paytrie-webhook.controller.ts`)

**Pattern: respond 200 first, process async.**

Paytrie (like Flashnet/MoonPay webhooks) retries on non-2xx responses. The controller returns `{ received: true }` immediately and runs `processWebhook` in the background with `.catch` logging. This avoids duplicate processing from provider retries while keeping the HTTP handler fast.

If synchronous processing is required later, switch to `await` inside try/catch and still return 200 only after success — but expect longer webhook latency and retry storms on timeouts.

## Verification

```bash
npm run build
# optional targeted tests
npm test -- --testPathPattern="blend-withdraw|transfer-execution"
```

## Related patterns (not changed in this pass)

- Retry with exponential backoff: existing in Flashnet submit, LI.FI flows.
- Circuit breakers: not introduced; consecutive-error caps on polls are the lightweight guard.
- Other axios providers already using `timeout: 30000` (Relay, Bridge, LI.FI, etc.) — Zerion/Pricing brought in line.
