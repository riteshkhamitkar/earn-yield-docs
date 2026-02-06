# Savings Module — API Reference & Testing Guide

## Overview

The **Savings Module** is a user-centric abstraction layer over the existing **Yield Module**. It allows users to create named savings goals (e.g., "Trip to Mexico", "Emergency Fund") backed by yield-bearing vault tokens like `yoUSD`.

### Architecture

```
Frontend  →  Savings Controller  →  SavingsService  →  Prisma (SavingsGoal)
          →  Yield Controller    →  YieldService     →  OneBalance V3 API → Yo Protocol
```

- **Goal creation/management** is handled by the Savings module.
- **Deposits/Withdrawals** flow through the existing Yield module, with an optional `goalId` parameter linking the transaction to a savings goal.
- **Activity filtering** via the existing `/balances/transactions` endpoint now supports `?goalId=` query param. All endpoints are prefixed with `/api/v1`.

---

## Base URL

```
http://localhost:3000/api/v1
```

> All endpoints below use this base URL prefix (set via `app.setGlobalPrefix('api/v1')` in `main.ts`).

## Authentication

All endpoints require a Bearer JWT token in the `Authorization` header:

```
Authorization: Bearer <your_jwt_token>
```

---

## 1. Savings Goal CRUD APIs

### 1.1 Create a Savings Goal

**`POST /api/v1/savings/goals`**

Creates a new savings goal for the authenticated user.

#### Request Body

| Field          | Type    | Required | Description                            |
|:-------------- |:------- |:-------- |:-------------------------------------- |
| `name`         | string  | ✅       | Goal name (unique per user)            |
| `icon`         | string  | ✅       | Emoji or icon identifier (e.g., "✈️") |
| `targetAmount` | number  | ✅       | Target savings amount (≥ 0)            |
| `currency`     | string  | ✅       | "USD" or "EUR"                         |
| `isDefault`    | boolean | ❌       | Mark as default goal (default: false)  |

#### Postman Example

```
POST http://localhost:3000/api/v1/savings/goals
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "name": "Trip to Mexico",
  "icon": "✈️",
  "targetAmount": 5000,
  "currency": "USD"
}
```

#### Response (201)

```json
{
  "id": "cm6abcdef0001...",
  "userId": "user_123",
  "name": "Trip to Mexico",
  "icon": "✈️",
  "targetAmount": "5000.00",
  "currency": "USD",
  "status": "ACTIVE",
  "isDefault": false,
  "yieldTokenSymbol": "yoUSD",
  "createdAt": "2026-02-06T09:00:00.000Z",
  "updatedAt": "2026-02-06T09:00:00.000Z"
}
```

#### Error (409 — Duplicate Name)

```json
{
  "statusCode": 409,
  "message": "Goal with name \"Trip to Mexico\" already exists",
  "error": "Conflict"
}
```

---

### 1.2 Update a Savings Goal

**`PATCH /api/v1/savings/goals/:goalId`**

Updates one or more fields of an existing goal. Only provide the fields you want to change.

#### URL Parameters

| Param    | Description          |
|:-------- |:-------------------- |
| `goalId` | The savings goal ID  |

#### Request Body (all optional)

| Field          | Type    | Description                                  |
|:-------------- |:------- |:-------------------------------------------- |
| `name`         | string  | New goal name                                |
| `icon`         | string  | New icon                                     |
| `targetAmount` | number  | New target amount                            |
| `status`       | string  | "ACTIVE", "COMPLETED", or "ARCHIVED"         |
| `isDefault`    | boolean | Set/unset as default                         |

#### Postman Example

```
PATCH http://localhost:3000/api/v1/savings/goals/cm6abcdef0001
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "targetAmount": 7500,
  "icon": "🏖️"
}
```

#### Response (200)

```json
{
  "id": "cm6abcdef0001",
  "name": "Trip to Mexico",
  "icon": "🏖️",
  "targetAmount": "7500.00",
  "currency": "USD",
  "status": "ACTIVE",
  "isDefault": false,
  "yieldTokenSymbol": "yoUSD",
  "updatedAt": "2026-02-06T09:05:00.000Z"
}
```

