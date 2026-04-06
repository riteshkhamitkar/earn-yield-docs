# External documentation (official links)

Use these **vendor sources** when reviewing correctness, security, or sandbox behavior. Links are to **official** documentation sites.

---

## Infinia

| Topic | URL | Why it matters for this repo |
|-------|-----|-------------------------------|
| Docs home / navigation | https://docs.infiniaweb.com/ | Entry point for guides and API reference. |
| **Webhooks** (signing, retries, idempotency, IP allowlists) | https://docs.infiniaweb.com/reference/webhooks | Describes **`X-Infinia-Signature`**, **`X-Idempotency-Key`**, retry policy. Compare with our **`?token=`** callback approach on transfer/payout URLs. |
| **Create Payout** (Payouts v2) | https://docs.infiniaweb.com/reference/v2_create_payout__post | **HTTP Basic** auth, **`x-company-id`**, request schema; **sandbox test amounts** (e.g. 99997/99996 vs 99999/99998) documented in the operation description. |
| **Get Payout** (by id) | https://docs.infiniaweb.com/reference/v2_get_payout__payout_id___get | Polling payout status (we refresh from API in places). |
| Product / marketing site | https://www.infiniaweb.com/en | Context only; not API contract. |

**Base URLs (from our config defaults and integration doc):**

- Sandbox API: `https://app2test.infiniaweb.com/infinia_api`
- Production API: `https://app.infiniaweb.com/infinia_api`

*(Confirm with Infinia if your tenant uses a different path.)*

---

## Sumsub

| Topic | URL | Why it matters for this repo |
|-------|-----|-------------------------------|
| **Generate share token** | https://docs.sumsub.com/reference/generate-share-token | **`POST /resources/accessTokens/shareToken`** with `applicantId`, `forClientId`, optional `ttlInSecs`. Matches `SumsubProvider.generateShareToken` used for Infinia **account owner** creation. |
| Reusable KYC (concept) | https://docs.sumsub.com/sumsub/docs/reusable-kyc | Background on sharing verified applicants with partners. |
| Copy applicant (concept) | https://docs.sumsub.com/sumsub/docs/copy-applicant | Related sharing model; useful if Infinia’s integration is described in those terms. |

---

## Anzo backend (internal)

| Topic | Path |
|-------|------|
| Full Infinia API reference (this repo) | [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md) |
| KYC / Sumsub architecture (if present) | [../KYC_SUMSUB_ARCHITECTURE.md](../KYC_SUMSUB_ARCHITECTURE.md), [../MODULAR_KYC_SYSTEM.md](../MODULAR_KYC_SYSTEM.md) |

---

## Optional / ecosystem (not Infinia-specific)

| Topic | URL | Note |
|-------|-----|------|
| NestJS | https://docs.nestjs.com/ | Framework; guards, modules, Fastify adapter. |
| Prisma | https://www.prisma.io/docs | Schema, migrations, client. |
| class-validator | https://github.com/typestack/class-validator | DTO validation on controllers. |

These are **not** required to approve the Infinia integration but help if you are new to the stack.

---

## Version note

Vendor docs and API behavior change. If something in [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md) disagrees with the live Infinia reference, **trust the vendor doc** and file a ticket to update our code or docs.
