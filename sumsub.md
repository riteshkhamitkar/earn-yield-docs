# Modular KYC System with Sumsub Integration

## Implementation Analysis & Technical Documentation

**PRs Covered:** #202 (Modular KYC system) + #203 (Sumsub integration + auto-submission)
**Base Branch:** `main` → `feature/modular-kyc-system` → `feature/kyc-sumsub-integration`

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Database Changes (Prisma Schema)](#2-database-changes-prisma-schema)
3. [Environment Variables](#3-environment-variables)
4. [Complete End-to-End Flow](#4-complete-end-to-end-flow)
5. [API Endpoints](#5-api-endpoints)
6. [Sumsub Integration Details](#6-sumsub-integration-details)
7. [Bridge Auto-Submission Flow](#7-bridge-auto-submission-flow)
8. [Webhook Handling](#8-webhook-handling)
9. [Frontend Integration Guide](#9-frontend-integration-guide)
10. [Backward Compatibility Analysis](#10-backward-compatibility-analysis)
11. [Scalability & Complexity Analysis](#11-scalability--complexity-analysis)
12. [Issues, Risks & Recommendations](#12-issues-risks--recommendations)
13. [Environment Setup Checklist](#13-environment-setup-checklist)

---

## 1. Architecture Overview

### Current System (Before PRs)

```
User → Persona SDK (ID verification) → Persona webhook → PersonaKycHandler
  → Extracts PII from Persona → Builds Bridge customer payload → POST /v0/customers
  → Bridge webhook (customer.updated) → BridgeKycHandler.handleKycApproval
  → Creates wallet + virtual accounts → ACTIVE
```

**ID Verification Provider:** Persona (via native mobile SDK)
**PII Source:** Persona auto-extracts from ID documents
**TOS:** Bridge-hosted TOS link, `signed_agreement_id` captured via WebView postMessage

### New System (After PRs)

```
User → KYC Module (step-by-step)
  Step 1: POST /kyc/questionnaire → User enters PII → KycProfile created
  Step 2: POST /kyc/sumsub/session → Sumsub SDK token → User does ID verification in-app
  Step 3: POST /kyc/tos → User accepts Bridge TOS → signed_agreement_id saved
  (Step 4: optional Proof of Address via Sumsub)

Sumsub webhook (applicantReviewed GREEN)
  → SumsubKycHandler → marks ID_VERIFICATION approved
  → PartnerSubmissionHandler.checkAndSubmitAllPartners()
  → All required steps done? → Build NormalizedKycData from KycProfile + Sumsub
  → POST /v0/customers to Bridge
  → Bridge webhook (customer.updated) → BridgeKycHandler.handleKycApproval (existing)
  → Wallet + VAs created → ACTIVE
```

**ID Verification Provider:** Sumsub (replaces Persona for new users)
**PII Source:** User-entered questionnaire form (KycProfile) + Sumsub verified data
**TOS:** Same Bridge TOS link flow, now routed through KYC module

### Key Design Decision: Provider-Agnostic KYC Layer

The `KycProfile` model stores PII **independently** of any verification provider. The `KycVerification` model tracks step completion **per partner**. This means:

- Adding a new partner (e.g., "ramp") = add to `KycPartner` enum + `PARTNER_KYC_CONFIGS`
- Switching ID verification providers = swap `SumsubProvider` without touching KYC orchestration
- PII collected once, reused across partners

---

## 2. Database Changes (Prisma Schema)

### New Models

#### `KycProfile` (1 per user, provider-agnostic PII store)

| Field | Type | Purpose |
|-------|------|---------|
| `id` | `String @id` | CUID primary key |
| `userId` | `String @unique` | FK to User, 1:1 |
| `firstName`, `lastName` | `String` | Required PII |
| `middleName` | `String?` | Optional |
| `dateOfBirth` | `String?` | YYYY-MM-DD format |
| `phone` | `String` | E.164 format |
| `phoneCountryCode` | `String` | ISO alpha-2 |
| `streetLine1..country` | `String/String?` | Address fields |
| `occupation`, `annualSalary`, `accountPurpose`, `expectedMonthlyVolume` | `String` | Compliance questionnaire |
| `taxId`, `taxIdCountry` | `String?` | Optional tax identification |
| `sumsubApplicantId` | `String? @unique` | Sumsub applicant reference |
| `sumsubInspectionId` | `String?` | Sumsub inspection reference |
| `sumsubReviewStatus` | `String?` | `init`, `pending`, `completed`, `onHold` |
| `sumsubReviewAnswer` | `String?` | `GREEN`, `RED` |

**Indexes:** `@@index([sumsubApplicantId])`

**Relations:**
- `User` (1:1 via `userId`)
- `KycVerification[]` (1:many)
- `BridgeCustomer?` (1:1 via `bridgeCustomer`)

#### `KycVerification` (tracks step completion per partner)

| Field | Type | Purpose |
|-------|------|---------|
| `id` | `String @id` | CUID primary key |
| `kycProfileId` | `String` | FK to KycProfile |
| `partnerId` | `String` | Logical partner key: `"bridge"`, `"ramp"`, etc. |
| `step` | `String` | `"questionnaire"`, `"id_verification"`, `"tos"`, `"proof_of_address"` |
| `status` | `String @default("not_started")` | `not_started`, `in_progress`, `pending_review`, `approved`, `rejected` |
| `metadata` | `Json?` | Step-specific data (e.g., `signedAgreementId` for TOS) |
| `rejectionReason` | `String?` | Human-readable rejection reason |
| `rejectionCount` | `Int @default(0)` | Retry tracking |
| `completedAt` | `DateTime?` | When step was approved |

**Indexes:** `@@unique([kycProfileId, partnerId, step])`, `@@index([kycProfileId])`, `@@index([partnerId])`, `@@index([status])`

### Modified Models

#### `User` — new relation field

```prisma
KycProfile KycProfile? // Modular KYC system
```

#### `BridgeCustomer` — new FK + relation

```prisma
kycProfileId String? @unique  // FK to modular KYC system (nullable for migration)
kycProfile   KycProfile? @relation(fields: [kycProfileId], references: [id])
```

The `kycProfileId` is **nullable** — existing `BridgeCustomer` records (from legacy Persona flow) continue to work. The `BridgeCustomer` still has its own `personaInquiryId`, `signedAgreementId`, `status` fields for backward compatibility.

### Migration Required

```sql
-- New tables
CREATE TABLE "KycProfile" (...);
CREATE TABLE "KycVerification" (...);

-- Modified tables
ALTER TABLE "User" ADD COLUMN IF NOT EXISTS -- no actual column, just Prisma relation
ALTER TABLE "BridgeCustomer" ADD COLUMN "kycProfileId" TEXT UNIQUE;
ALTER TABLE "BridgeCustomer" ADD CONSTRAINT "BridgeCustomer_kycProfileId_fkey"
  FOREIGN KEY ("kycProfileId") REFERENCES "KycProfile"("id");
```

**Run:** `npx prisma migrate dev --name add-modular-kyc-system`

---

## 3. Environment Variables

### New Variables Required

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `SUMSUB_APP_TOKEN` | **Yes** | — | Sumsub application token from Dashboard → Developer → App Tokens ([Sumsub docs: App tokens](https://docs.sumsub.com/docs/app-tokens)) |
| `SUMSUB_SECRET_KEY` | **Yes** | — | Sumsub secret key for HMAC-SHA256 API request signing ([Sumsub docs: Authentication](https://docs.sumsub.com/reference/authentication)) |
| `SUMSUB_WEBHOOK_SECRET` | **Yes** | — | Separate secret for webhook payload verification. Set in Sumsub Dashboard → Developer → Webhooks ([Sumsub docs: Webhook security](https://docs.sumsub.com/docs/webhook-manager)) |
| `SUMSUB_LEVEL_NAME` | No | `basic-kyc-level` | Verification level name configured in Sumsub Dashboard ([Sumsub docs: Verification levels](https://docs.sumsub.com/docs/verification-levels)) |
| `SUMSUB_BASE_URL` | No | `https://api.sumsub.com` | Sumsub API base URL |

### Existing Variables (unchanged)

| Variable | Used By |
|----------|---------|
| `BRIDGE_API_KEY` | Bridge customer creation, TOS link generation |
| `BRIDGE_API_URL` | Bridge API base URL |
| `BRIDGE_WEBHOOK_PUBLIC_KEY` | Bridge webhook signature verification |

### Add to `validation.schema.ts`

```typescript
// Sumsub Identity Verification
SUMSUB_APP_TOKEN: Joi.string().optional(),
SUMSUB_SECRET_KEY: Joi.string().optional(),
SUMSUB_WEBHOOK_SECRET: Joi.string().optional(),
SUMSUB_LEVEL_NAME: Joi.string().optional().default('basic-kyc-level'),
SUMSUB_BASE_URL: Joi.string().optional().default('https://api.sumsub.com'),
```

### Add to `.env.example`

```env
# Sumsub Identity Verification
SUMSUB_APP_TOKEN=your-sumsub-app-token
SUMSUB_SECRET_KEY=your-sumsub-secret-key
SUMSUB_WEBHOOK_SECRET=your-sumsub-webhook-secret
SUMSUB_LEVEL_NAME=basic-kyc-level
SUMSUB_BASE_URL=https://api.sumsub.com
```

---

## 4. Complete End-to-End Flow

### Happy Path: New User → Bridge Active

```
┌─────────────────────────────────────────────────────────────────┐
│                     FRONTEND (Mobile App)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. GET /kyc/status/bridge                                       │
│     → Returns checklist: questionnaire(not_started),             │
│       id_verification(not_started), tos(not_started)             │
│                                                                  │
│  2. User fills questionnaire form                                │
│     POST /kyc/questionnaire { firstName, lastName, ... }         │
│     → KycProfile upserted, questionnaire step → approved         │
│     → PartnerSubmissionHandler.checkAndSubmit (not all done yet)  │
│                                                                  │
│  3. GET /kyc/tos-link/bridge                                     │
│     → Returns { tosLink: "https://dashboard.bridge.xyz/..." }    │
│     User opens TOS WebView, accepts terms                        │
│     → WebView postMessage returns signed_agreement_id            │
│                                                                  │
│  4. POST /kyc/tos { signedAgreementId, partnerId: "bridge" }    │
│     → TOS step → approved, metadata.signedAgreementId saved      │
│     → PartnerSubmissionHandler.checkAndSubmit (not all done yet)  │
│                                                                  │
│  5. POST /kyc/sumsub/session                                     │
│     → Creates Sumsub applicant (or finds existing)               │
│     → Generates SDK access token (600s TTL)                      │
│     → Returns { token }                                          │
│                                                                  │
│  6. Initialize Sumsub Mobile SDK with token                      │
│     User takes selfie + uploads ID document                      │
│     SDK communicates directly with Sumsub servers                │
│     (No backend calls during this step)                          │
│                                                                  │
│  7. GET /kyc/status/bridge (poll)                                │
│     → Check if id_verification is approved yet                   │
│                                                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SUMSUB SERVERS                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Sumsub reviews applicant (automated + manual if needed)         │
│  Sends webhook: POST /api/v1/kyc/webhook/sumsub                 │
│  Payload:                                                        │
│  {                                                               │
│    "type": "applicantReviewed",                                  │
│    "applicantId": "5cb56e8e...",                                 │
│    "reviewResult": { "reviewAnswer": "GREEN" },                  │
│    "reviewStatus": "completed"                                   │
│  }                                                               │
│                                                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ANZO BACKEND                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SumsubWebhookController.handleWebhook()                         │
│    → SumsubWebhookGuard verifies HMAC-SHA256 signature           │
│    → SumsubKycHandler.handleApplicantApproved(applicantId)       │
│      → Fetches full applicant data from Sumsub API               │
│      → Updates KycProfile: sumsubReviewAnswer=GREEN              │
│      → Marks ID_VERIFICATION → approved (all partners)           │
│      → Marks PROOF_OF_ADDRESS → approved (all partners)          │
│                                                                  │
│  PartnerSubmissionHandler.checkAndSubmitAllPartners(userId)       │
│    → For Bridge: checks questionnaire✓ id_verification✓ tos✓     │
│    → All required steps done!                                    │
│                                                                  │
│  PartnerSubmissionHandler.submitToBridge()                        │
│    → Fetches user email, validates PII present                   │
│    → Fetches Sumsub applicant data (DOB, nationality, etc.)      │
│    → Downloads ID document images from Sumsub as base64          │
│    → Builds NormalizedKycData → buildBridgeCustomerPayload()     │
│    → POST /v0/customers to Bridge API                            │
│    → Upserts BridgeCustomer with real Bridge customer ID          │
│                                                                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BRIDGE.XYZ SERVERS                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Bridge receives customer + ID docs + signed_agreement_id        │
│  Bridge runs own KYC verification                                │
│  Sends webhook: POST /api/v1/bridge/webhook                      │
│  Event: customer.updated (kyc_status → approved)                 │
│                                                                  │
│  Existing BridgeKycHandler.handleKycApproval() runs:             │
│    → Creates Bridge wallet (Solana)                              │
│    → Creates USD virtual account                                 │
│    → Creates EUR virtual account                                 │
│    → BridgeCustomer.status → ACTIVE                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Timing Breakdown

| Step | Time | Actor |
|------|------|-------|
| Questionnaire submission | ~1s | User + Backend |
| TOS acceptance | ~10-30s | User (WebView) |
| Sumsub SDK verification | 1-5 min | User (camera + upload) |
| Sumsub automated review | 1-60 min | Sumsub |
| Auto-submission to Bridge | ~2-5s | Backend (webhook-triggered) |
| Bridge KYC review | 1-60 min | Bridge |
| Wallet + VA creation | ~3-5s | Backend (webhook-triggered) |

---

## 5. API Endpoints

All endpoints require JWT auth via `AuthGuard`. Base path: `/api/v1/kyc`

### GET `/kyc/status/:partnerId`

Returns the KYC checklist for a partner with step completion status.

**Request:** `GET /api/v1/kyc/status/bridge`

**Response:**
```json
{
  "partnerId": "bridge",
  "partnerName": "Bridge",
  "overallStatus": "in_progress",
  "steps": [
    {
      "step": "questionnaire",
      "title": "Questionnaire",
      "subtitle": "Answer a few questions about yourself",
      "required": true,
      "status": "approved",
      "completedAt": "2026-03-18T10:00:00.000Z"
    },
    {
      "step": "id_verification",
      "title": "ID document verification",
      "subtitle": "Verify your identity",
      "required": true,
      "status": "not_started",
      "completedAt": null
    },
    {
      "step": "tos",
      "title": "Terms and conditions",
      "subtitle": "Review and accept the terms and conditions",
      "required": true,
      "status": "approved",
      "completedAt": "2026-03-18T10:01:00.000Z"
    },
    {
      "step": "proof_of_address",
      "title": "Proof of address",
      "subtitle": "Verify your address",
      "required": false,
      "status": "not_started",
      "completedAt": null
    }
  ]
}
```

**overallStatus logic:**
- `"approved"` — all required steps approved
- `"rejected"` — any required step rejected
- `"in_progress"` — any step in_progress/pending_review, or at least one approved but not all
- `"not_started"` — no steps started

**Legacy compatibility:** If `BridgeCustomer.status === 'ACTIVE'`, returns all steps as `approved` immediately — no KycProfile lookup needed.

### GET `/kyc/questionnaire`

Returns saved questionnaire data for form pre-filling.

**Response (exists):**
```json
{
  "exists": true,
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "1990-01-01",
  "streetLine1": "123 Main St",
  "city": "New York",
  "state": "NY",
  "postalCode": "10001",
  "country": "USA",
  "phone": "+14155551234",
  "phoneCountryCode": "US",
  "occupation": "software_engineer",
  "annualSalary": "100000_149999",
  "accountPurpose": "personal_payments",
  "expectedMonthlyVolume": "5000_9999",
  "taxId": null,
  "taxIdCountry": null
}
```

**Response (no profile):** `{ "exists": false }`

### POST `/kyc/questionnaire`

Saves/updates questionnaire PII. Upserts `KycProfile`, marks questionnaire step as approved for all partners.

**Request:**
```json
{
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "1990-01-01",
  "streetLine1": "123 Main St",
  "city": "New York",
  "state": "NY",
  "postalCode": "10001",
  "country": "USA",
  "phone": "+14155551234",
  "phoneCountryCode": "US",
  "occupation": "software_engineer",
  "annualSalary": "100000_149999",
  "accountPurpose": "personal_payments",
  "expectedMonthlyVolume": "5000_9999"
}
```

**Response:** `{ "success": true, "stepStatus": "approved" }`

**Validation:** `class-validator` decorators on `SaveQuestionnaireDto`

### POST `/kyc/sumsub/session`

Creates or retrieves a Sumsub applicant, returns SDK access token.

**Request:**
```json
{
  "step": "id_verification"
}
```

`step` is optional. Can be `"id_verification"` or `"proof_of_address"` to select different Sumsub verification levels.

**Response:** `{ "token": "eyJhbGciOiJIUzI1NiIs..." }`

**Sumsub API calls made:**
1. `GET /resources/applicants/-;externalUserId={userId}/one` — check existing ([Sumsub docs](https://docs.sumsub.com/reference/get-applicant-data-via-externaluserid))
2. `POST /resources/applicants?levelName={level}` — create if needed ([Sumsub docs](https://docs.sumsub.com/reference/create-applicant))
3. `POST /resources/accessTokens/sdk?userId={userId}&levelName={level}&ttlInSecs=600` — token ([Sumsub docs](https://docs.sumsub.com/docs/reusable-kyc-via-sdk))

**Guards:**
- `ConflictException` if `sumsubReviewAnswer === 'GREEN'` (already verified)

### POST `/kyc/tos`

Saves TOS acceptance for a partner.

**Request:**
```json
{
  "signedAgreementId": "d536a227-06d3-4de1-acd3-8b5131730480",
  "partnerId": "bridge"
}
```

**Response:** `{ "success": true, "stepStatus": "approved" }`

### GET `/kyc/tos-link/:partnerId`

Returns the TOS URL for a partner.

**Response (Bridge):**
```json
{
  "partnerId": "bridge",
  "tosLink": "https://dashboard.bridge.xyz/accept-terms-of-service?..."
}
```

**Bridge API call:** `POST /customers/tos_links` ([Bridge docs](https://apidocs.bridge.xyz/api-reference/customers/create-a-tos-link))

### POST `/kyc/webhook/sumsub` (no auth, webhook-guarded)

Receives Sumsub webhook events. Protected by `SumsubWebhookGuard` (HMAC-SHA256).

**Handled events:**
- `applicantReviewed` with `reviewAnswer: "GREEN"` → approve ID verification
- `applicantReviewed` with `reviewAnswer: "RED"` → reject ID verification

---

## 6. Sumsub Integration Details

### Authentication: HMAC-SHA256 Request Signing

Per [Sumsub Authentication docs](https://docs.sumsub.com/reference/authentication), every API request is signed:

```
signature = HMAC-SHA256(secretKey, timestamp + method + pathWithQuery + body)
```

Headers sent:
- `X-App-Token`: Application token
- `X-App-Access-Ts`: Unix timestamp (seconds)
- `X-App-Access-Sig`: Hex-encoded HMAC signature

The `SumsubProvider` implements this via an Axios request interceptor.

### Webhook Signature Verification

Per [Sumsub Webhook Security docs](https://docs.sumsub.com/docs/webhook-manager), incoming webhooks include:

- `X-Payload-Digest`: HMAC hex digest of the raw body
- `X-Payload-Digest-Alg`: Algorithm identifier (e.g., `HMAC_SHA256_HEX`)

The `SumsubWebhookGuard` verifies using `crypto.timingSafeEqual` to prevent timing attacks.

**Important:** Requires `rawBody: true` in NestJS app config (already set in `main.ts` line 20).

**Current implementation note:** The guard hardcodes SHA256 and does not read `X-Payload-Digest-Alg` header. This should work for default Sumsub config but is not fully spec-compliant — see [Recommendations](#12-issues-risks--recommendations).

### Sumsub API Endpoints Used

| Endpoint | Purpose | Reference |
|----------|---------|-----------|
| `POST /resources/applicants?levelName=` | Create applicant | [Sumsub: Create applicant](https://docs.sumsub.com/reference/create-applicant) |
| `POST /resources/accessTokens/sdk?userId=&levelName=&ttlInSecs=600` | Generate SDK token | [Sumsub: Generate access token](https://docs.sumsub.com/reference/generate-access-token) |
| `GET /resources/applicants/{id}/one` | Get applicant data | [Sumsub: Get applicant data](https://docs.sumsub.com/reference/get-applicant-data) |
| `GET /resources/applicants/-;externalUserId={id}/one` | Find by external ID | [Sumsub: Get applicant by externalUserId](https://docs.sumsub.com/reference/get-applicant-data-via-externaluserid) |
| `GET /resources/applicants/{id}/info/idDoc` | Get ID documents | [Sumsub: Get ID verification results](https://docs.sumsub.com/reference/get-id-verification-results) |
| `GET /resources/inspections/{id}/resources/{imageId}` | Download document image | [Sumsub: Get document images](https://docs.sumsub.com/reference/get-document-images) |

### Sumsub Verification Level

Configure in Sumsub Dashboard → Verification Levels:
- Level name must match `SUMSUB_LEVEL_NAME` env var
- Should include: **Identity document** + **Selfie/Liveness** at minimum
- Optionally include: **Proof of address** if `PROOF_OF_ADDRESS` step is required

### Sumsub Webhook Events

Register webhook URL in Sumsub Dashboard → Developer → Webhooks:
- **URL:** `https://your-domain.com/api/v1/kyc/webhook/sumsub`
- **Events:** `applicantReviewed` (minimum required)
- **Secret:** Set as `SUMSUB_WEBHOOK_SECRET` env var
- **Algorithm:** HMAC_SHA256_HEX (default)

Webhook payload structure ([Sumsub docs: Webhook types](https://docs.sumsub.com/docs/user-verification-webhooks)):

```json
{
  "applicantId": "5cb56e8e0a975a35f333cb83",
  "inspectionId": "5cb56e8e0a975a35f333cb84",
  "correlationId": "req-a260b669-...",
  "externalUserId": "userId",
  "levelName": "basic-kyc-level",
  "type": "applicantReviewed",
  "reviewResult": {
    "reviewAnswer": "GREEN"
  },
  "reviewStatus": "completed",
  "createdAtMs": "2020-02-21 13:23:19.321"
}
```

Rejection payload includes additional fields:
```json
{
  "reviewResult": {
    "reviewAnswer": "RED",
    "rejectLabels": ["SCREENSHOTS", "UNSATISFACTORY_PHOTOS"],
    "reviewRejectType": "FINAL",
    "moderationComment": "We could not verify your profile..."
  }
}
```

---

## 7. Bridge Auto-Submission Flow

### NormalizedKycData Assembly

The `PartnerSubmissionHandler.submitToBridge()` assembles data from three sources:

| Data Field | Source | Fallback |
|------------|--------|----------|
| `firstName`, `lastName` | `KycProfile` (questionnaire) | — (required) |
| `email` | `User` model | — (required) |
| `birthDate` | `KycProfile.dateOfBirth` | `Sumsub applicant.fixedInfo.dob` |
| `phone` | `KycProfile.phone` | — |
| `nationality` | `KycProfile.country` (alpha-3) | `Sumsub applicant.fixedInfo.nationality` |
| `address` | `KycProfile` fields | — |
| `signedAgreementId` | `KycVerification.metadata` (TOS step) | — |
| `governmentId.type` | Sumsub `idDocType` → mapped | `national_id` |
| `governmentId.images` | Downloaded from Sumsub API as base64 | — |
| `taxId` | Sumsub `fixedInfo.tin` | `KycProfile.taxId` |
| `verifiedAt` | Sumsub `review.reviewDate` | — |

### Bridge Customer Payload

Built via `buildBridgeCustomerPayload()` in `bridge-payload.builder.ts`. Maps to Bridge's `POST /v0/customers` ([Bridge docs](https://apidocs.bridge.xyz/api-reference/customers/create-a-customer)):

```typescript
{
  type: 'individual',
  first_name: string,        // Required
  last_name: string,         // Required
  email: string,             // Required
  birth_date?: string,       // YYYY-MM-DD
  phone?: string,            // E.164 format
  nationality?: string,      // ISO alpha-3
  residential_address?: {
    street_line_1: string,
    city: string,
    subdivision?: string,    // ISO 3166-2 without country prefix
    postal_code: string,
    country: string,         // ISO alpha-3
  },
  signed_agreement_id?: string,  // From TOS step
  endorsements: ['base'],
  identifying_information?: [{
    type: string,            // passport, national_id, drivers_license, ssn, etc.
    issuing_country: string, // ISO alpha-3
    number?: string,
    image_front?: string,    // base64 data URI
    image_back?: string,     // base64 data URI
    expiration?: string,     // YYYY-MM-DD
  }],
  verified_govid_at?: string,   // ISO 8601 — Sumsub review date
  verified_selfie_at?: string,  // ISO 8601 — Sumsub review date
}
```

### Document Type Mapping

Sumsub `idDocType` → Bridge `identifying_information.type`:

| Sumsub | Bridge |
|--------|--------|
| `PASSPORT` | `passport` |
| `ID_CARD` | `national_id` |
| `DRIVERS` | `drivers_license` |
| `RESIDENCE_PERMIT` | `permanent_residency_id` |

---

## 8. Webhook Handling

### Sumsub Webhook Flow

```
Sumsub Server → POST /api/v1/kyc/webhook/sumsub
  → SumsubWebhookGuard.canActivate()
    → Reads X-Payload-Digest header
    → Reads request.rawBody (Buffer)
    → HMAC-SHA256(SUMSUB_WEBHOOK_SECRET, rawBody)
    → timingSafeEqual(computed, received)
  → SumsubWebhookController.handleWebhook()
    → If applicantReviewed + GREEN:
      → SumsubKycHandler.handleApplicantApproved(applicantId)
    → If applicantReviewed + RED:
      → SumsubKycHandler.handleApplicantRejected(applicantId, ...)
  → Returns { ok: true } (200)
```

**Error handling:** Errors are thrown (not swallowed) so Sumsub receives 5xx and retries with exponential backoff.

### Bridge Webhook Flow (unchanged)

```
Bridge Server → POST /api/v1/bridge/webhook
  → BridgeWebhookGuard (RSA-SHA256 signature)
  → customer.updated event → BridgeKycHandler.handleKycApproval()
  → Creates wallet + virtual accounts → ACTIVE
```

This flow is **completely untouched** by these PRs. The existing `BridgeKycHandler.handleKycApproval()` works the same way regardless of whether the customer was created via the old Persona flow or the new Sumsub flow.

---

## 9. Frontend Integration Guide

### Step 1: Check KYC Status

```
GET /api/v1/kyc/status/bridge
Authorization: Bearer <jwt>
```

Render a checklist UI from `steps[]`. Each step has `title`, `subtitle`, `required`, `status`.

### Step 2: Questionnaire Form

Display form with fields matching `SaveQuestionnaireDto`. Pre-fill from `GET /kyc/questionnaire`.

```
POST /api/v1/kyc/questionnaire
Authorization: Bearer <jwt>
Content-Type: application/json
{ firstName, lastName, dateOfBirth, ... }
```

### Step 3: TOS Acceptance

1. `GET /api/v1/kyc/tos-link/bridge` → get `tosLink` URL
2. Open `tosLink` in WebView
3. Listen for `postMessage` containing `signed_agreement_id`
4. `POST /api/v1/kyc/tos { signedAgreementId, partnerId: "bridge" }`

### Step 4: Sumsub SDK (ID Verification)

1. `POST /api/v1/kyc/sumsub/session` → get `token`
2. Initialize Sumsub Mobile SDK:

**React Native:**
```javascript
import SNSMobileSDK from '@nicklaus/react-native-sumsub-msdk';

const launchSumsub = (accessToken) => {
  const snsMobileSDK = SNSMobileSDK.init(accessToken, () => {
    // Token expiration handler — call POST /kyc/sumsub/session again
    return fetch('/api/v1/kyc/sumsub/session', { method: 'POST', ... })
      .then(r => r.json())
      .then(d => d.token);
  })
  .withHandlers({
    onStatusChanged: (event) => {
      // event.prevStatus, event.newStatus
      if (event.newStatus === 'Approved') {
        // Refresh KYC status
      }
    },
  })
  .build();

  snsMobileSDK.launch();
};
```

**iOS (Swift):**
```swift
let sdk = SNSMobileSDK(accessToken: token)
sdk.tokenExpirationHandler { onComplete in
    // Fetch new token from POST /kyc/sumsub/session
    onComplete(newToken)
}
sdk.onStatusDidChange { sdk, prevStatus in
    // Check sdk.status
}
sdk.present(from: viewController)
```

3. After SDK completes, poll `GET /kyc/status/bridge` until `overallStatus === "approved"`

### Step 5: Poll for Completion

After Sumsub SDK finishes, the backend auto-submits to Bridge (triggered by Sumsub webhook). Poll status:

```
GET /api/v1/kyc/status/bridge (every 5-10 seconds)
```

When `overallStatus === "approved"`, the existing Bridge status endpoint confirms activation:

```
GET /api/v1/bridge/kyc/status
→ { status: "ACTIVE", hasVirtualAccount: true }
```

---

## 10. Backward Compatibility Analysis

### Will anything break? No.

| Component | Impact | Reason |
|-----------|--------|--------|
| **Existing Persona flow** | None | `PersonaKycHandler`, `PersonaWebhookController`, `PersonaProvider` are untouched. Legacy users continue through `POST /bridge/kyc/persona/session` → Persona webhook → Bridge submission. |
| **Existing BridgeCustomer records** | None | New `kycProfileId` column is nullable. Existing records have `kycProfileId = null` and work fine. |
| **Bridge webhooks** | None | `BridgeWebhookController` and `BridgeKycHandler.handleKycApproval()` are untouched. They trigger on `customer.updated` regardless of how the customer was created. |
| **KYC status endpoint** | Enhanced | `GET /kyc/status/bridge` adds a legacy check: if `BridgeCustomer.status === 'ACTIVE'`, returns all steps as approved (short-circuits without needing a KycProfile). |
| **User model** | Safe | Only adds a Prisma relation field (`KycProfile KycProfile?`). No schema column change. |
| **App module** | Safe | `KycModule` is added to imports. No existing module removed. |
| **Bridge module** | Minimal | `BridgeProvider` is added to `exports[]` so `KycModule` can import it. No logic changes. |

### Two KYC Paths Can Coexist

```
Path A (Legacy):  BridgeController → PersonaKycHandler → PersonaProvider → Bridge
Path B (New):     KycController → SumsubKycHandler → SumsubProvider → PartnerSubmissionHandler → Bridge
```

Both paths ultimately create a `BridgeCustomer` record and submit to `POST /v0/customers`. Both rely on the same `customer.updated` Bridge webhook for post-approval setup.

---

## 11. Scalability & Complexity Analysis

### Time Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `getKycStatus` | O(S) | S = number of steps for partner (typically 3-4). One DB query with JOIN. |
| `saveQuestionnaire` | O(P) | P = number of partners. One upsert per partner that has a questionnaire step. Currently P=1. |
| `createSumsubSession` | O(1) | 2-3 Sumsub API calls (find + create + token). |
| `saveTos` | O(1) | One DB read + one upsert. |
| `handleApplicantApproved` | O(P × S) | For each partner, upserts verification steps. Currently P=1, S=2 (ID + PoA). |
| `checkAndSubmitAllPartners` | O(P) | Iterates partners, checks step completion. One DB query per partner. |
| `submitToBridge` | O(1) | Multiple Sumsub API calls (applicant + docs + images) + one Bridge API call. |

### Space Complexity

| Model | Rows per User | Notes |
|-------|--------------|-------|
| `KycProfile` | 1 | One per user (unique userId) |
| `KycVerification` | P × S | ~4 rows per partner (one per step). Currently 4 for Bridge. |
| `BridgeCustomer` | 1 | Unchanged |

**With 100K users and 3 partners:** 100K KycProfile rows + ~1.2M KycVerification rows. Well within PostgreSQL comfort zone with proper indexing (all declared).

### Database Index Coverage

All query patterns are covered:
- `KycProfile.userId` — unique index (find by user)
- `KycProfile.sumsubApplicantId` — indexed (webhook lookup)
- `KycVerification.kycProfileId` — indexed (JOIN from profile)
- `KycVerification.partnerId` — indexed (filter by partner)
- `KycVerification.status` — indexed (filter by status)
- `KycVerification.(kycProfileId, partnerId, step)` — unique compound (upsert target)

### Network Calls per Flow

| Step | Backend→External | Notes |
|------|-----------------|-------|
| Questionnaire | 0 | Pure DB operation |
| TOS link | 1 (Bridge) | `POST /customers/tos_links` |
| TOS save | 0 | Pure DB operation |
| Sumsub session | 1-3 (Sumsub) | Find/create applicant + generate token |
| Sumsub webhook (GREEN) | 3-5 (Sumsub) + 1 (Bridge) | Get applicant + get docs + download images + create customer |

---

## 12. Issues, Risks & Recommendations

### Issues Found in the PR Code

#### 1. Webhook Guard: Missing `X-Payload-Digest-Alg` Check (Medium)

The `SumsubWebhookGuard` assumes SHA256 but does not read the `X-Payload-Digest-Alg` header. Per [Sumsub webhook docs](https://docs.sumsub.com/docs/webhook-manager), the algorithm can be SHA1, SHA256, or SHA512.

**Risk:** If Sumsub changes the default algorithm, webhook verification will silently fail.

**Fix:** Read the `X-Payload-Digest-Alg` header and select algorithm dynamically (as shown in the Sumsub JavaScript example).

#### 2. KycProfile Created with Empty Required Fields (Low)

Both `saveTos` and `createSession` can create a KycProfile with empty strings for required fields (`firstName: ''`, etc.) when the questionnaire hasn't been filled yet.

**Risk:** If auto-submission triggers before questionnaire is done (shouldn't happen due to step checks), Bridge receives empty PII.

**Mitigated by:** `PartnerSubmissionHandler.submitToBridge()` validates `profile.firstName && profile.lastName` before submitting.

#### 3. PROOF_OF_ADDRESS Auto-Approved on ID Verification (Low)

`handleApplicantApproved` marks **both** `ID_VERIFICATION` and `PROOF_OF_ADDRESS` as approved when Sumsub returns GREEN, regardless of whether the Sumsub level actually includes address verification.

**Risk:** For Bridge, `PROOF_OF_ADDRESS` is `required: false`, so this doesn't affect the happy path. But if a future partner requires it as a separate step, this would short-circuit it.

**Fix:** Check the Sumsub verification level configuration or the applicant's `requiredIdDocsStatus` to determine which steps were actually verified.

#### 4. Missing `BridgeProvider` Export Change in PR #202 (Fixed in PR #203)

PR #202 (`feature/modular-kyc-system`) has `KycModule` importing `BridgeModule`, but PR #202's `bridge.module.ts` doesn't export `BridgeProvider`. This is fixed in PR #203 which changes `exports: [BridgeService]` to `exports: [BridgeService, BridgeProvider]`.

**Risk:** PR #202 alone would compile but `getTosLink` would fail at runtime since it calls `bridgeProvider.createTosLink()` through the service. However, PR #202's `getTosLink` was a placeholder (`return { tosEndpoint: '...' }`), so this is fine.

#### 5. Dynamic Import in `markStepForAllPartners` (Low)

`SumsubKycHandler.markStepForAllPartners()` uses dynamic `await import(...)` to avoid circular dependencies:

```typescript
const { PARTNER_KYC_CONFIGS } = await import('../config/partner-steps.config');
```

**Risk:** Unnecessary complexity. `PARTNER_KYC_CONFIGS` is already imported at the top of the file via the barrel export. This dynamic import is not needed.

#### 6. No `@ApiTags` / Swagger Decorators (Minor)

The `KycController` doesn't have Swagger/OpenAPI decorators. Not a functional issue, but reduces API documentation quality.

### Recommendations

1. **Add `SUMSUB_*` vars to `validation.schema.ts`** — currently missing from the validation schema.
2. **Add retry/circuit-breaker for Sumsub image downloads** — downloading 2 images (front+back) as base64 during webhook processing adds latency. Consider async processing or a queue.
3. **Consider background processing for `submitToBridge`** — the Sumsub webhook handler downloads images and calls Bridge API synchronously. If any step takes >30s, Sumsub may retry the webhook.
4. **Add the `X-Payload-Digest-Alg` check** to `SumsubWebhookGuard`.
5. **Test with Sumsub Sandbox** — Sumsub provides [test applicants](https://docs.sumsub.com/docs/test-in-sandbox) for integration testing.

---

## 13. Environment Setup Checklist

### Sumsub Dashboard Setup

1. **Create App Token** at Dashboard → Developer → App Tokens
   - Copy `App Token` → `SUMSUB_APP_TOKEN`
   - Copy `Secret Key` → `SUMSUB_SECRET_KEY`

2. **Configure Verification Level** at Dashboard → Verification Levels
   - Create level named to match `SUMSUB_LEVEL_NAME` (default: `basic-kyc-level`)
   - Add steps: Identity Document + Selfie/Liveness
   - Optionally add: Proof of Address

3. **Configure Webhook** at Dashboard → Developer → Webhooks
   - URL: `https://{your-domain}/api/v1/kyc/webhook/sumsub`
   - Secret: generate and set as `SUMSUB_WEBHOOK_SECRET`
   - Events: `applicantReviewed`
   - Algorithm: HMAC_SHA256_HEX

4. **Test in Sandbox** — Use [Sumsub test documents](https://docs.sumsub.com/docs/test-in-sandbox) before going live.

### Backend `.env`

```env
# Required for Sumsub integration
SUMSUB_APP_TOKEN=sbx:xxxxxxxxxxxxxxxxxxxx
SUMSUB_SECRET_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SUMSUB_WEBHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SUMSUB_LEVEL_NAME=basic-kyc-level
SUMSUB_BASE_URL=https://api.sumsub.com

# Already required (Bridge)
BRIDGE_API_KEY=xxxxx
BRIDGE_API_URL=https://api.bridge.xyz/v0
BRIDGE_WEBHOOK_PUBLIC_KEY=-----BEGIN PUBLIC KEY-----...
```

### Database Migration

```bash
npx prisma migrate dev --name add-modular-kyc-system
npx prisma generate
```

### NestJS rawBody Requirement

Already configured in `main.ts`:
```typescript
const app = await NestFactory.create(AppModule, { rawBody: true });
```

This is required for both Sumsub and Persona webhook signature verification.

---

## Summary

The implementation is **architecturally sound** and follows a clean separation of concerns:

- **Config-driven** partner steps (easy to add partners)
- **Provider-agnostic** PII storage (KycProfile)
- **Auto-submission** on step completion (no manual trigger)
- **Full backward compatibility** with existing Persona/Bridge flows
- **Proper webhook security** (HMAC-SHA256 with timing-safe comparison)

The main areas for improvement are minor: adding the digest algorithm header check, adding env validation, and considering async processing for image downloads during webhook handling. Nothing in these PRs breaks existing functionality.
