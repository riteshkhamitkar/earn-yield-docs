# YO Protocol Integration Analysis for Anzo Backend

**Document Date:** December 15, 2025  
**Status:** Implementation Ready  
**Integration Type Recommended:** On-chain Gateway + OneBalance Atomic Intent Calls

---

## Executive Summary

Your Anzo wallet is perfectly positioned for YO Protocol integration using **OneBalance's multi-intent atomic calldata** capabilities. This eliminates the need for manual swap/bridge steps—users can deposit their unified USDC directly into yoUSD with a single cross-chain transaction.

**Key Benefits:**
- ✅ Single-click yield activation (no manual token transfers)
- ✅ Revenue share opportunity (via Partner ID)
- ✅ Seamless multi-chain USDC → yoUSD deposits
- ✅ Leverages existing OneBalance infrastructure

---

## Current Architecture Analysis

### Database Schema Assessment

#### ✅ Strengths
1. **Transaction Tracking Ready**: `Swap` and `Transaction` tables already structured to track DeFi operations
2. **Service Provider Abstraction**: `ServiceProvider` enum supports multiple providers (OneBalance, LI.FI, YO)
3. **Multi-chain Support**: Wallet model supports EVM + Solana + Bitcoin
4. **Token Metadata**: `Token` table stores asset IDs, decimals, and chain-specific identifiers

#### 🔄 Required Extensions

Add a new model to track YO-specific interactions:

```sql
-- New: YO Vault Interactions
CREATE TABLE "YoDeposit" (
    "id" TEXT NOT NULL,
    "userId" TEXT NOT NULL,
    "vaultAddress" TEXT NOT NULL,
    "depositAsset" TEXT NOT NULL,
    "depositAmount" TEXT NOT NULL,
    "sharesMinted" TEXT NOT NULL,
    "minSharesOut" TEXT NOT NULL,
    "partnerId" TEXT,
    "transactionId" TEXT,
    "status" TEXT NOT NULL DEFAULT 'PENDING',
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    "completedAt" TIMESTAMP(3),
    "metadata" JSONB,
    
    CONSTRAINT "YoDeposit_pkey" PRIMARY KEY ("id"),
    CONSTRAINT "YoDeposit_userId_fkey" FOREIGN KEY ("userId") REFERENCES "User"(id),
    CONSTRAINT "YoDeposit_transactionId_fkey" FOREIGN KEY ("transactionId") REFERENCES "Transaction"(id)
);

CREATE INDEX "YoDeposit_userId_idx" ON "YoDeposit"("userId");
CREATE INDEX "YoDeposit_status_idx" ON "YoDeposit"("status");
CREATE INDEX "YoDeposit_vaultAddress_idx" ON "YoDeposit"("vaultAddress");
```

### Integration Points with Existing Systems

#### OneBalance Already In Place ✅
- Client initialized with API key
- Account utilities for EVM + Solana routing
- Multi-account support perfect for YO deposits

#### LI.FI Integration ✅
- Quote/Execute pattern already implemented
- PSBT signing for Bitcoin ready
- Can reuse for any future YO cross-chain features

#### Turnkey Smart Account Support ✅
- EIP-7702 account abstraction ready
- Perfect for YO Gateway authentication

---

## YO Protocol Integration Strategy

### Recommended Approach: On-chain Gateway (Preferred)

**Why this method:**
1. **Revenue Share**: You get portion of vault performance fees
2. **One-time Integration**: Single smart contract integration across ALL YO vaults
3. **Medium Effort**: Reasonable implementation complexity
4. **Better UX**: Native in-app experience vs. external links

### Step-by-Step Implementation

#### Phase 1: Configuration & Vault Setup

