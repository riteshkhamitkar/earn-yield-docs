# Bridge Off-Ramping: Explained Simply! 🎢

## What is Off-Ramping? (Explained like you're 5! 🧸)
Imagine you went to a huge arcade and played a bunch of fun games. You won a lot of **Arcade Tickets** (this is your Crypto). But when you leave the arcade, you can't buy ice cream with arcade tickets! 

To get ice cream, you need to go to the **Prize Counter** (the Bridge) and trade your Arcade Tickets for **Real Money** (USD) to put into your **Piggy Bank** (your Bank Account). 

**Off-ramping** is just the process of taking your Crypto, giving it to the Bridge, and having the Bridge put Real Money straight into your Bank Account!

***

## The APIs (The Magic Buttons) 🔘
Here are all the API endpoints we built to make this magic happen. All of these live under the `/api/v1/bridge` path and require the user to be logged in:

1. **Add a Bank Account** 🏦
   - **`POST /external-accounts`**
   - *What it does:* Tells the Bridge where to send the real money. You give it your routing number, account number, or IBAN.
   
2. **View My Bank Accounts** 👀
   - **`GET /external-accounts`**
   - *What it does:* Shows a list of all the piggy banks (bank accounts) you've already added.

3. **Remove a Bank Account** ❌
   - **`DELETE /external-accounts/:externalAccountId`**
   - *What it does:* Deletes a bank account if you don't want to use it anymore.

4. **Start the Off-Ramp (Trade Tickets for Money!)** 🚀
   - **`POST /off-ramp/initiate`**
   - *What it does:* You say "I want to off-ramp 100 USDC to my Bank." The Bridge replies with a "Deposit Address" (a special crypto mailbox).
   
5. **Check Status** 🕵️‍♂️
   - **`GET /off-ramp/:transferId/status`**
   - *What it does:* Asks "Is my money in my bank yet?" and returns the current status (like `payment_processed`).

***

## The Whole Workflow (Step-by-Step) 👣

