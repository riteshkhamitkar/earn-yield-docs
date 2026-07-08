# Stripe Embedded Components Onramp — Design Spec

**Status:** Approved (pending user review)
**Date:** 2026-07-08
**Owner:** Backend team
**Depends on:** Stripe private-preview onboarding (feature gates for Embedded Components onramp APIs + Link OAuth APIs, `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` provisioning, app registered as trusted application).

**Reference docs:**
- Integration guide: https://docs.stripe.com/crypto/onramp/embedded-components-integration-guide
- CryptoOnrampSession object: https://docs.stripe.com/api/crypto/onramp_sessions/object
- Checkout API: https://docs.stripe.com/api/crypto/onramp_sessions/checkout
- Quickstart: https://docs.stripe.com/crypto/onramp/embedded-components-quickstart

---

## 1. Overview

### Goal

Add Stripe's Embedded Components crypto onramp as a fourth fiat-to-crypto provider alongside Coinbase, MoonPay, and Swapped. The frontend (React Native) uses Stripe's `@stripe/stripe-react-native` SDK with the Onramp pod; the backend (NestJS) handles the OAuth token custody, customer lookups, session creation, checkout proxy, and webhook reconciliation.

### Scope (v1)

**In scope:**
- USDC purchases across four networks: Solana, Ethereum, Base, Polygon
- Server-side wallet derivation (reuse existing `MoonPayBuyService.getWalletAddressForUser`)
- Per-user Stripe Link OAuth token storage with AES-256-GCM encryption
- Headless `CryptoOnrampSession` creation + checkout proxy
- Synchronous webhook for `crypto.onramp_session.updated` events
- Stripe plugged into the existing `OnrampAggregatorService` as a fourth quote provider
- Account-deletion hook that nulls OAuth tokens