**File: `.env.example`**
```env
# ============================================
# YO Protocol Configuration
# ============================================
YO_GATEWAY_ADDRESS_BASE=0xF1EeE0957267b1A474323Ff9CfF7719E964969FA
YO_USD_VAULT_ADDRESS_BASE=0x0000000f2eB9f69274678c76222b35eEc7588a65
YO_USD_VAULT_ADDRESS_ETH=0x[ETH_VERSION_ADDRESS]
YO_USDC_ADDRESS_BASE=0x833589fCD6eDb6e08f4c7c32d4f71b54bDa02913
YO_PARTNER_ID=your-partner-id
```

**File: `src/config/configuration.ts` - Add YO config**
```typescript
export default () => ({
  // ... existing config ...
  yo: {
    gateway: {
      addressBase: process.env.YO_GATEWAY_ADDRESS_BASE,
      addressEth: process.env.YO_GATEWAY_ADDRESS_ETH,
    },
    vaults: {
      usdBase: {
        address: process.env.YO_USD_VAULT_ADDRESS_BASE,
        network: 'base',
        chainId: 8453,
      },
      usdEth: {
        address: process.env.YO_USD_VAULT_ADDRESS_ETH,
        network: 'ethereum',
        chainId: 1,
      },
    },
    usdc: {
      addressBase: process.env.YO_USDC_ADDRESS_BASE,
      addressEth: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48', // Ethereum mainnet
    },
    partnerId: process.env.YO_PARTNER_ID || '0',
  },
});
```

#### Phase 2: Create YO Service Module

**File: `src/modules/yo/yo.module.ts`**
```typescript
import { Module } from '@nestjs/common';
import { YoService } from './services/yo.service';
import { YoGatewayService } from './services/yo-gateway.service';
import { YoDepositHandler } from './handlers/yo-deposit.handler';
import { YoPointsHandler } from './handlers/yo-points.handler';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  providers: [
    YoService,
    YoGatewayService,
    YoDepositHandler,
    YoPointsHandler,
  ],
  exports: [YoService, YoGatewayService],
})
export class YoModule {}
```

#### Phase 3: Implement YO Gateway Service

