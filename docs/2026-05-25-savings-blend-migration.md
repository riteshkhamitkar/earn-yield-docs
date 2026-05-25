# Savings → Blend Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all 19 broken/inconsistent surfaces after USD savings migrated from Yo Protocol (yoUSD vault) to Blend (Gnosis Safe), while keeping EUR savings on Yo Protocol completely untouched.

**Architecture:** Add a `protocol` column (`'blend'` | `'yo'`) to `SavingsGoal` as the single routing discriminator. All `savings.service.ts` logic branches on this field. Hardcoded YOUSD/YOEUR special-cases are removed from portfolio, dashboard, and transfer handlers, relying on the Token table for enrichment instead.

**Tech Stack:** NestJS, Prisma (PostgreSQL), TypeScript. Blend SDK (`@blend-money/node`). Yo Protocol API (`YoApiService`). Zerion for portfolio positions.

---

## File Map

| File | Task | Change |
|------|------|--------|
| `prisma/schema.prisma` | 1 | Add `protocol String @default("yo")` to `SavingsGoal` |
| `prisma/migrations/20260525000000_savings_blend_migration/migration.sql` | 1 | New migration SQL |
| `prisma/scripts/sync-yield-tokens-to-token-table.ts` | 2 | New script — upsert yoUSD + yoEUR into Token table |
| `src/modules/savings/savings.service.ts` | 3 | Fixes 1–9: protocol branching, Blend APY, details routing |
| `src/modules/balances/handlers/transaction-history.handler.ts` | 4 | Fix 10: blendUSD icon; Fix 18: stale fallback; Fix 19: stale comment |
| `src/modules/dashboard/handlers/dashboard.handler.ts` | 5 | Fix 13: remove YOUSD→stocks hardcode |
| `src/modules/balances/handlers/asset-details.handler.ts` | 5 | Fix 14: remove YOUSD hardcodes |
| `src/modules/balances/handlers/holding-assets.handler.ts` | 5 | Fix 15+16: remove YOUSD/YOEUR hardcodes + wrong priceApiId |
| `src/modules/transfers/transfers.service.ts` | 6 | Fix 17: remove YOUSD/YOEUR sendable exception |

---

## Task 1: Schema Migration — Add `protocol` column to `SavingsGoal`

**Files:**
- Modify: `prisma/schema.prisma`
- Create: `prisma/migrations/20260525000000_savings_blend_migration/migration.sql`

- [ ] **Step 1.1 — Add `protocol` field to `SavingsGoal` in `schema.prisma`**

Open `prisma/schema.prisma`. Find the `SavingsGoal` model (around line 262). Add the `protocol` field between `isDefault` and `yieldTokenSymbol`:

```prisma
model SavingsGoal {
  id               String            @id @default(cuid())
  userId           String
  name             String
  icon             String
  targetAmount     Decimal           @db.Decimal(18, 2)
  currency         String
  status           String            @default("ACTIVE")
  isDefault        Boolean           @default(false)
  protocol         String            @default("yo")   // "blend" | "yo"
  yieldTokenSymbol String
  createdAt        DateTime          @default(now())
  updatedAt        DateTime          @updatedAt

  User             User              @relation(fields: [userId], references: [id], onDelete: Cascade)
  Transaction      Transaction[]
  YieldDeposit     YieldDeposit[]
  YieldRedemption  YieldRedemption[]

  @@unique([userId, name])
  @@index([userId])
}
```

- [ ] **Step 1.2 — Create the migration SQL file**

Create the directory and file:
```
prisma/migrations/20260525000000_savings_blend_migration/migration.sql
```

Contents:
```sql
-- Add protocol discriminator column (default 'yo' = Yo Protocol)
ALTER TABLE "SavingsGoal" ADD COLUMN "protocol" TEXT NOT NULL DEFAULT 'yo';

-- Mark all existing USD goals as Blend-backed
UPDATE "SavingsGoal" SET protocol = 'blend' WHERE currency = 'USD';

-- Dev/test data wipe — safe because all FK constraints are ON DELETE SET NULL:
--   Transaction.savingsGoalId     → NULL (record kept)
--   YieldDeposit.savingsGoalId    → NULL (record kept)
--   YieldRedemption.savingsGoalId → NULL (record kept)
-- Nothing outside SavingsGoal is deleted.
DELETE FROM "SavingsGoal";
```

- [ ] **Step 1.3 — Apply the migration**

```bash
npx prisma migrate dev --name savings_blend_migration
```

Expected output:
```
Applying migration `20260525000000_savings_blend_migration`
The following migration(s) have been applied:
migrations/
  └─ 20260525000000_savings_blend_migration/
    └─ migration.sql
```

- [ ] **Step 1.4 — Verify TypeScript still compiles**

```bash
npx tsc --noEmit
```

Expected: no output (exit code 0). If Prisma client needs regenerating: `npx prisma generate` first.

- [ ] **Step 1.5 — Commit**