**Out of scope (v1):**
- EU onboarding (`source_currency: 'eur'`, EU KYC fields)
- BTC / ETH / SOL native purchases (USDC-only for v1)
- Wallet listing / payment-token listing server endpoints (SDK handles these)
- `STRIPE_OAUTH_ENCRYPTION_KEY` rotation tooling (documented limitation)
- Webhook event durability table (synchronous processing + Stripe's 3-day retry is sufficient)
- Sumsub → Stripe KYC bridging (Stripe runs its own KYC)

### Non-goals

- Replacing Coinbase / MoonPay / Swapped. Stripe is additive.
- Storing raw OAuth tokens in plaintext anywhere (logs, DB, error messages).
- Custodying the user's Stripe Link password or any card details. PCI scope stays with Stripe.

### Existing patterns this mirrors

| Concern | Reference pattern |
|---|---|
| External customer table | `BridgeCustomer` (schema.prisma:375), `SeismicCustomer` (schema.prisma:941) |
| HMAC webhook + Transaction writes + P2002 idempotency | `moonpay-onramps-webhook.controller.ts` |
| Provider / Service split with axios client | `bridge.provider.ts:67` + `bridge.service.ts:40` |
| Server-side wallet derivation, never trusting client | `moonpay-buy.service.ts:655` (`getWalletAddressForUser`) |
| Multi-provider quote fan-out | `onramp-aggregator.service.ts:138` (`getQuotes`) |
| Centralized error mapping | `bridge-error.util.ts:16` (`handleBridgeError`) |
| Joi env validation with `.when()` for feature flags | `src/config/validation.schema.ts:181` (Seismic) |
| Soft-delete account flagging | `account.handler.ts:126` |
| `Transaction` row writes for deposits | `TransactionType.DEPOSIT`, `service: 'moonpay_onramp'` → `'stripe_onramp'` |

---

## 2. Architecture decisions (locked)

| Decision | Choice | Rationale |
|---|---|---|
| KYC strategy | Stripe self-KYC, mirror status only | Stripe runs its own KYC inside the SDK; matches `SeismicCustomer` pattern. Avoids touching `KycProfile` / `KycVerification`. |
| Token storage | AES-256-GCM with scrypt-derived env key, JSON envelope in single column | First encryption util in codebase; reusable. GCM tag gives tamper detection. |
| Currency scope | USDC only, multi-network (Solana/Base/Ethereum/Polygon) | Matches existing aggregator assumption (USDC-on-Base default). Smaller quote surface. |
| Webhook strategy | Synchronous, 503-on-failure (MoonPay pattern) | Stripe retries for ~3 days; no need for `*WebhookEvent` durability table. |
| Aggregator placement | Stripe as 4th quote provider, returning users only | Consistent UX. Stripe can't quote without an authenticated Link session. |
| Account deletion | Null out cipher fields on delete | Cheap, safe, matches soft-delete model. |
| Vault architecture | Approach A: single `StripeOnrampCustomer` table, vault inline | Mirrors `BridgeCustomer`. Simplest queries, single upsert. |
| Module structure | Provider / TokenVault / Service split + dedicated webhook controller | Matches Bridge. Clean HTTP-vs-business separation, easily testable. |
| Wallet derivation | Reuse `MoonPayBuyService.getWalletAddressForUser` | Single source of truth across all onramp providers. |

---

## 3. Prisma schema

One new model. Reuses the existing `Transaction` model with `service: 'stripe_onramp'` (no new transaction table).

```prisma
// ============================================
// Stripe Embedded Components Onramp
// (Fiat-to-crypto via Stripe Link OAuth + headless CryptoOnrampSession)
// ============================================

/// Maps an Anzo user to their Stripe Crypto Customer and per-user OAuth state.
/// crypto_customer_id comes from the SDK's `authorize` callback; access/refresh
/// tokens come from POST /v1/link_auth_intent/{id}/tokens and are stored
/// AES-256-GCM encrypted (see src/common/utils/crypto.util.ts).
model StripeOnrampCustomer {
  id     String @id @default(cuid())
  userId String @unique
  user   User   @relation(fields: [userId], references: [id], onDelete: Cascade)

  /// Stripe Crypto Customer ID (cus_xxx). Returned in the SDK authorize callback.
  /// Required on every /v1/crypto/customers/{id}/* call alongside the OAuth token.
  cryptoCustomerId String @unique

  /// The email used for the LinkAuthIntent lookup. Stripe keys the Link account
  /// off this; pinning it lets us detect email changes and re-auth cleanly.
  linkEmail String

  // ── OAuth token vault (AES-256-GCM encrypted at the app layer) ──
  /// JSON envelope: { ciphertext, iv, tag } (all base64). Encrypts the access_token.
  /// Null when the user has never authenticated or has been soft-deleted.
  accessTokenCipher     String?
  accessTokenExpiresAt  DateTime? // 1h TTL from Stripe
  /// JSON envelope: { ciphertext, iv, tag } (all base64). Encrypts the refresh_token.
  refreshTokenCipher    String?
  refreshTokenExpiresAt DateTime? // 90d TTL from Stripe
  /// Reused for seamless sign-in next session (authorize with stored intent).
  lastAuthIntentId      String?

  // ── KYC mirror (Stripe is source of truth; this is a cache) ──
  /// not_started | submitted | verified | rejected
  kycStatus    String @default("not_started")
  kycCheckedAt DateTime?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([cryptoCustomerId])
  @@index([kycStatus])
}
```

**Field rationale:**
- `cryptoCustomerId @unique` — same shape as `BridgeCustomer.customerId` / `SeismicCustomer.seismicCustomerId`.
- `linkEmail` — pinned because the user could change their Anzo email; Stripe keys off the original.
- `accessTokenCipher` / `refreshTokenCipher` — single string column each holding serialized `{ ciphertext, iv, tag }`. Simpler than 3 columns per token.
- `kycStatus` + `kycCheckedAt` — mirrors `SeismicCustomer.kycStatus`. No `KycVerification` writes (Pattern B / Stripe self-KYC).
- `lastAuthIntentId` — enables Stripe's seamless sign-in flow for returning users.

**Transaction rows** reuse the existing `Transaction` model with:
- `id` and `quoteId` = `stripe_onramp_${sessionId}` (deterministic, idempotency key)
- `type = TransactionType.DEPOSIT`
- `service = 'stripe_onramp'`
- `metadata` = `{ ...sessionObject, provider: 'Stripe Embedded Components', stripeSessionId }`

**No changes** to `User`, `Transaction`, or any other existing model. The relation `StripeOnrampCustomer.user` is one-way and the cascade is on the Stripe side only.

---

## 4. Module structure

```
src/modules/stripe-onramp/
├── stripe-onramp.module.ts                  # @Module wiring
├── stripe-onramp.controller.ts              # AuthGuard-protected user endpoints
├── stripe-onramp.service.ts                 # Business logic + Stripe API calls
├── stripe-onramp.provider.ts                # Axios client + raw Stripe API wrappers
├── stripe-onramp.token-vault.service.ts     # AES encrypt/decrypt + auto-refresh
├── webhooks/
│   ├── stripe-onramp-webhook.controller.ts  # crypto.onramp_session.updated
│   └── stripe-onramp-webhook.types.ts       # CryptoOnrampSession shape
├── guards/
│   └── stripe-onramp-webhook.guard.ts       # Stripe-Signature HMAC verify
├── dto/
│   ├── create-link-auth-intent.dto.ts       # { email }
│   ├── retrieve-tokens.dto.ts               # { authIntentId, cryptoCustomerId, linkEmail }
│   ├── create-onramp-session.dto.ts         # { cryptoPaymentToken, sourceAmount, destinationCurrency, destinationNetwork }
│   └── checkout.dto.ts                      # { mandateData? }
├── types/
│   └── stripe-onramp.types.ts               # CryptoCustomer, CryptoOnrampSession, CheckoutResult
└── utils/
    └── stripe-error.util.ts                 # Centralized Stripe → NestJS error mapping

# Files touched outside the new module:
src/common/utils/crypto.util.ts              # NEW — AES-256-GCM encrypt/decrypt (reusable)
src/modules/onramp-aggregator/                # MODIFIED — add Stripe as 4th provider
src/modules/account/account.handler.ts        # MODIFIED — null tokens on delete
src/config/validation.schema.ts               # MODIFIED — add STRIPE_* Joi rules
src/config/configuration.ts                   # MODIFIED — add stripe config block
prisma/schema.prisma                          # MODIFIED — add StripeOnrampCustomer
prisma/migrations/<timestamp>_add_stripe_onramp_customer/migration.sql
.env.example (or .env)                        # MODIFIED — add STRIPE_* vars
```

### Module wiring

```ts
@Module({
  imports: [ConfigModule, DatabaseModule, MoonPayBuyModule],
  controllers: [StripeOnrampController, StripeOnrampWebhookController],
  providers: [
    StripeOnrampService,
    StripeOnrampProvider,           // axios client + raw Stripe API calls
    StripeOnrampTokenVaultService,  // encrypt/decrypt/refresh
    StripeOnrampWebhookGuard,
  ],
  exports: [StripeOnrampService],
})
export class StripeOnrampModule {}
```

`MoonPayBuyModule` is imported to inject `MoonPayBuyService.getWalletAddressForUser`. Same dependency that `OnrampAggregatorModule` and `SwappedModule` already declare. Keeps wallet derivation in one place.

### Three service layers

1. **`StripeOnrampProvider`** — thin axios wrapper. One client instance with `Authorization: Bearer sk_live_...` header. Per-request, the Service injects `Stripe-OAuth-Token: <user_token>` and `Stripe-Version` headers. Request/response interceptors log `requestId` + duration and route errors through `stripe-error.util.ts`. Mirrors `bridge.provider.ts:67` and `moonpay-buy.provider.ts:52`.

2. **`StripeOnrampTokenVaultService`** — owns the OAuth lifecycle. See §6 for the full method set.

3. **`StripeOnrampService`** — business logic facade. Methods map 1:1 to the endpoints in §5. Orchestrates Provider + Vault + Prisma.

**Why split Provider from Service?** The Provider is pure HTTP with no DB dependency, so it's trivially mockable in unit tests. The Service stays focused on business rules (idempotency, KYC checks, Transaction writes). This is exactly the Bridge split.

### Aggregator integration

One method added to `StripeOnrampService`:

```ts
/**
 * Quote for the aggregator. Returns null if:
 * - User has no StripeOnrampCustomer row (never authenticated with Stripe)
 * - Stripe's fee lookup fails
 *
 * Stripe can't quote without an authenticated Link session, so unlike
 * Coinbase/MoonPay, it only appears for returning users.
 */
async getQuote(params: {
  userId: string;
  fiatAmount: number;
  destinationNetwork: string;
}): Promise<OnrampQuoteOption | null>
```

In `OnrampAggregatorService.getQuotes()` (around line 138), add a 4th task:

```ts
tasks.push(
  this.stripe
    .getQuote({ userId, fiatAmount, destinationNetwork: 'base' })
    .then(quote => (quote ? { ...quote, provider: 'stripe' as const } : null))
    .catch(() => null),
);
```

Update `OnrampProvider` type union: `'coinbase' | 'swapped-ramp' | 'moonpay' | 'stripe'`.

---

## 5. Endpoints

Six endpoints total — five auth-guarded for the client, one unguarded for the webhook. All client endpoints under `POST /api/v1/stripe-onramp/*`.

### Client-facing endpoints

All `@UseGuards(AuthGuard)`. `req.user = { userId, email, subOrgId }` (per existing `auth.guard.ts`).

#### `POST /api/v1/stripe-onramp/link-auth-intent`

Creates a Stripe LinkAuthIntent. Called by the client after `hasLinkAccount` (and `registerLinkUser` if needed).

- **Request:** `{ email: string }`
- **Response:** `{ authIntentId: string, expiresAt: number }` (epoch seconds)
- **Backend:**
  ```http
  POST https://login.link.com/v1/link_auth_intent
  Authorization: Bearer $STRIPE_SECRET_KEY
  Content-Type: application/json

  {
    "email": "<email>",
    "oauth_client_id": "$STRIPE_OAUTH_CLIENT_ID",
    "oauth_scopes": "kyc.status:read,crypto:ramp"
  }
  ```
- **Errors:** `ServiceUnavailableException` on Stripe failure or missing config.

#### `POST /api/v1/stripe-onramp/tokens`

Exchanges a consented `authIntentId` for OAuth tokens and persists them (encrypted). Called from the client's `authorize` callback after `result.status === 'Consented'`.

- **Request:**
  ```ts
  {
    authIntentId: string,      // from /link-auth-intent
    cryptoCustomerId: string,  // from SDK authorize callback result.customerId
    linkEmail: string          // the email used for LinkAuthIntent
  }
  ```
- **Response:** `{ stored: true }`
- **Backend:**
  ```http
  POST https://login.link.com/v1/link_auth_intent/{authIntentId}/tokens
  Authorization: Bearer $STRIPE_SECRET_KEY
  ```
  Stripe response:
  ```json
  {
    "access_token": "liwltoken_xxx",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh": {
      "refresh_token": "liwlrefresh_xxx",
      "expires_in": 7776000
    }
  }
  ```
  → `tokenVault.storeTokens(userId, { cryptoCustomerId, linkEmail, authIntentId, accessToken, accessExpiresIn: 3600, refreshToken, refreshExpiresIn: 7776000 })`
- **Errors:** `BadRequestException` if `authIntentId` not consented. `ServiceUnavailableException` on Stripe failure.

#### `GET /api/v1/stripe-onramp/customer`

Returns KYC status. Client uses this to decide whether to call `attachKycInfo` / `verifyIdentity`.

- **Response:**
  ```ts
  {
    cryptoCustomerId: string,
    kycStatus: 'not_started' | 'submitted' | 'verified' | 'rejected',
    kycRequired: boolean   // true iff kyc_verified status === 'not_started'
  }
  ```
- **Backend:**
  ```http
  GET https://api.stripe.com/v1/crypto/customers/{cryptoCustomerId}
  Authorization: Bearer $STRIPE_SECRET_KEY
  Stripe-OAuth-Token: <decrypted access token from vault>
  Stripe-Version: 2026-05-27.preview;crypto_onramp_beta=v2
  ```
  Inspect `verifications[]` for an entry with `type: 'kyc_verified'`. Cache result on `StripeOnrampCustomer.kycStatus` + `kycCheckedAt` (5-minute TTL).
- **Errors:** `UnauthorizedException('Stripe re-authentication required')` if no stored token or refresh fails.

#### `POST /api/v1/stripe-onramp/session`

Creates a headless CryptoOnrampSession. Also creates a PENDING `Transaction` row so the webhook has something to update.

- **Request:**
  ```ts
  {
    cryptoPaymentToken: string,                            // from SDK's createCryptoPaymentToken
    sourceAmount: number,                                  // fiat in USD
    destinationCurrency: 'usdc',
    destinationNetwork: 'solana' | 'ethereum' | 'base' | 'polygon'
  }
  ```
- **Response:** `{ sessionId: string }`
- **Backend:**
  1. `walletAddress = await moonpay.getWalletAddressForUser(userId, destinationNetwork === 'solana')`
  2. `accessToken = await tokenVault.getValidAccessToken(userId)`
  3. `customer = await prisma.stripeOnrampCustomer.findUnique({ where: { userId } })`
  4. POST to Stripe:
     ```http
     POST https://api.stripe.com/v1/crypto/onramp_sessions
     Authorization: Bearer $STRIPE_SECRET_KEY
     Stripe-OAuth-Token: <accessToken>
     Stripe-Version: 2026-05-27.preview;crypto_onramp_beta=v2
     Content-Type: application/x-www-form-urlencoded

     transaction_details[crypto_customer_id]=<cryptoCustomerId>
     &transaction_details[crypto_payment_token]=<cryptoPaymentToken>
     &transaction_details[source_currency]=usd
     &transaction_details[source_amount]=<sourceAmount>
     &transaction_details[destination_currency]=usdc
     &transaction_details[destination_network]=<destinationNetwork>
     &transaction_details[destination_networks][]=<destinationNetwork>
     &transaction_details[wallet_address]=<walletAddress>
     &transaction_details[customer_ip_address]=<req.ip>
     &ui_mode=headless
     ```
  5. Create PENDING Transaction:
     ```ts
     {
       id: `stripe_onramp_${session.id}`,
       quoteId: `stripe_onramp_${session.id}`,   // unique — idempotency
       userId, type: DEPOSIT, status: PENDING, service: 'stripe_onramp',
       fromAsset: 'USD', fromAmount: String(sourceAmount),
       toAsset: 'USDC', toNetwork: destinationNetwork, toAddress: walletAddress,
       updatedAt: now,
       metadata: { ...session, provider: 'Stripe Embedded Components',
                   stripeSessionId: session.id },
     }
     ```
     P2002 catch → return existing session id (rare; means a duplicate request).
- **Errors:** `NotFoundException` if no wallet or no Stripe customer. `BadRequestException` on invalid destination network. `ServiceUnavailableException` on Stripe failure.

#### `POST /api/v1/stripe-onramp/checkout/:sessionId`

Proxies Stripe's checkout endpoint from inside the SDK's `performCheckout` callback. **Must never be called directly from outside `performCheckout`** — the SDK handles 3DS / next-action flows between calls.

- **Request body (optional):** `{ mandateData?: { ip_address, user_agent, acceptance } }` (only on ACH retries)
- **Response:** `{ clientSecret: string }`
- **Backend:**
  1. Look up Transaction by id `stripe_onramp_${sessionId}` to get `userId`.
  2. `accessToken = await tokenVault.getValidAccessToken(userId)`
  3. POST to Stripe:
     ```http
     POST https://api.stripe.com/v1/crypto/onramp_sessions/{sessionId}/checkout
     Authorization: Bearer $STRIPE_SECRET_KEY
     Stripe-OAuth-Token: <accessToken>
     Stripe-Version: 2026-05-27.preview;crypto_onramp_beta=v2
     ```
     (include `mandate_data[...]` if present in the request body)
  4. Return `{ clientSecret: response.client_secret }`
- **Does NOT update Transaction status.** The webhook does that.
- **Errors:** `ServiceUnavailableException` on Stripe failure (SDK will retry via its callback loop). If Stripe returns a mandate-required error, propagate to client; client collects acceptance and re-calls with `mandateData`.

### Webhook endpoint

#### `POST /api/v1/stripe-onramp/webhook`

Receives `crypto.onramp_session.updated` events. **No AuthGuard**; protected by HMAC guard. `@SkipThrottle()`.

- **Auth:** `Stripe-Signature: t=<timestamp>,v1=<hex>` header, HMAC-SHA256 over `${timestamp}.${rawBody}` with `STRIPE_WEBHOOK_SECRET`. Verified by `StripeOnrampWebhookGuard` (mirrors `moonpay-onramps-webhook.guard.ts`).
- **Event types handled:** `crypto.onramp_session.updated`
- **Event types ignored:** anything else (log + 200)
- **Status mapping** (Stripe session status → `TransactionStatus`):

  | Stripe status | TransactionStatus | Action |
  |---|---|---|
  | `initialized` | `PENDING` | no-op (already pending from /session) |
  | `requires_payment` | `PENDING` | no-op |
  | `fulfillment_processing` | `EXECUTING` | update status; don't set `completedAt` |
  | `fulfillment_complete` | `COMPLETED` | set `transactionHash = transaction_details.transaction_id`, set `completedAt = now` |
  | `rejected` | `FAILED` | set `metadata.failureReason` |

- **Idempotency** (three layers, mirrors `moonpay-onramps-webhook.controller.ts`):
  1. Deterministic Transaction id: `stripe_onramp_${session.id}`
  2. Terminal-state guard: skip update if row already `COMPLETED` or `FAILED`
  3. P2002 catch (rare; only if the row was never created and a duplicate webhook races)

- **Failure mode:** throw `ServiceUnavailableException` on any transient error → Stripe retries for ~3 days. "Terminal" non-retryable conditions (unknown session id, unhandled event type) return 200 normally.

---

## 6. OAuth token vault

The most security-sensitive piece. Three parts: the encryption util, the storage shape, the lifecycle service.

### `src/common/utils/crypto.util.ts` (new, reusable)

Plain exported functions (not a service). Trivially testable, reusable for future secrets.

```ts
import { createCipheriv, createDecipheriv, randomBytes, scryptSync } from 'crypto';

const ALGO = 'aes-256-gcm';
const KEY_BYTES = 32;
const IV_BYTES = 12;

let cachedKey: Buffer | null = null;

/**
 * Encrypt a plaintext string with AES-256-GCM.
 * Key is derived once via scrypt from the env value (fixed salt).
 * Returns JSON envelope: { ciphertext, iv, tag } (all base64).
 */
export function encrypt(plaintext: string, envKey: string): string {
  const key = deriveKey(envKey);
  const iv = randomBytes(IV_BYTES);
  const cipher = createCipheriv(ALGO, key, iv);
  const ciphertext = Buffer.concat([
    cipher.update(plaintext, 'utf8'),
    cipher.final(),
  ]);
  const tag = cipher.getAuthTag();
  return JSON.stringify({
    ciphertext: ciphertext.toString('base64'),
    iv: iv.toString('base64'),
    tag: tag.toString('base64'),
  });
}

/**
 * Decrypt a JSON envelope produced by encrypt().
 * Throws if the GCM auth tag is invalid (tamper detection).
 * Callers should treat any throw as "token is corrupt, force re-auth".
 */
export function decrypt(envelope: string, envKey: string): string {
  const { ciphertext, iv, tag } = JSON.parse(envelope);
  const key = deriveKey(envKey);
  const decipher = createDecipheriv(ALGO, key, Buffer.from(iv, 'base64'));
  decipher.setAuthTag(Buffer.from(tag, 'base64'));
  return Buffer.concat([
    decipher.update(Buffer.from(ciphertext, 'base64')),
    decipher.final(),
  ]).toString('utf8');
}

function deriveKey(envKey: string): Buffer {
  if (cachedKey && cachedKey.length === KEY_BYTES) return cachedKey;
  // Fixed salt: threat model is "DB exfil without env". If env leaks, no encryption helps.
  const salt = Buffer.from('anzo-stripe-oauth-salt', 'utf8');
  cachedKey = scryptSync(envKey, salt, KEY_BYTES, { N: 16384, r: 8, p: 1 });
  return cachedKey;
}
```

**Design choices:**
- **AES-256-GCM** for authenticated encryption (built-in tamper detection via the `tag`).
- **scrypt** with N=16384 (OWASP minimum for interactive auth) to add brute-force cost if the env value is weak.
- **Fixed salt** — threat model is "DB exfiltration without env". If env can leak, encryption is moot.
- **Key cached at module load** — rotation requires redeploy + a re-encrypt migration (documented limitation, out of scope v1).

### `StripeOnrampTokenVaultService`

```ts
@Injectable()
export class StripeOnrampTokenVaultService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly config: ConfigService,
    private readonly provider: StripeOnrampProvider,
  ) {}

  /** Called from POST /tokens after Stripe returns the token bundle. */
  async storeTokens(userId: string, bundle: {
    cryptoCustomerId: string;
    linkEmail: string;
    authIntentId: string;
    accessToken: string;
    accessExpiresIn: number;     // seconds
    refreshToken: string;
    refreshExpiresIn: number;    // seconds
  }): Promise<void> { /* upsert StripeOnrampCustomer */ }

  /**
   * Returns a live access token. Refreshes transparently if expired.
   * Throws UnauthorizedException if there's no stored refresh token
   * (user must re-authenticate via the SDK flow).
   *
   * Implementation outline (full code in implementation plan):
   *   1. SELECT accessTokenCipher, accessTokenExpiresAt, refreshTokenCipher
   *   2. If accessToken missing OR expires within 60s → refreshAndReturn()
   *   3. Else → decrypt(accessTokenCipher) and return
   */
  async getValidAccessToken(userId: string): Promise<string> { /* impl per outline */ }

  /** Refresh via POST /auth/token. Atomic write of both tokens. */
  private async refreshAndReturn(
    userId: string,
    refreshCipherEnvelope: string,
  ): Promise<string> { /* ... */ }

  /** Called by AccountHandler.deleteAccount. Idempotent. */
  async clearTokens(userId: string): Promise<void> { /* updateMany → null */ }

  private requireEnvKey(): string { /* throw if STRIPE_OAUTH_ENCRYPTION_KEY missing */ }
}
```

### Failure / edge-case matrix

| Scenario | Behavior |
|---|---|
| Access token expires mid-session | `getValidAccessToken` checks expiry; refreshes if < 60s remaining. Caller never sees a 401 due to expiry. |
| Refresh token is one-time-use | Stripe returns a new refresh token on every refresh. We overwrite both tokens atomically in `refreshAndReturn`. |
| Refresh fails (revoked or 90-day expiry) | `provider.refreshAccessToken` throws → propagates as `UnauthorizedException('Stripe re-authentication required')` → client must redo `hasLinkAccount` → `authorize` flow. |
| DB corruption / tamper | GCM `tag` check fails in `decrypt` → throws → same re-auth path. |
| Concurrent refresh (two requests simultaneously) | Both call `/auth/token` with the same refresh token; only one succeeds at Stripe. The other throws, propagates up, client retries. Acceptable — refresh is rare (1x/hour). |
| Account deletion | `clearTokens(userId)` nulls all cipher fields. Refresh tokens become useless. |
| `STRIPE_OAUTH_ENCRYPTION_KEY` rotation | Out of scope v1. Requires redeploy + migration script that decrypts with old key and re-encrypts with new. Documented as known limitation. |

### New env vars

```bash
# Stripe Embedded Components Crypto Onramp (private preview)
STRIPE_SECRET_KEY=sk_live_...                          # account-level secret (server only)
STRIPE_PUBLISHABLE_KEY=pk_live_...                     # shared with client
STRIPE_OAUTH_CLIENT_ID=...                             # provisioned by Stripe during onboarding
STRIPE_OAUTH_CLIENT_SECRET=...                         # provisioned by Stripe during onboarding
STRIPE_WEBHOOK_SECRET=whsec_...                        # from Stripe webhook endpoint config
STRIPE_OAUTH_ENCRYPTION_KEY=<32-byte base64>           # generate via: openssl rand -base64 32
STRIPE_API_VERSION=2026-05-27.preview;crypto_onramp_beta=v2
# Optional feature flag (matches FF_SEISMIC_* convention)
FF_STRIPE_ONRAMP=false
```

Joi validation (`src/config/validation.schema.ts`): make `STRIPE_*` vars conditionally required on `FF_STRIPE_ONRAMP === 'true'` (mirrors the Seismic pattern at line 181). Allows the module to ship disabled by default without breaking startup.

---

## 7. Account deletion integration

Add a step to `AccountHandler.deleteAccount` (`src/modules/account/account.handler.ts:126`) immediately after the `User` flag flip:

```ts
// After: prisma.user.update({ where: { id: userId }, data: { isDeleted: true, ... } });
// Null out Stripe OAuth tokens so a leaked DB row can't be replayed.
try {
  await this.stripeTokenVault.clearTokens(userId);
} catch (err) {
  this.logger.warn(`Failed to clear Stripe tokens for ${userId}: ${String(err)}`);
  // Non-fatal: AuthGuard blocks deleted users with 403 ACCOUNT_DELETED, so
  // even intact tokens can't be used through our backend. Continue.
}
```

Wrap in try/catch and fail-open — a Stripe cleanup failure should NOT block account deletion (matches the existing funds-check fail-open behavior at `account.handler.ts:103`).

`AccountHandler` will need `StripeOnrampTokenVaultService` injected. This introduces a new dependency from `AccountModule` → `StripeOnrampModule`; use `forwardRef` if there's a circular dependency (there shouldn't be).