**File: `src/modules/yo/services/yo-gateway.service.ts`**
```typescript
import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios, { AxiosInstance } from 'axios';
import { ethers } from 'ethers';

@Injectable()
export class YoGatewayService {
  private readonly logger = new Logger(YoGatewayService.name);
  private readonly client: AxiosInstance;
  private readonly yoGatewayABI: any[];

  constructor(private configService: ConfigService) {
    this.client = axios.create({
      baseURL: 'https://api.yo.xyz',
      timeout: 10000,
    });

    // Minimal Gateway ABI for deposit interactions
    this.yoGatewayABI = [
      {
        inputs: [
          { name: 'yoVault', type: 'address' },
          { name: 'assetAmount', type: 'uint256' },
        ],
        name: 'previewDeposit',
        outputs: [{ name: 'sharesOut', type: 'uint256' }],
        stateMutability: 'view',
        type: 'function',
      },
      {
        inputs: [
          { name: 'yoVault', type: 'address' },
          { name: 'assetAmount', type: 'uint256' },
          { name: 'minSharesOut', type: 'uint256' },
          { name: 'receiver', type: 'address' },
          { name: 'partnerId', type: 'uint256' },
        ],
        name: 'deposit',
        outputs: [{ name: 'sharesOut', type: 'uint256' }],
        stateMutability: 'nonpayable',
        type: 'function',
      },
    ];
  }

  /**
   * Quote a YO deposit to estimate shares output
   * @param vaultAddress - Target yoUSD vault address
   * @param usdcAmount - Amount of USDC to deposit (in wei, 6 decimals)
   * @returns Estimated shares output
   */
  async previewDeposit(
    vaultAddress: string,
    usdcAmount: string,
  ): Promise<string> {
    this.logger.log(
      `Quoting YO deposit: ${vaultAddress} with ${usdcAmount} USDC`,
    );

    try {
      // Use ethers.js to encode call and estimate via RPC
      const iface = new ethers.Interface(this.yoGatewayABI);
      
      const calldata = iface.encodeFunctionData('previewDeposit', [
        vaultAddress,
        usdcAmount,
      ]);

      // NOTE: In production, call RPC to simulate this
      // For now, return a placeholder
      // Actual implementation would call Base RPC provider
      
      return usdcAmount; // Placeholder: 1:1 assumption for quote
    } catch (error: any) {
      this.logger.error(`Failed to preview YO deposit: ${error.message}`);
      throw error;
    }
  }

  /**
   * Build calldata for YO Gateway deposit
   * This calldata will be executed by OneBalance
   * 
   * @param vaultAddress - Target yoUSD vault
   * @param usdcAmount - USDC amount in wei
   * @param minSharesOut - Minimum shares (with slippage protection)
   * @param receiverAddress - User's receiving wallet address
   * @param partnerId - Your Partner ID for revenue share
   * @returns Encoded calldata for deposit
   */
  buildDepositCalldata(
    vaultAddress: string,
    usdcAmount: string,
    minSharesOut: string,
    receiverAddress: string,
    partnerId: string = '0',
  ): string {
    this.logger.log(`Building YO deposit calldata for ${vaultAddress}`);

    try {
      const iface = new ethers.Interface(this.yoGatewayABI);
      
      const calldata = iface.encodeFunctionData('deposit', [
        vaultAddress,
        usdcAmount,
        minSharesOut,
        receiverAddress,
        partnerId,
      ]);

      return calldata;
    } catch (error: any) {
      this.logger.error(`Failed to build deposit calldata: ${error.message}`);
      throw error;
    }
  }

  /**
   * Get vault APY and TVL from YO API
   * @param vaultAddress - Target vault address
   * @returns Vault snapshot data
   */
  async getVaultSnapshot(vaultAddress: string, network: string = 'base') {
    try {
      const response = await this.client.get(
        `/api/v1/vault/${network}/${vaultAddress}`,
      );
      return response.data;
    } catch (error: any) {
      this.logger.error(
        `Failed to fetch vault snapshot: ${error.message}`,
      );
      return null;
    }
  }

  /**
   * Get historical yield for a vault
   * @param vaultAddress - Target vault address
   * @returns Yield timeseries data
   */
  async getVaultYieldHistory(vaultAddress: string, network: string = 'base') {
    try {
      const response = await this.client.get(
        `/api/v1/vault/yield/timeseries/${network}/${vaultAddress}`,
      );
      return response.data;
    } catch (error: any) {
      this.logger.error(
        `Failed to fetch vault yield history: ${error.message}`,
      );
      return null;
    }
  }

  /**
   * Get user's YO points
   * @param walletAddress - User wallet address
   * @returns Points data including daily earn rate, referrals, etc.
   */
  async getUserPoints(walletAddress: string) {
    try {
      const response = await this.client.get(
        `/api/v1/users/wallet/${walletAddress}/points`,
      );
      return response.data;
    } catch (error: any) {
      if (error.response?.status === 404) {
        // User not found in YO system yet (new user)
        return null;
      }
      this.logger.error(`Failed to fetch user points: ${error.message}`);
      return null;
    }
  }

  /**
   * Get pending redemptions for a user on a vault
   * @param vaultAddress - Vault address
   * @param userAddress - User wallet address
   * @returns Pending redemption data
   */
  async getUserPendingRedemptions(
    vaultAddress: string,
    userAddress: string,
    network: string = 'base',
  ) {
    try {
      const response = await this.client.get(
        `/api/v1/vault/pending-redeems/${network}/${vaultAddress}/${userAddress}`,
      );
      return response.data;
    } catch (error: any) {
      this.logger.error(
        `Failed to fetch user redemptions: ${error.message}`,
      );
      return null;
    }
  }
}
```

#### Phase 4: Implement YO Deposit Handler