#### Error (404 — Not Found)

```json
{
  "statusCode": 404,
  "message": "Goal not found",
  "error": "Not Found"
}
```

---

### 1.3 Get a Single Goal

**`GET /api/v1/savings/goals/:goalId`**

Returns a single goal with its current contribution summary.

#### Postman Example

```
GET http://localhost:3000/api/v1/savings/goals/cm6abcdef0001
Authorization: Bearer {{token}}
```

#### Response (200)

```json
{
  "id": "cm6abcdef0001",
  "userId": "user_123",
  "name": "Trip to Mexico",
  "icon": "✈️",
  "targetAmount": "5000.00",
  "currency": "USD",
  "status": "ACTIVE",
  "isDefault": false,
  "yieldTokenSymbol": "yoUSD",
  "createdAt": "2026-02-06T09:00:00.000Z",
  "updatedAt": "2026-02-06T09:00:00.000Z",
  "currentAmount": 1200.50,
  "totalDeposited": 1500.00,
  "totalWithdrawn": 299.50
}
```

---

### 1.4 Savings Home Screen

**`GET /api/v1/savings/home`**

Returns the aggregated savings dashboard for the user: summary totals, all goals with contributions, and recent activity.

#### Postman Example

```
GET http://localhost:3000/api/v1/savings/home
Authorization: Bearer {{token}}
```

#### Response (200)

```json
{
  "summary": {
    "totalBalance": 3500.00,
    "totalEarned": 45.23,
    "baseApy": 9.0
  },
  "goals": [
    {
      "id": "cm6abcdef0001",
      "name": "Trip to Mexico",
      "icon": "✈️",
      "targetAmount": 5000.00,
      "currency": "USD",
      "status": "ACTIVE",
      "isDefault": false,
      "yieldTokenSymbol": "yoUSD",
      "currentAmount": 1200.50,
      "totalDeposited": 1500.00,
      "totalWithdrawn": 299.50,
      "apy": 9.0,
      "createdAt": "2026-02-06T09:00:00.000Z"
    },
    {
      "id": "cm6abcdef0002",
      "name": "Emergency Fund",
      "icon": "🛟",
      "targetAmount": 10000.00,
      "currency": "USD",
      "status": "ACTIVE",
      "isDefault": true,
      "yieldTokenSymbol": "yoUSD",
      "currentAmount": 2300.00,
      "totalDeposited": 2300.00,
      "totalWithdrawn": 0,
      "apy": 9.0,
      "createdAt": "2026-01-15T10:00:00.000Z"
    }
  ],
  "activityPreview": [
    {
      "id": "tx_uuid_1",
      "type": "DEPOSIT",
      "status": "COMPLETED",
      "fromAsset": "USDC",
      "toAsset": "yoUSD",
      "fromAmount": "500",
      "toAmount": "498.5",
      "createdAt": "2026-02-05T12:00:00.000Z",
      "savingsGoalId": "cm6abcdef0001",
      "savingsGoal": {
        "name": "Trip to Mexico",
        "icon": "✈️"
      }
    }
  ]
}
```

---

## 2. Deposit into a Savings Goal (via Yield Module)

The existing yield deposit flow is used. Simply add `goalId` to the **Step 1 quote request**. The `goalId` is cached server-side and automatically linked during execution (Step 3), preventing tampering.

### 2.1 Step 1: Get Deposit Quote

**`POST /api/v1/yield/deposit/quote`**

#### Request Body

| Field         | Type   | Required | Description                           |
|:------------- |:------ |:-------- |:------------------------------------- |
| `vaultSymbol` | string | ✅       | e.g., "yoUSD"                         |
| `amount`      | string | ✅       | Amount of underlying asset to deposit |
| `slippage`    | number | ❌       | Slippage in bps (default: 50 = 0.5%) |
| `goalId`      | string | ❌       | **Savings Goal ID** to link deposit   |

#### Postman Example

```
POST http://localhost:3000/api/v1/yield/deposit/quote
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "vaultSymbol": "yoUSD",
  "amount": "100",
  "goalId": "cm6abcdef0001"
}
```

#### Response (200)