---

## 8. Error handling & retry strategy

### Centralized error mapper

`src/modules/stripe-onramp/utils/stripe-error.util.ts` — mirrors `bridge-error.util.ts:16`:

| Stripe failure | NestJS exception | Notes |
|---|---|---|
| Network / timeout (`ECONNABORTED`, `ENETUNREACH`) | `ServiceUnavailableException` | Transient |
| HTTP 400 | `BadRequestException(safeMessage)` | Client input issue |
| HTTP 401 | `UnauthorizedException('Stripe re-authentication required')` | OAuth token issue |
| HTTP 403 | `ForbiddenException('Stripe feature gate missing')` | Onboarding issue |
| HTTP 404 | `NotFoundException()` | Resource not found |
| HTTP 429 | `HttpException(429, 'Rate limited by Stripe')` | Backoff |
| HTTP ≥ 500 | `ServiceUnavailableException('Stripe unavailable')` | Upstream outage |
| Default | `InternalServerErrorException` | Unknown |

Always: log full Stripe error body + `requestId` header (never surface raw Stripe messages to clients).

### Retry strategy

| Operation | Retry policy | Why |
|---|---|---|
| `createLinkAuthIntent` | No in-app retry | User-initiated; they'll retry by tapping |
| `retrieveTokens` | No in-app retry | Idempotent at Stripe by `authIntentId` |
| `refreshAccessToken` | **Never** retry | Refresh tokens are one-time-use; retrying burns a second token |
| `getCryptoCustomer` (KYC check) | No retry — cached 5min | Idempotent read |
| `createOnrampSession` | No in-app retry | Each call creates a new session |
| `checkout` | No in-app retry | SDK handles retries via its callback loop |
| Webhook handler | Stripe-side retry (~3 days) | Throw 503 → Stripe redelivers |