```bash
git add prisma/schema.prisma prisma/migrations/20260525000000_savings_blend_migration
git commit -m "feat(savings): add protocol column to SavingsGoal for blend/yo routing"
```

---

## Task 2: Token Table Script — Add yoUSD and yoEUR

**Files:**
- Create: `prisma/scripts/sync-yield-tokens-to-token-table.ts`

Context: yoUSD and yoEUR were deleted from the `Token` table. They need to be re-added so Zerion portfolio positions for these vault tokens get correct name/logo enrichment. Data is sourced from the `YieldToken` table (already correct). Both tokens get `supportedForSwap: false` so they never appear in the send/swap picker.

- [ ] **Step 2.1 — Create the script**

Create `prisma/scripts/sync-yield-tokens-to-token-table.ts`:

```typescript
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  console.log('--- Syncing yoUSD and yoEUR into Token table ---');

  // 1. Find or create the 'earn' group
  let earnGroup = await prisma.group.findUnique({ where: { key: 'earn' } });
  if (!earnGroup) {
    const maxOrderGroup = await prisma.group.findFirst({ orderBy: { order: 'desc' } });
    earnGroup = await prisma.group.create({
      data: {
        key: 'earn',
        name: 'Earn',
        order: (maxOrderGroup?.order ?? 0) + 1,
      },
    });
    console.log('  Created earn group');
  } else {
    console.log(`  Found earn group (id=${earnGroup.id})`);
  }

  // 2. Find or create the 'yield-vaults' category under earn
  let yieldCategory = await prisma.category.findFirst({
    where: { groupId: earnGroup.id, key: 'yield-vaults' },
  });
  if (!yieldCategory) {
    const maxOrderCat = await prisma.category.findFirst({
      where: { groupId: earnGroup.id },
      orderBy: { order: 'desc' },
    });
    yieldCategory = await prisma.category.create({
      data: {
        key: 'yield-vaults',
        name: 'Yield Vaults',
        groupId: earnGroup.id,
        order: (maxOrderCat?.order ?? 0) + 1,
      },
    });
    console.log('  Created yield-vaults category');
  } else {
    console.log(`  Found yield-vaults category (id=${yieldCategory.id})`);
  }

  // 3. Fetch yoUSD and yoEUR from YieldToken table
  const yieldTokens = await prisma.yieldToken.findMany({
    where: { symbol: { in: ['yoUSD', 'yoEUR'] } },
  });

  if (yieldTokens.length === 0) {
    console.error('ERROR: No yoUSD or yoEUR found in YieldToken table. Run populate-yield-tokens.ts first.');
    process.exit(1);
  }

  // 4. Upsert each into Token table
  for (const yt of yieldTokens) {
    await prisma.token.upsert({
      where: { symbol: yt.symbol },
      update: {
        name: yt.name,
        logoUrl: yt.logoUrl,
        assetId: yt.assetId,
        decimals: yt.decimals,
        supportedForSwap: false,
        groupId: earnGroup.id,
        categoryId: yieldCategory.id,
        priceApiId: null, // vault tokens have no CoinGecko price ID
      },
      create: {
        symbol: yt.symbol,
        name: yt.name,
        logoUrl: yt.logoUrl,
        assetId: yt.assetId,
        decimals: yt.decimals,
        supportedForSwap: false,
        groupId: earnGroup.id,
        categoryId: yieldCategory.id,
        priceApiId: null,
      },
    });
    console.log(`  ✓ ${yt.symbol} → Token table (group=earn, supportedForSwap=false, priceApiId=null)`);
  }

  console.log('\n✓ Done. yoUSD and yoEUR are now in the Token table.');
}

main()
  .catch(e => { console.error(e); process.exit(1); })
  .finally(async () => await prisma.$disconnect());
```

- [ ] **Step 2.2 — Run the script**

```bash
npx ts-node -r tsconfig-paths/register prisma/scripts/sync-yield-tokens-to-token-table.ts
```

Expected output:
```
--- Syncing yoUSD and yoEUR into Token table ---
  Found earn group (id=...) OR Created earn group
  Found yield-vaults category (id=...) OR Created yield-vaults category
  ✓ yoUSD → Token table (group=earn, supportedForSwap=false, priceApiId=null)
  ✓ yoEUR → Token table (group=earn, supportedForSwap=false, priceApiId=null)

✓ Done. yoUSD and yoEUR are now in the Token table.
```

- [ ] **Step 2.3 — Commit**

```bash
git add prisma/scripts/sync-yield-tokens-to-token-table.ts
git commit -m "feat(tokens): script to sync yoUSD and yoEUR into Token table"
```

---

## Task 3: `savings.service.ts` — Core Fixes 1–9

**Files:**
- Modify: `src/modules/savings/savings.service.ts`

This is the core task. All changes are in `savings.service.ts`. Read the entire file before starting.

### Fix 1 — `createGoal`: set `protocol` and `yieldTokenSymbol` correctly

- [ ] **Step 3.1 — Replace `createGoal` method body (lines 57–82)**

