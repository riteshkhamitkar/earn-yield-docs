# YO Protocol Integration - Code Examples & Quick Reference

## Quick Implementation Path

### 1. Add YO Config (5 minutes)

```typescript
// src/config/configuration.ts - Add this section

export default () => ({
  // ... existing config ...
  yo: {
    // YO Gateway smart contract (routes to all vaults)
    gateway: {
      addressBase: process.env.YO_GATEWAY_ADDRESS_BASE || '0xF1EeE0957267b1A474323Ff9CfF7719E964969FA',
      addressEth: process.env.YO_GATEWAY_ADDRESS_ETH,
    },
    // Specific vault addresses
    vaults: {
      usdBase: {
        address: process.env.YO_USD_VAULT_ADDRESS_BASE || '0x0000000f2eB9f69274678c76222b35eEc7588a65',
        network: 'base',
        chainId: 8453,
        decimals: 18,
      },
      usdEth: {
        address: process.env.YO_USD_VAULT_ADDRESS_ETH,
        network: 'ethereum',
        chainId: 1,
        decimals: 18,
      },
    },
    // USDC token contract addresses
    usdc: {
      addressBase: process.env.YO_USDC_ADDRESS_BASE || '0x833589fCD6eDb6e08f4c7c32d4f71b54bDa02913',
      addressEth: process.env.YO_USDC_ADDRESS_ETH || '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
      decimals: 6,
    },
    // Your integrator ID for revenue sharing
    partnerId: process.env.YO_PARTNER_ID || '0',
    // YO API base
    apiUrl: 'https://api.yo.xyz',
  },
});
```

### 2. Create YO Gateway Service (30 minutes)

```typescript
// src/modules/yo/services/yo-gateway.service.ts

import { Injectable, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import axios from 'axios';
import { ethers } from 'ethers';

@Injectable()
export class YoGatewayService {
  private readonly logger = new Logger(YoGatewayService.name);
  private readonly httpClient = axios.create({
    baseURL: 'https://api.yo.xyz',
    timeout: 10000,
  });

  // YO Gateway ABI (minimal - only what we need)
  private readonly GATEWAY_ABI = [
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
    {
      inputs: [
        { name: 'spender', type: 'address' },
        { name: 'amount', type: 'uint256' },
      ],
      name: 'approve',
      outputs: [{ name: '', type: 'bool' }],
      stateMutability: 'nonpayable',
      type: 'function',
    },
  ];

  constructor(private configService: ConfigService) {}

  /**
   * Build encoded calldata for deposit transaction
   * This is what OneBalance will execute
   */
  buildDepositCalldata(
    vaultAddress: string,
    usdcAmountWei: string,
    minSharesOut: string,
    receiverAddress: string,
  ): string {
    const iface = new ethers.Interface(this.GATEWAY_ABI);
    
    const partnerId = this.configService.get('yo.partnerId') || '0';
    
    return iface.encodeFunctionData('deposit', [
      vaultAddress,
      usdcAmountWei,
      minSharesOut,
      receiverAddress,
      partnerId,
    ]);
  }

  /**
   * Build approval calldata for USDC token
   */
  buildApproveCalldata(spenderAddress: string, amountWei: string): string {
    const iface = new ethers.Interface(this.GATEWAY_ABI);
    return iface.encodeFunctionData('approve', [spenderAddress, amountWei]);
  }

  /**
   * Fetch vault snapshot from YO API
   */
  async getVaultSnapshot(vaultAddress: string, network: string = 'base') {
    try {
      const response = await this.httpClient.get(
        `/api/v1/vault/${network}/${vaultAddress}`
      );
      return response.data;
    } catch (error: any) {
      this.logger.error(`Failed to fetch vault snapshot: ${error.message}`);
      throw error;
    }
  }

  /**
   * Fetch user's YO points
   */
  async getUserPoints(walletAddress: string) {
    try {
      const response = await this.httpClient.get(
        `/api/v1/users/wallet/${walletAddress}/points`
      );
      return response.data;
    } catch (error: any) {
      if (error.response?.status === 404) {
        // User not yet in YO system
        return null;
      }
      this.logger.error(`Failed to fetch points: ${error.message}`);
      throw error;
    }
  }

  /**
   * Fetch vault yield history
   */
  async getVaultYield(vaultAddress: string, network: string = 'base') {
    try {
      const response = await this.httpClient.get(
        `/api/v1/vault/yield/timeseries/${network}/${vaultAddress}`
      );
      return response.data;
    } catch (error: any) {
      this.logger.error(`Failed to fetch vault yield: ${error.message}`);
      throw error;
    }
  }
}
```