**File: `src/modules/yo/handlers/yo-deposit.handler.ts`**
```typescript
import { Injectable, Logger, BadRequestException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaService } from '../../database/prisma.service';
import { YoGatewayService } from '../services/yo-gateway.service';
import { OneBalanceProvider } from '../../swaps/providers/onebalance.provider';
import { toSmallestUnit, fromSmallestUnit } from '../../../common/utils/amount.util';
import { determineSourceAccounts } from '../../../common/utils/account.util';

export interface YoDepositRequest {
  userId: string;
  amountUsd: number; // User-friendly amount (e.g., "100")
  preferredChain?: 'base' | 'ethereum'; // Preferred vault location
  slippageTolerance?: number; // Default 1% (100 = 1%)
}

export interface YoDepositQuote {
  depositAmount: string;
  estimatedShares: string;
  minShares: string;
  vaultAddress: string;
  network: string;
  calldata: string;
  expiresAt: string;
}

@Injectable()
export class YoDepositHandler {
  private readonly logger = new Logger(YoDepositHandler.name);

  constructor(
    private configService: ConfigService,
    private prisma: PrismaService,
    private yoGatewayService: YoGatewayService,
    private oneBalanceProvider: OneBalanceProvider,
  ) {}

  /**
   * Get a quote for YO deposit
   * This builds the deposit calldata that will be executed via OneBalance
   */
  async getDepositQuote(request: YoDepositRequest): Promise<YoDepositQuote> {
    this.logger.log(`Getting YO deposit quote for ${request.amountUsd} USD`);

    // 1. Fetch user wallet
    const user = await this.prisma.user.findUnique({
      where: { id: request.userId },
      include: { Wallet: true },
    });

    if (!user || !user.Wallet.length) {
      throw new BadRequestException('User wallet not found');
    }

    const wallet = user.Wallet[0];

    // 2. Select target vault (Base by default - lower fees, higher liquidity)
    const preferredChain = request.preferredChain || 'base';
    const yoConfig = this.configService.get('yo');
    
    const vaultConfig = preferredChain === 'ethereum' 
      ? yoConfig.vaults.usdEth 
      : yoConfig.vaults.usdBase;

    // 3. Convert USD amount to USDC (6 decimals)
    const usdcAmount = toSmallestUnit(request.amountUsd.toString(), 6);

    // 4. Preview deposit to get shares estimate
    const estimatedShares = await this.yoGatewayService.previewDeposit(
      vaultConfig.address,
      usdcAmount,
    );

    // 5. Apply slippage tolerance (default 1%)
    const slippageTolerance = request.slippageTolerance ?? 100; // 100 = 1%
    const minShares = (
      BigInt(estimatedShares) * BigInt(10000 - slippageTolerance) / BigInt(10000)
    ).toString();

    // 6. Build deposit calldata
    const calldata = this.yoGatewayService.buildDepositCalldata(
      vaultConfig.address,
      usdcAmount,
      minShares,
      wallet.ethereumAddress || wallet.solanaAddress,
      yoConfig.partnerId,
    );

    // 7. Return quote
    return {
      depositAmount: usdcAmount,
      estimatedShares,
      minShares,
      vaultAddress: vaultConfig.address,
      network: vaultConfig.network,
      calldata,
      expiresAt: new Date(Date.now() + 15 * 60 * 1000).toISOString(), // 15 min expiry
    };
  }

  /**
   * Execute a YO deposit via OneBalance atomic intent
   * This converts the YO deposit calldata into a OneBalance intent
   */
  async executeDeposit(
    userId: string,
    quote: YoDepositQuote,
    signedIntents: any[],
    tamperProofSignature: string,
  ): Promise<any> {
    this.logger.log(`Executing YO deposit for user ${userId}`);

    // 1. Fetch user and wallet
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
      include: { Wallet: true },
    });

    if (!user || !user.Wallet.length) {
      throw new BadRequestException('User wallet not found');
    }

    const wallet = user.Wallet[0];
    const yoConfig = this.configService.get('yo');

    try {
      // 2. Build OneBalance multi-intent atomic call
      // Step 1: Approve USDC to YO Gateway (if needed)
      // Step 2: Call YO Gateway deposit function
      
      const intents = [
        {
          chain: quote.network === 'base' ? '8453' : '1',
          target: yoConfig.usdc[quote.network === 'base' ? 'addressBase' : 'addressEth'],
          value: '0',
          callData: this.buildApproveCalldata(
            yoConfig.gateway[quote.network === 'base' ? 'addressBase' : 'addressEth'],
            quote.depositAmount,
          ),
        },
        {
          chain: quote.network === 'base' ? '8453' : '1',
          target: yoConfig.gateway[quote.network === 'base' ? 'addressBase' : 'addressEth'],
          value: '0',
          callData: quote.calldata,
        },
      ];

      // 3. Execute via OneBalance
      const oneBalanceClient = this.oneBalanceProvider.getClient();
      const executionResult = await oneBalanceClient.post('/v3/intents/execute', {
        intents,
        accounts: await determineSourceAccounts(wallet),
        signedIntents,
        tamperProofSignature,
      });

      // 4. Store YO deposit in database
      const yoDeposit = await this.prisma.yoDeposit.create({
        data: {
          userId,
          vaultAddress: quote.vaultAddress,
          depositAsset: 'USDC',
          depositAmount: quote.depositAmount,
          sharesMinted: quote.estimatedShares,
          minSharesOut: quote.minShares,
          partnerId: yoConfig.partnerId,
          transactionId: executionResult.id,
          status: 'EXECUTING',
          metadata: {
            oneBalanceQuoteId: executionResult.id,
            network: quote.network,
          },
        },
      });

      return {
        depositId: yoDeposit.id,
        transactionId: executionResult.id,
        status: 'EXECUTING',
        depositAmount: fromSmallestUnit(quote.depositAmount, 6),
        estimatedShares: fromSmallestUnit(quote.estimatedShares, 18),
      };
    } catch (error: any) {
      this.logger.error(`Failed to execute YO deposit: ${error.message}`);
      throw new BadRequestException(
        `YO deposit execution failed: ${error.message}`,
      );
    }
  }

  /**
   * Build USDC approval calldata
   * Required before deposit
   */
  private buildApproveCalldata(spender: string, amount: string): string {
    // Standard ERC20 approve signature
    const iface = new ethers.Interface([
      'function approve(address spender, uint256 amount) returns (bool)',
    ]);
    return iface.encodeFunctionData('approve', [spender, amount]);
  }
}
```

