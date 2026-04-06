# Flows (step by step)

Plain-language flows for **review** and **QA**. For exact JSON fields and curl, use [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md).

---

## Flow A — First-time setup (onboarding)

**Goal:** User exists in Anzo; you create their **Infinia footprint** (company + accounts).

1. User completes **identity KYC** in your app (Sumsub SDK flow you already have).
2. Backend receives **Sumsub webhook** and stores applicant id / review status on **`KycProfile`** (existing KYC pipeline).
3. User (or app) calls **`POST /api/v1/infinia/onboard`**.
4. Backend (`InfiniaOnboardingHandler`):
   - Outside **development**, checks **Sumsub** is ready (`SUMSUB_INFINIA_FOR_CLIENT_ID`, GREEN review, applicant id) — see handler for exact rules.
   - Creates an Infinia **company** (short **idempotency** keys — sandbox is sensitive to long keys).
   - Enables required **products** (internal transfer, payouts MX/BR, account validation, etc.).
   - If configured, generates a **Sumsub share token** and creates an Infinia **account owner** in **EXTERNAL** KYC mode.
   - Creates three **accounts**: USDC_BASE, MXN, BRL.
   - Saves **`infiniaCompanyId`**, **`infiniaOwnerId`** (nullable if owner step skipped), and **`InfiniaAccount`** rows.

**Idempotent behavior:** If the user already has a company and stored accounts, onboard returns **current status** instead of failing.

---

## Flow B — Deposit USDC (real path)

**Goal:** User’s Infinia **USDC_BASE** balance is funded so an internal transfer can debit it.

1. Client calls **`GET /api/v1/infinia/deposit`** (requires **`InfiniaOwnerRequiredGuard`** where applicable).
2. Backend loads the USDC account from Infinia and returns **deposit instructions** (e.g. **Base** chain address for USDC).
3. User sends **USDC on Base** from your wallet stack (e.g. Turnkey) to that address.
4. When Infinia sees the balance, later **quote** and **payout** can succeed.

**Sandbox:** Balance may already be high, or use **`POST /api/v1/infinia/sandbox/deposit`** when `INFINIA_SANDBOX_ENABLED=true`.

---

## Flow C — Off-ramp (quote → payout → bank)

**Goal:** Convert USDC to MXN or BRL and pay out to CLABE or PIX.

1. **`POST /api/v1/infinia/quote`** with `targetCurrency` (MXN or BRL) and either `sourceAmount` or `targetAmount`, optional `lockSeconds`.
2. Backend creates an Infinia **internal-transfer quote**, applies **FX markup** from config, returns **`quoteId`** and displayed amounts/rate.
3. *(Optional but recommended)* **`POST /api/v1/infinia/validate-bank`** for CLABE or PIX.
4. **`POST /api/v1/infinia/payout`** with `quoteId`, `sourceAmount` matching the quote (tight tolerance), destination object (MX vs BR), optional recipient name.
5. Backend:
   - Validates quote still **ACTIVE** and owned by user (`external_id` prefix).
   - Creates Infinia **internal transfer** tied to the quote; may set **`callback_url`** to your **transfer webhook** (with auth query param) when `APP_URL` is not local.
   - Inserts **`InfiniaTransaction`** with status **`AWAITING_TRANSFER`** (and related metadata).

**Then async:**

6. Infinia processes the internal transfer. When **COMPLETED**, they **POST** to **`/api/v1/infinia/webhook/transfer/:idempotencyKey`** (or you use **sandbox sync-transfer** on localhost).
7. **`handleTransferWebhook`** does **not** trust the body for amounts; it triggers **`triggerPayoutForTransaction`**, which **GETs** the transfer from Infinia, reads **destination amount**, creates **payout** via **Payouts v2**, sets **`payoutOriginId`**, updates DB toward **PENDING** as appropriate.
8. Infinia sends **payout** updates to **`/api/v1/infinia/webhook/payout/:originId`**.
9. **`handlePayoutWebhook`** maps Infinia statuses to stored statuses (including refund-related states — see `mapPayoutStatus` in `infinia.service.ts`).

**Client polling:** **`GET /api/v1/infinia/transactions/:id`** can refresh state from Infinia while pending.

---

## Webhooks — how Infinia calls back

| Route | When | Path params |
|-------|------|-------------|
| Transfer | Internal FX transfer status changes | `:idempotencyKey` (same key used when creating the transfer) |
| Payout | Payout status changes | `:originId` (your **`payoutOriginId`**, e.g. `avvio-{userId}-{timestamp}`) |

**Auth (`InfiniaWebhookGuard`):**

- **Preferred:** **`?token=`** = hex **HMAC-SHA256**(`INFINIA_WEBHOOK_SECRET`, `"transfer:{idempotencyKey}"` or `"payout:{originId}"`). This is what **`InfiniaPayoutHandler`** appends when building callback URLs.
- **Legacy:** **`?secret=`** or **`Authorization: Bearer`** with the same secret (for old registrations).

**Important:** Infinia’s general doc [Webhooks](https://docs.infiniaweb.com/reference/webhooks) describes **`X-Infinia-Signature`** on the raw body. This integration primarily relies on **URL token** auth for these callbacks; see [ARCHITECTURE.md](./ARCHITECTURE.md#design-choices-worth-knowing-review-angle).

**Response:** Webhook controllers return **`{ "received": true }` quickly and process async** (errors are logged; Infinia retries on non-2xx per their docs).

---

## Flow D — Local development without public URL

1. `APP_URL` may be localhost → **callback_url** omitted on internal transfer creation.
2. After **`POST /payout`**, call **`POST /api/v1/infinia/sandbox/sync-transfer/:transactionId`** (sandbox flag on) to poll Infinia for transfer **COMPLETED** and run the same payout step as the webhook.

---

## Status vocabulary (high level)

`InfiniaTransaction.status` moves roughly:

- **`AWAITING_TRANSFER`** — internal transfer created; waiting for completion webhook/sync.
- **`PROCESSING`** — used during atomic handoff to payout creation (see handler).
- **`PENDING`** — payout submitted; fiat rail in progress.
- **`COMPLETED`** / **`FAILED`** — terminal for happy / sad path.
- **`REFUNDED`** / **`PARTIALLY_REFUNDED`** — may appear after payout events (mapped in service).

Exact transitions are implemented in **`infinia.service.ts`** and **`infinia-payout.handler.ts`**; the API reference doc lists more examples.