### 3. Create Deposit Handler (45 minutes)

```typescript
// src/modules/yo/handlers/yo-deposit.handler.ts

import { Injectable, Logger, BadRequestException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { PrismaService } from '../../database/prisma.service';
import { YoGatewayService } from '../services/yo-gateway.service';
import { OneBalanceProvider } from '../../swaps/providers/onebalance.provider';
import { toSmallestUnit, fromSmallestUnit } from '../../../common/utils/amount.util';

export interface YoDepositQuoteRequest {
  userId: string;
  amountUsd: number; // e.g., 100
  chain?: 'base' | 'ethereum';
  slippage?: number; // 100 = 1% (default)
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
   * Generate a quote for YO deposit
   * Returns atomic intent calldata ready for OneBalance execution
   */
  async generateQuote(request: YoDepositQuoteRequest) {
    this.logger.log(`Generating YO deposit quote for ${request.amountUsd} USD`);

    // Fetch user wallet
    const user = await this.prisma.user.findUnique({
      where: { id: request.userId },
      include: { Wallet: true },
    });

    if (!user?.Wallet.length) {
      throw new BadRequestException('User wallet not found');
    }

    const wallet = user.Wallet[0];
    const yoConfig = this.configService.get('yo');
    const chain = request.chain || 'base';

    // Select vault
    const vaultConfig = chain === 'ethereum' 
      ? yoConfig.vaults.usdEth 
      : yoConfig.vaults.usdBase;

    if (!vaultConfig.address) {
      throw new BadRequestException(`YO vault not configured for ${chain}`);
    }

    // Convert USD amount to USDC (6 decimals)
    const usdcWei = toSmallestUnit(request.amountUsd.toString(), 6);

    // Build calldata for approval + deposit
    const usdcConfig = yoConfig.usdc;
    const gatewayAddress = yoConfig.gateway[chain === 'ethereum' ? 'addressEth' : 'addressBase'];

    // Approval calldata (always needed before deposit)
    const approveCalldata = this.yoGatewayService.buildApproveCalldata(
      gatewayAddress,
      usdcWei
    );

    // Get estimate of shares (for UI display)
    // Note: In production, you'd call previewDeposit on actual vault
    const estimatedShares = usdcWei; // Placeholder: 1:1 assumption
    
    // Apply slippage (default 1%)
    const slippage = request.slippage || 100; // 100 = 1%
    const minSharesOut = (
      BigInt(estimatedShares) * BigInt(10000 - slippage) / BigInt(10000)
    ).toString();

    // Deposit calldata
    const depositCalldata = this.yoGatewayService.buildDepositCalldata(
      vaultConfig.address,
      usdcWei,
      minSharesOut,
      wallet.ethereumAddress || wallet.solanaAddress,
    );

    // Return quote object
    const quoteId = `yo-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;

    const quote = {
      quoteId,
      vaultAddress: vaultConfig.address,
      network: vaultConfig.network,
      chainId: vaultConfig.chainId,
      depositAmount: usdcWei,
      estimatedShares,
      minSharesOut,
      slippage,
      intents: [
        {
          // Intent 1: Approve USDC to gateway
          target: usdcConfig[chain === 'ethereum' ? 'addressEth' : 'addressBase'],
          calldata: approveCalldata,
          value: '0',
        },
        {
          // Intent 2: Deposit via gateway
          target: gatewayAddress,
          calldata: depositCalldata,
          value: '0',
        },
      ],
      expiresAt: new Date(Date.now() + 15 * 60 * 1000).toISOString(),
    };

    // Store quote in memory/cache (short-lived)
    // In production, use Redis or memcached
    await this.storeQuote(quoteId, quote);

    return quote;
  }

  /**
   * Execute YO deposit via OneBalance atomic intents
   */
  async executeDeposit(
    userId: string,
    quoteId: string,
    signedIntents: any[],
    tamperProofSig: string,
  ) {
    this.logger.log(`Executing YO deposit for user ${userId}, quote ${quoteId}`);

    // Retrieve quote
    const quote = await this.retrieveQuote(quoteId);
    if (!quote) {
      throw new BadRequestException('Quote expired or not found');
    }

    // Check expiry
    if (new Date() > new Date(quote.expiresAt)) {
      throw new BadRequestException('Quote expired');
    }

    // Fetch user
    const user = await this.prisma.user.findUnique({
      where: { id: userId },
      include: { Wallet: true },
    });

    if (!user?.Wallet.length) {
      throw new BadRequestException('User wallet not found');
    }

    try {
      // Execute via OneBalance
      const oneBalanceClient = this.oneBalanceProvider.getClient();
      const executionResult = await oneBalanceClient.post('/v3/intents/execute', {
        intents: quote.intents,
        signedIntents,
        tamperProofSignature: tamperProofSig,
      });

      // Store YO deposit record
      const yoDeposit = await this.prisma.yoDeposit.create({
        data: {
          userId,
          vaultAddress: quote.vaultAddress,
          depositAsset: 'USDC',
          depositAmount: quote.depositAmount,
          sharesMinted: quote.estimatedShares,
          minSharesOut: quote.minSharesOut,
          partnerId: this.configService.get('yo.partnerId'),
          transactionId: executionResult.transactionId,
          status: 'EXECUTING',
          metadata: {
            quoteId,
            network: quote.network,
            oneBalanceQuoteId: executionResult.id,
          },
        },
      });

      // Clean up stored quote
      await this.deleteQuote(quoteId);

      return {
        depositId: yoDeposit.id,
        transactionId: executionResult.transactionId,
        status: 'EXECUTING',
        shares: fromSmallestUnit(quote.estimatedShares, 18),
      };
    } catch (error: any) {
      this.logger.error(`YO deposit failed: ${error.message}`);
      throw new BadRequestException(`Deposit execution failed: ${error.message}`);
    }
  }

  // Quote storage (use Redis in production)
  private quotes = new Map<string, any>();

  private async storeQuote(quoteId: string, quote: any) {
    this.quotes.set(quoteId, quote);
    // Auto-delete after 15 minutes
    setTimeout(() => this.quotes.delete(quoteId), 15 * 60 * 1000);
  }

  private async retrieveQuote(quoteId: string) {
    return this.quotes.get(quoteId);
  }

  private async deleteQuote(quoteId: string) {
    this.quotes.delete(quoteId);
  }
}
```

### 4. Create Controller (20 minutes)

```typescript
// src/modules/yo/yo.controller.ts

