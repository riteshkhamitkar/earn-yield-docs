# Complete Architecture — Anzo + Seismic Orchestration Integration

> **Scope:** End-to-end architecture for fiat on-ramp, off-ramp, KYC, and account management using Seismic Orchestration  
> **Audience:** Backend engineering, frontend, product, and DevOps teams  
> **Date:** June 11, 2026  
> **Status:** Architecture ready for implementation  
> **Verification:** All API behaviors confirmed against live Seismic docs + Lyron (Seismic team)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [New User Onboarding Flow](#3-new-user-onboarding-flow)
4. [Existing User Migration Flow](#4-existing-user-migration-flow)
5. [Pay-In (On-Ramp) Architecture](#5-pay-in-on-ramp-architecture)
6. [Pay-Out (Off-Ramp) Architecture](#6-pay-out-off-ramp-architecture)
7. [Account Management](#7-account-management)
8. [Webhook Processing](#8-webhook-processing)
9. [Database Schema](#9-database-schema)
10. [API Surface](#10-api-surface)
11. [Environment Configuration](#11-environment-configuration)
12. [Error Handling & Resilience](#12-error-handling--resilience)
13. [Security Considerations](#13-security-considerations)
14. [Implementation Phases](#14-implementation-phases)

---

## 1. Executive Summary

Anzo is migrating its fiat infrastructure from Bridge.xyz + Sumsub to **Seismic Orchestration** — a unified platform handling KYC, virtual accounts, quotes, and transfers.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| KYC approach | **Retain Sumsub + inline profile** (Path B) | Mobile SDK already integrated; less user friction |
| Existing user migration | **API-driven profile submission** | Confirmed by Seismic team; no re-verification needed |
| Virtual accounts | **Native Seismic API** (not Bridge compat) | Bridge compat returns 501 for VAs |
| Crypto destination | **Turnkey external account** (non-custodial) | Preserves Anzo's non-custodial model |
| Quotes | **Seismic native quotes** with `named: "require"` | Best price discovery; required for named VAs |
| Idempotency | **Mandatory on all mutating calls** | Prevents duplicate accounts on webhook retries |

### Architecture Principles

1. **Provider abstraction** — `FiatRailsProvider` interface with `BridgeFiatRailsProvider` and `SeismicFiatRailsProvider`
2. **Preserve non-custodial wallets** — USDC lands on Turnkey `ethereumAddress`
3. **Parity with current VA set** — USD + EUR fiat virtual per approved user
4. **Feature flags** — Dual-run Bridge + Seismic during migration
5. **Idempotency everywhere** — All mutating calls use deterministic keys

---

## 2. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Anzo Mobile App                                 │
│  • Sumsub SDK (ID + liveness)                                            │
│  • Questionnaire form                                                    │
│  • VA deposit instructions screen                                       │
│  • Payout quote → prepare → execute                                      │
│  • Transaction history / activity feed                                   │
└──────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                        Anzo Backend (NestJS)                              │
│                                                                           │
│  ┌─────────────┐   ┌──────────────────┐   ┌─────────────────────────┐  │
│  │  KycModule  │   │  SeismicModule   │   │  BridgeModule (legacy)  │  │
│  │  (Sumsub)   │──▶│  • SeismicProvider│   │  (feature-flagged)      │  │
│  │  • Session  │   │  • SeismicService │   │                         │  │
│  │  • Webhook  │   │  • AccountHandler │   │                         │  │
│  │  • Profile  │   │  • QuoteHandler   │   │                         │  │
│  └─────────────┘   │  • PayoutHandler  │   │                         │  │
│                     │  • WebhookHandler │   │                         │  │
│  ┌─────────────┐   └────────┬─────────┘   └─────────────────────────┘  │
│  │ RelayModule │            │                                           │
│  │ (USDC txns) │◀───────────┘                                           │
│  └─────────────┘                                                        │
│  ┌─────────────┐                                                        │
│  │ PaytrieMod. │  (Canada — unchanged)                                    │
│  └─────────────┘                                                        │
└──────────────────────────────────────────────────────────────────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
              ▼                      ▼                      ▼
       ┌──────────┐          ┌──────────────┐       ┌──────────────┐
       │  Sumsub   │          │   Seismic    │       │   Bridge     │
       │  (KYC)    │          │Orchestration │       │  (legacy)    │
       └──────────┘          └──────────────┘       └──────────────┘
```

### Module Responsibilities

| Module | Responsibility | New/Modified |
|--------|---------------|-------------|
| `SeismicModule` | All Seismic API interactions | **NEW** |
| `KycModule` | Sumsub SDK session + webhook | Modified (add Seismic profile sync) |
| `RelayModule` | USDC transfers (payout step 1) | Unchanged |
| `BridgeModule` | Legacy Bridge flows | Deprecated (FF-controlled) |
| `PaytrieModule` | Canada CAD | Unchanged |

---

## 3. New User Onboarding Flow

### 3.1 High-Level Flow

```
User Registration (Anzo)
    │
    ▼
User taps "Add Money via Bank"
    │
    ▼
┌─────────────────────────────┐
│  Step 1: Sumsub ID + Selfie │
│  (existing mobile SDK)      │
└─────────────────────────────┘
    │
    ├── Sumsub webhook: applicantReviewed (GREEN)
    │
    ▼
┌─────────────────────────────┐
│  Step 2: Questionnaire      │
│  (existing Anzo form)       │
│  • Address, occupation      │
│  • Source of funds          │
│  • Employment status        │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│  Step 3: Create Seismic     │
│  Customer + Submit Profile  │
│  (NEW - backend)            │
└─────────────────────────────┘
    │
    ├── POST /v0/customers
    ├── PUT /v0/customers/{id}/kyc/profile
    ├── POST /v0/customers/{id}/kyc/submit
    │
    ▼
┌─────────────────────────────┐
│  Step 4: Seismic Review     │
│  (< 5 min automated)        │
└─────────────────────────────┘
    │
    ├── Webhook: customer.kyc.status_changed (approved)
    │
    ▼
┌─────────────────────────────┐
│  Step 5: Provision Accounts │
│  (async)                    │
│  • USD fiat virtual         │
│  • EUR fiat virtual         │
│  • Turnkey crypto external  │
└─────────────────────────────┘
    │
    ├── Webhook: account.created (×3)
    │
    ▼
┌─────────────────────────────┐
│  Step 6: User sees deposit  │
│  instructions (USD + EUR)   │
└─────────────────────────────┘
```

### 3.2 Detailed API Flow

#### Step 1: Sumsub KYC (Unchanged)

```
Mobile → POST /api/v1/kyc/sumsub/session
  → Returns Sumsub SDK access token
  → User completes ID scan + selfie

Sumsub → POST /api/v1/kyc/webhook/sumsub
  → applicantReviewed with reviewAnswer = GREEN
  → Anzo: update KycVerification, trigger partner submission
```

#### Step 2: Questionnaire (Unchanged)

```
Mobile → POST /api/v1/kyc/questionnaire
  → Saves to KycProfile table
  • firstName, lastName, dateOfBirth, nationality
  • streetLine1, city, state, postalCode, country
  • occupation, employmentStatus, sourceOfFunds
  • expectedMonthlyVolume, politicallyExposedPerson
```

#### Step 3: Create Seismic Customer (NEW)

```typescript
// Backend handler (NEW)
async function onboardToSeismic(userId: string) {
  const user = await prisma.user.findUnique({
    where: { id: userId },
    include: { kycProfile: true, wallets: true },
  });

  // 1. Create customer on Seismic
  const { id: seismicCustomerId, kyc_link } = await seismicApi.customers.create({
    type: 'individual',
    profile: {
      contact: { email: user.email },
    },
  });

  // 2. Persist SeismicCustomer
  await prisma.seismicCustomer.create({
    data: {
      userId,
      seismicCustomerId,
      kycStatus: 'not_started',
      lifecycleStatus: 'PENDING_KYC',
      kycLinkUrl: kyc_link.url,
      kycLinkToken: kyc_link.token,
    },
  });

  // 3. Upload ID documents from Sumsub to Seismic
  const documentIds = await uploadDocumentsFromSumsub(
    user.kycProfile!.sumsubApplicantId!,
    seismicCustomerId
  );

  // 4. Build complete KYC profile
  const kycProfile = buildSeismicKycProfile(user, documentIds);

  // 5. Submit profile
  await seismicApi.customers.updateKycProfile(seismicCustomerId, kycProfile);

  // 6. Submit for verification
  const { state } = await seismicApi.customers.submitKyc(seismicCustomerId);

  // 7. Update status
  await prisma.seismicCustomer.update({
    where: { seismicCustomerId },
    data: { kycStatus: state.kyc_status },
  });

  return { seismicCustomerId, status: state.kyc_status };
}
```

#### Step 4-5: Webhook Processing

```typescript
// When customer.kyc.status_changed → approved
async function onKycApproved(seismicCustomerId: string) {
  await prisma.seismicCustomer.update({
    where: { seismicCustomerId },
    data: { kycStatus: 'approved', lifecycleStatus: 'PROVISIONING_ACCOUNTS' },
  });

  // Provision all accounts in parallel
  const user = await getUserBySeismicCustomerId(seismicCustomerId);

  await Promise.all([
    // USD VA
    seismicApi.accounts.createFiatVirtual(seismicCustomerId, {
      currency: 'USD', rails: ['ach', 'wire'], schema: 'us_bank',
    }),
    // EUR VA
    seismicApi.accounts.createFiatVirtual(seismicCustomerId, {
      currency: 'EUR', rails: ['sepa'], schema: 'iban',
    }),
    // Turnkey crypto external
    seismicApi.accounts.createCryptoExternal(seismicCustomerId, {
      currency: 'USDC', rails: ['base'],
      address: user.wallets[0].ethereumAddress,
    }),
  ]);
}

// When account.created → check if all accounts ready
async function onAccountCreated(accountData: any) {
  await upsertSeismicAccount(accountData);

  const customer = await prisma.seismicCustomer.findUnique({
    where: { seismicCustomerId: accountData.customer_id },
    include: { accounts: true },
  });

  // Check all required accounts exist and are active
  const hasUsd = customer.accounts.some(
    a => a.currency === 'USD' && a.status === 'active'
  );
  const hasEur = customer.accounts.some(
    a => a.currency === 'EUR' && a.status === 'active'
  );
  const hasCrypto = customer.accounts.some(
    a => a.currency === 'USDC' && a.status === 'active'
  );

  if (hasUsd && hasEur && hasCrypto) {
    await prisma.seismicCustomer.update({
      where: { id: customer.id },
      data: { lifecycleStatus: 'ACTIVE' },
    });

    // Push notification + activity event
    await notifyUser(customer.userId, 'BANK_ACCOUNTS_READY');
  }
}
```

---

## 4. Existing User Migration Flow

See `04-existing-user-migration.md` for the complete migration flow.

**Summary:**
1. Fetch all data from Sumsub (applicant data + document images)
2. Create Seismic customer
3. Upload documents to Seismic
4. Submit complete KYC profile via `PUT /customers/{id}/kyc/profile`
5. Seismic reviews (< 5 min)
6. On approval → provision accounts (same as new user flow)

---

## 5. Pay-In (On-Ramp) Architecture

### 5.1 On-Ramp Flow: Fiat → Crypto

```
User has USD/EUR VA deposit instructions
    │
    ├── User sends fiat to VA bank account (ACH/Wire/SEPA)
    │
    ▼
Seismic receives fiat
    │
    ├── Seismic processes: fiat → USDC
    │
    ▼
USDC sent to user's Turnkey address on Base
    │
    ├── order.status_changed → completed
    │
    ▼
User sees USDC in Anzo wallet
```

### 5.2 Deposit Instructions API

```
GET /api/v1/seismic/virtual-accounts

Response:
{
  "accounts": [
    {
      "currency": "USD",
      "status": "active",
      "depositInstructions": {
        "bankName": "...",
        "routingNumber": "...",
        "accountNumber": "...",
        "beneficiaryName": "...",
        "paymentRails": ["ach", "wire"]
      },
      "destination": {
        "currency": "USDC",
        "network": "base",
        "address": "0x..."
      }
    },
    {
      "currency": "EUR",
      "status": "active",
      "depositInstructions": {
        "bankName": "...",
        "iban": "...",
        "bic": "...",
        "beneficiaryName": "...",
        "paymentRails": ["sepa"]
      },
      "destination": {
        "currency": "USDC",
        "network": "base",
        "address": "0x..."
      }
    }
  ]
}
```

### 5.3 Quote Creation (On-Ramp)

**Critical:** Must include `named: "require"` for on-ramp quotes.

```typescript
async function createOnrampQuote(
  seismicCustomerId: string,
  amount: string
) {
  return seismicApi.quotes.create({
    end_customer_id: seismicCustomerId,
    end_customer_type: 'individual',
    amount,
    named: 'require',  // ← REQUIRED for named VAs
    source: { currency: 'USD' },  // Fan out across all USD rails
    destination: {
      currency: 'USDC',
      rails: ['base'],
    },
  }, { wait: true });
}
```

### 5.4 Deposit Notification

When `order.status_changed` → `completed`:

```typescript
async function onDepositCompleted(orderData: any) {
  // Create activity event
  await prisma.seismicActivityEvent.create({
    data: {
      userId: orderData.customer_id,
      type: 'SEISMIC_DEPOSIT_IN',
      title: 'Deposit Received',
      subtitle: `${orderData.source.amount} ${orderData.source.currency} → ${orderData.destination.amount} USDC`,
      sourceId: orderData.order_id,
      metadata: orderData,
    },
  });

  // Push notification
  await pushNotify(userId, {
    title: 'Deposit Complete',
    body: `Your ${orderData.source.amount} ${orderData.source.currency} deposit has been converted to USDC.`,
  });
}
```

---

## 6. Pay-Out (Off-Ramp) Architecture

### 6.1 Off-Ramp Flow: Crypto → Fiat

```
User requests payout (e.g., $500 → EUR bank)
    │
    ▼
Create quote (crypto → fiat)
    │
    ├── POST /v0/quotes
    ├── source: USDC on Base (Turnkey external account)
    ├── destination: EUR via SEPA (user's external bank)
    └── Returns ranked quotes
    │
    ▼
User selects / confirms quote
    │
    ▼
Accept quote → create order
    │
    ├── POST /v0/quotes/{id}/accept-best
    └── Returns order with funding_instructions
    │
    ▼
User signs Relay transaction
    │
    ├── USDC sent from Turnkey → Seismic deposit address
    │
    ▼
Webhook: order.status_changed → completed
    │
    ├── Fiat sent to user's bank account
    │
    ▼
User receives EUR in bank
```

### 6.2 Quote Creation (Off-Ramp)

```typescript
async function createOfframpQuote(
  seismicCustomerId: string,
  amount: string,
  sourceAccountId: string,    // Turnkey crypto external account
  destinationAccountId: string // User's fiat external account
) {
  return seismicApi.quotes.create({
    end_customer_id: seismicCustomerId,
    end_customer_type: 'individual',
    amount,
    source: {
      account: sourceAccountId,
      rails: ['base'],
    },
    destination: {
      account: destinationAccountId,
      rails: ['sepa'],
    },
  }, { wait: true });
}
```

### 6.3 Payout Handler

```typescript
async function executePayout(
  userId: string,
  quoteSnapshotId: string
) {
  // 1. Accept best quote
  const order = await seismicApi.quotes.acceptBest(quoteSnapshotId);

  // 2. Store order
  await prisma.seismicOrder.create({
    data: {
      userId,
      seismicOrderId: order.id,
      status: order.status,
      type: 'OFFRAMP',
      sourceCurrency: order.source.currency,
      sourceAmount: order.source.amount,
      sourceRail: order.source.rail,
      destinationCurrency: order.destination.currency,
      destinationAmount: order.destination.amount,
      destinationRail: order.destination.rail,
      fees: order.fees,
      rate: order.rate,
      fundingInstructions: order.funding_instructions,
    },
  });

  // 3. If funding required (on-chain transfer)
  if (order.funding_instructions) {
    const relayQuote = await relayProvider.getQuote({
      from: userWalletAddress,
      to: order.funding_instructions.deposit_address,
      amount: order.funding_instructions.amount,
      token: 'USDC',
      chain: 'base',
    });

    return {
      orderId: order.id,
      relaySteps: relayQuote.steps,
    };
  }

  return { orderId: order.id };
}
```

### 6.4 Payout Status Tracking

```
GET /api/v1/seismic/payouts/{orderId}/status

Response:
{
  "orderId": "ord_...",
  "status": "processing",
  "source": { "currency": "USDC", "amount": "500" },
  "destination": { "currency": "EUR", "amount": "460" },
  "rate": "0.92",
  "fees": { "total": "3.00", "currency": "USD" },
  "estimatedCompletion": "2026-01-16T10:00:00Z"
}
```

### 6.5 Order Status Lifecycle

```
awaiting_funds → pending → processing → completed
     ↓              ↓           ↓
  canceled      failed      failed
```

| Status | Meaning |
|--------|---------|
| `awaiting_funds` | Waiting for crypto deposit |
| `pending` | Funds received, processing started |
| `processing` | Fiat payment in progress |
| `completed` | Fiat delivered to bank |
| `failed` | Order failed |
| `canceled` | Canceled by user or system |

---

## 7. Account Management

### 7.1 Virtual Account Provisioning (Auto)

Triggered on KYC approval. Creates 3 accounts:

| # | Type | Currency | Shape | Purpose |
|---|------|----------|-------|---------|
| 1 | Fiat virtual | USD | `us_bank` | Receive ACH/Wire deposits |
| 2 | Fiat virtual | EUR | `iban` | Receive SEPA deposits |
| 3 | Crypto external | USDC | `evm` | Receive USDC on Base (Turnkey) |

### 7.2 External Bank Accounts (User-Added)

```
POST /api/v1/seismic/external-accounts

Request:
{
  "currency": "EUR",
  "iban": "DE89370400440532013000",
  "bic": "COBADEFFXXX",
  "accountHolderName": "Alice Liddell"
}

→ POST Seismic /v0/customers/{id}/accounts/fiat/external
→ Store SeismicAccount with purpose: "payout_destination"
```

### 7.3 List External Accounts

```
GET /api/v1/seismic/external-accounts

Response:
{
  "accounts": [
    {
      "id": "acc_...",
      "currency": "EUR",
      "kind": "fiat",
      "shape": "iban",
      "details": {
        "iban": "DE89...",
        "bic": "COBADEFFXXX",
        "accountHolderName": "Alice Liddell"
      },
      "status": "active"
    }
  ]
}
```

### 7.4 User "Ready" Gating

```typescript
function isUserReady(seismicCustomer: SeismicCustomer & { accounts: SeismicAccount[] }): boolean {
  if (seismicCustomer.kycStatus !== 'approved') return false;

  const hasUsd = seismicCustomer.accounts.some(
    a => a.currency === 'USD' && a.kind === 'fiat' && a.status === 'active'
  );
  const hasEur = seismicCustomer.accounts.some(
    a => a.currency === 'EUR' && a.kind === 'fiat' && a.status === 'active'
  );
  const hasCrypto = seismicCustomer.accounts.some(
    a => a.currency === 'USDC' && a.kind === 'crypto' && a.status === 'active'
  );

  return hasUsd && hasEur && hasCrypto;
}
```

---

## 8. Webhook Processing

### 8.1 Unified Webhook Endpoint

```
POST /api/v1/webhooks/seismic
```

### 8.2 Event Handlers

```typescript
const handlers: Record<string, (data: any) => Promise<void>> = {
  'customer.kyc.status_changed': handleKycStatusChanged,
  'customer.kyb.status_changed': handleKybStatusChanged,
  'account.created': handleAccountCreated,
  'account.updated': handleAccountUpdated,
  'order.status_changed': handleOrderStatusChanged,
};
```

### 8.3 Handler Implementations

```typescript
async function handleKycStatusChanged(data: any) {
  const { customer_id, status, kyc_profile } = data;

  await prisma.seismicCustomer.update({
    where: { seismicCustomerId: customer_id },
    data: {
      kycStatus: status,
      profileSnapshot: kyc_profile || undefined,
    },
  });

  if (status === 'approved') {
    await onKycApproved(customer_id);
  } else if (status === 'rejected') {
    await notifyUser(customer_id, 'KYC_REJECTED');
  }
}

async function handleAccountCreated(data: any) {
  const { account_id, customer_id, account } = data;

  await prisma.seismicAccount.upsert({
    where: { seismicAccountId: account_id },
    create: {
      userId: await getUserIdBySeismicCustomerId(customer_id),
      seismicCustomerId: customer_id,
      seismicAccountId: account_id,
      kind: account.kind,
      shape: account.shape,
      currency: account.currency,
      rails: account.rails,
      purpose: inferPurpose(account),
      status: account.status,
      details: account.details,
    },
    update: {
      status: account.status,
      details: account.details,
    },
  });

  await tryActivateCustomer(customer_id);
}

async function handleOrderStatusChanged(data: any) {
  const { order_id, status, order } = data;

  await prisma.seismicOrder.updateMany({
    where: { seismicOrderId: order_id },
    data: { status },
  });

  if (status === 'completed') {
    await notifyOrderCompleted(order_id, order);
  } else if (status === 'failed') {
    await notifyOrderFailed(order_id, order);
  }
}
```

### 8.4 Webhook Requirements

- Return **200 within 15 seconds** (Svix hard limit)
- Process async (queue/job runner)
- Deduplicate by Svix message ID
- All handlers must be idempotent

---

## 9. Database Schema

### 9.1 New Models

```prisma
model SeismicCustomer {
  id                String   @id @default(cuid())
  userId            String   @unique
  user              User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  seismicCustomerId String   @unique
  customerType      String   @default("individual")
  kycStatus         String   @default("not_started")
  lifecycleStatus   String   @default("PENDING_KYC")
  // PENDING_KYC | KYC_SUBMITTED | PROVISIONING_ACCOUNTS | ACTIVE | KYC_REJECTED | MIGRATING

  kycLinkUrl        String?
  kycLinkToken      String?
  profileSnapshot   Json?
  referenceId       String?  @unique

  bridgeCustomerId  String?  @unique
  sumsubApplicantId String?
  migratedAt        DateTime?

  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt

  accounts          SeismicAccount[]
  accountRequests   SeismicAccountRequest[]
  orders            SeismicOrder[]

  @@index([seismicCustomerId])
  @@index([kycStatus])
  @@index([lifecycleStatus])
}

model SeismicAccount {
  id               String   @id @default(cuid())
  seismicCustomerId String
  seismicCustomer  SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)
  userId           String

  seismicAccountId String   @unique
  kind             String   // fiat | crypto
  shape            String?  // us | iban | evm | sol
  currency         String
  rails            String[]
  purpose          String   // onramp_deposit | wallet_destination | payout_destination
  status           String   // active | pending | closed
  details          Json?

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt

  @@unique([seismicCustomerId, currency, purpose])
  @@index([userId])
}

model SeismicAccountRequest {
  id                  String   @id @default(cuid())
  seismicCustomerId   String
  seismicCustomer     SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)

  creationRequestId   String   @unique
  idempotencyKey      String   @unique
  kind                String
  currency            String
  purpose             String
  status              String   @default("pending")
  seismicAccountId    String?

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt
}

model SeismicOrder {
  id               String   @id @default(cuid())
  userId           String
  seismicCustomerId String
  seismicCustomer  SeismicCustomer @relation(fields: [seismicCustomerId], references: [id], onDelete: Cascade)

  seismicOrderId   String   @unique
  quoteSnapshotId  String?
  type             String   // ONRAMP | OFFRAMP | SWAP
  status           String   // awaiting_funds | pending | processing | completed | failed | canceled

  sourceCurrency   String
  sourceAmount     String
  sourceRail       String?
  sourceAccountId  String?

  destinationCurrency String
  destinationAmount   String?
  destinationRail     String?
  destinationAccountId String?

  fees             Json?
  rate             String?
  fundingInstructions Json?

  createdAt        DateTime @default(now())
  updatedAt        DateTime @updatedAt
  completedAt      DateTime?

  @@index([userId])
  @@index([status])
}

model SeismicActivityEvent {
  id         String   @id @default(cuid())
  userId     String
  externalId String   @unique
  type       String   // SEISMIC_KYC_APPROVED, SEISMIC_DEPOSIT_IN, SEISMIC_PAYOUT_OUT
  status     String   @default("COMPLETED")
  title      String?
  subtitle   String?
  sourceId   String?
  metadata   Json?
  occurredAt DateTime @default(now())
  createdAt  DateTime @default(now())

  @@index([userId])
}
```

### 9.2 Modified Models

```prisma
model User {
  // ... existing fields ...
  seismicCustomer   SeismicCustomer?
}

model KycProfile {
  // ... existing fields ...
  seismicMigrationStatus      String?   // pending | in_progress | completed | failed
  seismicMigrationAttemptedAt DateTime?
  seismicMigrationError       String?
}
```

---

## 10. API Surface

### 10.1 New Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/v1/seismic/onboarding/start` | Yes | Create Seismic customer + return KYC link |
| `GET` | `/api/v1/seismic/onboarding/status` | Yes | KYC + lifecycle + account readiness |
| `GET` | `/api/v1/seismic/virtual-accounts` | Yes | USD/EUR deposit instructions |
| `POST` | `/api/v1/seismic/external-accounts` | Yes | Register payout bank |
| `GET` | `/api/v1/seismic/external-accounts` | Yes | List payout banks |
| `DELETE` | `/api/v1/seismic/external-accounts/:id` | Yes | Deactivate |
| `POST` | `/api/v1/seismic/quotes/onramp` | Yes | Quote fiat → crypto |
| `POST` | `/api/v1/seismic/quotes/offramp` | Yes | Quote crypto → fiat |
| `POST` | `/api/v1/seismic/quotes/:id/accept` | Yes | Accept quote |
| `GET` | `/api/v1/seismic/orders` | Yes | Transaction history |
| `GET` | `/api/v1/seismic/orders/:id` | Yes | Order details |
| `POST` | `/api/v1/seismic/migrate` | Yes | Migrate existing user |
| `POST` | `/api/v1/seismic/migrate/batch` | Admin | Batch migration |
| `POST` | `/api/v1/webhooks/seismic` | No (Svix) | Seismic webhook |

### 10.2 Legacy Endpoint Mapping

| Current Endpoint | New Seismic Endpoint |
|-----------------|---------------------|
| `GET /api/v1/bridge/virtual-account` | `GET /api/v1/seismic/virtual-accounts` |
| `POST /api/v1/bridge/payout/quote` | `POST /api/v1/seismic/quotes/offramp` |
| `POST /api/v1/bridge/payout/prepare` | `POST /api/v1/seismic/quotes/:id/accept` |
| `POST /api/v1/bridge/external-accounts` | `POST /api/v1/seismic/external-accounts` |

---

## 11. Environment Configuration

### 11.1 New Variables (add to `.env` + `validation.schema.ts`)

```bash
# Seismic Orchestration
SEISMIC_API_KEY=seismic_sandbox_...
SEISMIC_BASE_URL=https://orchestration-sandbox.seismictest.net/api
SEISMIC_WEBHOOK_SECRET=whsec_...
SEISMIC_ENV=sandbox

# Feature Flags
FF_SEISMIC_ONBOARDING=false       # New users → Seismic
FF_SEISMIC_PAYOUTS=false          # Payouts via Seismic
FF_SEISMIC_MIGRATION=false        # Existing user migration
FF_LEGACY_BRIDGE=true             # Keep Bridge for unmigrated users
```

### 11.2 Joi Validation

```typescript
SEISMIC_API_KEY: Joi.string().when('FF_SEISMIC_ONBOARDING', {
  is: 'true', then: Joi.required(), otherwise: Joi.optional(),
}),
SEISMIC_BASE_URL: Joi.string().uri().default(
  'https://orchestration-sandbox.seismictest.net/api',
),
SEISMIC_WEBHOOK_SECRET: Joi.string().optional(),
SEISMIC_ENV: Joi.string().valid('sandbox', 'production').default('sandbox'),
FF_SEISMIC_ONBOARDING: Joi.string().valid('true', 'false').default('false'),
FF_SEISMIC_PAYOUTS: Joi.string().valid('true', 'false').default('false'),
FF_SEISMIC_MIGRATION: Joi.string().valid('true', 'false').default('false'),
FF_LEGACY_BRIDGE: Joi.string().valid('true', 'false').default('true'),
```

---

## 12. Error Handling & Resilience

### 12.1 Idempotency Patterns

| Operation | Idempotency Key Pattern |
|-----------|------------------------|
| Create customer | `customer-{anzoUserId}` |
| USD VA | `va-usd-{seismicCustomerId}` |
| EUR VA | `va-eur-{seismicCustomerId}` |
| Turnkey external | `wallet-base-{seismicCustomerId}` |
| Fiat external | `ext-{seismicCustomerId}-{hash}` |
| On-ramp quote | `quote-onramp-{requestId}` |
| Off-ramp quote | `quote-offramp-{requestId}` |
| Accept quote | `accept-{snapshotId}` |

### 12.2 Retry Strategy

| Error Type | Retry | Strategy |
|-----------|-------|----------|
| 5xx | Yes | Exponential backoff, max 5 attempts |
| 401 | Yes | Refresh token (SDK handles auto) |
| 409 (idempotency conflict) | No | Different payload; fix and retry |
| 422 (validation) | No | Fix input |
| QuoteExpired | No | Re-create quote |
| AlreadyAccepted | No | Treat as success |

### 12.3 Gap-Fill for Missing Accounts

```typescript
// Scheduled job runs every 5 minutes
cron.schedule('*/5 * * * *', async () => {
  // Find users with approved KYC but stuck in PROVISIONING_ACCOUNTS
  const stuck = await prisma.seismicCustomer.findMany({
    where: {
      kycStatus: 'approved',
      lifecycleStatus: 'PROVISIONING_ACCOUNTS',
      updatedAt: { lt: new Date(Date.now() - 10 * 60 * 1000) }, // 10 min stale
    },
    include: { accounts: true },
  });

  for (const customer of stuck) {
    // Check which accounts are missing and re-provision
    await gapFillAccounts(customer);
  }
});
```

---

## 13. Security Considerations

### 13.1 API Key Security

- `SEISMIC_API_KEY` is **org-scoped** — never expose to client
- Store in environment variables, never commit to repo
- Rotate keys via Seismic dashboard if compromised

### 13.2 Webhook Security

- Verify Svix signatures using `SEISMIC_WEBHOOK_SECRET`
- Reject unverified webhooks with 400
- Process webhooks asynchronously (return 200 fast)

### 13.3 PII Handling

- Sumsub document images are **transient** — download, upload to Seismic, discard
- Never store raw ID images in Anzo DB
- Log only customer IDs, never PII

### 13.4 Idempotency for Safety

- All provisioning calls use deterministic idempotency keys
- Prevents duplicate VAs on webhook retries
- Enables safe re-runs of migration jobs

---

## 14. Implementation Phases

### Phase 0: Setup (Week 1)

| Task | Owner | Days |
|------|-------|------|
| Seismic org signup + MFA | Ops | 1 |
| Org KYB submission | Ops/Legal | 3 |
| Sandbox API key minting | Eng | 0.5 |
| Environment variables setup | Eng | 0.5 |

### Phase 1: Foundation (Week 1-2)

| Task | Days |
|------|------|
| Install `seismic-orchestration` SDK | 0.5 |
| Create `SeismicModule` + `SeismicProvider` | 2 |
| Implement webhook handler | 2 |
| Database schema migration | 1 |

### Phase 2: New User Onboarding (Week 2-3)

| Task | Days |
|------|------|
| Implement customer creation | 1 |
| Implement KYC profile submission | 2 |
| Implement document upload from Sumsub | 2 |
| Integrate into existing KYC flow | 2 |
| Test end-to-end | 2 |

### Phase 3: Account Provisioning (Week 3)

| Task | Days |
|------|------|
| Implement async VA provisioning | 2 |
| Implement `account.created` webhook handler | 1 |
| Implement `tryActivateCustomer()` | 1 |
| Test gap-fill + edge cases | 2 |

### Phase 4: Existing User Migration (Week 4)

| Task | Days |
|------|------|
| Implement migration service | 3 |
| Implement batch migration endpoint | 1 |
| Test with internal accounts | 2 |
| Pilot with 50 users | 2 |

### Phase 5: Pay-In / Pay-Out (Week 5-6)

| Task | Days |
|------|------|
| Implement on-ramp quotes | 2 |
| Implement off-ramp quotes | 2 |
| Implement quote acceptance + order tracking | 2 |
| Integrate with Relay for on-chain transfers | 2 |
| Test end-to-end payouts | 2 |

### Phase 6: Rollout (Week 7-8)

| Task | Days |
|------|------|
| Feature flag enablement (5% → 50% → 100%) | 5 |
| Monitor + fix issues | 3 |
| Deprecate Bridge for new users | 2 |

**Total: ~8 weeks**

---

*Document version: 1.0.0 | Last updated: June 11, 2026 | Architecture approved for implementation*
