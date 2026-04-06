# Review checklist (another developer’s Infinia implementation)

Use this when you are **reviewing** the feature before merge or release. Check boxes as you complete each area.

---

## Security & secrets

- [ ] **No secrets in git** — `INFINIA_USERNAME`, `INFINIA_PASSWORD`, `INFINIA_WEBHOOK_SECRET` only in env / secret manager.
- [ ] **Webhook endpoints** are not behind JWT but require **`?token=`** HMAC or legacy secret (`InfiniaWebhookGuard`).
- [ ] **Production** cannot rely on `X-Dev-User-Id` (`AuthGuard` behavior — verify in `src/common/guards/auth.guard.ts`).
- [ ] **Quote hijacking** — `getQuoteById` / payout path enforce `external_id` prefix with `userId`.
- [ ] **Amount tampering** — payout `sourceAmount` vs quote tolerance enforced in payout handler.
- [ ] **Shared sandbox company** — `getValidationResult` blocks cross-user leakage when multiple users share `infiniaCompanyId` (intentional guard).

---

## Data & migrations

- [ ] Prisma **migrations** for `User` columns and `InfiniaAccount` / `InfiniaTransaction` are applied in staging before testing.
- [ ] **`npx prisma generate`** run in CI so client types match schema.
- [ ] **Unique constraints** understood: e.g. `payoutOriginId`, `payoutId` — no double-insert races without handling.

---

## Correctness vs Infinia API

- [ ] Internal transfer and payout paths match current Infinia reference ([EXTERNAL_DOCUMENTATION.md](./EXTERNAL_DOCUMENTATION.md)).
- [ ] **Sandbox amounts** for payout outcomes align with [Create Payout](https://docs.infiniaweb.com/reference/v2_create_payout__post) examples.
- [ ] **`x-company-id`** sent on child-company-scoped calls (provider).
- [ ] **Idempotency keys** short enough for sandbox (~40 char issue documented in onboarding handler).

---

## Async / reliability

- [ ] Transfer **COMPLETED** triggers payout exactly once (atomic update / idempotency in `triggerPayoutForTransaction` and webhook handler).
- [ ] **Replayed webhooks** do not corrupt terminal state (see `handlePayoutWebhook` / `handleTransferWebhook` logs and guards).
- [ ] **Refund** path: `COMPLETED` → `REFUNDED` / `PARTIALLY_REFUNDED` still processed (service allows specific transitions).

---

## Operational

- [ ] **`APP_URL`** correct in each environment (callback URLs built from it).
- [ ] **`INFINIA_SANDBOX_ENABLED`** only `true` where intended (never accidental in prod).
- [ ] **`SUMSUB_INFINIA_FOR_CLIENT_ID`** set for non-dev onboarding if you require full owner KYC.
- [ ] Logs **redact** callback URLs where implemented (`infinia.provider` redaction helper — spot-check).

---

## Documentation

- [ ] [../INFINIA_LATAM_OFFRAMP.md](../INFINIA_LATAM_OFFRAMP.md) matches current routes and webhook auth (**`?token=`** + legacy).
- [ ] This reviewer pack ([README.md](./README.md)) still points to the right files after refactors.

---

## Tests & build

- [ ] `npm run build` passes.
- [ ] Any **e2e** or integration tests for Infinia (if added) are not hitting prod with real credentials.

---

## Sign-off

| Reviewer | Date | Notes |
|----------|------|-------|
| | | |