```typescript
async createGoal(userId: string, dto: CreateGoalDto) {
  const isEur = dto.currency.toUpperCase() === 'EUR';
  const protocol = isEur ? 'yo' : 'blend';
  const yieldTokenSymbol = isEur ? 'yoEUR' : 'blendUSD';

  try {
    return await this.prisma.savingsGoal.create({
      data: {
        userId,
        name: dto.name,
        icon: dto.icon,
        targetAmount: dto.targetAmount,
        currency: dto.currency.toUpperCase(),
        protocol,
        yieldTokenSymbol,
        isDefault: dto.isDefault ?? false,
      },
    });
  } catch (e: any) {
    if (e.code === 'P2002') {
      throw new ConflictException(
        `Goal with name "${dto.name}" already exists`,
      );
    }
    throw e;
  }
}
```

### Fix 2 — `getGoal`: branch on `goal.protocol`

- [ ] **Step 3.2 — Replace the entire `getGoal` method (lines 260–339)**

```typescript
async getGoal(userId: string, goalId: string) {
  const [goal, user] = await Promise.all([
    this.prisma.savingsGoal.findFirst({ where: { id: goalId, userId } }),
    this.prisma.user.findUnique({ where: { id: userId }, include: { Wallet: true } }),
  ]);

  if (!goal) throw new NotFoundException('Goal not found');

  // ── Blend-backed USD goal ──────────────────────────────────────────────
  if (goal.protocol === 'blend') {
    const blendTxs = await this.prisma.transaction.findMany({
      where: {
        userId,
        savingsGoalId: goalId,
        service: 'blend',
        status: TransactionStatus.COMPLETED,
        type: { in: [TransactionType.DEPOSIT, TransactionType.REDEMPTION] },
      },
      select: { type: true, fromAmount: true, toAmount: true },
    });

    let totalDeposited = 0;
    let totalWithdrawn = 0;
    for (const tx of blendTxs) {
      const amount = parseFloat(tx.fromAmount || tx.toAmount || '0');
      if (isNaN(amount) || amount <= 0) continue;
      if (tx.type === TransactionType.DEPOSIT) {
        totalDeposited += amount;
      } else {
        totalWithdrawn += amount;
      }
    }

    return {
      id: goal.id,
      name: goal.name,
      icon: goal.icon,
      targetAmount: Number(goal.targetAmount),
      currency: goal.currency,
      status: goal.status,
      isDefault: goal.isDefault,
      protocol: goal.protocol,
      yieldTokenSymbol: goal.yieldTokenSymbol,
      currentAmount: Math.max(0, totalDeposited - totalWithdrawn),
      totalDeposited,
      totalWithdrawn,
      createdAt: goal.createdAt,
      updatedAt: goal.updatedAt,
    };
  }

  // ── Yo Protocol EUR goal ───────────────────────────────────────────────
  if (!user?.Wallet?.[0]?.ethereumAddress) throw new BadRequestException('User wallet not found');

  const vaultToken = await this.prisma.yieldToken.findUnique({
    where: { symbol: goal.yieldTokenSymbol },
    select: { network: true, vaultAddress: true, decimals: true },
  });
  if (!vaultToken) throw new BadRequestException('Vault for goal not found');

  const history = await this.yoApi.getUserHistory(
    vaultToken.network,
    vaultToken.vaultAddress,
    user.Wallet[0].ethereumAddress,
  );

  const [localTxs, internalTxs] = await Promise.all([
    this.prisma.transaction.findMany({
      where: { userId, savingsGoalId: goalId, transactionHash: { not: null } },
      select: { transactionHash: true },
    }),
    this.prisma.transaction.findMany({
      where: {
        userId,
        savingsGoalId: goalId,
        type: { in: [TransactionType.INTERNAL_TRANSFER_IN, TransactionType.INTERNAL_TRANSFER_OUT] },
        status: TransactionStatus.COMPLETED,
      },
      select: { type: true, fromAmount: true, toAmount: true },
    }),
  ]);
  const goalTxHashes = new Set(localTxs.map(t => t.transactionHash!.toLowerCase()));

  let totalDeposited = 0;
  let totalWithdrawn = 0;
  for (const tx of history || []) {
    const txHash = (tx.transactionHash || (tx as any).txHash)?.toLowerCase();
    if (txHash && goalTxHashes.has(txHash)) {
      const amount = parseFloat(fromSmallestUnit(tx.assets.raw, vaultToken.decimals));
      if (tx.type === 'Deposit') totalDeposited += amount;
      else totalWithdrawn += amount;
    }
  }
  for (const tx of internalTxs) {
    const amount = parseFloat(
      tx.type === TransactionType.INTERNAL_TRANSFER_IN ? tx.toAmount! : tx.fromAmount!,
    );
    if (tx.type === TransactionType.INTERNAL_TRANSFER_IN) totalDeposited += amount;
    else totalWithdrawn += amount;
  }

  return {
    id: goal.id,
    name: goal.name,
    icon: goal.icon,
    targetAmount: Number(goal.targetAmount),
    currency: goal.currency,
    status: goal.status,
    isDefault: goal.isDefault,
    protocol: goal.protocol,
    yieldTokenSymbol: goal.yieldTokenSymbol,
    currentAmount: Math.max(0, totalDeposited - totalWithdrawn),
    totalDeposited,
    totalWithdrawn,
    createdAt: goal.createdAt,
    updatedAt: goal.updatedAt,
  };
}
```

