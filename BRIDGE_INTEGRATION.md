# Bridge.xyz Integration Guide

## Overview

This document provides a comprehensive guide for the Bridge.xyz integration in the Anzo backend. Bridge.xyz enables fiat on-ramp (USD → Crypto) and off-ramp (Crypto → USD) functionality.

### Key Features

- **On-Ramp**: User deposits USD to a personal virtual bank account → Bridge converts to USDC → USDC sent to user's Anzo Solana wallet
- **Off-Ramp**: User sends USDC from their Anzo wallet → Bridge converts to USD → USD deposited to user's real bank account

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Frontend                                   │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Anzo Backend                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Bridge Module                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │   │
│  │  │ Controller  │  │   Service   │  │ Webhook Controller  │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────┘  │   │
│  │         │               │                    │               │   │
│  │         └───────────────┼────────────────────┘               │   │
│  │                         │                                    │   │
│  │  ┌─────────────────────────────────────────────────────┐    │   │
│  │  │              Bridge Provider (Axios)                 │    │   │
│  │  └─────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  Database (PostgreSQL)                       │   │
│  │  BridgeCustomer | BridgeWallet | BridgeVirtualAccount       │   │
│  │  BridgeExternalAccount | BridgeTransaction                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Bridge.xyz API                                │
│                   https://api.bridge.xyz/v0                         │
└─────────────────────────────────────────────────────────────────────┘
```

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

# Public key for webhook signature verification
# Get this from Bridge dashboard > Webhooks > Signing Keys
BRIDGE_WEBHOOK_PUBLIC_KEY="-----BEGIN PUBLIC KEY-----
MIIBIjAN...your-public-key...
-----END PUBLIC KEY-----"

# Developer fee percentage (optional, default: 0.5)
BRIDGE_DEVELOPER_FEE_PERCENT=0.5
```

## Database Schema

The integration adds these Prisma models:

- `BridgeCustomer` - Stores Bridge customer ID and KYC status
- `BridgeWallet` - Stores Bridge-managed intermediate wallet
- `BridgeVirtualAccount` - Stores virtual bank account for on-ramping
- `BridgeExternalAccount` - Stores user's real bank accounts for off-ramping
- `BridgeTransaction` - Tracks all Bridge transfers

Run migrations after updating schema:

```bash
npx prisma migrate dev --name add_bridge_integration
```

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

**Response:**
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

#### GET /api/v1/bridge/kyc/status

Get KYC status for the authenticated user.

**Response:**
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

### 2. Virtual Account (On-Ramp)

#### GET /api/v1/bridge/virtual-account

Get the user's virtual bank account details for on-ramping.

**Response:**
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
    "city": "San Francisco",
    "state": "CA",
    "postalCode": "94107",
    "country": "USA"
  }
}
```

**Response:**
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

#### GET /api/v1/bridge/external-accounts

List all external bank accounts.

**Response:**
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

#### DELETE /api/v1/bridge/external-accounts/:externalAccountId

Delete/deactivate an external bank account.

**Response:**
```json
{
  "success": true
}
```

### 4. Off-Ramp (Crypto → USD)

#### POST /api/v1/bridge/off-ramp/initiate

Initiate an off-ramp transfer.

**Request:**
```json
{
  "amount": "100.00",
  "sourceChain": "solana",
  "sourceCurrency": "usdc",
  "externalAccountId": "ea_abc123",
  "destinationPaymentRail": "ach"
}
```

**Response:**
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

#### GET /api/v1/bridge/off-ramp/:transferId/status

Get the status of an off-ramp transfer.

**Response:**
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

### 5. Transaction History

#### GET /api/v1/bridge/transactions

List all Bridge transactions.

**Query Parameters:**
- `type` (optional): `ON_RAMP_PAYOUT` or `OFF_RAMP`

**Response:**
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

### 6. Webhooks

#### POST /api/v1/bridge/webhook

**Internal endpoint** - Receives webhooks from Bridge.xyz.

This endpoint verifies the `X-Webhook-Signature` header and processes events:
- `customer.created` - New customer created
- `customer.updated` - KYC/TOS status changed (triggers account setup on approval)
- `transfer.created` - New transfer initiated
- `transfer.updated` - Transfer status changed
- `virtual_account.activity` - Funds received/sent

## Workflows

### On-Ramp Flow (USD → Crypto)

```
1. User calls POST /kyc/initiate
   ↓