**No in-app retry loops.** Stripe's own retry behavior plus the SDK's callback loop covers everything.

### Idempotency (three layers, mirrors MoonPay pattern)

1. **Deterministic Transaction IDs**: `stripe_onramp_${sessionId}` — unique by construction.
2. **P2002 catch on create**: every Stripe `prisma.transaction.create` wraps in the P2002-guard pattern.
3. **Terminal-state guard on update**: webhook's status update refuses to downgrade (`if existing.status === COMPLETED → skip`).

---

## 9. Testing

Mirrors existing spec conventions (`*.spec.ts` next to source).

### Unit tests

- **`crypto.util.spec.ts`** — encrypt/decrypt round-trip, tamper detection (flip a byte in tag → throws), wrong key → throws, large input
- **`stripe-onramp.token-vault.service.spec.ts`** — storeTokens writes both ciphers; getValidAccessToken returns cached if fresh; refreshes if expired; clearTokens nulls fields; throws if no refresh token; decrypt failure forces re-auth
- **`stripe-onramp.service.spec.ts`** — each endpoint method's happy path + Stripe error mapping (mock Provider at DI boundary)
- **`stripe-onramp-webhook.controller.spec.ts`** — all 5 status mappings; idempotency (double delivery = 1 update); terminal-state guard; P2002 handling; unhandled event type returns 200
- **`stripe-onramp-webhook.guard.spec.ts`** — valid signature passes; missing header fails; expired timestamp fails (> 5min); future timestamp fails (> 60s); bad signature fails; replay within window passes; missing env fails closed