### Fix 3 — `getHomeData`: filter blendUSD from `vaultSymbolsToFetch`

- [ ] **Step 3.3 — Replace the two lines that build `vaultSymbolsToFetch` (lines 473–475)**

```typescript
// Before:
const vaultSymbolsToFetch = new Set<string>();
for (const symbol of vaultBalanceMap.keys()) vaultSymbolsToFetch.add(symbol);
for (const goal of goals) vaultSymbolsToFetch.add(goal.yieldTokenSymbol);

// After:
const vaultSymbolsToFetch = new Set<string>();
for (const symbol of vaultBalanceMap.keys()) vaultSymbolsToFetch.add(symbol);
for (const goal of goals) {
  if (goal.yieldTokenSymbol !== 'blendUSD') {
    vaultSymbolsToFetch.add(goal.yieldTokenSymbol);
  }
}
```

### Fix 4+8 — `getHomeData`: capture `blendApy` from Blend SDK and use it

- [ ] **Step 3.4 — Update the `blendData` block inside `getHomeData` (around line 430–449)**

Find this section:
```typescript
(async () => {
  if (this.blendSdk && blendAccount) {
    try {
      const sdk = this.blendSdk.getSdk();
      const client = sdk.forAccount(blendAccount.accountId);
      const [balance, returns] = await Promise.all([
        client.account.balance(),
        client.account.returns().catch(() => null),
      ]);
      return {
        balance: balance.total?.USD ?? balance.total?.usd ?? 0,
        earned: returns?.returns?.USD ?? returns?.returns?.usd ?? 0,
      };
    } catch (e) {
      this.logger.warn(`Failed to fetch Blend balance for home data: ${(e as Error).message}`);
    }
  }
  return { balance: 0, earned: 0 };
```

Replace with:
```typescript
(async () => {
  if (this.blendSdk && blendAccount) {
    try {
      const sdk = this.blendSdk.getSdk();
      const client = sdk.forAccount(blendAccount.accountId);
      const [balance, returns, yieldData] = await Promise.all([
        client.account.balance(),
        client.account.returns().catch(() => null),
        sdk.discover.yield().catch(() => null),
      ]);
      const rawApy = (yieldData?.yieldBreakdown?.[0]?.summary?.theoreticalOverall ?? 0) * 100;
      return {
        balance: balance.total?.USD ?? balance.total?.usd ?? 0,
        earned: returns?.returns?.USD ?? returns?.returns?.usd ?? 0,
        apy: rawApy > 0 ? rawApy : null,
      };
    } catch (e) {
      this.logger.warn(`Failed to fetch Blend balance for home data: ${(e as Error).message}`);
    }
  }
  return { balance: 0, earned: 0, apy: null };
```

- [ ] **Step 3.5 — Update the destructuring below the Promise.all (line ~451)**

```typescript
// Before:
const blendBalance = blendData.balance;
const blendEarned = blendData.earned;

// After:
const blendBalance = blendData.balance;
const blendEarned = blendData.earned;
const blendApy = blendData.apy;
```

- [ ] **Step 3.6 — Update `savingsAccounts` USD APY (line ~709–720)**

```typescript
// Before:
const savingsAccounts = [
  {
    currency: 'USD',
    balance: generalUsdSavingsUsd <= 0.01 ? 0 : generalUsdSavingsUsd,
    apy: vaultApyMap.get('yoUSD') ?? null,
  },
  ...
```

```typescript
// After:
const savingsAccounts = [
  {
    currency: 'USD',
    balance: generalUsdSavingsUsd <= 0.01 ? 0 : generalUsdSavingsUsd,
    apy: blendApy,
  },
  ...
```

- [ ] **Step 3.7 — Update per-goal APY in the `savingsGoals` map builder (line ~692)**

```typescript
// Before:
apy: vaultApyMap.get(g.yieldTokenSymbol) ?? null,

// After:
apy: g.protocol === 'blend' ? blendApy : (vaultApyMap.get(g.yieldTokenSymbol) ?? null),
```

Also add `protocol` to the returned goal object:
```typescript
// Add protocol field to the returned object:
protocol: g.protocol,
```

### Fix 5 — `getDetails`: branch on protocol for USD vs EUR

- [ ] **Step 3.8 — Replace the entire `getDetails` method (lines 966–994)**