```json
{
  "quoteId": "yo_dep_abc123...",
  "expiresAt": "2026-02-06T09:15:00.000Z",
  "from": { "asset": "USDC", "amount": "100" },
  "to": { "asset": "yoUSD", "amount": "99.85" },
  "chainOperation": { ... },
  "tamperProofSignature": "0x..."
}
```

### 2.2 Step 2: Call Quote (Sign Target)

**`POST /api/v1/yield/deposit/call-quote`**

```json
{
  "quoteId": "yo_dep_abc123...",
  "signedChainOperation": "<signed JSON string>",
  "tamperProofSignature": "0x..."
}
```

### 2.3 Step 3: Execute

**`POST /api/v1/yield/deposit/execute`**

```json
{
  "quoteId": "yo_dep_abc123...",
  "signedOriginOperations": [ ... ],
  "tamperProofSignature": "0x..."
}
```

> **Note:** The `goalId` from Step 1 is automatically persisted to `Transaction.savingsGoalId`, `YieldDeposit.savingsGoalId` during execution. No need to re-send it.

---

## 3. Redeem from a Savings Goal (via Yield Module)

Same pattern — add `goalId` to the redemption quote request.

### 3.1 Step 1: Get Redemption Quote

**`POST /api/v1/yield/redeem/quote`**

```
POST http://localhost:3000/api/v1/yield/redeem/quote
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "vaultSymbol": "yoUSD",
  "shares": "50",
  "goalId": "cm6abcdef0001"
}
```

Steps 2 and 3 follow the same pattern as deposit (call-quote → execute).

---

## 4. Activity Filtering by Goal

### `GET /api/v1/balances/transactions?goalId=<goalId>`

Returns only transactions linked to a specific savings goal. When `goalId` is provided, only database-stored transactions are returned (fast path, no external API calls).

#### Postman Example

```
GET http://localhost:3000/api/v1/balances/transactions?goalId=cm6abcdef0001&limit=20
Authorization: Bearer {{token}}
```

#### Response (200)

```json
{
  "transactions": [
    {
      "id": "tx_uuid_1",
      "quoteId": "ob_quote_xyz",
      "type": "DEPOSIT",
      "status": "COMPLETED",
      "from": { "asset": "USDC", "amount": "500", "network": "eip155:8453" },
      "to": { "asset": "yoUSD", "amount": "498.5", "network": "eip155:8453" },
      "fee": "$0.00",
      "timestamp": "2026-02-05T12:00:00.000Z",
      "completedAt": "2026-02-05T12:01:00.000Z",
      "transactionHash": "0xabc...",
      "service": "onebalance",
      "savingsGoal": { "name": "Trip to Mexico", "icon": "✈️" }
    }
  ],
  "total": 1,
  "continuation": null
}
```

---

## 5. Database Schema

### SavingsGoal Table

| Column             | Type          | Description                        |
|:------------------ |:------------- |:---------------------------------- |
| `id`               | String (cuid) | Primary key                        |
| `userId`           | String        | FK → User.id                       |
| `name`             | String        | Goal name (unique per user)        |
| `icon`             | String        | Emoji/icon identifier              |
| `targetAmount`     | Decimal(18,2) | Target savings amount              |
| `currency`         | String        | "USD" or "EUR"                     |
| `status`           | String        | ACTIVE / COMPLETED / ARCHIVED      |
| `isDefault`        | Boolean       | Default goal flag                  |
| `yieldTokenSymbol` | String        | Maps to YieldToken.symbol          |
| `createdAt`        | DateTime      | Auto                               |
| `updatedAt`        | DateTime      | Auto                               |

**Unique constraint:** `(userId, name)` — no duplicate goal names per user.

### New FK Columns Added

| Table             | Column          | Type    | Description                         |
|:----------------- |:--------------- |:------- |:----------------------------------- |
| `Transaction`     | `savingsGoalId` | String? | Nullable FK → SavingsGoal.id        |
| `YieldDeposit`    | `savingsGoalId` | String? | Nullable FK → SavingsGoal.id        |
| `YieldRedemption` | `savingsGoalId` | String? | Nullable FK → SavingsGoal.id        |

