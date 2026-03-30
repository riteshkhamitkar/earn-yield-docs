# Avvio Backend — Load Testing API Endpoints

**Base URL:** `https://api.avvio.xyz/api/v1`  
**Protocol:** HTTPS  
**Framework:** NestJS on Fastify  
**Auth:** Bearer JWT token in `Authorization` header (where marked 🔒)  
**Rate Limiting:** Throttle decorator applied on specific routes (noted below)

> **Note to Tester:** Endpoints marked 🔒 require a valid JWT token. Endpoints marked 🌐 are public.  
> Webhook endpoints are excluded — they are called by third-party services (Bridge, MoonPay, Persona) and are not relevant for client-side load testing.

---

## Table of Contents

1. [Health](#1-health)
2. [Auth](#2-auth)
3. [User](#3-user)
4. [Dashboard](#4-dashboard)
5. [Balances](#5-balances)
6. [Assets](#6-assets)
7. [Tokens](#7-tokens)
8. [Transfers](#8-transfers)
9. [Swaps](#9-swaps)
10. [Yield (Earn)](#10-yield-earn)
11. [Savings](#11-savings)
12. [Payment Links](#12-payment-links)
13. [Bridge (KYC / Off-Ramp / Fiat)](#13-bridge-kyc--off-ramp--fiat)
14. [Business (Organizations)](#14-business-organizations)
15. [Card](#15-card)
16. [Paymaster](#16-paymaster)
17. [Address Book](#17-address-book)
18. [Notifications](#18-notifications)
19. [MoonPay Commerce](#19-moonpay-commerce)
20. [AI Chat](#20-ai-chat)

---

## 1. Health

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 1 | `GET` | `/health` | 🌐 | — | Health check / uptime probe |

---

## 2. Auth

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 2 | `POST` | `/auth/initiate-registration` | 🌐 | 10 req/min | Initiate user registration (Privy) |
| 3 | `POST` | `/auth/sync-session` | 🌐 | 10 req/min | Sync session after login |
| 4 | `POST` | `/auth/me` | 🔒 | 20 req/min | Get current authenticated user |
| 5 | `POST` | `/auth/refresh` | 🌐 | 10 req/min | Refresh JWT token |

**Sample Body (`/auth/initiate-registration`):**
```json
{
  "privyToken": "<privy_auth_token>"
}
```

---

## 3. User

All endpoints require authentication.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 6 | `POST` | `/user/username` | 🔒 | ✅ | Set / update username |
| 7 | `GET` | `/user/username/check?username={username}` | 🔒 | ✅ | Check username availability |
| 8 | `GET` | `/user/lookup?username={username}` | 🔒 | — | Lookup user by username |
| 9 | `GET` | `/user/preferences` | 🔒 | — | Get user preferences |
| 10 | `PATCH` | `/user/preferences` | 🔒 | — | Update user preferences |
| 11 | `GET` | `/user/currencies` | 🔒 | — | Get supported fiat currencies |
| 12 | `PUT` | `/user/avatar` | 🔒 | — | Update user avatar |
| 13 | `GET` | `/user/avatar` | 🔒 | — | Get user avatar |

---

## 4. Dashboard

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 14 | `GET` | `/dashboard?currency={currency}` | 🔒 | — | Get dashboard (portfolio summary) |
| 15 | `GET` | `/dashboard/{userId}?currency={currency}` | 🌐 | — | Get dashboard by user ID (public) |

---

## 5. Balances

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 16 | `GET` | `/balances/portfolio?currency={currency}` | 🔒 | — | Full portfolio overview |
| 17 | `GET` | `/balances/asset/{assetId}?currency={currency}` | 🔒 | — | Single asset balance |
| 18 | `GET` | `/balances/transactions?limit={n}&continuation={token}&currency={cur}&goalId={id}&includeSavings={bool}` | 🔒 | — | Transaction history (paginated) |
| 19 | `GET` | `/balances/wallet/addresses` | 🔒 | — | Get user wallet addresses |
| 20 | `GET` | `/balances/supported-assets` | 🌐 | — | List all supported assets |
| 21 | `GET` | `/balances/holding-assets?category={cat}&priceApiId={id}&currency={cur}` | 🔒 | — | Get holding assets with filters |
| 22 | `GET` | `/balances/holding-asset-by-price-api-id?priceApiId={id}&currency={cur}` | 🔒 | — | Get single holding asset by price API ID |

---

## 6. Assets

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 23 | `GET` | `/assets/{categoryId}?currency={currency}` | 🔒 | — | Get asset details by category |

---

## 7. Tokens

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 24 | `GET` | `/tokens/list?section={section}&currency={currency}` | 🌐 | — | Token list (trending, etc.) |
| 25 | `GET` | `/tokens/currencies` | 🌐 | — | Supported currencies |

---

## 8. Transfers

All endpoints require authentication.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 26 | `POST` | `/transfers/validate-address` | 🔒 | — | Validate a recipient address |
| 27 | `POST` | `/transfers/send` | 🔒 | — | Initiate a crypto transfer |
| 28 | `POST` | `/transfers/execute` | 🔒 | — | Execute a prepared transfer |
| 29 | `GET` | `/transfers/recent-addresses` | 🔒 | — | Get recent recipient addresses |
| 30 | `GET` | `/transfers/recipients` | 🔒 | — | Get all recipients |
| 31 | `GET` | `/transfers/{quoteId}/status` | 🔒 | — | Check transfer status by quote ID |

**Sample Body (`/transfers/send`):**
```json
{
  "toAddress": "0x...",
  "amount": "10.5",
  "assetId": "usdc-base",
  "chainId": 8453
}
```

---

## 9. Swaps

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 32 | `GET` | `/swaps/supported-assets` | 🌐 | — | List swap-supported assets |
| 33 | `GET` | `/swaps/history` | 🔒 | — | Swap history |
| 34 | `POST` | `/swaps/quote` | 🔒 | — | Get swap quote |
| 35 | `POST` | `/swaps/execute` | 🔒 | — | Execute a swap |
| 36 | `POST` | `/swaps/lifi/execute` | 🔒 | — | Execute swap via LiFi |
| 37 | `POST` | `/swaps/consolidate/execute` | 🔒 | — | Execute consolidation for swap |
| 38 | `POST` | `/swaps/consolidate/link` | 🔒 | — | Link consolidation to swap |
| 39 | `GET` | `/swaps/consolidate/status/{quoteId}` | 🔒 | — | Consolidation status |
| 40 | `GET` | `/swaps/{quoteId}/status` | 🔒 | — | Swap status by quote ID |

**Sample Body (`/swaps/quote`):**
```json
{
  "fromAssetId": "usdc-base",
  "toAssetId": "eth-base",
  "amount": "100"
}
```

---

## 10. Yield (Earn)

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 41 | `GET` | `/yield/vaults` | 🌐 | — | List all yield vaults |
| 42 | `GET` | `/yield/positions` | 🔒 | — | User's yield positions |
| 43 | `GET` | `/yield/positions/{vaultSymbol}/history` | 🔒 | — | Position history for a vault |
| 44 | `GET` | `/yield/points` | 🔒 | — | User's yield points |
| 45 | `POST` | `/yield/deposit/quote` | 🔒 | — | Get deposit quote |
| 46 | `POST` | `/yield/deposit/call-quote` | 🔒 | — | Get call-data for deposit |
| 47 | `POST` | `/yield/deposit/execute` | 🔒 | — | Execute yield deposit |
| 48 | `POST` | `/yield/deposit/consolidate` | 🔒 | — | Execute consolidation for deposit |
| 49 | `GET` | `/yield/deposit/consolidate/status/{quoteId}` | 🔒 | — | Consolidation status |
| 50 | `POST` | `/yield/redeem/quote` | 🔒 | — | Get redemption quote |
| 51 | `POST` | `/yield/redeem/call-quote` | 🔒 | — | Get call-data for redemption |
| 52 | `POST` | `/yield/redeem/execute` | 🔒 | — | Execute yield redemption |
| 53 | `GET` | `/yield/status/{quoteId}` | 🔒 | — | Yield operation status |

---

## 11. Savings

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 54 | `POST` | `/savings/goals` | 🔒 | — | Create a savings goal |
| 55 | `POST` | `/savings/transfer` | 🔒 | — | Transfer between savings goals |
| 56 | `DELETE` | `/savings/goals/{goalId}` | 🔒 | — | Delete a savings goal |
| 57 | `PATCH` | `/savings/goals/{goalId}` | 🔒 | — | Update a savings goal |
| 58 | `GET` | `/savings/goals/{goalId}` | 🔒 | — | Get savings goal details |
| 59 | `GET` | `/savings/home?activityLimit={n}&currency={cur}` | 🔒 | — | Savings home screen data |
| 60 | `GET` | `/savings/home/{userId}?activityLimit={n}` | 🌐 | — | Savings home by user ID (public) |
| 61 | `GET` | `/savings/details?goalId={id}&currency={cur}` | 🔒 | — | Savings details |

---

## 12. Payment Links

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 62 | `POST` | `/payment-links/prepare` | 🔒 | 10 req/min | Prepare a payment link |
| 63 | `POST` | `/payment-links/record` | 🔒 | 10 req/min | Record a payment link on-chain |
| 64 | `POST` | `/payment-links/pay` | 🔒 | 20 req/min | Pay a payment link |
| 65 | `POST` | `/payment-links/withdraw` | 🔒 | 5 req/min | Withdraw from a send link |
| 66 | `POST` | `/payment-links/reclaim` | 🔒 | — | Reclaim a send link |
| 67 | `POST` | `/payment-links/claim` | 🔒 | — | Claim payments |
| 68 | `POST` | `/payment-links/cancel` | 🔒 | — | Cancel a payment |
| 69 | `POST` | `/payment-links/{slug}/close` | 🔒 | — | Close a payment link |
| 70 | `GET` | `/payment-links?limit={n}&offset={n}` | 🔒 | — | List user's payment links |
| 71 | `GET` | `/payment-links/{slug}` | 🌐 | — | Get link details (public) |
| 72 | `GET` | `/payment-links/{slug}/payments` | 🔒 | — | Get payments for a link |

---

## 13. Bridge (KYC / Off-Ramp / Fiat)

All endpoints require authentication.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 73 | `POST` | `/bridge/kyc/initiate` | 🔒 | — | Initiate KYC process |
| 74 | `GET` | `/bridge/kyc/status` | 🔒 | — | Get KYC status |
| 75 | `GET` | `/bridge/kyc/tos-link` | 🔒 | — | Get Terms of Service link |
| 76 | `POST` | `/bridge/kyc/persona/session` | 🔒 | — | Create Persona KYC session |
| 77 | `GET` | `/bridge/virtual-account` | 🔒 | — | Get virtual bank account |
| 78 | `GET` | `/bridge/schemas?currency={currency}` | 🔒 | — | Get bank schemas for currency |
| 79 | `POST` | `/bridge/external-accounts` | 🔒 | — | Create external bank account |
| 80 | `GET` | `/bridge/external-accounts` | 🔒 | — | List external bank accounts |
| 81 | `DELETE` | `/bridge/external-accounts/{externalAccountId}` | 🔒 | — | Delete external bank account |
| 82 | `POST` | `/bridge/off-ramp/initiate` | 🔒 | — | Initiate off-ramp (crypto → fiat) |
| 83 | `POST` | `/bridge/transfers/payout` | 🔒 | — | Initiate fiat payout |
| 84 | `GET` | `/bridge/off-ramp/{transferId}/status` | 🔒 | — | Off-ramp transfer status |
| 85 | `GET` | `/bridge/transactions?type={type}` | 🔒 | — | List bridge transactions |

---

## 14. Business (Organizations)

All endpoints require authentication. All have rate limits.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 86 | `POST` | `/business/organizations` | 🔒 | ✅ | Create organization |
| 87 | `GET` | `/business/organizations` | 🔒 | ✅ | List user's organizations |
| 88 | `GET` | `/business/organizations/{orgId}` | 🔒 | ✅ | Get organization details |
| 89 | `POST` | `/business/organizations/{orgId}/wallets` | 🔒 | ✅ | Add wallet to organization |
| 90 | `POST` | `/business/organizations/{orgId}/invites` | 🔒 | ✅ | Invite member |
| 91 | `GET` | `/business/organizations/{orgId}/invites` | 🔒 | ✅ | List pending invites |
| 92 | `DELETE` | `/business/organizations/{orgId}/invites/{inviteId}` | 🔒 | ✅ | Revoke an invite |
| 93 | `POST` | `/business/invites/{token}/accept` | 🔒 | ✅ | Accept an invite |
| 94 | `PATCH` | `/business/organizations/{orgId}/members/{memberId}` | 🔒 | ✅ | Update member role |
| 95 | `DELETE` | `/business/organizations/{orgId}/members/{memberId}` | 🔒 | ✅ | Remove member |
| 96 | `POST` | `/business/organizations/{orgId}/policies` | 🔒 | ✅ | Create spending policy |
| 97 | `GET` | `/business/organizations/{orgId}/policies` | 🔒 | ✅ | List policies |
| 98 | `PATCH` | `/business/organizations/{orgId}/policies/{policyId}` | 🔒 | ✅ | Update policy |
| 99 | `DELETE` | `/business/organizations/{orgId}/policies/{policyId}` | 🔒 | ✅ | Delete policy |

---

## 15. Card

All endpoints require authentication.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 100 | `GET` | `/card/notify/status` | 🔒 | 30 req/min | Check card notification status |
| 101 | `POST` | `/card/notify` | 🔒 | 10 req/min | Register for card notification |

---

## 16. Paymaster

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 102 | `POST` | `/paymaster/sign` | 🔒 | 10 req/min | Sign a transaction via paymaster |
| 103 | `POST` | `/paymaster/onebalance/modify` | 🔒 | 10 req/min | Modify a OneBalance operation |
| 104 | `POST` | `/paymaster/onebalance/sign` | 🔒 | 10 req/min | Sign a OneBalance operation |
| 105 | `GET` | `/paymaster/address` | 🔒 | — | Get paymaster address |

---

## 17. Address Book

All endpoints require authentication.

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 106 | `POST` | `/address-book` | 🔒 | — | Create a contact |
| 107 | `GET` | `/address-book/unified` | 🔒 | — | Get unified recipients list |
| 108 | `GET` | `/address-book` | 🔒 | — | List all contacts |
| 109 | `PUT` | `/address-book/{id}` | 🔒 | — | Update a contact |
| 110 | `DELETE` | `/address-book/{id}` | 🔒 | — | Delete a contact |

---

## 18. Notifications

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 111 | `POST` | `/notifications/subscribe` | 🔒 | — | Subscribe to push notifications |

---

## 19. MoonPay Commerce

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 112 | `GET` | `/moonpay-commerce/deposit-token` | 🔒 | — | Get MoonPay deposit token |

---

## 20. AI Chat

| # | Method | Endpoint | Auth | Rate Limit | Description |
|---|--------|----------|------|------------|-------------|
| 113 | `POST` | `/ai/chat` | 🔒 | 30 req/min | Send message to AI chatbot |

---

## Load Testing Recommendations

### Priority Tiers

**Tier 1 — Critical Path (highest load):**
These are hit on every app session and should handle the most concurrent users.

| Endpoint | Why |
|----------|-----|
| `GET /health` | Uptime probe, baseline |
| `POST /auth/sync-session` | Every login |
| `POST /auth/me` | Every app open |
| `POST /auth/refresh` | Token refresh loop |
| `GET /dashboard` | Main home screen |
| `GET /balances/portfolio` | Portfolio on home |
| `GET /balances/holding-assets` | Asset list |
| `GET /balances/transactions` | Tx history |
| `GET /tokens/list` | Token browse |

**Tier 2 — Core Features (medium load):**
These are used frequently but not on every session.

| Endpoint | Why |
|----------|-----|
| `POST /transfers/send` | Sending crypto |
| `POST /transfers/execute` | Executing transfer |
| `GET /transfers/{quoteId}/status` | Polling status |
| `POST /swaps/quote` | Swap quotes |
| `POST /swaps/execute` | Executing swaps |
| `GET /swaps/{quoteId}/status` | Polling swap status |
| `POST /payment-links/prepare` | Creating links |
| `POST /payment-links/pay` | Paying links |
| `GET /payment-links/{slug}` | Viewing public links |
| `GET /savings/home` | Savings screen |
| `GET /yield/vaults` | Yield vault list |

**Tier 3 — Secondary Features (lower load):**

| Endpoint | Why |
|----------|-----|
| `POST /yield/deposit/quote` | Yield operations |
| `POST /bridge/kyc/initiate` | KYC flow |
| `POST /bridge/off-ramp/initiate` | Off-ramp |
| `POST /business/organizations` | Business features |
| `POST /paymaster/sign` | Paymaster signing |
| `POST /ai/chat` | AI chatbot |
| `POST /card/notify` | Card waitlist |

### Rate Limit Summary

| Endpoint | Limit |
|----------|-------|
| `POST /auth/initiate-registration` | 10 req/min |
| `POST /auth/sync-session` | 10 req/min |
| `POST /auth/me` | 20 req/min |
| `POST /auth/refresh` | 10 req/min |
| `POST /payment-links/prepare` | 10 req/min |
| `POST /payment-links/record` | 10 req/min |
| `POST /payment-links/pay` | 20 req/min |
| `POST /payment-links/withdraw` | 5 req/min |
| `POST /paymaster/sign` | 10 req/min |
| `POST /paymaster/onebalance/modify` | 10 req/min |
| `POST /paymaster/onebalance/sign` | 10 req/min |
| `GET /card/notify/status` | 30 req/min |
| `POST /card/notify` | 10 req/min |
| `POST /ai/chat` | 30 req/min |
| All `/business/*` routes | Rate limited (varies) |
| `/user/username`, `/user/username/check` | Rate limited (varies) |

### Auth Setup for Load Tests

1. **Obtain a valid JWT:** Call `POST /auth/sync-session` with a valid Privy token.
2. **Attach to all 🔒 requests:**  
   ```
   Authorization: Bearer <jwt_token>
   ```
3. **Token refresh:** Periodically call `POST /auth/refresh` to keep the token alive.

### Suggested Tools

- **k6** (Grafana) — scriptable, supports scenarios & thresholds
- **Artillery** — YAML-based, good for API load testing
- **Locust** — Python-based, distributed load generation

---

**Total Endpoints: 113**  
**Authenticated (🔒): 93** | **Public (🌐): 20**  
**Webhook endpoints (excluded): 4** (Bridge, Persona, MoonPay Commerce, MoonPay OnRamps)

*Document generated from source code scan on March 30, 2026.*