### Integration tests

- Mock `StripeOnrampProvider` at the DI boundary; verify the Service orchestrates correctly across provider + vault + prisma
- End-to-end: `/session` → `/checkout/:id` → webhook against an in-memory DB

### What we cannot test in CI

- Real Stripe API calls (private preview, requires onboarding)
- Real OAuth OTP flow
- Live KYC verification

### Manual test plan (post-onboarding, sandbox)

Use Stripe's documented sandbox test values:
- OTP code: `000000`
- KYC name: `John Verified`, SSN: `000000000`, address line 1: `address_full_match`, state: 2-letter code (e.g. `WA`)
- Card: `4242 4242 4242 4242`, amount ≤ $100 USD

Steps:
1. Complete OTP flow with `000000`
2. Submit KYC with the test values above
3. Use test card `4242 4242 4242 4242` with amount ≤ $100
4. Verify `Transaction` row reaches `COMPLETED` in DB
5. Trigger `crypto.onramp_session.updated` webhook from Stripe CLI; verify status mapping
6. Soft-delete account → verify cipher fields nulled
7. Restore account → verify user can re-authenticate from scratch

---

## 10. Out of scope / future work

- **EU onboarding** — `source_currency: 'eur'`, EU KYC fields (`nationalities`, `birth_country`). Separate spec.
- **BTC / ETH / SOL native purchases** — would require aggregator refactoring (currently USDC-on-Base default).
- **Wallet/payment-token listing endpoints** — SDK handles these in v1. Add server-side if needed for admin tooling.
- **`STRIPE_OAUTH_ENCRYPTION_KEY` rotation tooling** — needs a migration script + dual-key decryption support.
- **Webhook event durability table** — if webhook volume grows or processing becomes unreliable, add `StripeOnrampWebhookEvent` (Seismic pattern).
- **Sumsub → Stripe KYC bridging** — if you want to skip Stripe's in-SDK KYC for users already Sumsub-verified, that's Pattern A from the KYC discussion. Not in v1.
- **Seamless sign-in** — Stripe's `authenticateUserWithToken` SDK method, enabled by storing `linkAuthTokenClientSecret`. Reduces friction for returning users but requires a separate Stripe feature gate.