2. User completes KYC via Bridge's hosted flow (kyc_link)
   ↓
3. User accepts TOS via Bridge's hosted flow (tos_link)
   ↓
4. Bridge sends customer.updated webhook with approved endorsement
   ↓
5. Backend automatically creates Bridge wallet + virtual account
   ↓
6. User calls GET /virtual-account to see deposit instructions
   ↓
7. User wires/ACHs USD to the virtual account
   ↓
8. Bridge converts USD to USDC automatically
   ↓
9. USDC appears in user's Anzo Solana wallet
```

### Off-Ramp Flow (Crypto → USD)

```
1. User calls POST /external-accounts to add their bank
   ↓
2. User calls POST /off-ramp/initiate with amount and bank
   ↓
3. Backend returns deposit address for USDC
   ↓
4. Frontend sends USDC to deposit address (signed via Turnkey)
   ↓
5. Bridge detects incoming USDC
   ↓
6. Bridge converts USDC to USD and sends ACH to user's bank
   ↓
7. Bridge sends transfer.updated webhook (payment_processed)
   ↓
8. User receives USD in their bank account
```

## Testing with Postman

### Collection Setup

1. Create a new Postman collection named "Anzo Bridge API"
2. Set base URL variable: `{{baseUrl}}` = `http://localhost:3000/api/v1`
3. Set auth token variable: `{{authToken}}` = Your JWT token

### Request Examples

Import these requests into Postman:

#### 1. Initiate KYC
```
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
```
GET {{baseUrl}}/bridge/kyc/status
Authorization: Bearer {{authToken}}
```

#### 3. Get Virtual Account
```
GET {{baseUrl}}/bridge/virtual-account
Authorization: Bearer {{authToken}}
```

#### 4. Add External Account
```
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

#### 5. List External Accounts
```
GET {{baseUrl}}/bridge/external-accounts
Authorization: Bearer {{authToken}}
```

#### 6. Initiate Off-Ramp
```
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

#### 7. Check Off-Ramp Status
```
GET {{baseUrl}}/bridge/off-ramp/{{transferId}}/status
Authorization: Bearer {{authToken}}
```

#### 8. List Transactions
```
GET {{baseUrl}}/bridge/transactions?type=OFF_RAMP
Authorization: Bearer {{authToken}}
```

## Webhook Testing

To test webhooks locally:

1. Use ngrok to expose your local server:
   ```bash
   ngrok http 3000
   ```

2. Configure the webhook URL in Bridge dashboard:
   ```
   https://your-ngrok-url.ngrok.io/api/v1/bridge/webhook
   ```

3. Copy the webhook public key from Bridge and add to `.env`

## Error Handling

All errors follow this format:

```json
{
  "statusCode": 400,
  "message": "Error description",
  "error": "Bad Request"
}
```

Common errors:
- `400` - Invalid request (missing fields, validation errors)
- `401` - Not authenticated
- `404` - Resource not found (no Bridge account, no virtual account, etc.)
- `409` - Conflict (user already has Bridge account)
- `500` - Internal server error (Bridge API failure)

## Security Considerations

1. **Webhook Verification**: All webhooks are verified using RSA-SHA256 signature
2. **Replay Prevention**: Webhooks older than 10 minutes are rejected
3. **API Authentication**: All user endpoints require JWT authentication
4. **Data Privacy**: Bank account numbers are stored with only last 4 digits visible

## Documentation References

- [Bridge.xyz API Docs](https://apidocs.bridge.xyz)
- [KYC Links](https://apidocs.bridge.xyz/get-started/introduction/quick-start/create-your-first-customer)
- [Virtual Accounts](https://apidocs.bridge.xyz/platform/orchestration/virtual_accounts/virtual-account)
- [Wallets](https://apidocs.bridge.xyz/get-started/guides/wallets/onramp)
- [External Accounts](https://apidocs.bridge.xyz/get-started/guides/move-money/offramp_liquidation)
- [Transfers](https://apidocs.bridge.xyz/get-started/guides/wallets/offramp)
- [Webhooks](https://apidocs.bridge.xyz/get-started/introduction/quick-start/setting-up-webhooks)