```typescript
async getDetails(userId: string, goalId?: string, currency?: string) {
  // Determine whether this is Blend (USD) or Yo Protocol (EUR)
  let isBlend = false;

  if (goalId) {
    const goal = await this.prisma.savingsGoal.findFirst({
      where: { id: goalId, userId },
      select: { protocol: true },
    });
    if (!goal) throw new NotFoundException('Goal not found');
    isBlend = goal.protocol === 'blend';
  } else {
    // No goalId: use currency to determine — anything non-EUR is Blend
    isBlend = !currency || currency.toUpperCase() !== 'EUR';
  }

  if (isBlend) {
    const sdk = this.blendSdk.getSdk();
    const yieldData = await sdk.discover.yield().catch(() => null);
    const breakdown = yieldData?.yieldBreakdown?.[0];
    const summary = breakdown?.summary;
    const apy = summary?.theoreticalOverall ? summary.theoreticalOverall * 100 : 0;

    return {
      success: true,
      data: {
        pastPerformance: [
          { label: 'Current APY', value: apy > 0 ? `${apy.toFixed(2)}%` : 'N/A' },
          { label: 'Performance fees', value: 'None' },
        ],
        about: [
          { label: 'Provider', value: 'Blend', info: 'Non-custodial Yield Coordination Engine. Your savings live in your own isolated Gnosis Safe — no shared pools, no mixing.' },
          { label: 'Total value locked', value: 'N/A', info: 'Total savings across all Blend users.' },
          { label: 'Performance fees', value: '$0.00', info: 'You keep 100% of what you earn. No hidden fees.' },
        ],
        whereYourSavingsGo: (breakdown?.heldAssets ?? []).map((asset: any) => ({
          name: asset.name ?? asset.symbol ?? 'Unknown',
          pool: null,
          percentage: null,
          apy: null,
          icon: asset.logoUrl ?? asset.logo ?? null,
        })),
        howItWorks: HOW_IT_WORKS,
        security: SECURITY,
        comparison: this.buildComparison(undefined),
      },
    };
  }

  // EUR: existing Yo Protocol path (unchanged)
  const vaultSymbol = 'yoEUR';
  const vaults = await this.yieldPositions.getLiveApyData();
  const vault = vaults.find((v: VaultApyInfo) => v.symbol === vaultSymbol);

  return {
    success: true,
    data: {
      pastPerformance: this.buildPastPerformance(vault),
      about: this.buildAbout(vault),
      whereYourSavingsGo: this.buildAllocations(vault),
      howItWorks: HOW_IT_WORKS,
      security: SECURITY,
      comparison: this.buildComparison(vault),
    },
  };
}
```

### Fix 6 — `transferBetweenGoals`: use `blendUSD` for USD vaultSymbol

- [ ] **Step 3.9 — Replace line 118 in `transferBetweenGoals`**

```typescript
// Before:
const vaultSymbol = dto.currency.toUpperCase() === 'EUR' ? 'yoEUR' : 'yoUSD';

// After:
const vaultSymbol = dto.currency.toUpperCase() === 'EUR' ? 'yoEUR' : 'blendUSD';
```

### Fix 7 — `getDashboardSummary`: return Blend APY instead of yoUSD APY

- [ ] **Step 3.10 — Update `getDashboardSummary` blendBalance block and APY (lines 363–396)**

Find this block:
```typescript
const [positions, blendBalance] = await Promise.all([
  this.zerion.getEvmPositions(evmAddress).catch(() => []),
  (async () => {
    if (this.blendSdk && blendAccount) {
      try {
        const sdk = this.blendSdk.getSdk();
        const client = sdk.forAccount(blendAccount.accountId);
        const balance = await client.account.balance();
        return balance.total?.USD ?? balance.total?.usd ?? 0;
      } catch (e) {
        this.logger.warn(`Failed to fetch Blend balance for dashboard summary: ${(e as Error).message}`);
      }
    }
    return 0;
  })(),
]);
```

Replace with:
```typescript
const [positions, blendData] = await Promise.all([
  this.zerion.getEvmPositions(evmAddress).catch(() => []),
  (async () => {
    if (this.blendSdk && blendAccount) {
      try {
        const sdk = this.blendSdk.getSdk();
        const client = sdk.forAccount(blendAccount.accountId);
        const [balance, yieldData] = await Promise.all([
          client.account.balance(),
          sdk.discover.yield().catch(() => null),
        ]);
        const rawApy = (yieldData?.yieldBreakdown?.[0]?.summary?.theoreticalOverall ?? 0) * 100;
        return {
          balance: balance.total?.USD ?? balance.total?.usd ?? 0,
          apy: rawApy > 0 ? rawApy : null,
        };
      } catch (e) {
        this.logger.warn(`Failed to fetch Blend balance for dashboard summary: ${(e as Error).message}`);
      }
    }
    return { balance: 0, apy: null };
  })(),
]);
```