---

## 11. Open questions / blockers

| Item | Status | Owner |
|---|---|---|
| Stripe private-preview onboarding approved | **BLOCKER** — no API calls succeed until this is done, including in sandbox | Stripe account exec |
| `OAUTH_CLIENT_ID` / `OAUTH_CLIENT_SECRET` provisioned | **BLOCKER** — needed for LinkAuthIntent + refresh | Stripe account exec |
| App registered as trusted application (bundle ID / package name) | **BLOCKER** — SDK calls fail without app attestation | Stripe account exec |
| Feature gates enabled on Stripe account | **BLOCKER** — `Unrecognized request URL` 404 if missing | Stripe account exec |
| `STRIPE_OAUTH_ENCRYPTION_KEY` generated | Done locally via `openssl rand -base64 32` | Backend |
| Webhook endpoint registered in Stripe dashboard | After backend deploy | Ops |

---

## 12. Implementation touch points (summary)

Counts below are totals across source, spec, and migration files. Implementation-plan breakdown will itemize per file.

**New source files (13):**
- `src/common/utils/crypto.util.ts` (reusable encryption helper)
- `src/modules/stripe-onramp/stripe-onramp.module.ts`
- `src/modules/stripe-onramp/stripe-onramp.controller.ts`
- `src/modules/stripe-onramp/stripe-onramp.service.ts`
- `src/modules/stripe-onramp/stripe-onramp.provider.ts`
- `src/modules/stripe-onramp/stripe-onramp.token-vault.service.ts`
- `src/modules/stripe-onramp/webhooks/stripe-onramp-webhook.controller.ts`
- `src/modules/stripe-onramp/webhooks/stripe-onramp-webhook.types.ts`
- `src/modules/stripe-onramp/guards/stripe-onramp-webhook.guard.ts`
- `src/modules/stripe-onramp/utils/stripe-error.util.ts`
- `src/modules/stripe-onramp/dto/create-link-auth-intent.dto.ts`
- `src/modules/stripe-onramp/dto/retrieve-tokens.dto.ts`
- `src/modules/stripe-onramp/dto/create-onramp-session.dto.ts`
- `src/modules/stripe-onramp/dto/checkout.dto.ts`
- `src/modules/stripe-onramp/types/stripe-onramp.types.ts`

