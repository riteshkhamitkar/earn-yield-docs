# Yield Module - Complete Implementation Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Database Schema](#database-schema)
4. [API Endpoints](#api-endpoints)
5. [Services & Handlers](#services--handlers)
6. [External API Integrations](#external-api-integrations)
7. [Flow Diagrams](#flow-diagrams)
8. [Configuration](#configuration)

---

## Overview

The Yield Module enables users to deposit assets into YO Protocol vaults (yoUSD, yoETH, yoBTC, yoEUR, yoGOLD) and redeem yield-bearing tokens. It leverages:
- **OneBalance V3**: For cross-chain atomic execution
- **YO Protocol Smart Contracts**: For on-chain vault interactions
- **YO Protocol REST API**: For live APY, TVL, and user data

### Supported Vaults

| Vault | Underlying Asset | Network | Chain ID |
|-------|-----------------|---------|----------|
| yoUSD | USDC (Base) | base | 8453 |
| yoETH | WETH (Base) | base | 8453 |
| yoBTC | cbBTC (Base) | base | 8453 |
| yoEUR | EURC (Base) | base | 8453 |
| yoGOLD | XAUt (Ethereum) | ethereum | 1 |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        YieldModule                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐                                           │
│  │ YieldController  │  ← HTTP Endpoints                         │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │  YieldService    │  ← Orchestrator (delegates to handlers)   │
│  └────────┬─────────┘                                           │
│           │                                                      │
│     ┌─────┴─────┬─────────────┬─────────────┐                   │
│     ▼           ▼             ▼             ▼                   │
│ ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────────────────┐   │
│ │ YoQuote │ │YoExec   │ │ YoStatus  │ │YieldPositionsService│   │
│ │ Handler │ │Handler  │ │ Handler   │ │                    │   │
│ └────┬────┘ └────┬────┘ └─────┬─────┘ └────────┬───────────┘   │
│      │           │            │                 │               │
│      └───────────┴────────────┴─────────────────┘               │
│                          │                                       │
│             ┌────────────┴────────────┐                         │
│             ▼                         ▼                         │
│    ┌─────────────────┐      ┌─────────────────┐                │
│    │ YoGatewayService│      │   YoApiService  │                │
│    │ (Smart Contract)│      │   (REST API)    │                │
│    └────────┬────────┘      └────────┬────────┘                │
│             │                        │                          │
└─────────────┼────────────────────────┼──────────────────────────┘
              │                        │
              ▼                        ▼
    ┌─────────────────┐      ┌─────────────────┐
    │  YO Gateway     │      │  api.yo.xyz     │
    │  (On-Chain RPC) │      │  (REST API)     │
    └─────────────────┘      └─────────────────┘
```

---

## Database Schema

### Table 1: `Token` (Existing - Stores Underlying Assets)

Underlying assets (USDC, WETH, cbBTC, etc.) are stored in the existing `Token` table because:
- They can be used for swaps via OneBalance
- They have pricing information (`priceApiId`)
- Single source of truth for asset metadata

```prisma
model Token {
  id               String   @id @default(cuid())
  symbol           String   @unique  // "USDC_base", "WETH_base"
  name             String
  logoUrl          String
  group            String   // "assets"
  category         String   // "crypto"
  assetId          String   @unique  // CAIP-19: "eip155:8453/erc20:0x833589..."
  decimals         Int
  supportedForSwap Boolean  @default(true)
  priceApiId       String?  // CoinGecko ID for pricing
  // ... other fields
}
```

### Table 2: `YieldToken` (New - Vault Token Metadata)

Stores yield-bearing vault tokens (yoUSD, yoETH, etc.) separately because:
- They are NOT swappable (different lifecycle)
- They need vault-specific metadata
- They link to underlying assets via `underlyingAssetId`

```prisma
model YieldToken {
  id                String    @id @default(cuid())
  symbol            String    @unique  // "yoUSD", "yoETH"
  name              String             // "YO Vault USD"
  logoUrl           String
  assetId           String    @unique  // CAIP-19: "eip155:8453/erc20:0x0000000f..."
  decimals          Int                // Matches underlying (6 for yoUSD)
  vaultAddress      String    @unique  // "0x0000000f2eb9..."
  underlyingAssetId String             // Links to Token.assetId
  chainId           Int                // 8453 (Base) or 1 (Ethereum)
  network           String             // "base" or "ethereum"
  apy               Decimal?           // Cached APY from API
  apyUpdatedAt      DateTime?
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
}
```

### Table 3: `YieldDeposit` (Transaction Record)

```prisma
model YieldDeposit {
  id            String       @id @default(cuid())
  userId        String
  vaultSymbol   String       // "yoUSD"
  vaultAddress  String
  depositAsset  String       // "USDC_base"
  depositAmount String       // Amount in smallest unit
  sharesMinted  String       // Estimated shares
  minSharesOut  String       // Slippage protection
  partnerId     String?
  transactionId String?      @unique
  status        String       @default("PENDING")
  createdAt     DateTime     @default(now())
  updatedAt     DateTime     @updatedAt
  completedAt   DateTime?
  metadata      Json?
  User          User         @relation(...)
  Transaction   Transaction? @relation(...)
}
```

### Table 4: `YieldRedemption` (Transaction Record)

```prisma
model YieldRedemption {
  id                  String       @id @default(cuid())
  userId              String
  vaultSymbol         String       // "yoUSD"
  vaultAddress        String
  sharesBurned        String       // Shares to redeem
  expectedAsset       String       // "USDC_base"
  expectedAssetAmount String       // Estimated amount out
  minAssetOut         String       // Slippage protection
  transactionId       String?      @unique
  status              String       @default("PENDING")
  createdAt           DateTime     @default(now())
  updatedAt           DateTime     @updatedAt
  completedAt         DateTime?
  metadata            Json?
  User                User         @relation(...)
  Transaction         Transaction? @relation(...)
}
```

---

## API Endpoints

### Public Endpoints (No Auth)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/yield/vaults` | List all vaults with live APY |

### Protected Endpoints (Requires `AuthGuard`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/yield/positions` | Get user's yield positions with earnings |
| `GET` | `/yield/positions/:vaultSymbol/history` | Get deposit/withdrawal history for vault |
| `GET` | `/yield/points` | Get user's YO points |
| `POST` | `/yield/deposit/quote` | Get deposit quote |
| `POST` | `/yield/deposit/execute` | Execute deposit |
| `POST` | `/yield/redeem/quote` | Get redemption quote |
| `POST` | `/yield/redeem/execute` | Execute redemption |
| `GET` | `/yield/status/:quoteId` | Poll transaction status |

### Request/Response DTOs

#### `YieldDepositQuoteDto`
```typescript
{
  vaultSymbol: string;  // "yoUSD"
  amount: string;       // "100.50"
  slippage?: number;    // 50 = 0.5% (default)
}
```

#### `YieldDepositExecuteDto`
```typescript
{
  quoteId: string;              // "yo_dep_uuid..."
  signedChainOperation: string; // JSON string of signed operation
  tamperProofSignature: string; // From quote response
}
```

#### Quote Response
```typescript
{
  quoteId: string;
  expiresAt: string;           // ISO date
  from: { asset: string; amount: string };
  to: { asset: string; amount: string };
  chainOperation: object;      // For client to sign
  tamperProofSignature: string;
}
```

---

## Services & Handlers

### 1. `YieldService` (Orchestrator)
**File:** `services/yield.service.ts`

Simple delegation layer that routes calls to appropriate handlers:

| Method | Delegates To | Purpose |
|--------|-------------|---------|
| `requestDepositQuote()` | `YoQuoteHandler` | Generate deposit quote |
| `executeDeposit()` | `YoExecutionHandler` | Execute deposit |
| `requestRedemptionQuote()` | `YoQuoteHandler` | Generate redemption quote |
| `executeRedemption()` | `YoExecutionHandler` | Execute redemption |
| `getStatus()` | `YoStatusHandler` | Poll transaction status |
| `listYieldTokens()` | `YieldPositionsService` | Get vaults with live APY |
| `getUserPositions()` | `YieldPositionsService` | Get user positions |
| `getUserVaultHistory()` | `YieldPositionsService` | Get user history |
| `getUserPoints()` | `YieldPositionsService` | Get user YO points |

---

### 2. `YoQuoteHandler` (Quote Generation)
**File:** `handlers/yo-quote.handler.ts`

Handles the first step of the 2-step OneBalance flow.

#### `generateDepositQuote(userId, dto)`

```
1. Fetch user wallet + vault config + underlying token from DB
2. Convert amount to smallest unit (wei)
3. Call YO Gateway RPC: quoteConvertToShares() → get estimated shares
4. Apply slippage: minSharesOut = shares * (1 - slippage%)
5. Build calldata: gateway.deposit(vault, amount, minSharesOut, receiver, partnerId)
6. Call OneBalance: POST /v3/quote/prepare-call-quote
   - accounts: EVM accounts from wallet
   - targetChain: eip155:8453 (Base)
   - calls: [{ to: gateway, data: calldata }]
   - tokensRequired: [{ assetType: USDC, amount: wei }]
   - allowanceRequirements: [{ assetType: USDC, spender: gateway }]
   - fromAssetId: "ob:usdc" (aggregated asset)
7. Cache quote with 15-minute expiry
8. Return: quoteId, chainOperation, tamperProofSignature
```

#### `generateRedemptionQuote(userId, dto)`

Same flow but:
- Calls `quoteConvertToAssets()` instead
- Builds `redeem()` calldata
- `tokensRequired` is the vault token (yoUSD)

---

### 3. `YoExecutionHandler` (Transaction Execution)
**File:** `handlers/yo-execution.handler.ts`

Handles the second step - executing the signed transaction.

#### `executeDeposit(userId, dto)` / `executeRedemption(userId, dto)`

```
1. Validate quote: exists, not expired, belongs to user, signature matches
2. Call OneBalance: POST /v3/quote/call-quote
   - id: quoteId
   - chainOperation: JSON.parse(signedChainOperation)
   - tamperProofSignature
3. Call OneBalance: POST /v3/quote/execute-quote
   - id: quoteId
   - originChainsOperations: from step 2
   - destinationChainOperation: from step 2
   - tamperProofSignature
4. Create Transaction record (status: EXECUTING)
5. Create YieldDeposit/YieldRedemption record
6. Return: transactionId, success message
```

---

### 4. `YoStatusHandler` (Status Polling)
**File:** `handlers/yo-status.handler.ts`

Polls OneBalance for transaction status updates.

#### `getStatus(userId, quoteId)`

```
1. Find transaction by quoteId
2. If terminal (COMPLETED/FAILED), return cached status
3. Call OneBalance: GET /v3/status/{quoteId}
4. Map status: SUCCESS → COMPLETED, FAILURE → FAILED
5. Update Transaction + YieldDeposit/YieldRedemption records
6. Return formatted response with txHash if available
```

---

### 5. `YieldPositionsService` (User Data & APY)
**File:** `services/yield-positions.service.ts`

Manages user positions and live vault data.

#### `getLiveApyData()`

```
1. Fetch all vaults from YieldToken table
2. For each vault:
   - Call YO API: GET /api/v1/vault/{network}/{vaultAddress}
   - Extract: apy (1d/7d/30d), tvl, sharePrice, protocols
   - Fall back to DB values if API unavailable
3. Return array of VaultApyInfo
```

**Caching:** 5-minute TTL for vault snapshots

#### `getUserPositions(userId)`

```
1. Get user's wallet address
2. For each vault:
   - Call OneBalance: GET /v3/balances/aggregated-balance
     - Get user's yoToken balance
   - Call YO Gateway RPC: quoteConvertToAssets() → current value
   - Call YO API: GET /api/v1/history/user/{network}/{vault}/{user}
     - Calculate total deposited/withdrawn
   - Calculate unrealized earnings: currentValue - netDeposited
3. Return positions with earnings data
```

#### `getUserPoints(userId)`

```
1. Get user's wallet address
2. Call YO API: GET /api/v1/users/wallet/{address}/points
3. Return points data
```

---

### 6. `YoGatewayService` (Smart Contract Interaction)
**File:** `services/yo-gateway.service.ts`

Handles all on-chain interactions with YO Gateway contract.

#### Contract ABI (4 functions):
```solidity
function deposit(address yoVault, uint256 assetAmount, uint256 minSharesOut, address receiver, uint256 partnerId) returns (uint256)
function redeem(address yoVault, uint256 shares, uint256 minAssetsOut, address receiver, address owner) returns (uint256)
function quoteConvertToShares(address yoVault, uint256 assets) view returns (uint256)
function quoteConvertToAssets(address yoVault, uint256 shares) view returns (uint256)
```

| Method | Purpose |
|--------|---------|
| `buildDepositCalldata()` | Encode deposit call |
| `buildRedeemCalldata()` | Encode redeem call |
| `quoteConvertToShares()` | RPC: assets → shares |
| `quoteConvertToAssets()` | RPC: shares → assets |
| `getPartnerId()` | Get configured partner ID |

**Address Checksumming:** All addresses are lowercased then checksummed via `ethers.getAddress()` to prevent checksum errors.

---

### 7. `YoApiService` (YO REST API Client)
**File:** `services/yo-api.service.ts`

HTTP client for YO Protocol REST API (https://api.yo.xyz).

| Method | API Endpoint | Purpose |
|--------|-------------|---------|
| `getVaultSnapshot()` | `GET /api/v1/vault/{network}/{vaultAddress}` | TVL, APY, protocols |
| `getUserHistory()` | `GET /api/v1/history/user/{network}/{vault}/{user}` | Deposit/withdrawal history |
| `getUserPendingRedemptions()` | `GET /api/v1/vault/pending-redeems/{network}/{vault}/{user}` | Pending redemptions |
| `getUserPoints()` | `GET /api/v1/users/wallet/{address}/points` | User points |

---

## External API Integrations

### 1. OneBalance V3 API

| Endpoint | Method | Used In | Purpose |
|----------|--------|---------|---------|
| `/v3/quote/prepare-call-quote` | POST | `YoQuoteHandler` | Generate unsigned quote |
| `/v3/quote/call-quote` | POST | `YoExecutionHandler` | Submit signed operation |
| `/v3/quote/execute-quote` | POST | `YoExecutionHandler` | Execute the transaction |
| `/v3/status/{quoteId}` | GET | `YoStatusHandler` | Poll transaction status |
| `/v3/balances/aggregated-balance` | GET | `YieldPositionsService` | Get user token balances |

### 2. YO Protocol API (api.yo.xyz)

| Endpoint | Method | Used In | Purpose |
|----------|--------|---------|---------|
| `/api/v1/vault/{network}/{vaultAddress}` | GET | `YoApiService` | Vault snapshot (APY, TVL) |
| `/api/v1/history/user/{network}/{vault}/{user}` | GET | `YoApiService` | User history |
| `/api/v1/vault/pending-redeems/{network}/{vault}/{user}` | GET | `YoApiService` | Pending redemptions |
| `/api/v1/users/wallet/{address}/points` | GET | `YoApiService` | User points |

### 3. YO Gateway RPC (On-Chain)

| Function | Used In | Purpose |
|----------|---------|---------|
| `quoteConvertToShares()` | `YoGatewayService` | Get shares estimate for deposit |
| `quoteConvertToAssets()` | `YoGatewayService` | Get assets estimate for redemption |

---

## Flow Diagrams

### Deposit Flow (2 Steps)

```
┌──────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
│  Client  │         │  Anzo    │         │OneBalance│         │YO Gateway│
└────┬─────┘         └────┬─────┘         └────┬─────┘         └────┬─────┘
     │                    │                    │                    │
     │ POST /deposit/quote│                    │                    │
     │ {vaultSymbol, amt} │                    │                    │
     │───────────────────>│                    │                    │
     │                    │                    │                    │
     │                    │ quoteConvertToShares(vault, amount)     │
     │                    │─────────────────────────────────────────>
     │                    │                    │      shares        │
     │                    │<─────────────────────────────────────────
     │                    │                    │                    │
     │                    │ POST prepare-call-quote                 │
     │                    │ {accounts, calls, tokensRequired}       │
     │                    │───────────────────>│                    │
     │                    │    chainOperation  │                    │
     │                    │<───────────────────│                    │
     │                    │                    │                    │
     │ {quoteId, chainOp} │                    │                    │
     │<───────────────────│                    │                    │
     │                    │                    │                    │
     │ [CLIENT SIGNS chainOperation with Turnkey]                   │
     │                    │                    │                    │
     │ POST /deposit/execute                   │                    │
     │ {quoteId, signedOp}│                    │                    │
     │───────────────────>│                    │                    │
     │                    │ POST call-quote    │                    │
     │                    │ {id, signedOp}     │                    │
     │                    │───────────────────>│                    │
     │                    │  originChainOps    │                    │
     │                    │<───────────────────│                    │
     │                    │                    │                    │
     │                    │ POST execute-quote │                    │
     │                    │ {id, originOps}    │                    │
     │                    │───────────────────>│                    │
     │                    │    {success}       │                    │
     │                    │<───────────────────│                    │
     │                    │                    │                    │
     │ {transactionId}    │                    │                    │
     │<───────────────────│                    │                    │
     │                    │                    │                    │
     │ GET /status/:id    │                    │                    │
     │───────────────────>│                    │                    │
     │                    │ GET /v3/status/:id │                    │
     │                    │───────────────────>│                    │
     │                    │<───────────────────│                    │
     │ {status, txHash}   │                    │                    │
     │<───────────────────│                    │                    │
```

---

## Configuration

### Environment Variables

```env
# YO Protocol Configuration
YO_PARTNER_ID=0                    # Partner ID for rewards
YO_GATEWAY_BASE=0x...              # YO Gateway on Base
YO_GATEWAY_ETH=0x...               # YO Gateway on Ethereum
YO_API_URL=https://api.yo.xyz     # YO REST API

# RPC URLs (for on-chain quotes)
BASE_RPC_URL=https://base.rpc...
ETHEREUM_RPC_URL=https://eth.rpc...

# OneBalance (existing)
ONEBALANCE_API_KEY=...
```

### Config Module (`src/config/configuration.ts`)

```typescript
yo: {
  partnerId: process.env.YO_PARTNER_ID || '0',
  gateway: {
    base: process.env.YO_GATEWAY_BASE,
    eth: process.env.YO_GATEWAY_ETH,
  },
  apiUrl: process.env.YO_API_URL || 'https://api.yo.xyz',
}
```

---

## Module Dependencies

```typescript
@Module({
  imports: [
    AuthModule,      // For AuthGuard
    DatabaseModule,  // For PrismaService
    SwapsModule,     // For OneBalanceProvider
  ],
  controllers: [YieldController],
  providers: [
    YieldService,
    YoGatewayService,
    YoApiService,
    YieldPositionsService,
    YoQuoteHandler,
    YoExecutionHandler,
    YoStatusHandler,
  ],
  exports: [YieldService, YieldPositionsService],
})
export class YieldModule {}
```

---

## Error Handling

All handlers throw `BadRequestException` for user errors:

| Error | Cause |
|-------|-------|
| "User EVM wallet not found" | User has no Ethereum address |
| "Invalid vault symbol" | Vault not in YieldToken table |
| "Underlying asset not configured" | Token table missing underlying |
| "YO Gateway not configured" | Missing env variable |
| "Quote not found or expired" | Quote ID invalid or > 15 min old |
| "Quote does not belong to this user" | User ID mismatch |
| "Signature mismatch" | tamperProofSignature changed |

---

## File Summary

```
src/modules/yield/
├── dto/
│   ├── yield-deposit.dto.ts      # Quote & Execute DTOs for deposit
│   └── yield-redeem.dto.ts       # Quote & Execute DTOs for redemption
├── handlers/
│   ├── yo-quote.handler.ts       # OneBalance prepare-call-quote
│   ├── yo-execution.handler.ts   # OneBalance call-quote + execute-quote
│   └── yo-status.handler.ts      # Status polling & DB updates
├── services/
│   ├── yield.service.ts          # Orchestrator
│   ├── yo-gateway.service.ts     # Smart contract calldata & RPC
│   ├── yo-api.service.ts         # YO REST API client
│   └── yield-positions.service.ts # User positions & live APY
├── yield.controller.ts           # HTTP endpoints
└── yield.module.ts               # NestJS module definition
```

---

## Seeding Script

**File:** `prisma/scripts/populate-yield-tokens.ts`

Run: `npm run db:populate-yield`

This script:
1. Upserts underlying assets (USDC_base, WETH_base, etc.) into `Token` table
2. Upserts vault tokens (yoUSD, yoETH, etc.) into `YieldToken` table

All addresses are lowercase for consistent checksumming.