Then replace the two usages below it:
```typescript
// Before:
let totalBalance = 0;
for (const pos of positions) {
  if (pos.contractAddress && vaultAddressSet.has(pos.contractAddress.toLowerCase())) {
    totalBalance += pos.balanceUsd;
  }
}
totalBalance += blendBalance;

const yoUsdApy = liveApyData.find(v => v.symbol === 'yoUSD');
const baseApy = yoUsdApy?.apy7d ?? yoUsdApy?.apy ?? null;

return { totalBalance, baseApy };

// After:
let totalBalance = 0;
for (const pos of positions) {
  if (pos.contractAddress && vaultAddressSet.has(pos.contractAddress.toLowerCase())) {
    totalBalance += pos.balanceUsd;
  }
}
totalBalance += blendData.balance;

return { totalBalance, baseApy: blendData.apy };
```

### Fix 9 — Stale `'yoUSD'` fallback in `formattedInternalTxs` builder

- [ ] **Step 3.11 — Update the fallback in formattedInternalTxs (line ~838)**

```typescript
// Before:
const vaultSymbol = goal?.yieldTokenSymbol || tx.fromAsset || tx.toAsset || 'yoUSD';

// After:
const vaultSymbol = goal?.yieldTokenSymbol || tx.fromAsset || tx.toAsset || null;
```

- [ ] **Step 3.12 — Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: no output (exit code 0).

- [ ] **Step 3.13 — Commit**

```bash
git add src/modules/savings/savings.service.ts
git commit -m "feat(savings): migrate USD savings logic to Blend protocol branching"
```

---

## Task 4: `transaction-history.handler.ts` — Fixes 10, 18, 19

**Files:**
- Modify: `src/modules/balances/handlers/transaction-history.handler.ts`

### Fix 10 — Add `blendUSD` icon fallback in `resolveSavingsIcon`

- [ ] **Step 4.1 — Add blendUSD to the hardcoded icon fallbacks in `resolveSavingsIcon` (around line 379)**

Find this block:
```typescript
if (lower === 'eurc' || lower === 'unknown:euroc') {
  return 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/EURC.svg';
}
if (lower === 'btc' || lower === 'ob:btc') {
  return 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/bitcoin.svg';
}
```

Add after those two blocks:
```typescript
if (lower === 'blendusd') {
  return 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/blend.svg';
}
```

### Fix 18 — Replace stale `'yoUSD'` fallback in `formatSavingsTx` (line ~417)

- [ ] **Step 4.2 — Change fallback on line 417**

```typescript
// Before:
const vaultSymbol = tx.savingsGoal?.yieldTokenSymbol || fromAsset || toAsset || 'yoUSD';

// After:
const vaultSymbol = tx.savingsGoal?.yieldTokenSymbol || fromAsset || toAsset || null;
```

### Fix 18 (second occurrence) — Same pattern in the second formatter around line 1589

- [ ] **Step 4.3 — Find and replace the second `|| 'yoUSD'` fallback**

Search for: `|| tx.fromAsset || tx.toAsset || 'yoUSD'`

```typescript
// Before:
const vaultSymbol = tx.savingsGoal?.yieldTokenSymbol || tx.fromAsset || tx.toAsset || 'yoUSD';

// After:
const vaultSymbol = tx.savingsGoal?.yieldTokenSymbol || tx.fromAsset || tx.toAsset || null;
```

### Fix 19 — Update stale comment at line 1215

- [ ] **Step 4.4 — Update the stale comment**

```typescript
// Before:
// The `Token` table only carries `yoUSD`; the rest of the savings
// vault tokens (`yoEUR`, `yoBTC`, `yoETH`, `yoGOLD`) live in
// `YieldToken`. Pull any of those that the activity feed referenced
// but weren't covered above so savings rows always render with a
// proper logo.

// After:
// yoUSD and yoEUR are now in the Token table (sync-yield-tokens-to-token-table script).
// yoBTC, yoETH, yoGOLD live only in YieldToken. Pull any yo* symbols
// the activity feed referenced but weren't covered above so savings
// rows always render with a proper logo.
```

- [ ] **Step 4.5 — Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: no output.

- [ ] **Step 4.6 — Commit**

```bash
git add src/modules/balances/handlers/transaction-history.handler.ts
git commit -m "fix(activity): add blendUSD icon fallback, remove stale yoUSD defaults"
```

---

## Task 5: Remove YOUSD/YOEUR Hardcodes from Portfolio & Dashboard

**Files:**
- Modify: `src/modules/dashboard/handlers/dashboard.handler.ts`
- Modify: `src/modules/balances/handlers/asset-details.handler.ts`
- Modify: `src/modules/balances/handlers/holding-assets.handler.ts`

### Fix 13 — `dashboard.handler.ts`: remove YOUSD→stocks

- [ ] **Step 5.1 — Remove the YOUSD hardcode block (line 213–215)**

Find and remove this entire block:
```typescript
if (sym === 'YOUSD') {
  stocksValue += balanceUsd;
} else if (cashGroup?.symbols.has(sym) || cashGroup?.assetIds.has(aid)) {
```

Replace with just:
```typescript
if (cashGroup?.symbols.has(sym) || cashGroup?.assetIds.has(aid)) {
```

### Fix 14 — `asset-details.handler.ts`: remove YOUSD hardcodes

- [ ] **Step 5.2 — Remove both YOUSD special cases (lines 100–105 in `filterAssetsByGroup`)**