(Note: that's 15 if you count the 4 DTOs individually; 13 above collapses the DTO folder.)

**New spec files (5):**
- `src/common/utils/crypto.util.spec.ts`
- `src/modules/stripe-onramp/stripe-onramp.service.spec.ts`
- `src/modules/stripe-onramp/stripe-onramp.token-vault.service.spec.ts`
- `src/modules/stripe-onramp/webhooks/stripe-onramp-webhook.controller.spec.ts`
- `src/modules/stripe-onramp/guards/stripe-onramp-webhook.guard.spec.ts`

**New migration (1):**
- `prisma/migrations/<timestamp>_add_stripe_onramp_customer/migration.sql`

**Modified files (8):**
- `prisma/schema.prisma` — add `StripeOnrampCustomer` model
- `src/app.module.ts` — import `StripeOnrampModule`
- `src/config/validation.schema.ts` — add `STRIPE_*` Joi rules (conditional on `FF_STRIPE_ONRAMP`)
- `src/config/configuration.ts` — add `stripe` config block
- `src/modules/onramp-aggregator/onramp-aggregator.service.ts` — add Stripe as 4th quote provider, extend `OnrampProvider` type
- `src/modules/onramp-aggregator/onramp-aggregator.module.ts` — import `StripeOnrampModule`
- `src/modules/account/account.handler.ts` — inject `StripeOnrampTokenVaultService`, call `clearTokens` on delete
- `src/modules/account/account.module.ts` — import `StripeOnrampModule`
- `.env` (and `.env.example` if present) — add Stripe vars
