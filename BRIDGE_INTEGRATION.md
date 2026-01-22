# Bridge.xyz Integration Guide

## Overview

This document provides a comprehensive guide for the Bridge.xyz integration in the Anzo backend. Bridge.xyz enables fiat on-ramp (USD → Crypto) and off-ramp (Crypto → USD) functionality.

### Key Features

- **On-Ramp**: User deposits USD to a personal virtual bank account → Bridge converts to USDC → USDC sent to user's Anzo Solana wallet
- **Off-Ramp**: User sends USDC from their Anzo wallet (Solana or EVM) → Bridge converts to USD → USD deposited to user's real bank account (US ACH or SEPA)

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Frontend                                    │
│                    (React Native / Web App)                              │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │ REST API
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Anzo Backend                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                       Bridge Module                                │  │
│  │                                                                    │  │
│  │  ┌────────────────┐     ┌─────────────────────────────────────┐   │  │
│  │  │  Controller    │     │         Service (Facade)            │   │  │
│  │  │  - KYC         │────▶│  Delegates to specialized handlers  │   │  │
│  │  │  - On-Ramp     │     └──────────────┬──────────────────────┘   │  │
│  │  │  - Off-Ramp    │                    │                          │  │
│  │  │  - Transactions│                    ▼                          │  │
│  │  └────────────────┘     ┌─────────────────────────────────────┐   │  │
│  │                         │           Handlers                   │   │  │
│  │  ┌────────────────┐     │  ┌─────────────┐ ┌───────────────┐  │   │  │
│  │  │    Webhook     │     │  │  KYC        │ │   On-Ramp     │  │   │  │
│  │  │   Controller   │────▶│  │  Handler    │ │   Handler     │  │   │  │
│  │  └────────────────┘     │  └─────────────┘ └───────────────┘  │   │  │
│  │                         │  ┌─────────────┐ ┌───────────────┐  │   │  │
│  │                         │  │  Off-Ramp   │ │  Transaction  │  │   │  │
│  │                         │  │  Handler    │ │   Handler     │  │   │  │
│  │                         │  └─────────────┘ └───────────────┘  │   │  │
│  │                         └──────────────┬──────────────────────┘   │  │
│  │                                        │                          │  │
│  │                                        ▼                          │  │
│  │                         ┌─────────────────────────────────────┐   │  │
│  │                         │      Bridge Provider (Axios)        │   │  │
│  │                         │    - API Client with interceptors   │   │  │
│  │                         │    - Idempotency key generation     │   │  │
│  │                         └──────────────┬──────────────────────┘   │  │
│  └────────────────────────────────────────┼──────────────────────────┘  │
│                                           │                              │
│  ┌────────────────────────────────────────┼──────────────────────────┐  │
│  │                    Database (PostgreSQL via Prisma)               │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌────────────────────────────┐ │  │
│  │  │BridgeCustomer│ │ BridgeWallet │ │   BridgeVirtualAccount     │ │  │
│  │  └──────────────┘ └──────────────┘ └────────────────────────────┘ │  │
│  │  ┌──────────────────────────┐ ┌─────────────────────────────────┐ │  │
│  │  │  BridgeExternalAccount   │ │      BridgeTransaction          │ │  │
│  │  └──────────────────────────┘ └─────────────────────────────────┘ │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────┬───────────────────────────────────┘
                                      │ HTTPS
                                      ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Bridge.xyz API                                  │
│                     https://api.bridge.xyz/v0                           │
│  ┌────────────┐ ┌─────────────┐ ┌─────────┐ ┌──────────────────────────┐│
│  │ KYC Links  │ │   Wallets   │ │ Virtual │ │   External Accounts      ││
│  │            │ │             │ │ Accounts│ │                          ││
│  └────────────┘ └─────────────┘ └─────────┘ └──────────────────────────┘│
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │                         Transfers API                              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Module Structure