import { Controller, Post, Get, Body, Param, UseGuards, Request, BadRequestException } from '@nestjs/common';
import { YoDepositHandler } from './handlers/yo-deposit.handler';
import { YoGatewayService } from './services/yo-gateway.service';
import { AuthGuard } from '../../common/guards/auth.guard';

@Controller('yo')
export class YoController {
  constructor(
    private depositHandler: YoDepositHandler,
    private gatewayService: YoGatewayService,
  ) {}

  /**
   * POST /api/v1/yo/deposit/quote
   * Get a quote for YO deposit
   * 
   * Body:
   * {
   *   "amountUsd": 100,
   *   "chain": "base" // optional, defaults to base
   * }
   */
  @Post('deposit/quote')
  @UseGuards(AuthGuard)
  async getDepositQuote(
    @Request() req,
    @Body() body: { amountUsd: number; chain?: 'base' | 'ethereum' },
  ) {
    if (!body.amountUsd || body.amountUsd <= 0) {
      throw new BadRequestException('Invalid amount');
    }

    return await this.depositHandler.generateQuote({
      userId: req.user.userId,
      amountUsd: body.amountUsd,
      chain: body.chain || 'base',
    });
  }

  /**
   * POST /api/v1/yo/deposit/execute
   * Execute a YO deposit with signed intents
   * 
   * Body:
   * {
   *   "quoteId": "yo-123456-abc",
   *   "signedIntents": [...],
   *   "tamperProofSignature": "0x..."
   * }
   */
  @Post('deposit/execute')
  @UseGuards(AuthGuard)
  async executeDeposit(
    @Request() req,
    @Body() body: {
      quoteId: string;
      signedIntents: any[];
      tamperProofSignature: string;
    },
  ) {
    return await this.depositHandler.executeDeposit(
      req.user.userId,
      body.quoteId,
      body.signedIntents,
      body.tamperProofSignature,
    );
  }

  /**
   * GET /api/v1/yo/vault/:vaultAddress
   * Get vault info (APY, TVL, etc.)
   */
  @Get('vault/:vaultAddress')
  async getVaultInfo(@Param('vaultAddress') vaultAddress: string) {
    // Validate address format
    if (!vaultAddress.match(/^0x[a-fA-F0-9]{40}$/)) {
      throw new BadRequestException('Invalid vault address');
    }

    const snapshot = await this.gatewayService.getVaultSnapshot(vaultAddress, 'base');
    const yield_ = await this.gatewayService.getVaultYield(vaultAddress, 'base');

    return {
      address: vaultAddress,
      snapshot,
      yield: yield_,
    };
  }