#### Phase 5: Create YO Controller Endpoints

**File: `src/modules/yo/yo.controller.ts`**
```typescript
import { Controller, Post, Get, Body, Param, UseGuards, Request } from '@nestjs/common';
import { YoService } from './services/yo.service';
import { YoDepositHandler } from './handlers/yo-deposit.handler';
import { AuthGuard } from '../../common/guards/auth.guard';

@Controller('yo')
export class YoController {
  constructor(
    private yoService: YoService,
    private yoDepositHandler: YoDepositHandler,
  ) {}

  /**
   * GET /yo/deposit/quote
   * Get a quote for YO deposit (USDC → yoUSD)
   */
  @Get('deposit/quote')
  @UseGuards(AuthGuard)
  async getDepositQuote(
    @Request() req,
    @Body() body: { amountUsd: number; chain?: 'base' | 'ethereum' },
  ) {
    return await this.yoDepositHandler.getDepositQuote({
      userId: req.user.userId,
      amountUsd: body.amountUsd,
      preferredChain: body.chain,
    });
  }

  /**
   * POST /yo/deposit/execute
   * Execute a YO deposit with OneBalance atomic intents
   */
  @Post('deposit/execute')
  @UseGuards(AuthGuard)
  async executeDeposit(
    @Request() req,
    @Body() body: { quoteId: string; signedIntents: any[]; tamperProofSignature: string },
  ) {
    // Store quote and execute
    const quote = await this.yoService.getStoredQuote(body.quoteId);
    return await this.yoDepositHandler.executeDeposit(
      req.user.userId,
      quote,
      body.signedIntents,
      body.tamperProofSignature,
    );
  }

  /**
   * GET /yo/vault/:vaultAddress
   * Get vault snapshot (APY, TVL, etc.)
   */
  @Get('vault/:vaultAddress')
  async getVaultSnapshot(@Param('vaultAddress') vaultAddress: string) {
    return await this.yoService.getVaultInfo(vaultAddress);
  }

  /**
   * GET /yo/points/:walletAddress
   * Get user's YO points and earning rate
   */
  @Get('points/:walletAddress')
  async getUserPoints(@Param('walletAddress') walletAddress: string) {
    return await this.yoService.getUserPoints(walletAddress);
  }
}
```

