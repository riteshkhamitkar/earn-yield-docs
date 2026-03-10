# Wallet-Based Payout (Off-Ramp) — Implementation Guide

## Overview

This document covers the implementation of **wallet-based payouts** — the ability for a user to send fiat money from their Bridge-managed wallet to an external bank account (e.g., a Mexican CLABE account for MXN, a UK sort code for GBP, etc.).

### The "Stablecoin Sandwich" Architecture

Bridge Virtual Accounts are **receive-only** — they cannot initiate outgoing payments. The correct architecture uses a Bridge-managed Custodial Wallet as the intermediary:

```
[User's Bank (USD)] ──(ACH/Wire)──> [Bridge Virtual Account]
                                           │
                                    (auto-conversion)
                                           ▼
                                    [Bridge Wallet (USDC)]   ← Central Hub
                                           │
                                    (Transfer API)
                                           ▼
                               [Recipient's Bank (MXN/EUR/GBP/BRL)]
```

**On-Ramp:** External bank → Virtual Account → auto-converts to USDC → Bridge Wallet
**Off-Ramp (Payout):** Bridge Wallet → Transfer API → auto-converts to fiat → External bank account

---

## Database Schema

**No database changes were required.** The existing Prisma schema already supports this flow:

| Model | Role |
|---|---|
| `BridgeCustomer` | Links your user to their Bridge customer ID |
| `BridgeWallet` | The user's Bridge-managed wallet (USDC holding account) |
| `BridgeVirtualAccount` | User's virtual bank accounts for receiving deposits (USD/EUR) |
| `BridgeExternalAccount` | Saved payee bank accounts (where money is sent TO) |
| `BridgeTransaction` | Tracks all transfers — both on-ramp and off-ramp |

---

## What Was Implemented

### 1. Extended External Account Types

**File:** `src/modules/bridge/dto/create-external-account.dto.ts`

Previously supported: `us`, `ach`, `sepa`, `wire`
Now also supports: **`clabe`** (Mexico), **`pix`** (Brazil), **`gb`** (UK)

New nested DTOs added:

| DTO | Account Type | Fields |
|---|---|---|
| `CLABEAccountDetailsDto` | `clabe` | `accountNumber` (18-digit CLABE) |
| `PIXAccountDetailsDto` | `pix` | `pixKey`, `documentNumber` (CPF/CNPJ) |
| `GBAccountDetailsDto` | `gb` | `sortCode` (6-digit), `accountNumber` (8-digit) |

### 2. New Payout DTO

**File:** `src/modules/bridge/dto/initiate-payout.dto.ts`

```typescript
{
  externalAccountId: string;  // Internal DB ID of the saved payee
  amount: string;             // Exact amount recipient gets (e.g., "10000.00")
  currency: string;           // Recipient's currency (e.g., "mxn")
  sourceCurrency?: string;    // Optional, defaults to "usdc"
}
```

### 3. New Payout Endpoint

**Endpoint:** `POST /api/v1/bridge/transfers/payout`

**File:** `src/modules/bridge/bridge.controller.ts`

### 4. Payout Handler

**File:** `src/modules/bridge/handlers/bridge-offramp.handler.ts`

New method `initiatePayout()` that:
1. Fetches the external account, customer, and wallet in one query
2. Maps `accountType` → Bridge `payment_rail` automatically
3. Builds a **fixed-output** transfer payload (amount in destination, not source)
4. Calls Bridge's `POST /v0/transfers` API
5. Saves the transaction to `BridgeTransaction`

### 5. Provider Updates

**File:** `src/modules/bridge/providers/bridge.provider.ts`

- `createTransfer()`: `amount` is now optional at top level; `destination.amount` added for fixed-output transfers
- `createExternalAccount()`: Now accepts `clabe`, `pix_key`, and `gb` nested objects

---

## API Reference

### Create External Account (Payee)

`POST /api/v1/bridge/external-accounts`

**Mexican CLABE example:**
```json
{
  "currency": "mxn",
  "accountType": "clabe",
  "bankName": "BBVA Mexico",
  "accountName": "Maria's Business Account",
  "accountOwnerType": "individual",
  "accountOwnerName": "Maria Garcia",
  "firstName": "Maria",
  "lastName": "Garcia",
  "clabeAccount": {
    "accountNumber": "626899715090851234"
  },
  "address": {
    "streetLine1": "Av. Reforma 123",
    "city": "Mexico City",
    "state": "CDMX",
    "postalCode": "06600",
    "country": "MEX"
  }
}
```

