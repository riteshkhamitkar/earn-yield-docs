# SOL Sponsorship Policy Management — Design

**Status:** Approved (2026-07-06)
**Author:** Brainstormed with product owner
**Implements:** Turnkey per-sub-org policy enforcement for sponsored Solana transactions, plus backend eligibility gate and rent-subsidy cap.

---

## Problem

Fake accounts drain the daily SOL sponsorship budget by exploiting two automatic server-paid sponsorships that fire on every Solana-involving swap/transfer quote, with no per-user gating:

1. **`SolanaDirectTransferService.broadcastWithSponsor()`** — invokes Turnkey Gas Station via `sol_send_transaction` with `sponsor: true`. Turnkey pre-funds the user's signer for rent-exempt amounts (~0.007 SOL ≈ $1) when ATA creation is needed. This is throttled by the Turnkey dashboard's per-sub-org cap, currently set to **$1/sub-org/day** and **$300/org/day**.
2. **`SolanaDirectTransferService.subsidizeSolRent()`** — sends up to 0.007 SOL directly from the server paymaster wallet (`SOLANA_PAYMASTER_ADDRESS`) to the user. This **bypasses Turnkey entirely**; the dashboard cap does not see or throttle it.

The current org-wide daily budget is $300. A fake account that initiates Solana activity triggers ~$1 of sponsorship immediately. 300 fake accounts → daily budget exhausted. The only existing defenses are reactive:

- `scripts/find-suspicious-sponsorship-users.ts` (manual, post-hoc).
- `MicroSolanaSwapAbuseGuardService` (currently **disabled** via `MICRO_SOLANA_SWAP_AUTO_BLOCK_ENABLED=false`), and only covers Relay swaps.
- Manual admin user blocking via `POST /admin/users/block`.

There is no proactive per-user eligibility check and no enforcement at the Turnkey signing layer.

### Secondary vector: rent-extraction via `CloseAccount`

