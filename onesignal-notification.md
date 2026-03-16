# OneSignal Push Notifications Integration

## Document Purpose

This is a **detailed architectural and implementation plan** for integrating OneSignal push notifications into the Anzo backend (NestJS) and the Avvio mobile app (React Native). **No code has been implemented.** This document serves as a blueprint for development, with Context7-verified OneSignal references, real-life user flows, and complete scope for both backend and frontend.

**Context7 OneSignal Documentation:** [https://context7.com/websites/onesignal_en](https://context7.com/websites/onesignal_en)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Citations & Architectural Justifications](#2-citations--architectural-justifications)
3. [Database Schema Changes](#3-database-schema-changes)
4. [Backend .env Changes](#4-backend-env-changes)
5. [Backend Scope](#5-backend-scope)
6. [Frontend Scope](#6-frontend-scope)
7. [Real-Life User Workflows](#7-real-life-user-workflows)
7a. [Whole Real-Life Scenarios (Detailed)](#7a-whole-real-life-scenarios-detailed)
8. [File-by-File Analysis: Deposit & Swap Flows](#8-file-by-file-analysis-deposit--swap-flows)
9. [Notification Examples: Deposit & Swap](#9-notification-examples-deposit--swap)
10. [iOS & Android Compatibility](#10-ios--android-compatibility)
11. [Context7-Verified OneSignal References](#11-context7-verified-onesignal-references)

---

## 1. Executive Summary

### Objective

Integrate OneSignal for sending push notifications (e.g., **Deposit received**, **Swap completed**) to users of the Avvio mobile app. The Anzo backend will send notifications; the Avvio frontend will handle device registration and user association.

### Core Architectural Decision

**Use Anzo `userId` as OneSignal's `external_id`.**

- **Rationale:** OneSignal requires an identifier to link multiple devices (subscriptions) to a single user. Your `User.id` (cuid, e.g. `clx7abc123xyz`) is the ideal candidate.
- **Flow:** Backend sends `include_external_user_ids: [userId]` or `include_aliases: { external_id: [userId] }`. OneSignal delivers to all devices linked to that `external_id`.

### Identifiers Involved

| Identifier | Where | Purpose |
|------------|-------|---------|
| **Anzo `userId`** | `User.id` in DB | Your internal user ID (cuid). Used as OneSignal `external_id`. |
| **OneSignal `subscription_id`** | OneSignal SDK on device | Unique per device. Frontend sends this to backend for storage. |
| **OneSignal `external_id`** | Set via `OneSignal.login(userId)` | Links device to Anzo user. Backend targets by this when sending. |

---

## 2. Citations & Architectural Justifications

| Decision | Justification | Citation |
|----------|---------------|----------|
| New `NotificationsModule` | Anzo uses feature-per-module structure (e.g. `SwapsModule`, `BridgeModule`). | `anzo-backend-main/src/app.module.ts` |
| Service & Handler pattern | `rules.md` Section 2.2 mandates this pattern for all business logic. | `anzo-backend-main/rules.md` |
| `PushSubscription` table | One user can have multiple devices. Need one-to-many with Prisma. | `anzo-backend-main/prisma/schema.prisma`, `src/modules/database/prisma.service.ts` |
| `userId` as `external_id` | OneSignal Aliases: "You must set an External ID... This ID only stays consistent across Subscriptions." | OneSignal Docs – [aliases](https://documentation.onesignal.com/docs/en/aliases) |
| `OneSignalProvider` | Mirrors existing providers (`TurnkeyProvider`, `OneBalanceProvider`). | `anzo-backend-main/src/modules/auth/providers/turnkey.provider.ts` |
| `include_external_user_ids` / `include_aliases` | OneSignal Create Notification API uses these to target users by `external_id`. | OneSignal Docs – [Create Notification API](https://documentation.onesignal.com/docs/en/push-fallback-method) |
| Frontend registers via API | Frontend must send `subscription_id` to backend after `OneSignal.login(userId)`. | OneSignal Docs – [React Native SDK setup](https://documentation.onesignal.com/docs/en/react-native-sdk-setup) |
| `OneSignal.login(userId)` | Associates device with `external_id`. "Logging in a new user automatically transfers push subscription to the newly logged-in user." | OneSignal React Native – [MIGRATION_GUIDE.md](https://github.com/onesignal/react-native-onesignal/blob/main/MIGRATION_GUIDE.md) |
| `.env` & `validation.schema.ts` | `rules.md` Section 6 and existing config require this for secrets. | `anzo-backend-main/src/config/validation.schema.ts` |
| Deep linking via `data` / `additionalData` | "Instead of Launch URL, you can use the **Additional Data** field (`data` in the API) to send custom data... handle this data... via the SDK's Notification Click Listener." | OneSignal Docs – [Deep linking](https://documentation.onesignal.com/docs/en/deep-linking) |

---

## 3. Database Schema Changes

### New Model: `PushSubscription`

**File:** `prisma/schema.prisma`

```prisma
model PushSubscription {
  id             String   @id @default(cuid())
  userId         String
  subscriptionId String   @unique   // OneSignal subscription_id from SDK
  platform       String             // "ios" or "android"
  isActive       Boolean  @default(true)
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}
```

### Update `User` Model

Add to existing `User` model:

```prisma
pushSubscriptions PushSubscription[]
```

### Migration Command

```bash
npx prisma migrate dev --name add_push_subscription_model
```

### Rationale

- **`subscriptionId` @unique:** One device = one subscription; prevents duplicates.
- **`userId`:** Links to Anzo user; supports multiple devices per user.
- **`platform`:** Allows analytics and platform-specific handling.
- **`onDelete: Cascade`:** Cleans up subscriptions when user is deleted.

---

## 4. Backend .env Changes

### New Variables (Required)

Add to `.env`:

```env
# ============================================
# OneSignal Push Notifications
# ============================================
ONESIGNAL_APP_ID=YOUR_ONESIGNAL_APP_ID_FROM_DASHBOARD
ONESIGNAL_REST_API_KEY=YOUR_ONESIGNAL_REST_API_KEY_FROM_DASHBOARD
```

### Where to Obtain Values

| Variable | Source |
|----------|--------|
| `ONESIGNAL_APP_ID` | OneSignal Dashboard → Your App → Keys & IDs |
| `ONESIGNAL_REST_API_KEY` | OneSignal Dashboard → Your App → Keys & IDs → REST API Key |

### Config Updates

**`src/config/validation.schema.ts`** – add validation:

```typescript
ONESIGNAL_APP_ID: Joi.string().required(),
ONESIGNAL_REST_API_KEY: Joi.string().required(),
```

**`src/config/configuration.ts`** – add:

```typescript
onesignal: {
  appId: process.env.ONESIGNAL_APP_ID,
  restApiKey: process.env.ONESIGNAL_REST_API_KEY,
},
```

### Full .env Reference (OneSignal Section Only)

```env
# ============================================
# OneSignal Push Notifications
# ============================================
# App ID from OneSignal dashboard (Keys & IDs)
ONESIGNAL_APP_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# REST API Key - used for Server API to create notifications
ONESIGNAL_REST_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

---

## 5. Backend Scope

### New Module: `NotificationsModule`

**Directory:** `src/modules/notifications/`

```
notifications/
├── dto/
│   └── subscribe.dto.ts
├── handlers/
│   ├── subscription.handler.ts
│   └── message-sending.handler.ts
├── providers/
│   └── onesignal.provider.ts
├── notifications.controller.ts
├── notifications.service.ts
└── notifications.module.ts
```

### New API Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `POST` | `/api/v1/notifications/subscribe` | JWT (AuthGuard) | Register device subscription. Body: `{ subscriptionId, platform }` |

### Integration Points for Notifications

| Module | File | Event | userId Source |
|--------|------|-------|---------------|
| **Swaps** | `swap-status.handler.ts` | Status → `COMPLETED` | `swap.userId` |
| **Swaps** | `swap-execution.handler.ts` | Background poll → `COMPLETED` | `swap.userId` |
| **MoonPay Commerce** | `moonpay-commerce-webhook.controller.ts` | `DEPOSIT_TX_CONFIRMED` | `wallet.userId` |
| **MoonPay On-Ramps** | `moonpay-onramps-webhook.controller.ts` | `transaction_updated` (status=completed) | `existing.userId` or `userId` from `findUserByWalletAddress` |

### Backend Architecture Diagram

```
[ SwapsModule / MoonPay Webhooks / Transfers ]
                    │
                    ▼ (inject NotificationsService)
           [ NotificationsService ]
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
 SubscriptionHandler    MessageSendingHandler
        │                       │
        ▼                       ▼
   [ PrismaService ]     [ OneSignalProvider ]
        │                       │
        ▼                       ▼
[ PushSubscription ]    [ OneSignal REST API ]
```

---

## 6. Frontend Scope

### Objective

- Initialize OneSignal SDK.
- Call `OneSignal.login(userId)` after Anzo login.
- Send `subscriptionId` and `platform` to `POST /api/v1/notifications/subscribe`.
- Handle notification taps for deep linking.
- Call `OneSignal.logout()` on logout.

### Required Frontend Work

| Task | Location | Details |
|------|----------|---------|
| Install SDK | project root | `yarn add react-native-onesignal` |
| Initialize OneSignal | App entry (e.g. `App.tsx`) | `OneSignal.initialize(ONESIGNAL_APP_ID)` |
| Get subscription ID | After login | `await OneSignal.User.pushSubscription.getIdAsync()` |
| Login / Logout | Auth provider | `OneSignal.login(userId)` / `OneSignal.logout()` |
| POST subscribe | New API method | `POST /api/v1/notifications/subscribe` with JWT |
| Click listener | Root navigator | `OneSignal.Notifications.addClickListener` |
| Deep link parsing | Click handler | Parse `additionalData.deepLink` → navigate |

### Frontend .env

```env
ONESIGNAL_APP_ID=YOUR_ONESIGNAL_APP_ID_FROM_DASHBOARD
```

---

## 7. Real-Life User Workflows

### Flow 1: Device Registration (One-Time per Device)

1. **User opens Avvio** → OneSignal SDK initializes; APNs/FCM returns `subscription_id`.
2. **User logs in** (passkey/Turnkey) → Backend returns JWT containing `userId` (Anzo `User.id`).
3. **Frontend calls** `OneSignal.login(userId)` → OneSignal links device to `external_id`.
4. **Frontend calls** `POST /api/v1/notifications/subscribe` with `{ subscriptionId, platform }`.
5. **Backend** creates/updates `PushSubscription` for that user.

### Flow 2: Swap Completed Notification

1. **User initiates swap** (e.g. 0.1 ETH → USDC) via Avvio.
2. **Backend** executes via OneBalance/LI.FI; status polling runs.
3. **Swap status** becomes `COMPLETED` (in `SwapStatusHandler` or background poll).
4. **Backend** calls `NotificationsService.sendToUser(userId, message)`.
5. **MessageSendingHandler** sends to OneSignal with `include_external_user_ids: [userId]` (or `include_aliases`).
6. **OneSignal** delivers to all devices for that `external_id`.
7. **User sees** "Swap Complete!" on lock screen.

### Flow 3: Deposit Received Notification

1. **User completes buy** via MoonPay On-Ramp or MoonPay Commerce.
2. **MoonPay** sends webhook: `transaction_updated` (completed) or `DEPOSIT_TX_CONFIRMED`.
3. **Backend** creates/updates `Transaction` with `status: COMPLETED`.
4. **Backend** calls `NotificationsService.sendToUser(userId, message)`.
5. **User sees** "Deposit received" push notification.

### Flow 4: Deep Link on Notification Tap

1. **User taps** "Swap Complete!" notification.
2. **OS** opens Avvio app.
3. **SDK** fires `addClickListener`; `event.notification.additionalData.deepLink` = `avvio://activity/transaction/tx_id`.
4. **Frontend** parses URL and navigates to `ActivityDetailScreen` with `transactionId`.

---

## 7a. Whole Real-Life Scenarios (Detailed)

Below are complete, minute-by-minute scenarios with concrete data, API calls, and system state.

---

### Scenario A: Alice – Swap Completed (OneBalance, ETH → USDC)

**Characters:** Alice (user), Avvio app (iOS), Anzo backend, OneBalance, OneSignal.

| Time | Actor | Action | Details |
|------|-------|--------|---------|
| **T+0** | Alice | Opens Avvio | App was installed yesterday. She logged in then; device is already registered. |
| **T+0** | Avvio | OneSignal already initialized | `subscription_id: 7a3f2b1c-8d9e-4a5b-9c0d-1e2f3a4b5c6d` linked to `external_id: clx7alice123` |
| **T+1** | Alice | Taps "Swap" | Wants to swap 0.1 ETH → USDC |
| **T+2** | Avvio | `POST /swaps/quote` | `{ fromAsset: "ETH", toAsset: "USDC", fromAmount: "0.1" }` |
| **T+2** | Backend | Creates quote | `quoteId: ob_quote_abc789`, Transaction `clx7tx001`, Swap linked to `userId: clx7alice123` |
| **T+3** | Alice | Signs with passkey | Turnkey signs; Avvio gets `signedOperations`, `tamperProofSignature` |
| **T+4** | Avvio | `POST /swaps/execute` | Sends signed data, `quoteId` |
| **T+4** | Backend | OneBalance API | `executeQuote()` → HTTP 200, `success: true` |
| **T+4** | Backend | Starts background poll | `pollExecutionStatusInBackground(ob_quote_abc789)` runs async |
| **T+4** | Backend | Updates DB | Swap & Transaction → `EXECUTING` |
| **T+5** | Avvio | Shows "Processing…" | Polls `GET /swaps/ob_quote_abc789/status` every 3 sec |
| **T+90** | Backend | Background poll | OneBalance returns `status: "COMPLETED"`, `transactionHash: "0x123…"` |
| **T+90** | Backend | Updates DB | Swap & Transaction → `COMPLETED`, `completedAt: now` |
| **T+90** | Backend | Sends notification | `NotificationsService.sendToUser("clx7alice123", { headings: { en: "Swap Complete!" }, … })` |
| **T+90** | MessageSendingHandler | OneSignal API | `POST https://api.onesignal.com/notifications` with `include_external_user_ids: ["clx7alice123"]` |
| **T+91** | OneSignal | Targets devices | Finds device with `external_id: clx7alice123` → `subscription_id: 7a3f2b1c-...` |
| **T+91** | APNs | Delivers push | To Alice's iPhone |
| **T+92** | Alice | Sees notification | Lock screen: "Swap Complete!" / "Your swap of 0.1 ETH to 345.21 USDC is complete." |
| **T+95** | Alice | Taps notification | iOS opens Avvio |
| **T+95** | Avvio | addClickListener | `additionalData.deepLink = "avvio://activity/transaction/clx7tx001"` |
| **T+96** | Avvio | Navigates | `ActivityDetailScreen` with `transactionId: clx7tx001` |
| **T+96** | Alice | Sees transaction | Completed swap details on screen |

---

### Scenario B: Raj – Deposit Received (MoonPay On-Ramp, Buy with Card)

**Characters:** Raj (user), Avvio app (Android), Anzo backend, MoonPay, OneSignal.

| Time | Actor | Action | Details |
|------|-------|--------|---------|
| **T+0** | Raj | Opens Avvio | First time; not yet logged in |
| **T+1** | Raj | Logs in | Google Sign-In → Turnkey → Backend returns JWT with `userId: clx7raj456` |
| **T+2** | Avvio | OneSignal.login | `OneSignal.login("clx7raj456")` – device linked |
| **T+2** | Avvio | Gets subscription ID | `await OneSignal.User.pushSubscription.getIdAsync()` → `sub_android_xyz789` |
| **T+3** | Avvio | POST subscribe | `POST /notifications/subscribe` + JWT, `{ subscriptionId: "sub_android_xyz789", platform: "android" }` |
| **T+3** | Backend | Saves subscription | `PushSubscription` created: `userId: clx7raj456`, `subscriptionId: sub_android_xyz789` |
| **T+10** | Raj | Taps "Buy Crypto" | Chooses 50 USDC, pays with Visa |
| **T+11** | Avvio | Opens MoonPay widget | MoonPay On-Ramp; Raj completes payment |
| **T+12** | MoonPay | Processes payment | Card charged; crypto purchase initiated |
| **T+15** | MoonPay | Sends webhook | `transaction_created` → Backend creates `Transaction` PENDING |
| **T+120** | MoonPay | Crypto delivered | Sends to Raj's wallet `0xRaj...` on Ethereum |
| **T+121** | MoonPay | Sends webhook | `transaction_updated` with `status: "completed"`, `walletAddress: "0xRaj..."` |
| **T+121** | Backend | findUserByWalletAddress | `prisma.wallet.findFirst({ ethereumAddress: "0xRaj..." })` → `userId: clx7raj456` |
| **T+121** | Backend | Updates transaction | `Transaction` → `COMPLETED`, `toAmount: "50"`, `toAsset: "USDC"` |
| **T+121** | Backend | Sends notification | `NotificationsService.sendToUser("clx7raj456", { headings: { en: "Deposit Received" }, contents: { en: "Your purchase of 50 USDC is complete." } })` |
| **T+122** | OneSignal | Delivers | Targets `external_id: clx7raj456` → Android device |
| **T+123** | Raj | Sees notification | "Deposit Received – Your purchase of 50 USDC is complete." |

---

### Scenario C: Maria – Deposit Received (MoonPay Commerce / Helio)

**Characters:** Maria (user), external sender, MoonPay Commerce, Anzo backend, OneSignal.

| Time | Actor | Action | Details |
|------|-------|--------|---------|
| **T+0** | Maria | Has Avvio | Already registered; `userId: clx7maria789` |
| **T+1** | External sender | Sends 200 USDC | From another wallet to Maria's Avvio deposit address `0xMaria...` |
| **T+2** | MoonPay Commerce | Tracks on-chain | Detects deposit to Maria's address |
| **T+5** | MoonPay Commerce | Webhook | `DEPOSIT_TX_CONFIRMED` – tx confirmed on Ethereum |
| **T+5** | Backend | handleDepositTxConfirmed | `recipientAddress: 0xMaria...` → `prisma.wallet.findFirst` → `userId: clx7maria789` |
| **T+5** | Backend | Creates Transaction | `type: DEPOSIT`, `status: COMPLETED`, `toAsset: "USDC"`, `toAmount: "200"` |
| **T+5** | Backend | Sends notification | `NotificationsService.sendToUser("clx7maria789", { headings: { en: "Deposit Received" }, contents: { en: "You received 200 USDC on ethereum." } })` |
| **T+6** | OneSignal | Delivers | To all of Maria's devices (iPhone + iPad) |
| **T+6** | Maria | Sees on both devices | Same notification on iPhone and iPad |

---

### Scenario D: Multi-Device – Same User, Two Phones

**Setup:** User `clx7bob999` has iPhone (work) and Android (personal). Both are registered.

| Step | What Happens |
|------|--------------|
| 1 | Bob logs in on iPhone → `OneSignal.login("clx7bob999")`, `POST subscribe` with `subscriptionId: sub_ios_aaa`, `platform: "ios"` |
| 2 | Bob logs in on Android → `OneSignal.login("clx7bob999")`, `POST subscribe` with `subscriptionId: sub_android_bbb`, `platform: "android"` |
| 3 | DB has 2 rows: `(userId: clx7bob999, subscriptionId: sub_ios_aaa)`, `(userId: clx7bob999, subscriptionId: sub_android_bbb)` |
| 4 | OneSignal has 2 subscriptions both with `external_id: clx7bob999` |
| 5 | Bob does a swap; backend sends `include_external_user_ids: ["clx7bob999"]` |
| 6 | OneSignal delivers to both devices → Bob gets push on iPhone and Android |

---

### Scenario E: First-Time User – Full Onboarding to First Notification

**Characters:** New user Priya, fresh install.

| Step | Actor | Action |
|------|-------|--------|
| 1 | Priya | Downloads Avvio from App Store |
| 2 | Priya | Opens app – iOS asks "Allow notifications?" |
| 3 | Priya | Taps "Allow" |
| 4 | Avvio | OneSignal.initialize() – SDK gets APNs token, OneSignal creates `subscription_id` |
| 5 | Priya | Taps "Get Started" – creates passkey / Turnkey wallet |
| 6 | Backend | Creates User `clx7priya001`, returns JWT |
| 7 | Avvio | OneSignal.login("clx7priya001") – device linked to user |
| 8 | Avvio | POST subscribe – backend stores `PushSubscription` |
| 9 | Priya | Does first swap (0.01 ETH → USDC) |
| 10 | Backend | Swap completes → sends "Swap Complete!" notification |
| 11 | Priya | Sees first push: "Swap Complete! – Your swap of 0.01 ETH to 35.21 USDC is complete." |

---

### Scenario F: User Logs Out, Then Another User Logs In (Shared Device)

**Characters:** Device shared by Alex and Sam.

| Step | Actor | Action |
|------|-------|--------|
| 1 | Alex | Logged in; device has `external_id: clx7alex111` |
| 2 | Alex | Taps Logout |
| 3 | Avvio | `OneSignal.logout()` – device no longer tied to Alex |
| 4 | Sam | Logs in on same device |
| 5 | Avvio | `OneSignal.login("clx7sam222")` – device now linked to Sam |
| 6 | Avvio | POST subscribe – backend upserts `PushSubscription` for Sam |
| 7 | Sam | Does a swap |
| 8 | Backend | Sends notification to `clx7sam222` |
| 9 | Sam | Gets push (Alex does not) |

---

### Scenario G: Notification Tap → Deep Link → Correct Screen

**Setup:** Alice has "Swap Complete!" on lock screen with `deepLink: avvio://activity/transaction/clx7tx001`.

| Step | Action |
|------|--------|
| 1 | Alice taps notification |
| 2 | iOS opens Avvio (foreground or background) |
| 3 | OneSignal SDK fires `addClickListener` |
| 4 | Handler reads `event.notification.additionalData.deepLink` = `"avvio://activity/transaction/clx7tx001"` |
| 5 | Parse: path = `activity/transaction/clx7tx001` → `transactionId = "clx7tx001"` |
| 6 | `NavigationService.navigate("ActivityDetailScreen", { transactionId: "clx7tx001" })` |
| 7 | Screen loads transaction `clx7tx001` from API |
| 8 | Alice sees swap details (0.1 ETH → 345.21 USDC, timestamp, explorer link) |

---

## 8. File-by-File Analysis: Deposit & Swap Flows

### Swap Completion – Files Involved

| File | Lines | Role |
|------|-------|------|
| `src/modules/swaps/swaps.controller.ts` | 106–110 | `GET /swaps/:quoteId/status` → `getSwapStatus(req.user.userId, quoteId)` |
| `src/modules/swaps/swaps.service.ts` | 334–399 | `getSwapStatus()` → routes to `statusHandler.getSingleSwapStatus` or `getLifiSwapStatus` |
| `src/modules/swaps/handlers/swap-status.handler.ts` | 24–104 | `getSingleSwapStatus()` – polls OneBalance, maps status to `COMPLETED` (line 46) |
| `src/modules/swaps/handlers/swap-status.handler.ts` | 114–220 | `getLifiSwapStatus()` – polls LI.FI, maps to `COMPLETED` (line 177) |
| `src/modules/swaps/handlers/swap-execution.handler.ts` | 160–206 | `pollExecutionStatusInBackground()` – background poll, sets `COMPLETED` (line 171) |

**When status becomes COMPLETED:**
- Frontend polls `GET /swaps/:quoteId/status` → `SwapStatusHandler` runs.
- OneBalance/LI.FI returns `COMPLETED` or `SUCCESS`.
- Handler updates `Swap` and `Transaction` in DB.
- **Notification trigger:** Right after DB update when `mappedStatus === TransactionStatus.COMPLETED`.

**userId available at:** `swap.userId` (Swap belongs to User).

### Deposit Received – Files Involved

| File | Lines | Role |
|------|-------|------|
| `src/modules/moonpay-commerce/webhooks/moonpay-commerce-webhook.controller.ts` | 80–82 | `DEPOSIT_TX_CONFIRMED` → `handleDepositTxConfirmed()` |
| `src/modules/moonpay-commerce/webhooks/moonpay-commerce-webhook.controller.ts` | 111–177 | Creates `Transaction` with `status: COMPLETED` for `wallet.userId` |
| `src/modules/moonpay-onramps/webhooks/moonpay-onramps-webhook.controller.ts` | 82–84 | `transaction_updated` → `handleTransactionUpdated()` |
| `src/modules/moonpay-onramps/webhooks/moonpay-onramps-webhook.controller.ts` | 196–234 | Updates existing tx to `COMPLETED`; has `existing.userId` |
| `src/modules/moonpay-onramps/webhooks/moonpay-onramps-webhook.controller.ts` | 293–351 | `createCompletedTransaction()` – creates new COMPLETED tx; has `userId` from `findUserByWalletAddress` |

**When deposit is confirmed:**
- **MoonPay Commerce (Helio):** Webhook `DEPOSIT_TX_CONFIRMED` → `handleDepositTxConfirmed` → creates `Transaction` with `userId: wallet.userId`.
- **MoonPay On-Ramps (Buy):** Webhook `transaction_updated` with `data.status === 'completed'` → updates or creates `Transaction` with `status: COMPLETED`; `userId` from `existing.userId` or `findUserByWalletAddress(data.walletAddress)`.

**userId available at:**
- MoonPay Commerce: `wallet.userId` (from `prisma.wallet.findFirst` where `ethereumAddress = recipientAddress`).
- MoonPay On-Ramps: `existing.userId` or `userId` from `findUserByWalletAddress(data.walletAddress)`.

---

## 9. Notification Examples: Deposit & Swap

### Example 1: Swap Completed

**Trigger:** When swap status transitions to `COMPLETED` in:

- `SwapStatusHandler.getSingleSwapStatus()` (lines 46–47, 69–78)
- `SwapStatusHandler.getLifiSwapStatus()` (lines 177–178, 191–200)
- `SwapExecutionHandler.pollExecutionStatusInBackground()` (lines 171–198)

**Integration point (conceptual – not implemented):**

```typescript
// In SwapStatusHandler.getSingleSwapStatus() – after DB update when mappedStatus === COMPLETED
if (mappedStatus === TransactionStatus.COMPLETED && swap.status !== TransactionStatus.COMPLETED) {
  // ... existing prisma.swap.update and prisma.transaction.update ...
  
  // NEW: Send notification
  await this.notificationsService.sendToUser(swap.userId, {
    headings: { en: 'Swap Complete!' },
    contents: {
      en: `Your swap of ${swap.fromAmount} ${swap.fromAsset} to ${swap.toAmount} ${swap.toAsset} is complete.`,
    },
    data: {
      deepLink: `avvio://activity/transaction/${transaction.id}`,
    },
  });
}
```

**Notification payload (OneSignal Create Notification):**

```json
{
  "app_id": "ONESIGNAL_APP_ID",
  "headings": { "en": "Swap Complete!" },
  "contents": {
    "en": "Your swap of 0.1 ETH to 345.21 USDC is complete."
  },
  "include_external_user_ids": ["clx7abc123xyz"],
  "data": {
    "deepLink": "avvio://activity/transaction/clx7tx456abc"
  },
  "channel_for_external_user_ids": "push"
}
```

### Example 2: Deposit Received

**Trigger A – MoonPay Commerce:** `MoonPayCommerceWebhookController.handleDepositTxConfirmed()` (lines 111–177) – creates `Transaction` with `status: COMPLETED` for `wallet.userId`.

**Trigger B – MoonPay On-Ramps:** `MoonPayOnrampsWebhookController.handleTransactionUpdated()` (lines 196–234) or `createCompletedTransaction()` (lines 293–351) – when `data.status === 'completed'`.

**Integration point (conceptual – not implemented):**

```typescript
// In MoonPayCommerceWebhookController.handleDepositTxConfirmed() – after prisma.transaction.create
await this.notificationsService.sendToUser(wallet.userId, {
  headings: { en: 'Deposit Received' },
  contents: {
    en: `You received ${data.amount} ${assetSymbol} on ${data.blockchain}.`,
  },
  data: {
    deepLink: `avvio://activity/transaction/${newTransactionId}`,
  },
});
```

```typescript
// In MoonPayOnrampsWebhookController – after status update to COMPLETED (handleTransactionUpdated or createCompletedTransaction)
await this.notificationsService.sendToUser(existing.userId, {
  headings: { en: 'Deposit Received' },
  contents: {
    en: `Your purchase of ${data.quoteCurrencyAmount} ${assetSymbol} is complete.`,
  },
  data: {
    deepLink: `avvio://activity/transaction/${existing.id}`,
  },
});
```

**Notification payload (Deposit):**

```json
{
  "app_id": "ONESIGNAL_APP_ID",
  "headings": { "en": "Deposit Received" },
  "contents": {
    "en": "You received 100 USDC on ethereum."
  },
  "include_external_user_ids": ["clx7abc123xyz"],
  "data": {
    "deepLink": "avvio://activity/transaction/clx7tx789def"
  },
  "channel_for_external_user_ids": "push"
}
```

---

## 10. iOS & Android Compatibility

### OneSignal Support

OneSignal supports **iOS** (APNs) and **Android** (FCM) via a single app configuration. The same `app_id` and backend API calls work for both platforms.

### iOS Requirements

| Requirement | Details |
|-------------|---------|
| Push Notifications capability | Add in Xcode → Signing & Capabilities |
| APNs credentials | p8 token (recommended) or p12 certificate in OneSignal dashboard |
| Notification Service Extension | Optional, for rich media and confirmed delivery |

### Android Requirements

| Requirement | Details |
|-------------|---------|
| FCM / Firebase | Configure in Firebase Console; add `google-services.json` |
| OneSignal dashboard | Upload FCM server key (legacy) or use Firebase Cloud Messaging v1 |

### API Parity

The Create Notification API uses the same parameters for iOS and Android. Platform-specific options (e.g. `mutable_content` for iOS, `big_picture` for Android) can be added later.

**Context7 Reference:** OneSignal docs state that you can manage multiple platforms (iOS, Android, Huawei, Amazon, Web) under a single OneSignal app.

---

## 11. Context7-Verified OneSignal References

### Sending to User by external_id

- **Create Notification API:** `include_external_user_ids: ["user-id"]` or `include_aliases: { "external_id": ["user-id"] }`
- **Transactional messages:** Use `include_aliases` with `external_id` for targeting.

**Source:** OneSignal Documentation – [Transactional messages](https://documentation.onesignal.com/docs/en/transactional-messages), [Push fallback method](https://documentation.onesignal.com/docs/en/push-fallback-method)

### Node.js / Server SDK

- Use `@onesignal/node-onesignal` with `createConfiguration({ restApiKey })` and `DefaultApi`.
- Target users: `notification.include_aliases = { external_id: ['user123'] }` or `include_external_user_ids`.
- For push: `notification.target_channel = 'push'`.

**Source:** Context7 – [OneSignal Node API](https://context7.com/onesignal/onesignal-node-api/llms.txt)

### React Native SDK

- Initialize: `OneSignal.initialize(ONE_SIGNAL_APP_ID)`.
- Login: `OneSignal.login('USER_EXTERNAL_ID')` – associates device with external_id.
- Logout: `OneSignal.logout()`.
- Subscription ID: `await OneSignal.User.pushSubscription.getIdAsync()` (preferred over deprecated `getPushSubscriptionId()`).

**Source:** Context7 – [OneSignal React Native](https://github.com/onesignal/react-native-onesignal/blob/main/MIGRATION_GUIDE.md)

### Deep Linking

- Use `data` (Additional Data) in the API instead of Launch URL for flexibility.
- Handle in app via `OneSignal.Notifications.addClickListener` and `event.notification.additionalData`.

**Source:** OneSignal Documentation – [Deep linking](https://documentation.onesignal.com/docs/en/deep-linking)

### Aliases / external_id

- "You must set an External ID before using custom aliases."
- "OneSignal uses a unique onesignal_id to identify users. This ID only stays consistent across Subscriptions if they have the same external_id."

**Source:** OneSignal Documentation – [Aliases](https://documentation.onesignal.com/docs/en/aliases)

---

## Summary Checklist

| Area | Item |
|------|------|
| **Backend** | Add `PushSubscription` model, run migration |
| **Backend** | Add `ONESIGNAL_APP_ID`, `ONESIGNAL_REST_API_KEY` to `.env` and config |
| **Backend** | Create `NotificationsModule` (provider, handlers, service, controller) |
| **Backend** | Register `NotificationsModule` in `AppModule` |
| **Backend** | Integrate `NotificationsService` into `SwapStatusHandler`, MoonPay webhooks |
| **Frontend** | Install `react-native-onesignal`, configure iOS/Android |
| **Frontend** | Initialize SDK, call `login`/`logout`, POST subscribe, handle deep links |

---

*Document generated for Anzo backend. All OneSignal behavior verified against Context7 documentation (https://context7.com/websites/onesignal_en).*