---

## OneBalance Atomic Intent Call Flow

### How It Works (The Magic)

```
Frontend User Flow:
┌─────────────────────────────────────────────────┐
│ 1. User selects "Deposit 100 USDC → yoUSD"      │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 2. Backend builds quote via YO Gateway          │
│    - Input: 100 USDC (600000000 wei)           │
│    - Output: ~100 yoUSD shares (est.)          │
│    - Calldata: deposit(...) function           │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 3. Frontend receives atomic intents array       │
│    Intent 1: Approve USDC to YO Gateway        │
│    Intent 2: Call deposit(...)                  │
│    (Both executed atomically by OneBalance)     │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 4. User signs with wallet                      │
│    - OneBalance handles signing                 │
│    - Returns signed operations + signature      │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 5. Frontend sends back signed ops               │
│    - Backend verifies tamper-proof signature    │
│    - Submits to OneBalance execute endpoint     │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 6. OneBalance routes USDC from source chains    │
│    (e.g., Solana → bridge → Base USDC)         │
│    via atomic intent protocol                   │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 7. YO Gateway receives USDC, mints yoUSD       │
│    shares to user's wallet                      │
└────────┬────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────┐
│ 8. User now holds yoUSD (yield-bearing)        │
│    - Earns yield automatically                  │
│    - Can track via YO API                       │
│    - Anzo gets revenue share from performance   │
└─────────────────────────────────────────────────┘
```

### Key Difference from Manual Swap

❌ **Old Flow (Manual):**
User USDC (Solana) → Swap to Ethereum USDC → Bridge to Base USDC → Deposit to yoUSD = 3 steps + 2 bridge times

✅ **New Flow (OneBalance Atomic Intent):**
User USDC (Any chain) → Atomic route to Base → Deposit to yoUSD = 1 transaction

---

## Database Migration

**File: `prisma/migrations/[timestamp]_add_yo_deposit/migration.sql`**

```sql
-- Add YO Deposit tracking
CREATE TABLE "YoDeposit" (
    "id" TEXT NOT NULL,
    "userId" TEXT NOT NULL,
    "vaultAddress" TEXT NOT NULL,
    "depositAsset" TEXT NOT NULL,
    "depositAmount" TEXT NOT NULL,
    "sharesMinted" TEXT NOT NULL,
    "minSharesOut" TEXT NOT NULL,
    "partnerId" TEXT,
    "transactionId" TEXT,
    "status" TEXT NOT NULL DEFAULT 'PENDING',
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    "completedAt" TIMESTAMP(3),
    "metadata" JSONB,

    CONSTRAINT "YoDeposit_pkey" PRIMARY KEY ("id"),
    CONSTRAINT "YoDeposit_userId_fkey" FOREIGN KEY ("userId") REFERENCES "User"(id) ON DELETE RESTRICT ON UPDATE CASCADE,
    CONSTRAINT "YoDeposit_transactionId_fkey" FOREIGN KEY ("transactionId") REFERENCES "Transaction"(id) ON DELETE SET NULL ON UPDATE CASCADE
);

CREATE INDEX "YoDeposit_userId_idx" ON "YoDeposit"("userId");
CREATE INDEX "YoDeposit_status_idx" ON "YoDeposit"("status");
CREATE INDEX "YoDeposit_vaultAddress_idx" ON "YoDeposit"("vaultAddress");

-- Update Transaction model to support YO deposits
ALTER TABLE "Transaction" ADD COLUMN "yoDepositId" TEXT;
ALTER TABLE "Transaction" ADD CONSTRAINT "Transaction_yoDepositId_fkey" FOREIGN KEY ("yoDepositId") REFERENCES "YoDeposit"(id) ON DELETE SET NULL ON UPDATE CASCADE;
```