```
src/modules/bridge/
├── bridge.controller.ts           # API endpoints for frontend
├── bridge.service.ts              # Facade service (97 lines)
├── bridge.module.ts               # Module definition
├── index.ts
│
├── handlers/                      # Specialized business logic handlers
│   ├── bridge-kyc.handler.ts      # KYC initiation, status, approval (280 lines)
│   ├── bridge-onramp.handler.ts   # Virtual account operations (60 lines)
│   ├── bridge-offramp.handler.ts  # External accounts & off-ramp (327 lines)
│   ├── bridge-transaction.handler.ts  # Transactions & webhooks (104 lines)
│   └── index.ts
│
├── providers/
│   └── bridge.provider.ts         # Axios client for Bridge API (265 lines)
│
├── dto/                           # Request validation
│   ├── initiate-kyc.dto.ts
│   ├── create-external-account.dto.ts
│   ├── initiate-off-ramp.dto.ts
│   └── index.ts
│
├── guards/
│   └── bridge-webhook.guard.ts    # Webhook signature verification
│
├── utils/
│   └── bridge-error.util.ts       # Centralized error handling
│
└── webhooks/
    ├── bridge-webhook.controller.ts  # Webhook endpoint
    ├── bridge-webhook.handler.ts     # Signature verification
    ├── bridge-webhook.types.ts       # TypeScript interfaces
    └── index.ts
```

---

## Environment Variables

Add these to your `.env` file:

```env
# ============================================
# Bridge.xyz Integration
# ============================================

# Your Bridge API key from the Bridge dashboard
BRIDGE_API_KEY=your-bridge-api-key

# Bridge API URL (defaults to production)
BRIDGE_API_URL=https://api.bridge.xyz/v0

# Public key for webhook signature verification (RSA)
# Get this from Bridge dashboard > Webhooks > Signing Keys
BRIDGE_WEBHOOK_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
MIIBIjAN...your-public-key...
-----END PUBLIC KEY-----"

# Developer fee percentage (optional, default: 0.5)
# This fee is charged on each transaction
BRIDGE_DEVELOPER_FEE_PERCENT=0.5
```

---

## Database Schema

### Entity Relationship Diagram

```
┌─────────────────┐     ┌─────────────────┐     ┌────────────────────────┐
│      User       │────▶│ BridgeCustomer  │────▶│    BridgeWallet        │
│                 │  1:1│                 │  1:1│   (Intermediate USDC)  │
└─────────────────┘     │                 │     └────────────────────────┘
                        │                 │
                        │                 │────▶┌────────────────────────┐
                        │                 │  1:1│  BridgeVirtualAccount  │
                        │                 │     │   (On-Ramp Bank)       │
                        │                 │     └────────────────────────┘
                        │                 │
                        │                 │────▶┌────────────────────────┐
                        │                 │  1:n│ BridgeExternalAccount  │
                        └─────────────────┘     │  (Off-Ramp Bank)       │
                                                └────────────────────────┘
┌─────────────────────────────────────────────────────────────────────────┐
│                        BridgeTransaction                                 │
│   Tracks all transfers (on-ramp payouts & off-ramp transactions)        │
└─────────────────────────────────────────────────────────────────────────┘
```

### Model Details

#### BridgeCustomer

Stores the core Bridge customer identity and KYC status.

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Primary key (cuid) |
| `userId` | String | Foreign key to User (unique) |
| `customerId` | String | Bridge customer ID, e.g., `cust_abc123` |
| `kycLinkId` | String? | The KYC link ID from Bridge |
| `kycStatus` | String | `not_started`, `under_review`, `incomplete`, `approved`, `rejected` |
| `tosStatus` | String | `pending`, `approved` |
| `status` | String | `PENDING_KYC`, `KYC_APPROVED`, `ACTIVE`, `FAILED` |

#### BridgeWallet

Intermediate Bridge-managed wallet for holding USDC during on-ramp.

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Primary key (cuid) |
| `bridgeCustomerId` | String | Foreign key (unique) |
| `walletId` | String | Bridge wallet ID, e.g., `wallet_123` |
| `chain` | String | Blockchain, e.g., `solana`, `ethereum` |
| `address` | String | Wallet's blockchain address |

#### BridgeVirtualAccount