All FKs use `onDelete: SetNull` — archiving/deleting a goal does NOT delete transactions.

---

## 6. Postman Testing Workflow

### Step-by-Step Test Sequence

1. **Authenticate** — Get a JWT token via your auth flow.

2. **Create a Goal**:
   ```
   POST /api/v1/savings/goals
   Body: { "name": "Trip to Mexico", "icon": "✈️", "targetAmount": 5000, "currency": "USD" }
   ```
   → Save the returned `id` as `{{goalId}}`.

3. **Verify Home (empty state)**:
   ```
   GET /api/v1/savings/home
   ```
   → Should show the goal with `currentAmount: 0`.

4. **Deposit with Goal**:
   ```
   POST /api/v1/yield/deposit/quote
   Body: { "vaultSymbol": "yoUSD", "amount": "100", "goalId": "{{goalId}}" }
   ```
   → Follow the 3-step flow (quote → call-quote → execute).

5. **Verify Home (with deposit)**:
   ```
   GET /api/v1/savings/home
   ```
   → `currentAmount` should reflect the deposit.

6. **Filter Activity**:
   ```
   GET /api/v1/balances/transactions?goalId={{goalId}}
   ```
   → Should show only the deposit transaction linked to this goal.

7. **Update the Goal**:
   ```
   PATCH /api/v1/savings/goals/{{goalId}}
   Body: { "targetAmount": 7500 }
   ```

8. **Get Savings Details (general)**:
   ```
   GET /api/v1/savings/details
   ```
   → Returns performance, allocation, howItWorks, security, comparison.

9. **Get Savings Details (for a goal)**:
   ```
   GET /api/v1/savings/details?goalId={{goalId}}
   ```
   → Same structure, but vault is determined by the goal's currency.

10. **Get Single Goal**:
    ```
    GET /api/v1/savings/goals/{{goalId}}
    ```

11. **Archive the Goal**:
   ```
   PATCH /api/v1/savings/goals/{{goalId}}
   Body: { "status": "ARCHIVED" }
   ```

### Postman Environment Variables

| Variable  | Description                          |
|:--------- |:------------------------------------ |
| `baseUrl` | `http://localhost:3000/api/v1`       |
| `token`   | JWT auth token                       |
| `goalId`  | ID of a created savings goal         |

---

## 6B. Savings Details (Performance, Allocation, Educational)

### `GET /api/v1/savings/details?goalId=<optional>`

Returns detailed savings information: historical performance, provider info, protocol allocation breakdown, educational content, and yield comparison.

- If `goalId` is provided, details are shown for that goal's vault (e.g., `yoUSD` or `yoEUR`).
- If `goalId` is omitted, defaults to `yoUSD`.

#### Query Parameters

| Param    | Type   | Required | Description                                      |
|:-------- |:------ |:-------- |:------------------------------------------------ |
| `goalId` | string | ❌       | Savings goal ID. Omit for general account details |

#### Postman Example

```
GET http://localhost:3000/api/v1/savings/details
Authorization: Bearer {{token}}
```

Or for a specific goal:

```
GET http://localhost:3000/api/v1/savings/details?goalId=cm6abcdef0001
Authorization: Bearer {{token}}
```

#### Response (200)

```json
{
  "success": true,
  "data": {
    "pastPerformance": [
      { "label": "Last 7 days", "value": "7.09%" },
      { "label": "Last 30 days", "value": "9.09%" },
      { "label": "Daily yield", "value": "8.91%" }
    ],
    "about": [
      { "label": "Provider", "value": "YO Vault USD", "info": "The yield vault provider managing the underlying protocol allocations." },
      { "label": "Total value locked", "value": "20M", "info": "Total amount of funds deposited across all users in this vault." },
      { "label": "Performance fees", "value": "$0.00", "info": "No performance fees are charged on your savings." }
    ],
    "whereYourSavingsGo": [
      { "name": "Morpho", "percentage": "23.06%", "icon": "https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/morpho.png" },
      { "name": "Aave", "percentage": "18.23%", "icon": "https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/aave.png" }
    ],
    "howItWorks": [
      { "step": 1, "title": "Your money is allocated", "description": "We place your savings in trusted protocols that earn yield." },
      { "step": 2, "title": "Optimized automatically", "description": "We track risk-adjusted returns and move funds when needed." },
      { "step": 3, "title": "Withdraw anytime", "description": "Because bridging may be required, withdrawals can take up to 24h." }
    ],
    "security": [
      { "icon": "encrypted", "description": "Your private keys are secured by Turnkey..." },
      { "icon": "ar_on_you-1", "description": "Every transaction needs your approval. Face ID or Touch ID signs each action, so nothing moves without you." }
    ],
    "comparison": [
      { "name": "Avvio", "rate": "8.91%", "isAvvio": true },
      { "name": "Fintech", "rate": "1.5 - 3.75%" },
      { "name": "Traditional banking", "rate": "0 - 1%" }
    ]
  }
}
```