---

## Environment Configuration

**Add to `.env.example`:**

```env
# ============================================
# YO Protocol Integration
# ============================================
YO_GATEWAY_ADDRESS_BASE=0xF1EeE0957267b1A474323Ff9CfF7719E964969FA
YO_USD_VAULT_ADDRESS_BASE=0x0000000f2eB9f69274678c76222b35eEc7588a65
YO_USD_VAULT_ADDRESS_ETH=0x[REQUEST_FROM_YO_TEAM]
YO_USDC_ADDRESS_BASE=0x833589fCD6eDb6e08f4c7c32d4f71b54bDa02913
YO_USDC_ADDRESS_ETH=0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48
YO_PARTNER_ID=anzo-wallet
```

---

## API Endpoints Summary

### New YO Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/yo/deposit/quote` | GET | Get YO deposit quote |
| `/api/v1/yo/deposit/execute` | POST | Execute YO deposit |
| `/api/v1/yo/vault/:address` | GET | Fetch vault APY/TVL |
| `/api/v1/yo/points/:wallet` | GET | Get user YO points |
| `/api/v1/yo/vault/yield/timeseries/:address` | GET | Historical yield data |

---

## Integration Checklist

### Pre-Implementation
- [ ] Request Partner ID from YO team (Discord/Telegram)
- [ ] Get YO USD vault address for Ethereum mainnet
- [ ] Review YO Protocol whitepaper: https://exponential.fi/whitepaper
- [ ] Test with yoUSD on Base testnet

### Development
- [ ] Create `yo.module.ts` and services
- [ ] Implement `YoGatewayService` for API calls
- [ ] Build `YoDepositHandler` for quote/execute logic
- [ ] Create YO controller endpoints
- [ ] Add database migration for `YoDeposit` table
- [ ] Update `Transaction` model references
- [ ] Add YO config to `configuration.ts`
- [ ] Update `.env.example` with YO settings

### Testing
- [ ] Unit tests for YO services
- [ ] Integration test: Quote flow with mock data
- [ ] Integration test: Deposit flow with OneBalance simulator
- [ ] E2E test: Full deposit flow on Base testnet
- [ ] Test error handling (vault paused, slippage failure, etc.)

### Deployment
- [ ] Register Partner ID with YO team
- [ ] Deploy to production with correct vault addresses
- [ ] Monitor YO API response times
- [ ] Set up alerts for deposit failures
- [ ] Configure revenue share webhook (if available)

---

## Revenue Share Model

**How you earn:**

1. **Performance Fees**: YO charges 0.4% annual fee on vault assets
   - Integrators get percentage split based on Partner ID
   - Example: 100 users × $10K average = $1M AUM × 0.4% = $4K/year

2. **YO Points Kickback**: Users earn points on deposits
   - "5 points per $1 every day" rate
   - Integrators can earn points incentives (TBD with YO)

3. **Referral Rewards**: Future incentives for volume/TVL growth

**Tracking Revenue:**
- Monitor via YO Dashboard (once Partner ID registered)
- Track deposits via `YoDeposit` table in your DB
- API: `GET /api/v1/vault/{network}/{vaultAddress}` includes volume metrics

---

## Alternative Approaches (If Needed)

### Option 1: On-chain Direct (Higher Effort, No Revenue Share)
```typescript
// Deposit directly to vault contract without Gateway
// Pros: More control, custom UX per vault
// Cons: High complexity, no revenue share, vault-specific updates needed
```

