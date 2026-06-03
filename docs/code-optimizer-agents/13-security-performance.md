# Agent 13 — Security-Performance

Remediation pass for cryptographic misuse, missing rate limits, ReDoS-prone patterns, and timing-unsafe comparisons.

## Findings & fixes

| Severity | Area | Issue | Fix |
|----------|------|-------|-----|
| Critical | `bridge-webhook.guard.ts` | Webhooks accepted when `BRIDGE_WEBHOOK_PUBLIC_KEY` unset | Fail closed; allow bypass only when `NODE_ENV !== 'production'` **and** `BRIDGE_WEBHOOK_SKIP_VERIFY=true` |
| Critical | `kyc.controller` `GET tos/callback` | Open callback forged `userId` could approve TOS for any account | `TosCallbackGuard`: valid Bearer JWT **or** HMAC-signed query params (`expires` + `sig`) from `getTosLink` |
| High | `auth.guard.ts` | JWT verify without algorithm allowlist (algorithm confusion) | `verify(token, secret, { algorithms: ['HS256'] })` |
| High | `internal-api.guard.ts` | Plain `!==` key compare (timing leak) | `timingSafeEqual` with equal-length buffer guard |
| Medium | `swaps.controller` / `transfers.controller` | Quote/execute endpoints unthrottled (expensive upstream calls) | `@Throttle`: quote/send 10/min, execute 5/min |
| Medium | `payment-links.controller` | Fund-moving routes partially throttled | `@Throttle` 5/min on `reclaim`, `claim`, `cancel` (withdraw already limited) |
| Medium | `tokens.controller` `GET list` | Public list at global 100/min default | `@Throttle` 30/min per IP |
| Medium | `swaps.service` | ReDoS-prone regex on provider error strings | `messageIndicatesAmountTooSmall()` using fixed substring checks |
| Medium | `sumsub.provider` | `timingSafeEqual` without length guard on digest | Hex format check + length match before compare |
| Low | `username.util` | `Math.random()` for word selection | `crypto.randomInt(WORDS.length)` |

## New / changed files

### KYC TOS callback signing

- `src/modules/kyc/utils/tos-callback-sign.util.ts` — HMAC-SHA256 over `userId|partnerId|expires` (24h TTL), signed with `JWT_SECRET`
- `src/modules/kyc/guards/tos-callback.guard.ts` — dual auth path for Bridge WebView redirect
- `kyc.service.getTosLink` — appends `expires` and `sig` to `redirect_uri`

**Breaking note:** TOS links issued before this change lack `sig`/`expires`; users must re-fetch `GET /kyc/tos-link/:partnerId` before accepting TOS.

### Rate limit reference

| Endpoint | Limit | Window |
|----------|-------|----------|
| `POST /swaps/quote` | 10 | 60s |
| `POST /swaps/execute`, `POST /swaps/lifi/execute` | 5 | 60s |
| `POST /transfers/send` | 10 | 60s |
| `POST /transfers/execute` | 5 | 60s |
| `POST /payment-links/withdraw|reclaim|claim|cancel` | 5 | 60s |
| `GET /tokens/list` | 30 | 60s |

Global default remains 100/min via `ThrottlerModule` + `CustomThrottlerGuard`.

## Verification

```bash
npm run build
npm test -- --testPathPattern="swaps.service|kyc"
```

Manual checks:

1. Bridge webhook without public key → 401 (unless dev skip env set).
2. `GET /kyc/tos/callback` without JWT or valid `sig` → 401.
3. `GET /kyc/tos-link/bridge` (authed) → redirect URI includes `expires` and `sig`.
4. Internal API wrong key → 401; correct key still works.

## Deferred / out of scope

- Bcrypt/argon2 audit (no password hashing hot paths in this pass).
- SQL injection vectors (Prisma parameterized queries used throughout).
- Per-user throttling on authenticated routes (IP-based `@Throttle` only; consider userId tracker later).
