# Savings Internal Transfer & Delete Goal — Frontend Integration Guide

> **Base URL:** `http://localhost:3000/api/v1`  
> **Auth:** All endpoints require `Authorization: Bearer <jwt>` header.

This document describes the **off-chain** features for moving funds between Savings Goals and General Savings, and for deleting (archiving) goals. **No on-chain transactions or gas fees** are involved.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Internal Transfer API](#2-internal-transfer-api)
3. [Delete Goal API](#3-delete-goal-api)
4. [Response Changes (GET /home, GET /goals/:id)](#4-response-changes-get-home-get-goalsid)
5. [Calculation Logic (How Balances Work)](#5-calculation-logic-how-balances-work)
6. [All Scenarios & Examples](#6-all-scenarios--examples)
7. [Frontend Integration Checklist](#7-frontend-integration-checklist)
8. [Error Handling](#8-error-handling)
9. [TypeScript Interfaces](#9-typescript-interfaces)

---

## 1. Overview

### What's New

| Feature | Description | Endpoint |
|---------|-------------|----------|
| **Internal Transfer** | Instantly move funds between two Savings Goals, or between a Goal and General Savings | `POST /savings/transfer` |
| **Delete Goal** | Soft-delete a goal (archive it) and automatically move its balance to General Savings | `DELETE /savings/goals/:goalId` |

### Key Concepts

- **General Savings:** The portion of your on-chain vault balance **not** assigned to any goal.  
  `General Savings = Total On-Chain Vault Value - Sum of All Goal Balances`
- **Off-chain:** These operations only update our internal ledger. No blockchain transactions, no gas.
- **Instant:** Transfers complete immediately. No waiting for confirmation.
- **Soft Delete:** Deleted goals are archived (`status: "ARCHIVED"`). They remain in the DB for transaction history but no longer appear in the active goals list.

---

## 2. Internal Transfer API

### Endpoint

```
POST /api/v1/savings/transfer
```

### Request Body

```json
{
  "fromGoalId": "clx123...",   // Optional. Omit = transfer FROM General Savings
  "toGoalId": "clx456...",     // Optional. Omit = transfer TO General Savings
  "amount": 50.00,
  "currency": "USD"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `fromGoalId` | string | No* | Source goal ID. Omit when transferring **from** General Savings. |
| `toGoalId` | string | No* | Destination goal ID. Omit when transferring **to** General Savings. |
| `amount` | number | Yes | Amount in **USD** (min 0.01) |
| `currency` | string | Yes | `"USD"` or `"EUR"` — identifies the underlying vault (yoUSD or yoEUR) |

**Constraint:** At least one of `fromGoalId` or `toGoalId` must be provided. You cannot transfer within the same goal.

### Success Response (200)

```json
{
  "success": true,
  "message": "Transfer completed successfully"
}
```

### Validation Rules (Backend Enforced)

1. `fromGoalId` and `toGoalId` cannot both be omitted.
2. `fromGoalId` and `toGoalId` cannot be the same.
3. Source must have sufficient balance (goal or General Savings).
4. Source goal (if any) must use the same currency (vault) as requested.
5. `amount` must be ≥ 0.01.

---

## 3. Delete Goal API

### Endpoint

```
DELETE /api/v1/savings/goals/:goalId
```

### Success Response

**204 No Content** — Empty body on success.

### Behavior

1. **If the goal has a balance > $0.01:** An internal transfer is executed first: `fromGoalId: goalId`, `toGoalId: null` (General Savings).
2. **Then:** The goal's `status` is set to `"ARCHIVED"`.
3. **Result:** The goal disappears from `GET /savings/home` → `goals[]` (only ACTIVE goals are returned). Its former balance appears in General Savings.

### Errors

- **404 Not Found:** Goal does not exist, or does not belong to the user, or is already archived.

---

## 4. Response Changes (GET /home, GET /goals/:id)

### GET /api/v1/savings/home

**Unchanged fields:** `summary`, `savingsAccounts`, `goals` structure.

**Changes:**

1. **`goals`** — Only returns goals with `status: "ACTIVE"`. Archived goals are excluded.
2. **`goals[].currentAmount`** — Now **includes** internal transfer adjustments (deposits and redemptions from internal moves).
3. **`activityPreview`** — Now includes **internal transfer** entries (in addition to on-chain YIELD_DEPOSIT / YIELD_WITHDRAWAL).

#### Activity Preview: Internal Transfer Entry Shape

```json
{
  "id": "clx789...",
  "type": "INTERNAL_TRANSFER_IN",
  "status": "COMPLETED",
  "fromAsset": "yoUSD",
  "toAsset": "yoUSD",
  "fromAmount": "50",
  "toAmount": "50",
  "createdAt": "2025-02-24T12:00:00.000Z",
  "transactionHash": null,
  "explorerUrl": null,
  "savingsGoal": { "name": "Vacation Fund", "icon": "🏖️" },
  "isInternal": true,
  "relatedGoalName": "General Savings"
}
```

For `INTERNAL_TRANSFER_OUT`, the shape is similar but `type` is `"INTERNAL_TRANSFER_OUT"`.

**Note:** One logical transfer creates **two** child records (IN + OUT). Both may appear in `activityPreview`. Frontend may choose to:
- Display both as separate line items, or
- Group by parent and show one consolidated "Transfer" row using `relatedGoalName` for context.

### GET /api/v1/savings/goals/:goalId

**Changes:**

- **`totalDeposited`** — Includes amounts from `INTERNAL_TRANSFER_IN` transactions.
- **`totalWithdrawn`** — Includes amounts from `INTERNAL_TRANSFER_OUT` transactions.
- **`currentAmount`** — `Math.max(0, totalDeposited - totalWithdrawn)` — now reflects both on-chain and internal movements.

---

## 5. Calculation Logic (How Balances Work)

### Goal Balance (`currentAmount`)

```
goal.currentAmount =
  (sum of on-chain DEPOSIT amounts linked to this goal)
  - (sum of on-chain REDEMPTION amounts linked to this goal)
  + (sum of INTERNAL_TRANSFER_IN amounts for this goal)
  - (sum of INTERNAL_TRANSFER_OUT amounts for this goal)
```

All amounts are in vault token units (yoUSD / yoEUR), then converted to USD for display using live exchange rates where needed.

### General Savings Balance

```
General Savings (per vault) =
  (Total On-Chain Vault Value for that vault)
  - (Sum of currentAmount for all ACTIVE goals using that vault)
```

- Total on-chain vault value comes from YO API / positions.
- Goal amounts include both on-chain and internal transfer adjustments.
- General Savings is a **remainder**: whatever is in the vault but not assigned to any goal.

### Internal Transfer Record Structure (Backend)

Each transfer creates:

1. **Parent Transaction** — `type: "INTERNAL_TRANSFER"` (not surfaced in activity preview).
2. **Child 1 — OUT** — `type: "INTERNAL_TRANSFER_OUT"`, `savingsGoalId` = source goal (or `null` if from General).
3. **Child 2 — IN** — `type: "INTERNAL_TRANSFER_IN"`, `savingsGoalId` = destination goal (or `null` if to General).

Metadata on children includes `relatedGoalName` (the other side of the transfer) for UI display.

---

## 6. All Scenarios & Examples

### Scenario A: Transfer $50 from Goal A to Goal B

**Request:**
```json
POST /api/v1/savings/transfer
{
  "fromGoalId": "goal_A_id",
  "toGoalId": "goal_B_id",
  "amount": 50,
  "currency": "USD"
}
```

**Result:**
- Goal A: `currentAmount` decreases by $50
- Goal B: `currentAmount` increases by $50
- General Savings: unchanged

---

### Scenario B: Transfer $10 from General Savings to Goal C

**Request:**
```json
POST /api/v1/savings/transfer
{
  "fromGoalId": null,   // or omit
  "toGoalId": "goal_C_id",
  "amount": 10,
  "currency": "USD"
}
```

**Result:**
- General Savings: decreases by $10
- Goal C: `currentAmount` increases by $10

---

### Scenario C: Transfer $20 from Goal D to General Savings

**Request:**
```json
POST /api/v1/savings/transfer
{
  "fromGoalId": "goal_D_id",
  "toGoalId": null,     // or omit
  "amount": 20,
  "currency": "USD"
}
```

**Result:**
- Goal D: `currentAmount` decreases by $20
- General Savings: increases by $20

---

### Scenario D: Delete Goal A (balance $5)

**Request:**
```
DELETE /api/v1/savings/goals/goal_A_id
```

**Backend behavior:**
1. Fetches goal A's `currentAmount` ($5)
2. Executes internal transfer: $5 from goal A → General Savings
3. Sets goal A's `status` to `"ARCHIVED"`

**Result:**
- Goal A: no longer appears in `goals[]`
- General Savings: increases by $5

---

### Scenario E: Delete Goal with Zero Balance

**Request:**
```
DELETE /api/v1/savings/goals/goal_E_id
```

**Backend behavior:**
- Skips internal transfer (balance &lt; $0.01)
- Sets goal E's `status` to `"ARCHIVED"`

---

### Scenario F: Currency Mismatch

User has Goal A (yoUSD) and Goal B (yoEUR). Cannot transfer between them in a single request — different vaults. Frontend should only show transfer options between goals sharing the same `currency` / `yieldTokenSymbol`.

---

## 7. Frontend Integration Checklist

- [ ] Add "Transfer" button/action on goal detail or home
- [ ] Build Transfer modal/drawer:
  - [ ] Source selector: Goal A, Goal B, … or "General Savings"
  - [ ] Destination selector: Goal A, Goal B, … or "General Savings"
  - [ ] Amount input (USD)
  - [ ] Currency from selected source (or default yoUSD)
- [ ] Call `POST /savings/transfer` on confirm
- [ ] On success: refresh `GET /savings/home` (or optimistic update)
- [ ] Add "Delete Goal" action with confirmation dialog
- [ ] Call `DELETE /savings/goals/:goalId` on confirm
- [ ] On 204: remove goal from UI, refresh home data
- [ ] Handle `activityPreview` entries with `isInternal: true`:
  - [ ] Display "Internal Transfer" or similar label
  - [ ] Use `relatedGoalName` for "from/to" context
  - [ ] Hide or disable `explorerUrl` when `transactionHash` is null

---

## 8. Error Handling

| HTTP Code | Error | Meaning | Frontend Action |
|-----------|-------|---------|-----------------|
| 400 | Bad Request | Validation failed | Show message: e.g. "Insufficient balance", "Cannot transfer to same goal", "Invalid source or destination" |
| 404 | Not Found | Goal not found / archived | Navigate back, show "Goal not found" |
| 401 | Unauthorized | Invalid or missing JWT | Refresh token, re-auth |

Example 400 body:
```json
{
  "statusCode": 400,
  "message": "Insufficient balance. You only have $25.00 available to transfer."
}
```

---

## 9. TypeScript Interfaces

```typescript
// Transfer request
interface TransferSavingsDto {
  fromGoalId?: string | null;
  toGoalId?: string | null;
  amount: number;
  currency: string;
}

// Activity preview entry (includes internal transfers)
interface ActivityPreviewEntry {
  id: string;
  type: 'YIELD_DEPOSIT' | 'YIELD_WITHDRAWAL' | 'INTERNAL_TRANSFER_IN' | 'INTERNAL_TRANSFER_OUT';
  status: string;
  fromAsset: string;
  toAsset: string;
  fromAmount: string;
  toAmount: string;
  createdAt: string;
  transactionHash: string | null;
  explorerUrl: string | null;
  savingsGoal: { name: string; icon: string } | null;
  isInternal?: boolean;
  relatedGoalName?: string;
}

// Helper to display transfer description
function getTransferDescription(entry: ActivityPreviewEntry): string {
  if (!entry.isInternal) return entry.type === 'YIELD_DEPOSIT' ? 'Deposit' : 'Withdrawal';
  const direction = entry.type === 'INTERNAL_TRANSFER_IN' ? 'from' : 'to';
  return `Transfer ${direction} ${entry.relatedGoalName ?? 'General Savings'}`;
}
```

---

## Summary

| Action | Endpoint | Notes |
|--------|----------|-------|
| Transfer between goals / general | `POST /savings/transfer` | Off-chain, instant |
| Delete (archive) goal | `DELETE /savings/goals/:goalId` | Auto-transfers balance to General Savings |
| Refresh balances | `GET /savings/home` | Use after transfer or delete |
| Goal detail with totals | `GET /savings/goals/:goalId` | Includes internal transfer stats |

All monetary values remain in **USD** for display. Internal transfers are stored in vault token units and converted using live EUR/USD rates when needed. No schema changes or migrations were required; the existing Transaction and SavingsGoal models support these features.