1. **KYC First:** Before playing, the user MUST be verified (Know Your Customer). They must have an `ACTIVE` status in the system.
2. **Link a Bank:** The user calls `POST /external-accounts` with their checking/savings details (US/ACH or SEPA). The system saves this in the database `BridgeExternalAccount`.
3. **Initiate Trade:** The user decides to cash out crypto. The app calls `POST /off-ramp/initiate` providing the `amount`, the `sourceChain` (e.g. Solana or Ethereum), and the `externalAccountId` they added in step 2.
4. **Get Deposit Instructions:** The backend talks to the Bridge Provider. Bridge creates a specific crypto address just for this transfer! The backend saves this in the `BridgeTransaction` table and returns the `depositAddress` to the frontend.
5. **Send Crypto:** The user (or the app's wallet, like Turnkey) sends the exact crypto amount to that `depositAddress`.
6. **Bridge Does the Magic:** Bridge detects the incoming crypto, safely converts it to USD, and sends an ACH or SEPA wire to the user's bank account.
7. **Polling Status:** The app periodically calls `GET /off-ramp/:transferId/status` until it sees the status change to `payment_processed`. Done! 🎉

***

## The Implementation Details (For the Grown-Ups 👩‍💻)

Here is exactly how off-ramping is implemented under the hood in `anzo-backend`.

### 1. The Controller ([bridge.controller.ts](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/bridge.controller.ts))
The [BridgeController](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/bridge.controller.ts#36-178) exposes the REST endpoints to the frontend. It uses NestJS decorators (`@Post`, `@Get`, `@Delete`) and applies the `AuthGuard`. It simply passes the user's ID and the DTOs (Data Transfer Objects) down to the `BridgeService`, which in turn delegates off-ramp specifics to the [BridgeOfframpHandler](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/handlers/bridge-offramp.handler.ts#21-364).

### 2. The Handler ([bridge-offramp.handler.ts](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/handlers/bridge-offramp.handler.ts))
This is the brains of the operation. It has three main responsibilities: account management, transfer initiation, and status checking.

#### **A. Account Management ([createExternalAccount](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/bridge.controller.ts#97-103))**
- Validates the user exists in the `BridgeCustomer` table and has an `ACTIVE` KYC status.
- Maps your internal DTOs (`usAccount`, `sepaAccount`) into the payload Bridge.xyz expects.
- Calls `bridgeProvider.createExternalAccount()` with user info (Bank name, routing/account number, owner name, address).
- Saves the resulting Bridge ID (`external_account_id`) to your local database via Prisma into the `BridgeExternalAccount` model so it can be queried later.

#### **B. Transfer Initiation ([initiateOffRamp](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/handlers/bridge-offramp.handler.ts#204-315))**
- Verifies the user's KYC status and ensures the requested `BridgeExternalAccount` is active and exists locally.
- Determines the `fromAddress` based on the user's saved wallets in your DB (looking at `user.Wallet[0].solanaAddress` or `ethereumAddress` depending on the `sourceChain`).
- Calls `bridgeProvider.createTransfer()` specifying:
  - `amount`
  - `source` (e.g., source payment rail like `solana`, currency like `usdc`, and `from_address`)
  - `destination` (payment rail like `ach`, currency `usd`, and the user's Bank `external_account_id`)
  - `developer_fee` (pulled from env variables `BRIDGE_DEVELOPER_FEE_PERCENT`, defaulting to 0.0001).
- **CRITICAL PIECE:** Bridge returns `source_deposit_instructions`. This contains the `to_address` (where you must tell the user to send their crypto) and the exact `amount`.
- Records everything in the `BridgeTransaction` table (type: `'OFF_RAMP'`, initial status, fees, amounts, deposit address).

#### **C. Status Checking ([getOffRampStatus](file:///c:/Users/Ritesh/Desktop/anzo/anzo-backend/src/modules/bridge/handlers/bridge-offramp.handler.ts#316-363))**
- Looks up the local `BridgeTransaction` using the `transferId`.
- Asks `bridgeProvider.getTransfer(transferId)` for the live state.
- If the live state differs from the local DB state, it updates the `BridgeTransaction` record.
- If the state is `payment_processed`, it stamps `completedAt`.

### 3. Database Layer (`PrismaService` & Schema)
The feature is **fully implemented** in the current service's database schema. It heavily relies on three tables to track state locally, meaning we don't need to constantly hit the Bridge.xyz APIs to fetch basic user data.

Here is the exact schema implementation for the Off-Ramping flow:

#### Table 1: `BridgeCustomer`
Tells us if the user exists and if they have passed KYC. You cannot off-ramp without this.
```prisma
model BridgeCustomer {
  id         String   @id @default(cuid())
  userId     String   @unique
  customerId String   @unique // From Bridge API: e.g., 'cust_alice'
  status     String   @default("PENDING_KYC") // Must be ACTIVE to off-ramp
  // ... other relations and metadata
}
```

#### Table 2: `BridgeExternalAccount`
Stores safe partial data of the user's real-world bank account where the USD is sent.
```prisma
model BridgeExternalAccount {
  id                String   @id @default(cuid())
  bridgeCustomerId  String
  externalAccountId String   @unique // From Bridge API: e.g., 'ea_...'
  accountType       String   // e.g., "us", "ach", "sepa"
  currency          String   // e.g., "usd", "eur"
  bankName          String
  accountOwnerName  String
  accountOwnerType  String   // "individual" or "business"
  last4             String   // Last 4 digits of account number
  routingNumber     String?  // For US accounts
  checkingOrSavings String?  // "checking" or "savings"
  iban              String?  // For SEPA accounts
  isActive          Boolean  @default(true) // Can be soft-deleted
  // ... relations mapped to BridgeCustomer
}
```

#### Table 3: `BridgeTransaction`
Keeps an auditable ledger of all off-ramp attempts. It stores exactly how much crypto was sent, what fees were taken, and the deposit instructions given to the user.
```prisma
model BridgeTransaction {
  id               String    @id @default(cuid())
  userId           String
  bridgeCustomerId String
  transferId       String    @unique // From Bridge API: e.g., 'transfer_abc123'
  type             String    // 'OFF_RAMP'
  status           String    // AWAITING_FUNDS, PAYMENT_PROCESSED, etc.
  amount           String
  developerFee     String?
  sourceAsset      String    // e.g., 'USDC'
  sourcePaymentRail String   // e.g., 'solana', 'ethereum'
  destinationAsset String    // e.g., 'USD'
  destinationPaymentRail String // e.g., 'ach', 'sepa'
  // ⬇️ Critical for Off-Ramp: Where the user sends their crypto
  depositAddress   String?   
  depositAmount    String?   
  // ⬇️ Receipt/fee breakdown
  initialAmount    String?
  exchangeFee      String?
  gasFee           String?
  finalAmount      String?
  createdAt        DateTime  @default(now())
  completedAt      DateTime?
}
```
