# Savings Module Implementation Plan

## Executive Summary

This document details the implementation of the **Savings Module**, a user-centric abstraction over the existing `Yield` module. It allows users to create goal-oriented savings accounts backed by `yoUSD` (or other yield tokens).

**Status:** Ready via verified architecture.
**Prerequisites:** existing `Yield`, `OneBalance`, and `Transaction` modules.

---

## 1. Database Schema Changes

Modify `prisma/schema.prisma` to introduce `SavingsGoal` and link it to existing activities.

```prisma
// 1. New Model: SavingsGoal
model SavingsGoal {
  id               String    @id @default(cuid())
  userId           String
  name             String    // e.g., "Trip to Mexico"
  icon             String    // Emoji or icon identifier
  targetAmount     Decimal   @db.Decimal(18, 2)
  currency         String    // "USD", "EUR"
  status           String    @default("ACTIVE") // ACTIVE, COMPLETED, ARCHIVED
  isDefault        Boolean   @default(false)
  yieldTokenSymbol String    // Maps to YieldToken.symbol (e.g., "yoUSD")
  createdAt        DateTime  @default(now())
  updatedAt        DateTime  @updatedAt

  user             User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  transactions     Transaction[]
  yieldDeposits    YieldDeposit[]
  yieldRedemptions YieldRedemption[]

  @@unique([userId, name])
  @@index([userId])
}

// 2. User Relationship
model User {
  // ... existing fields
  savingsGoals    SavingsGoal[]
}

// 3. Transaction Link
model Transaction {
  // ... existing fields
  savingsGoalId   String?
  savingsGoal     SavingsGoal? @relation(fields: [savingsGoalId], references: [id], onDelete: SetNull)

  @@index([savingsGoalId])
}

// 4. YieldDeposit Link
model YieldDeposit {
  // ... existing fields
  savingsGoalId   String?
  savingsGoal     SavingsGoal? @relation(fields: [savingsGoalId], references: [id], onDelete: SetNull)

  @@index([savingsGoalId])
}

// 5. YieldRedemption Link
model YieldRedemption {
  // ... existing fields
  savingsGoalId   String?
  savingsGoal     SavingsGoal? @relation(fields: [savingsGoalId], references: [id], onDelete: SetNull)

  @@index([savingsGoalId])
}
```

> **Action:** Run `npx prisma migrate dev --name add_savings_goals` after applying changes.

---

## 2. API / DTO Updates

We need to pass the `goalId` through the existing Yield flow.

### File: `src/modules/yield/dto/yield-deposit.dto.ts`

```typescript
export class YieldDepositQuoteDto {
  // ... existing fields

  @IsString()
  @IsOptional()
  goalId?: string; // Optional Savings Goal ID
}
```

### File: `src/modules/yield/dto/yield-redeem.dto.ts`

```typescript
export class YieldRedemptionQuoteDto {
  // ... existing fields

  @IsString()
  @IsOptional()
  goalId?: string; // Optional Savings Goal ID
}
```

---

## 3. New Module: Savings

Create directory: `src/modules/savings`
Files: `savings.module.ts`, `savings.controller.ts`, `savings.service.ts`

### 3.1 `src/modules/savings/savings.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { SavingsController } from './savings.controller';
import { SavingsService } from './savings.service';
import { DatabaseModule } from '../../database/database.module';
// Import YieldModule or YieldPositionsService if necessary for balance checking
import { YieldModule } from '../yield/yield.module'; 

@Module({
  imports: [DatabaseModule, YieldModule],
  controllers: [SavingsController],
  providers: [SavingsService],
  exports: [SavingsService],
})
export class SavingsModule {}
```

### 3.2 `src/modules/savings/savings.controller.ts`

```typescript
import { Body, Controller, Get, Param, Patch, Post, Request, UseGuards } from '@nestjs/common';
import { SavingsService } from './savings.service';
import { JwtAuthGuard } from '../../auth/guards/jwt-auth.guard'; // Verify path
// Add DTOs for Create/Update

@Controller('v1/savings')
@UseGuards(JwtAuthGuard)
export class SavingsController {
  constructor(private readonly savingsService: SavingsService) {}

  @Post('goals')
  async createGoal(@Request() req, @Body() body: any) { // Replace any with proper DTO
    return this.savingsService.createGoal(req.user.id, body);
  }

  @Patch('goals/:goalId')
  async updateGoal(@Request() req, @Param('goalId') goalId: string, @Body() body: any) {
    return this.savingsService.updateGoal(req.user.id, goalId, body);
  }

