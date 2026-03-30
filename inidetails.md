# Infinia Payouts Integration — Verified Implementation Plan

> **Version:** 1.0  
> **Date:** 2026-03-27  
> **Status:** Pre-implementation analysis  
> **Author:** AI-assisted (verified against live Infinia API OpenAPI specs)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Discrepancies Found in the Original Proposal](#2-discrepancies-found-in-the-original-proposal)
3. [Infinia API — Verified Endpoint Reference](#3-infinia-api--verified-endpoint-reference)
4. [Detailed Architectural Plan](#4-detailed-architectural-plan)
5. [Schema Changes (Prisma)](#5-schema-changes-prisma)
6. [Environment Variables](#6-environment-variables)
7. [Workflow Example — End-to-End User Story](#7-workflow-example--end-to-end-user-story)
8. [Dependencies](#8-dependencies)
9. [Contextual Integration & Robustness](#9-contextual-integration--robustness)
10. [Verified Supported Countries & Destination Types](#10-verified-supported-countries--destination-types)
11. [Webhook Integration](#11-webhook-integration)
12. [Sandbox Testing Guide](#12-sandbox-testing-guide)
13. [Open Questions & Decisions Required](#13-open-questions--decisions-required)

---

## 1. Executive Summary

This document outlines a **verified** plan to integrate Infinia's Payouts v2 API into the Anzo backend, enabling users to off-ramp funds from their wallets to bank accounts across Latin America, Europe, and the UK.

The plan was created after:

- Fetching and analyzing the **complete OpenAPI 3.1.0 specifications** from `docs.infiniaweb.com` for all three payout endpoints
- Analyzing the **full Infinia webhook documentation** (HMAC-SHA256 signature verification, retry policies, IP allowlists)
- Analyzing the **Infinia Accounts API** (`GET /v1/accounts/{account_id}/`) for balance checking and source account management
- Deep-diving into the **entire Anzo backend codebase** — module structure, Prisma schema, configuration, Bridge.xyz integration patterns, provider architecture, and `rules.md`

The integration follows Anzo's established patterns precisely: a self-contained `InfiniaModule` with Controller → Service → Handler → Provider layering, additive Prisma models, Joi-validated environment variables, and HMAC-verified webhook endpoints.

---

## 2. Discrepancies Found in the Original Proposal

The original plan contained several inaccuracies when compared to the actual Infinia API OpenAPI specifications. **All items below are corrections based on the verified docs.**

### 2.1 Incorrect / Missing Status Values

| Original Plan | Actual API (`v2_Status` enum) |
|---|---|
| `IN_PROGRESS`, `COMPLETED`, `FAILED` | `COMPLETED`, `IN_PROGRESS`, `FAILED`, `REFUNDED`, `PARTIALLY_REFUNDED`, `ERROR` |

The plan was missing **3 statuses**: `ERROR`, `REFUNDED`, `PARTIALLY_REFUNDED`. The `ERROR` status is distinct from `FAILED` and is what the sandbox returns for test amounts 99999/99998.

### 2.2 Missing Countries

| Missing from Original Plan | Details |
|---|---|
| **UNITED_KINGDOM** | Listed as a separate country enum value (not just under EUROPE). Uses `ACCOUNT_UNITED_KINGDOM_CHAPS_FPS` destination type with Sort Code + Account Number or IBAN. |
| **BOLIVIA** | Fully supported with `ACH` destination type, requiring bank code, branch, account number, name, and document. |

### 2.3 Missing Destination Types

| Country | Missing Type | Details |
|---|---|---|
| Argentina | **`QR_CODE`** | Accepts a `qrCode` string for QR-code-based payouts |
| Colombia | **`COBRE_BALANCE`** | Requires `currency` and `externalAccount` identifier |
| Colombia | **`ACCOUNT_COLOMBIA`** (expanded) | Requires additional fields: `fullName`, `email`, `phoneNumber`, `bankCode` (integer, not string) |

### 2.4 Incorrect Required Fields

| Field | Original Plan | Actual Requirement |
|---|---|---|
| `sourceAccountId` | Listed as required | **Optional** per the OpenAPI spec. Only `originId`, `amount`, and `destinationAccount` are required. |
| Mexico CLABE `reference` | Listed as optional | **Required** per `MexicoClabe` schema |
| Europe `reference` | Not mentioned | **Required** at the `EuropePayout` level |
| Bolivia `reference` | Not mentioned | **Required** in `BoliviaACH` schema |
| Chile `name` | Not mentioned | **Required** in `ChileAccount` schema |

### 2.5 Missing API Features

| Feature | Details |
|---|---|
| **`callbackUrl`** on Create Payout | The API supports a per-payout `callbackUrl` (max 2083 chars, URI format). When set, Infinia sends a POST callback on status change with the same schema as Get Payout. This is the **preferred** way to receive status updates instead of polling. |
| **`merchantType`** | Optional enum: `UNLICENSED_GAMBLING`, `LICENSED_GAMBLING`, `FOREX`, `CRYPTO_EXCHANGE`, `DIGITAL_WALLET`, `E_COMMERCE`, `ONLINE_GAMING`, `RETAIL`, `OTHER` |
| **`merchantName`** | Optional string identifying the merchant |
| **`x-company-id`** header | All endpoints support an optional header for child-company authentication in multi-tenant setups |
| **`adultCheck`** (Argentina) | Boolean that triggers age verification before payout execution |
| **Proxy Account** (Europe/UK) | `proxyAccount` object for preventing name-mismatch failures on SEPA/CHAPS transfers |
| **`instant`** flag (Europe/UK) | Boolean (default `true`) to control whether SEPA Instant or FPS is used |
| **`acceptsRetries`** (Crypto) | Boolean for global/crypto payouts to enable automatic retries on failure |

### 2.6 Internal Currency Conversion — Not a Documented v2 Endpoint

The original plan's "Step A: Internal Currency Conversion" described triggering an Internal Transfer to convert USDC to MXN before the payout. **There is no documented v2 endpoint for internal transfers.** The Accounts API lists `INTERNAL_TRANSFER` as a product type on accounts, but the actual mechanism is:

- Anzo funds the Infinia source account with crypto (USDC/USDT) or fiat
- Infinia handles the FX conversion internally when executing the payout
- The `sourceAccountId` tells Infinia which pre-funded account to debit

**Decision required:** Clarify with Infinia whether the FX conversion is automatic upon payout execution or requires a separate step via an undocumented API.

### 2.7 Response Schema Issues

The `v2_PayoutResponse` includes fields not mentioned in the original plan:

- `transactionId` (Infinia's internal transaction ID, different from `id`)
- `voucherId` (voucher reference)
- `bank` (enum of bank partners: BIND, VOLUTI, STPMX, etc.)
- `msgStatus` (human-readable status message)
- `sourceAccount` (full source account details in response)
- `destination` (free-form JSON of destination details)
- `creditorAccount` (resolved creditor account details)
- `refunds[]` (array of refund records)
- `extraInfo` (additional transaction metadata)
- `company` (company that created the payout)
- `createdAtIso` / `updatedAtIso` (ISO 8601 timestamps)

---

## 3. Infinia API — Verified Endpoint Reference

All endpoints verified from the actual OpenAPI 3.1.0 specs fetched from `docs.infiniaweb.com`.

### 3.1 Payouts v2

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/v2/payouts/` | Create a new payout | HTTPBasic |
| `GET` | `/v2/payouts/` | List payouts (paginated, filterable) | HTTPBasic |
| `GET` | `/v2/payouts/{payout_id}/` | Get payout details by ID | HTTPBasic |

**Base URL (Sandbox):** `https://app2test.infiniaweb.com/infinia_api`  
**Base URL (Production):** `https://app.infiniaweb.com/infinia_api` (to be confirmed)

### 3.2 Accounts v1

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/v1/accounts/{account_id}/` | Get account details (balance, funding instructions, products) | HTTPBasic |

### 3.3 Payments v1 (Payins — Future Scope)

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/v1/payments` | Create a payment instruction (on-ramp) | HTTPBasic |

### 3.4 Create Payout — Full Request Schema

```typescript
interface PayoutCreateRequest {
  // REQUIRED
  originId: string;          // Max 100 chars. Idempotency key. Reuse = rejection.
  amount: number;            // Transaction amount in destination currency
  destinationAccount: DestinationAccount; // Country-specific destination

  // OPTIONAL
  sourceAccountId?: string;  // Source account to debit (recommended over sourceAccount)
  sourceAccount?: SourceAccount; // DEPRECATED — use sourceAccountId
  callbackUrl?: string;      // URL for status-change POST notifications (max 2083 chars)
  merchantType?: MerchantTypes; // Industry classification
  merchantName?: string;     // Merchant name
}
```

### 3.5 Create Payout — Full Response Schema

```typescript
interface PayoutSuccessCreateResponse {
  status: 'success' | '';
  data: {
    id: string;                // Infinia payout ID
    transactionId?: string;    // Infinia internal transaction ID
    originId?: string;         // Echo of your idempotency key
    voucherId?: string;        // Voucher reference
    bank: Banks;               // Bank partner used (BIND, STPMX, VOLUTI, etc.)
    country: Country;          // Destination country
    currency: Currency;        // Destination currency
    createdAt: string;         // Creation timestamp
    updatedAt?: string;        // Last update timestamp
    amount: number;            // Payout amount
    msgStatus?: string;        // Human-readable status message
    status: Status;            // COMPLETED | IN_PROGRESS | FAILED | ERROR | REFUNDED | PARTIALLY_REFUNDED
    sourceAccount?: SourceAccount;
    destination?: object;      // Free-form destination details
    refunds?: RefundSchema[];  // Refund records (if any)
    creditorAccount?: CreditorAccount;
    extraInfo?: object;        // Additional metadata
    callbackUrl?: string;
    merchantName?: string;
    merchantType?: MerchantTypes;
    company: { id: number; name: string };
    createdAtIso?: string;     // ISO 8601 timestamp
    updatedAtIso?: string;     // ISO 8601 timestamp
  };
}
```

### 3.6 List Payouts — Query Parameters

| Parameter | Type | Description |
|---|---|---|
| `bank` | enum | Filter by bank partner |
| `country` | enum | Filter by country |
| `page_number` | integer | Default: 1 |
| `page_size` | integer | Default: 50 |
| `payout_id` | string | Filter by specific payout ID |
| `include_child_companies` | boolean | Include child company payouts (default: false) |
| `date_from` | string | Format: `YYYY-MM-DD HH:MM:SS` (UTC) |
| `date_to` | string | Format: `YYYY-MM-DD HH:MM:SS` (UTC) |
| `status` | enum | Filter by status |
| `currency` | enum | Filter by currency |
| `amount_min` | number | Minimum amount filter |
| `amount_max` | number | Maximum amount filter |
| `origin_id` | string | Filter by origin ID |

---

## 4. Detailed Architectural Plan

### 4.1 Module Structure

Following the exact patterns from `BridgeModule`, `SwapsModule`, and Anzo's `rules.md`:

```
src/modules/infinia/
├── infinia.module.ts              # Module registration
├── infinia.controller.ts          # User-facing REST endpoints
├── infinia.service.ts             # Facade service (delegates to handlers)
├── providers/
│   └── infinia.provider.ts        # Axios HTTP client for Infinia API
├── handlers/
│   ├── payee.handler.ts           # CRUD for saved payee accounts
│   ├── payout.handler.ts          # Payout creation, status tracking
│   └── payout-status.handler.ts   # Webhook processing & polling logic
├── dto/
│   ├── create-payee.dto.ts        # Validated DTO for adding a payee
│   ├── create-payout.dto.ts       # Validated DTO for initiating a payout
│   └── list-payouts.dto.ts        # Validated query DTO for listing payouts
├── guards/
│   └── infinia-webhook.guard.ts   # HMAC-SHA256 webhook signature verification
├── webhooks/
│   └── infinia-webhook.controller.ts  # Webhook endpoint for status callbacks
├── utils/
│   └── infinia.types.ts           # TypeScript interfaces matching Infinia OpenAPI schemas
└── index.ts                       # Barrel exports
```

### 4.2 Component Responsibilities

**`infinia.module.ts`**
- Imports: `ConfigModule`, `DatabaseModule`
- Controllers: `InfiniaController`, `InfiniaWebhookController`
- Providers: `InfiniaService`, `InfiniaProvider`, `PayeeHandler`, `PayoutHandler`, `PayoutStatusHandler`, `InfiniaWebhookGuard`
- Exports: `InfiniaService` (for potential cross-module usage)

**`infinia.controller.ts`** — `@Controller('infinia')` with `@UseGuards(AuthGuard)`
| Method | Route | Handler | Description |
|---|---|---|---|
| `POST` | `/payees` | `PayeeHandler.create()` | Save a new payee bank account |
| `GET` | `/payees` | `PayeeHandler.list()` | List user's saved payees |
| `GET` | `/payees/:id` | `PayeeHandler.getById()` | Get payee details |
| `DELETE` | `/payees/:id` | `PayeeHandler.delete()` | Remove a saved payee |
| `POST` | `/payouts` | `PayoutHandler.create()` | Initiate a fiat payout |
| `GET` | `/payouts` | `PayoutHandler.list()` | List user's payout history |
| `GET` | `/payouts/:id` | `PayoutHandler.getById()` | Get payout status/details |
| `GET` | `/payouts/:id/status` | `PayoutHandler.getStatus()` | Poll payout status from Infinia |
| `GET` | `/supported-countries` | `PayeeHandler.getSupportedCountries()` | Return country/destination-type metadata |

**`infinia-webhook.controller.ts`** — `@Controller('infinia/webhook')` (NO AuthGuard)
| Method | Route | Guard | Description |
|---|---|---|---|
| `POST` | `/` | `InfiniaWebhookGuard` | Receive payout status callbacks from Infinia |

**`infinia.provider.ts`** — Modeled after `BridgeProvider`
- Creates `axios` instance with HTTPBasic auth (`Authorization: Basic base64(user:password)`)
- Request/response interceptors for logging
- Methods: `createPayout()`, `getPayout()`, `listPayouts()`, `getAccount()`
- Configurable timeout (30s)

**`infinia.service.ts`** — Thin facade per `rules.md` §2.2
- Delegates all calls to handlers
- No direct business logic

### 4.3 Why This Architecture

| Anzo Pattern | Infinia Implementation | Precedent |
|---|---|---|
| Controller → Service → Handler | `InfiniaController` → `InfiniaService` → `PayeeHandler`/`PayoutHandler` | Bridge, Swaps, Auth |
| Dedicated Provider for HTTP | `InfiniaProvider` with Axios + interceptors | `BridgeProvider`, `OneBalanceProvider`, `LifiService` |
| Separate Webhook Controller + Guard | `InfiniaWebhookController` + `InfiniaWebhookGuard` | `BridgeWebhookController` + `BridgeWebhookGuard` |
| DTOs with class-validator | All input validated via DTOs | Global `ValidationPipe` with `whitelist: true` |
| Module isolation | New module, no changes to existing modules | Every feature module in codebase |

---

## 5. Schema Changes (Prisma)

All changes are **purely additive** — no existing tables or columns are modified.

### 5.1 New Model: `InfiniaPayee`

Stores saved payee bank account details. Uses a `Json` field for country-specific banking information (analogous pattern to `BridgeExternalAccount` but more flexible due to the variety of destination types across 10+ countries).

```prisma
model InfiniaPayee {
  id              String   @id @default(cuid())
  userId          String
  nickname        String   // User-defined label, e.g., "Bob MX", "Maria BR"
  country         String   // ARGENTINA, BRAZIL, MEXICO, CHILE, COLOMBIA, EUROPE, PERU, PARAGUAY, BOLIVIA, UNITED_KINGDOM
  currency        String   // ARS, BRL, MXN, CLP, COP, EUR, GBP, PEN, PYG, BOB, USD
  destinationType String   // ALIAS, CBU, QR_CODE, CLABE, CHAVE PIX, ACCOUNT_BRAZIL, BR_CODE, etc.
  details         Json     // Country-specific fields (clabe, alias, chavePix, iban, bankCode, etc.)
  isActive        Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  user    User            @relation(fields: [userId], references: [id], onDelete: Cascade)
  payouts InfiniaPayout[]

  @@unique([userId, nickname])
  @@index([userId])
  @@index([country])
}
```

### 5.2 New Model: `InfiniaPayout`

Tracks the lifecycle of each payout transaction. The `metadata` field stores the full Infinia API response for audit/debugging.

```prisma
model InfiniaPayout {
  id                  String    @id @default(cuid())
  userId              String
  infiniaPayoutId     String    @unique  // The 'id' returned by Infinia API
  infiniaTransactionId String?            // The 'transactionId' from Infinia (different from payoutId)
  originId            String    @unique  // Our idempotency key sent to Infinia
  payeeId             String             // FK to InfiniaPayee
  status              String             // COMPLETED, IN_PROGRESS, FAILED, ERROR, REFUNDED, PARTIALLY_REFUNDED
  amount              String             // Amount as string to avoid floating point issues
  currency            String             // Destination currency
  country             String             // Destination country
  sourceAccountId     String?            // Infinia source account used
  bank                String?            // Bank partner used (BIND, STPMX, etc.)
  msgStatus           String?            // Human-readable status from Infinia
  callbackUrl         String?            // The callback URL we registered
  errorMessage        String?            // Error details if failed
  metadata            Json?              // Full Infinia response for audit
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
  completedAt         DateTime?

  user  User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  payee InfiniaPayee @relation(fields: [payeeId], references: [id])

  @@index([userId])
  @@index([status])
  @@index([infiniaPayoutId])
  @@index([originId])
}
```

### 5.3 User Model Update (Additive)

Add two relation fields to the existing `User` model:

```prisma
model User {
  // ... existing fields ...
  InfiniaPayee   InfiniaPayee[]
  InfiniaPayout  InfiniaPayout[]
}
```

### 5.4 Migration Command

```bash
npx prisma migrate dev --name add_infinia_payout_models
```

### 5.5 Design Decisions

| Decision | Rationale |
|---|---|
| `amount` as `String` (not `Decimal`) | Consistent with how `Transaction`, `Swap`, `BridgeTransaction` store amounts in the existing schema. Avoids floating-point precision issues. |
| `details` as `Json` | Country-specific banking fields vary dramatically (CLABE vs Pix Key vs IBAN vs Sort Code). A typed JSON column provides flexibility without requiring country-specific tables. Validated at the DTO/handler level. |
| Separate `InfiniaPayee` from `BridgeExternalAccount` | Different APIs, different field requirements, different lifecycle. Keeps modules isolated per `rules.md` §2.1. |
| `originId` as `@unique` | Prevents duplicate payouts. Infinia rejects duplicate originIds, but the DB constraint provides defense-in-depth. |
| Storing `metadata` as `Json` | Full Infinia response is preserved for debugging, audit trails, and handling future API changes without schema migrations. |

---

## 6. Environment Variables

### 6.1 New Variables

Add to `.env` and `.env.example`:

```env
# ============================================
# Infinia Web Integration (Fiat Payouts)
# ============================================
# Sandbox: https://app2test.infiniaweb.com/infinia_api
# Production: https://app.infiniaweb.com/infinia_api (confirm with Infinia)
INFINIA_API_URL=https://app2test.infiniaweb.com/infinia_api
INFINIA_API_USER=your-infinia-api-user
INFINIA_API_PASSWORD=your-infinia-api-password
INFINIA_SOURCE_ACCOUNT_ID=your-infinia-source-account-id
INFINIA_WEBHOOK_SECRET=your-webhook-hmac-secret
```

### 6.2 Joi Validation Schema Update

Add to `src/config/validation.schema.ts`:

```typescript
// Infinia Payouts Integration
INFINIA_API_URL: Joi.string().optional().default('https://app2test.infiniaweb.com/infinia_api'),
INFINIA_API_USER: Joi.string().optional(),      // Optional for gradual rollout
INFINIA_API_PASSWORD: Joi.string().optional(),   // Optional for gradual rollout
INFINIA_SOURCE_ACCOUNT_ID: Joi.string().optional(),
INFINIA_WEBHOOK_SECRET: Joi.string().optional(), // Required for webhook verification
```

Making these `optional()` follows the same gradual-rollout pattern used for Bridge (`BRIDGE_API_KEY: Joi.string().optional()`).

### 6.3 Configuration Mapping Update

Add to `src/config/configuration.ts`:

```typescript
infinia: {
  apiUrl: process.env.INFINIA_API_URL || 'https://app2test.infiniaweb.com/infinia_api',
  apiUser: process.env.INFINIA_API_USER,
  apiPassword: process.env.INFINIA_API_PASSWORD,
  sourceAccountId: process.env.INFINIA_SOURCE_ACCOUNT_ID,
  webhookSecret: process.env.INFINIA_WEBHOOK_SECRET,
},
```

### 6.4 Variable Explanations

| Variable | Purpose | Source |
|---|---|---|
| `INFINIA_API_URL` | Base URL for all Infinia API calls. Toggle between sandbox and production. | Infinia dashboard |
| `INFINIA_API_USER` | Username for HTTPBasic authentication | Infinia onboarding |
| `INFINIA_API_PASSWORD` | Password for HTTPBasic authentication | Infinia onboarding |
| `INFINIA_SOURCE_ACCOUNT_ID` | Pre-configured source account from which payouts are debited. This is Anzo's funded account within Infinia. | Infinia dashboard / Account API |
| `INFINIA_WEBHOOK_SECRET` | HMAC-SHA256 shared secret for verifying `X-Infinia-Signature` on webhook callbacks | Infinia webhook configuration |

---

## 7. Workflow Example — End-to-End User Story

**User Story:** Alice, an Anzo user, wants to send 10,000 MXN to Bob's bank account in Mexico.

### Step 1: Add Payee

Alice saves Bob's bank details in the Anzo app.

**Frontend Call:** `POST /api/v1/infinia/payees`

```json
{
  "nickname": "Bob MX",
  "country": "MEXICO",
  "currency": "MXN",
  "destinationType": "CLABE",
  "details": {
    "clabe": "123456789012345678",
    "reference": "Payment from Alice"
  }
}
```

**Backend Action:**
1. `InfiniaController.createPayee()` → `InfiniaService.createPayee()` → `PayeeHandler.create()`
2. Handler validates DTO (class-validator ensures `clabe` is 18 digits, `reference` is present since it's required for Mexico)
3. Handler validates that the country/currency/destinationType combination is valid
4. Creates `InfiniaPayee` record linked to Alice's `userId`
5. Returns the saved payee with its `id`

### Step 2: Initiate Payout

Alice selects "Bob MX" from her payees, enters 10,000 MXN.

**Frontend Call:** `POST /api/v1/infinia/payouts`

```json
{
  "payeeId": "cuid_of_bob_mx_payee",
  "amount": "10000.00",
  "currency": "MXN"
}
```

**Backend Processing:**

1. `PayoutHandler.create()` receives the request
2. Looks up the `InfiniaPayee` record by `payeeId`, verifies it belongs to Alice
3. Generates a unique `originId` (UUID v4) for idempotency
4. Constructs the Infinia API payload by merging payee details:

```json
{
  "originId": "anzo_payout_550e8400-e29b-41d4-a716-446655440000",
  "amount": 10000.00,
  "sourceAccountId": "${INFINIA_SOURCE_ACCOUNT_ID}",
  "callbackUrl": "https://api.anzo.app/api/v1/infinia/webhook",
  "destinationAccount": {
    "country": "MEXICO",
    "currency": "MXN",
    "destinationType": {
      "type": "CLABE",
      "clabe": "123456789012345678",
      "reference": "Payment from Alice"
    }
  },
  "merchantType": "DIGITAL_WALLET",
  "merchantName": "Anzo"
}
```

5. `InfiniaProvider.createPayout()` sends the request to `POST /v2/payouts/`
6. Infinia responds with:

```json
{
  "status": "success",
  "data": {
    "id": "inf_payout_12345",
    "transactionId": "inf_tx_67890",
    "originId": "anzo_payout_550e8400-...",
    "bank": "STPMX",
    "country": "MEXICO",
    "currency": "MXN",
    "amount": 10000.00,
    "status": "IN_PROGRESS",
    "createdAt": "2026-03-27 14:30:00",
    "company": { "id": 42, "name": "Anzo" }
  }
}
```

7. Handler creates `InfiniaPayout` record in the database:

```
id:                  cuid()
userId:              alice_user_id
infiniaPayoutId:     "inf_payout_12345"
infiniaTransactionId: "inf_tx_67890"
originId:            "anzo_payout_550e8400-..."
payeeId:             cuid_of_bob_mx_payee
status:              "IN_PROGRESS"
amount:              "10000.00"
currency:            "MXN"
country:             "MEXICO"
sourceAccountId:     "${INFINIA_SOURCE_ACCOUNT_ID}"
bank:                "STPMX"
callbackUrl:         "https://api.anzo.app/api/v1/infinia/webhook"
metadata:            { full Infinia response }
```

8. Returns the created payout to Alice with status `IN_PROGRESS`

### Step 3: Status Update via Callback

When the payout completes (typically minutes for SPEI in Mexico), Infinia sends a POST to our callback URL.

**Infinia → Anzo:** `POST /api/v1/infinia/webhook`

Headers:
```
Content-Type: application/json
event: payout
X-Infinia-Signature: <HMAC-SHA256 signature>
X-Idempotency-Key: 550e8400-e29b-41d4-a716-446655440001
```

Body (same schema as Get Payout response):
```json
{
  "id": "inf_payout_12345",
  "status": "COMPLETED",
  "amount": 10000.00,
  "currency": "MXN",
  "country": "MEXICO",
  "updatedAt": "2026-03-27 14:32:15",
  ...
}
```

**Backend Processing:**
1. `InfiniaWebhookGuard` verifies `X-Infinia-Signature` using HMAC-SHA256 with `INFINIA_WEBHOOK_SECRET`
2. Check `X-Idempotency-Key` against processed events (skip if already processed)
3. `PayoutStatusHandler` looks up `InfiniaPayout` by `infiniaPayoutId`
4. Updates `status` to `COMPLETED`, sets `completedAt`
5. Sends push notification to Alice via OneSignal
6. Responds with `200 OK` immediately (async processing)

### Step 4: Fallback — Manual Status Check

If the callback is missed, Alice can refresh the payout status.

**Frontend Call:** `GET /api/v1/infinia/payouts/{payout_id}/status`

**Backend Action:**
1. `PayoutHandler.getStatus()` calls `InfiniaProvider.getPayout(infiniaPayoutId)`
2. Compares Infinia's current status with our stored status
3. If different, updates the `InfiniaPayout` record
4. Returns the latest status to Alice

---

## 8. Dependencies

**No new npm packages required.** All dependencies are already present in `package.json`:

| Dependency | Version | Usage |
|---|---|---|
| `axios` | ^1.13.2 | HTTP client for Infinia REST API (same as Bridge, LI.FI, OneBalance) |
| `class-validator` | existing | DTO validation decorators |
| `class-transformer` | existing | DTO transformation |
| `@nestjs/config` | ^11.x | ConfigService for environment variables |
| `crypto` (Node.js built-in) | N/A | HMAC-SHA256 for webhook signature verification |

---

## 9. Contextual Integration & Robustness

### 9.1 Pattern Alignment

| Anzo Architecture Rule | How Infinia Integration Complies |
|---|---|
| **Monolith-first** (`rules.md` §2.1) | Self-contained `InfiniaModule` in `src/modules/infinia/`. No microservices. Cross-module calls via DI. |
| **Service & Handler Pattern** (`rules.md` §2.2) | `InfiniaService` = facade. `PayeeHandler`, `PayoutHandler`, `PayoutStatusHandler` = business logic. Controllers only inject Service. |
| **DTOs with class-validator** (`rules.md` §4.2) | All endpoints have DTOs with `@IsString()`, `@IsEnum()`, `@ValidateNested()`, `@IsOptional()`, etc. |
| **ConfigService for secrets** (`rules.md` §6) | All env vars accessed via `ConfigService`, defined in `.env.example`, validated in Joi schema. |
| **AuthGuard on protected routes** (`rules.md` §4.5) | Class-level `@UseGuards(AuthGuard)` on `InfiniaController`. Webhook controller uses `InfiniaWebhookGuard` instead. |
| **Additive DB changes** (`rules.md` §5) | New models only. No existing tables modified. Proper indexing on FKs and query columns. |
| **Documentation** (`rules.md` §8) | This document fulfills the requirement for new module documentation. |

### 9.2 Non-Breaking Guarantees

- **Zero changes to existing modules:** No modifications to Bridge, Swaps, Transfers, Balances, Auth, or any other module
- **Zero changes to existing Prisma models:** Only additive `InfiniaPayee` and `InfiniaPayout` models + User relation fields
- **Existing env vars unchanged:** Only new optional variables added
- **No dependency version changes:** Uses existing `axios`, `class-validator`, etc.
- **Gradual rollout:** All Infinia env vars are `optional()` in Joi, so the module gracefully degrades if credentials are not configured (same pattern as Bridge)

### 9.3 Comparison with Bridge.xyz Integration

| Aspect | Bridge.xyz (existing) | Infinia (proposed) |
|---|---|---|
| Auth | API Key header | HTTPBasic (username:password) |
| Webhook verification | RSA-SHA256 with public key | HMAC-SHA256 with shared secret |
| Idempotency | `Idempotency-Key` header | `originId` field in request body |
| External accounts | Stored in `BridgeExternalAccount` (per customer) | Stored in `InfiniaPayee` (per user) |
| Transactions | Stored in `BridgeTransaction` | Stored in `InfiniaPayout` |
| KYC | Full KYC flow (Bridge + Persona) | No KYC required (Infinia handles at company level) |
| Scope | On-ramp + Off-ramp | Off-ramp (payouts) only |

---

## 10. Verified Supported Countries & Destination Types

Complete data extracted from the `v2_create_payout__post` OpenAPI schema:

### 10.1 Fiat Payouts

| Country | `country` Code | Currencies | Destination Types | Key Required Fields |
|---|---|---|---|---|
| Argentina | `ARGENTINA` | ARS, USD | `ALIAS`, `CBU`, `QR_CODE` | alias / cbu / qrCode |
| Bolivia | `BOLIVIA` | BOB, USD | `ACH` | accountNumber, name, documentNumber, **reference** (required), bankCode, bankBranch |
| Brazil | `BRAZIL` | BRL | `CHAVE PIX`, `ACCOUNT_BRAZIL`, `BR_CODE` | chavePix / (ispb, number, issuer, accountType, documentNumber) / brCode |
| Chile | `CHILE` | CLP | `ACCOUNT_CHILE` | bankCode, accountType (CC/SVGS/VISTA/RUT/SLRY), accountNumber, documentType (RUT/RUN/PAS/CE), documentNumber, **name** (required) |
| Colombia | `COLOMBIA` | COP | `ACCOUNT_COLOMBIA`, `BREB_KEY`, `COBRE_BALANCE` | fullName, documentType, documentNumber, bankCode (integer), accountType, accountNumber / brebKey / externalAccount |
| Europe | `EUROPE` | EUR, GBP | `ACCOUNT_EUROPE_SEPA`, `ACCOUNT_UNITED_KINGDOM_CHAPS_FPS` | **reference** (required at country level); IBAN, beneficiaryType, beneficiaryFirstName, beneficiaryCountry / (sortCode+accountNumber or IBAN) |
| Mexico | `MEXICO` | MXN, USD | `CLABE` | clabe, **reference** (required) |
| Paraguay | `PARAGUAY` | PYG | `ACCOUNT_PARAGUAY` | bankCode (integer), name, accountNumber, documentType (CI/PAS/CRP/CRC/RUC/DNI), documentNumber |
| Peru | `PERU` | PEN | `ACCOUNT_PERU` | bankCode, accountType (CC/SVGS/EFE), documentType (RUC/DNI/PAS/CE), documentNumber, name, **state** (required) |

### 10.2 Crypto Payouts (Global — Lower Priority)

| `country` Code | Currencies | Destination Types | Key Fields |
|---|---|---|---|
| `GLOBAL` | USDT, USDC, EURC | `ETHEREUM`, `TRON`, `POLYGON`, `BASE` | address, acceptsRetries (optional, default false) |

### 10.3 Additional Country-Specific Features

| Country | Special Feature | Field | Description |
|---|---|---|---|
| Argentina | Age verification | `adultCheck` (boolean) | Rejects payout if beneficiary is under 18 |
| Europe (SEPA) | Proxy Account | `proxyAccount` object | Opens proxy account to prevent name-mismatch failures |
| Europe (SEPA) | Instant transfer | `instant` (boolean, default true) | Controls SEPA Instant vs standard SEPA |
| UK (CHAPS/FPS) | Instant transfer | `instant` (boolean, default true) | Controls FPS Instant vs CHAPS |
| UK (CHAPS) | Address required | `beneficiaryCity`, `beneficiaryPostalCode` | Required for non-instant (CHAPS) payments |

### 10.4 v2_Country Enum (Complete)

```
ARGENTINA, BRAZIL, MEXICO, PARAGUAY, PERU, COLOMBIA, CHILE, EUROPE, GLOBAL, BOLIVIA, UNITED_KINGDOM
```

### 10.5 v2_Currency Enum (Complete)

```
ARS, USD, BRL, MXN, PYG, CLP, PEN, EUR, GBP, USDT, USDC, BOB, EURC, COP,
USDC_POL, USDT_POL, EURC_BASE, USDC_BASE, USDT_BASE, USDC_ETH, USDT_ETH, EURC_ETH
```

### 10.6 v2_Banks Enum (Complete)

```
BIND, VOLUTI, NIXFIN, TRACE, GANADERO, INFINIA, PRIVILEGE, STPMX, GNB, OPENPAYD,
BINDPSP, COINAG, VITALCRED, BRLA, NVIO, AVENIA, KOIBANX, PRIVY, BACKOFFICE,
OPENFX, KEEL, MANTECA, COBRE
```

---

## 11. Webhook Integration

### 11.1 Infinia Webhook Architecture

Infinia supports two mechanisms for receiving payout status updates:

1. **Per-payout `callbackUrl`** — Set on each payout creation request. Infinia POSTs to this URL when the payout status changes. **This is the recommended approach.**
2. **Global webhook subscriptions** — Configured in the Infinia dashboard for event types like `payout`, `payment`, `movement`.

### 11.2 Webhook Security

Every webhook request includes:

| Header | Purpose |
|---|---|
| `X-Infinia-Signature` | HMAC-SHA256 signature (base64-encoded) of the raw body, keyed with the shared secret |
| `X-Idempotency-Key` | UUID for deduplication (store for 24h to prevent reprocessing) |
| `event` | Event type: `payout`, `payment`, or `movement` |

**Signature Verification (JavaScript):**

```javascript
const crypto = require('crypto');

function isValidSignature(rawBody, signatureHeader, clientId) {
  const computedHmac = crypto
    .createHmac('sha256', clientId)
    .update(rawBody, 'utf8')
    .digest();

  let providedHmac;
  try {
    providedHmac = Buffer.from(signatureHeader, 'base64');
  } catch {
    return false;
  }

  if (computedHmac.length !== providedHmac.length) {
    return false;
  }

  return crypto.timingSafeEqual(computedHmac, providedHmac);
}
```

### 11.3 Retry Policy

| Attempt | Timing | Notes |
|---|---|---|
| 1st delivery | Immediate | On status change |
| 1st retry | +15 minutes | If no 2xx response |
| 2nd retry | +30 minutes | From original attempt |
| 3rd retry | +45 minutes | From original attempt |
| 4th retry (final) | +60 minutes | Marked `ERROR` if fails |

After all retries fail, the webhook is marked as `ERROR`. Re-send is possible via the Infinia dashboard.

### 11.4 Webhook Source IPs (for firewall allowlisting)

| Environment | IP Addresses |
|---|---|
| **Production** | `54.152.206.172/32`, `34.192.190.151/32` |
| **Sandbox** | `44.214.68.30/32`, `52.20.45.108/32` |

### 11.5 InfiniaWebhookGuard Implementation Pattern

Follows the exact same pattern as `BridgeWebhookGuard` (`src/modules/bridge/guards/bridge-webhook.guard.ts`) but uses HMAC-SHA256 instead of RSA-SHA256:

- Reads `INFINIA_WEBHOOK_SECRET` from `ConfigService`
- If not configured, logs warning and allows requests (development mode)
- Extracts `X-Infinia-Signature` header
- Computes HMAC-SHA256 of raw body using the webhook secret
- Compares using `crypto.timingSafeEqual()` (timing-safe)
- Stores processed `X-Idempotency-Key` values to prevent reprocessing

---

## 12. Sandbox Testing Guide

### 12.1 Test Amounts

Infinia's sandbox uses specific amounts to simulate outcomes:

| Amount | Resulting Status |
|---|---|
| `99997` or `99996` | `COMPLETED` |
| `99999` or `99998` | `ERROR` |
| Any other amount | `IN_PROGRESS` (stays in progress indefinitely in sandbox) |

### 12.2 Sandbox Base URL

```
https://app2test.infiniaweb.com/infinia_api
```

### 12.3 Test Payloads by Country

All examples use `sourceAccountId: "-1"` (sandbox placeholder):

**Argentina (ALIAS):**
```json
{
  "sourceAccountId": "-1",
  "destinationAccount": {
    "country": "ARGENTINA",
    "currency": "ARS",
    "destinationType": { "type": "ALIAS", "alias": "BARCO.PEZ.CONTROL" }
  },
  "originId": "test-ar-001",
  "amount": 99997
}
```

**Brazil (Pix):**
```json
{
  "sourceAccountId": "-1",
  "destinationAccount": {
    "country": "BRAZIL",
    "currency": "BRL",
    "destinationType": { "type": "CHAVE PIX", "chavePix": "+5512575253111" }
  },
  "originId": "test-br-001",
  "amount": 99997
}
```

**Mexico (CLABE):**
```json
{
  "sourceAccountId": "-1",
  "destinationAccount": {
    "country": "MEXICO",
    "currency": "MXN",
    "destinationType": {
      "type": "CLABE",
      "clabe": "123456789012345678",
      "reference": "Test payout"
    }
  },
  "originId": "test-mx-001",
  "amount": 99997
}
```

**Europe (SEPA):**
```json
{
  "sourceAccountId": "-1",
  "destinationAccount": {
    "country": "EUROPE",
    "currency": "EUR",
    "reference": "Test Europe Payout",
    "destinationType": {
      "type": "ACCOUNT_EUROPE_SEPA",
      "iban": "ES912100041S1234567890",
      "beneficiaryType": "INDIVIDUAL",
      "beneficiaryFirstName": "John",
      "beneficiaryCountry": "ES"
    }
  },
  "originId": "test-eu-001",
  "amount": 99997
}
```

---

## 13. Open Questions & Decisions Required

### 13.1 Business Decisions

| # | Question | Impact | Options |
|---|---|---|---|
| 1 | **FX Conversion:** Does Infinia auto-convert USDC → fiat on payout execution, or do we need a separate step? | Determines whether the payout flow is 1-step or 2-step | A) Single-step (Infinia handles FX). B) Pre-fund fiat accounts. **Clarify with Infinia.** |
| 2 | **Source Account Funding:** How will Anzo's Infinia source account be funded with USDC/USDT? | Determines operational treasury flow | A) Manual top-up. B) Automated crypto transfer from Anzo treasury wallet. |
| 3 | **User Balance Verification:** Should we verify the user has sufficient crypto balance before initiating a payout? | Prevents failed payouts and bad UX | A) Check balance via OneBalance/Zerion first. B) Let Infinia reject if insufficient. |
| 4 | **Anzo Fee:** Should we charge a fee on Infinia payouts (like the existing `ANZO_FEE_PERCENTAGE` on swaps)? | Revenue consideration | A) Yes, percentage-based. B) Flat fee. C) No fee initially. |
| 5 | **Crypto Payouts:** Should we include `GLOBAL` (crypto-to-crypto) destination types in v1? | Scope decision | A) Fiat only in v1. B) Include crypto payouts. |
| 6 | **merchantType:** What value should we use? | Required for Infinia compliance | `DIGITAL_WALLET` or `CRYPTO_EXCHANGE` seem most appropriate. |

### 13.2 Technical Decisions

| # | Question | Impact | Recommendation |
|---|---|---|---|
| 7 | **Status tracking:** Use per-payout `callbackUrl` or polling? | Determines webhook vs cron architecture | **callbackUrl** (recommended by Infinia, real-time, simpler than polling) |
| 8 | **Fallback polling:** Implement periodic polling as backup for missed callbacks? | Reliability | Yes — implement a lightweight periodic job that checks `IN_PROGRESS` payouts older than 1 hour |
| 9 | **Idempotency key format:** UUID vs prefixed string? | Traceability | `anzo_payout_{uuidv4}` — prefix makes it easy to identify in Infinia dashboard |
| 10 | **Multi-company support:** Do we need `x-company-id` header support? | Multi-tenant capability | Not for v1. Can be added later if Anzo has sub-entities. |
| 11 | **Tie into existing `Transaction` model?** | Unified transaction history | Yes — create a `Transaction` record with `type: 'INFINIA_PAYOUT'` and `service: 'infinia'` alongside the `InfiniaPayout` record. This surfaces payouts in the user's unified transaction history. |

### 13.3 Infinia Onboarding Checklist

Before implementation can begin, the following must be obtained from Infinia:

- [ ] HTTPBasic API credentials (username + password) for sandbox
- [ ] HTTPBasic API credentials for production
- [ ] Source account ID (the funded account from which payouts are debited)
- [ ] Webhook shared secret for HMAC-SHA256 verification
- [ ] Confirmation of production base URL
- [ ] Confirmation of FX conversion behavior (auto vs. manual)
- [ ] Confirm whether `INTERNAL_TRANSFER` product requires a separate API call
- [ ] Rate limits (if any) on payout creation

---

## 14. Payee (External Account) Details — What the User Must Enter

### 14.1 Critical Architecture Difference vs. Bridge.xyz

**Infinia does NOT have a "Create External Account" API endpoint.** Unlike Bridge.xyz (which has `POST /customers/{id}/external_accounts` to register bank accounts), Infinia expects destination account details to be passed **inline** with every payout request in the `destinationAccount` field.

This means:
- **Anzo stores payee details locally** in the `InfiniaPayee` Prisma model
- **On each payout**, the backend reconstructs the `destinationAccount` payload from the saved `InfiniaPayee` record
- **Optionally**, before saving a payee, Anzo can call Infinia's **Bank Account Validation API** to verify the account is valid

> **Citations:**
> - Payout creation: [v2_create_payout__post](https://docs.infiniaweb.com/reference/v2_create_payout__post) — `destinationAccount` is inline, not a reference to a pre-saved entity
> - Local bank validation: [v1_create_account_identity__post](https://docs.infiniaweb.com/reference/v1_create_account_identity__post) — Validates AR (CBU/ALIAS/CVU), BR (Pix/Account), MX (CLABE), CO (Bre-B), GB (Sort Code), US (Routing+Account), UY, and more
> - International bank validation: [v1_create_bank_account_validation_global_global__post](https://docs.infiniaweb.com/reference/v1_create_bank_account_validation_global_global__post) — Validates via SWIFT code or IBAN

### 14.2 Bank Account Validation API (Optional Pre-Save Step)

Before saving a payee, Anzo can validate the account exists using:

| Endpoint | Method | URL |
|---|---|---|
| Local Validation | `POST` | `/v1/bank-account-validation/` |
| International Validation | `POST` | `/v1/bank-account-validation/global/` |

**How it works:**
1. Send account details → receive a `validation_id`
2. Poll the result (may take a few seconds)
3. Response includes owner name, document type/number, account type, currency — can be used to confirm before saving

**Supported for local validation:** AR (CBU/ALIAS/CVU), BR (Pix Key/Account+Branch), MX (CLABE), CO (Bre-B Key), GB (Sort Code+Account), US (Routing+Account), CN, IN, ID, KR, NG, NP, PK, UY, VN

### 14.3 Complete Per-Country Field Requirements

Every field marked **REQUIRED** must be collected from the user. Fields marked *optional* can enhance the payout but are not mandatory.

---

#### Argentina (`ARGENTINA`)

**Currencies:** ARS, USD

**Option A: ALIAS** (most common — simple text alias)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `alias` | string | **REQUIRED** | Bank alias (word-based) | `BARCO.PEZ.CONTROL` |

**Option B: CBU** (22-digit bank identifier)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `cbu` | string | **REQUIRED** | CBU number (22 digits) | `0170032450000643578149` |

**Option C: QR_CODE** (QR code data)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `qrCode` | string | **REQUIRED** | QR code data containing transaction details | `(QR data string)` |

**Special fields at country level:**
| Field | Type | Required | Description |
|---|---|---|---|
| `adultCheck` | boolean | *optional* | If true, payout only executes if beneficiary is 18+ |

**Validation available:** Yes — CBU, ALIAS, CVU via local validation API

---

#### Bolivia (`BOLIVIA`)

**Currencies:** BOB, USD

**Only option: ACH**
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `accountNumber` | string | **REQUIRED** | Beneficiary's bank account number | `123456` |
| `name` | string | **REQUIRED** | Beneficiary's full name | `John Doe` |
| `documentNumber` | string | **REQUIRED** | Beneficiary's document number | `123456` |
| `reference` | string | **REQUIRED** | Reference for the payout | `Payment from Anzo` |
| `bankCode` | string | **REQUIRED** | Bank code ([full list](https://docs.infiniaweb.com/page/bo-bank-codes)) | `123` |
| `bankBranch` | string | **REQUIRED** | Branch code ([full list](https://docs.infiniaweb.com/page/bo-bank-branches)) | `001` |
| `clientId` | string | *conditional* | **Required only for Banco Ganadero** | `12345` |

---

#### Brazil (`BRAZIL`)

**Currencies:** BRL

**Option A: CHAVE PIX** (most common — Pix key)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `chavePix` | string | **REQUIRED** | Pix key (CPF/CNPJ, email, phone, or random key). Phone must include leading `+` in international format. | `+5512575253111` |
| `reference` | string | *optional* | Reference text (max 100 chars) | `Payment from Anzo` |
| `documentNumber` | string | *optional* | Owner CPF/CNPJ — triggers verification if provided | `12345678901` |

**Option B: ACCOUNT_BRAZIL** (TED bank transfer)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `ispb` | string | **REQUIRED** | ISPB code of the bank | `60701190` |
| `number` | string | **REQUIRED** | Creditor account number | `123456` |
| `issuer` | string | **REQUIRED** | Agency/branch of account | `0001` |
| `accountType` | string | **REQUIRED** | `TRAN`, `CACC`, `SVGS`, or `SLRY` | `CACC` |
| `documentNumber` | string | **REQUIRED** | Owner's CPF/CNPJ | `12345678901` |
| `name` | string | *optional* | Owner name (defaults to "NN" if not sent) | `Maria Silva` |

**Option C: BR_CODE** (BR Code / QR)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `brCode` | string | **REQUIRED** | BR code from an account | `(BR code string)` |
| `reference` | string | *optional* | Reference text (max 100 chars) | `Payment` |
| `documentNumber` | string | *optional* | Owner CPF/CNPJ for verification | `12345678901` |

**Validation available:** Yes — Pix Key, Account+Branch via local validation API

---

#### Chile (`CHILE`)

**Currencies:** CLP

**Only option: ACCOUNT_CHILE**
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `currency` | string | **REQUIRED** | Must be `CLP` | `CLP` |
| `bankCode` | string | **REQUIRED** | Bank code | `123` |
| `accountType` | string | **REQUIRED** | `CC`, `SVGS`, `VISTA`, `RUT`, or `SLRY` | `CC` |
| `accountNumber` | string | **REQUIRED** | Account number | `123456` |
| `documentType` | string | **REQUIRED** | `RUT`, `RUN`, `PAS`, or `CE` | `RUT` |
| `documentNumber` | string | **REQUIRED** | Document number | `123456` |
| `name` | string | **REQUIRED** | Account owner's name | `John Doe` |

---

#### Colombia (`COLOMBIA`)

**Currencies:** COP

**Option A: ACCOUNT_COLOMBIA** (ACH transfer)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `currency` | string | **REQUIRED** | Must be `COP` | `COP` |
| `fullName` | string | **REQUIRED** | Full legal name of account holder | `Juan Perez` |
| `documentType` | string | **REQUIRED** | `CC`, `NIT`, `CE`, `PA`, `PPT`, `TI`, `RC`, `TE`, `DIE`, or `ND` | `CC` |
| `documentNumber` | string | **REQUIRED** | ID number matching document type | `123456789` |
| `bankCode` | integer | **REQUIRED** | Bank code ([full list](https://docs.infiniaweb.com/page/co-bank-codes)) | `1007` |
| `accountType` | string | **REQUIRED** | `CHECKING`, `SAVINGS`, or `ELECTRONIC_DEPOSIT` | `SAVINGS` |
| `accountNumber` | string | **REQUIRED** | Account number | `123456789` |
| `email` | string | *optional* | Account holder's email | `juan@email.com` |
| `phoneNumber` | string | *optional* | Phone in international format | `+573001234567` |
| `reference` | string | *optional* | Transaction reference | `Payout` |

**Option B: BREB_KEY** (Bre-B instant transfer)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `currency` | string | **REQUIRED** | Must be `COP` | `COP` |
| `brebKey` | string | **REQUIRED** | Bre-B key of recipient | `3001234567` |
| `reference` | string | *optional* | Transaction reference | `Payment` |

**Option C: COBRE_BALANCE** (Cobre reconciliation)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `currency` | string | **REQUIRED** | Must be `COP` | `COP` |
| `externalAccount` | string | **REQUIRED** | External account identifier in Cobre | `ext_123` |

**Validation available:** Yes — Bre-B Key via local validation API

---

#### Europe (`EUROPE`)

**Currencies:** EUR, GBP  
**Note:** `reference` is **REQUIRED** at the country level (not inside destinationType).

**Option A: ACCOUNT_EUROPE_SEPA** (SEPA transfer)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `iban` | string | **REQUIRED** | IBAN of beneficiary | `ES912100041S1234567890` |
| `beneficiaryType` | string | **REQUIRED** | `INDIVIDUAL` or `BUSINESS` | `INDIVIDUAL` |
| `beneficiaryFirstName` | string | **REQUIRED** | First name | `John` |
| `beneficiaryLastName` | string | *optional* | Last name (not required if BUSINESS) | `Doe` |
| `beneficiaryCountry` | string | **REQUIRED** | ISO 3166-1 alpha-2 country code | `ES` |
| `instant` | boolean | *optional* | Use SEPA Instant (default: `true`) | `true` |
| `proxyAccount` | object | *optional* | Proxy account to prevent name-mismatch failures | See §14.4 |

**Option B: ACCOUNT_UNITED_KINGDOM_CHAPS_FPS** (UK transfer)
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `accountNumber` | string | *conditional* | Required if IBAN not provided | `12345678` |
| `sortCode` | string | *conditional* | Required if IBAN not provided | `123456` |
| `iban` | string | *conditional* | Required if accountNumber/sortCode not provided | `GB29NWBK60161331926819` |
| `beneficiaryType` | string | **REQUIRED** | `INDIVIDUAL` or `BUSINESS` | `INDIVIDUAL` |
| `beneficiaryFirstName` | string | **REQUIRED** | First name | `Jane` |
| `beneficiaryLastName` | string | *optional* | Last name (not required if BUSINESS) | `Smith` |
| `beneficiaryCountry` | string | **REQUIRED** | Must be `GB` | `GB` |
| `beneficiaryCity` | string | *conditional* | Required for CHAPS (non-instant) | `London` |
| `beneficiaryPostalCode` | string | *conditional* | Required for CHAPS (non-instant) | `SW1A 1AA` |
| `instant` | boolean | *optional* | Use FPS Instant (default: `true`) | `true` |
| `proxyAccount` | object | *optional* | Proxy account data | See §14.4 |

---

#### Mexico (`MEXICO`)

**Currencies:** MXN, USD

**Only option: CLABE**
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `clabe` | string | **REQUIRED** | 18-digit CLABE number | `123456789012345678` |
| `reference` | string | **REQUIRED** | Payment reference | `Payment from Anzo` |
| `latitude` | string | *optional* | Geo location (max 30 chars) | `19.4326` |
| `longitude` | string | *optional* | Geo location (max 30 chars) | `-99.1332` |

**Validation available:** Yes — CLABE via local validation API

---

#### Paraguay (`PARAGUAY`)

**Currencies:** PYG

**Only option: ACCOUNT_PARAGUAY**
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `bankCode` | integer | **REQUIRED** | Bank code ([full list](https://docs.infiniaweb.com/page/py-bank-codes)) | `123` |
| `name` | string | **REQUIRED** | Beneficiary account name | `John Doe` |
| `accountNumber` | string | **REQUIRED** | Account number | `123456` |
| `documentType` | string | **REQUIRED** | `CI`, `PAS`, `CRP`, `CRC`, `RUC`, or `DNI` | `CI` |
| `documentNumber` | string | **REQUIRED** | Document number. RUC requires check digit (e.g., `80221430-7`), CI does not. | `6532142` |

---

#### Peru (`PERU`)

**Currencies:** PEN

**Only option: ACCOUNT_PERU**
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `currency` | string | **REQUIRED** | Must be `PEN` | `PEN` |
| `bankCode` | string | **REQUIRED** | Bank code | `123` |
| `accountType` | string | **REQUIRED** | `CC`, `SVGS`, or `EFE` | `CC` |
| `accountNumber` | string | *optional* | Account number | `123456` |
| `cciNumber` | string | *optional* | Interbank account code (CCI) | `1234567` |
| `documentType` | string | **REQUIRED** | `RUC`, `DNI`, `PAS`, or `CE` | `DNI` |
| `documentNumber` | string | **REQUIRED** | Document number | `123456` |
| `name` | string | **REQUIRED** | Account owner name | `John Doe` |
| `state` | string | **REQUIRED** | State/region | `LIMA` |
| `email` | string | *optional* | Email address | `john@email.com` |

---

#### Global — Crypto (`GLOBAL`) — Lower Priority

**Currencies:** USDT, USDC, EURC

**Options:** `ETHEREUM`, `TRON`, `POLYGON`, `BASE`
| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `address` | string | **REQUIRED** | Wallet address (ERC-20/TRC-20 format) | `0x1234...5678` |
| `acceptsRetries` | boolean | *optional* | Allow automatic retries on failure (default: `false`) | `false` |

---

### 14.4 Proxy Account (Europe/UK Only)

When set on SEPA or CHAPS/FPS payouts, Infinia opens a proxy account in the account holder's name to prevent transaction failures caused by name mismatches between the sender and beneficiary.

| Field | Type | Required | Description |
|---|---|---|---|
| `beneficiaryAddressLine` | string | **REQUIRED** | Street address |
| `beneficiaryCity` | string | **REQUIRED** | City |
| `beneficiaryCountry` | string | **REQUIRED** | ISO alpha-2 country code |
| `beneficiaryBirthdate` | string | *optional* | YYYY-MM-DD (not required for corporate) |
| `beneficiaryEmail` | string | **REQUIRED** | Email address |
| `businessType` | string | *conditional* | Required if beneficiary is a company: `LIMITED_LIABILITY`, `SOLE_TRADER`, `PARTNERSHIP`, `PUBLIC_LIMITED_COMPANY`, `JOINT_STOCK_COMPANY`, `CHARITY` |
| `businessRegistrationNumber` | string | *conditional* | Required if business |
| `businessContactName` | string | *conditional* | Required if business |
| `businessPhoneNumber` | string | *conditional* | Required if business |
| `businessAddressLine` | string | *optional* | Trading address |
| `businessCity` | string | *optional* | Trading city |
| `businessCountry` | string | *optional* | Trading country (ISO alpha-2) |

> **Citation:** Proxy Account schema from [v2_create_payout__post](https://docs.infiniaweb.com/reference/v2_create_payout__post) → `ProxyAccountData` component

### 14.5 Summary: Minimum Fields Per Country (Simplest Path)

This table shows the **simplest destination type** for each country and the minimum fields the user must fill in:

| Country | Simplest Type | Min Fields | What User Enters |
|---|---|---|---|
| Argentina | `ALIAS` | 1 | Alias text |
| Bolivia | `ACH` | 6 | Account number, name, document, reference, bank code, branch |
| Brazil | `CHAVE PIX` | 1 | Pix key (phone/email/CPF/random) |
| Chile | `ACCOUNT_CHILE` | 6 | Bank code, account type, account number, doc type, doc number, name |
| Colombia | `BREB_KEY` | 1 | Bre-B key |
| Europe (EUR) | `ACCOUNT_EUROPE_SEPA` | 4 + reference | IBAN, beneficiary type, first name, country + reference at country level |
| Europe (GBP) | `ACCOUNT_UNITED_KINGDOM_CHAPS_FPS` | 5 + reference | Sort code, account number, beneficiary type, first name, country + reference |
| Mexico | `CLABE` | 2 | CLABE number, reference |
| Paraguay | `ACCOUNT_PARAGUAY` | 5 | Bank code, name, account number, doc type, doc number |
| Peru | `ACCOUNT_PERU` | 7 | Bank code, account type, doc type, doc number, name, state, currency |

### 14.6 Account Owner API (Company-Level KYC)

Infinia also has an Account Owner API for company-level KYC (not per-user). This is relevant for Anzo's company onboarding with Infinia, not for individual end-users.

**Individual Owner Required Fields:**
| Field | Required | Description |
|---|---|---|
| `first_name` | Yes | First name |
| `last_name` | Yes | Last name |
| `date_of_birth` | Yes | YYYY-MM-DD |
| `tax_id` | Yes | Tax identification number |
| `tax_id_country` | Yes | ISO alpha-2 |
| `email` | Yes | Email address |
| `phone_number` | Yes | With country code (e.g., +5491133334444) |
| `address` | Yes | Object: line_1, city, state, postal_code, country |

**Optional KYC Documents:** `identity_document_id`, `selfie_document_id`, `proof_of_address_document_id`

> **Citation:** Account Owner schema from [v1_5_update_account_owner](https://docs.infiniaweb.com/reference/v1_5_update_account_owner) → `IndividualDataSchema`

---

## Citations & References

| Reference | Source | URL |
|---|---|---|
| Create Payout (OpenAPI 3.1.0) | Infinia Docs | https://docs.infiniaweb.com/reference/v2_create_payout__post |
| List Payouts (OpenAPI 3.1.0) | Infinia Docs | https://docs.infiniaweb.com/reference/v2_list_payouts__get |
| Get Payout Details (OpenAPI 3.1.0) | Infinia Docs | https://docs.infiniaweb.com/reference/v2_get_payout__payout_id___get |
| Get Account Details (OpenAPI 3.1.0) | Infinia Docs | https://docs.infiniaweb.com/reference/v1_3_get_account |
| Webhooks Documentation | Infinia Docs | https://docs.infiniaweb.com/reference/webhooks |
| Anzo Architecture Rules | Internal | `rules.md` |
| Bridge Integration (pattern reference) | Internal | `src/modules/bridge/` |
| Existing Prisma Schema | Internal | `prisma/schema.prisma` |
| Configuration & Validation | Internal | `src/config/configuration.ts`, `src/config/validation.schema.ts` |