Find and remove:
```typescript
if (groupKey === 'stocks' && symUpper === 'YOUSD') {
  return true;
}
if (groupKey === 'earn' && symUpper === 'YOUSD') {
  return false;
}
```

### Fix 15 — `holding-assets.handler.ts`: remove YOUSD hardcodes in `filterAssetsByCategory`

- [ ] **Step 5.3 — Remove both YOUSD special cases (lines 344–349 in `filterAssetsByCategory`)**

Find and remove:
```typescript
if (categoryKey === 'stocks' && symUpper === 'YOUSD') {
  return true;
}
if (categoryKey === 'earn' && symUpper === 'YOUSD') {
  return false;
}
```

### Fix 16 — `holding-assets.handler.ts`: remove YOUSD/YOEUR hardcoded fallbacks

- [ ] **Step 5.4 — Fix `isKnownAsset` line (line 175)**

```typescript
// Before:
const isKnownAsset = !!(knownToken || knownByAssetId) || symbolUpper === 'YOUSD' || symbolUpper === 'YOEUR';

// After:
const isKnownAsset = !!(knownToken || knownByAssetId);
```

- [ ] **Step 5.5 — Remove YOEUR hardcoded name/logo/decimals/priceApiId (lines 198–212)**

```typescript
// Before:
assetMap.set(assetId, {
  symbol: position.symbol,
  name: knownToken?.name || (symbolUpper === 'YOEUR' ? 'Yield Optimizer EUR' : position.name),
  logoUrl: knownToken?.logoUrl || (symbolUpper === 'YOEUR' ? 'https://pub-8510aec09b7640f389010c481a0a453e.r2.dev/logos/yoEUR.svg' : position.logoUrl),
  balance: formatBalanceString(position.balance),
  balanceUsd: position.balanceUsd,
  decimals: knownToken?.decimals || (symbolUpper === 'YOEUR' ? 6 : position.decimals),
  supportedForSwap: true,
  assetId,
  chainId: position.chainId,
  contractAddress: position.contractAddress,
  isKnownAsset,
  isSolana: position.isSolana,
  price: position.priceUsd ? position.priceUsd * usdToTargetRate : null,
  priceChange24h: position.priceChange24h,
  priceApiId: knownToken?.priceApiId || (symbolUpper === 'YOEUR' ? 'yield-optimizer-eur' : null),
});

// After (fallback to Zerion data for any non-Token-table asset):
assetMap.set(assetId, {
  symbol: position.symbol,
  name: knownToken?.name || position.name,
  logoUrl: knownToken?.logoUrl || position.logoUrl,
  balance: formatBalanceString(position.balance),
  balanceUsd: position.balanceUsd,
  decimals: knownToken?.decimals || position.decimals,
  supportedForSwap: true,
  assetId,
  chainId: position.chainId,
  contractAddress: position.contractAddress,
  isKnownAsset,
  isSolana: position.isSolana,
  price: position.priceUsd ? position.priceUsd * usdToTargetRate : null,
  priceChange24h: position.priceChange24h,
  priceApiId: knownToken?.priceApiId || null,
});
```

- [ ] **Step 5.6 — Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: no output.

- [ ] **Step 5.7 — Commit**

```bash
git add src/modules/dashboard/handlers/dashboard.handler.ts \
        src/modules/balances/handlers/asset-details.handler.ts \
        src/modules/balances/handlers/holding-assets.handler.ts
git commit -m "refactor: remove hardcoded YOUSD/YOEUR special cases, use Token table"
```

---

## Task 6: `transfers.service.ts` — Fix 17: Remove YOUSD/YOEUR Sendable Exception

**Files:**
- Modify: `src/modules/transfers/transfers.service.ts`

- [ ] **Step 6.1 — Remove the YOUSD/YOEUR exception from the yield asset exclusion filter (lines 437–443)**

Find:
```typescript
// Skip yield/vault tokens (yoUSD/yoEUR/yoETH/yoBTC/yoGOLD/...). They
// are managed via the savings flow and must not appear as sendable.
// Exception: allow legacy yoUSD/yoEUR to exist as sendable/swappable wallet tokens.
const symUpper = holding.symbol.toUpperCase();
if (symUpper !== 'YOUSD' && symUpper !== 'YOEUR') {
  if (yieldAssetIds.has(holding.assetId.toLowerCase())) continue;
}
```

Replace with:
```typescript
// Skip yield/vault tokens (yoUSD/yoEUR/yoETH/yoBTC/yoGOLD/...).
// They are managed via the savings/earn flow and must not appear as sendable.
if (yieldAssetIds.has(holding.assetId.toLowerCase())) continue;
```

- [ ] **Step 6.2 — Verify TypeScript compiles**

```bash
npx tsc --noEmit
```

Expected: no output.

- [ ] **Step 6.3 — Commit**

```bash
git add src/modules/transfers/transfers.service.ts
git commit -m "fix(transfers): remove legacy yoUSD/yoEUR sendable exception, treat as yield tokens"
```