**UK Bank example:**
```json
{
  "currency": "gbp",
  "accountType": "gb",
  "bankName": "Barclays",
  "accountOwnerType": "individual",
  "accountOwnerName": "James Smith",
  "firstName": "James",
  "lastName": "Smith",
  "gbAccount": {
    "sortCode": "200000",
    "accountNumber": "12345678"
  },
  "address": {
    "streetLine1": "10 Downing St",
    "city": "London",
    "postalCode": "SW1A 2AA",
    "country": "GBR"
  }
}
```

**Brazilian PIX example:**
```json
{
  "currency": "brl",
  "accountType": "pix",
  "bankName": "Nubank",
  "accountOwnerType": "individual",
  "accountOwnerName": "Lucas Silva",
  "firstName": "Lucas",
  "lastName": "Silva",
  "pixAccount": {
    "pixKey": "lucas.silva@email.com",
    "documentNumber": "12345678901"
  },
  "address": {
    "streetLine1": "Rua Augusta 100",
    "city": "Sao Paulo",
    "state": "SP",
    "postalCode": "01310-000",
    "country": "BRA"
  }
}
```

**US ACH example (unchanged):**
```json
{
  "currency": "usd",
  "accountType": "us",
  "bankName": "Chase",
  "accountName": "My Checking",
  "accountOwnerType": "individual",
  "accountOwnerName": "Carlos Ramirez",
  "firstName": "Carlos",
  "lastName": "Ramirez",
  "usAccount": {
    "routingNumber": "021000021",
    "accountNumber": "123456789012",
    "checkingOrSavings": "checking"
  },
  "address": {
    "streetLine1": "123 Main St",
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "USA"
  }
}
```

### List External Accounts

`GET /api/v1/bridge/external-accounts`

**Response:**
```json
[
  {
    "id": "clxyz123...",
    "externalAccountId": "ea_abc123",
    "bankName": "BBVA Mexico",
    "accountName": "Maria's Business Account",
    "last4": "1234",
    "accountType": "clabe",
    "currency": "mxn",
    "accountOwnerName": "Maria Garcia"
  }
]
```

### Delete External Account

`DELETE /api/v1/bridge/external-accounts/:externalAccountId`

### Initiate Payout (NEW)

`POST /api/v1/bridge/transfers/payout`

**Request:**
```json
{
  "externalAccountId": "clxyz123...",
  "amount": "10000.00",
  "currency": "mxn"
}
```

- `externalAccountId`: The internal DB `id` from the list endpoint (NOT the Bridge external account ID)
- `amount`: The exact amount the recipient should receive
- `currency`: The recipient's currency

**Response:**
```json
{
  "transactionId": "clxyz456...",
  "bridgeTransferId": "transfer_abc123",
  "status": "payment_submitted",
  "sourceAmountDebited": "595.23",
  "sourceCurrency": "USDC",
  "destinationAmount": "10000.00",
  "destinationCurrency": "MXN",
  "fees": {
    "developerFee": "0.50",
    "exchangeFee": "2.97",
    "gasFee": "0.00"
  }
}
```

### Existing Off-Ramp (Unchanged)

`POST /api/v1/bridge/off-ramp/initiate` — This existing endpoint remains for sending crypto from the user's **own wallet** (not Bridge-managed). The new payout endpoint is for sending from the **Bridge-managed wallet**.

### Check Transfer Status

`GET /api/v1/bridge/off-ramp/:transferId/status` — Works for both off-ramp types.

---

## Payment Rail Mapping

The system automatically maps the `accountType` stored in the database to Bridge's `payment_rail`:

| Account Type | Payment Rail | Currency | Country |
|---|---|---|---|
| `us` / `ach` | `ach` | USD | United States |
| `wire` | `wire` | USD | United States |
| `sepa` | `sepa` | EUR | Eurozone (IBAN countries) |
| `clabe` | `spei` | MXN | Mexico |
| `pix` | `pix` | BRL | Brazil |
| `gb` | `faster_payments` | GBP | United Kingdom |

This mapping lives in `BridgeOfframpHandler.PAYMENT_RAIL_MAP`.

---

## Webhook Handling

Transfer status updates are handled automatically by the existing webhook infrastructure:

**Endpoint:** `POST /api/v1/bridge/webhook`
**Event:** `transfer.updated`

When Bridge sends a `transfer.updated` webhook, `BridgeTransactionHandler.handleTransferUpdated()` updates the `BridgeTransaction` record. Terminal states (`PAYMENT_PROCESSED`, `FAILED`, `RETURNED`, `CANCELED`, `REFUNDED`) set the `completedAt` timestamp.

No changes were needed to webhook handling — it already handles all transfer types.

---

## Frontend Integration Guide

### Flow 1: User Adds a Payee