User's virtual bank account for depositing USD (on-ramp).

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Primary key (cuid) |
| `virtualAccountId` | String | Bridge virtual account ID |
| `status` | String | `pending`, `activated`, `deactivated` |
| `sourceCurrency` | String | e.g., `usd` |
| `destinationCurrency` | String | e.g., `usdc` |
| `destinationPaymentRail` | String | e.g., `solana`, `ethereum` |
| `destinationAddress` | String | User's Anzo wallet address |
| `bankName` | String | e.g., `Lead Bank` |
| `bankRoutingNumber` | String? | US routing number |
| `bankAccountNumber` | String | Account number |
| `bankBeneficiaryName` | String | Beneficiary name |
| `iban` | String? | For SEPA accounts |
| `bic` | String? | For SEPA accounts |
| `paymentRail` | String | e.g., `ach_push`, `wire`, `sepa` |
| `paymentRails` | String[] | Available payment rails |

#### BridgeExternalAccount

User's real-world bank accounts for receiving USD (off-ramp).

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Primary key (cuid) |
| `externalAccountId` | String | Bridge external account ID |
| `accountType` | String | `us`, `ach`, `sepa` |
| `currency` | String | `usd`, `eur` |
| `bankName` | String | Bank name |
| `accountName` | String? | User-defined name |
| `accountOwnerName` | String | Account owner's name |
| `accountOwnerType` | String | `individual` or `business` |
| `last4` | String | Last 4 digits of account |
| `routingNumber` | String? | US accounts |
| `iban` | String? | SEPA accounts |
| `bic` | String? | SEPA accounts |
| `isActive` | Boolean | Soft delete flag |

#### BridgeTransaction

Tracks all Bridge transfers.

| Field | Type | Description |
|-------|------|-------------|
| `id` | String | Primary key (cuid) |
| `transferId` | String | Bridge transfer ID |
| `type` | String | `ON_RAMP_PAYOUT`, `OFF_RAMP` |
| `status` | String | See status table below |
| `amount` | String | Transfer amount |
| `developerFee` | String? | Fee amount |
| `sourceAsset` | String | e.g., `USD`, `USDC` |
| `sourceCurrency` | String | Currency code |
| `sourcePaymentRail` | String | e.g., `solana`, `ach` |
| `destinationAsset` | String | e.g., `USDC`, `USD` |
| `destinationCurrency` | String | Currency code |
| `destinationPaymentRail` | String | e.g., `ach`, `solana` |
| `depositAddress` | String? | For off-ramp crypto deposits |
| `depositAmount` | String? | Amount to deposit |
| `finalAmount` | String? | Final amount after fees |
| `completedAt` | DateTime? | Completion timestamp |

**Transfer Status Values:**

| Status | Description |
|--------|-------------|
| `awaiting_funds` | Waiting for user to deposit funds |
| `funds_received` | Funds received, processing |
| `payment_submitted` | Payment sent to destination |
| `payment_processed` | Payment completed successfully |
| `canceled` | Transfer canceled |
| `returned` | Transfer returned/refunded |
| `error` | Transfer failed |

### Run Migrations

```bash
npx prisma migrate dev --name add_bridge_integration
npx prisma generate
```

---

## API Endpoints

Base URL: `/api/v1/bridge`

All endpoints (except webhooks) require authentication via `Authorization: Bearer <token>` header.

### 1. KYC & Onboarding

#### POST /api/v1/bridge/kyc/initiate

Initiate KYC process for the authenticated user.

