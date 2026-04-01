# KYC Redesign: Deep Analysis & Implementation Plan

> **Author:** AI Analysis based on Founder's requirements  
> **Date:** March 30, 2026  
> **Status:** Analysis Complete — Ready for Implementation  

---

## Table of Contents

1. [What the Founder is Saying (Plain English)](#1-what-the-founder-is-saying-plain-english)
2. [Current System (OLD Flow)](#2-current-system-old-flow)
3. [New System (What We're Building)](#3-new-system-what-were-building)
4. [Provider Deep Dive — Sumsub](#4-provider-deep-dive--sumsub)
5. [Provider Deep Dive — Bridge.xyz](#5-provider-deep-dive--bridgexyz)
6. [How the Two Providers Work Together](#6-how-the-two-providers-work-together)
7. [Country-Specific Requirements](#7-country-specific-requirements)
8. [Real-Life Example (Rohan from India)](#8-real-life-example-rohan-from-india)
9. [Real-Life Example (Priya from USA)](#9-real-life-example-priya-from-usa)
10. [Real-Life Example (Arjun from Germany/EU)](#10-real-life-example-arjun-from-germanyeu)
11. [Database Schema Changes](#11-database-schema-changes)
12. [Environment Variables](#12-environment-variables)
13. [Full API Endpoint Design](#13-full-api-endpoint-design)
14. [Complete Implementation Workflow](#14-complete-implementation-workflow)
15. [What Changes vs What Stays](#15-what-changes-vs-what-stays)
16. [Reference Links](#16-reference-links)

---

## 1. What the Founder is Saying (Plain English)

The founder wants to **redesign the KYC (Know Your Customer) flow** so that when a user taps "EUR Bank Account" or "USA Bank Account" in the Anzo app, they go through a structured verification process. Here's the breakdown of what he said:

### Statement-by-Statement Analysis

| # | What Founder Said | What It Means |
|---|---|---|
| 1 | "We use shared Sumsub with email for ID verification" | **Replace Persona with Sumsub** for document verification (passport/ID scan + selfie). "Shared" means Sumsub's **shared token** — we generate an access token on our backend using the user's email as the identifier, and the mobile app uses Sumsub's native SDK to do ID verification. The user's email links their Sumsub applicant to our system. |
| 2 | "Have a custom page to collect name, DOB, address, phone number, source of funds, occupation etc" | **Build our own questionnaire form** in the app (not inside Sumsub or Persona). We collect all PII (Personally Identifiable Information) ourselves. This gives us control over the UI and the data. Bridge requires these fields for international customers, especially "high-risk populations." |
| 3 | "Use Bridge/partner's TOS" | **Bridge has its own Terms of Service** that every user must accept before they can use Bridge's platform. We show Bridge's TOS in a WebView, and when the user accepts, Bridge gives us a `signed_agreement_id`. This ID is required when creating a Bridge customer. |
| 4 | "Ideally Sumsub will take care of it, if not we can have a separate flow" | For **Proof of Address (PoA)** — Sumsub can verify PoA documents if needed. If Sumsub's level includes PoA checking, great. If not, we can build a separate upload flow. PoA is only needed in edge cases (conflicting location signals, high-risk regions). |
| 5 | "Any missing info we need, we can create an entry point and link directly to the partner flow" | If Bridge comes back saying "we need more info" (endorsement status = `incomplete`), we should be able to **deep-link the user** to a page where they can provide that missing info. This is the `additionalInfoRequired` pattern we already have in the current code. |

### The Links He Shared — What They Mean

| Link | Purpose |
|---|---|
| [Bridge Customers API](https://apidocs.bridge.xyz/platform/customers/customers/api#government-ids-by-country) | Shows how to create customers via API (instead of Bridge's hosted KYC flow). Lists which government IDs are supported per country. **This is the core of the new flow** — we submit customer data directly via API. |
| [Bridge Individuals Compliance](https://apidocs.bridge.xyz/platform/customers/compliance/individuals) | Lists what data Bridge requires for individual customers: name, address, DOB, email, national ID number, and for high-risk populations: occupation, employment status, source of funds, etc. **This defines what our questionnaire form must collect.** |
| [Bridge Individual ID Numbers by Country](https://apidocs.bridge.xyz/platform/customers/compliance/individuals#individual-identification-numbers-by-country) | Country-specific tax/national ID types. For India it's `pan` (PAN card), for USA it's `ssn`, for Germany it's `steuer_id`, etc. **Our questionnaire needs to ask for the right ID type based on the user's country.** |
| [Bridge Proof of Address](https://apidocs.bridge.xyz/platform/customers/compliance/poa) | When PoA is required: conflicting location signals (e.g., Indian passport but claims to live in USA). Accepted documents: bank statement, utility bill, government letter (within 90 days). |

### "Shared Sumsub Token" — What It Actually Means

The founder mentioned "shared Sumsub with email." This refers to Sumsub's **access token** mechanism:

1. **Backend** calls Sumsub API: `POST /resources/accessTokens/sdk` with the user's `userId` (their Anzo user ID) and `levelName` (the verification level configured in Sumsub dashboard)
2. Sumsub returns an `access token` (e.g., `_act-b8ebfb63-...`)
3. **Mobile app** initializes Sumsub SDK with this token
4. The user completes ID verification (scan ID + selfie) inside Sumsub's native UI
5. **Sumsub webhook** notifies our backend when verification is done

The "email" part means we pass the user's email as `applicantIdentifiers.email` when generating the token, so Sumsub can associate the applicant with that email.

> **Citation:** [Sumsub Generate Access Token API](https://docs.sumsub.com/reference/generate-access-token)

---

## 2. Current System (OLD Flow)

### Architecture: Persona + Bridge Hosted KYC

```
┌──────────────────────────────────────────────────────────────┐
│                      Mobile App                               │
│                                                               │
│  User taps "Get Bank Account"                                 │
│       │                                                       │
│       ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Option A: Bridge Hosted WebView                        │  │
│  │  - Bridge's own KYC UI collects ALL data                │  │
│  │  - Full name, DOB, address, ID upload, TOS              │  │
│  │  - POST /api/v1/bridge/kyc/initiate                     │  │
│  │  - Returns { kycLink, tosLink }                         │  │
│  │  - User completes in WebView                            │  │
│  └─────────────────────────────────────────────────────────┘  │
│       │                                                       │
│       ▼                                                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Option B: Persona Native SDK (newer path)              │  │
│  │  - Persona collects ALL data (name, DOB, address,       │  │
│  │    phone, tax ID, government ID scan + selfie)          │  │
│  │  - POST /api/v1/bridge/kyc/persona/session              │  │
│  │  - Returns { sessionToken, inquiryId }                  │  │
│  │  - Opens Persona SDK in-app                             │  │
│  │  - Separate TOS WebView for Bridge TOS                  │  │
│  └─────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                   Backend (NestJS)                             │
│                                                               │
│  src/modules/bridge/                                          │
│  ├── handlers/bridge-kyc.handler.ts                           │
│  │   (Bridge hosted flow: createKycLink → getKycStatus)       │
│  ├── handlers/persona-kyc.handler.ts                          │
│  │   (Persona flow: createSession → handleInquiryApproved)    │
│  ├── providers/bridge.provider.ts                             │
│  │   (Bridge API: createKycLink, createCustomer, createWallet)│
│  ├── providers/persona.provider.ts                            │
│  │   (Persona API: createInquiry, getInquiry, getDocument)    │
│  ├── webhooks/bridge-webhook.controller.ts                    │
│  │   (Handles customer.updated → triggers wallet setup)       │
│  └── webhooks/persona-webhook.controller.ts                   │
│      (Handles inquiry.approved → creates Bridge customer)     │
└──────────────────────────────────────────────────────────────┘
```

### Current Database Tables (KYC-Related)

| Table | Purpose | Records |
|---|---|---|
| `BridgeCustomer` | Links Anzo user to Bridge customer. Stores KYC status, rejection info, Persona inquiry ID, signed agreement ID | One per user |
| `BridgeWallet` | Bridge-managed wallet (Solana) for on-ramp | One per BridgeCustomer |
| `BridgeVirtualAccount` | Virtual bank accounts (USD + EUR) for deposits | Up to 2 per BridgeCustomer |
| `BridgeExternalAccount` | User's real bank accounts for off-ramp | Many per BridgeCustomer |
| `BridgeTransaction` | On-ramp/off-ramp transfer records | Many per user |

### Current Environment Variables (KYC-Related)

```
BRIDGE_API_KEY=sk-live-0b1d8f1c21c0e83434817b01a2671a38
BRIDGE_API_URL=https://api.bridge.xyz/v0
BRIDGE_WEBHOOK_PUBLIC_KEY=-----BEGIN PUBLIC KEY-----...
BRIDGE_DEVELOPER_FEE_PERCENT=0.5
PERSONA_API_KEY=persona_production_301e031a-...
PERSONA_TEMPLATE_ID=itmpl_B75iExCrBQ5HooFV6m1XsAQBmBgD
PERSONA_WEBHOOK_SECRET=wbhsec_6bffb580-...
```

### Problems with Current Flow

1. **Persona controls all PII collection** — We don't own the UI or the data. Persona's screens collect name, address, phone, tax ID, ID documents, everything.
2. **Complex PII extraction** — After Persona approves, our backend (`persona-kyc.handler.ts`) has to:
   - Fetch the full inquiry from Persona API
   - Extract 20+ fields (name, DOB, address, phone, national ID, government ID images)
   - Convert country codes (alpha-2 → alpha-3)
   - Map Persona's ID types to Bridge's ID types
   - Download ID images as base64
   - Build a Bridge customer payload
   - Handle edge cases (French subdivisions, phone formatting, missing fields)
3. **Tightly coupled to Bridge** — Adding another partner (e.g., a European banking partner) would mean duplicating all this logic.
4. **No control over the questionnaire** — We can't ask Bridge-specific questions like "source of funds" or "occupation" through Persona.

---

## 3. New System (What We're Building)

### Risk-Tiered KYC Model

The questionnaire adapts based on the user's country risk level. The country is collected **first**, and determines which fields to show:

```
collect country
     │
     ▼
LOW RISK (e.g., USA, Canada, UK, EU)
  → Basic info (name, DOB, address, phone, tax ID)
  → Government ID scan (Sumsub)
  → Bridge TOS
     │
     ▼
MEDIUM RISK (e.g., conflicting location signals, controlled regions)
  → Basic info + Government ID scan + Bridge TOS
  → Proof of Address (utility bill, bank statement, etc.)
     │
     ▼
HIGH RISK (e.g., high-risk region, age 60+, high-volume transactors)
  → Basic info + Government ID scan + Bridge TOS
  → Proof of Address
  → Financial questions: occupation, employment status,
    source of funds, account purpose,
    expected monthly volume, intermediary status
```

> **Citation:** [Bridge Individuals Compliance](https://apidocs.bridge.xyz/platform/customers/compliance/individuals)  
> *"For high risk populations (i.e. resident of high risk region or a person 60 years or older) and/or high volume transactors, Bridge collects information about the customer's source of funds and expected use."*

**Implementation approach:** We always collect the full set of fields in our questionnaire (it's easier for the user to fill one form), but only **send** the financial fields to Bridge when the country/age triggers the high-risk path. Bridge will reject the customer if high-risk fields are missing for users who need them, so it's safer to always collect and send them.

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                       Mobile App                              │
│                                                               │
│  User taps "EUR Bank Account" or "USA Bank Account"           │
│       │                                                       │
│       ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              KYC Checklist Screen                         │ │
│  │              (Adapts based on country risk tier)          │ │
│  │                                                           │ │
│  │  Step 1: ☐ Questionnaire (OUR custom form)               │ │
│  │  ─────────────────────────────────────────                │ │
│  │  ALL tiers: Name, DOB, Address, Phone, Tax ID            │ │
│  │  HIGH RISK: + Occupation, Source of Funds,                │ │
│  │             Employment Status, Expected Monthly Volume    │ │
│  │                                                           │ │
│  │  Step 2: ☐ ID Verification (Sumsub native SDK)           │ │
│  │  ─────────────────────────────────────────                │ │
│  │  ALL tiers: Government ID (passport/DL/national ID)      │ │
│  │  + Selfie/Liveness check                                  │ │
│  │                                                           │ │
│  │  Step 3: ☐ Terms & Conditions (Bridge TOS WebView)       │ │
│  │  ─────────────────────────────────────────                │ │
│  │  ALL tiers: Bridge TOS, captures signed_agreement_id      │ │
│  │                                                           │ │
│  │  Step 4: ☐ Proof of Address (MEDIUM + HIGH risk only)    │ │
│  │  ─────────────────────────────────────────                │ │
│  │  Sumsub SDK or custom upload for PoA document             │ │
│  └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                   Backend (NestJS)                             │
│                                                               │
│  NEW: src/modules/kyc/                                        │
│  ├── kyc.controller.ts          (All KYC endpoints)           │
│  ├── kyc.service.ts             (Facade: status, questionnaire│
│  │                               TOS, auto-submission)        │
│  ├── config/                                                  │
│  │   └── partner-steps.config.ts (Which steps per partner)    │
│  ├── handlers/                                                │
│  │   ├── sumsub-kyc.handler.ts  (Sumsub applicant + webhook)  │
│  │   └── partner-submission.handler.ts (Auto-submit to Bridge)│
│  ├── providers/                                               │
│  │   └── sumsub.provider.ts     (Sumsub API client, HMAC auth)│
│  └── webhooks/                                                │
│      ├── sumsub-webhook.controller.ts                         │
│      └── sumsub-webhook.guard.ts (HMAC-SHA256 verification)   │
│                                                               │
│  EXISTING: src/modules/bridge/ (KEPT, not removed)            │
│  ├── providers/bridge.provider.ts  (createCustomer, etc.)     │
│  ├── webhooks/bridge-webhook.controller.ts                    │
│  └── handlers/bridge-kyc.handler.ts (handleKycApproval stays) │
└──────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴──────────┐
                    ▼                    ▼
            ┌──────────────┐    ┌──────────────┐
            │   Sumsub     │    │  Bridge.xyz  │
            │ (ID + selfie)│    │ (fiat ramps) │
            └──────────────┘    └──────────────┘
```

### Key Differences: Old vs New

| Aspect | Old (Persona) | New (Sumsub + Our Form) |
|---|---|---|
| **PII Collection** | Persona collects everything in their UI | We collect via our own custom form |
| **ID Verification** | Persona SDK (scan + selfie) | Sumsub SDK (scan + selfie) |
| **Data Ownership** | PII stored in Persona, we extract after | PII stored in our DB (`KycProfile` table) |
| **TOS** | Bridge TOS via WebView (same) | Bridge TOS via WebView (same) |
| **Bridge Submission** | Backend extracts from Persona → builds payload → submits to Bridge | Backend reads from KycProfile + Sumsub images → builds payload → submits to Bridge |
| **Adding New Partners** | Duplicate entire Persona extraction logic | Add config entry + payload builder |
| **Source of Funds** | Not collected (Persona doesn't ask) | Collected in our questionnaire |
| **Tax ID** | Extracted from Persona (unreliable for some countries) | Collected in our questionnaire (user types it) |
| **Proof of Address** | Not supported | Sumsub can handle it or custom upload |

---

## 4. Provider Deep Dive — Sumsub

### What is Sumsub?

Sumsub is an identity verification platform. In our new flow, we use it **only for document verification** (scanning a government ID + liveness/selfie check). We do NOT use Sumsub for collecting PII — that's our questionnaire.

### How Sumsub Integration Works

#### Step 1: Setup in Sumsub Dashboard

1. Create an account at [sumsub.com](https://sumsub.com)
2. Create an **App Token** (gives you `SUMSUB_APP_TOKEN` and `SUMSUB_SECRET_KEY`)
3. Create a **Verification Level** (e.g., `basic-kyc-level`) that includes:
   - Identity document check (passport, driver's license, national ID)
   - Liveness/selfie check
   - Optionally: Proof of Address check
4. Set up a **Webhook** endpoint (our backend URL)

> **Citation:** [Sumsub App Tokens](https://docs.sumsub.com/docs/app-tokens)  
> **Citation:** [Sumsub Verification Levels](https://docs.sumsub.com/docs/verification-levels)

#### Step 2: Backend Generates Access Token

Our backend calls Sumsub's API to generate an SDK access token:

```
POST https://api.sumsub.com/resources/accessTokens/sdk
Headers:
  X-App-Token: <SUMSUB_APP_TOKEN>
  X-App-Access-Sig: <HMAC-SHA256 signature>
  X-App-Access-Ts: <unix timestamp>
Body:
{
  "userId": "anzo_user_clxyz123",        // Our user ID (becomes externalUserId)
  "levelName": "basic-kyc-level",         // Verification level from dashboard
  "applicantIdentifiers": {
    "email": "rohan@gmail.com"            // User's email
  },
  "ttlInSecs": 600                        // Token valid for 10 minutes
}
```

Response:
```json
{
  "token": "_act-b8ebfb63-5f24-4b89-9c08-000000000000",
  "userId": "anzo_user_clxyz123"
}
```

> **Citation:** [Sumsub Generate Access Token](https://docs.sumsub.com/reference/generate-access-token)

#### Step 3: Mobile App Opens Sumsub SDK

The mobile app initializes Sumsub's native SDK with the access token. The SDK:
- Shows camera UI to scan government ID (front + back if needed)
- Takes a selfie with liveness detection
- Uploads everything to Sumsub's servers
- Shows progress/result to the user

> **Citation:** [Sumsub WebSDK](https://docs.sumsub.com/docs/about-web-sdk) | [Sumsub React Native Module](https://docs.sumsub.com/docs/react-native-module)

#### Step 4: Sumsub Processes & Sends Webhook

Sumsub processes the documents (OCR, face match, fraud detection) and sends a webhook to our backend:

```
POST /api/v1/kyc/webhook/sumsub
Headers:
  X-Payload-Digest: <HMAC-SHA256 of body using SUMSUB_WEBHOOK_SECRET>
Body:
{
  "type": "applicantReviewed",
  "applicantId": "65abc123def456",
  "inspectionId": "65abc123def789",
  "correlationId": "anzo_user_clxyz123",
  "externalUserId": "anzo_user_clxyz123",
  "reviewResult": {
    "reviewAnswer": "GREEN",     // GREEN = approved, RED = rejected
    "reviewRejectType": null
  },
  "reviewStatus": "completed"
}
```

> **Citation:** [Sumsub Webhooks](https://docs.sumsub.com/docs/webhooks) | [Sumsub User Verification Webhooks](https://docs.sumsub.com/docs/user-verification-webhooks)

#### Step 5: Backend Downloads ID Images from Sumsub

After webhook confirms approval, our backend fetches the document images from Sumsub:

```
GET https://api.sumsub.com/resources/inspections/{inspectionId}/resources/{imageId}
```

We download the front/back of the ID as base64 to include in the Bridge customer payload.

> **Citation:** [Sumsub Get Document Images](https://docs.sumsub.com/reference/get-document-images)

### Sumsub Authentication (HMAC-SHA256)

Every Sumsub API request must be signed:

```
Signature = HMAC-SHA256(
  key: SUMSUB_SECRET_KEY,
  message: timestamp + method + url + body
)
```

Headers required:
- `X-App-Token`: Your app token
- `X-App-Access-Sig`: The HMAC signature
- `X-App-Access-Ts`: Unix timestamp (seconds)

> **Citation:** [Sumsub Authentication](https://docs.sumsub.com/reference/authentication)

---

## 5. Provider Deep Dive — Bridge.xyz

### What is Bridge.xyz?

Bridge.xyz is a fiat on-ramp/off-ramp platform. It lets users:
- Deposit USD/EUR from bank accounts → receive USDC (on-ramp)
- Send USDC → receive USD/EUR in bank accounts (off-ramp)

Bridge handles the money movement, but requires KYC verification before a user can transact.

### Bridge Customer Creation (The Core of the New Flow)

Instead of using Bridge's hosted KYC (which is the old `createKycLink` flow), we now use the **Customers API** to submit customer data directly:

```
POST https://api.bridge.xyz/v0/customers
Headers:
  Api-Key: <BRIDGE_API_KEY>
  Idempotency-Key: <unique UUID>
  Content-Type: application/json
```

> **Citation:** [Bridge Customers API](https://apidocs.bridge.xyz/platform/customers/customers/api)

### What Data Bridge Requires

#### For ALL Individual Customers

| Field | Source in New Flow | Bridge API Field |
|---|---|---|
| First name | Our questionnaire → `KycProfile.firstName` | `first_name` |
| Last name | Our questionnaire → `KycProfile.lastName` | `last_name` |
| Email | `User.email` | `email` |
| Date of birth | Our questionnaire → `KycProfile.dateOfBirth` | `birth_date` (YYYY-MM-DD) |
| Phone | Our questionnaire → `KycProfile.phone` | `phone` (E.164) |
| Street address | Our questionnaire → `KycProfile.streetLine1` | `residential_address.street_line_1` |
| City | Our questionnaire → `KycProfile.city` | `residential_address.city` |
| State/Province | Our questionnaire → `KycProfile.state` | `residential_address.subdivision` |
| Postal code | Our questionnaire → `KycProfile.postalCode` | `residential_address.postal_code` |
| Country | Our questionnaire → `KycProfile.country` | `residential_address.country` (alpha-3) |
| Terms of Service | Bridge TOS WebView → `signed_agreement_id` | `signed_agreement_id` |
| National ID number | For non-USA: our questionnaire → `KycProfile.taxId` | `identifying_information[].number` |
| SSN | For USA only: our questionnaire → `KycProfile.taxId` | `identifying_information[].type: "ssn"` |
| Government ID image | Sumsub → downloaded as base64 | `identifying_information[].image_front/back` |

> **Citation:** [Bridge Individuals Compliance](https://apidocs.bridge.xyz/platform/customers/compliance/individuals)

#### For HIGH-RISK Populations (International, 60+, High Volume)

These additional fields are required. **All enum values must match Bridge's API exactly.**

| Field | Source | Bridge API Field |
|---|---|---|
| Occupation | Our questionnaire → `KycProfile.occupation` | `most_recent_occupation` (numeric code as string) |
| Employment status | Our questionnaire → `KycProfile.employmentStatus` | `employment_status` (enum) |
| Source of funds | Our questionnaire → `KycProfile.sourceOfFunds` | `source_of_funds` (enum) |
| Account purpose | Our questionnaire → `KycProfile.accountPurpose` | `account_purpose` (enum) |
| Expected monthly volume | Our questionnaire → `KycProfile.expectedMonthlyVolume` | `expected_monthly_payments_usd` (enum) |
| Acting as intermediary | Our questionnaire → `KycProfile.actingAsIntermediary` | `acting_as_intermediary` (enum) |

##### Bridge Enum Values — Exact Values for Questionnaire Dropdowns

**`employment_status`** (Bridge enum):
| Value | Label for UI |
|---|---|
| `employed` | Employed |
| `self_employed` | Self-Employed |
| `unemployed` | Unemployed |
| `retired` | Retired |
| `student` | Student |

**`source_of_funds`** (Bridge enum):
| Value | Label for UI |
|---|---|
| `salary` | Salary |
| `business_income` | Business Income |
| `investments` | Investments |
| `inheritance` | Inheritance |
| `gift` | Gift |
| `savings` | Savings |
| `pension_or_government_benefits` | Pension or Government Benefits |
| `legal_settlement` | Legal Settlement |
| `sale_of_property_or_assets` | Sale of Property or Assets |
| `other` | Other |

**`account_purpose`** (Bridge enum):
| Value | Label for UI |
|---|---|
| `personal_use` | Personal Use |
| `business_use` | Business Use |
| `investment` | Investment |
| `savings` | Savings |
| `sending_money_to_friends_or_family` | Sending Money to Friends or Family |
| `paying_for_goods_or_services` | Paying for Goods or Services |
| `other` | Other |

**`expected_monthly_payments_usd`** (Bridge enum):
| Value | Label for UI |
|---|---|
| `less_than_1000` | Less than $1,000 |
| `1000_4999` | $1,000 – $4,999 |
| `5000_9999` | $5,000 – $9,999 |
| `10000_49999` | $10,000 – $49,999 |
| `50000_or_more` | $50,000 or more |

**`acting_as_intermediary`** (Bridge enum):
| Value | Label for UI |
|---|---|
| `yes` | Yes |
| `no` | No |

**`most_recent_occupation`** — Bridge uses **numeric SOC (Standard Occupational Classification) codes** passed as strings, NOT descriptive text. There are 580+ valid occupation codes.

Example occupation codes:

| Code | Occupation |
|---|---|
| `151252` | Software developer |
| `132011` | Accountant and auditor |
| `172051` | Civil engineer |
| `291210` | Physician (general) |
| `251000` | Postsecondary teacher |
| `111011` | Chief executive |
| `131199` | Business operations specialist, other |
| `172011` | Aerospace engineer |
| `291181` | Audiologist |
| `412010` | Cashier |
| `113012` | Administrative services manager |

> **Full occupation list (580+ codes):** [Bridge SoF-EU Most Recent Occupation List](https://apidocs.bridge.xyz/page/sof-eu-most-recent-occupation-list)

**Implementation note:** The mobile app should present a searchable dropdown that shows human-readable occupation names (e.g., "Software Developer") but stores/sends the numeric code (e.g., `"151252"`) to `KycProfile.occupation`. The full mapping should be maintained as a static config file or fetched once and cached.

> **Citation:** [Bridge Individuals Compliance - High Risk](https://apidocs.bridge.xyz/platform/customers/compliance/individuals)  
> **Citation:** [Bridge SoF-EU Occupation List](https://apidocs.bridge.xyz/page/sof-eu-most-recent-occupation-list)

#### Proof of Address (When Required)

Bridge may require PoA when:
- Individual provides a government ID from a controlled/prohibited region but claims to live elsewhere
- Conflicting location signals detected

Accepted documents (must be within 90 days):
- Bank statement
- Utility bill
- Government-issued letter
- Residential lease agreement (can be older if current)

> **Citation:** [Bridge Proof of Address](https://apidocs.bridge.xyz/platform/customers/compliance/poa)

### Bridge Endorsements

Bridge uses "endorsements" to control what services a customer can access:
- `base` — Access to US rails (ACH/Wire)
- `sepa` — Access to European SEPA transfers

When you create a customer, you request endorsements. Bridge reviews the customer and approves/rejects each endorsement independently.

### Bridge TOS (Terms of Service)

Before creating a customer, Bridge requires a `signed_agreement_id`.

#### How to get a TOS URL

**For new customers (no Bridge customer yet):**
```
POST https://api.bridge.xyz/v0/customers/tos_links
Headers:
  Api-Key: <BRIDGE_API_KEY>
  Idempotency-Key: <unique UUID>

Response:
{
  "url": "https://dashboard.bridge.xyz/accept-terms-of-service?session_token=4d5d8c45-..."
}
```

**For existing customers (new TOS version acceptance):**
```
GET https://api.bridge.xyz/v0/customers/{customer_id}/tos_acceptance_link
Headers:
  Api-Key: <BRIDGE_API_KEY>

Response:
{
  "url": "https://dashboard.bridge.xyz/accept-terms-of-service?email=...&t=..."
}
```

#### How to capture `signed_agreement_id`

There are **two methods**. The `redirect_uri` approach is preferred for the new flow.

##### Method 1: `redirect_uri` (PREFERRED for new flow)

Append a `redirect_uri` query parameter to the TOS URL before opening it in the WebView:

```
https://dashboard.bridge.xyz/accept-terms-of-service?session_token=xxx&redirect_uri=https://api.your-app.com/api/v1/kyc/tos/callback
```

When the user accepts the TOS, Bridge **redirects the WebView** to:
```
https://api.your-app.com/api/v1/kyc/tos/callback?signed_agreement_id=sa_xyz789
```

**Backend implementation:**

The callback endpoint (`GET /api/v1/kyc/tos/callback`) reads the `signed_agreement_id` from the query parameter, saves it to `KycVerification.metadata`, marks the TOS step as `approved`, and returns an HTML page that tells the mobile app to close the WebView (via a deep link or `window.close()`).

Alternatively, the `redirect_uri` can be a deep link back into the mobile app (e.g., `anzo://kyc/tos-callback?signed_agreement_id=sa_xyz789`), letting the app handle it natively.

**Why this is better:** It works reliably across all WebView implementations and avoids cross-origin `postMessage` issues.

##### Method 2: `postMessage` (FALLBACK — currently used in Persona flow)

For iFrame/WebView implementations, listen for the `postMessage` event:

```javascript
window.addEventListener('message', (event) => {
  if (event.data?.signedAgreementId) {
    // Send to backend
    saveSignedAgreementId(event.data.signedAgreementId);
  }
});
```

**When to use:** As a fallback if `redirect_uri` is not feasible in a particular WebView context.

> **Citation:** [Bridge Terms of Service](https://apidocs.bridge.xyz/platform/customers/customers/tos)  
> *"You can pass a `redirect_uri` query parameter that will redirect the ToS page back to your application with a `signed_agreement_id` as a query parameter."*  
> **Citation:** [Bridge ToS Links API](https://apidocs.bridge.xyz/api-reference/customers/request-a-hosted-url-for-tos-acceptance-for-new-customer-creation)  
> **Citation:** [Bridge ToS Acceptance Link API (existing customers)](https://apidocs.bridge.xyz/api-reference/customers/retrieve-a-hosted-url-for-tos-acceptance-for-an-existing-customer)

---

## 6. How the Two Providers Work Together

```
┌─────────────────────────────────────────────────────────────────┐
│                         THE FLOW                                 │
│                                                                  │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────────────┐  │
│  │  STEP 1       │    │  STEP 2        │    │  STEP 3           │  │
│  │  Questionnaire│    │  Sumsub SDK    │    │  Bridge TOS       │  │
│  │  (Our Form)   │    │  (ID + Selfie) │    │  (WebView)        │  │
│  │               │    │                │    │                    │  │
│  │  User types:  │    │  User scans:   │    │  User accepts:    │  │
│  │  - Name       │    │  - Passport/ID │    │  - Bridge's TOS   │  │
│  │  - DOB        │    │  - Selfie      │    │                    │  │
│  │  - Address    │    │                │    │  Returns:          │  │
│  │  - Phone      │    │  Sumsub        │    │  signed_agreement  │  │
│  │  - Tax ID     │    │  returns:      │    │  _id               │  │
│  │  - Occupation │    │  - Approval    │    │                    │  │
│  │  - Source of  │    │  - Doc images  │    │                    │  │
│  │    funds      │    │  - Nationality │    │                    │  │
│  └──────┬───────┘    └──────┬─────────┘    └────────┬───────────┘  │
│         │                    │                        │              │
│         ▼                    ▼                        ▼              │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    ALL STEPS COMPLETE?                         │   │
│  │                    (Required steps only)                       │   │
│  │                                                               │   │
│  │  YES → Auto-submit to Bridge:                                 │   │
│  │                                                               │   │
│  │  POST /v0/customers                                           │   │
│  │  {                                                            │   │
│  │    type: "individual",                                        │   │
│  │    first_name: KycProfile.firstName,        ← Questionnaire   │   │
│  │    last_name: KycProfile.lastName,           ← Questionnaire   │   │
│  │    email: User.email,                                         │   │
│  │    birth_date: KycProfile.dateOfBirth,       ← Questionnaire   │   │
│  │    phone: KycProfile.phone,                  ← Questionnaire   │   │
│  │    residential_address: { ... },              ← Questionnaire   │   │
│  │    signed_agreement_id: "sa_...",             ← TOS WebView    │   │
│  │    verified_govid_at: "2026-03-30T...",       ← Sumsub         │   │
│  │    verified_selfie_at: "2026-03-30T...",      ← Sumsub         │   │
│  │    identifying_information: [                                  │   │
│  │      { type: "pan", number: "ABCDE1234F",    ← Questionnaire  │   │
│  │        issuing_country: "IND" },                               │   │
│  │      { type: "passport", image_front: "...", ← Sumsub          │   │
│  │        issuing_country: "IND" }                                │   │
│  │    ],                                                          │   │
│  │    most_recent_occupation: "151252",             ← Questionnaire│   │
│  │    source_of_funds: "salary",                   ← Questionnaire│   │
│  │    expected_monthly_payments_usd: "1000_4999"   ← Questionnaire│   │
│  │  }                                                             │   │
│  └──────────────────────────────────────────────────────────────┘  │
│                              │                                     │
│                              ▼                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Bridge processes → approves/rejects → sends webhook          │  │
│  │  → handleKycApproval() creates wallet + virtual accounts      │  │
│  │  → User is now ACTIVE and can on-ramp/off-ramp                │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 7. Country-Specific Requirements

### Tax/National ID Type by Country

The questionnaire form should dynamically show the correct label based on the user's country. Here are the most common ones:

| Country | ID Type Code | What to Ask For | Example |
|---|---|---|---|
| USA | `ssn` | Social Security Number | 123-45-6789 |
| India | `pan` | Permanent Account Number (PAN) | ABCDE1234F |
| UK | `nino` | National Insurance Number | QQ 12 34 56 C |
| Germany | `steuer_id` | Tax Identification Number | 12345678901 |
| France | `spi` | Social Security Number | 1 23 45 67 890 123 45 |
| Canada | `sin` | Social Insurance Number | 123-456-789 |
| Australia | `tfn` | Tax File Number | 123 456 789 |
| Brazil | `cpf` | CPF Number | 123.456.789-01 |
| Singapore | `nric` | NRIC Number | S1234567A |
| Nigeria | `tin` or `bvn` | Tax ID or Bank Verification Number | 1234567890 |

> **Full mapping:** Already exists in `persona-kyc.handler.ts` as `COUNTRY_TO_BRIDGE_ID_TYPE` (112 countries mapped)  
> **Citation:** [Bridge Individual ID Numbers by Country](https://apidocs.bridge.xyz/platform/customers/compliance/individuals#individual-identification-numbers-by-country)

### Government ID Documents by Country

Bridge supports passport verification for virtually all countries. Driver's license and national ID support varies:

| Country | Passport | Driver's License | National ID | Notes |
|---|---|---|---|---|
| USA | Yes | Yes (all 51 states) | Yes (all states) | Also: permanent resident card, visa |
| India | Yes | Yes (back required) | Yes | Also: PAN card, Voter ID |
| UK | Yes | Yes | Yes | Also: voter ID |
| Germany | Yes | Yes | Yes (back required) | Also: residency permit |
| France | Yes | Yes | Yes (back required) | Also: residency permit |
| Nigeria | Yes | Yes | Yes (back required) | Also: voter ID |
| Brazil | Yes | Yes (back required) | Yes (back required) | Also: residency permit |

> **Citation:** [Bridge Government IDs by Country](https://apidocs.bridge.xyz/platform/customers/customers/api#government-ids-by-country)

---

## 8. Real-Life Example (Rohan from India)

### Scenario
Rohan Sharma, 28, is a software developer in Bangalore. He wants to use Anzo to receive USD payments from his US-based freelance clients directly into his Indian bank account.

### Step-by-Step Flow

#### Step 1: Rohan taps "Get USD Bank Account" in Anzo app

The app calls:
```
GET /api/v1/kyc/status/bridge
```

Response:
```json
{
  "partnerId": "bridge",
  "overallStatus": "not_started",
  "steps": [
    { "step": "questionnaire", "status": "not_started", "required": true },
    { "step": "id_verification", "status": "not_started", "required": true },
    { "step": "tos", "status": "not_started", "required": true },
    { "step": "proof_of_address", "status": "not_started", "required": false }
  ]
}
```

#### Step 2: Rohan fills the Questionnaire

The app shows a form. Based on his country (India), the Tax ID field shows "PAN (Permanent Account Number)".

```
POST /api/v1/kyc/questionnaire
{
  "firstName": "Rohan",
  "lastName": "Sharma",
  "dateOfBirth": "1997-08-15",
  "phone": "+919876543210",
  "phoneCountryCode": "IN",
  "streetLine1": "42, Koramangala 5th Block",
  "city": "Bangalore",
  "state": "KA",
  "postalCode": "560095",
  "country": "IND",
  "taxId": "ABCRS1234K",
  "taxIdCountry": "IND",
  "occupation": "151252",
  "employmentStatus": "employed",
  "sourceOfFunds": "salary",
  "accountPurpose": "paying_for_goods_or_services",
  "expectedMonthlyVolume": "1000_4999",
  "actingAsIntermediary": "no"
}
```

This saves to `KycProfile` and marks the questionnaire step as `approved`.

#### Step 3: Rohan does ID Verification via Sumsub

App calls:
```
POST /api/v1/kyc/sumsub/session
```

Backend creates a Sumsub applicant with `userId: rohan_user_id` and returns:
```json
{ "token": "_act-abc123..." }
```

App opens Sumsub native SDK. Rohan:
1. Scans his Indian passport (front page)
2. Takes a selfie with liveness detection
3. Sumsub processes (usually < 1 minute)

Sumsub sends webhook to our backend:
```json
{
  "type": "applicantReviewed",
  "externalUserId": "rohan_user_id",
  "reviewResult": { "reviewAnswer": "GREEN" }
}
```

Backend marks `id_verification` as `approved`.

#### Step 4: Rohan accepts Bridge TOS

App calls:
```
GET /api/v1/kyc/tos-link/bridge
```

Returns:
```json
{
  "tosLink": "https://dashboard.bridge.xyz/accept-terms-of-service?session_token=xxx&redirect_uri=https://api.anzo.com/api/v1/kyc/tos/callback"
}
```

The backend **appends `redirect_uri`** to Bridge's TOS URL before returning it to the app.

App opens WebView with this URL. Rohan reads and accepts. Bridge **redirects** the WebView to:
```
https://api.anzo.com/api/v1/kyc/tos/callback?signed_agreement_id=sa_xyz789
```

The callback endpoint saves `signed_agreement_id` and marks the TOS step as approved. The WebView redirects to a deep link (`anzo://kyc/tos-complete`) so the app closes the WebView and refreshes the checklist.

**Fallback:** If `redirect_uri` doesn't work (e.g., some WebView configs), the app listens for `postMessage` from the TOS page and calls:
```
POST /api/v1/kyc/tos
{ "signedAgreementId": "sa_xyz789", "partnerId": "bridge" }
```

#### Step 5: Auto-submission to Bridge

All required steps are now complete. The backend automatically:

1. Reads Rohan's `KycProfile` (name, DOB, address, phone, PAN number)
2. Downloads Rohan's passport images from Sumsub (base64)
3. Gets `signed_agreement_id` from TOS step
4. Builds Bridge customer payload:

```json
{
  "type": "individual",
  "first_name": "Rohan",
  "last_name": "Sharma",
  "email": "rohan.sharma@gmail.com",
  "phone": "+919876543210",
  "birth_date": "1997-08-15",
  "residential_address": {
    "street_line_1": "42, Koramangala 5th Block",
    "city": "Bangalore",
    "subdivision": "KA",
    "postal_code": "560095",
    "country": "IND"
  },
  "signed_agreement_id": "sa_xyz789",
  "verified_govid_at": "2026-03-30T10:30:00Z",
  "verified_selfie_at": "2026-03-30T10:30:00Z",
  "most_recent_occupation": "151252",
  "employment_status": "employed",
  "source_of_funds": "salary",
  "account_purpose": "paying_for_goods_or_services",
  "expected_monthly_payments_usd": "1000_4999",
  "acting_as_intermediary": "no",
  "identifying_information": [
    {
      "type": "pan",
      "issuing_country": "IND",
      "number": "ABCRS1234K"
    },
    {
      "type": "passport",
      "issuing_country": "IND",
      "image_front": "data:image/jpg;base64,/9j/4AAQ..."
    }
  ]
}
```

5. Sends `POST /v0/customers` to Bridge
6. Bridge processes → approves → sends `customer.updated` webhook
7. Backend creates wallet + virtual accounts (USD + EUR)
8. Rohan gets a push notification: "Verification Complete! You can now deposit and withdraw funds."

---

## 9. Real-Life Example (Priya from USA)

### Scenario
Priya Patel, 32, lives in San Francisco. She wants to on-ramp USD to USDC via ACH.

### Key Differences for US Customers

- **Tax ID** = SSN (Social Security Number), not PAN
- **No "high risk" extra fields needed** (USA is not a high-risk region, assuming she's under 60)
- **Bridge may approve instantly** via database checks (no photo ID needed for < $10K/tx)

### Questionnaire

```json
{
  "firstName": "Priya",
  "lastName": "Patel",
  "dateOfBirth": "1993-11-22",
  "phone": "+14155559876",
  "phoneCountryCode": "US",
  "streetLine1": "456 Market Street, Apt 7B",
  "city": "San Francisco",
  "state": "CA",
  "postalCode": "94103",
  "country": "USA",
  "taxId": "123-45-6789",
  "taxIdCountry": "USA",
  "occupation": "131199",
  "employmentStatus": "employed",
  "sourceOfFunds": "salary",
  "accountPurpose": "personal_use",
  "expectedMonthlyVolume": "5000_9999",
  "actingAsIntermediary": "no"
}
```

### Bridge Payload (Simpler for US)

```json
{
  "type": "individual",
  "first_name": "Priya",
  "last_name": "Patel",
  "email": "priya.patel@gmail.com",
  "birth_date": "1993-11-22",
  "residential_address": {
    "street_line_1": "456 Market Street, Apt 7B",
    "city": "San Francisco",
    "subdivision": "CA",
    "postal_code": "94103",
    "country": "USA"
  },
  "signed_agreement_id": "sa_abc123",
  "verified_govid_at": "2026-03-30T...",
  "verified_selfie_at": "2026-03-30T...",
  "identifying_information": [
    {
      "type": "ssn",
      "issuing_country": "USA",
      "number": "123-45-6789"
    },
    {
      "type": "drivers_license",
      "issuing_country": "USA",
      "image_front": "data:image/jpg;base64,...",
      "image_back": "data:image/jpg;base64,..."
    }
  ]
}
```

Bridge may instantly approve (status goes `not_started` → `active`) because US customers with valid SSN can be verified via database checks.

---

## 10. Real-Life Example (Arjun from Germany/EU)

### Scenario
Arjun Mehta, 35, lives in Berlin. He wants to use EUR SEPA deposits.

### Key Differences for EU Customers

- **Tax ID** = `steuer_id` (German tax ID)
- **May need Proof of Address** if conflicting signals (Indian passport + German address)
- **Needs `sepa` endorsement** in addition to `base`

### Bridge Payload

```json
{
  "type": "individual",
  "first_name": "Arjun",
  "last_name": "Mehta",
  "email": "arjun.mehta@gmail.com",
  "phone": "+491721234567",
  "birth_date": "1990-06-12",
  "nationality": "IND",
  "residential_address": {
    "street_line_1": "Friedrichstraße 42",
    "city": "Berlin",
    "subdivision": "BE",
    "postal_code": "10117",
    "country": "DEU"
  },
  "signed_agreement_id": "sa_def456",
  "endorsements": ["base", "sepa"],
  "verified_govid_at": "2026-03-30T...",
  "verified_selfie_at": "2026-03-30T...",
  "identifying_information": [
    {
      "type": "steuer_id",
      "issuing_country": "DEU",
      "number": "12345678901"
    },
    {
      "type": "passport",
      "issuing_country": "IND",
      "image_front": "data:image/jpg;base64,..."
    }
  ],
  "documents": [
    {
      "purposes": ["proof_of_address"],
      "file": "data:image/jpg;base64,..."
    }
  ]
}
```

Note: Bridge may request PoA because Arjun has an Indian passport but claims to live in Germany (conflicting location signals).

---

## 11. Database Schema Changes

### New Tables Needed

#### `KycProfile` (NEW)

Stores all PII collected from our questionnaire. One per user.

```prisma
model KycProfile {
  id                     String   @id @default(cuid())
  userId                 String   @unique
  
  // Personal Info
  firstName              String
  middleName             String?
  lastName               String
  dateOfBirth            String?  // YYYY-MM-DD
  nationality            String?  // ISO alpha-3
  
  // Contact
  phone                  String   // E.164 format
  phoneCountryCode       String   // ISO alpha-2 (for dial code lookup)
  
  // Address
  streetLine1            String
  streetLine2            String?
  city                   String
  state                  String?  // State/province/subdivision
  postalCode             String
  country                String   // ISO alpha-3
  
  // Financial Profile (for Bridge high-risk requirements)
  // All values below must use exact Bridge API enum values
  occupation             String?  // Bridge numeric SOC code as string (e.g., "151252" = Software Developer)
  employmentStatus       String?  // Bridge enum: employed | self_employed | unemployed | retired | student
  sourceOfFunds          String?  // Bridge enum: salary | business_income | investments | inheritance | gift | savings | pension_or_government_benefits | legal_settlement | sale_of_property_or_assets | other
  accountPurpose         String?  // Bridge enum: personal_use | business_use | investment | savings | sending_money_to_friends_or_family | paying_for_goods_or_services | other
  expectedMonthlyVolume  String?  // Bridge enum: less_than_1000 | 1000_4999 | 5000_9999 | 10000_49999 | 50000_or_more
  actingAsIntermediary   String?  // Bridge enum: yes | no
  
  // Tax Identification
  taxId                  String?  // Country-specific (SSN, PAN, NINO, etc.)
  taxIdType              String?  // Bridge type code (ssn, pan, steuer_id, etc.)
  taxIdCountry           String?  // ISO alpha-3
  
  // Sumsub Integration
  sumsubApplicantId      String?  @unique
  sumsubInspectionId     String?
  sumsubReviewStatus     String?  // init, pending, completed, onHold
  sumsubReviewAnswer     String?  // GREEN, RED
  sumsubVerifiedAt       DateTime? // When Sumsub approved the documents
  
  createdAt              DateTime @default(now())
  updatedAt              DateTime @updatedAt
  
  user                   User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  verifications          KycVerification[]
  bridgeCustomer         BridgeCustomer?   // Back-reference: Prisma requires both sides
  
  @@index([sumsubApplicantId])
}
```

#### `KycVerification` (NEW)

Tracks completion of each KYC step per user per partner.

```prisma
model KycVerification {
  id              String    @id @default(cuid())
  kycProfileId    String
  partnerId       String    // "bridge", future: "partner_x"
  step            String    // "questionnaire", "id_verification", "tos", "proof_of_address"
  status          String    @default("not_started") // not_started, in_progress, pending_review, approved, rejected
  metadata        Json?     // Step-specific data (e.g., { signedAgreementId } for TOS)
  rejectionReason String?
  rejectionCount  Int       @default(0)
  completedAt     DateTime?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  kycProfile      KycProfile @relation(fields: [kycProfileId], references: [id], onDelete: Cascade)
  
  @@unique([kycProfileId, partnerId, step])
  @@index([kycProfileId])
  @@index([partnerId, step])
}
```

### Modified Tables

#### `User` — Add relation

```prisma
model User {
  // ... existing fields ...
  KycProfile       KycProfile?       // NEW: Add this relation
}
```

#### `BridgeCustomer` — Add optional FK

```prisma
model BridgeCustomer {
  // ... all existing fields stay ...
  kycProfileId     String?  @unique  // NEW: Optional link to KycProfile
  kycProfile       KycProfile? @relation(fields: [kycProfileId], references: [id])
}
```

### Tables That Stay Unchanged

| Table | Change | Why |
|---|---|---|
| `BridgeWallet` | None | Still created after Bridge approves |
| `BridgeVirtualAccount` | None | Still created after Bridge approves |
| `BridgeExternalAccount` | None | Off-ramp accounts, unrelated to KYC flow |
| `BridgeTransaction` | None | Transfer tracking, unrelated to KYC flow |
| All other tables | None | Not related to KYC |

### Migration Strategy for Existing Users

- Users with `BridgeCustomer.status === 'ACTIVE'` are **grandfathered**
- They do NOT need a `KycProfile`
- The `GET /kyc/status/bridge` endpoint returns all steps as `approved` for them
- New users go through the new modular flow
- Persona endpoints stay active during transition for in-flight verifications

---

## 12. Environment Variables

### New Variables to Add

```bash
# ============================================
# Sumsub Integration (NEW — Required)
# ============================================
# Get these from Sumsub Dashboard > Developers > App Tokens
SUMSUB_APP_TOKEN=sbx:xxxxxxxxxxxxxxxxxxxx
SUMSUB_SECRET_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# The verification level name configured in Sumsub Dashboard
# This determines what checks Sumsub performs (ID + selfie, optionally PoA)
SUMSUB_LEVEL_NAME=basic-kyc-level

# Sumsub API base URL (use https://api.sumsub.com for production)
SUMSUB_BASE_URL=https://api.sumsub.com

# Webhook secret for verifying Sumsub webhook signatures (HMAC-SHA256)
# Get this from Sumsub Dashboard > Developers > Webhooks
SUMSUB_WEBHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Existing Variables — No Changes Needed

```bash
# These stay exactly as they are:
BRIDGE_API_KEY=sk-live-0b1d8f1c21c0e83434817b01a2671a38
BRIDGE_API_URL=https://api.bridge.xyz/v0      # (from validation.schema default)
BRIDGE_WEBHOOK_PUBLIC_KEY=-----BEGIN PUBLIC KEY-----...
BRIDGE_DEVELOPER_FEE_PERCENT=0.5
```

### Existing Variables — Keep During Transition, Remove Later

```bash
# These stay active during transition for in-flight Persona verifications:
PERSONA_API_KEY=persona_production_301e031a-...
PERSONA_TEMPLATE_ID=itmpl_B75iExCrBQ5HooFV6m1XsAQBmBgD
PERSONA_WEBHOOK_SECRET=wbhsec_6bffb580-...
# Remove after all Persona flows are migrated to Sumsub
```

### Changes to `validation.schema.ts`

Add to the Joi schema:

```typescript
// Sumsub Integration
SUMSUB_APP_TOKEN: Joi.string().optional(),     // Optional during rollout
SUMSUB_SECRET_KEY: Joi.string().optional(),
SUMSUB_LEVEL_NAME: Joi.string().optional().default('basic-kyc-level'),
SUMSUB_BASE_URL: Joi.string().optional().default('https://api.sumsub.com'),
SUMSUB_WEBHOOK_SECRET: Joi.string().optional(),
```

---

## 13. Full API Endpoint Design

### New KYC Module Endpoints

All under `/api/v1/kyc`, all authenticated (AuthGuard) except webhooks.

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/kyc/status/:partnerId` | Get KYC checklist with step statuses |
| `GET` | `/kyc/questionnaire` | Get saved PII for form pre-fill |
| `POST` | `/kyc/questionnaire` | Save/update PII from form |
| `POST` | `/kyc/sumsub/session` | Create Sumsub applicant + get SDK token |
| `GET` | `/kyc/tos-link/:partnerId` | Get TOS URL (with `redirect_uri` appended) |
| `POST` | `/kyc/tos` | Save TOS acceptance (fallback for `postMessage` method) |
| `GET` | `/kyc/tos/callback` | TOS redirect callback — captures `signed_agreement_id` from `redirect_uri` |
| `POST` | `/kyc/webhook/sumsub` | Sumsub webhook (no auth, HMAC verified) |

### Existing Bridge Endpoints (Kept)

These stay for backward compatibility and for post-approval flows:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/bridge/kyc/status` | Get Bridge customer status (existing flow) |
| `POST` | `/bridge/kyc/initiate` | Initiate hosted KYC (existing flow) |
| `POST` | `/bridge/webhook` | Bridge webhook (customer.updated) |
| `GET` | `/bridge/onramp/*` | On-ramp endpoints |
| `POST` | `/bridge/offramp/*` | Off-ramp endpoints |

### Existing Persona Endpoints (Kept During Transition)

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/bridge/kyc/persona/session` | Create/resume Persona inquiry |
| `GET` | `/bridge/kyc/tos-link` | Get Bridge TOS link |
| `POST` | `/bridge/kyc/persona/webhook` | Persona webhook |

---

## 14. Complete Implementation Workflow

### Phase 1: Core Setup

1. **Create Sumsub Account**
   - Sign up at [sumsub.com](https://sumsub.com)
   - Create App Token → get `SUMSUB_APP_TOKEN` + `SUMSUB_SECRET_KEY`
   - Create verification level `basic-kyc-level` with ID document + liveness check
   - Set up webhook pointing to `https://your-domain/api/v1/kyc/webhook/sumsub`
   - Get `SUMSUB_WEBHOOK_SECRET`

2. **Add env variables** to `.env` (see Section 12)

3. **Run Prisma migration** for new tables (`KycProfile`, `KycVerification`, update `User` and `BridgeCustomer`)

### Phase 2: Backend Implementation

4. **Create `src/modules/kyc/` module** with the structure from Section 3

5. **Implement Sumsub Provider** (`sumsub.provider.ts`)
   - HMAC-SHA256 authentication
   - `createApplicant(userId, email, levelName)` → returns applicant ID
   - `getAccessToken(userId, levelName, email)` → returns SDK token
   - `getApplicant(applicantId)` → returns applicant data
   - `getDocumentImages(inspectionId)` → returns base64 images

6. **Implement KYC Controller & Service**
   - Questionnaire CRUD (save/load KycProfile)
   - Sumsub session creation
   - TOS link & acceptance
   - Status endpoint (reads KycVerification records)

7. **Implement Sumsub Webhook Handler**
   - Verify HMAC signature
   - Handle `applicantReviewed` event
   - Mark `id_verification` step as approved/rejected
   - Check if all partner steps are complete → trigger auto-submission

8. **Implement Partner Submission Handler**
   - `checkAndSubmitAllPartners()` — called after every step completion
   - Assembles data from KycProfile + Sumsub + TOS
   - Builds Bridge customer payload using the existing `COUNTRY_TO_BRIDGE_ID_TYPE` mapping
   - Calls `BridgeProvider.createCustomer(payload)`
   - Creates/updates `BridgeCustomer` record

### Phase 3: Mobile App

9. **Build Questionnaire Screen** — custom form collecting all PII
10. **Integrate Sumsub React Native SDK** — `@sumsub/react-native-mobilesdk-module`
11. **Build TOS WebView** — capture `signed_agreement_id` via `postMessage`
12. **Build KYC Checklist Screen** — shows steps + status

### Phase 4: Transition

13. **Feature flag** on frontend — new users get Sumsub flow, existing users keep Persona
14. **Monitor** both flows in parallel
15. **Remove Persona** once all in-flight verifications complete

---

## 15. What Changes vs What Stays

### Files That Stay (No Changes)

| File/Module | Why |
|---|---|
| `bridge.provider.ts` | Still used for `createCustomer`, `createWallet`, `createVirtualAccount` |
| `bridge-kyc.handler.ts` → `handleKycApproval()` | Still handles post-Bridge-approval wallet/VA setup |
| `bridge-webhook.controller.ts` | Still receives `customer.updated` webhooks |
| `bridge-onramp.handler.ts` | Unrelated to KYC |
| `bridge-offramp.handler.ts` | Unrelated to KYC |
| All other modules | Unrelated to KYC |

### Files That Get New Additions

| File | Change |
|---|---|
| `prisma/schema.prisma` | Add `KycProfile`, `KycVerification` models; add relation to `User` and `BridgeCustomer` |
| `src/config/validation.schema.ts` | Add Sumsub env vars |
| `src/app.module.ts` | Import new `KycModule` |
| `.env` | Add 5 new Sumsub variables |

### New Files to Create

| Path | Purpose |
|---|---|
| `src/modules/kyc/kyc.module.ts` | Module registration |
| `src/modules/kyc/kyc.controller.ts` | All KYC endpoints |
| `src/modules/kyc/kyc.service.ts` | Facade service |
| `src/modules/kyc/config/partner-steps.config.ts` | Step definitions per partner |
| `src/modules/kyc/handlers/sumsub-kyc.handler.ts` | Sumsub applicant management + webhook processing |
| `src/modules/kyc/handlers/partner-submission.handler.ts` | Auto-submit to partners |
| `src/modules/kyc/providers/sumsub.provider.ts` | Sumsub API client |
| `src/modules/kyc/webhooks/sumsub-webhook.controller.ts` | Webhook endpoint |
| `src/modules/kyc/webhooks/sumsub-webhook.guard.ts` | HMAC signature verification |
| `src/modules/kyc/dto/save-questionnaire.dto.ts` | Form validation |
| `src/modules/kyc/dto/save-tos.dto.ts` | TOS validation |
| `src/modules/kyc/dto/sumsub-session.dto.ts` | Session request validation |
| `src/modules/kyc/config/occupation-codes.config.ts` | Bridge occupation code → label mapping (580+ entries) |

### Files to Eventually Remove (After Transition)

| File | When |
|---|---|
| `src/modules/bridge/handlers/persona-kyc.handler.ts` | After Persona migration complete |
| `src/modules/bridge/providers/persona.provider.ts` | After Persona migration complete |
| `src/modules/bridge/webhooks/persona-webhook.controller.ts` | After Persona migration complete |
| `src/modules/bridge/guards/persona-webhook.guard.ts` | After Persona migration complete |
| `src/modules/bridge/dto/persona-session.dto.ts` | After Persona migration complete |

---

## 16. Reference Links

### Bridge.xyz Documentation

| Resource | URL |
|---|---|
| Customers API (Create customers directly) | https://apidocs.bridge.xyz/platform/customers/customers/api |
| Government IDs by Country | https://apidocs.bridge.xyz/platform/customers/customers/api#government-ids-by-country |
| Individuals Compliance (Required fields) | https://apidocs.bridge.xyz/platform/customers/compliance/individuals |
| ID Numbers by Country | https://apidocs.bridge.xyz/platform/customers/compliance/individuals#individual-identification-numbers-by-country |
| Proof of Address Requirements | https://apidocs.bridge.xyz/platform/customers/compliance/poa |
| Terms of Service | https://apidocs.bridge.xyz/platform/customers/customers/tos |
| Endorsements | https://apidocs.bridge.xyz/platform/customers/customers/endorsements |
| Rejection Reasons | https://apidocs.bridge.xyz/platform/customers/customers/rejection_reasons |
| SEPA/Euro Transactions | https://apidocs.bridge.xyz/platform/customers/customers/sepa-euro-transactions |
| Webhooks | https://apidocs.bridge.xyz/platform/customers/customers/webhooks |
| SoF-EU Occupation List (580+ codes) | https://apidocs.bridge.xyz/page/sof-eu-most-recent-occupation-list |
| ToS Links API (new customer) | https://apidocs.bridge.xyz/api-reference/customers/request-a-hosted-url-for-tos-acceptance-for-new-customer-creation |
| ToS Acceptance Link API (existing customer) | https://apidocs.bridge.xyz/api-reference/customers/retrieve-a-hosted-url-for-tos-acceptance-for-an-existing-customer |
| API Reference | https://apidocs.bridge.xyz/api-reference |

### Sumsub Documentation

| Resource | URL |
|---|---|
| About Sumsub API | https://docs.sumsub.com/reference/about-sumsub-api |
| Authentication (HMAC-SHA256) | https://docs.sumsub.com/reference/authentication |
| Generate Access Token (SDK) | https://docs.sumsub.com/reference/generate-access-token |
| Create Applicant | https://docs.sumsub.com/reference/create-applicant |
| Get Document Images | https://docs.sumsub.com/reference/get-document-images |
| Verification Levels | https://docs.sumsub.com/docs/verification-levels |
| WebSDK Overview | https://docs.sumsub.com/docs/about-web-sdk |
| WebSDK Integration Guide | https://docs.sumsub.com/docs/get-started-with-web-sdk |
| React Native Module | https://docs.sumsub.com/docs/react-native-module |
| Webhooks Overview | https://docs.sumsub.com/docs/webhooks |
| User Verification Webhooks | https://docs.sumsub.com/docs/user-verification-webhooks |
| App Tokens | https://docs.sumsub.com/docs/app-tokens |
| Sandbox Testing | https://docs.sumsub.com/docs/test-in-sandbox |
| Applicant Statuses | https://docs.sumsub.com/docs/applicant-statuses |
| Get Applicant Review Status | https://docs.sumsub.com/reference/get-applicant-review-status |

### Internal Documentation

| Resource | Path |
|---|---|
| Modular KYC System Design | `docs/MODULAR_KYC_SYSTEM.md` |
| Prisma Schema | `prisma/schema.prisma` |
| Bridge KYC Handler (current) | `src/modules/bridge/handlers/bridge-kyc.handler.ts` |
| Persona KYC Handler (current) | `src/modules/bridge/handlers/persona-kyc.handler.ts` |
| Env Validation Schema | `src/config/validation.schema.ts` |
| Country → Bridge ID Type Map | `src/modules/bridge/handlers/persona-kyc.handler.ts` (line 63-112) |

---

## Summary: TL;DR for Ritesh

**What your founder wants:**

1. **Replace Persona with Sumsub** for scanning passports/IDs + selfie check
2. **Build your own form** (not inside Sumsub) to collect: name, DOB, address, phone, tax ID, occupation, source of funds
3. **Keep Bridge TOS** in a WebView (same as before)
4. **Auto-submit everything to Bridge** when all steps are done — using Bridge's Customers API directly (instead of Bridge's hosted KYC)
5. **Handle PoA** via Sumsub if needed, or a custom upload
6. **Make it work for all countries** — different tax ID types per country (PAN for India, SSN for USA, etc.)

**What you need to build:**
- 1 new NestJS module (`src/modules/kyc/`)
- 2 new database tables (`KycProfile`, `KycVerification`)
- 5 new environment variables (Sumsub credentials)
- ~12 new TypeScript files
- Mobile: 1 questionnaire screen, Sumsub SDK integration, TOS WebView, checklist screen

**What you DON'T need to change:**
- Bridge module (keep it, reuse `createCustomer`, `handleKycApproval`)
- All existing tables (keep them)
- On-ramp/off-ramp flows (unchanged)
- Bridge webhooks (still needed for `customer.updated`)

**The design doc already exists** at `docs/MODULAR_KYC_SYSTEM.md` with the architectural plan. This analysis document adds the "why" and the real-world context.