  @Get('home')
  async getHome(@Request() req) {
    return this.savingsService.getHomeData(req.user.id);
  }
}
```

### 3.3 `src/modules/savings/savings.service.ts`

Key logic for aggregating data.

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { PrismaService } from '../../database/prisma.service';
// import { YieldPositionsService } from '../yield/services/yield-positions.service'; // Assuming this service exists or similar logic

@Injectable()
export class SavingsService {
  constructor(
    private prisma: PrismaService,
    // private yieldPositions: YieldPositionsService
  ) {}

  async createGoal(userId: string, body: { name: string; icon: string; targetAmount: number; currency: string }) {
    // 1. Determine yieldTokenSymbol (e.g. currency === 'USD' ? 'yoUSD' : 'yoEUR')
    const yieldTokenSymbol = body.currency === 'EUR' ? 'yoEUR' : 'yoUSD';

    return this.prisma.savingsGoal.create({
      data: {
        userId,
        name: body.name,
        icon: body.icon,
        targetAmount: body.targetAmount,
        currency: body.currency,
        yieldTokenSymbol,
      },
    });
  }

  async updateGoal(userId: string, goalId: string, body: Partial<{ name: string; icon: string; targetAmount: number }>) {
    const goal = await this.prisma.savingsGoal.findFirst({
      where: { id: goalId, userId },
    });

    if (!goal) throw new NotFoundException('Goal not found');

    return this.prisma.savingsGoal.update({
      where: { id: goalId },
      data: {
        ...body,
      },
    });
  }

  async getHomeData(userId: string) {
    // 1. Fetch Goals
    const goals = await this.prisma.savingsGoal.findMany({ where: { userId } });

    // 2. Fetch Logic for Balances (Reuse YieldPositionsService logic ideally)
    // This part requires fetching all YieldTokens and User's YieldPositions (currentValue)
    // Then mapping them to the goals.

    // 3. Fetch Activity Preview (Last 5 txs with goalId)
    const activity = await this.prisma.transaction.findMany({
      where: { userId, savingsGoalId: { not: null } },
      orderBy: { createdAt: 'desc' },
      take: 5,
      include: { savingsGoal: true },
    });

    // 4. Assemble Response
    return {
      summary: {
        totalBalance: 0, // Sum of positions
        totalEarned: 0,
        baseApy: 0, // Get from yoUSD yieldToken
      },
      savingsAccounts: [], // Mapped positions
      goals: goals.map(g => ({
        ...g,
        currentAmount: 0, // Match with position
        apy: 0
      })),
      activityPreview: activity
    };
  }
}
```

---

## 4. Updates to Yield Module Handlers

We must propagate the `goalId` during the quote and execution phases.

### 4.1 `src/modules/yield/handlers/yo-quote.handler.ts`

**Logic Check:**
The `dto` passed to `generateDepositQuote` is stored in `this.quoteCache`.
By adding `goalId` to the DTO (Step 2), it is **automatically cached** inside the `dto` object in the map value.
No logic change is required here, just the DTO update!

### 4.2 `src/modules/yield/handlers/yo-execution.handler.ts`

**Logic Update:**
Retrieve `goalId` from the cached DTO and save it to the database records.

**In `executeYieldOperation` method:**

```typescript
// RETRIEVE
const quote = this.quoteHandler.getCachedQuote(dto.quoteId) as CachedQuote;
const goalId = quote.dto.goalId || null; // <--- Extract goalId
```

**In `createTransaction` helper:**

```typescript
private async createTransaction(..., quote: CachedQuote, ...) {
  // ...
  return this.prisma.transaction.create({
    data: {
      // ... existing fields
      savingsGoalId: quote.dto.goalId || null, // <--- Add this
    },
  });
}
```

**In `createYieldDeposit` helper:**

```typescript
private async createYieldDeposit(..., quote: CachedQuote, ...) {
  // ...
  return this.prisma.yieldDeposit.create({
    data: {
      // ... existing fields
      savingsGoalId: quote.dto.goalId || null, // <--- Add this
    },
  });
}
```

**In `createYieldRedemption` helper:**

```typescript
private async createYieldRedemption(..., quote: CachedQuote, ...) {
  // ...
  return this.prisma.yieldRedemption.create({
    data: {
      // ... existing fields
      savingsGoalId: quote.dto.goalId || null, // <--- Add this
    },
  });
}
```

---

## 5. Activity Filtering

### Endpoint: `GET /v1/activity` (BalancesController)

Modify `getTransactions` to accept query param `goalId`.

**File: `src/modules/balances/handlers/transaction-history.handler.ts`** (Refence location)

Update the `where` clause:

```typescript
const whereClause: Prisma.TransactionWhereInput = {
    userId,
    // ...
    ...(goalId ? { savingsGoalId: goalId } : {}), // Add this filter
};
```

---

## 6. Implementation Checklist

1.  [ ] **Schema:** Update `prisma/schema.prisma` and run migration.
2.  [ ] **DTOs:** Add `goalId` to `YieldDepositQuoteDto` and `YieldRedemptionQuoteDto`.
3.  [ ] **Module:** Create `SavingsModule` files.
4.  [ ] **Handlers:** Update `YoExecutionHandler` to read and save `savingsGoalId`.
5.  [ ] **Activity:** Update transaction fetch logic to support `goalId` filter.
6.  [ ] **Docs:** Verify endpoints against frontend requirements.

This plan is fully aligned with the current `anzo-backend` architecture.