**Request:**
```json
{
  "fullName": "John Doe",
  "email": "john.doe@example.com",
  "type": "individual",
  "endorsement": "base"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `fullName` | string | Yes | User's full legal name |
| `email` | string | Yes | User's email address |
| `type` | string | No | `individual` (default) or `business` |
| `endorsement` | string | No | `base` (default) - required capability |

**Response (200 OK):**
```json
{
  "customerId": "cust_abc123",
  "kycLinkId": "kyc_xyz789",
  "kycLink": "https://bridge.withpersona.com/verify?...",
  "tosLink": "https://dashboard.bridge.xyz/accept-terms-of-service?...",
  "kycStatus": "not_started",
  "tosStatus": "pending",
  "status": "PENDING_KYC"
}
```

**Error Responses:**
- `409 Conflict` - User already has an active Bridge account

---

#### GET /api/v1/bridge/kyc/status

Get KYC status for the authenticated user.

**Response (200 OK):**
```json
{
  "hasAccount": true,
  "customerId": "cust_abc123",
  "status": "ACTIVE",
  "kycStatus": "approved",
  "tosStatus": "approved",
  "hasVirtualAccount": true,
  "hasWallet": true
}
```

**Response (No Account):**
```json
{
  "hasAccount": false,
  "status": null,
  "kycStatus": null,
  "tosStatus": null,
  "hasVirtualAccount": false,
  "hasWallet": false
}
```

---

### 2. Virtual Account (On-Ramp)

#### GET /api/v1/bridge/virtual-account

Get the user's virtual bank account details for on-ramping.

**Response (200 OK):**
```json
{
  "status": "activated",
  "currency": "usd",
  "depositInstructions": {
    "bankName": "Lead Bank",
    "bankAddress": "1801 Main St., Kansas City, MO 64108",
    "bankRoutingNumber": "101019644",
    "bankAccountNumber": "215268120000",
    "bankBeneficiaryName": "John Doe",
    "bankBeneficiaryAddress": "123 Main St, San Francisco, CA 94107, US",
    "iban": null,
    "bic": null,
    "paymentRail": "ach_push",
    "availablePaymentRails": ["ach_push", "wire"]
  },
  "destination": {
    "currency": "usdc",
    "paymentRail": "solana",
    "address": "4oG1sWkP8vcrnhbkzZc1m9RTT2VUjZHKx92qiVFK7FuZ"
  }
}
```

**Error Responses:**
- `404 Not Found` - No Bridge account or virtual account
- `400 Bad Request` - Account not yet active

---

### 3. External Accounts (Off-Ramp Destination)

#### POST /api/v1/bridge/external-accounts

Add an external bank account for off-ramping.

**Request (US ACH):**
```json
{
  "currency": "usd",
  "accountType": "us",
  "bankName": "Chase",
  "accountName": "My Chase Checking",
  "accountOwnerType": "individual",
  "accountOwnerName": "John Doe",
  "firstName": "John",
  "lastName": "Doe",
  "usAccount": {
    "routingNumber": "021000021",
    "accountNumber": "123456789012",
    "checkingOrSavings": "checking"
  },
  "address": {
    "streetLine1": "123 Main Street",
    "streetLine2": "Apt 4B",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94107",
    "country": "USA"
  }
}
```

**Request (SEPA - Europe):**
```json
{
  "currency": "eur",
  "accountType": "sepa",
  "bankName": "Deutsche Bank",
  "accountName": "My German Account",
  "accountOwnerType": "individual",
  "accountOwnerName": "Hans Mueller",
  "firstName": "Hans",
  "lastName": "Mueller",
  "sepaAccount": {
    "iban": "DE89370400440532013000",
    "bic": "DEUTDEDB"
  },
  "address": {
    "streetLine1": "Hauptstrasse 1",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DEU"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "clm123...",
  "externalAccountId": "ea_abc123",
  "bankName": "Chase",
  "accountName": "My Chase Checking",
  "last4": "9012",
  "accountType": "us",
  "currency": "usd",
  "isActive": true
}
```

---

#### GET /api/v1/bridge/external-accounts

List all external bank accounts.

**Response (200 OK):**
```json
[
  {
    "id": "clm123...",
    "externalAccountId": "ea_abc123",
    "bankName": "Chase",
    "accountName": "My Chase Checking",
    "last4": "9012",
    "accountType": "us",
    "currency": "usd",
    "accountOwnerName": "John Doe"
  }
]
```

---

#### DELETE /api/v1/bridge/external-accounts/:externalAccountId

Delete/deactivate an external bank account.

**Response (200 OK):**
```json
{
  "success": true
}
```

---

### 4. Off-Ramp (Crypto → USD)

#### POST /api/v1/bridge/off-ramp/initiate

Initiate an off-ramp transfer.

**Request (Solana):**
```json
{
  "amount": "100.00",
  "sourceChain": "solana",
  "sourceCurrency": "usdc",
  "externalAccountId": "ea_abc123",
  "destinationPaymentRail": "ach"
}
```

**Request (EVM with custom fromAddress):**
```json
{
  "amount": "100.00",
  "sourceChain": "ethereum",
  "sourceCurrency": "usdc",
  "externalAccountId": "ea_abc123",
  "destinationPaymentRail": "ach",
  "fromAddress": "0x1234567890abcdef1234567890abcdef12345678"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | Yes | Amount to off-ramp |
| `sourceChain` | string | Yes | `solana` or `ethereum` |
| `sourceCurrency` | string | No | Default: `usdc` |
| `externalAccountId` | string | Yes | Bridge external account ID |
| `destinationPaymentRail` | string | No | `ach` (default), `wire`, `sepa` |
| `fromAddress` | string | No | Override source wallet address |

**Response (200 OK):**
```json
{
  "transferId": "transfer_xyz789",
  "status": "awaiting_funds",
  "amount": "100.00",
  "sourceCurrency": "usdc",
  "destinationCurrency": "usd",
  "depositInstructions": {
    "paymentRail": "solana",
    "depositAddress": "BridgeDepositAddress123...",
    "depositAmount": "100.00",
    "depositCurrency": "usdc",
    "fromAddress": "UserSolanaAddress..."
  },
  "estimatedFees": {
    "developerFee": "0.50",
    "exchangeFee": "0.00",
    "gasFee": "0.01"
  },
  "estimatedFinalAmount": "99.49"
}
```

**Important:** After receiving the response, the frontend must:
1. Extract `depositInstructions.depositAddress`
2. Send exactly `depositAmount` of USDC to that address
3. Use Turnkey SDK to sign the transaction

---

#### GET /api/v1/bridge/off-ramp/:transferId/status

Get the status of an off-ramp transfer.

**Response (200 OK):**
```json
{
  "transferId": "transfer_xyz789",
  "type": "OFF_RAMP",
  "status": "PAYMENT_PROCESSED",
  "amount": "100.00",
  "sourceAsset": "USDC",
  "destinationAsset": "USD",
  "depositAddress": "BridgeDepositAddress123...",
  "fees": {
    "developerFee": "0.50",
    "exchangeFee": "0.00",
    "gasFee": "0.01"
  },
  "finalAmount": "99.49",
  "createdAt": "2024-01-15T10:30:00.000Z",
  "completedAt": "2024-01-15T10:35:00.000Z"
}
```

---

### 5. Transaction History

#### GET /api/v1/bridge/transactions

List all Bridge transactions.

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `type` | string | Optional: `ON_RAMP_PAYOUT` or `OFF_RAMP` |

**Response (200 OK):**
```json
[
  {
    "id": "clm123...",
    "transferId": "transfer_xyz789",
    "type": "OFF_RAMP",
    "status": "PAYMENT_PROCESSED",
    "amount": "100.00",
    "sourceAsset": "USDC",
    "destinationAsset": "USD",
    "finalAmount": "99.49",
    "createdAt": "2024-01-15T10:30:00.000Z",
    "completedAt": "2024-01-15T10:35:00.000Z"
  }
]
```

---

### 6. Webhooks

#### POST /api/v1/bridge/webhook

**Internal endpoint** - Receives webhooks from Bridge.xyz.

This endpoint verifies the `X-Webhook-Signature` header using RSA-SHA256 and processes events.

**Supported Events:**

| Event | Description |
|-------|-------------|
| `customer.created` | New customer created |
| `customer.updated` | KYC/TOS status changed (triggers account setup on approval) |
| `transfer.created` | New transfer initiated |
| `transfer.updated` | Transfer status changed |
| `virtual_account.created` | Virtual account created |
| `virtual_account.activity` | Funds received/sent |
| `wallet.created` | Bridge wallet created |

**Webhook Payload Example:**
```json
{
  "event_id": "evt_abc123",
  "event_category": "customer",
  "event_type": "updated",
  "event_object": {
    "id": "cust_abc123",
    "kyc_status": "approved",
    "tos_status": "approved",
    "endorsements": [
      {
        "name": "base",
        "status": "approved"
      }
    ]
  }
}
```

---

## Detailed Workflows

### On-Ramp Flow (USD → Crypto)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        ON-RAMP WORKFLOW                                  │
│                    (USD → USDC to Solana Wallet)                        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ User    │     │  Frontend   │     │  Backend    │     │  Bridge.xyz │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
     │                 │                   │                   │
     │ 1. Click "Get   │                   │                   │
     │    USD Account" │                   │                   │
     │─────────────────▶                   │                   │
     │                 │                   │                   │
     │                 │ 2. POST /kyc/initiate                 │
     │                 │──────────────────▶│                   │
     │                 │                   │                   │
     │                 │                   │ 3. POST /v0/kyc_links
     │                 │                   │──────────────────▶│
     │                 │                   │                   │
     │                 │                   │◀──────────────────│
     │                 │                   │   customer_id,    │
     │                 │                   │   kyc_link,       │
     │                 │◀──────────────────│   tos_link        │
     │                 │                   │                   │
     │                 │ 4. Render KYC     │                   │
     │◀─────────────────  WebView         │                   │
     │                 │                   │                   │
     │ 5. Complete KYC │                   │                   │
     │    in Persona   │                   │                   │
     │─────────────────────────────────────────────────────────▶
     │                 │                   │                   │
     │ 6. Accept TOS   │                   │                   │
     │─────────────────────────────────────────────────────────▶
     │                 │                   │                   │
     │                 │                   │ 7. Webhook:       │
     │                 │                   │    customer.updated
     │                 │                   │◀──────────────────│
     │                 │                   │   (base: approved)│
     │                 │                   │                   │
     │                 │                   │ 8. Auto-create:   │
     │                 │                   │    - Bridge Wallet│
     │                 │                   │    - Virtual Acct │
     │                 │                   │                   │
     │                 │ 9. GET /virtual-account               │
     │                 │──────────────────▶│                   │
     │                 │◀──────────────────│                   │
     │                 │   Bank details    │                   │
     │                 │                   │                   │
     │◀─────────────────  10. Display      │                   │
     │  Bank Details   │      bank info    │                   │
     │                 │                   │                   │
     │ 11. Wire USD to │                   │                   │
     │     virtual acct│                   │                   │
     │─────────────────────────────────────────────────────────▶
     │                 │                   │                   │
     │                 │                   │ 12. Webhook:      │
     │                 │                   │     virtual_account
     │                 │                   │     .activity     │
     │                 │                   │◀──────────────────│
     │                 │                   │                   │
     │                 │                   │ 13. Bridge auto-  │
     │                 │                   │     converts USD  │
     │                 │                   │     → USDC        │
     │                 │                   │                   │
     │ 14. USDC arrives│                   │ 15. USDC sent to  │
     │     in Anzo     │                   │     user's Solana │
     │     wallet      │◀──────────────────│     wallet        │
     │                 │                   │                   │
     ▼                 ▼                   ▼                   ▼
```

**Customer Status Transitions:**
```
PENDING_KYC → KYC_APPROVED → ACTIVE
      │                           │
      └──────→ FAILED ←───────────┘
```

---

### Off-Ramp Flow (Crypto → USD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OFF-RAMP WORKFLOW                                 │
│                  (USDC from Solana/EVM → USD to Bank)                   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ User    │     │  Frontend   │     │  Backend    │     │  Bridge.xyz │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
     │                 │                   │                   │
     │ 1. Add Bank     │                   │                   │
     │    Account      │                   │                   │
     │─────────────────▶                   │                   │
     │                 │                   │                   │
     │                 │ 2. POST /external-accounts            │
     │                 │──────────────────▶│                   │
     │                 │                   │                   │
     │                 │                   │ 3. POST /v0/customers
     │                 │                   │    /{id}/external_accounts
     │                 │                   │──────────────────▶│
     │                 │                   │◀──────────────────│
     │                 │◀──────────────────│                   │
     │                 │  external_account │                   │
     │                 │  _id saved        │                   │
     │                 │                   │                   │
     │ 4. Click        │                   │                   │
     │    "Sell USDC"  │                   │                   │
     │─────────────────▶                   │                   │
     │                 │                   │                   │
     │                 │ 5. POST /off-ramp/initiate            │
     │                 │   {amount, sourceChain: "solana",     │
     │                 │    externalAccountId}                 │
     │                 │──────────────────▶│                   │
     │                 │                   │                   │
     │                 │                   │ 6. POST /v0/transfers
     │                 │                   │──────────────────▶│
     │                 │                   │◀──────────────────│
     │                 │                   │  deposit_address  │
     │                 │                   │  (awaiting_funds) │
     │                 │◀──────────────────│                   │
     │                 │  depositInstructions                  │
     │                 │                   │                   │
     │◀─────────────────  7. Prompt user   │                   │
     │                 │     to confirm    │                   │
     │                 │                   │                   │
     │ 8. Confirm send │                   │                   │
     │─────────────────▶                   │                   │
     │                 │                   │                   │
     │                 │ 9. Build Solana   │                   │
     │                 │    transaction    │                   │
     │                 │                   │                   │
     │                 │ 10. Sign via      │                   │
     │                 │     Turnkey SDK   │                   │
     │                 │                   │                   │
     │                 │ 11. Broadcast     │                   │
     │                 │     transaction   │                   │
     │                 │─────────────────────────────────────▶ │
     │                 │    (USDC to Bridge deposit address)   │
     │                 │                   │                   │
     │                 │                   │ 12. Webhook:      │
     │                 │                   │     transfer      │
     │                 │                   │     .updated      │
     │                 │                   │◀──────────────────│
     │                 │                   │  (funds_received) │
     │                 │                   │                   │
     │                 │                   │ 13. Bridge        │
     │                 │                   │     converts USDC │
     │                 │                   │     → USD         │
     │                 │                   │                   │
     │                 │                   │ 14. Webhook:      │
     │                 │                   │     transfer      │
     │                 │                   │     .updated      │
     │                 │                   │◀──────────────────│
     │                 │                   │ (payment_processed)
     │                 │                   │                   │
     │ 15. USD arrives │                   │ 16. ACH sent to   │
     │     in bank     │                   │     user's bank   │
     │     account     │                   │                   │
     │                 │                   │                   │
     ▼                 ▼                   ▼                   ▼
```

---

## Testing with Postman

### Collection Setup

1. Create a new Postman collection named "Anzo Bridge API"
2. Set base URL variable: `{{baseUrl}}` = `http://localhost:3000/api/v1`
3. Set auth token variable: `{{authToken}}` = Your JWT token

### Request Examples

#### 1. Initiate KYC
```http
POST {{baseUrl}}/bridge/kyc/initiate
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "fullName": "Test User",
  "email": "test@example.com",
  "type": "individual"
}
```

#### 2. Check KYC Status
```http
GET {{baseUrl}}/bridge/kyc/status
Authorization: Bearer {{authToken}}
```

#### 3. Get Virtual Account
```http
GET {{baseUrl}}/bridge/virtual-account
Authorization: Bearer {{authToken}}
```

#### 4. Add External Account (US ACH)
```http
POST {{baseUrl}}/bridge/external-accounts
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "currency": "usd",
  "accountType": "us",
  "bankName": "Test Bank",
  "accountName": "My Checking",
  "accountOwnerType": "individual",
  "accountOwnerName": "Test User",
  "firstName": "Test",
  "lastName": "User",
  "usAccount": {
    "routingNumber": "110000000",
    "accountNumber": "000123456789",
    "checkingOrSavings": "checking"
  },
  "address": {
    "streetLine1": "123 Test Street",
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94107",
    "country": "USA"
  }
}
```

#### 5. Add External Account (SEPA)
```http
POST {{baseUrl}}/bridge/external-accounts
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "currency": "eur",
  "accountType": "sepa",
  "bankName": "Deutsche Bank",
  "accountName": "My Euro Account",
  "accountOwnerType": "individual",
  "accountOwnerName": "Hans Mueller",
  "firstName": "Hans",
  "lastName": "Mueller",
  "sepaAccount": {
    "iban": "DE89370400440532013000",
    "bic": "DEUTDEDB"
  },
  "address": {
    "streetLine1": "Hauptstrasse 1",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DEU"
  }
}
```

#### 6. List External Accounts
```http
GET {{baseUrl}}/bridge/external-accounts
Authorization: Bearer {{authToken}}
```

#### 7. Delete External Account
```http
DELETE {{baseUrl}}/bridge/external-accounts/ea_abc123
Authorization: Bearer {{authToken}}
```

#### 8. Initiate Off-Ramp (Solana)
```http
POST {{baseUrl}}/bridge/off-ramp/initiate
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "amount": "10.00",
  "sourceChain": "solana",
  "sourceCurrency": "usdc",
  "externalAccountId": "ea_abc123"
}
```

#### 9. Initiate Off-Ramp (Ethereum)
```http
POST {{baseUrl}}/bridge/off-ramp/initiate
Authorization: Bearer {{authToken}}
Content-Type: application/json

{
  "amount": "50.00",
  "sourceChain": "ethereum",
  "sourceCurrency": "usdc",
  "externalAccountId": "ea_abc123",
  "destinationPaymentRail": "wire"
}
```

#### 10. Check Off-Ramp Status
```http
GET {{baseUrl}}/bridge/off-ramp/{{transferId}}/status
Authorization: Bearer {{authToken}}
```

#### 11. List Transactions
```http
GET {{baseUrl}}/bridge/transactions?type=OFF_RAMP
Authorization: Bearer {{authToken}}
```

---

## Webhook Testing

### Local Development

1. **Use ngrok to expose your local server:**
   ```bash
   ngrok http 3000
   ```

2. **Configure the webhook URL in Bridge dashboard:**
   ```
   https://your-ngrok-url.ngrok.io/api/v1/bridge/webhook
   ```

3. **Copy the webhook public key from Bridge and add to `.env`**

### Webhook Signature Verification

The webhook handler verifies signatures using RSA-SHA256:

```
Signature Header Format:
X-Webhook-Signature: t=1234567890,v0=base64EncodedSignature

Verification Process:
1. Extract timestamp (t) and signature (v0) from header
2. Construct signed payload: {timestamp}.{rawBody}
3. Verify signature using RSA-SHA256 with Bridge's public key
4. Reject if timestamp > 10 minutes old (replay protection)
```

---

## Error Handling

All errors follow this format:

```json
{
  "statusCode": 400,
  "message": "Error description",
  "error": "Bad Request"
}
```

### Error Codes

| Code | Error | Description |
|------|-------|-------------|
| `400` | Bad Request | Invalid request (missing fields, validation errors) |
| `401` | Unauthorized | Not authenticated or Bridge API auth failed |
| `404` | Not Found | No Bridge account, virtual account, or transaction |
| `409` | Conflict | User already has Bridge account |
| `500` | Internal Error | Bridge API failure or unexpected error |

### Common Error Scenarios

| Scenario | Error |
|----------|-------|
| Initiate KYC when already active | `409: User already has an active Bridge account` |
| Get virtual account before KYC | `404: No Bridge account found` |
| Off-ramp before adding bank | `404: External account not found` |
| Off-ramp with inactive account | `400: Account not active` |
| Missing wallet for chain | `400: No wallet address found for chain: solana` |

---

## Security Considerations

1. **Webhook Verification**: All webhooks are verified using RSA-SHA256 signature
2. **Replay Prevention**: Webhooks older than 10 minutes are rejected
3. **API Authentication**: All user endpoints require JWT authentication
4. **Data Privacy**: Bank account numbers are stored with only last 4 digits visible
5. **Idempotency**: All mutating API calls use idempotency keys to prevent duplicates

---

## Performance Notes

### Time Complexity

| Operation | Complexity | Notes |
|-----------|------------|-------|
| `initiateKyc` | O(1) | Single DB lookup + API call |
| `getKycStatus` | O(1) | Single DB query with includes |
| `getVirtualAccount` | O(1) | Single DB query |
| `createExternalAccount` | O(1) | Single DB insert + API call |
| `listExternalAccounts` | O(n) | n = number of external accounts (typically < 10) |
| `initiateOffRamp` | O(1) | Single DB query + insert + API call |
| `listTransactions` | O(n) | n = capped at 50 results |

### Database Indexes

All models have appropriate indexes for efficient queries:
- `BridgeCustomer`: `userId`, `customerId`, `status`
- `BridgeTransaction`: `userId`, `transferId`, `status`, `type`
- `BridgeExternalAccount`: `bridgeCustomerId`, `externalAccountId`

---

## Documentation References

- [Bridge.xyz API Docs](https://apidocs.bridge.xyz)
- [KYC Links](https://apidocs.bridge.xyz/get-started/introduction/quick-start/create-your-first-customer)
- [Virtual Accounts](https://apidocs.bridge.xyz/platform/orchestration/virtual_accounts/virtual-account)
- [Wallets](https://apidocs.bridge.xyz/get-started/guides/wallets/onramp)
- [External Accounts](https://apidocs.bridge.xyz/get-started/guides/move-money/offramp_liquidation)
- [Transfers](https://apidocs.bridge.xyz/get-started/guides/wallets/offramp)
- [Webhooks](https://apidocs.bridge.xyz/get-started/introduction/quick-start/setting-up-webhooks)