#### Data Source Matrix

| Response Field        | Source                                          | Type           |
|:----------------------|:------------------------------------------------|:---------------|
| `pastPerformance`     | YO Protocol API → `VaultSnapshot.apy/apy7d/apy30d` | **DYNAMIC** (live from YO API, cached 5 min) |
| `about.Provider`      | YO Protocol API → `VaultSnapshot.name`          | **DYNAMIC**    |
| `about.TVL`           | YO Protocol API → `VaultSnapshot.tvl`           | **DYNAMIC**    |
| `about.Performance fees` | Hardcoded `$0.00`                            | **STATIC**     |
| `whereYourSavingsGo`  | YO Protocol API → `VaultSnapshot.protocolStats` (top 5) | **DYNAMIC** |
| `whereYourSavingsGo[].icon` | YO Protocol API → protocol `icon`/`logo` field; falls back to `PROTOCOL_ICONS_FALLBACK` map | **DYNAMIC** (fallback STATIC) |
| `howItWorks`          | `HOW_IT_WORKS` constant in `savings.service.ts` | **STATIC**     |
| `security`            | `SECURITY` constant in `savings.service.ts`     | **STATIC**     |
| `comparison[Avvio].rate` | Live APY from vault (same as `pastPerformance` daily) | **DYNAMIC** |
| `comparison[others]`  | `COMPARISON_TEMPLATE` constant in `savings.service.ts` | **STATIC** |

> **To update static content**, edit the constants at the top of `src/modules/savings/savings.service.ts`.
> **Protocol icons** are fetched dynamically from the YO API. If the API doesn't return an icon for a protocol, the `PROTOCOL_ICONS_FALLBACK` map in `savings.service.ts` is used. Add new entries there for any new protocols without API icons.

---

## 7. Frontend Integration Guide

This section maps every screen / UI flow to the exact API calls, response fields, and TypeScript interfaces a frontend developer needs.

### 7.1 TypeScript Interfaces

