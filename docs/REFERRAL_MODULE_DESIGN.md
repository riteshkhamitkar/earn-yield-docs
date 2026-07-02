# Referral Program — Backend Design Spec

> Backend design for the Avvio Referral Program: invite friends, track progress,
> and reward referrers $5 when referred users open a Seismic bank account and
> make their first fiat deposit. Aligns with the mobile spec
> (`referral_home_spec.md`) and Figma "Track referrals" / "Claim code" screens.

- **Date:** 2026-07-02
- **Status:** Proposed — pending team approval
- **Owner:** Backend
- **Base path:** `/api/v1`
- **Architecture:** Approach 1 — direct service hooks in Seismic handlers

---

## Table of contents

1. [Context & goals](#1-context--goals)
2. [Product decisions (locked in)](#2-product-decisions-locked-in)
3. [What already exists](#3-what-already-exists)
4. [Architecture](#4-architecture)
5. [Data model](#5-data-model)
6. [API specification](#6-api-specification)
7. [Business rules & state machine](#7-business-rules--state-machine)
8. [Seismic integration hooks](#8-seismic-integration-hooks)
9. [Username history & change limits](#9-username-history--change-limits)
10. [Admin / ops API](#10-admin--ops-api)
11. [In-app user flows](#11-in-app-user-flows)
12. [End-to-end walkthrough (real-life example)](#12-end-to-end-walkthrough-real-life-example)
13. [Edge cases & error catalog](#13-edge-cases--error-catalog)
14. [Non-goals & future work (v2)](#14-non-goals--future-work-v2)
15. [Testing plan](#15-testing-plan)

---

## 1. Context & goals

### Problem

Avvio wants a referral program where existing users invite friends and earn **$5
per successful referral**. A referral is "successful" when the friend:

1. Signs up and **manually claims** the referrer's username as a promo code
2. Opens a **Seismic virtual bank account** (USD, EUR, or GBP)
3. Receives their **first inbound fiat deposit** into that account

The mobile app needs backend APIs for:

- **Referral Home** — marketing copy, share link, FAQs, claim-code entry point
- **Track referrals** — balance earned, list of referred friends, 3-step progress
- **Claim modal** — apply a friend's username as referral code

Payout is **manual for v1** (ops transfers $5 outside the app). The backend
tracks eligibility and exposes an admin endpoint to mark referrals as paid.

### Goals

| Goal | Detail |
|------|--------|
| Username-as-code | Referral link = `https://avvio.com/invite/{username}`; promo code = current `@username` |
| Manual claim v1 | User must explicitly enter code via `POST /referrals/claim` (no auto deep-link attribution yet) |
| Real-time progress | Referral status updates when Seismic webhooks fire |
| Ops-friendly payout | Admin queue + `mark-paid` endpoint |
| Founder username rules | Max 3 username changes; all past usernames permanently reserved |

### Stakeholder inputs

| Source | Decision |
|--------|----------|
| Rahul (mobile) | Manual claim only for v1; may add auto-claim from link later |
| Rahul (mobile) | Backend controls `canClaimCode` via API field |
| Team | Claim blocked if user already has bank account |
| Team | Only **current** username works for claim; old usernames → `INVALID_CODE` |
| Rahul (Slack) | $5 reward is manual transfer for now |
| Team | 3 progress steps only (drop "Invite sent" for v1) |
| Team | First deposit = any amount, Seismic USD/EUR/GBP virtual account inbound |
| Team | Bridge is **not** used — Seismic-only |

---

## 2. Product decisions (locked in)

### Attribution

```
Referral relationship is created ONLY by:
  POST /api/v1/referrals/claim  { "code": "dave" }

Opening avvio.com/invite/dave does NOT create a backend referral record in v1.
The link is for marketing / app-store routing only.
```

### Claim eligibility (`canClaimCode`)

```typescript
canClaimCode =
  user has NOT already claimed a referral code
  AND user does NOT have a Seismic bank account yet
```

**"Bank account exists"** means any of:

- `SeismicCustomer.lifecycleStatus === 'ACTIVE'`, OR
- At least one `SeismicAccount` where `kind = 'fiat'`, `purpose = 'onramp_deposit'`,
  `status = 'active'`

**Mid-provisioning edge case:** If user is in `PROVISIONING_ACCOUNTS` but not yet
`ACTIVE`, they **can still claim** (accounts not fully ready).

### Promo code resolution

- Code is normalized: strip leading `@`, lowercase, trim
- Lookup **only** `User.username` (current username)
- Historical usernames are **not** resolved for claims
- If Dave renamed `dave` → `dave_v2`, claiming `"dave"` returns `INVALID_CODE`

### Reward trigger

- **When:** First inbound Seismic fiat deposit to a virtual account (USD, EUR, GBP)
- **Amount:** Any amount qualifies
- **Rail:** Seismic `account.deposit_received` webhook, `kind = 'received'`
- **Excludes:** Bridge, crypto on-ramps, MoonPay, wallet top-ups

### Progress steps (Track referrals UI)

| Step | Backend trigger |
|------|-----------------|
| **Signed up** | Claim succeeds → `Referral.status = SIGNED_UP` |
| **Bank account opened** | `SEISMIC_ACCOUNTS_READY` event |
| **Deposited money** | First Seismic fiat deposit → `DEPOSITED` + `REWARD_ELIGIBLE` |

"Invite sent" is **dropped for v1** — referral row only exists after claim.

### Payout model

```
First deposit → payoutStatus = REWARD_ELIGIBLE
Ops sends $5 manually (bank transfer, etc.)
Ops calls POST /admin/referrals/:id/mark-paid
Referrer Track referrals balance updates
```

---

## 3. What already exists

| Capability | Where | Reused for |
|------------|-------|------------|
| Usernames | `User.username`, `UsernameHandler` | Referral code, invite URL |
| Auth | `AuthGuard`, JWT Bearer | All mobile referral routes |
| Seismic bank ready | `seismic-account.handler.ts` → `SEISMIC_ACCOUNTS_READY` | Step 2 progress |
| Seismic deposits | `seismic-webhook.handler.ts` → `SEISMIC_DEPOSIT_RECEIVED` | Step 3 + reward |
| Admin auth | `InternalApiGuard`, `INTERNAL_API_KEY` | Ops payout endpoints |
| Promo banner | `promo-referral-earn5` → `avvio://referral` | Entry point (client-only) |

**Does not exist today:** Referral models, referral APIs, username history table,
first-deposit tracking, referral progress hooks.

---

## 4. Architecture

### Approach: Direct service hooks in Seismic handlers (recommended)

New `ReferralsModule` exports `ReferralProgressHandler`. Existing Seismic handlers
call it synchronously after bank-ready and deposit events — same pattern as invoice
reconciliation in `seismic-webhook.handler.ts`.

```
┌─────────────────────┐     claim/summary/track      ┌──────────────────────┐
│   AvvioMobile       │ ────────────────────────────▶│  ReferralsModule     │
│   (JWT auth)        │                              │  - summary.handler   │
└─────────────────────┘                              │  - claim.handler     │
                                                     │  - track.handler     │
                                                     │  - progress.handler  │
                                                     └──────────┬───────────┘
                                                                │
                     ┌──────────────────────────────────────────┤
                     │                                          │
                     ▼                                          ▼
          ┌─────────────────────┐                    ┌─────────────────────┐
          │ seismic-account     │                    │ seismic-webhook     │
          │ .handler.ts         │                    │ .handler.ts         │
          │ onBankAccountReady()│                    │ onDepositReceived() │
          └─────────────────────┘                    └─────────────────────┘

┌─────────────────────┐     mark-paid / list queue   ┌──────────────────────┐
│   Ops (Postman)     │ ────────────────────────────▶│  AdminModule         │
│   x-internal-api-key│                              │  referrals routes    │
└─────────────────────┘                              └──────────────────────┘
```

### Module layout

```
src/modules/referrals/
  referrals.module.ts
  referrals.controller.ts              # GET summary, POST claim, GET track
  handlers/
    referral-summary.handler.ts        # summary + canClaimCode logic
    referral-claim.handler.ts            # create referral row
    referral-track.handler.ts            # balance + referred users list
    referral-progress.handler.ts         # advance status (called from Seismic)
  dto/
    claim-referral.dto.ts
  utils/
    referral-config.util.ts              # static marketing copy, FAQs, reward amount
  admin/
    referral-admin.controller.ts         # ops endpoints (InternalApiGuard)
    referral-admin.handler.ts
```

**Module dependencies:**

- `DatabaseModule` (Prisma)
- `ReferralsModule` imported by `SeismicModule` (for `ReferralProgressHandler`)
- `ReferralsModule` may import `UserModule` helpers for username validation

**Design-for-isolation notes:**

- `referral-progress.handler.ts` is the **only** place that mutates referral status
  from webhook events (single writer, idempotent)
- `referral-config.util.ts` is pure static data — marketing can update copy without
  DB migrations (same pattern as `promo-banners.util.ts`)
- Mobile APIs never expose admin routes

---

## 5. Data model

### Prisma schema additions

```prisma
model UsernameHistory {
  id        String   @id @default(cuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  username  String   @unique // permanently reserved — no other user may take it
  createdAt DateTime @default(now())

  @@index([userId])
}

model Referral {
  id               String               @id @default(cuid())
  referrerId       String
  referrer         User                 @relation("ReferralsMade", fields: [referrerId], references: [id])
  refereeId        String               @unique // one referrer per user, ever
  referee          User                 @relation("ReferralReceived", fields: [refereeId], references: [id])
  status           ReferralStatus       @default(SIGNED_UP)
  payoutStatus     ReferralPayoutStatus @default(PENDING)
  claimedAt        DateTime             @default(now())
  bankOpenedAt     DateTime?
  depositedAt      DateTime?
  rewardEligibleAt DateTime?
  paidAt           DateTime?
  depositEventId   String?              @unique // Seismic deposit idempotency key
  createdAt        DateTime             @default(now())
  updatedAt        DateTime             @updatedAt

  @@index([referrerId])
  @@index([payoutStatus])
  @@index([status])
}

enum ReferralStatus {
  SIGNED_UP
  ACCOUNT_OPENED
  DEPOSITED
}

enum ReferralPayoutStatus {
  PENDING
  REWARD_ELIGIBLE
  PAID
}
```

**User model additions:**

```prisma
// On User model — add relations:
usernameChangeCount Int @default(0)  // optional denormalized counter; or count UsernameHistory rows
ReferralsMade       Referral[] @relation("ReferralsMade")
ReferralReceived    Referral?  @relation("ReferralReceived")
UsernameHistory     UsernameHistory[]
```

### Field semantics

| Field | Purpose |
|-------|---------|
| `refereeId @unique` | Each user can be referred at most once |
| `depositEventId @unique` | Prevents double-reward on webhook redelivery |
| `payoutStatus` | Separate from funnel `status`; tracks ops payout |
| `UsernameHistory.username @unique` | Global permanent reservation |

### Reward amount

Stored in `referral-config.util.ts` as `REFERRAL_REWARD_USD = 5` (not per-row).
If reward amount changes in future, only new referrals after config change are
affected (v1 uses fixed $5 everywhere).

---

## 6. API specification

All mobile endpoints require `Authorization: Bearer <jwt>`.

Response envelope follows mobile spec where noted; otherwise matches existing
NestJS patterns (`HttpException` for errors).

---

### 6.1 `GET /api/v1/referrals/summary`

Fetches promotional copy, invite URL, FAQs, and whether the claim option should
be shown.

**Request**

```http
GET /api/v1/referrals/summary
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "success": true,
  "data": {
    "referralCode": "dave",
    "inviteUrl": "https://avvio.com/invite/dave",
    "shareMessage": "Join me on Avvio! Open an account and deposit to get $5. Use my link: https://avvio.com/invite/dave",
    "rewardAmountFormatted": "$5",
    "canClaimCode": true,
    "promotions": {
      "headline": "EARN $5 FOR EVERY FRIEND YOU INVITE",
      "subheadline": "For every friend who joins Avvio, you get $5. No limit.",
      "steps": [
        {
          "title": "INVITE FRIENDS",
          "description": "You invite your friends to try Avvio with your link.",
          "icon": "https://cdn.avvio.com/icons/referrals/step_invite.png"
        },
        {
          "title": "THEY OPEN A BANK ACCOUNT",
          "description": "A USD, EUR or GBP virtual bank account.",
          "icon": "https://cdn.avvio.com/icons/referrals/step_bank.png"
        },
        {
          "title": "THEY MAKE A DEPOSIT",
          "description": "They deposit any amounts into USD, EUR or GBP.",
          "icon": "https://cdn.avvio.com/icons/referrals/step_deposit.png"
        },
        {
          "title": "CHA-CHING YOU MAKE US$5",
          "description": "Once they all complete these steps you earn US$5 per friend. No limits.",
          "icon": "https://cdn.avvio.com/icons/referrals/step_reward.png"
        }
      ]
    },
    "faqs": [
      {
        "question": "How it works?",
        "answer": "Avvio is a unified money account that lets you hold, move, and grow your money in more place while staying in control."
      },
      {
        "question": "How does Avvio work?",
        "answer": "Avvio connects your wallets and bank accounts seamlessly."
      },
      {
        "question": "Is my money under my control?",
        "answer": "Yes, you maintain full custody and control over your accounts at all times."
      },
      {
        "question": "How does Avvio help my money grow?",
        "answer": "Earn competitive yield on your deposits directly within the app."
      },
      {
        "question": "Can I access my money anytime?",
        "answer": "Yes, instant withdrawals and transfers are available 24/7."
      },
      {
        "question": "What happens if Avvio shuts down?",
        "answer": "Your funds are safeguarded by regulated banking partners and can be retrieved independently."
      }
    ]
  }
}
```

**Field derivation**

| Field | Source |
|-------|--------|
| `referralCode` | Authenticated user's current `User.username` |
| `inviteUrl` | `https://avvio.com/invite/{username}` |
| `shareMessage` | Template from `referral-config.util.ts` |
| `canClaimCode` | See [§2 Claim eligibility](#claim-eligibility-canclaimcode) |
| `promotions`, `faqs` | Static config file (server-side, no app release needed to update) |

**`canClaimCode` examples**

| User state | `canClaimCode` |
|------------|----------------|
| New user, no claim, no bank | `true` |
| Already claimed a code | `false` |
| Never claimed, Seismic ACTIVE | `false` |
| Never claimed, KYC in progress | `true` |
| Never claimed, PROVISIONING_ACCOUNTS | `true` |

---

### 6.2 `POST /api/v1/referrals/claim`

Apply a friend's username as referral code.

**Request**

```http
POST /api/v1/referrals/claim
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "code": "dave"
}
```

Also accepts `"@dave"` — normalized to `dave`.

**Response `200 OK`**

```json
{
  "success": true,
  "data": {
    "referralId": "clx889900abcdef",
    "referrerName": "Dave Miller",
    "referrerHandle": "@dave",
    "status": "SIGNED_UP",
    "message": "Referral code applied successfully! Complete your bank account setup and deposit to earn rewards."
  }
}
```

**Error responses**

| HTTP | `error` | When |
|------|---------|------|
| 400 | `SELF_REFERRAL` | User entered their own username |
| 400 | `ALREADY_CLAIMED` | User already has a referral row as referee |
| 400 | `BANK_ACCOUNT_EXISTS` | User already has Seismic bank account |
| 400 | `INVALID_FORMAT` | Username format validation failed |
| 404 | `INVALID_CODE` | Username not found (current username only) |

**Error body shape (matches mobile spec)**

```json
{
  "success": false,
  "error": "INVALID_CODE",
  "message": "This referral code does not exist."
}
```

```json
{
  "success": false,
  "error": "SELF_REFERRAL",
  "message": "You cannot apply your own referral code."
}
```

```json
{
  "success": false,
  "error": "ALREADY_CLAIMED",
  "message": "You have already applied a referral code."
}
```

```json
{
  "success": false,
  "error": "BANK_ACCOUNT_EXISTS",
  "message": "You cannot apply a referral code after opening a bank account."
}
```

---

### 6.3 `GET /api/v1/referrals/track`

Returns referrer's earned balance and list of referred users with 3-step progress.
Powers the **Track referrals** screen (balance header + per-friend progress cards).

**Request**

```http
GET /api/v1/referrals/track
Authorization: Bearer <jwt>
```

**Response `200 OK`**

```json
{
  "success": true,
  "data": {
    "rewardAmountFormatted": "$5",
    "balance": {
      "earnedUsd": "10.00",
      "paidUsd": "5.00",
      "pendingUsd": "5.00"
    },
    "referrals": [
      {
        "referralId": "clx111aaa",
        "refereeHandle": "@alex",
        "refereeName": "Alex Chen",
        "status": "DEPOSITED",
        "payoutStatus": "REWARD_ELIGIBLE",
        "steps": {
          "signedUp": true,
          "bankAccountOpened": true,
          "deposited": true
        },
        "claimedAt": "2026-06-15T10:30:00.000Z",
        "bankOpenedAt": "2026-06-18T14:00:00.000Z",
        "depositedAt": "2026-06-20T09:15:00.000Z"
      },
      {
        "referralId": "clx222bbb",
        "refereeHandle": "@sam",
        "refereeName": "Sam Rivera",
        "status": "ACCOUNT_OPENED",
        "payoutStatus": "PENDING",
        "steps": {
          "signedUp": true,
          "bankAccountOpened": true,
          "deposited": false
        },
        "claimedAt": "2026-06-25T08:00:00.000Z",
        "bankOpenedAt": "2026-06-28T11:00:00.000Z",
        "depositedAt": null
      }
    ]
  }
}
```

**Balance calculation**

```
earnedUsd   = count(referrals where payoutStatus IN [REWARD_ELIGIBLE, PAID]) × $5
paidUsd     = count(referrals where payoutStatus = PAID) × $5
pendingUsd  = earnedUsd - paidUsd
```

**`refereeName` source:** `User.name` or `KycProfile` first name if available;
fallback to `@username`.

**Sort order:** Most recently `claimedAt` first.

**Empty state:** `referrals: []`, all balances `"0.00"`.

---

## 7. Business rules & state machine

### Referral lifecycle (referee funnel)

```
                    POST /claim
                        │
                        ▼
                  ┌───────────┐
                  │ SIGNED_UP │
                  └─────┬─────┘
                        │ SEISMIC_ACCOUNTS_READY
                        ▼
               ┌─────────────────┐
               │ ACCOUNT_OPENED  │
               └────────┬────────┘
                        │ first Seismic fiat deposit (received)
                        ▼
                  ┌───────────┐
                  │ DEPOSITED │
                  └───────────┘
```

### Payout lifecycle (referrer reward)

```
PENDING ──(first deposit)──▶ REWARD_ELIGIBLE ──(admin mark-paid)──▶ PAID
```

### Idempotency rules

| Event | Guard |
|-------|-------|
| Claim | `refereeId @unique` — second claim rejected |
| Bank opened | Only advance if `status = SIGNED_UP` |
| Deposit | `depositEventId @unique`; only first deposit triggers reward |
| Mark paid | If already `PAID`, return success (idempotent) |

### Out-of-order webhook handling

If deposit arrives before bank-opened event is processed:

```
onDepositReceived():
  if status = SIGNED_UP → set status = DEPOSITED (skip ACCOUNT_OPENED or set both timestamps)
  if status = ACCOUNT_OPENED → set status = DEPOSITED
  if status = DEPOSITED → no-op (already done)
```

This ensures reward is never missed due to webhook ordering.

---

## 8. Seismic integration hooks

### Hook 1 — Bank account opened

**File:** `src/modules/seismic/handlers/seismic-account.handler.ts`

**Trigger:** After `SEISMIC_ACCOUNTS_READY` activity is recorded (customer transitions
to `ACTIVE`).

**Call:**

```typescript
await this.referralProgress.onBankAccountOpened(userId);
```

**Handler logic:**

```typescript
async onBankAccountOpened(userId: string): Promise<void> {
  const referral = await prisma.referral.findUnique({ where: { refereeId: userId } });
  if (!referral || referral.status !== 'SIGNED_UP') return;

  await prisma.referral.update({
    where: { id: referral.id },
    data: { status: 'ACCOUNT_OPENED', bankOpenedAt: new Date() },
  });
}
```

### Hook 2 — First fiat deposit

**File:** `src/modules/seismic/handlers/seismic-webhook.handler.ts`

**Trigger:** Inside `recordDepositEvent()`, when `kind === 'received'` and account
currency is USD, EUR, or GBP.

**Call:**

```typescript
await this.referralProgress.onDepositReceived(userId, {
  depositId,
  currency: account.currency,
});
```

**Handler logic:**

```typescript
async onDepositReceived(userId: string, event: { depositId: string; currency: string }): Promise<void> {
  if (!['USD', 'EUR', 'GBP'].includes(event.currency.toUpperCase())) return;

  const referral = await prisma.referral.findUnique({ where: { refereeId: userId } });
  if (!referral || referral.status === 'DEPOSITED') return; // already rewarded

  // Idempotent upsert via depositEventId
  await prisma.referral.update({
    where: { id: referral.id },
    data: {
      status: 'DEPOSITED',
      payoutStatus: 'REWARD_ELIGIBLE',
      depositedAt: new Date(),
      rewardEligibleAt: new Date(),
      depositEventId: `seismic-deposit-received-${event.depositId}`,
      // backfill bankOpenedAt if missing
      ...(referral.status === 'SIGNED_UP' ? { bankOpenedAt: new Date() } : {}),
    },
  });
}
```

### Bank account check (for `canClaimCode` and claim validation)

Shared helper in `referral-summary.handler.ts`:

```typescript
async hasSeismicBankAccount(userId: string): Promise<boolean> {
  const customer = await prisma.seismicCustomer.findUnique({
    where: { userId },
    include: {
      accounts: {
        where: { kind: 'fiat', purpose: 'onramp_deposit', status: 'active' },
        take: 1,
      },
    },
  });
  if (!customer) return false;
  if (customer.lifecycleStatus === 'ACTIVE') return true;
  return customer.accounts.length > 0;
}
```

---

## 9. Username history & change limits

Founder requirements implemented alongside referrals (extends existing
`UsernameHandler`).

### Rules

1. **Max 3 username changes** per user (initial auto-generated username does not
   count as a "change"; only explicit `POST /user/username` updates count)
2. **Permanent reservation:** Every username ever held is inserted into
   `UsernameHistory` and cannot be taken by any other user
3. **Referral lookup:** Only current `User.username` — history is NOT used for
   claim resolution

### Change flow

```
User requests new username "dave_v2"
  │
  ├─ Validate format
  ├─ Check "dave_v2" not in User.username OR UsernameHistory.username
  ├─ Check user.usernameChangeCount < 3
  │
  ├─ Insert current username into UsernameHistory (if not already there)
  ├─ Update User.username = "dave_v2"
  ├─ Increment usernameChangeCount
  │
  └─ Return success

Invite URL automatically becomes avvio.com/invite/dave_v2 on next GET /summary
```

### New error for username change

```json
{
  "success": false,
  "reason": "USERNAME_CHANGE_LIMIT_REACHED",
  "message": "You have reached the maximum number of username changes (3)."
}
```

### Initial username seeding

On first explicit username set (including auto-generated at signup), insert into
`UsernameHistory` so the name is reserved from day one.

---

## 10. Admin / ops API

All routes under `/api/v1/admin/referrals/*`, guarded by `InternalApiGuard`
(header: `x-internal-api-key: <INTERNAL_API_KEY>`).

See `docs/INTERNAL_ADMIN_API.md` for auth setup.

---

### 10.1 `GET /api/v1/admin/referrals`

List referrals for ops payout queue.

**Query params**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `payoutStatus` | string | — | Filter: `REWARD_ELIGIBLE`, `PAID`, `PENDING` |
| `limit` | number | 50 | Max 200 |
| `offset` | number | 0 | Pagination |

**Response `200 OK`**

```json
{
  "success": true,
  "data": {
    "total": 12,
    "referrals": [
      {
        "referralId": "clx111aaa",
        "referrerId": "user-dave-id",
        "referrerHandle": "@dave",
        "referrerEmail": "dave@example.com",
        "refereeId": "user-alex-id",
        "refereeHandle": "@alex",
        "refereeEmail": "alex@example.com",
        "status": "DEPOSITED",
        "payoutStatus": "REWARD_ELIGIBLE",
        "rewardUsd": "5.00",
        "depositedAt": "2026-06-20T09:15:00.000Z",
        "rewardEligibleAt": "2026-06-20T09:15:00.000Z",
        "paidAt": null
      }
    ]
  }
}
```

---

### 10.2 `POST /api/v1/admin/referrals/:id/mark-paid`

Mark a referral reward as paid after ops sends $5 manually.

**Request**

```http
POST /api/v1/admin/referrals/clx111aaa/mark-paid
x-internal-api-key: <key>
Content-Type: application/json

{
  "note": "Paid via Wise transfer ref WISE-12345"
}
```

`note` is optional, stored in referral metadata or audit log.

**Response `200 OK`**

```json
{
  "success": true,
  "data": {
    "referralId": "clx111aaa",
    "payoutStatus": "PAID",
    "paidAt": "2026-06-21T16:00:00.000Z"
  }
}
```

**Errors**

| HTTP | When |
|------|------|
| 404 | Referral not found |
| 400 | `payoutStatus = PENDING` (deposit not completed yet) |
| 200 | Already `PAID` — idempotent success |

After mark-paid, Dave's `GET /referrals/track` shows increased `paidUsd`.

---

## 11. In-app user flows

### Screen map (mobile)

| Screen | API(s) | Notes |
|--------|--------|-------|
| Referral Home | `GET /referrals/summary` | Hero, steps, FAQ, share link |
| Claim modal | `POST /referrals/claim` | Shown when `canClaimCode: true` |
| Track referrals | `GET /referrals/track` | Balance + progress cards |
| Dashboard promo | — | `avvio://referral` deeplink → Referral Home |

### Flow A — Referrer (Dave) invites a friend

```
1. Dave opens Referral Home
   GET /referrals/summary
   → sees inviteUrl: avvio.com/invite/dave
   → canClaimCode: false (Dave already has bank account)

2. Dave taps "Copy" or "Share Avvio"
   → OS share sheet with shareMessage + inviteUrl
   (no backend call)

3. Dave opens Track referrals
   GET /referrals/track
   → balance $0, empty referrals list (or existing referrals)
```

### Flow B — Referee (Alex) claims code and completes funnel

```
1. Alex signs up (sync-session)
   → auto username: orbit_7k3

2. Alex opens Referral Home
   GET /referrals/summary
   → canClaimCode: true (no bank account, no prior claim)
   → sees "I have a referral code" link

3. Alex taps link → Claim modal
   → enters "dave"
   POST /referrals/claim { "code": "dave" }
   → success, status SIGNED_UP
   → canClaimCode becomes false (has referral row)

4. Alex completes KYC → Seismic provisions accounts
   → SEISMIC_ACCOUNTS_READY webhook
   → Referral status → ACCOUNT_OPENED

5. Alex receives first bank transfer to USD VA
   → Seismic deposit_received webhook
   → Referral status → DEPOSITED, payoutStatus → REWARD_ELIGIBLE

6. Dave opens Track referrals
   GET /referrals/track
   → @alex card: all 3 steps ✓
   → balance.pendingUsd: "5.00"

7. Ops pays Dave $5, calls mark-paid
   → Dave balance.paidUsd: "5.00"
```

### Flow C — User tries to claim too late

```
1. Sam signs up, completes KYC, bank account ACTIVE
2. Sam opens Referral Home
   GET /referrals/summary → canClaimCode: false
3. "I have a referral code" link is hidden
4. If Sam somehow POST /claim → 400 BANK_ACCOUNT_EXISTS
```

### Flow D — Dave renames username

```
1. Dave's username: dave → changes to dave_v2
2. Dave's inviteUrl becomes avvio.com/invite/dave_v2
3. Alex (who saw old link) tries POST /claim { "code": "dave" }
   → 404 INVALID_CODE
4. Alex must use "dave_v2"
```

---

## 12. End-to-end walkthrough (real-life example)

### Characters

| Person | Username | Role |
|--------|----------|------|
| Dave Miller | `@dave` | Referrer |
| Alex Chen | `@alex` (after rename from `orbit_7k3`) | Referee |
| Sam Rivera | `@sam` | Referee (incomplete funnel) |

---

### Timeline

#### Day 1 — Dave shares, Alex signs up

| Time | Event | System state |
|------|-------|--------------|
| 09:00 | Dave shares `avvio.com/invite/dave` via WhatsApp | No backend record |
| 10:00 | Alex downloads app, completes signup | Alex: `orbit_7k3`, no referral |
| 10:05 | Alex opens Referral Home | `canClaimCode: true` |
| 10:06 | Alex claims `"dave"` | Referral created: `SIGNED_UP`, `PENDING` |
| 10:06 | Dave's Track referrals | `@orbit_7k3` shown, step 1 ✓ only |

#### Day 3 — Alex opens bank account

| Time | Event | System state |
|------|-------|--------------|
| 14:00 | Alex completes KYC, Seismic accounts go ACTIVE | `SEISMIC_ACCOUNTS_READY` |
| 14:00 | Referral progress hook fires | Status → `ACCOUNT_OPENED` |
| 14:01 | Dave's Track referrals | Steps 1–2 ✓ for Alex |

#### Day 5 — Alex deposits, Dave earns $5

| Time | Event | System state |
|------|-------|--------------|
| 09:15 | Alex receives €50 SEPA to EUR virtual account | `SEISMIC_DEPOSIT_RECEIVED` |
| 09:15 | Referral progress hook fires | Status → `DEPOSITED`, `REWARD_ELIGIBLE` |
| 09:16 | Dave's Track referrals | All 3 steps ✓, `pendingUsd: "5.00"` |
| 09:16 | Ops queue | Alex referral appears in `REWARD_ELIGIBLE` list |

#### Day 6 — Ops pays Dave

| Time | Event | System state |
|------|-------|--------------|
| 16:00 | Ops sends Dave $5 via bank transfer | External |
| 16:05 | Ops: `POST /admin/referrals/clx111/mark-paid` | `PAID`, `paidAt` set |
| 16:06 | Dave's Track referrals | `paidUsd: "5.00"`, `pendingUsd: "0.00"` |

#### Parallel — Sam stuck at step 2

| Time | Event | System state |
|------|-------|--------------|
| Day 2 | Sam claims Dave's code | `SIGNED_UP` |
| Day 4 | Sam opens bank account | `ACCOUNT_OPENED` |
| Day 10 | Sam still hasn't deposited | Dave sees 2/3 steps for @sam |
| — | No reward until Sam deposits | `payoutStatus: PENDING` |

---

### Case matrix

| # | Scenario | Expected result |
|---|----------|-----------------|
| 1 | Valid claim before bank | `200`, `SIGNED_UP` |
| 2 | Claim own username | `400 SELF_REFERRAL` |
| 3 | Second claim attempt | `400 ALREADY_CLAIMED` |
| 4 | Claim after bank ACTIVE | `400 BANK_ACCOUNT_EXISTS`, `canClaimCode: false` |
| 5 | Invalid / old username | `404 INVALID_CODE` |
| 6 | Dave refers 10 friends who all deposit | `earnedUsd: "50.00"` |
| 7 | Webhook redelivery same deposit | No duplicate reward (`depositEventId`) |
| 8 | Deposit before bank webhook | Still reaches `DEPOSITED` + `REWARD_ELIGIBLE` |
| 9 | Crypto-only top-up (no fiat VA deposit) | No referral progress on deposit |
| 10 | Mark paid twice | Idempotent `200` |
| 11 | Mark paid before deposit | `400` — not eligible yet |
| 12 | Dave renames username | Old code invalid; new code works |

---

## 13. Edge cases & error catalog

### Mobile-facing errors

| Code | HTTP | Message |
|------|------|---------|
| `SELF_REFERRAL` | 400 | You cannot apply your own referral code. |
| `ALREADY_CLAIMED` | 400 | You have already applied a referral code. |
| `BANK_ACCOUNT_EXISTS` | 400 | You cannot apply a referral code after opening a bank account. |
| `INVALID_CODE` | 404 | This referral code does not exist. |
| `INVALID_FORMAT` | 400 | Referral code must be a valid username (3–30 chars). |

### Username change errors

| Code | HTTP | Message |
|------|------|---------|
| `USERNAME_TAKEN` | 200* | Existing soft failure with suggestions |
| `USERNAME_CHANGE_LIMIT_REACHED` | 200* | Max 3 changes reached |
| `INVALID_FORMAT` | 200* | Existing validation message |

\*Existing username endpoint returns soft failures in body with HTTP 200.

### Data integrity

| Risk | Mitigation |
|------|------------|
| Double payout | `depositEventId @unique`; mark-paid idempotent |
| Self-referral alt account | Business/process; no technical fingerprinting in v1 |
| Deleted referrer/referee | Soft-deleted users (`isDeleted`) — referrals remain for audit; claim blocked for deleted accounts |
| Blocked users | Can still have referral rows; claim blocked if referee blocked |

---

## 14. Non-goals & future work (v2)

| Item | Notes |
|------|-------|
| Auto deep-link attribution | Rahul may add later; store pending code at signup |
| "Zero fees for 30 days" | Shown in Figma claim success — no fee engine in backend v1 |
| "Invite sent" step | Requires pre-signup invite tracking / contacts API |
| Contacts list on Track screen | Client-side device contacts; no backend v1 |
| Automated $5 wallet credit | Manual ops payout v1; schema supports future automation |
| Bridge deposit counting | Seismic-only per team decision |
| Referral analytics / PostHog | Client-side tracking per mobile spec |
| Username history for claim lookup | Explicitly rejected — current username only |

---

## 15. Testing plan

### Unit tests

- `referral-claim.handler` — all error paths (self, already claimed, bank exists, invalid)
- `referral-summary.handler` — `canClaimCode` logic matrix
- `referral-progress.handler` — state transitions, idempotency, out-of-order events
- `referral-track.handler` — balance math
- `username.handler` — history insert, 3-change limit, permanent reservation

### Integration tests

- Full funnel: claim → mock bank ready → mock deposit → `REWARD_ELIGIBLE`
- Admin mark-paid → track balance update
- Webhook redelivery does not double-reward

### E2E / manual QA checklist

- [ ] New user sees `canClaimCode: true` before KYC
- [ ] User with ACTIVE Seismic customer sees `canClaimCode: false`
- [ ] Claim hides after success (`canClaimCode: false`)
- [ ] Track referrals shows 3 steps correctly per state
- [ ] Dave sees $5 pending after Alex deposits
- [ ] Mark-paid moves balance from pending to paid
- [ ] Old username claim fails after rename
- [ ] Fourth username change blocked

---

## Appendix A — Config file shape

`src/modules/referrals/utils/referral-config.util.ts`

```typescript
export const REFERRAL_REWARD_USD = 5;
export const REFERRAL_INVITE_BASE_URL = 'https://avvio.com/invite';
export const REFERRAL_MAX_USERNAME_CHANGES = 3;

export const REFERRAL_PROMOTIONS = { headline, subheadline, steps };
export const REFERRAL_FAQS = [ ... ];
export function buildShareMessage(username: string): string { ... }
```

Marketing copy updates require a backend deploy (same as dashboard promo banners).

---

## Appendix B — Module registration

```typescript
// app.module.ts
imports: [ ..., ReferralsModule ]

// seismic.module.ts
imports: [ ..., ReferralsModule ]
providers: [ ..., ReferralProgressHandler injected into account + webhook handlers ]
```

---

## Approval checklist

Before implementation, please confirm:

- [ ] Manual claim only (no auto deep-link v1)
- [ ] `canClaimCode` = no prior claim + no Seismic bank account
- [ ] Current username only for claims (old usernames invalid)
- [ ] 3 username changes max + permanent reservation
- [ ] Seismic fiat deposit any amount triggers $5 eligibility
- [ ] Manual ops payout + admin mark-paid
- [ ] 3 progress steps (no "Invite sent")
- [ ] `GET /referrals/track` API approved for mobile
- [ ] Marketing copy served from static config file

**Once approved, next step:** implementation plan and phased rollout.
