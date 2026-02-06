# Savings Module — API Documentation

> Base URL: `http://localhost:3000/api/v1`
> All endpoints require `Authorization: Bearer <jwt>` header.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Database Schema](#2-database-schema)
3. [API Endpoints](#3-api-endpoints)
   - 3A. [Create Goal](#3a-create-goal)
   - 3B. [Update Goal](#3b-update-goal)
   - 3C. [Get Goal](#3c-get-goal)
   - 3D. [Home Screen](#3d-home-screen)
   - 3E. [Savings Details](#3e-savings-details)
   - 3F. [Activity (filtered by goal)](#3f-activity-filtered-by-goal)
4. [Deposit & Withdraw into a Goal](#4-deposit--withdraw-into-a-goal)
5. [Data Source Matrix](#5-data-source-matrix)
6. [Protocol Icons](#6-protocol-icons)
7. [Frontend Integration Guide](#7-frontend-integration-guide)
8. [Postman Testing Workflow](#8-postman-testing-workflow)
9. [YO API Direct Testing](#9-yo-api-direct-testing)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  SavingsModule                                      │
│  ├── SavingsController  (REST endpoints)            │
│  ├── SavingsService     (business logic)            │
│  └── protocol-icons.ts  (icon URL map)              │
├─────────────────────────────────────────────────────┤
│  Depends on:                                        │
│  ├── YieldModule → YieldPositionsService (APY/TVL)  │
│  ├── YieldModule → YoExecutionHandler (goalId link) │
│  └── BalancesModule → transaction history filter    │
└─────────────────────────────────────────────────────┘
```

**Key files:**

| File | Purpose |
|------|---------|
| `src/modules/savings/savings.module.ts` | Module registration |
| `src/modules/savings/savings.controller.ts` | 5 endpoints |
| `src/modules/savings/savings.service.ts` | All business logic |
| `src/modules/savings/protocol-icons.ts` | Protocol → icon URL map |
| `src/modules/savings/dto/create-goal.dto.ts` | Create goal validation |
| `src/modules/savings/dto/update-goal.dto.ts` | Update goal validation |

---

## 2. Database Schema

### SavingsGoal model

```prisma
model SavingsGoal {
  id               String    @id @default(cuid())
  userId           String
  name             String
  icon             String
  targetAmount     Decimal   @db.Decimal(18, 2)
  currency         String
  status           String    @default("ACTIVE") // ACTIVE, COMPLETED, ARCHIVED
  isDefault        Boolean   @default(false)
  yieldTokenSymbol String    // "yoUSD" or "yoEUR"
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  User             User      @relation(...)
  Transaction      Transaction[]
  YieldDeposit     YieldDeposit[]
  YieldRedemption  YieldRedemption[]

  @@unique([userId, name])
  @@index([userId])
}
```

### Linked models (nullable FK)

| Model | Field | Behavior |
|-------|-------|----------|
| `Transaction` | `savingsGoalId String?` | `onDelete: SetNull` |
| `YieldDeposit` | `savingsGoalId String?` | `onDelete: SetNull` |
| `YieldRedemption` | `savingsGoalId String?` | `onDelete: SetNull` |

---

## 3. API Endpoints

### 3A. Create Goal

```
POST /api/v1/savings/goals
```

**Body:**

```json
{
  "name": "Trip to Mexico",
  "icon": "🏖️",
  "targetAmount": 5000,
  "currency": "USD",
  "isDefault": false
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | ✅ | Unique per user |
| `icon` | string | ✅ | Emoji or icon identifier |
| `targetAmount` | number | ✅ | ≥ 0 |
| `currency` | string | ✅ | `"USD"` or `"EUR"` |
| `isDefault` | boolean | ❌ | Default: `false` |

**Response:** The created `SavingsGoal` object.

**Errors:**
- `409 Conflict` — Goal with same name already exists.

---

### 3B. Update Goal

```
PATCH /api/v1/savings/goals/:goalId
```

**Body (all fields optional):**

```json
{
  "name": "Trip to Hawaii",
  "icon": "🌴",
  "targetAmount": 8000,
  "status": "COMPLETED",
  "isDefault": true
}
```

| Field | Type | Required | Allowed values |
|-------|------|----------|----------------|
| `name` | string | ❌ | — |
| `icon` | string | ❌ | — |
| `targetAmount` | number | ❌ | ≥ 0 |
| `status` | string | ❌ | `ACTIVE`, `COMPLETED`, `ARCHIVED` |
| `isDefault` | boolean | ❌ | — |

**Response:** The updated `SavingsGoal` object.

**Errors:**
- `404 Not Found` — Goal not found or not owned by user.

---

### 3C. Get Goal

```
GET /api/v1/savings/goals/:goalId
```

**Response:**

```json
{
  "id": "clx...",
  "name": "Trip to Mexico",
  "icon": "🏖️",
  "targetAmount": 5000,
  "currency": "USD",
  "status": "ACTIVE",
  "isDefault": false,
  "yieldTokenSymbol": "yoUSD",
  "createdAt": "2025-01-15T...",
  "updatedAt": "2025-01-15T...",
  "currentAmount": 1250.50,
  "totalDeposited": 1500.00,
  "totalWithdrawn": 249.50
}
```

`currentAmount`, `totalDeposited`, `totalWithdrawn` are computed from `YieldDeposit` and `YieldRedemption` records linked to this goal.

---

### 3D. Home Screen

```
GET /api/v1/savings/home
```

**Response:**

```json
{
  "summary": {
    "totalBalance": 3200.50,
    "totalEarned": 45.30,
    "baseApy": 6.31
  },
  "goals": [
    {
      "id": "clx...",
      "name": "Trip to Mexico",
      "icon": "🏖️",
      "targetAmount": 5000,
      "currency": "USD",
      "status": "ACTIVE",
      "isDefault": false,
      "yieldTokenSymbol": "yoUSD",
      "currentAmount": 1250.50,
      "totalDeposited": 1500.00,
      "totalWithdrawn": 249.50,
      "apy": 6.31,
      "createdAt": "2025-01-15T..."
    }
  ],
  "activityPreview": [
    {
      "id": "tx_...",
      "type": "YIELD_DEPOSIT",
      "status": "COMPLETED",
      "fromAsset": "USDC",
      "toAsset": "yoUSD",
      "fromAmount": "100",
      "toAmount": "93.8",
      "createdAt": "2025-01-15T...",
      "savingsGoalId": "clx...",
      "savingsGoal": { "name": "Trip to Mexico", "icon": "🏖️" }
    }
  ]
}
```

**Data sources:**
| Field | Source |
|-------|--------|
| `summary.baseApy` | YO API via `YieldPositionsService` (cached 5 min) |
| `summary.totalEarned` | YO API user positions |
| `goals[].currentAmount` | DB: `YieldDeposit` − `YieldRedemption` amounts |
| `activityPreview` | DB: last 5 transactions linked to any goal |

---

### 3E. Savings Details

```
GET /api/v1/savings/details
GET /api/v1/savings/details?goalId=clx...
```

**Query params:**

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `goalId` | string | ❌ | Filters vault by goal's `yieldTokenSymbol`. Defaults to `yoUSD`. |

**Response:**

```json
{
  "success": true,
  "data": {
    "pastPerformance": [
      { "label": "Last 7 days", "value": "6.66%" },
      { "label": "Last 30 days", "value": "7.05%" },
      { "label": "Daily yield", "value": "6.31%" }
    ],
    "about": [
      { "label": "Provider", "value": "YO Vault USD", "info": "The yield vault provider..." },
      { "label": "Total value locked", "value": "30272791.648344", "info": "Total amount..." },
      { "label": "Performance fees", "value": "$0.00", "info": "No performance fees..." }
    ],
    "whereYourSavingsGo": [
      { "name": "Auto", "percentage": "21.73%", "icon": "https://...auto.png" },
      { "name": "InfiniFi", "percentage": "9.13%", "icon": "https://...infinifi.png" },
      { "name": "Resolv", "percentage": "9.09%", "icon": "https://...resolv.png" },
      { "name": "Morpho", "percentage": "9.07%", "icon": "https://...morpho.png" },
      { "name": "Aave", "percentage": "8.36%", "icon": "https://...aave.png" }
    ],
    "howItWorks": [
      { "step": 1, "title": "Your money is allocated", "description": "..." },
      { "step": 2, "title": "Optimized automatically", "description": "..." },
      { "step": 3, "title": "Withdraw anytime", "description": "..." }
    ],
    "security": [
      { "icon": "encrypted", "description": "Your private keys are secured by Turnkey..." },
      { "icon": "ar_on_you-1", "description": "Every transaction needs your approval..." }
    ],
    "comparison": [
      { "name": "Avvio", "rate": "6.31%", "isAvvio": true },
      { "name": "Fintech", "rate": "1.5 - 3.75%" },
      { "name": "Traditional banking", "rate": "0 - 1%" }
    ]
  }
}
```

**Data sources per section:**

| Section | Source | Type |
|---------|--------|------|
| `pastPerformance` | YO API vault stats (`yield.1d`, `7d`, `30d`) | DYNAMIC |
| `about.Provider` | YO API vault name | DYNAMIC |
| `about.Total value locked` | YO API `stats.tvl.formatted` | DYNAMIC |
| `about.Performance fees` | Hardcoded `$0.00` | STATIC |
| `whereYourSavingsGo` | YO API `protocolStats` (top 5 by allocation, >0%) | DYNAMIC |
| `whereYourSavingsGo[].icon` | `protocol-icons.ts` map | STATIC |
| `howItWorks` | Hardcoded in service | STATIC |
| `security` | Hardcoded in service | STATIC |
| `comparison[0].rate` (Avvio) | YO API daily yield | DYNAMIC |
| `comparison[1-2]` | Hardcoded | STATIC |

---

### 3F. Activity (filtered by goal)

```
GET /api/v1/balances/transactions?goalId=clx...
```

When `goalId` is provided, the endpoint skips external API calls (OneBalance/Zerion) and returns only DB transactions linked to that goal.

**Query params:**

| Param | Type | Required |
|-------|------|----------|
| `goalId` | string | ❌ |
| `limit` | number | ❌ (default: 200) |
| `continuation` | string | ❌ |
| `currency` | string | ❌ |

**Response:**

```json
{
  "transactions": [
    {
      "id": "tx_...",
      "quoteId": "q_...",
      "type": "YIELD_DEPOSIT",
      "status": "COMPLETED",
      "from": { "asset": "USDC", "amount": "100", "network": "base" },
      "to": { "asset": "yoUSD", "amount": "93.8", "network": "base" },
      "fee": "$0.00",
      "timestamp": "2025-01-15T...",
      "completedAt": "2025-01-15T...",
      "transactionHash": "0x...",
      "service": "YO_PROTOCOL",
      "savingsGoal": { "name": "Trip to Mexico", "icon": "🏖️" }
    }
  ],
  "total": 1,
  "continuation": null
}
```

---

## 4. Deposit & Withdraw into a Goal

Deposits and withdrawals go through the existing **Yield Module** endpoints. The `goalId` is passed in step 1, cached server-side, and automatically linked to all DB records.

### Deposit flow

```
Step 1: POST /api/v1/yield/deposit/quote
        Body: { "vaultSymbol": "yoUSD", "amount": "100", "goalId": "clx..." }
        → Returns: quoteId, chainOperation to sign

Step 2: POST /api/v1/yield/deposit/call-quote
        Body: { "quoteId": "...", "signedChainOperation": "...", "tamperProofSignature": "..." }
        → Returns: originChainsOperations to sign

Step 3: POST /api/v1/yield/deposit/execute
        Body: { "quoteId": "...", "signedOriginOperations": [...], "tamperProofSignature": "..." }
        → Executes. Transaction, YieldDeposit records are created with savingsGoalId.
```

### Withdraw flow

```
Step 1: POST /api/v1/yield/redeem/quote
        Body: { "vaultSymbol": "yoUSD", "shares": "93.8", "goalId": "clx..." }

Step 2: POST /api/v1/yield/redeem/call-quote
        Body: { "quoteId": "...", "signedChainOperation": "...", "tamperProofSignature": "..." }

Step 3: POST /api/v1/yield/redeem/execute
        Body: { "quoteId": "...", "signedOriginOperations": [...], "tamperProofSignature": "..." }
```

> **Note:** `goalId` is only passed in step 1. It is cached server-side with the quote and automatically propagated to `Transaction`, `YieldDeposit`, and `YieldRedemption` records on execution.

---

## 5. Data Source Matrix

| Data point | Source | Caching |
|-----------|--------|---------|
| Vault APY (1d/7d/30d) | YO API `GET /api/v1/vault/{network}/{address}` → `stats.yield` | 5 min in-memory |
| TVL | YO API → `stats.tvl.formatted` | 5 min in-memory |
| Share price | YO API → `stats.sharePrice.formatted` | 5 min in-memory |
| Protocol allocations (top 5) | YO API → `stats.protocolStats` sorted by allocation desc, >0% only | 5 min in-memory |
| Protocol icons | `src/modules/savings/protocol-icons.ts` | Static file |
| Goal CRUD | Prisma → `SavingsGoal` table | None |
| Goal contributions | Prisma → `YieldDeposit` + `YieldRedemption` sum in code | None |
| Activity preview | Prisma → `Transaction` table | None |
| User positions / earnings | YO API + OneBalance | 5 min in-memory |
| howItWorks, security, comparison template | Hardcoded in `savings.service.ts` | Static |

---

## 6. Protocol Icons

Icons are managed in a single file: `src/modules/savings/protocol-icons.ts`

```typescript
export const PROTOCOL_ICONS: Record<string, string> = {
  morpho: 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/morpho.png',
  aave: 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/aave.png',
  auto: 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/auto.png',
  infinifi: 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/infinifi.png',
  resolv: 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/resolv.png',
  // ... full list in file
};
```

**To add a new protocol:** Add one line to this map. Key = protocol name in lowercase. No other code changes needed.

**Lookup logic** in `savings.service.ts`:
```typescript
icon: PROTOCOL_ICONS[(p.protocol || '').toLowerCase()] || null
```

**Why not from YO API?** The YO API does not return protocol icons. This map is the single source of truth.

---

## 7. Frontend Integration Guide

### 7.1 TypeScript Interfaces

```typescript
// Goal
interface SavingsGoal {
  id: string;
  name: string;
  icon: string;
  targetAmount: number;
  currency: string;
  status: 'ACTIVE' | 'COMPLETED' | 'ARCHIVED';
  isDefault: boolean;
  yieldTokenSymbol: string;
  currentAmount: number;
  totalDeposited: number;
  totalWithdrawn: number;
  apy: number | null;
  createdAt: string;
}

// Home screen
interface SavingsHome {
  summary: {
    totalBalance: number;
    totalEarned: number;
    baseApy: number | null;
  };
  goals: SavingsGoal[];
  activityPreview: Transaction[];
}

// Details screen
interface SavingsDetails {
  success: boolean;
  data: {
    pastPerformance: { label: string; value: string }[];
    about: { label: string; value: string; info: string }[];
    whereYourSavingsGo: { name: string; percentage: string; icon: string | null }[];
    howItWorks: { step: number; title: string; description: string }[];
    security: { icon: string; description: string }[];
    comparison: { name: string; rate: string; isAvvio?: boolean }[];
  };
}

// Transaction
interface Transaction {
  id: string;
  quoteId: string;
  type: string;
  status: string;
  from: { asset: string; amount: string; network: string };
  to: { asset: string; amount: string; network: string };
  fee: string;
  timestamp: string;
  completedAt: string | null;
  transactionHash: string | null;
  service: string;
  savingsGoal: { name: string; icon: string } | null;
}
```

### 7.2 Screen → API Mapping

| Screen | API Call | When to fetch |
|--------|----------|---------------|
| Savings home | `GET /api/v1/savings/home` | On screen load + pull-to-refresh |
| Goal detail | `GET /api/v1/savings/goals/:goalId` | On tap into a goal |
| Savings details / info | `GET /api/v1/savings/details?goalId=...` | On tap "Details" / "Learn more" |
| Goal activity | `GET /api/v1/balances/transactions?goalId=...` | On tap "Activity" inside a goal |

### 7.3 Deposit Flow (3-step signing)

```
1. POST /api/v1/yield/deposit/quote  { vaultSymbol, amount, goalId }
   → Get quoteId + chainOperation

2. Sign chainOperation with user's key (Turnkey)

3. POST /api/v1/yield/deposit/call-quote  { quoteId, signedChainOperation, tamperProofSignature }
   → Get originChainsOperations[]

4. Sign each originChainsOperation with user's key

5. POST /api/v1/yield/deposit/execute  { quoteId, signedOriginOperations, tamperProofSignature }
   → Done. Transaction created with goalId linked.
```

Withdraw flow is identical but uses `/api/v1/yield/redeem/*` endpoints with `shares` instead of `amount`.

### 7.4 Progress Bar Calculation

```typescript
const progress = goal.currentAmount / goal.targetAmount;
const percentage = Math.min(progress * 100, 100);
```

### 7.5 Error Handling

| HTTP Code | Meaning | Action |
|-----------|---------|--------|
| 401 | Token expired | Refresh JWT and retry |
| 404 | Goal not found | Navigate back |
| 409 | Duplicate goal name | Show "name already exists" |

---

## 8. Postman Testing Workflow

### Setup

Set environment variable:
```
baseUrl = http://localhost:3000/api/v1
```

All requests need header:
```
Authorization: Bearer <jwt_token>
```

### Step-by-step

**1. Create a goal**
```
POST {{baseUrl}}/savings/goals
Body:
{
  "name": "Emergency Fund",
  "icon": "🚨",
  "targetAmount": 10000,
  "currency": "USD"
}
→ Save response.id as goalId
```

**2. Get home screen**
```
GET {{baseUrl}}/savings/home
→ See the goal in goals[], summary shows totalBalance
```

**3. Get goal details**
```
GET {{baseUrl}}/savings/goals/{{goalId}}
→ currentAmount, totalDeposited, totalWithdrawn
```

**4. Deposit into goal**
```
POST {{baseUrl}}/yield/deposit/quote
Body:
{
  "vaultSymbol": "yoUSD",
  "amount": "100",
  "goalId": "{{goalId}}"
}
→ Follow 3-step signing flow (see Section 4)
```

**5. Check savings details**
```
GET {{baseUrl}}/savings/details
→ pastPerformance, whereYourSavingsGo, howItWorks, etc.

GET {{baseUrl}}/savings/details?goalId={{goalId}}
→ Same but scoped to goal's vault
```

**6. View goal activity**
```
GET {{baseUrl}}/balances/transactions?goalId={{goalId}}
→ Only transactions for this goal
```

**7. Update goal**
```
PATCH {{baseUrl}}/savings/goals/{{goalId}}
Body:
{
  "targetAmount": 15000,
  "status": "COMPLETED"
}
```

---

## 9. YO API Direct Testing

To test the YO API directly (no auth needed):

```
GET https://api.yo.xyz/api/v1/vault/base/0x0000000f2eb9f69274678c76222b35eec7588a65
```

**Key response fields used by our backend:**

| Field | Used for |
|-------|----------|
| `data.stats.yield["1d"]` | Daily APY → `pastPerformance`, `comparison[0].rate` |
| `data.stats.yield["7d"]` | 7-day APY → `pastPerformance` |
| `data.stats.yield["30d"]` | 30-day APY → `pastPerformance` |
| `data.stats.tvl.formatted` | TVL → `about[1].value` |
| `data.name` | Vault name → `about[0].value` |
| `data.stats.protocolStats[].protocol` | Protocol name → `whereYourSavingsGo[].name` |
| `data.stats.protocolStats[].allocation.formatted` | Allocation → `whereYourSavingsGo[].percentage` |

**Note:** The YO API does **not** return protocol icons. Icons come from `src/modules/savings/protocol-icons.ts`.
