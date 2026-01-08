# Yield Module - Complete Implementation Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Performance Optimizations](#performance-optimizations)
4. [Database Schema](#database-schema)
5. [API Endpoints](#api-endpoints)
6. [Services & Handlers](#services--handlers)
7. [External API Integrations](#external-api-integrations)
8. [Flow Diagrams](#flow-diagrams)
9. [Configuration](#configuration)
10. [API Testing Guide](#api-testing-guide)

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

## Performance Optimizations

### ✅ Implemented Optimizations (Latest Update)

#### 1. **Parallelized Operations**
- **`getUserPositions()`**: Changed from sequential `for` loop to `Promise.all()` for parallel vault position fetching
- **`getPositionForVault()`**: Parallelized 4 independent API calls (currentValue, history, snapshot, pendingRedeem)
- **`getUserVaultHistory()`**: Parallelized user and vault DB queries
- **`executeYieldOperation()`**: Parallelized call-quote with transaction creation, and execute-quote with yield record creation

**Time Complexity Improvement**: O(n) sequential → O(1) parallel for n vaults

#### 2. **Database Query Optimization**
- **`getVaultAndWallet()`**: Parallelized user and vault queries instead of sequential
- **`getUserPoints()`**: Optimized to use `select` instead of `include` (fetches only required fields)
- **`updateTransaction()`**: Uses Prisma `$transaction` for atomic updates, eliminates unnecessary re-fetch

**Query Reduction**: 2 sequential queries → 1 parallel query, eliminated 1 re-fetch

#### 3. **Code Deduplication**
- **`executeDeposit()` & `executeRedemption()`**: Refactored into single `executeYieldOperation()` method
- **`getCachedQuote()`**: Combined expiration checks to reduce redundant operations

**Lines Reduced**: ~30 lines of duplicate code eliminated

#### 4. **Caching Improvements**
- **`YoGatewayService`**: Added contract instance caching to avoid recreating contracts on every call
- **`getVaultSnapshot()`**: Already uses 5-minute TTL cache (maintained)

**Performance**: Reduced object creation overhead, faster repeated calls

#### 5. **Memory Optimization**
- **`updateTransaction()`**: Updates transaction object in memory instead of re-fetching from DB

**DB Calls Reduced**: Eliminated 1 unnecessary DB query per status update

---

## Database Schema

### Table 1: `YieldToken` (Self-Contained Vault Metadata)

Stores yield-bearing vault tokens (yoUSD, yoETH, etc.) with ALL necessary underlying asset metadata:
- Contains vault-specific metadata (address, chainId, APY)
- Contains underlying asset metadata (symbol, decimals, CAIP-19, aggregated ID)
- **NO dependency on Token table** - fully self-contained
- OneBalance handles cross-chain routing via `underlyingAggregatedAssetId`

```prisma
model YieldToken {
  id                          String    @id @default(cuid())
  symbol                      String    @unique  // "yoUSD", "yoETH"
  name                        String             // "YO Vault USD"
  logoUrl                     String
  assetId                     String    @unique  // CAIP-19: "eip155:8453/erc20:0x0000000f..."
  decimals                    Int                // Matches underlying (6 for yoUSD) - ERC-4626 standard
  vaultAddress                String    @unique  // "0x0000000f2eb9..."
  underlyingAssetId           String             // CAIP-19 for contract calls: "eip155:8453/erc20:0x833589..."
  underlyingSymbol            String             // Display symbol: "USDC", "ETH", "cbBTC"
  underlyingAggregatedAssetId String             // OneBalance aggregated: "ob:usdc", "ob:eth"
  chainId                     Int                // 8453 (Base) or 1 (Ethereum)
  network                     String             // "base" or "ethereum"
  apy                         Decimal?           // Cached APY from API
  apyUpdatedAt                DateTime?
  createdAt                   DateTime  @default(now())
  updatedAt                   DateTime  @updatedAt
}
```

**Key Fields Explained:**
- `underlyingAssetId`: The CAIP-19 format used in `tokensRequired` and `allowanceRequirements` for the exact contract on the target chain
- `underlyingSymbol`: Human-readable symbol for UI display (e.g., "USDC", "ETH")
- `underlyingAggregatedAssetId`: Used in `fromAssetId` for OneBalance to route from ANY chain the user has the token on

### Table 2: `YieldDeposit` (Transaction Record)

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

### Table 3: `YieldRedemption` (Transaction Record)

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

### ⚠️ Important: API Base Path

All endpoints are prefixed with `/api/v1` (global prefix set in `main.ts`).

### Public Endpoints (No Auth)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/yield/vaults` | List all vaults with live APY |

### Protected Endpoints (Requires `AuthGuard`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/yield/positions` | Get user's yield positions with earnings |
| `GET` | `/api/v1/yield/positions/:vaultSymbol/history` | Get deposit/withdrawal history for vault |
| `GET` | `/api/v1/yield/points` | Get user's YO points |
| `POST` | `/api/v1/yield/deposit/quote` | Get deposit quote |
| `POST` | `/api/v1/yield/deposit/execute` | Execute deposit |
| `POST` | `/api/v1/yield/redeem/quote` | Get redemption quote |
| `POST` | `/api/v1/yield/redeem/execute` | Execute redemption |
| `GET` | `/api/v1/yield/status/:quoteId` | Poll transaction status |

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
1. Fetch user wallet + vault config (parallelized queries)
2. Convert amount to smallest unit (wei)
3. Call YO Gateway RPC: quoteConvertToShares() → get estimated shares
4. Apply slippage: minSharesOut = shares * (1 - slippage%)
5. Build calldata: gateway.deposit(vault, amount, minSharesOut, receiver, partnerId)
6. Call OneBalance: POST /v3/quote/prepare-call-quote
   - accounts: EVM accounts from wallet (multi-account support)
   - targetChain: eip155:8453 (Base)
   - calls: [{ to: gateway, data: calldata, value: '0x0' }]
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
- `fromAssetId` is the vault token assetId

---

### 3. `YoExecutionHandler` (Transaction Execution)
**File:** `handlers/yo-execution.handler.ts`

Handles the second step - executing the signed transaction. **Optimized**: Unified deposit/redemption execution.

#### `executeDeposit(userId, dto)` / `executeRedemption(userId, dto)`

```
1. Validate quote: exists, not expired, belongs to user, signature matches
2. Parallelize:
   a. Call OneBalance: POST /v3/quote/call-quote
      - id: quoteId
      - chainOperation: JSON.parse(signedChainOperation)
      - tamperProofSignature
   b. Create Transaction record (status: EXECUTING)
3. Parallelize:
   a. Call OneBalance: POST /v3/quote/execute-quote
      - id: quoteId
      - originChainsOperations: from step 2a
      - destinationChainOperation: from step 2a
      - tamperProofSignature
   b. Create YieldDeposit/YieldRedemption record
4. Return: transactionId, success message
```

---

### 4. `YoStatusHandler` (Status Polling)
**File:** `handlers/yo-status.handler.ts`

Polls OneBalance for transaction status updates. **Optimized**: Eliminates unnecessary DB re-fetch.

#### `getStatus(userId, quoteId)`

```
1. Find transaction by quoteId
2. If terminal (COMPLETED/FAILED), return cached status
3. Call OneBalance: GET /v3/status/get-execution-status?quoteId={quoteId}
4. Map status: SUCCESS → COMPLETED, FAILURE → FAILED
5. If status changed:
   - Update Transaction + YieldDeposit/YieldRedemption records atomically
   - Update tx object in memory (no re-fetch needed)
6. Return formatted response with txHash if available
```

---

### 5. `YieldPositionsService` (User Data & APY)
**File:** `services/yield-positions.service.ts`

Manages user positions and live vault data. **Optimized**: Parallelized all independent operations.

#### `getLiveApyData()`

```
1. Fetch all vaults from YieldToken table
2. For each vault (parallelized):
   - Call YO API: GET /api/v1/vault/{network}/{vaultAddress}
   - Extract: apy (1d/7d/30d), tvl, sharePrice, protocols
   - Fall back to DB values if API unavailable
3. Return array of VaultApyInfo
```

**Caching:** 5-minute TTL for vault snapshots

#### `getUserPositions(userId)`

```
1. Get user's wallet address
2. Fetch all vaults
3. For each vault (parallelized with Promise.all):
   - Call OneBalance: GET /v3/balances/aggregated-balance
     - Get user's yoToken balance
   - Parallelize 4 API calls:
     a. YO Gateway RPC: quoteConvertToAssets() → current value
     b. YO API: GET /api/v1/history/user/{network}/{vault}/{user}
     c. Get vault snapshot (cached)
     d. YO API: GET /api/v1/vault/pending-redeems/{network}/{vault}/{user}
   - Calculate total deposited/withdrawn from history
   - Calculate unrealized earnings: currentValue - netDeposited
4. Filter positions with balance > 0
5. Return positions with earnings data
```

#### `getUserPoints(userId)`

```
1. Get user's wallet address (optimized query with select)
2. Call YO API: GET /api/v1/users/wallet/{address}/points
3. Return points data
```

---

### 6. `YoGatewayService` (Smart Contract Interaction)
**File:** `services/yo-gateway.service.ts`

Handles all on-chain interactions with YO Gateway contract. **Optimized**: Contract instance caching.

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

**Caching:** Contract instances are cached per chainId to avoid recreating them.

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

### 1. OneBalance V3 API ✅ Verified

**Base URL:** `https://be.onebalance.io/api`

| Endpoint | Method | Used In | Purpose | Status |
|----------|--------|---------|---------|--------|
| `/v3/quote/prepare-call-quote` | POST | `YoQuoteHandler` | Generate unsigned quote | ✅ Correct |
| `/v3/quote/call-quote` | POST | `YoExecutionHandler` | Submit signed operation | ✅ Correct |
| `/v3/quote/execute-quote` | POST | `YoExecutionHandler` | Execute the transaction | ✅ Correct |
| `/v3/status/get-execution-status` | GET | `YoStatusHandler` | Poll transaction status | ✅ Correct |
| `/v3/balances/aggregated-balance` | GET | `YieldPositionsService` | Get user token balances | ✅ Correct |

**Authentication:** All requests require `x-api-key` header with OneBalance API key.

**Multi-Account Support:** V3 endpoints support arrays of accounts (EVM + Solana) for cross-chain operations.

### 2. YO Protocol API (api.yo.xyz) ✅ Verified

**Base URL:** `https://api.yo.xyz`

| Endpoint | Method | Used In | Purpose | Status |
|----------|--------|---------|---------|--------|
| `/api/v1/vault/{network}/{vaultAddress}` | GET | `YoApiService` | Vault snapshot (APY, TVL) | ✅ Correct |
| `/api/v1/history/user/{network}/{vault}/{user}` | GET | `YoApiService` | User history | ✅ Correct |
| `/api/v1/vault/pending-redeems/{network}/{vault}/{user}` | GET | `YoApiService` | Pending redemptions | ✅ Correct |
| `/api/v1/users/wallet/{address}/points` | GET | `YoApiService` | User points | ✅ Correct |

**Response Format:** YO API returns `{ statusCode: 200, data: {...} }` wrapper.

### 3. YO Gateway RPC (On-Chain)

| Function | Used In | Purpose |
|----------|---------|---------|
| `quoteConvertToShares()` | `YoGatewayService` | Get shares estimate for deposit |
| `quoteConvertToAssets()` | `YoGatewayService` | Get assets estimate for redemption |

**RPC URLs:** Configured via `BASE_RPC_URL` and `ETHEREUM_RPC_URL` environment variables.

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
     │                    │ POST /v3/quote/prepare-call-quote       │
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
     │                    │ POST /v3/quote/call-quote               │
     │                    │ {id, signedOp}     │                    │
     │                    │───────────────────>│                    │
     │                    │  originChainOps    │                    │
     │                    │<───────────────────│                    │
     │                    │                    │                    │
     │                    │ POST /v3/quote/execute-quote            │
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
     │                    │ GET /v3/status/get-execution-status     │
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

## API Testing Guide

### Prerequisites

1. **Authentication**: All protected endpoints require JWT token in `Authorization` header
2. **User Setup**: User must have an Ethereum wallet address linked
3. **Vault Setup**: Run `npm run db:populate-yield` to seed vault tokens

### Testing Tools

- **Postman/Insomnia**: For manual API testing
- **cURL**: For command-line testing
- **Frontend Integration**: Test with actual Turnkey wallet signing

### Test Cases

#### 1. List Vaults (Public)

```bash
curl -X GET http://localhost:3000/api/v1/yield/vaults
```

**Expected Response:**
```json
[
  {
    "symbol": "yoUSD",
    "name": "YO Vault USD",
    "logoUrl": "...",
    "network": "base",
    "chainId": 8453,
    "vaultAddress": "0x...",
    "decimals": 6,
    "underlyingAssetId": "eip155:8453/erc20:0x833589fcd6edb6e08f4c7c32d4f71b54bda02913",
    "apy": 4.5,
    "apy7d": 4.3,
    "apy30d": 4.2,
    "tvl": "1000000",
    "sharePrice": "1.02",
    "protocols": [...],
    "lastUpdated": "2024-01-15T10:00:00Z"
  }
]
```

#### 2. Get User Positions

```bash
curl -X GET http://localhost:3000/api/v1/yield/positions \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Expected Response:**
```json
[
  {
    "vaultSymbol": "yoUSD",
    "vaultName": "YO Vault USD",
    "vaultAddress": "0x...",
    "network": "base",
    "chainId": 8453,
    "underlyingAsset": "USDC",
    "sharesBalance": "100.50",
    "currentValue": "102.51",
    "totalDeposited": "100.00",
    "unrealizedEarnings": "2.51",
    "earningsPercentage": "2.51",
    "apy": 4.5,
    "pendingRedemption": null
  }
]
```

#### 3. Get Deposit Quote

```bash
curl -X POST http://localhost:3000/api/v1/yield/deposit/quote \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vaultSymbol": "yoUSD",
    "amount": "100.50",
    "slippage": 50
  }'
```

**Expected Response:**
```json
{
  "quoteId": "yo_dep_uuid-...",
  "expiresAt": "2024-01-15T10:15:00Z",
  "from": {
    "asset": "USDC",
    "amount": "100.50"
  },
  "to": {
    "asset": "yoUSD",
    "amount": "98.52"
  },
  "chainOperation": {
    "userOp": {...},
    "typedDataToSign": {...}
  },
  "tamperProofSignature": "0x..."
}
```

**Next Steps:**
1. Sign `chainOperation.typedDataToSign` with Turnkey wallet
2. Call `/yield/deposit/execute` with signed operation

#### 4. Execute Deposit

```bash
curl -X POST http://localhost:3000/api/v1/yield/deposit/execute \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "quoteId": "yo_dep_uuid-...",
    "signedChainOperation": "{\"userOp\":{...},\"signature\":\"0x...\"}",
    "tamperProofSignature": "0x..."
  }'
```

**Expected Response:**
```json
{
  "quoteId": "yo_dep_uuid-...",
  "transactionId": "uuid-...",
  "success": true,
  "message": "Deposit submitted. Poll status for updates."
}
```

#### 5. Poll Status

```bash
curl -X GET http://localhost:3000/api/v1/yield/status/yo_dep_uuid-... \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Expected Response (Executing):**
```json
{
  "quoteId": "yo_dep_uuid-...",
  "transactionId": "uuid-...",
  "status": "EXECUTING",
  "type": "DEPOSIT",
  "fromAsset": "USDC",
  "toAsset": "yoUSD",
  "fromAmount": "100.50",
  "toAmount": "98.52",
  "transactionHash": null,
  "createdAt": "2024-01-15T10:00:00Z",
  "completedAt": null,
  "deposit": {
    "id": "uuid-...",
    "vaultSymbol": "yoUSD",
    "depositAmount": "100500000",
    "sharesMinted": "98520000",
    "status": "EXECUTING"
  },
  "oneBalanceStatus": {
    "status": "IN_PROGRESS",
    ...
  }
}
```

**Expected Response (Completed):**
```json
{
  "quoteId": "yo_dep_uuid-...",
  "transactionId": "uuid-...",
  "status": "COMPLETED",
  "transactionHash": "0x...",
  "completedAt": "2024-01-15T10:05:00Z",
  "deposit": {
    "status": "COMPLETED"
  }
}
```

#### 6. Get Redemption Quote

```bash
curl -X POST http://localhost:3000/api/v1/yield/redeem/quote \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vaultSymbol": "yoUSD",
    "shares": "98.52",
    "slippage": 50
  }'
```

#### 7. Execute Redemption

```bash
curl -X POST http://localhost:3000/api/v1/yield/redeem/execute \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "quoteId": "yo_red_uuid-...",
    "signedChainOperation": "{\"userOp\":{...},\"signature\":\"0x...\"}",
    "tamperProofSignature": "0x..."
  }'
```

#### 8. Get Vault History

```bash
curl -X GET http://localhost:3000/api/v1/yield/positions/yoUSD/history \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Expected Response:**
```json
[
  {
    "type": "deposit",
    "timestamp": 1705315200,
    "txHash": "0x...",
    "assetAmount": "100000000",
    "shareAmount": "98520000"
  },
  {
    "type": "withdrawal",
    "timestamp": 1705315800,
    "txHash": "0x...",
    "assetAmount": "50000000",
    "shareAmount": "49260000"
  }
]
```

#### 9. Get User Points

```bash
curl -X GET http://localhost:3000/api/v1/yield/points \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"
```

**Expected Response:**
```json
{
  "totalPoints": 1500,
  "rewardRates": [
    {
      "rate": "100",
      "desc": "Base rate per $1000 TVL"
    }
  ],
  "pointsEarnedYesterday": 50,
  "partnerMultiplier": 1.2,
  "partnerId": "0"
}
```

### Error Testing

#### Invalid Vault Symbol
```bash
curl -X POST http://localhost:3000/api/v1/yield/deposit/quote \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vaultSymbol": "INVALID",
    "amount": "100"
  }'
```

**Expected:** `400 Bad Request - Invalid vault symbol`

#### Expired Quote
```bash
# Wait 16 minutes after getting quote, then try to execute
curl -X POST http://localhost:3000/api/v1/yield/deposit/execute \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "quoteId": "yo_dep_old-quote-id",
    "signedChainOperation": "...",
    "tamperProofSignature": "..."
  }'
```

**Expected:** `400 Bad Request - Quote expired`

#### Insufficient Balance
```bash
curl -X POST http://localhost:3000/api/v1/yield/deposit/quote \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "vaultSymbol": "yoUSD",
    "amount": "999999999"
  }'
```

**Expected:** OneBalance API error - insufficient balance

### Performance Testing

Test parallel operations:

```bash
# Test parallel position fetching (should be fast)
time curl -X GET http://localhost:3000/api/v1/yield/positions \
  -H "Authorization: Bearer YOUR_JWT_TOKEN"

# Test parallel vault listing
time curl -X GET http://localhost:3000/api/v1/yield/vaults
```

---

## File Summary

```
src/modules/yield/
├── dto/
│   ├── yield-deposit.dto.ts      # Quote & Execute DTOs for deposit
│   └── yield-redeem.dto.ts       # Quote & Execute DTOs for redemption
├── handlers/
│   ├── yo-quote.handler.ts       # OneBalance prepare-call-quote (optimized)
│   ├── yo-execution.handler.ts   # OneBalance call-quote + execute-quote (optimized)
│   └── yo-status.handler.ts      # Status polling & DB updates (optimized)
├── services/
│   ├── yield.service.ts          # Orchestrator
│   ├── yo-gateway.service.ts     # Smart contract calldata & RPC (cached)
│   ├── yo-api.service.ts         # YO REST API client
│   └── yield-positions.service.ts # User positions & live APY (parallelized)
├── yield.controller.ts           # HTTP endpoints
└── yield.module.ts               # NestJS module definition
```

---

## Seeding Script

**File:** `prisma/scripts/populate-yield-tokens.ts`

Run: `npm run db:populate-yield`

This script upserts all vault tokens (yoUSD, yoETH, etc.) into the `YieldToken` table with:
- Vault metadata (address, chainId, network, APY)
- Underlying asset metadata (underlyingSymbol, underlyingAggregatedAssetId)

**Note:** Underlying assets (USDC_base, WETH_base, etc.) are **NOT** required in the Token table for yield operations. The YieldToken table is fully self-contained.

All addresses are lowercase for consistent checksumming.

---

## Verification Checklist

### ✅ OneBalance V3 API Integration
- [x] `/v3/quote/prepare-call-quote` - Correct endpoint and parameters
- [x] `/v3/quote/call-quote` - Correct endpoint and parameters
- [x] `/v3/quote/execute-quote` - Correct endpoint and parameters
- [x] `/v3/status/get-execution-status` - Correct endpoint with quoteId param
- [x] `/v3/balances/aggregated-balance` - Correct endpoint for balance checks
- [x] Multi-account support (accounts array) - Implemented
- [x] EIP-7702 delegation support - Implemented via account.util

### ✅ YO Protocol API Integration
- [x] `/api/v1/vault/{network}/{vaultAddress}` - Correct endpoint
- [x] `/api/v1/history/user/{network}/{vault}/{user}` - Correct endpoint
- [x] `/api/v1/vault/pending-redeems/{network}/{vault}/{user}` - Correct endpoint
- [x] `/api/v1/users/wallet/{address}/points` - Correct endpoint
- [x] Response parsing handles `{ statusCode, data }` wrapper

### ✅ YO Gateway Smart Contract
- [x] `deposit()` function signature matches
- [x] `redeem()` function signature matches
- [x] `quoteConvertToShares()` view function - Used correctly
- [x] `quoteConvertToAssets()` view function - Used correctly
- [x] Address checksumming - All addresses properly checksummed
- [x] Contract instance caching - Implemented

### ✅ Performance Optimizations
- [x] Parallelized `getUserPositions()` - Promise.all for all vaults
- [x] Parallelized `getPositionForVault()` - 4 API calls in parallel
- [x] Parallelized DB queries - User and vault queries
- [x] Code deduplication - Unified execution handler
- [x] Contract caching - Gateway contracts cached per chainId
- [x] Memory optimization - Eliminated unnecessary DB re-fetch

---

## Changelog

### Latest Update (2024-01-15)
- ✅ Parallelized all independent operations in `YieldPositionsService`
- ✅ Optimized DB queries with parallel execution
- ✅ Refactored duplicate code in `YoExecutionHandler`
- ✅ Added contract instance caching in `YoGatewayService`
- ✅ Eliminated unnecessary DB re-fetch in `YoStatusHandler`
- ✅ Reduced code by ~30 lines through deduplication
- ✅ Verified all OneBalance V3 API endpoints match documentation
- ✅ Verified all YO Protocol API endpoints match documentation
- ✅ Added comprehensive API testing guide

---

**Last Updated:** 2024-01-15  
**Version:** 2.0.0 (Optimized)