A separate abuse path documented in [Turnkey's Solana Rent Sponsorship guide](https://docs.turnkey.com/features/networks/solana-rent-refunds): when Turnkey pre-funds a user's signer for rent (ATA creation), the rent belongs to the signer. If the user later closes that account, the rent refund flows to the signer, not back to the sponsor. In sponsored flows this is a direct leak — the sponsor paid the rent, the user pockets the refund.

The existing code already strips SPL Token `CloseAccount` instructions in `SolanaDirectTransferService.broadcastWithSponsor` (discriminator `9`) before submission. This is a **server-side** mitigation — effective only as long as every sponsored call site routes through that filter. A user who can export or independently control their signer key (which Turnkey permits) can submit their own `CloseAccount` tx outside our backend, bypassing the filter entirely. This makes a Turnkey-policy-level deny the stronger mitigation.

## Goals

1. **Block fake-account abuse** of both sponsorship paths (`broadcastWithSponsor` and `subsidizeSolRent`).
2. **Use Turnkey's policy engine as a tamper-proof enforcement layer** at the sub-org level, so backend bugs, refactors, or a compromised API key cannot bypass the gate.
3. **Preserve the existing $1/sub-org/day dashboard cap** as a hard ceiling on top of any code-level controls.
4. **Minimal implementation surface** — a small number of focused services and one-line guards at each call site.
5. **KYC as the primary trust signal**, with a tight pre-KYC allowance (3 sends / $1 lifetime) for first-time users, transitioning to $1/day post-KYC.
6. **Backfill existing users** based on their current KYC status at rollout.
7. **Promote rent-extraction mitigation to the policy layer** so it survives users independently controlling their signer keys.

## Non-Goals (YAGNI)

- No changes to the 0.007 SOL rent subsidy amount.
- No changes to the Turnkey dashboard caps ($1/sub-org/day, $300/org/day remain).
- No EVM sponsorship gating (the abuse is Solana-specific; EVM flows are unaffected).
- No new admin endpoints (existing `/admin/users/block` covers manual action).
- No new analytics/alerting dashboard (existing `find-suspicious-sponsorship-users.ts` script suffices).
- No Turnkey policy attached to the `subsidizeSolRent` path itself — Turnkey cannot observe that path, so a policy would not help. A DB counter + KYC check is the correct tool for that path.

## Turnkey Platform Constraints (verified from docs)

These are hard facts about what the Turnkey API does and does not expose. They shape every design decision below.

1. **Per-sub-org spend limits are dashboard-only — there is NO API to set them programmatically.** Per the [transaction-management docs](https://docs.turnkey.com/features/transaction-management): *"You can set limit values and time intervals through the dashboard."* The only related API is [`get_gas_usage`](https://docs.turnkey.com/api-reference/queries/get-gas-usage), which is read-only (returns `usageUsd`, `windowLimitUsd`, `windowDurationMinutes`).
   - **Implication:** we cannot tighten a fresh user's sub-org limit at signup via API. The dashboard's $1/sub-org/day is the only Turnkey-enforced per-user ceiling.
   - **Workaround:** the 3-send / $1-lifetime pre-KYC quota must be enforced by our DB counter (L1). The Turnkey policy (L2) gates *whether* any sponsorship is allowed at all.

2. **`EFFECT_DENY` always wins; `EFFECT_ALLOW` requires at least one matching policy or root quorum.** Per the [policies overview](https://docs.turnkey.com/features/policies/overview). Evaluation is non-short-circuiting — any erroring clause fails the whole policy, so conditions should be kept simple and split into separate policies when in doubt.

3. **The policy DSL exposes activity parameters via `activity.params.*`.** Per the [policy language page](https://docs.turnkey.com/features/policies/language), `activity.params` is the canonical way to access an activity's parameters. The `sol_send_transaction` activity accepts a `sponsor` boolean parameter (per the [broadcast SVM transaction docs](https://docs.turnkey.com/api-reference/activities/broadcast-svm-transaction)), so the condition `activity.params.sponsor == true` is **structurally valid**. However, no canonical Turnkey example uses this exact condition — every sponsored-policy example in the docs uses the `solana.tx.*` namespace (instruction content). This is treated as a verification step before implementation (see Open Questions).

4. **Solana policy namespace (`solana.tx.*`) is well-documented** for instruction-level rules: `solana.tx.instructions`, `solana.tx.transfers`, `solana.tx.spl_transfers`, `solana.tx.program_keys`, `solana.tx.address_table_lookups`. Rent-leakage deny policies can be written directly against `solana.tx.instructions` (see the rent-extraction section below).

5. **Policies are scoped to a single organization.** Since each user lives in their own sub-org, a policy targeting a user must be created with that user's `subOrgId` as the `organizationId` parameter. There is no cross-org policy inheritance.

6. **Gas Sponsorship is an Enterprise-plan feature.** Your org already has it enabled (otherwise `broadcastWithSponsor` would be failing), so this is not a blocker — just noting for completeness.

## Design

### Architecture: defense in depth (3 layers)

```
┌─────────────────────────────────────────────────────────────┐
│  L1 — Backend Gate (smart, stateful)                        │
│  SponsorshipEligibilityService.assertEligible(userId, kind) │
│  • Knows KYC status, quota history                          │
│  • Pre-KYC send: 3 free / $1 lifetime, tracked in DB        │
│  • Post-KYC send: rely on Turnkey dashboard $1/day cap      │
│  • rent_subsidy: KYC required AND ≤1/day                    │
│  • Pre-flight: get_gas_usage() for friendly error UX        │
│  • Throws SPONSORSHIP_NOT_AVAILABLE with actionable reason  │
└─────────────────────────────────────────────────────────────┘
                          ↓ (passes)
┌─────────────────────────────────────────────────────────────┐
│  L2 — Turnkey Policies (tamper-proof, stateless)            │
│  SolanaSponsorshipPolicyService attaches two policies:      │
│  • DENY sponsored SOL tx (lifted on KYC approval)           │
│  • DENY CloseAccount + ATA-create lifecycle (always-on,     │
│    survives users controlling their own signer key)         │
└─────────────────────────────────────────────────────────────┘
                          ↓ (Turnkey evaluates)
┌─────────────────────────────────────────────────────────────┐
│  L3 — Dashboard Cap (hard ceiling, already configured)      │
│  $1/sub-org/day, $300/org/day (kept as-is)                  │
└─────────────────────────────────────────────────────────────┘
```

**Rationale for three layers:** each compensates for the other's blind spot.

- The **policy engine** is tamper-proof but cannot express KYC status (Turnkey doesn't know about Sumsub).
- The **backend gate** is smart (knows KYC, history, can give nuanced errors) but fragile (a refactor can forget the check).
- The **dashboard cap** is a hard ceiling but cannot express nuanced rules.

### Data Model Changes

Two new Prisma models. `User` gets one reverse relation added per model.

```prisma
/// Runtime counter for sponsorship eligibility checks (L1).
/// Loaded on every sponsored-quote path. One row per user, lazily created.
model SolanaSponsorshipQuota {
  id                   String   @id @default(cuid())
  userId               String   @unique

  /// Pre-KYC sponsored send count. Incremented on each allowed pre-KYC send.
  /// Reset is intentionally NOT done on KYC approval — post-KYC sends are
  /// gated by the Turnkey dashboard cap, not this counter.
  preKycSendsUsed      Int      @default(0)

  /// Cumulative USD value of pre-KYC sponsored sends (estimated at quote
  /// time). Pre-KYC cap is "3 sends OR $1 lifetime, whichever hits first".
  /// Post-KYC this field stops mattering (dashboard cap takes over) but is
  /// kept for audit.
  preKycSpendUsdCents  Int      @default(0)

  /// Per-day counter for subsidizeSolRent (the direct paymaster path that
  /// bypasses Turnkey). Reset when rentSubsidiesResetAt passes 24h.
  rentSubsidiesToday       Int       @default(0)
  rentSubsidiesResetAt     DateTime?

  lastSponsoredSendAt  DateTime?
  createdAt            DateTime @default(now())
  updatedAt            DateTime @updatedAt

  user                 User     @relation(fields: [userId], references: [id])

  @@index([userId])
}

/// Audit trail of Turnkey policies attached to each user's sub-org (L2).
/// Rarely written (signup + KYC approval). Used for reconciliation/backfill.
/// A user may have MULTIPLE active policies (e.g. deny-sponsored-pre-kyc AND
/// deny-closeaccount-lifecycle), so `userId` is NOT unique here. The unique
/// constraint is on (userId, policyName) so each policy type appears at most
/// once per user.
model SolanaSponsorshipPolicy {
  id                   String   @id @default(cuid())
  userId               String
  subOrgId             String                 // Turnkey sub-org the policy targets
  policyId             String?                // Turnkey policy ID once createPolicy returns
  policyName           String                 // e.g. "deny-sponsored-sol-pre-kyc"
  effect               String                 // "EFFECT_DENY" | "EFFECT_ALLOW"
  status               String   @default("active")  // "active" | "superseded" | "deleted"
  appliedAt            DateTime @default(now())
  supersededAt         DateTime?

  /// Idempotency key for retries / webhook redelivery. Set to a UUID per
  /// logical operation; reuse on retry to detect duplicate application.
  lastTurnkeyRequestId String?

  user                 User     @relation(fields: [userId], references: [id])

  @@unique([userId, policyName])
  @@index([userId])
  @@index([subOrgId])
}
```

`User` gains:
```prisma
  SolanaSponsorshipQuota  SolanaSponsorshipQuota?
  SolanaSponsorshipPolicy SolanaSponsorshipPolicy[]
```

### Components

#### 1. `SponsorshipEligibilityService` (L1 — backend gate)

Path: `src/modules/solana-sponsorship/sponsorship-eligibility.service.ts`

Single public method:

```typescript
async assertEligible(
  userId: string,
  kind: 'send' | 'rent_subsidy',
): Promise<void>
```

**Decision matrix:**

| `kind` | KYC status | Quota state | Decision |
|---|---|---|---|
| `send` | green (`sumsubReviewAnswer === 'GREEN'`) | n/a | **Allow** (Turnkey dashboard $1/day cap backstops) |
| `send` | not green | `preKycSendsUsed < 3` AND `preKycSpendUsdCents < 100` | **Allow**, increment counter + add estimated USD in transaction |
| `send` | not green | `preKycSendsUsed >= 3` OR `preKycSpendUsdCents >= 100` | **Deny** — `pre_kyc_exhausted` / `complete_kyc` |
| `rent_subsidy` | green | `rentSubsidiesToday === 0` (or last reset >24h ago) | **Allow**, increment + reset timestamp |
| `rent_subsidy` | green | `rentSubsidiesToday >= 1` | **Deny** — `daily_limit_reached` / `wait_24h` |
| `rent_subsidy` | not green | n/a | **Deny** — `kyc_required` / `complete_kyc` |

**Pre-KYC cap = "3 sends OR $1 lifetime, whichever hits first".** The USD estimate is computed at quote time using the rent/fee estimate already produced by `SolanaDirectTransferService.prepareTransfer`/`prepareSplTransfer` (the `ESTIMATED_FEE` constants). It's an estimate, not a precise per-tx charge — the goal is to stop a fake account from running 100 sub-cent sponsorships. Once either threshold trips, the user must complete KYC.

On deny, throws:
```typescript
throw new ForbiddenException({
  message: '<human-readable reason>',
  errorCode: 'SPONSORSHIP_NOT_AVAILABLE',
  data: {
    reason: 'pre_kyc_exhausted' | 'kyc_required' | 'daily_limit_reached',
    action: 'complete_kyc' | 'add_sol' | 'wait_24h',
  },
});
```

**KYC green check:** loads `KycProfile.sumsubReviewAnswer` (Sumsub enum; `'GREEN'` is approved). A `null` KycProfile is treated as not green.

**Atomicity:** the increment uses a conditional `updateMany` (`where: { preKycSendsUsed: <expected>, preKycSpendUsdCents: <expected> }`) so concurrent requests cannot double-spend the quota. If the row doesn't exist, lazily upsert it.

**Pre-flight gas-usage check (post-KYC sends only):** for KYC'd users where `kind === 'send'`, call Turnkey's `get_gas_usage({ organizationId: subOrgId })` before allowing. If `usageUsd >= windowLimitUsd` (i.e. user has already hit their dashboard $1/day cap), deny early with `daily_limit_reached` / `wait_24h` instead of letting the user proceed and have Turnkey reject the broadcast. This is a UX optimization, not a security control — Turnkey enforces the cap regardless. Failures of `get_gas_usage` (network blip, 5xx) are logged and skipped (don't block on a transient API issue).

**Configuration:** constants in the service for v1:
- `PRE_KYC_MAX_SENDS = 3`
- `PRE_KYC_MAX_SPEND_USD_CENTS = 100` ($1)
- `RENT_SUBSIDY_DAILY_LIMIT = 1`

Lift to env vars if tuning is needed later (`SPONSORSHIP_PRE_KYC_MAX_SENDS`, etc.). Default values match the agreed policy.

#### 2. `SolanaSponsorshipPolicyService` (L2 — Turnkey policy manager)

Path: `src/modules/solana-sponsorship/solana-sponsorship-policy.service.ts`

Wraps the existing `TurnkeyProvider` client. Three public methods:

```typescript
/** At signup: attach the always-on + pre-KYC policies to the new sub-org. */
async applyDefault(subOrgId: string, userId: string): Promise<void>

/** On Sumsub green webhook: flip the sub-org's pre-KYC policy to ALLOW. */
async upgradeToKyc(subOrgId: string, userId: string): Promise<void>

/** Read-only: returns the active policy state for reconciliation. */
async getPolicyState(subOrgId: string): Promise<PolicyStateDto>
```

**`applyDefault` attaches TWO policies to the new sub-org:**

**Policy A — Pre-KYC deny** (lifted on KYC approval):

```json
{
  "policyName": "deny-sponsored-sol-pre-kyc",
  "effect": "EFFECT_DENY",
  "condition": "activity.type == 'ACTIVITY_TYPE_SOL_SEND_TRANSACTION' && activity.params.sponsor == true",
  "notes": "Auto-applied at signup. Lifted when user completes KYC."
}
```

**Policy B — Rent-leakage deny** (always-on, never lifted; survives users controlling their own signer key):

```json
{
  "policyName": "deny-sol-account-lifecycle-leakage",
  "effect": "EFFECT_DENY",
  "condition": "solana.tx.instructions.any(i, i.program_key == 'ATokenGPvbdGVxr1b2hvZbsiqW5xWH25efTNsLJA8knL') || solana.tx.instructions.any(i, (i.program_key == 'TokenkegQfeZyiNwAJbNbGKPFXCWuBvf9Ss623VQ5DA' || i.program_key == 'TokenzQdBNbLqP5VEhdkAS6EPFLC1PHnBqCXEpPxuEb') && (i.instruction_data_hex == '09' || i.instruction_data_hex == '11'))",
  "notes": "Always-on. Blocks ATA-create program + CloseAccount(09) + SetAuthority(11) rent extraction in sponsored flows."
}
```

Policy B is the canonical Turnkey-recommended mitigation from the [Solana Rent Sponsorship guide](https://docs.turnkey.com/features/networks/solana-rent-refunds#policy-guidance). It complements (does not replace) the existing server-side `CloseAccount` instruction stripping in `broadcastWithSponsor` — the policy catches users who export their signer and submit `CloseAccount` outside our backend.

Both policies target the user's sub-org (passed as `organizationId` to the Turnkey call). Non-sponsored transactions (where `sponsor` is not `true`) are unaffected by Policy A. Policy B applies to all Solana transactions on the sub-org regardless of sponsorship status — this is intentional, since rent extraction works the same way regardless of who paid the original rent.

**Verification step (pre-implementation, mandatory):** the `activity.params.sponsor == true` condition in Policy A is **structurally valid** per the policy-language docs (the `sol_send_transaction` activity exposes `sponsor` as a parameter, and `activity.params.*` is the documented accessor) but no canonical Turnkey example uses this exact pattern. Before implementation, capture an actual `ACTIVITY_TYPE_SOL_SEND_TRANSACTION` activity payload (from your Turnkey dashboard activity log — one where `sponsor: true` was sent) and confirm the field name and value type. If the field is absent or named differently, fall back to one of these alternative conditions (in order of preference):

1. `activity.type == 'ACTIVITY_TYPE_SOL_SEND_TRANSACTION' && activity.params.sponsor == true` (primary)
2. `activity.type == 'ACTIVITY_TYPE_SOL_SEND_TRANSACTION'` without the sponsor filter — broader but still gates the entire sub-org's SOL send activity. Less ideal because it also blocks the user's non-sponsored self-paid transfers (the L1 backend gate would need to fall through to the standard `broadcastSignedOperation` path instead).
3. Use a per-user API key with sponsorship scoped off — Turnkey may support restricting API keys via policy. Investigate if options 1 and 2 both fail.

**`upgradeToKyc` does**:
1. Look up the active `SolanaSponsorshipPolicy` row for the user where `policyName = 'deny-sponsored-sol-pre-kyc'`.
2. If already `status='superseded'` (or no row exists), return (idempotent).
3. Call Turnkey `deletePolicy` on the deny policy's `policyId`.
4. (Optional) call `createPolicy(EFFECT_ALLOW)` with the same condition. Not strictly required — implicit deny applies, but the parent-org API key with `sponsor: true` should be allowed once the deny is gone. **Verify this against Turnkey's evaluation rules before rollout** (see Open Questions); if implicit deny blocks, add the explicit ALLOW.
5. Update DB: mark Policy A row `status='superseded'`. Policy B (rent-leakage) stays untouched.

**Idempotency:** `lastTurnkeyRequestId` (UUID per logical operation) is checked before any Turnkey call. On retry/redelivery, if the request ID matches an existing row, skip.

**Failure handling:** Turnkey API failures during `applyDefault` (at signup) must not block signup — log a warning and continue. The user simply has no policies attached, so the backend gate (L1) and dashboard cap (L3) still protect. The backfill script can be re-run anytime to attach any missing policies; operators should grep the signup logs for the warning after deploy and re-run backfill to backfill affected users. A nightly reconciliation job is intentionally out of scope here. For `upgradeToKyc`, a failure means the user stays denied — the Sumsub webhook should be retried (it already has retry semantics in `partner-submission.handler.ts`).

#### 3. `SolanaSponsorshipModule`

Path: `src/modules/solana-sponsorship/solana-sponsorship.module.ts`

Standard NestJS module exporting `SponsorshipEligibilityService` and `SolanaSponsorshipPolicyService`. Imported by:
- `AuthModule` (for `applyDefault` at signup)
- `KycModule` (for `upgradeToKyc` on webhook)
- `TransfersModule`, `SwapsModule` (for `assertEligibility` at the call sites)
- `AdminModule` (for the operator API below)

#### 4. `AdminSponsorshipPolicyService` + admin controller extensions

Path: `src/modules/admin/services/admin-sponsorship-policy.service.ts`

Extends the existing `AdminModule` (`src/modules/admin/admin.controller.ts`) with the four operator endpoints documented in the "Operator API" section below. Same `InternalApiGuard` pattern (`x-internal-api-key` header), same `@Throttle` limits. Wraps `SolanaSponsorshipPolicyService` for the actual Turnkey calls — this service handles input validation, batching, and dry-run logic.

### Sponsorship eligibility respects existing SOL balance (no-regression guarantee)

A critical no-regression guarantee: **users who already have enough SOL to cover their own gas MUST NOT trigger sponsorship, policy checks, or quota increments.** This is already the case in the existing codebase and the design preserves it.

The decision is made by `SolanaDirectTransferService.prepareTransfer` / `prepareSplTransfer` BEFORE the eligibility gate is reached:

```104:135:src/modules/transfers/services/solana-direct-transfer.service.ts
  async prepareTransfer(fromAddress, toAddress, lamports) {
    const ESTIMATED_FEE = 10_000;
    const balance = await this.connection.getBalance(fromPubkey, 'confirmed');

    let requiresSponsor = false;
    if (balance < lamports + ESTIMATED_FEE) {
      // ... only here does requiresSponsor potentially become true
    }
```

If `balance >= lamports + ESTIMATED_FEE`, the function returns `requiresSponsor: undefined`. The frontend then signs normally with the user's key (paying their own gas), and on `/transfers/execute` the `sponsor` parameter is `undefined`/`false`. The `executeDirectSolanaTransfer` handler routes to `broadcastSignedOperation` (the non-sponsored path), which **does not call `assertEligible`**:

```310:312:src/modules/transfers/handlers/transfer-execution.handler.ts
      } else {
        txHash = await this.solanaDirectTransfer.broadcastSignedOperation(operation);
      }
```

Net effect:
- User has SOL → pays own gas → no policy evaluation, no quota counter increment, no Turnkey dashboard spend. Completely unchanged behavior.
- User lacks SOL → sponsorship needed → `assertEligible` runs → policy + dashboard cap apply.

The eligibility gate is **only** invoked in the sponsored path. This must be preserved in implementation — the `assertEligible` call must live inside the `if (sponsor)` branch, never at the top of `executeDirectSolanaTransfer`.

The same principle applies to `subsidizeSolRent`: it already self-gates by checking `balance >= SUBSIDY_THRESHOLD` (0.007 SOL) and returns `null` without sending if the user has enough. The eligibility gate is added at the call sites (`subsidizeSolRentIfNeeded`) and is only reached if that early-return doesn't fire — but for safety, the gate is checked unconditionally at those sites too, since they're the choke point for the budget-draining path.

### Integration Points (call sites)

Five locations get a single guard. Each guard is added before the existing sponsorship logic — no flow changes.

| # | File | Function | Guard |
|---|---|---|---|
| 1 | `src/modules/transfers/handlers/transfer-execution.handler.ts` | `executeDirectSolanaTransfer` (the `if (sponsor)` branch, after the subOrgId-vs-DB check around line 286) | `await this.sponsorshipEligibility.assertEligible(transaction.userId, 'send')` (the estimated USD is read from the prepared operation's `ESTIMATED_FEE` / ATA-rent constants; for the L1 counter, we charge the worst-case fee against the quota) |
| 2 | `src/modules/swaps/handlers/swap-relay-quote.handler.ts` | `subsidizeSolRentIfNeeded` (around line 555) | `await this.sponsorshipEligibility.assertEligible(userId, 'rent_subsidy')` |
| 3 | `src/modules/transfers/handlers/transfer-relay-quote.handler.ts` | `subsidizeSolRentIfNeeded` (around line 1245) | Same as #2 |
| 4 | `src/modules/auth/handlers/session-sync.handler.ts` | New-user creation path (where `Wallet` row is created after Turnkey sub-org creation) | `await this.solanaSponsorshipPolicy.applyDefault(subOrgId, user.id)` (fire-and-forget with `.catch()` — must not block signup) |
| 5 | `src/modules/kyc/handlers/sumsub-kyc.handler.ts` | `handleApplicantReviewed` approved branch (after `KycProfile.sumsubReviewAnswer='GREEN'` is written) | `await this.solanaSponsorshipPolicy.upgradeToKyc(user.subOrgId, user.id)` (errors logged but don't fail the webhook; webhook has retry) |

**For #1**, the guard fires *before* the call to `solanaDirectTransfer.broadcastWithSponsor`. The existing subOrgId-vs-DB mismatch check stays. If the guard denies, the user gets the `SPONSORSHIP_NOT_AVAILABLE` error instead of having Turnkey reject the tx.

**For #2 and #3**, `subsidizeSolRentIfNeeded` currently swallows errors and returns silently. The guard's `ForbiddenException` should be caught at the same level (warn-logged) so a deny doesn't break the quote — the swap/transfer proceeds, the user just doesn't get the SOL pre-funding. The downstream OneBalance/Relay call will surface its own `INSUFFICIENT_FUNDS_FOR_RENT` error if relevant. This preserves existing behavior for legitimate users who fail the check while protecting the budget.

**For #4**, fire-and-forget is important: if Turnkey's API is briefly down at signup, the user still gets their account. The policy can be applied later by the backfill/reconciliation script.

### Error Handling & Frontend Contract

One new error code surfaced to clients:

```json
{
  "statusCode": 403,
  "message": "Sponsored SOL transfers require identity verification.",
  "errorCode": "SPONSORSHIP_NOT_AVAILABLE",
  "data": {
    "reason": "pre_kyc_exhausted",
    "action": "complete_kyc"
  }
}
```

`reason` enum: `pre_kyc_exhausted` | `kyc_required` | `daily_limit_reached`
`action` enum: `complete_kyc` | `add_sol` | `wait_24h`

**Frontend handling** (out of scope for backend implementation, but documented):
- `action=complete_kyc` → show KYC CTA
- `action=add_sol` → show "Add SOL to your wallet to continue" CTA
- `action=wait_24h` → show "Try again tomorrow" message

For pre-KYC users, sponsorship works silently for the first 3 sends (or until cumulative spend hits $1, whichever first), with the counter incrementing in the background. On the 4th attempt (or first attempt past $1), they see the KYC CTA. This matches the agreed "3 sends / $1 lifetime pre-KYC, then KYC required" behavior.

### Backfill Script

Path: `scripts/backfill-solana-sponsorship-policies.ts`

One-time script to attach policies to existing users based on current KYC status. See the "Operator API" section below for the safer, staged alternative — the script is the bulk-migration tool for the final step.

**Behavior:**
1. Query all `User` rows joined with `Wallet` (to get `subOrgId`) and `KycProfile` (for KYC status).
2. Skip users who already have an active `SolanaSponsorshipPolicy` row for the relevant `policyName` (idempotent per-policy).
3. For each remaining user, attach:
   - **Always**: Policy B (`deny-sol-account-lifecycle-leakage`) — applies to every user, KYC'd or not.
   - **If KYC is NOT green**: Policy A (`deny-sponsored-sol-pre-kyc`) — these users cannot sponsor until they KYC.
   - **If KYC IS green**: skip Policy A (the user is already trusted; they only get Policy B).
4. **Dry-run by default.** `--apply` flag executes for real. Dry-run prints a summary table of what would change.
5. **Rate-limited:** 5 requests/second to respect Turnkey API limits (configurable via `--rps`). Each user gets up to 2 Turnkey API calls (Policy A + Policy B), so 5 users/sec ≈ 10 Turnkey calls/sec.
6. Writes a summary report to `scripts/output/sponsorship-policy-backfill-<timestamp>.json` (git-ignored).
7. Resumable: tracks processed userIds in the report file; re-running skips them.

**Usage:**
```bash
npx ts-node scripts/backfill-solana-sponsorship-policies.ts                 # dry-run
npx ts-node scripts/backfill-solana-sponsorship-policies.ts --apply         # execute
npx ts-node scripts/backfill-solana-sponsorship-policies.ts --apply --rps 2 # slower
```

### Operator API — staged policy rollout (test on yourself first)

The bulk backfill script is a fire-and-forget tool. For **safe staged rollout** (test on your own account, verify, then expand), we expose three operator endpoints under the existing internal admin API. These reuse the existing `InternalApiGuard` (`x-internal-api-key` header), matching the pattern in `src/modules/admin/admin.controller.ts`.

#### `POST /api/v1/admin/sponsorship-policy/apply`

Attach Policy A and/or Policy B to one or more specific users by `userId`. Used for the "test on my own account" step.

**Request body:**
```json
{
  "userIds": ["<your-user-id>"],
  "policies": ["deny-sponsored-sol-pre-kyc", "deny-sol-account-lifecycle-leakage"],
  "dryRun": false
}
```

- `policies` is optional; omit (or pass both names) to apply both. Pass only `"deny-sol-account-lifecycle-leakage"` to test Policy B in isolation.
- `dryRun: true` validates inputs and returns what *would* happen without calling Turnkey.

**Response:**
```json
{
  "applied": [
    { "userId": "<id>", "policyName": "deny-sponsored-sol-pre-kyc", "turnkeyPolicyId": "policy_abc123", "status": "active" },
    { "userId": "<id>", "policyName": "deny-sol-account-lifecycle-leakage", "turnkeyPolicyId": "policy_def456", "status": "active" }
  ],
  "skipped": [],
  "errors": []
}
```

#### `POST /api/v1/admin/sponsorship-policy/remove`

Remove (supersede) one or both policies from a user. Used to roll back if a test goes wrong, or to lift Policy A manually for a user you've verified out-of-band.

**Request body:**
```json
{
  "userIds": ["<your-user-id>"],
  "policies": ["deny-sponsored-sol-pre-kyc"]
}
```

- Calls Turnkey `deletePolicy` for each named policy, marks the DB row `status='superseded'`.
- Omitting `policies` removes both.

#### `GET /api/v1/admin/sponsorship-policy/:userId`

Read-only — returns the current policy state for a user. Used to verify what's attached before/after a test.

**Response:**
```json
{
  "userId": "<id>",
  "subOrgId": "<turnkey-sub-org-id>",
  "kycStatus": "GREEN",
  "policies": [
    { "policyName": "deny-sol-account-lifecycle-leakage", "effect": "EFFECT_DENY", "status": "active", "appliedAt": "2026-07-06T..." },
    { "policyName": "deny-sponsored-sol-pre-kyc", "effect": "EFFECT_DENY", "status": "superseded", "appliedAt": "...", "supersededAt": "..." }
  ]
}
```

#### `POST /api/v1/admin/sponsorship-policy/backfill`

Triggers the same logic as the backfill script, but as a job with filters. Used for the bulk-migration step after you've verified on a few test accounts.

**Request body:**
```json
{
  "userIds": ["<id1>", "<id2>", "..."],
  "respectKycStatus": true,
  "dryRun": true,
  "rps": 5
}
```

- Pass an explicit list of `userIds` (e.g. the suspicious-users list + a sample of normal users) rather than "all users" — that way you control the blast radius. A separate `--all` mode is intentionally **not** provided; for true bulk migration use the script with its resumable state file.
- `respectKycStatus: true` means "only apply Policy A to non-KYC'd users" (default). Set `false` to force Policy A on everyone (not recommended).
- Returns immediately with a `jobId`; progress is logged server-side. (v1: just synchronously process and return the summary — async job queue is overkill for a one-off migration. If the batch is >500 users, use the script instead.)

#### Staged rollout procedure (the safe path)

This is the recommended sequence. Each step is reversible via `/remove`.

1. **Test on yourself:**
   ```bash
   curl -X POST http://localhost:3000/api/v1/admin/sponsorship-policy/apply \
     -H "x-internal-api-key: $INTERNAL_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"userIds":["<YOUR_USER_ID>"],"dryRun":true}'
   ```
   Review the dry-run output. Then run with `"dryRun":false`.

2. **Verify the policies are attached:**
   ```bash
   curl http://localhost:3000/api/v1/admin/sponsorship-policy/<YOUR_USER_ID> \
     -H "x-internal-api-key: $INTERNAL_API_KEY"
   ```
   Confirm both policies show `status: "active"`.

3. **Verify the Turnkey dashboard** shows the same policies on your sub-org.

4. **Test the user-facing behavior with your own account** (since you're now Policy-A-denied if you're not KYC'd, or do this on a fresh test account that isn't KYC'd):
   - Try a sponsored SOL send as the test user → should be denied with `SPONSORSHIP_NOT_AVAILABLE` / `pre_kyc_exhausted` (if you've used your 3 free sends) or work and tick the counter.
   - Try the rent-extraction path (submit a `CloseAccount` instruction) → should be denied by Policy B at Turnkey signing time.

5. **Roll out to a small batch** of real users (e.g. 10 KYC'd + 10 non-KYC'd):
   ```bash
   curl -X POST http://localhost:3000/api/v1/admin/sponsorship-policy/backfill \
     -H "x-internal-api-key: $INTERNAL_API_KEY" \
     -d '{"userIds":["<id1>","<id2>",...],"dryRun":false}'
   ```

6. **Verify a sample** via `/sponsorship-policy/:userId`. Spot-check 5 users in the Turnkey dashboard.

7. **Bulk migration**: run the backfill script (`scripts/backfill-solana-sponsorship-policies.ts --apply`) against the full user base during a low-traffic window. The script is resumable; if it crashes at user 247/500, re-running continues from where it left off.

8. **Flip `FF_SOLANA_SPONSORSHIP_POLICY=true`** so new signups and KYC webhooks attach/upgrade policies automatically.

9. **Monitor for 1 week.** Grep signup logs for policy-attach warnings. Re-run the script for any users that errored out.

10. **Remove the feature flag** once stable.

### Testing

**Unit tests:**
- `SponsorshipEligibilityService` decision matrix:
  - All 6 base branches (KYC/pre-KYC × send/rent_subsidy).
  - Pre-KYC send quota edges: 2 sends used + <$1 → allow; 3 sends used → deny; 2 sends used but cumulative ≥$1 → deny (whichever-hits-first rule).
  - Atomic increment via conditional `updateMany` under simulated concurrency (two `assertEligible` calls in parallel, only one succeeds).
  - `get_gas_usage` pre-flight: returns over-cap → deny with `daily_limit_reached`; API throws → allow (don't block on transient failure).
- `SolanaSponsorshipPolicyService`:
  - `applyDefault` creates BOTH Policy A and Policy B; idempotent on repeat (skips existing `policyName` rows).
  - `upgradeToKyc` deletes Policy A only, leaves Policy B untouched; idempotent on repeat.
  - `getPolicyState` returns correct active policies for reconciliation.
  - Failure paths: Turnkey throws on Policy A → still attempts Policy B (independent); both throw → logged, not fatal.
- KYC green check: cover `KycProfile` absent, `sumsubReviewAnswer` non-GREEN, and GREEN cases.

**Integration tests:**
- New test: signup flow → assert `SolanaSponsorshipPolicy` rows created for BOTH `deny-sponsored-sol-pre-kyc` and `deny-sol-account-lifecycle-leakage`. Mock Sumsub webhook → assert pre-KYC row flipped to `superseded`, lifecycle row still `active`.
- Extend `src/modules/transfers/services/solana-direct-transfer.subsidy.spec.ts` to cover the new guard (deny case throws with correct errorCode, allow case passes through and increments counter).

**Backfill script:**
- Idempotency test: run twice against a fixture DB; second run is a no-op.
- Dry-run test: `--apply` omitted → no Turnkey calls made.
- KYC-split test: KYC'd users get only Policy B; non-KYC'd users get Policy A + Policy B.

**Manual / smoke (post-deploy):**
- Create a fresh account, attempt sponsored SOL sends → confirm 3 work (pre-KYC allowance), 4th is denied with `pre_kyc_exhausted`.
- Complete KYC on that account → confirm subsequent sponsored sends work and are only capped by the dashboard $1/day.
- Independently verify Policy B catches a manually-constructed `CloseAccount` tx by submitting one through the Turnkey dashboard against a sandbox sub-org (should be denied by policy).
- Check a known-abuser account (from `find-suspicious-sponsorship-users.ts` output, post-backfill) → confirm sponsorship is denied.

### Rollout Plan

1. Merge Prisma migration + new services + call-site guards. The L1 guards (call sites #1, #2, #3) are active immediately; the L2 policy calls (#4, #5) are wired but gated behind `FF_SOLANA_SPONSORSHIP_POLICY` (default `false`).
2. Run backfill script in **dry-run** mode against production DB; review summary.
3. Run backfill in **--apply** mode in small batches (e.g. 50 users at a time) during low-traffic window. Monitor Turnkey API rate limits.
4. Verify a sample of users (5 KYC'd, 5 non-KYC'd) have the expected policy attached via the Turnkey dashboard.
5. Flip `FF_SOLANA_SPONSORSHIP_POLICY=true` so new signups and KYC webhooks also apply/upgrade policies going forward.
6. After 1 week of stable operation, remove the feature flag entirely.

**Feature flag:** `FF_SOLANA_SPONSORSHIP_POLICY` (boolean, default `false`). When false:
- L1 backend gate is still active (this is the core protection).
- L2 policy application at signup / KYC webhook is skipped.
- Backfill script still works (it's a one-off, not gated).

This lets us ship L1 immediately for protection while rolling out L2 carefully.

### Open Questions / Verification Steps

1. **Exact Turnkey policy field for `sponsor` (VERIFIED STRUCTURALLY, NEEDS RUNTIME CONFIRM):** the condition `activity.params.sponsor == true` is structurally valid — the `sol_send_transaction` activity exposes `sponsor` as a parameter (per [broadcast SVM transaction docs](https://docs.turnkey.com/api-reference/activities/broadcast-svm-transaction)) and `activity.params.*` is the documented accessor in the [policy language page](https://docs.turnkey.com/features/policies/language). However, no canonical Turnkey policy example uses this exact pattern (all sponsored-policy examples in the docs use the `solana.tx.*` namespace). Before implementation: capture an actual `ACTIVITY_TYPE_SOL_SEND_TRANSACTION` activity payload from the Turnkey dashboard (one where `sponsor: true` was sent) and confirm the field name and value type. If it differs, use one of the fallback conditions documented in the `SolanaSponsorshipPolicyService` section.

2. **Implicit-deny behavior after deleting the DENY policy (NEEDS RUNTIME CONFIRM):** confirm whether an explicit `EFFECT_ALLOW` policy is required for the parent-org API key to sponsor a sub-org tx after Policy A is deleted, or if removing the DENY is sufficient. Test against a sandbox sub-org before rollout. The docs say "all actions are implicitly denied by default" with exceptions including "users have implicit permission to approve an activity if they were included in consensus" — but it's unclear whether the parent-org API key acting on a sub-org qualifies. If implicit deny blocks, add an explicit `EFFECT_ALLOW` policy with the same condition in `upgradeToKyc`.

3. **Pre-KYC counter reset on KYC (DECIDED — no reset):** the design intentionally does NOT reset `preKycSendsUsed` or `preKycSpendUsdCents` on KYC approval. Post-KYC sends are gated by the Turnkey dashboard cap, not this counter, so the counter becomes irrelevant. No behavioral impact.

4. **Pre-KYC spend estimation accuracy (ACCEPTABLE APPROXIMATION):** the `preKycSpendUsdCents` counter is charged using the worst-case `ESTIMATED_FEE` constant from `SolanaDirectTransferService` (10k lamports tx fee, or 2.1M lamports with ATA rent). At a sample ~$150/SOL, 2.1M lamports ≈ $0.32, so a single ATA-creating SPL transfer costs the user ~$0.32 of their $1 lifetime pre-KYC budget — they get 3 sends max, but the $1 cap will trip first if all 3 need ATA creation. This is the intended behavior. The conversion lamports→USD is done via the same price oracle the rest of the app uses (CoinGecko key in `.env`); the static `ESTIMATED_FEE` constants in `SolanaDirectTransferService` are lamport-denominated and don't drift with price. For v1, a worst-case estimate is acceptable since the goal is just to bound abuse, not to bill precisely.

5. **Policy B scope (DECIDED — applies to all Solana txs, not just sponsored):** Policy B (`deny-sol-account-lifecycle-leakage`) intentionally blocks `CloseAccount` / `SetAuthority` on ALL of a user's Solana transactions, not just sponsored ones. This is correct because the rent-extraction vector works the same way regardless of who paid the original rent. The user impact: they cannot close token accounts through our backend. This is acceptable — closing token accounts is not part of any documented product flow, and users who really want to can still do it by exporting their key and submitting a tx outside our backend (we're not blocking the chain, just our backend's signing path).

### References

- Turnkey Policy Engine overview: https://docs.turnkey.com/features/policies/overview
- Turnkey Policy Language (DSL grammar, `solana.tx`, `activity.params`): https://docs.turnkey.com/features/policies/language
- Turnkey Solana Policy Examples (instruction conditions, SPL transfers): https://docs.turnkey.com/features/policies/examples/solana
- Turnkey Solana Rent Sponsorship (rent-extraction risk, CloseAccount mitigation, policy guidance): https://docs.turnkey.com/features/networks/solana-rent-refunds
- Turnkey Sending Sponsored Solana Transactions: https://docs.turnkey.com/features/transaction-management/sending-sponsored-solana-transactions
- Turnkey Solana Transaction Construction (payer model, account-creation caveats): https://docs.turnkey.com/features/networks/solana-transaction-construction
- Turnkey Create Policy API: https://docs.turnkey.com/api-reference/activities/create-policy
- Turnkey Broadcast SVM Transaction (confirms `sponsor` is an activity parameter): https://docs.turnkey.com/api-reference/activities/broadcast-svm-transaction
- Turnkey Get Gas Usage (dashboard cap query, read-only): https://docs.turnkey.com/api-reference/queries/get-gas-usage
- Existing sponsorship code:
  - `src/modules/transfers/services/solana-direct-transfer.service.ts` (`broadcastWithSponsor`, `subsidizeSolRent`, existing `CloseAccount` stripping at L665-L674)
  - `src/modules/transfers/handlers/transfer-execution.handler.ts` (`executeDirectSolanaTransfer` sponsor branch)
  - `src/modules/swaps/handlers/swap-relay-quote.handler.ts` (`subsidizeSolRentIfNeeded`)
  - `src/modules/transfers/handlers/transfer-relay-quote.handler.ts` (`subsidizeSolRentIfNeeded`)
- Existing KYC webhook: `src/modules/kyc/webhooks/sumsub-webhook.controller.ts`, `src/modules/kyc/handlers/sumsub-kyc.handler.ts`
- Existing signup / sub-org creation: `src/modules/auth/handlers/session-sync.handler.ts`
- Existing Turnkey client wrapper: `src/modules/auth/providers/turnkey.provider.ts`
- Existing reactive abuse script: `scripts/find-suspicious-sponsorship-users.ts`
- Existing (disabled) abuse guard: `src/modules/swaps/services/micro-solana-swap-abuse-guard.service.ts`