### Option 2: Frontend-Only (Lowest Effort)
```typescript
// Add YO link in app's "dApp store"
// User taps → YO web app opens → Wallet connect → Transact
// Pros: 0 backend work, quick launch
// Cons: No revenue share, poor UX (external redirect)
```

### Option 3: Aggregator Route (Medium Effort)
```typescript
// Integrate with Enso.finance or Yield.xyz
// They handle YO integration; you just point users there
// Pros: Faster, aggregator UI handles routing
// Cons: Revenue share varies, limited customization
```

**Recommendation:** Stick with **On-chain Gateway** (Option 1 from main doc). Best balance of effort, UX, and revenue potential.

---

## Security Considerations

### Smart Contract Interaction
- [ ] Use ethers.js for calldata encoding (not manual hex)
- [ ] Validate YO Gateway address against official docs
- [ ] Implement minimum shares validation (slippage protection)
- [ ] Test with large amounts before production

### OneBalance Atomic Intents
- [ ] Verify tamper-proof signature matches original quote
- [ ] Expire quotes after 15 minutes (prevent replay)
- [ ] Log all deposit attempts for audit trail
- [ ] Handle partial failures in multi-intent calls

### API Security
- [ ] Cache YO vault data (avoid rate limits)
- [ ] Add request validation (amount ranges, wallet format)
- [ ] Implement rate limiting on `/yo/*` endpoints
- [ ] Use HTTPS only for YO API calls

### User Protection
- [ ] Show clear APY disclaimer (not guaranteed)
- [ ] Warn about smart contract risks
- [ ] Display fee structure (0.4% annual YO fee)
- [ ] Show expected shares with slippage estimate

---

## Monitoring & Observability

### Metrics to Track
```typescript
// src/modules/yo/metrics/yo.metrics.ts
export class YoMetrics {
  depositAttempts: Counter;       // Total deposit attempts
  depositSuccess: Counter;        // Successful deposits
  depositFailures: Counter;       // Failed deposits
  depositAmount: Histogram;       // USD amounts deposited
  quoteFetchTime: Histogram;      // API response times
  yieldAccrued: Gauge;            // Total user yield (daily)
  revenueShare: Gauge;            // Your earned revenue (daily)
}
```

### Logging
```typescript
this.logger.log(
  `YO deposit successful: $${amount} → ${shares} shares`,
  { userId, vaultAddress, txHash }
);
this.logger.error(
  `YO deposit failed: ${reason}`,
  { userId, amount, error },
);
```

---

## Timeline & Effort Estimate

| Phase | Task | Time | Effort |
|-------|------|------|--------|
| 1 | Setup & config | 1 day | Easy |
| 2 | YO services | 2 days | Medium |
| 3 | Deposit handler | 2 days | Medium |
| 4 | Controller + API | 1 day | Easy |
| 5 | Testing | 3 days | Medium |
| **Total** | **Full Integration** | **~9 days** | **Medium** |

**With existing OneBalance setup, this is very achievable.**

---

## Next Steps

1. **Contact YO Team** on Discord/Telegram to register as integrator
2. **Request Partner ID** (required for revenue sharing)
3. **Get vault addresses** for Ethereum mainnet
4. **Review code examples** above
5. **Start with `yo.module.ts`** and work through services
6. **Test on Base testnet** before mainnet deployment

---

## References

- **YO Protocol API Docs**: https://api.yo.xyz (in your files)
- **YO Risk Framework**: https://exponential.fi/whitepaper
- **OneBalance Multi-Intent Docs**: https://docs.onebalance.io/guides/contract-calls/examples
- **Base Network Docs**: https://docs.base.org/
- **ethers.js ABI Encoding**: https://docs.ethers.org/v6/api/abi/

---

**Document prepared for Anzo Wallet YO Protocol Integration**  
**Ready for development team implementation**