```
1. User taps "Add Payee"
2. User selects country/account type (US, Mexico, UK, Brazil, EU)
3. App shows the appropriate form fields:
   - Mexico: CLABE number (18 digits)
   - UK: Sort code + account number
   - Brazil: PIX key + CPF/CNPJ
   - US: Routing + account number
   - EU: IBAN + optional BIC
4. User fills in owner name, address, etc.
5. App calls POST /api/v1/bridge/external-accounts
6. App shows success with the last 4 digits
```

### Flow 2: User Sends a Payout

```
1. User taps "Send Money"
2. App calls GET /api/v1/bridge/external-accounts → shows payee list
3. User selects a payee (e.g., Maria in Mexico)
4. User enters amount and currency (e.g., "10,000 MXN")
5. App calls POST /api/v1/bridge/transfers/payout
6. App shows confirmation:
   - "Sending 10,000.00 MXN to Maria Garcia"
   - "595.23 USDC will be debited from your wallet"
7. User gets status updates via polling GET /api/v1/bridge/off-ramp/:transferId/status
   OR via push notifications triggered by webhooks
```

### Transfer States

| State | Meaning | User-Facing |
|---|---|---|
| `AWAITING_FUNDS` | Transfer created, waiting for funds | "Processing..." |
| `PAYMENT_SUBMITTED` | Bridge submitted to the payment network | "Payment sent" |
| `PAYMENT_PROCESSED` | Recipient received the funds | "Completed" |
| `FAILED` | Transfer failed | "Failed" |
| `RETURNED` | Funds returned (e.g., invalid account) | "Returned" |
| `CANCELED` | Transfer was canceled | "Canceled" |

---

## Bridge API Citations

| Feature | API Endpoint | Documentation |
|---|---|---|
| Create External Account | `POST /v0/customers/{id}/external_accounts` | [API Reference](https://apidocs.bridge.xyz/api-reference/external-accounts/create-a-new-external-account) |
| List External Accounts | `GET /v0/customers/{id}/external_accounts` | [API Reference](https://apidocs.bridge.xyz/api-reference/external-accounts/get-all-external-accounts) |
| Delete External Account | `DELETE /v0/customers/{id}/external_accounts/{ea_id}` | [API Reference](https://apidocs.bridge.xyz/api-reference/external-accounts/delete-a-single-external-account-object) |
| Create Transfer | `POST /v0/transfers` | [API Reference](https://apidocs.bridge.xyz/api-reference/transfers/create-a-transfer) |
| Fixed-Output Transfers | — | [Integration Guide](https://apidocs.bridge.xyz/get-started/guides/move-money/fixed_outputs_integration_guide) |
| Off-Ramp Guide | — | [Off-Ramp Liquidation](https://apidocs.bridge.xyz/get-started/guides/move-money/offramp_liquidation) |
| Transfer States | — | [Transfer States](https://apidocs.bridge.xyz/platform/orchestration/transfers/transfer-states) |
| Webhook Events | `POST /webhook` | [Webhook Structure](https://apidocs.bridge.xyz/platform/additional-information/webhooks/structure) |
| Rail-Specific Details | — | [Rail Specific](https://apidocs.bridge.xyz/platform/orchestration/more/rail-specific) |
| Supported Countries | — | [Supported Countries](https://apidocs.bridge.xyz/platform/customers/compliance/supported-countries-list) |
| Create Bridge Wallet | `POST /v0/customers/{id}/wallets` | [API Reference](https://apidocs.bridge.xyz/api-reference/bridge-wallets/create-a-bridge-wallet) |
| Wallet On-Ramp | — | [Wallet On-Ramp Guide](https://apidocs.bridge.xyz/get-started/guides/wallets/onramp) |

---

## Files Modified

| File | Change |
|---|---|
| `src/modules/bridge/dto/create-external-account.dto.ts` | Added CLABE, PIX, GB account type enums and nested DTOs |
| `src/modules/bridge/dto/initiate-payout.dto.ts` | **New file** — DTO for wallet-based payout |
| `src/modules/bridge/dto/index.ts` | Added export for `InitiatePayoutDto` |
| `src/modules/bridge/providers/bridge.provider.ts` | Updated `createTransfer` for fixed-output; updated `createExternalAccount` for CLABE/PIX/GB |
| `src/modules/bridge/handlers/bridge-offramp.handler.ts` | Updated `createExternalAccount` for new types; added `initiatePayout` method |
| `src/modules/bridge/bridge.service.ts` | Added `initiatePayout` facade method |
| `src/modules/bridge/bridge.controller.ts` | Added `POST /transfers/payout` endpoint |

**No changes to:** Database schema, migrations, webhook handlers, KYC flow, module registration, or any other existing functionality.