```typescript
// ─── Goal ────────────────────────────────────────────────────────────────────
interface SavingsGoal {
  id: string;
  userId: string;
  name: string;
  icon: string;              // Emoji e.g. "✈️"
  targetAmount: string;      // Decimal string e.g. "5000.00"
  currency: string;          // "USD" | "EUR"
  status: "ACTIVE" | "COMPLETED" | "ARCHIVED";
  isDefault: boolean;
  yieldTokenSymbol: string;  // "yoUSD" | "yoEUR"
  createdAt: string;         // ISO 8601
  updatedAt: string;         // ISO 8601
}

// ─── Goal with contribution (GET /goals/:id) ────────────────────────────────
interface SavingsGoalDetail extends SavingsGoal {
  currentAmount: number;     // net deposited − withdrawn
  totalDeposited: number;
  totalWithdrawn: number;
}

// ─── Home Screen ─────────────────────────────────────────────────────────────
interface SavingsHomeResponse {
  summary: {
    totalBalance: number;    // Sum of all goals' net contributions
    totalEarned: number;     // Unrealized yield earnings
    baseApy: number | null;  // Current vault APY
  };
  goals: SavingsGoalHome[];
  activityPreview: ActivityItem[];
}

interface SavingsGoalHome {
  id: string;
  name: string;
  icon: string;
  targetAmount: number;
  currency: string;
  status: string;
  isDefault: boolean;
  yieldTokenSymbol: string;
  currentAmount: number;
  totalDeposited: number;
  totalWithdrawn: number;
  apy: number | null;
  createdAt: string;
}

interface ActivityItem {
  id: string;
  type: "DEPOSIT" | "REDEMPTION";
  status: "COMPLETED" | "EXECUTING" | "FAILED";
  fromAsset: string;
  toAsset: string;
  fromAmount: string;
  toAmount: string;
  createdAt: string;
  savingsGoalId: string | null;
  savingsGoal: { name: string; icon: string } | null;
}

// ─── Savings Details ─────────────────────────────────────────────────────────
interface SavingsDetailsResponse {
  success: boolean;
  data: {
    pastPerformance: { label: string; value: string }[];
    about: { label: string; value: string; info: string }[];
    whereYourSavingsGo: AllocationItem[];
    howItWorks: { step: number; title: string; description: string }[];
    security: { icon: string; description: string }[];
    comparison: ComparisonItem[];
  };
}

interface AllocationItem {
  name: string;
  percentage: string;       // e.g. "23.06%"
  icon: string | null;      // Protocol logo URL
}

interface ComparisonItem {
  name: string;
  rate: string;             // e.g. "8.91%" or "1.5 - 3.75%"
  isAvvio?: boolean;        // Only true for the Avvio row
}

// ─── Transaction History (filtered by goal) ──────────────────────────────────
interface TransactionHistoryResponse {
  transactions: TransactionItem[];
  total: number;
  continuation: string | null;  // Pass as query param for next page
}

interface TransactionItem {
  id: string;
  quoteId: string;
  type: "DEPOSIT" | "REDEMPTION";
  status: "COMPLETED" | "EXECUTING" | "FAILED";
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

| Screen / Feature | API Call | Key Response Fields |
|:---|:---|:---|
| **Savings Home** (dashboard) | `GET /api/v1/savings/home` | `summary.totalBalance`, `summary.totalEarned`, `summary.baseApy`, `goals[]`, `activityPreview[]` |
| **Goal Detail** page | `GET /api/v1/savings/goals/:goalId` | `currentAmount`, `totalDeposited`, `totalWithdrawn`, `targetAmount` (for progress bar) |
| **Savings Details** (performance page) | `GET /api/v1/savings/details` or `GET /api/v1/savings/details?goalId=X` | `pastPerformance`, `about`, `whereYourSavingsGo`, `howItWorks`, `security`, `comparison` |
| **Goal Activity** (transaction list) | `GET /api/v1/balances/transactions?goalId=X&limit=20` | `transactions[]`, `continuation` (for pagination) |
| **Create Goal** modal/screen | `POST /api/v1/savings/goals` | Returns the created `SavingsGoal` with `id` |
| **Edit Goal** modal/screen | `PATCH /api/v1/savings/goals/:goalId` | Returns the updated goal |
| **Archive Goal** action | `PATCH /api/v1/savings/goals/:goalId` with `{ "status": "ARCHIVED" }` | Returns updated goal with `status: "ARCHIVED"` |

### 7.3 Deposit Flow (Frontend Step-by-Step)

Depositing into a savings goal uses the existing Yield module — the frontend just passes `goalId` in the first step.

```
┌──────────────┐      ┌────────────────┐      ┌──────────────────┐
│  1. Quote     │ ───► │  2. Call-Quote  │ ───► │  3. Execute      │
│  (get terms)  │      │  (sign target)  │      │  (sign & submit) │
└──────────────┘      └────────────────┘      └──────────────────┘
```

```typescript
// Step 1: Request a deposit quote (include goalId)
const quoteRes = await fetch(`${BASE_URL}/api/v1/yield/deposit/quote`, {
  method: "POST",
  headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    vaultSymbol: "yoUSD",
    amount: "100",          // Amount in underlying asset (USDC)
    goalId: selectedGoalId, // ← Links this deposit to the savings goal
  }),
});
const quote = await quoteRes.json();
// quote.quoteId, quote.chainOperation.typedDataToSign, quote.tamperProofSignature

