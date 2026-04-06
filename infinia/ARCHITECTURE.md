# Architecture (simple explanation)

This document explains the Infinia integration **without** assuming you already know Infinia’s product vocabulary.

---

## What problem does this solve?

End users hold **USDC** in the app (funded via your existing wallet / Turnkey flows). For **Mexico** and **Brazil**, you need **fiat** delivered to a local bank account (**CLABE** or **PIX**). **Infinia** is the banking/payments partner that:

- Holds **virtual accounts** per user (crypto + fiat buckets),
- Runs **FX** between USDC and MXN/BRL inside their system,
- Initiates **local payouts** to the recipient’s bank details.

Anzo’s backend is the **orchestrator**: it talks to Infinia’s REST APIs, stores IDs and status in **Postgres**, and exposes **your** REST API to the mobile/web client.

---

## The three “worlds”

Think of three separate systems:

1. **Your client app** — talks only to Anzo with a **JWT** (or dev headers in non-production).
2. **Anzo backend** — NestJS, Prisma, validates requests, calls Infinia, receives webhooks.
3. **Infinia** — external SaaS; sandbox at `https://app2test.infiniaweb.com/infinia_api`, production at `https://app.infiniaweb.com/infinia_api` (see [EXTERNAL_DOCUMENTATION.md](./EXTERNAL_DOCUMENTATION.md)).

Data crosses the boundary in two directions:

- **Anzo → Infinia:** HTTP **Basic Auth** (platform username/password) and usually header **`x-company-id`** set to the user’s **child company** id at Infinia.
- **Infinia → Anzo:** HTTP **POST** to your **`/api/v1/infinia/webhook/...`** routes (no JWT). Security is described in [FLOWS.md](./FLOWS.md#webhooks-how-infinia-calls-back) and in code (`InfiniaWebhookGuard`).

---

## Main building blocks inside Anzo

### 1. Controller (user-facing API)

`infinia.controller.ts` groups routes under **`/api/v1/infinia`**. Almost all routes use **`AuthGuard`** so `req.user.userId` is the database user id.

Several routes also use **`InfiniaOwnerRequiredGuard`**: in **production** it requires a persisted **`infiniaOwnerId`** (Sumsub-linked owner succeeded); in other non-dev environments it may require at least **`infiniaCompanyId`**. **Development** skips this guard for faster local work.

### 2. Service (thin facade + webhooks)

`infinia.service.ts` forwards normal operations to **handlers**. It also implements **`handleTransferWebhook`** and **`handlePayoutWebhook`**, because webhooks need **Prisma** updates and coordination with payout logic.

### 3. Handlers (business logic)

| Handler | Responsibility (plain words) |
|---------|------------------------------|
| **Onboarding** | Create Infinia **company**, enable **products**, optionally create **account owner** via Sumsub, create **USDC_BASE / MXN / BRL** accounts, save ids on `User` + `InfiniaAccount`. |
| **FX quote** | Call Infinia internal-transfer **quote** API; apply configured **markup**; stamp quotes with an `external_id` prefix so only the owning user can reuse a quote. |
| **Bank validation** | Optional CLABE/PIX check through Infinia before payout. |
| **Payout** | Create **internal transfer** (USDC → fiat account), save `InfiniaTransaction`, register **callback URL** for transfer completion; when complete, create **payout** and register **payout callback**. |

### 4. Provider (HTTP client)

`infinia.provider.ts` centralizes Axios clients for Infinia’s **onboarding**, **accounts**, **payouts v2**, and **bank validation** paths. It attaches Basic auth and `x-company-id` per call.

### 5. Database (Prisma)

- **`User`**: `infiniaCompanyId`, `infiniaOwnerId` — which Infinia company and optional KYC owner belong to this app user.
- **`InfiniaAccount`**: one row per currency bucket (e.g. USDC_BASE, MXN, BRL) with Infinia’s numeric **account id**.
- **`InfiniaTransaction`**: one **off-ramp** attempt: amounts, FX, internal transfer id, payout id, **`payoutOriginId`** for payout webhooks, status, errors, metadata (can include webhook history).

---

## How this sits in the wider app

- **`AppModule`** imports **`InfiniaModule`** (`src/app.module.ts`).
- **KYC**: onboarding uses **`SumsubProvider.generateShareToken`** from the existing KYC module; see [EXTERNAL_DOCUMENTATION.md](./EXTERNAL_DOCUMENTATION.md) for Sumsub’s official share-token doc.
- **Config**: all Infinia-related env vars are validated in **`validation.schema.ts`** and exposed under **`configuration.ts`** → `infinia.*`.

---

## Design choices worth knowing (review angle)

1. **Child company per user** — isolates balances and compliance scope at Infinia; your backend always passes that company id.
2. **Quote ownership** — `external_id` includes the Anzo `userId` prefix so one user cannot execute another user’s quote.
3. **Webhook trust** — Infinia’s **platform** webhook doc describes **`X-Infinia-Signature`** over the body ([Webhooks](https://docs.infiniaweb.com/reference/webhooks)). This integration primarily secures **per-request callback URLs** with a **`?token=`** HMAC because the internal-transfer / payout APIs do not expose custom signing headers in our client; see guard and `InfiniaPayoutHandler` for details. Worth confirming with Infinia whether body signatures also arrive on these callbacks.
4. **Localhost** — Infinia is said to block **localhost** in callback URLs (SSRF/WAF). The payout handler **omits** callbacks in local dev; sandbox **`sync-transfer`** exists to advance state manually.

---

## Diagram (logical)

```
[ Mobile / Web app ]
        |  JWT
        v
[ Anzo: InfiniaController ]
        |
        v
[ InfiniaService ] -----> [ Handlers ] -----> [ InfiniaProvider ] ----HTTPS Basic----> [ Infinia API ]
        |                                                      ^
        |                                                      | x-company-id
        v                                                      |
[ Prisma / PostgreSQL ]                                        |
        ^                                                      |
        |                                                      |
        +------ [ WebhookController ] <----- POST callback ---+
                    (InfiniaWebhookGuard)
```

For sequence-style steps, see [FLOWS.md](./FLOWS.md).