---

## Task 7: Run Migration Script + Final Verification

- [ ] **Step 7.1 — Run the Token table sync script** (if not already done in Task 2)

```bash
npx ts-node -r tsconfig-paths/register prisma/scripts/sync-yield-tokens-to-token-table.ts
```

- [ ] **Step 7.2 — Full TypeScript compile check**

```bash
npx tsc --noEmit
```

Expected: no output (exit code 0).

- [ ] **Step 7.3 — Verify the database state**

Run this check to confirm `SavingsGoal` table is empty and has the new column:

```bash
npx prisma db execute --stdin <<'SQL'
SELECT COUNT(*) AS goal_count FROM "SavingsGoal";
SELECT column_name, column_default, is_nullable
FROM information_schema.columns
WHERE table_name = 'SavingsGoal' AND column_name = 'protocol';
SQL
```

Expected: `goal_count = 0`, and the `protocol` column exists with default `'yo'`.

- [ ] **Step 7.4 — Verify yoUSD + yoEUR are in Token table**

```bash
npx prisma db execute --stdin <<'SQL'
SELECT symbol, "supportedForSwap", "priceApiId", g.key AS group_key
FROM "Token" t
JOIN "Group" g ON t."groupId" = g.id
WHERE t.symbol IN ('yoUSD', 'yoEUR');
SQL
```

Expected: both rows present, `supportedForSwap = false`, `priceApiId = NULL`, `group_key = 'earn'`.

- [ ] **Step 7.5 — Manual smoke test: create a USD goal**

```bash
curl -X POST http://localhost:3000/api/v1/savings/goals \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Test Goal","icon":"🎯","targetAmount":100,"currency":"USD"}'
```

Expected response includes:
```json
{
  "protocol": "blend",
  "yieldTokenSymbol": "blendUSD",
  "currency": "USD"
}
```

- [ ] **Step 7.6 — Manual smoke test: create a EUR goal**

```bash
curl -X POST http://localhost:3000/api/v1/savings/goals \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"name":"EUR Goal","icon":"💶","targetAmount":100,"currency":"EUR"}'
```

Expected response includes:
```json
{
  "protocol": "yo",
  "yieldTokenSymbol": "yoEUR",
  "currency": "EUR"
}
```

- [ ] **Step 7.7 — Manual smoke test: GET savings home**

```bash
curl http://localhost:3000/api/v1/savings/home \
  -H "Authorization: Bearer <token>"
```

Expected: `savingsAccounts[0].apy` is Blend APY (a number like `4.28`), not `null` and not a stale yoUSD APY.

- [ ] **Step 7.8 — Final commit with tag**

```bash
git add -A
git commit -m "chore: savings blend migration complete — all 19 surfaces fixed"
```

---

## Self-Review Checklist

| Spec requirement | Task |
|---|---|
| Add `protocol` column to `SavingsGoal` | Task 1 |
| Dev/test data wipe (SavingsGoal only, FK SET NULL) | Task 1 Step 1.2 |
| `createGoal` sets `protocol='blend'` + `yieldTokenSymbol='blendUSD'` for USD | Task 3 Step 3.1 |
| `getGoal` branches on `protocol` — Blend DB query vs YO API | Task 3 Step 3.2 |
| `getHomeData` skips yoUSD in `vaultSymbolsToFetch` | Task 3 Step 3.3 |
| `getHomeData` USD APY → Blend APY | Task 3 Steps 3.4–3.6 |
| Per-goal APY uses `blendApy` for Blend goals | Task 3 Step 3.7 |
| `getDetails` USD → Blend SDK vault info | Task 3 Step 3.8 |
| `transferBetweenGoals` uses `blendUSD` for USD | Task 3 Step 3.9 |
| `getDashboardSummary` APY → Blend APY | Task 3 Step 3.10 |
| Stale `'yoUSD'` fallback in internal txs removed | Task 3 Step 3.11 |
| `blendUSD` icon fallback in activity feed | Task 4 Step 4.1 |
| Stale `|| 'yoUSD'` fallbacks in tx-history removed | Task 4 Steps 4.2–4.3 |
| Stale comment updated | Task 4 Step 4.4 |
| yoUSD + yoEUR added to Token table (earn group, supportedForSwap=false) | Task 2 |
| Remove YOUSD→stocks hardcode in dashboard | Task 5 Step 5.1 |
| Remove YOUSD hardcodes in asset-details | Task 5 Step 5.2 |
| Remove YOUSD hardcodes in holding-assets filterAssetsByCategory | Task 5 Step 5.3 |
| Remove YOUSD/YOEUR fallbacks in holding-assets asset builder | Task 5 Steps 5.4–5.5 |
| Remove wrong `'yield-optimizer-eur'` priceApiId | Task 5 Step 5.5 |
| Remove YOUSD/YOEUR sendable exception in transfers | Task 6 Step 6.1 |
| EUR savings (yoEUR, Yo Protocol) untouched | Confirmed — all EUR branches preserved |
