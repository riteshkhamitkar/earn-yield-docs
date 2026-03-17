# OneSignal Push Notifications — Integration Guide

## Overview

This document is the definitive guide for integrating OneSignal push notifications into the Anzo backend (NestJS) and the Avvio mobile app (React Native). It covers the complete architecture, every concept involved, the end-to-end workflow, the backend module design, the frontend integration steps, and how to trigger notifications from existing business logic.

**What this integration enables:**

- Sending real-time push notifications to users on iOS and Android when backend events occur (swap completed, transfer received, yield matured, etc.)
- Targeting users across all their devices with a single API call
- Supporting rich notification content, templates, and deep linking into specific app screens
- Storing device subscription records in the Anzo database for analytics and debugging

---

## Table of Contents

1. [Key Concepts — OneSignal Terminology](#1-key-concepts--onesignal-terminology)
2. [Why `external_id` (Not `subscription_id`) — Architecture Decision](#2-why-external_id-not-subscription_id--architecture-decision)
3. [How Push Notifications Work (iOS vs Android)](#3-how-push-notifications-work-ios-vs-android)
4. [End-to-End System Architecture](#4-end-to-end-system-architecture)
5. [Detailed Workflow — Step by Step](#5-detailed-workflow--step-by-step)
6. [Backend Implementation Plan](#6-backend-implementation-plan)
7. [Frontend Implementation Plan (Avvio Mobile)](#7-frontend-implementation-plan-avvio-mobile)
8. [Triggering Notifications from Business Logic](#8-triggering-notifications-from-business-logic)
9. [Templates & Custom Data](#9-templates--custom-data)
10. [Deep Linking](#10-deep-linking)
11. [Real-Life User Journey — Complete Example](#11-real-life-user-journey--complete-example)
12. [OneSignal REST API Reference (Relevant Endpoints)](#12-onesignal-rest-api-reference-relevant-endpoints)
13. [Troubleshooting & FAQ](#13-troubleshooting--faq)

---

## 1. Key Concepts — OneSignal Terminology

Before diving into the implementation, it is essential to understand how OneSignal models users and devices. These are official OneSignal concepts.

### 1.1. User

A **User** in OneSignal represents a single person using your application. A user can have multiple subscriptions (devices). Users are anonymous until you identify them with an External ID.

### 1.2. External ID

The **External ID** is **your** application's unique user identifier that you assign to a OneSignal User. In Anzo's case, this is the `userId` from the `User` table in your Prisma schema (the `id` field on the `User` model).

When the frontend calls `OneSignal.login(userId)`, it tells OneSignal: "The person using this device is the user with this ID in my system." OneSignal then links that device's subscription to that user.

**This is the single most important concept in this entire integration.** The backend will use this `external_id` to target notifications — it never needs to know about specific devices.

> **Source:** [OneSignal Docs — Users](https://documentation.onesignal.com/docs/en/users)
> "Once an External ID is used via `OneSignal.login`, any existing OneSignal ID tied to the current Subscription is replaced by the one associated with that External ID. Subscriptions across platforms (Push, Email, SMS) will be merged under the same OneSignal ID if they share the same External ID."

### 1.3. OneSignal ID

The **OneSignal ID** is an internal UUID v4 that OneSignal auto-generates to represent a user within their system. You do not set this. It is created automatically when:

- A user first opens the app with the OneSignal SDK integrated
- A new web push subscription is created
- A user is created via the OneSignal API or CSV import
- A user logs out (via `OneSignal.logout`)

Once `OneSignal.login(external_id)` is called, the OneSignal ID becomes tied to that External ID, and all future subscriptions that call `login` with the same External ID are merged under the same OneSignal ID.

### 1.4. Subscription

A **Subscription** represents a specific channel through which a user can receive messages. OneSignal supports four subscription types:

| Type | What It Represents |
|:---|:---|
| **Mobile Push** | An iOS or Android device that can receive push notifications |
| **Web Push** | A browser that has opted in to web push notifications |
| **Email** | An email address |
| **SMS** | A phone number |

Each subscription has a unique **Subscription ID** (sometimes called `subscription_id` or `player_id` in older docs). A single User can have many subscriptions — for example, one iPhone, one Android tablet, and one email address.

> **Source:** [OneSignal Docs — Subscriptions](https://documentation.onesignal.com/docs/en/subscriptions)
> "Mobile Subscriptions are automatically generated when a user installs your app and opens it with the OneSignal SDK integrated. Each mobile subscription is intrinsically linked to the specific device and push token it was created on."

### 1.5. Alias

An **Alias** is a key-value label that identifies a user. The `external_id` is the most common alias. You can also define custom aliases (e.g., `crm_id`, `email`). When sending a notification, you target users via their aliases:

```json
{
  "include_aliases": {
    "external_id": ["user_abc_123"]
  }
}
```

### 1.6. Relationship Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                        OneSignal User                           │
│                                                                 │
│  OneSignal ID: a1b2c3d4-xxxx-xxxx-xxxx-xxxxxxxxxxxx (auto)     │
│  External ID:  usr_anzo_12345 (your Anzo userId)               │
│                                                                 │
│  ┌─────────────────────┐  ┌─────────────────────┐              │
│  │  Subscription #1    │  │  Subscription #2    │              │
│  │  Type: iOS Push     │  │  Type: Android Push │              │
│  │  ID: sub_aaa...     │  │  ID: sub_bbb...     │              │
│  │  Token: APNs token  │  │  Token: FCM token   │              │
│  └─────────────────────┘  └─────────────────────┘              │
│                                                                 │
│  (Could also have Email, SMS, Web Push subscriptions)          │
└─────────────────────────────────────────────────────────────────┘
```

A single Anzo `userId` maps to a single OneSignal User, which can have multiple subscriptions (devices). When the backend sends a notification to `external_id: ["usr_anzo_12345"]`, OneSignal delivers it to **all** of that user's subscriptions automatically.

---

## 2. Why `external_id` (Not `subscription_id`) — Architecture Decision

There are two ways to target notifications in OneSignal:

1. **By Subscription ID** — Target a specific device directly
2. **By External ID (Alias)** — Target a user, and OneSignal delivers to all their devices

### Comparison

| Aspect | Subscription ID Approach | **External ID Approach (Recommended)** |
|:---|:---|:---|
| **What the backend needs** | The specific device's `subscription_id` | Only the Anzo `userId` |
| **Multi-device support** | Must query all subscription IDs for a user and include them all in the API call. If the user adds a new device, the backend must track it. | Automatic. OneSignal handles delivery to all devices linked to that `external_id`. Zero backend effort. |
| **Backend complexity** | High. Must store, query, and manage device subscriptions. Frontend must pass `subscription_id` with every action that could trigger a notification. | Low. Backend just says "send to user X." Completely decoupled from device management. |
| **Frontend responsibility** | Must pass `subscription_id` to backend on every relevant API call. Fragile — what if the subscription changes? | Only needs to call `OneSignal.login(userId)` once at login. That's it. |
| **Future scalability** | Does not scale to email/SMS channels. | Same `external_id` pattern works for email and SMS notifications in the future. |
| **OneSignal's recommendation** | Valid for one-off tests or targeting a single known device. | **Explicitly recommended best practice** for user-centric messaging. |

> **Source:** [OneSignal Docs — Transactional Messages](https://documentation.onesignal.com/docs/en/transactional-messages)
> The Create Message API targets users via aliases (`include_aliases` with `external_id`), email addresses, phone numbers, or subscription IDs. The alias-based approach is recommended for transactional messages.

**Decision: We use `external_id` (the Anzo `userId`) for all notification targeting.**

---

## 3. How Push Notifications Work (iOS vs Android)

### 3.1. The Push Notification Pipeline

Push notifications do not go directly from your server to the user's phone. They go through platform-specific gateways:

```
┌──────────────┐     ┌────────────────┐     ┌──────────────────┐     ┌──────────────┐
│ Anzo Backend │────▶│ OneSignal API  │────▶│ APNs (Apple) or  │────▶│ User's Phone │
│              │     │                │     │ FCM  (Google)    │     │              │
└──────────────┘     └────────────────┘     └──────────────────┘     └──────────────┘
```

- **APNs (Apple Push Notification service):** Apple's gateway for delivering notifications to iOS devices
- **FCM (Firebase Cloud Messaging):** Google's gateway for delivering notifications to Android devices

### 3.2. How OneSignal Knows the Platform

**The backend does NOT need to specify iOS or Android.** Here is how it works:

1. When the Avvio app is installed and opened, the OneSignal SDK initializes.
2. The SDK detects the platform automatically (`iOS` or `Android`) based on the operating system it is running on.
3. The SDK communicates with Apple (APNs) or Google (FCM) to obtain a **push token** — a unique identifier that the platform's push gateway uses to route the notification to that specific device.
4. The SDK sends this push token, along with the platform type, to OneSignal's servers.
5. OneSignal stores this as a **Subscription** — it knows: "Subscription `sub_abc` is an iOS device with APNs token `xyz`."
6. When `OneSignal.login(userId)` is called, OneSignal also knows: "This subscription belongs to user `userId`."

So when the backend later calls:

```json
{
  "include_aliases": { "external_id": ["userId"] },
  "target_channel": "push"
}
```

OneSignal looks up all push subscriptions for that user, determines which are iOS and which are Android, and sends through the correct gateway (APNs or FCM) for each one. **The backend is completely abstracted from platform specifics.**

### 3.3. iOS-Specific Requirements

- **APNs Authentication Key (`.p8` file):** Must be uploaded to the OneSignal dashboard. This is how OneSignal authenticates with Apple to send pushes on your behalf.
- **Notification Service Extension:** A small iOS extension required for features like badge counts, images in notifications, and confirmed delivery analytics. The frontend developer must add this to the Xcode project during setup.
- **Permission Prompt:** iOS requires the user to grant notification permission explicitly. The SDK provides `OneSignal.Notifications.requestPermission(true)` for this.

### 3.4. Android-Specific Requirements

- **Firebase Server Key / Service Account JSON:** Must be configured in the OneSignal dashboard. This is how OneSignal authenticates with Google FCM.
- **No Permission Prompt (Android 12 and below):** Push notification permission is granted by default on most Android versions. Android 13+ does require a runtime permission, which the OneSignal SDK can request.

### 3.5. OneSignal Dashboard Setup (Prerequisites)

Before any code is written, the following must be configured in the [OneSignal Dashboard](https://dashboard.onesignal.com):

1. Create a OneSignal App (this gives you the **App ID**)
2. Configure the **iOS** platform by uploading the APNs `.p8` key
3. Configure the **Android** platform by adding the Firebase Server Key or Google Service Account JSON
4. Generate a **REST API Key** from Settings > Keys & IDs (this is what the backend uses for API authentication)

---

## 4. End-to-End System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Avvio Mobile App (Frontend)                      │
│                        React Native + OneSignal SDK                     │
│                                                                         │
│  On Launch:                                                             │
│    1. OneSignal.initialize(APP_ID) → SDK registers device with APNs/FCM│
│    2. SDK receives subscription_id from OneSignal                       │
│                                                                         │
│  On Login:                                                              │
│    3. OneSignal.login(userId) → Links device to Anzo userId             │
│    4. POST /api/v1/notifications/subscribe → Stores subscription in DB  │
│                                                                         │
│  On Logout:                                                             │
│    5. OneSignal.logout() → Disassociates device from user               │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ HTTPS (REST API)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Anzo Backend (NestJS)                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                     Notifications Module                          │  │
│  │                                                                   │  │
│  │  ┌──────────────────┐    ┌──────────────────────────────────────┐ │  │
│  │  │   Controller     │    │        Service (Facade)              │ │  │
│  │  │   POST /subscribe│───▶│  subscribeDevice()                  │ │  │
│  │  └──────────────────┘    │  sendToUser()                       │ │  │
│  │                          └──────────────┬───────────────────────┘ │  │
│  │                                         │                         │  │
│  │                          ┌──────────────┴───────────────────────┐ │  │
│  │                          │           Handlers                   │ │  │
│  │                          │  ┌───────────────────────────────┐   │ │  │
│  │                          │  │  SubscriptionHandler          │   │ │  │
│  │                          │  │  - Upserts device in DB       │   │ │  │
│  │                          │  └───────────────────────────────┘   │ │  │
│  │                          │  ┌───────────────────────────────┐   │ │  │
│  │                          │  │  MessageSendingHandler        │   │ │  │
│  │                          │  │  - Builds OneSignal payload   │   │ │  │
│  │                          │  │  - Calls OneSignal REST API   │   │ │  │
│  │                          │  └───────────────────────────────┘   │ │  │
│  │                          └──────────────────────────────────────┘ │  │
│  │                                                                   │  │
│  │  ┌───────────────────────────────────────────────────────────┐   │  │
│  │  │  OneSignalProvider                                        │   │  │
│  │  │  - Initializes @onesignal/node-onesignal client           │   │  │
│  │  │  - Authenticates with REST API Key                        │   │  │
│  │  └───────────────────────────────────────────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Other Modules (Swaps, Transfers, Yield, Bridge, Savings...)     │  │
│  │  → Inject NotificationsService                                    │  │
│  │  → Call notificationsService.sendToUser(userId, message)          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Database (PostgreSQL via Prisma)                                  │  │
│  │  ┌─────────────────┐  ┌──────────────────────────┐               │  │
│  │  │      User       │─▶│   PushSubscription       │               │  │
│  │  │                 │  │   - subscriptionId        │               │  │
│  │  │                 │  │   - platform (ios/android)│               │  │
│  │  │                 │  │   - isActive              │               │  │
│  │  └─────────────────┘  └──────────────────────────┘               │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────┬─────────────────────────────────────┘
                                    │ HTTPS
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          OneSignal API                                   │
│                   https://api.onesignal.com                              │
│                                                                          │
│  POST /api/v1/notifications                                              │
│  {                                                                       │
│    "app_id": "...",                                                      │
│    "include_aliases": { "external_id": ["anzo_user_id"] },              │
│    "target_channel": "push",                                             │
│    "contents": { "en": "Your swap is complete!" }                        │
│  }                                                                       │
│                                                                          │
│  OneSignal resolves external_id → finds all subscriptions (devices)     │
│  → sends to APNs (iOS) and/or FCM (Android) as appropriate              │
└─────────────────────────────────────────────────────────────────────────┘
```

### Module Structure

```
src/modules/notifications/
├── notifications.module.ts          # Module definition
├── notifications.controller.ts      # API endpoint (POST /subscribe)
├── notifications.service.ts         # Facade service
│
├── handlers/
│   ├── subscription.handler.ts      # DB operations for device registration
│   └── message-sending.handler.ts   # Constructs & sends OneSignal API calls
│
├── providers/
│   └── onesignal.provider.ts        # Initializes OneSignal Node.js client
│
└── dto/
    └── subscribe.dto.ts             # Validation for subscribe endpoint
```

---

## 5. Detailed Workflow — Step by Step

### Phase 1: Device Registration (Happens Once Per Device, Per Login)

**Step 1 — App Launch: SDK Initialization**

When the Avvio app launches, the OneSignal React Native SDK is initialized with the App ID from the OneSignal dashboard.

```
OneSignal.initialize("YOUR_ONESIGNAL_APP_ID")
```

Behind the scenes, the SDK:
- Detects the platform (iOS or Android) automatically
- Contacts APNs (if iOS) or FCM (if Android) to obtain a **push token**
- Sends this push token to OneSignal's servers
- OneSignal creates a **Subscription** for this device and returns a **Subscription ID**

At this point, the device is registered with OneSignal but is **anonymous** — it is not linked to any Anzo user yet.

**Step 2 — User Login: Identity Association**

After the user logs into Avvio and the app has the Anzo `userId` from the JWT:

```
OneSignal.login("anzo_user_id_abc123")
```

This is the critical call. It tells OneSignal: "This device (Subscription) belongs to the user with `external_id` = `anzo_user_id_abc123`."

If the user has logged in from another device before, OneSignal merges both subscriptions under the same user. If this is their first login, OneSignal creates the association.

**Step 3 — Backend Registration (Optional but Recommended)**

The frontend sends the device's Subscription ID to the Anzo backend:

```
POST /api/v1/notifications/subscribe
Authorization: Bearer <jwt_token>
{
  "subscriptionId": "onesignal-subscription-uuid",
  "platform": "ios"
}
```

The backend stores this in the `PushSubscription` table. This record is **not required for sending notifications** (the backend uses `external_id`, not `subscription_id`), but it is valuable for:
- Debugging: "Which devices does this user have registered?"
- Analytics: "How many iOS vs Android devices do we have?"
- User preferences: Allowing users to manage notification settings per device in the future

**Step 4 — User Logout: Disassociation**

When the user logs out of Avvio:

```
OneSignal.logout()
```

This disassociates the device from the user in OneSignal. The subscription still exists, but it is now anonymous again. The next user to log in on this device will be linked instead.

### Phase 2: Sending a Notification (Happens Whenever a Backend Event Occurs)

**Step 5 — Business Event Triggers Notification**

An event occurs in the backend — for example, `SwapStatusHandler` detects that a swap has completed. The handler calls:

```typescript
await this.notificationsService.sendToUser(swap.userId, {
  headings: { en: "Swap Complete" },
  contents: { en: "Your swap of 0.1 ETH to 3,450 USDC is complete!" },
  data: { deepLink: "avvio://activity/transaction/txn_123" }
});
```

**Step 6 — Backend Calls OneSignal API**

The `MessageSendingHandler` constructs a request to the OneSignal REST API:

```
POST https://api.onesignal.com/api/v1/notifications
Authorization: key YOUR_REST_API_KEY
Content-Type: application/json

{
  "app_id": "YOUR_APP_ID",
  "include_aliases": {
    "external_id": ["anzo_user_id_abc123"]
  },
  "target_channel": "push",
  "headings": { "en": "Swap Complete" },
  "contents": { "en": "Your swap of 0.1 ETH to 3,450 USDC is complete!" },
  "data": { "deepLink": "avvio://activity/transaction/txn_123" }
}
```

Note the key fields:
- `include_aliases.external_id` — Targets the user by Anzo `userId`. OneSignal resolves this to all their devices.
- `target_channel: "push"` — Specifies this is a push notification (not email/SMS).
- `data` — Custom key-value payload delivered silently alongside the notification, used for deep linking.

**Step 7 — OneSignal Delivers**

OneSignal receives the API call and:
1. Looks up the user with `external_id = "anzo_user_id_abc123"`
2. Finds all push subscriptions for that user (e.g., one iOS device, one Android device)
3. Sends through APNs for iOS subscriptions
4. Sends through FCM for Android subscriptions
5. Returns a response with the notification ID and recipient count

**Step 8 — Device Receives Notification**

The phone receives the push notification and displays it in the notification tray. If the user taps the notification, the OneSignal SDK in the app fires a click event handler, which can read the `data` payload and navigate to the appropriate screen.

---

## 6. Backend Implementation Plan

This follows the Anzo backend's Service & Handler pattern as defined in `rules.md`.

### 6.1. Environment Variables

Add to `.env` and `.env.example`:

```env
# ============================================
# OneSignal Push Notifications
# ============================================
ONESIGNAL_APP_ID=your-onesignal-app-id-uuid
ONESIGNAL_REST_API_KEY=your-onesignal-rest-api-key
```

Add to `src/config/validation.schema.ts`:

```
ONESIGNAL_APP_ID: Joi.string().required(),
ONESIGNAL_REST_API_KEY: Joi.string().required(),
```

Add to `src/config/configuration.ts` (inside the returned config object):

```
onesignal: {
  appId: process.env.ONESIGNAL_APP_ID,
  restApiKey: process.env.ONESIGNAL_REST_API_KEY,
},
```

### 6.2. Database Schema

Add the `PushSubscription` model to `prisma/schema.prisma`:

```prisma
model PushSubscription {
  id              String   @id @default(cuid())
  userId          String
  subscriptionId  String   @unique   // OneSignal Subscription ID from the SDK
  platform        String              // "ios" or "android"
  isActive        Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
}
```

Add the relation to the `User` model:

```prisma
model User {
  // ... existing fields ...
  PushSubscription PushSubscription[]
}
```

Run migration: `npx prisma migrate dev --name add_push_subscription_model`

### 6.3. Install Dependency

```bash
npm install @onesignal/node-onesignal
```

> **Source:** [OneSignal Node.js SDK (GitHub)](https://github.com/onesignal/onesignal-node-api)
> This is the official server-side SDK for Node.js, providing an OpenAPI client to integrate OneSignal services with backend events.

### 6.4. Module Files

**`onesignal.provider.ts`** — Initializes the OneSignal API client using the REST API Key from configuration. This provider is injected into the `MessageSendingHandler`. The client authenticates all requests to OneSignal using the REST API Key passed as `restApiKey` in the configuration.

> **Source:** [OneSignal Node.js SDK — Configuration](https://github.com/onesignal/onesignal-node-api/blob/main/README.md)
> ```javascript
> const configuration = OneSignal.createConfiguration({
>   restApiKey: '<YOUR_REST_API_KEY>',
> });
> const client = new OneSignal.DefaultApi(configuration);
> ```

**`subscribe.dto.ts`** — Validates the request body for the `/subscribe` endpoint using `class-validator` decorators. Accepts `subscriptionId` (string, required) and `platform` (must be `"ios"` or `"android"`).

**`subscription.handler.ts`** — Handles database operations for device registration. Uses Prisma's `upsert` on the unique `subscriptionId` field — if the device has been registered before, it updates the record; otherwise, it creates a new one. This means if a user reinstalls the app or logs in from a different account on the same device, the record is updated correctly.

**`message-sending.handler.ts`** — The core handler that constructs and sends notifications via the OneSignal API. It:
- Creates a `Notification` object from the OneSignal SDK
- Sets `app_id` from configuration
- Sets `include_aliases.external_id` to `[userId]` — this is the targeting mechanism
- Sets `target_channel` to `"push"`
- Populates content (`headings`, `contents`) or a `template_id`
- Attaches any custom `data` payload (used for deep linking)
- Calls `client.createNotification(notification)` and logs the result
- **Does NOT re-throw errors** — a failed notification should never break a core business flow like a swap or transfer

**`notifications.service.ts`** — The facade service that controllers and other modules interact with. Exposes two methods:
- `subscribeDevice(userId, dto)` — Delegates to `SubscriptionHandler`
- `sendToUser(userId, message)` — Delegates to `MessageSendingHandler`

**`notifications.controller.ts`** — Exposes the `POST /notifications/subscribe` endpoint. Protected by `AuthGuard`. Extracts `req.user.userId` from the JWT and passes it to the service along with the validated DTO.

**`notifications.module.ts`** — Registers the controller, service, handlers, and provider. **Exports `NotificationsService`** so that other modules (Swaps, Transfers, Yield, etc.) can import `NotificationsModule` and inject the service.

### 6.5. Register in AppModule

Add `NotificationsModule` to the `imports` array in `src/app.module.ts`.

---

## 7. Frontend Implementation Plan (Avvio Mobile)

### 7.1. Install the OneSignal React Native SDK

```bash
yarn add react-native-onesignal
cd ios && pod install
```

### 7.2. iOS Setup (Xcode)

1. Add the **Notification Service Extension** target in Xcode. This is required for:
   - Badge count management
   - Image/media attachments in notifications
   - Confirmed delivery tracking (so OneSignal can tell you if a notification was actually displayed)

2. Upload the APNs `.p8` Authentication Key to the OneSignal dashboard under Settings > Platforms > iOS.

> **Source:** [OneSignal Docs — React Native SDK Setup](https://documentation.onesignal.com/docs/en/react-native-sdk-setup)

### 7.3. Android Setup

1. Add your Firebase project's **Server Key** or **Service Account JSON** to the OneSignal dashboard under Settings > Platforms > Android.
2. Ensure the `google-services.json` file is in the Android project.

### 7.4. SDK Initialization

In the app's root component (e.g., `App.tsx`), initialize the SDK:

```
OneSignal.Debug.setLogLevel(LogLevel.Verbose)   // Remove in production
OneSignal.initialize("YOUR_ONESIGNAL_APP_ID")
OneSignal.Notifications.requestPermission(true)  // Prompts user for permission
```

> **Source:** [OneSignal Docs — React Native SDK Setup](https://documentation.onesignal.com/docs/en/react-native-sdk-setup)
> ```javascript
> import {OneSignal, LogLevel} from 'react-native-onesignal';
> OneSignal.Debug.setLogLevel(LogLevel.Verbose);
> OneSignal.initialize('YOUR_APP_ID');
> OneSignal.Notifications.requestPermission(false);
> ```

### 7.5. User Login — The Critical Step

When the user logs in and the app has the Anzo `userId`:

```
OneSignal.login(userId)
```

This single call is what makes the entire `external_id` architecture work. It tells OneSignal to link the current device to the given Anzo user.

> **Source:** [OneSignal Docs — Mobile SDK Reference](https://documentation.onesignal.com/docs/en/mobile-sdk-reference)
> "Sets the user context to a provided external ID, unifying subscriptions and properties. If the external ID exists, the SDK switches to that user; otherwise, anonymous data is saved under the new ID. Retries automatically on network failures."

After calling `login`, also send the subscription ID to the Anzo backend:

```
const subscriptionId = OneSignal.User.pushSubscription.getPushSubscriptionId()
if (subscriptionId) {
  // POST to /api/v1/notifications/subscribe
  api.notifications.subscribe({
    subscriptionId: subscriptionId,
    platform: Platform.OS  // "ios" or "android"
  })
}
```

### 7.6. User Logout

```
OneSignal.logout()
```

This disassociates the device from the user. Important for shared devices or account switching.

### 7.7. Handling Notification Clicks (Deep Linking)

Set up a global listener for notification taps:

```
OneSignal.Notifications.addClickListener((event) => {
  const data = event.notification.additionalData
  if (data?.deepLink) {
    // Parse the deep link URL and navigate
    // e.g., "avvio://activity/transaction/txn_123"
    navigateToScreen(data.deepLink)
  }
})
```

---

## 8. Triggering Notifications from Business Logic

Once the `NotificationsModule` is set up and exported, any module in the Anzo backend can send notifications by:

1. Importing `NotificationsModule` in their own module (or relying on it being globally available)
2. Injecting `NotificationsService` into their handler
3. Calling `sendToUser(userId, message)`

### Example Integration Points

| Module | Event | Notification Content |
|:---|:---|:---|
| **Swaps** | Swap completed | "Your swap of 0.1 ETH to 3,450 USDC is complete!" |
| **Swaps** | Swap failed | "Your swap of 0.1 ETH failed. Please try again." |
| **Transfers** | Transfer sent | "You sent 100 USDC to 0xABC...123." |
| **Transfers** | Transfer received (if detectable) | "You received 50 USDC." |
| **Yield** | Deposit confirmed | "Your deposit of 500 USDC into yoUSD is confirmed." |
| **Yield** | Redemption ready | "Your yoUSD redemption of 500 USDC is ready to claim." |
| **Bridge** | KYC approved | "Your identity verification is complete. You can now on-ramp." |
| **Bridge** | On-ramp payout received | "Your deposit of $500 has been converted. 499.50 USDC arriving." |
| **Bridge** | Off-ramp payment sent | "$500.00 has been sent to your bank account." |
| **Savings** | Goal reached | "Congratulations! Your savings goal 'Trip to Mexico' is complete!" |

### Pattern for Integration

In any handler that should trigger a notification:

```typescript
// 1. Import
import { NotificationsService } from '../../notifications/notifications.service';

// 2. Inject in constructor
constructor(
  // ... other deps
  private notificationsService: NotificationsService,
) {}

// 3. Call after the business event
await this.notificationsService.sendToUser(userId, {
  headings: { en: "Event Title" },
  contents: { en: "Event description with details." },
  data: { deepLink: "avvio://screen/param" },
});
```

The notification call should be non-blocking — wrap in a try/catch or fire-and-forget if the notification is not critical to the business operation.

---

## 9. Templates & Custom Data

### 9.1. Using Templates

Instead of hardcoding notification text in the backend, you can create templates in the OneSignal dashboard and reference them by ID. This lets non-technical team members edit notification copy without code changes.

**Creating a template:**
1. Go to OneSignal Dashboard > Messages > Templates
2. Create a new template with title and body
3. Use Liquid syntax for dynamic content: `{{ custom_data.amount }}` or `{{ custom_data.asset }}`
4. Copy the Template ID (UUID)

**Sending with a template:**

```json
{
  "app_id": "YOUR_APP_ID",
  "include_aliases": { "external_id": ["userId"] },
  "template_id": "8458af75-4da2-4ecf-afb5-f242a8926cc3",
  "custom_data": {
    "amount": "0.1",
    "asset": "ETH",
    "to_amount": "3450",
    "to_asset": "USDC"
  }
}
```

> **Source:** [OneSignal Docs — Transactional Messages](https://documentation.onesignal.com/docs/en/transactional-messages)
> The API supports `template_id` with `custom_data` for personalized template-based notifications.

### 9.2. Using Direct Content (Inline)

For dynamic content that doesn't fit a template, send headings and contents directly:

```json
{
  "app_id": "YOUR_APP_ID",
  "include_aliases": { "external_id": ["userId"] },
  "target_channel": "push",
  "headings": { "en": "Swap Complete" },
  "contents": { "en": "Your swap of 0.1 ETH to 3,450 USDC is complete!" }
}
```

The `en` key represents English. You can add other language keys for multi-language support (e.g., `"es"`, `"fr"`). OneSignal will deliver the content matching the user's device language, falling back to `"en"` if no match is found.

### 9.3. The `data` Payload

The `data` field is a JSON object that is delivered silently alongside the notification. It is **not displayed** to the user — it is available to the app's code when the notification is received or tapped. Use it for:

- **Deep linking:** `{ "deepLink": "avvio://activity/transaction/txn_123" }`
- **Screen routing:** `{ "screen": "SwapDetail", "transactionId": "txn_123" }`
- **Any metadata** the frontend needs to handle the notification intelligently

---

## 10. Deep Linking

When a user taps a push notification, you want to take them directly to the relevant screen in the app, not just open the app to the home screen.

### 10.1. How It Works

1. **Backend** includes a `data` payload with the notification:
   ```json
   {
     "data": {
       "deepLink": "avvio://activity/transaction/txn_abc123"
     }
   }
   ```

2. **Frontend** has a global click listener that fires when the user taps a notification:
   ```
   OneSignal.Notifications.addClickListener((event) => {
     const data = event.notification.additionalData
     if (data?.deepLink) {
       // Parse URL and navigate to the correct screen
     }
   })
   ```

3. The frontend parses the deep link URL scheme and navigates using React Navigation (or your routing solution).

> **Source:** [OneSignal Docs — Deep Linking](https://documentation.onesignal.com/docs/en/deep-linking)
> "Configure push notifications to open specific content within your app using deep links. Utilize the `url` property for the launch URL or the `data` property (recommended for iOS to suppress browser redirect)."

### 10.2. Recommended Deep Link URL Scheme

Define a consistent URL scheme for the Avvio app:

| Deep Link | Screen | Parameters |
|:---|:---|:---|
| `avvio://activity/transaction/{id}` | Transaction detail | `transactionId` |
| `avvio://swap/status/{id}` | Swap status | `swapId` |
| `avvio://savings/goal/{id}` | Savings goal detail | `goalId` |
| `avvio://bridge/status` | Bridge KYC/account status | none |
| `avvio://home` | Home screen | none |

### 10.3. iOS vs Android Deep Link Behavior

- **Android:** The OneSignal SDK opens the app directly and passes the data payload. The click listener fires and can navigate accordingly.
- **iOS:** By default, if a `url` property is set, iOS may open a browser first. **Use the `data` property instead of `url`** for iOS to avoid the browser redirect. The click listener approach (reading from `additionalData`) works correctly on both platforms.

---

## 11. Real-Life User Journey — Complete Example

Let's walk through the complete flow for a user named **Alice** who swaps ETH to USDC.

### Part 1: Setup (One-Time)

1. **Alice installs the Avvio app on her iPhone.**
   - The app calls `OneSignal.initialize(APP_ID)`.
   - The OneSignal SDK contacts Apple's APNs, gets a push token, and registers the device with OneSignal.
   - OneSignal creates an anonymous subscription: `sub_iphone_abc`.

2. **Alice logs into Avvio.**
   - The app authenticates with Anzo backend, receives a JWT containing `userId: "usr_alice_001"`.
   - The app calls `OneSignal.login("usr_alice_001")`.
   - OneSignal now knows: Subscription `sub_iphone_abc` (iOS) belongs to user with `external_id: "usr_alice_001"`.

3. **Alice also has an Android tablet.**
   - She installs Avvio and logs in with the same account.
   - The app calls `OneSignal.login("usr_alice_001")` on the tablet.
   - OneSignal now knows: `external_id: "usr_alice_001"` has two subscriptions — `sub_iphone_abc` (iOS) and `sub_android_xyz` (Android).

### Part 2: The Swap

4. **Alice initiates a swap of 0.1 ETH to USDC on her iPhone.**
   - Frontend calls `POST /api/v1/swaps/execute`.
   - Backend creates a Transaction record, submits to OneBalance, starts polling for status.

5. **The swap completes.**
   - `SwapStatusHandler` detects the completed status.
   - It updates the Transaction record to `COMPLETED`.
   - It calls:
     ```
     notificationsService.sendToUser("usr_alice_001", {
       headings: { en: "Swap Complete" },
       contents: { en: "Your swap of 0.1 ETH to 3,450.21 USDC is complete!" },
       data: { deepLink: "avvio://activity/transaction/txn_abc" }
     })
     ```

6. **Backend sends the OneSignal API request.**
   ```json
   POST https://api.onesignal.com/api/v1/notifications
   {
     "app_id": "...",
     "include_aliases": { "external_id": ["usr_alice_001"] },
     "target_channel": "push",
     "headings": { "en": "Swap Complete" },
     "contents": { "en": "Your swap of 0.1 ETH to 3,450.21 USDC is complete!" },
     "data": { "deepLink": "avvio://activity/transaction/txn_abc" }
   }
   ```

7. **OneSignal delivers to BOTH devices.**
   - OneSignal finds `external_id: "usr_alice_001"` → two subscriptions.
   - Sends via APNs to Alice's iPhone.
   - Sends via FCM to Alice's Android tablet.
   - Both devices receive the notification.

### Part 3: User Interaction

8. **Alice taps the notification on her iPhone.**
   - The OneSignal SDK fires the click listener.
   - The app reads `additionalData.deepLink = "avvio://activity/transaction/txn_abc"`.
   - The app navigates to the Transaction Detail screen showing the completed swap.

---

## 12. OneSignal REST API Reference (Relevant Endpoints)

### 12.1. Create Notification

This is the primary endpoint the backend uses to send notifications.

```
POST https://api.onesignal.com/api/v1/notifications
```

**Authentication:** Include the REST API Key in the header:
```
Authorization: key YOUR_REST_API_KEY
Content-Type: application/json
```

> **Source:** [OneSignal Docs — Transactional Messages](https://documentation.onesignal.com/docs/en/transactional-messages)

**Request Body — Target by External ID:**

```json
{
  "app_id": "YOUR_APP_ID",
  "include_aliases": {
    "external_id": ["user_id_1", "user_id_2"]
  },
  "target_channel": "push",
  "headings": { "en": "Title" },
  "contents": { "en": "Message body" },
  "data": { "key": "value" }
}
```

**Request Body — Target by Template:**

```json
{
  "app_id": "YOUR_APP_ID",
  "include_aliases": {
    "external_id": ["user_id"]
  },
  "template_id": "template-uuid",
  "custom_data": {
    "amount": "100",
    "currency": "USDC"
  }
}
```

**Response (200 OK):**

```json
{
  "id": "notification-uuid",
  "recipients": 2,
  "external_id": null
}
```

**Key Parameters:**

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `app_id` | string | Yes | Your OneSignal App ID |
| `include_aliases` | object | Yes (for alias targeting) | Object with alias type as key and array of IDs as value. e.g., `{ "external_id": ["user1"] }` |
| `target_channel` | string | No | `"push"`, `"email"`, or `"sms"`. If not set, defaults to push. |
| `headings` | object | No | Notification title. Keys are language codes. `{ "en": "Title" }` |
| `contents` | object | Yes (if no template) | Notification body text. `{ "en": "Body" }` |
| `data` | object | No | Custom key-value data delivered silently with the notification |
| `template_id` | string | No | UUID of a OneSignal template to use instead of inline content |
| `custom_data` | object | No | Data to populate template variables (used with `template_id`) |
| `ios_badge_type` | string | No | `"Increase"`, `"SetTo"`, or `"None"` |
| `ios_badge_count` | number | No | Badge count to set or increment by |
| `mutable_content` | boolean | No | `true` to allow iOS Notification Service Extension to modify content |

### 12.2. Other Useful Endpoints

| Endpoint | Method | Purpose |
|:---|:---|:---|
| `/api/v1/notifications/{id}` | GET | Check delivery status of a sent notification |
| `/api/v1/apps/{app_id}/users/by/external_id/{external_id}` | GET | Look up a user and their subscriptions |
| `/api/v1/apps/{app_id}/users/by/external_id/{external_id}` | DELETE | Delete a user from OneSignal |

---

## 13. Troubleshooting & FAQ

### Q: What happens if the user has not granted notification permission?

OneSignal still creates a Subscription for the device, but it is marked as "unsubscribed" (opted out of push). The `OneSignal.login()` call still works — the user is still identified. If the backend sends a notification to that user, OneSignal will simply skip the unsubscribed device. No error is thrown.

### Q: What if `OneSignal.login()` is called before the SDK finishes initializing?

The SDK queues the login call and executes it once initialization is complete. It also retries automatically on network failures. This is handled internally by the SDK.

### Q: What if the user logs in on a new device?

The frontend calls `OneSignal.login(userId)` on the new device. OneSignal adds the new subscription to the existing user. Now the user has multiple subscriptions, and future notifications will be delivered to all of them.

### Q: What if the OneSignal API call fails? Does the swap/transfer fail too?

No. The `MessageSendingHandler` catches all errors and logs them without re-throwing. A notification failure should never impact core business logic. The swap/transfer/yield operation completes successfully regardless.

### Q: Can we send to multiple users at once?

Yes. The `include_aliases.external_id` field accepts an array:

```json
{
  "include_aliases": {
    "external_id": ["user_1", "user_2", "user_3"]
  }
}
```

### Q: What is the difference between `data` and `custom_data`?

- `data` — Delivered as the silent data payload alongside inline content (`headings` + `contents`). Available on the device via `event.notification.additionalData`.
- `custom_data` — Used to populate template variables when sending with `template_id`. Also available on the device via `additionalData`.

### Q: How do we handle notification preferences (e.g., user wants swap notifications but not marketing)?

This can be implemented later using OneSignal Tags. The frontend sets tags on the user (e.g., `{ "swap_notifications": "enabled", "marketing": "disabled" }`), and the backend (or OneSignal filters) can check these tags before sending. For the initial implementation, all users receive all notifications.

### Q: Rate limits?

The OneSignal REST API has rate limits based on your plan. For the free tier, it is approximately 1 notification per second per API call. For paid plans, the limits are significantly higher. Batch targeting (sending to multiple `external_id`s in one call) is more efficient than individual calls.

### Q: How do we test in development?

1. Set `OneSignal.Debug.setLogLevel(LogLevel.Verbose)` in the app to see detailed SDK logs.
2. Use the OneSignal Dashboard > Messages > New Push to send a test notification to a specific External ID.
3. On the backend, log the OneSignal API response to verify the notification ID and recipient count.
4. For local development, you can use the OneSignal REST API directly via curl or Postman:
   ```bash
   curl -X POST https://api.onesignal.com/api/v1/notifications \
     -H "Authorization: key YOUR_REST_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{
       "app_id": "YOUR_APP_ID",
       "include_aliases": { "external_id": ["your_test_user_id"] },
       "target_channel": "push",
       "contents": { "en": "Test notification from curl!" }
     }'
   ```

---

## Summary

| Component | Responsibility |
|:---|:---|
| **OneSignal Dashboard** | Platform setup (APNs/FCM keys), template creation, analytics |
| **Avvio Frontend (React Native)** | Initialize SDK, call `OneSignal.login(userId)` on login, `OneSignal.logout()` on logout, handle notification clicks for deep linking, send `subscriptionId` to backend |
| **Anzo Backend (NestJS)** | Store device subscriptions in DB, send notifications to users via `include_aliases.external_id` using the OneSignal REST API, trigger notifications from business events |
| **OneSignal API** | Resolves `external_id` to all user devices, routes through APNs (iOS) and FCM (Android), handles delivery, tracks analytics |

The `external_id` (Anzo `userId`) is the single identifier that ties everything together. The frontend sets it via `OneSignal.login()`. The backend targets it via `include_aliases.external_id`. OneSignal handles everything in between.