  /**
   * GET /api/v1/yo/points/:walletAddress
   * Get user's YO points info
   */
  @Get('points/:walletAddress')
  async getUserPoints(@Param('walletAddress') walletAddress: string) {
    if (!walletAddress.match(/^0x[a-fA-F0-9]{40}$/)) {
      throw new BadRequestException('Invalid wallet address');
    }

    const points = await this.gatewayService.getUserPoints(walletAddress);
    
    if (!points) {
      return { message: 'User not found in YO system' };
    }

    return points;
  }
}
```

### 5. Update Prisma Schema (10 minutes)

```prisma
// prisma/schema.prisma - Add this model

model YoDeposit {
  id            String   @id @default(cuid())
  userId        String
  vaultAddress  String
  depositAsset  String   // "USDC"
  depositAmount String   // Wei amount
  sharesMinted  String   // Estimated shares
  minSharesOut  String   // With slippage
  partnerId     String?
  transactionId String?
  status        String   @default("PENDING") // PENDING, EXECUTING, COMPLETED, FAILED
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  completedAt   DateTime?
  metadata      Json?

  User          User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  Transaction   Transaction? @relation(fields: [transactionId], references: [id], onDelete: SetNull)

  @@index([userId])
  @@index([status])
  @@index([vaultAddress])
  @@map("YoDeposit")
}
```

### 6. Register in Module (5 minutes)

```typescript
// src/modules/yo/yo.module.ts

import { Module } from '@nestjs/common';
import { YoController } from './yo.controller';
import { YoDepositHandler } from './handlers/yo-deposit.handler';
import { YoGatewayService } from './services/yo-gateway.service';
import { DatabaseModule } from '../database/database.module';

@Module({
  imports: [DatabaseModule],
  controllers: [YoController],
  providers: [YoGatewayService, YoDepositHandler],
  exports: [YoGatewayService],
})
export class YoModule {}
```

Then add to `app.module.ts`:

```typescript
import { YoModule } from './modules/yo/yo.module';

@Module({
  imports: [
    // ... existing modules ...
    YoModule,
  ],
})
export class AppModule {}
```

---

## Frontend Integration Example

```typescript
// Frontend would call these endpoints

// 1. Get quote
const quoteResponse = await fetch('/api/v1/yo/deposit/quote', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({
    amountUsd: 100,
    chain: 'base'
  })
});

const quote = await quoteResponse.json();
// Returns: { quoteId, vaultAddress, intents, expiresAt, ... }

// 2. Sign intents with wallet
const signedIntents = await wallet.signIntents(quote.intents);
const tamperProofSignature = quote.tamperProofSignature; // From backend

// 3. Execute deposit
const executeResponse = await fetch('/api/v1/yo/deposit/execute', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${token}` },
  body: JSON.stringify({
    quoteId: quote.quoteId,
    signedIntents,
    tamperProofSignature
  })
});

const result = await executeResponse.json();
// Returns: { depositId, transactionId, status, shares }
```

---

## Testing Checklist

```bash
# 1. Unit tests
npm test -- yo.gateway.service.ts
npm test -- yo-deposit.handler.ts

# 2. Integration test - Quote generation
curl -X POST http://localhost:3000/api/v1/yo/deposit/quote \
  -H "Authorization: Bearer ${TOKEN}" \
  -d '{"amountUsd": 100, "chain": "base"}'

# Expected: 200 with quoteId, intents, etc.

# 3. Integration test - Vault info
curl http://localhost:3000/api/v1/yo/vault/0x0000000f2eB9f69274678c76222b35eEc7588a65

# Expected: 200 with APY, TVL data

# 4. Integration test - User points
curl http://localhost:3000/api/v1/yo/points/0x[USER_ADDRESS]

# Expected: 200 with points data or 404 if new user
```

---

## Environment Variables

```env
# Required
YO_GATEWAY_ADDRESS_BASE=0xF1EeE0957267b1A474323Ff9CfF7719E964969FA
YO_USD_VAULT_ADDRESS_BASE=0x0000000f2eB9f69274678c76222b35eEc7588a65
YO_USDC_ADDRESS_BASE=0x833589fCD6eDb6e08f4c7c32d4f71b54bDa02913

# Optional (for Ethereum support)
YO_GATEWAY_ADDRESS_ETH=0x...
YO_USD_VAULT_ADDRESS_ETH=0x...
YO_USDC_ADDRESS_ETH=0x...

# Your Partner ID (request from YO team)
YO_PARTNER_ID=anzo-wallet
```

---

## Summary

This implementation provides:
- ✅ Quote generation for YO deposits
- ✅ Atomic intent calldata building
- ✅ OneBalance integration ready
- ✅ Vault info & points fetching
- ✅ Full database tracking
- ✅ Error handling & logging

Total implementation: ~2-3 days for experienced team
