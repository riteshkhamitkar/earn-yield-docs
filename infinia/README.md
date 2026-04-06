# Infinia LATAM integration — reviewer guide

This folder is for **code review and onboarding**: plain-language architecture, flows, and pointers to official vendor docs. It does **not** replace the exhaustive API reference.

| Document | Purpose |
|----------|---------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | What was built, how pieces connect, in simple terms |
| [FLOWS.md](./FLOWS.md) | Step-by-step user and system flows (onboarding, off-ramp, webhooks) |
| [EXTERNAL_DOCUMENTATION.md](./EXTERNAL_DOCUMENTATION.md) | **Real links** to Infinia, Sumsub, and related official docs |
| [REVIEW_CHECKLIST.md](./REVIEW_CHECKLIST.md) | What to verify when reviewing another developer’s implementation |

**Full API reference (endpoints, request/response shapes, curl):** [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md)

---

## One-paragraph summary

Anzo’s backend adds an **Infinia** module so each user can get a **child company** at Infinia with **virtual accounts** (USDC on Base, plus MXN and BRL). Users deposit USDC, request an **FX quote**, then **payout** to a Mexican **CLABE** or Brazilian **PIX** destination. **KYC** for the Infinia “account owner” uses **Sumsub** (share token generated server-side). **Infinia** calls our **webhook** URLs when internal transfers and payouts change state; we persist state in **PostgreSQL** via **Prisma**. Sandbox-only helpers exist for deposits and for **syncing** transfers when webhooks cannot reach localhost.

---

## Where the code lives

| Area | Path |
|------|------|
| Nest module | `src/modules/infinia/infinia.module.ts` |
| HTTP API (JWT) | `src/modules/infinia/infinia.controller.ts` |
| Webhooks (no JWT) | `src/modules/infinia/webhooks/infinia-webhook.controller.ts` |
| Facade + webhook orchestration | `src/modules/infinia/infinia.service.ts` |
| HTTP client to Infinia | `src/modules/infinia/infinia.provider.ts` |
| Business logic | `src/modules/infinia/handlers/*.handler.ts` |
| Webhook auth | `src/modules/infinia/guards/infinia-webhook.guard.ts` |
| Production guard (owner / company) | `src/modules/infinia/guards/infinia-owner-required.guard.ts` |
| Config | `src/config/configuration.ts` (`infinia.*`), `src/config/validation.schema.ts` |
| Prisma models + migrations | `prisma/schema.prisma`, `prisma/migrations/*infinia*` |
| Sumsub share token | `src/modules/kyc/providers/sumsub.provider.ts` (`generateShareToken`) |

Global API prefix: **`/api/v1`** (see `src/main.ts`).

---

## Suggested reading order

1. [ARCHITECTURE.md](./ARCHITECTURE.md) — mental model  
2. [FLOWS.md](./FLOWS.md) — timing and async behavior  
3. [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md) — contract details  
4. [EXTERNAL_DOCUMENTATION.md](./EXTERNAL_DOCUMENTATION.md) — verify against vendors  
5. [REVIEW_CHECKLIST.md](./REVIEW_CHECKLIST.md) — sign-off items  