// Step 2: Sign the destination chain operation (client-side wallet signing)
const signedChainOp = await wallet.signTypedData(quote.chainOperation.typedDataToSign);

const callQuoteRes = await fetch(`${BASE_URL}/api/v1/yield/deposit/call-quote`, {
  method: "POST",
  headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    quoteId: quote.quoteId,
    signedChainOperation: signedChainOp,
    tamperProofSignature: quote.tamperProofSignature,
  }),
});
const callQuote = await callQuoteRes.json();
// callQuote.originChainsOperations — array of operations to sign

// Step 3: Sign origin operations and execute
const signedOps = await Promise.all(
  callQuote.originChainsOperations.map((op: any) =>
    wallet.signTypedData(op.typedDataToSign)
  )
);

const execRes = await fetch(`${BASE_URL}/api/v1/yield/deposit/execute`, {
  method: "POST",
  headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    quoteId: quote.quoteId,
    signedOriginOperations: signedOps,
    tamperProofSignature: callQuote.tamperProofSignature,
  }),
});
const result = await execRes.json();
// result.success === true → deposit submitted

// Step 4: Poll for completion
const pollStatus = async (quoteId: string) => {
  const res = await fetch(`${BASE_URL}/api/v1/yield/status/${quoteId}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const status = await res.json();
  // status.status === "COMPLETED" | "EXECUTING" | "FAILED"
  return status;
};
```

> **Important:** The `goalId` is ONLY sent in Step 1. The backend caches it server-side and automatically links the deposit to the goal during execution. This prevents goal-switching mid-transaction.

### 7.4 Withdrawal (Redeem) Flow

Identical to deposit but uses the redeem endpoints:

```typescript
// Step 1: Redemption quote (include goalId)
const quoteRes = await fetch(`${BASE_URL}/api/v1/yield/redeem/quote`, {
  method: "POST",
  headers: { Authorization: `Bearer ${token}`, "Content-Type": "application/json" },
  body: JSON.stringify({
    vaultSymbol: "yoUSD",
    shares: "50",            // Amount in vault shares
    goalId: selectedGoalId,  // ← Links this withdrawal to the savings goal
  }),
});
// Steps 2–4: Same pattern as deposit (call-quote → execute → poll)
```

### 7.5 UI Component → Data Mapping

#### Home Screen (Savings Dashboard)

| UI Element | Data Source (from `GET /api/v1/savings/home`) |
|:---|:---|
| Total balance card | `summary.totalBalance` |
| Total earned badge | `summary.totalEarned` |
| APY rate display | `summary.baseApy` (format: `${baseApy}%`) |
| Goal card — name | `goals[i].name` |
| Goal card — icon | `goals[i].icon` |
| Goal card — progress bar | `goals[i].currentAmount / goals[i].targetAmount * 100` |
| Goal card — amount text | `$${goals[i].currentAmount} / $${goals[i].targetAmount}` |
| Goal card — APY | `goals[i].apy` |
| Recent activity list | `activityPreview[]` (max 5 items) |
| Activity item — goal tag | `activityPreview[i].savingsGoal.name` + `.icon` |

#### Savings Details Screen

| UI Element | Data Source (from `GET /api/v1/savings/details`) |
|:---|:---|
| Performance chart / list | `data.pastPerformance[]` — render each `{label, value}` |
| About section | `data.about[]` — render each `{label, value, info}` (info = tooltip) |
| Allocation pie chart / bars | `data.whereYourSavingsGo[]` — `name`, `percentage`, `icon` |
| How it works steps | `data.howItWorks[]` — `step`, `title`, `description` |
| Security badges | `data.security[]` — `icon` (icon name), `description` |
| Comparison table | `data.comparison[]` — highlight row where `isAvvio === true` |

#### Goal Detail Screen

| UI Element | Data Source (from `GET /api/v1/savings/goals/:id`) |
|:---|:---|
| Goal name + icon | `name`, `icon` |
| Progress ring / bar | `currentAmount / targetAmount * 100` |
| Amount display | `currentAmount` (current), `targetAmount` (target) |
| Deposited total | `totalDeposited` |
| Withdrawn total | `totalWithdrawn` |
| Transaction list | Separate call: `GET /api/v1/balances/transactions?goalId=X` |

### 7.6 Pagination (Transaction History)

```typescript
// First page
const page1 = await fetchTransactions(goalId, 20);
// page1.continuation = "tx_abc123" (or null if no more)

// Next page (pass continuation)
if (page1.continuation) {
  const page2 = await fetchTransactions(goalId, 20, page1.continuation);
}

async function fetchTransactions(goalId: string, limit: number, continuation?: string) {
  const params = new URLSearchParams({ goalId, limit: String(limit) });
  if (continuation) params.set("continuation", continuation);

  const res = await fetch(`${BASE_URL}/api/v1/balances/transactions?${params}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
}
```

### 7.7 Error Handling

All API errors follow this structure:

```typescript
interface ApiError {
  statusCode: number;
  message: string;
  error: string;   // e.g. "Conflict", "Not Found", "Bad Request"
}
```

| Status Code | When It Happens | Frontend Action |
|:---|:---|:---|
| `400` | Invalid input (e.g. missing required field, bad vault symbol) | Show validation error |
| `401` | JWT expired or missing | Redirect to login |
| `404` | Goal not found (deleted or wrong ID) | Show "not found" state, refresh goals list |
| `409` | Duplicate goal name | Show "name already taken" error on create form |

### 7.8 Recommended Fetch Strategy

| Screen | Fetch Timing | Cache / Refetch |
|:---|:---|:---|
| Savings Home | On tab focus + pull-to-refresh | Refetch on every visit (data changes with deposits) |
| Goal Detail | On navigation to screen | Refetch on back-from-deposit |
| Savings Details | On navigation | Cache 5 min (data is already cached server-side) |
| Transaction History | On navigation + infinite scroll | Append pages via `continuation` |
| Create/Edit Goal | On form submit | Invalidate home cache after success |

---

## 8. Security Notes

- **No goal tampering mid-transaction**: The `goalId` is captured during `POST /api/v1/yield/deposit/quote` (Step 1) and stored in the server-side quote cache. Steps 2 and 3 read it from the cache — the user cannot switch goals mid-flow.
- **Non-blocking**: If `goalId` is omitted, all existing Yield flows work exactly as before (`savingsGoalId` is stored as `null`).
- **Cascade safety**: Deleting a user cascades to delete their savings goals. Deleting a goal sets `savingsGoalId` to `null` on linked records (no data loss).

---

## 9. Files Modified / Created

### New Files
| File | Purpose |
|:---- |:------- |
| `src/modules/savings/savings.module.ts` | NestJS module |
| `src/modules/savings/savings.controller.ts` | REST controller |
| `src/modules/savings/savings.service.ts` | Business logic |
| `src/modules/savings/dto/create-goal.dto.ts` | Create DTO with validation |
| `src/modules/savings/dto/update-goal.dto.ts` | Update DTO with validation |

### Modified Files
| File | Change |
|:---- |:------ |
| `prisma/schema.prisma` | Added `SavingsGoal` model; added `savingsGoalId` FK to `Transaction`, `YieldDeposit`, `YieldRedemption`; added relation on `User` |
| `src/app.module.ts` | Registered `SavingsModule` |
| `src/modules/yield/dto/yield-deposit.dto.ts` | Added optional `goalId` field |
| `src/modules/yield/dto/yield-redeem.dto.ts` | Added optional `goalId` field |
| `src/modules/yield/interfaces/yield.interfaces.ts` | Added `goalId` to `CachedQuote.dto` type |
| `src/modules/yield/handlers/yo-execution.handler.ts` | Persists `savingsGoalId` from cached DTO to `Transaction`, `YieldDeposit`, `YieldRedemption` |
| `src/modules/balances/balances.controller.ts` | Added `goalId` query param to `GET /api/v1/balances/transactions` |
| `src/modules/balances/balances.service.ts` | Passes `goalId` to handler |
| `src/modules/balances/handlers/transaction-history.handler.ts` | Added `getGoalTransactions()` fast path when `goalId` is provided |
