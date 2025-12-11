# YIELD MODULE - QUICK CODE IMPLEMENTATION GUIDE

## 🎯 Get Started in 30 Minutes

This guide shows you the **exact code** you need to start integrating yield earning.

---

## Step 1: Install Dependencies

```bash
npm install \
  ethers \
  @typeorm/core typeorm \
  @nestjs/typeorm \
  @nestjs/axios \
  @nestjs/config \
  class-validator class-transformer \
  @types/node

# Dev dependencies
npm install --save-dev \
  @types/express \
  ts-node
```

---

## Step 2: Create DTOs (Data Validation)

**src/modules/yield/dto/deposit.dto.ts**
```typescript
import { IsString, IsNumber, IsEnum, Min, IsOptional } from 'class-validator';

export class DepositDto {
  @IsEnum(['aave', 'yo', 'yield-xyz'])
  protocol: 'aave' | 'yo' | 'yield-xyz';

  @IsNumber()
  @Min(0)
  amount: number;

  @IsString()
  token: string;

  @IsNumber()
  chainId: number;

  @IsString()
  @IsOptional()
  vaultAddress?: string; // For YO Protocol

  @IsString()
  @IsOptional()
  yieldId?: string; // For Yield.xyz
}

export class GetYieldsDto {
  @IsString()
  protocols: string; // Comma-separated: "aave,yo,yield-xyz"

  @IsString()
  @IsOptional()
  networks?: string; // "ethereum,base,polygon"

  @IsNumber()
  @IsOptional()
  chainId?: number;

  @IsNumber()
  @IsOptional()
  limit?: number;
}

export class WithdrawDto {
  @IsString()
  positionId: string;

  @IsNumber()
  @Min(0)
  amount: number;
}
```

---

## Step 3: Create Database Entities

**src/modules/yield/entities/yield-position.entity.ts**
```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('yield_positions')
export class YieldPosition {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userAddress: string;

  @Column()
  protocol: 'aave' | 'yo' | 'yield-xyz';

  @Column()
  chainId: number;

  @Column()
  token: string;

  @Column('decimal', { precision: 20, scale: 8 })
  depositAmount: number;

  @Column('decimal', { precision: 20, scale: 8, default: 0 })
  currentBalance: number;

  @Column('decimal', { precision: 10, scale: 4 })
  apy: number;

  @Column({
    type: 'enum',
    enum: ['pending', 'active', 'withdrawn', 'failed'],
    default: 'pending',
  })
  status: string;

  @Column({ nullable: true })
  vaultAddress?: string;

  @Column({ nullable: true })
  yieldId?: string;

  @Column({ nullable: true })
  lastTxHash?: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## Step 4: Create AAVE Adapter (Easiest to Start)

**src/modules/yield/adapters/aave.adapter.ts**
```typescript
import { Injectable } from '@nestjs/common';
import { ethers } from 'ethers';

// ABI snippets - get full ABI from https://docs.aave.com
const AAVE_POOL_ABI = [
  {
    name: 'supply',
    type: 'function',
    inputs: [
      { name: 'asset', type: 'address' },
      { name: 'amount', type: 'uint256' },
      { name: 'onBehalfOf', type: 'address' },
      { name: 'referralCode', type: 'uint16' },
    ],
    outputs: [],
    stateMutability: 'nonpayable',
  },
  {
    name: 'withdraw',
    type: 'function',
    inputs: [
      { name: 'asset', type: 'address' },
      { name: 'amount', type: 'uint256' },
      { name: 'to', type: 'address' },
    ],
    outputs: [{ type: 'uint256' }],
    stateMutability: 'nonpayable',
  },
  {
    name: 'getReserveData',
    type: 'function',
    inputs: [{ name: 'asset', type: 'address' }],
    outputs: [
      { name: 'configuration', type: 'tuple' },
      { name: 'liquidityIndex', type: 'uint128' },
      { name: 'currentLiquidityRate', type: 'uint128' },
      { name: 'variableBorrowIndex', type: 'uint128' },
      { name: 'currentVariableBorrowRate', type: 'uint128' },
      { name: 'currentStableBorrowRate', type: 'uint128' },
      { name: 'lastUpdateTimestamp', type: 'uint40' },
      { name: 'aTokenAddress', type: 'address' },
      { name: 'stableDebtTokenAddress', type: 'address' },
      { name: 'variableDebtTokenAddress', type: 'address' },
      { name: 'interestRateStrategyAddress', type: 'address' },
    ],
    stateMutability: 'view',
  },
];

@Injectable()
export class AaveAdapter {
  // AAVE Pool addresses by chain
  private poolAddresses = {
    1: '0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9', // Ethereum
    137: '0x794a61358D6845594F94dc1DB02A252b5b4814aD', // Polygon
    8453: '0xA238Dd5C597D265DAC63E86649Ffdf0EF07132dD', // Base
  };

  private rpcUrls = {
    1: process.env.ETHEREUM_RPC_URL,
    137: process.env.POLYGON_RPC_URL,
    8453: process.env.BASE_RPC_URL,
  };

  /**
   * Build deposit transaction (unsigned)
   */
  buildDepositTx(params: {
    userAddress: string;
    token: string;
    amount: string;
    chainId: number;
  }): any {
    const poolAddress = this.poolAddresses[params.chainId];
    const iface = new ethers.Interface(AAVE_POOL_ABI);

    // Encode the supply function call
    const txData = iface.encodeFunctionData('supply', [
      params.token,
      params.amount,
      params.userAddress,
      0, // referral code
    ]);

    return {
      to: poolAddress,
      data: txData,
      value: '0',
      chainId: params.chainId,
    };
  }

  /**
   * Build withdrawal transaction (unsigned)
   */
  buildWithdrawTx(params: {
    userAddress: string;
    token: string;
    amount: string;
    chainId: number;
  }): any {
    const poolAddress = this.poolAddresses[params.chainId];
    const iface = new ethers.Interface(AAVE_POOL_ABI);

    const txData = iface.encodeFunctionData('withdraw', [
      params.token,
      params.amount,
      params.userAddress,
    ]);

    return {
      to: poolAddress,
      data: txData,
      value: '0',
      chainId: params.chainId,
    };
  }

  /**
   * Get current APY for a token
   */
  async getApy(token: string, chainId: number): Promise<number> {
    const rpcUrl = this.rpcUrls[chainId];
    const poolAddress = this.poolAddresses[chainId];

    const provider = new ethers.JsonRpcProvider(rpcUrl);
    const pool = new ethers.Contract(poolAddress, AAVE_POOL_ABI, provider);

    try {
      const reserveData = await pool.getReserveData(token);
      // currentLiquidityRate is in RAY (27 decimals)
      const rayValue = reserveData.currentLiquidityRate;
      const apy = Number(rayValue) / 1e25; // Convert RAY to percentage
      return apy;
    } catch (error) {
      console.error('Error fetching APY:', error);
      return 0;
    }
  }

  getSpenderAddress(chainId: number): string {
    return this.poolAddresses[chainId];
  }
}
```

---

## Step 5: Create Yield Service (Business Logic)

**src/modules/yield/yield.service.ts**
```typescript
import { Injectable, BadRequestException, InternalServerErrorException, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ethers } from 'ethers';

import { YieldPosition } from './entities/yield-position.entity';
import { AaveAdapter } from './adapters/aave.adapter';
import { DepositDto, WithdrawDto, GetYieldsDto } from './dto';

@Injectable()
export class YieldService {
  private logger = new Logger('YieldService');

  constructor(
    @InjectRepository(YieldPosition)
    private positionRepo: Repository<YieldPosition>,
    private aaveAdapter: AaveAdapter,
    // Add other adapters here
  ) {}

  /**
   * Get available yield opportunities
   */
  async getOpportunities(dto: GetYieldsDto) {
    const opportunities = [];

    // Parse protocols
    const protocols = dto.protocols.split(',').map(p => p.trim());
    const networks = dto.networks?.split(',').map(n => n.trim()) || ['ethereum'];

    // Get AAVE yields if requested
    if (protocols.includes('aave')) {
      for (const network of networks) {
        const chainId = this.networkToChainId(network);
        if (!chainId) continue;

        try {
          // Example USDC address on Ethereum
          const usdc = '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48';
          const apy = await this.aaveAdapter.getApy(usdc, chainId);

          opportunities.push({
            id: `aave-${chainId}-usdc`,
            protocol: 'aave',
            token: 'USDC',
            apy,
            tvl: '2.5 billion', // Would fetch from contract
            chainId,
          });
        } catch (error) {
          this.logger.error(`Error fetching AAVE yields for ${network}:`, error);
        }
      }
    }

    return {
      count: opportunities.length,
      opportunities: opportunities.sort((a, b) => b.apy - a.apy),
    };
  }

  /**
   * Prepare deposit transaction
   */
  async prepareDeposit(dto: DepositDto, userAddress: string) {
    // Validate
    if (dto.amount <= 0) {
      throw new BadRequestException('Amount must be > 0');
    }

    if (!ethers.isAddress(userAddress)) {
      throw new BadRequestException('Invalid user address');
    }

    if (!ethers.isAddress(dto.token)) {
      throw new BadRequestException('Invalid token address');
    }

    try {
      let txData;

      // Route to appropriate adapter
      if (dto.protocol === 'aave') {
        txData = this.aaveAdapter.buildDepositTx({
          userAddress,
          token: dto.token,
          amount: ethers.parseUnits(dto.amount.toString(), 6).toString(), // Assume 6 decimals
          chainId: dto.chainId,
        });
      } else {
        throw new BadRequestException(`Unsupported protocol: ${dto.protocol}`);
      }

      // Create position record
      const position = this.positionRepo.create({
        userAddress,
        protocol: dto.protocol,
        chainId: dto.chainId,
        token: dto.token,
        depositAmount: dto.amount,
        apy: await this.aaveAdapter.getApy(dto.token, dto.chainId),
        status: 'pending',
      });

      await this.positionRepo.save(position);

      return {
        positionId: position.id,
        transaction: {
          type: 'deposit',
          data: txData,
        },
        message: `Deposit ${dto.amount} to ${dto.protocol} ready for signing`,
      };
    } catch (error) {
      this.logger.error('Error preparing deposit:', error);
      throw new InternalServerErrorException(`Failed to prepare deposit: ${error.message}`);
    }
  }

  /**
   * Get user's positions
   */
  async getUserPositions(userAddress: string) {
    return await this.positionRepo.find({
      where: { userAddress, status: 'active' },
      order: { createdAt: 'DESC' },
    });
  }

  /**
   * Update position after on-chain confirmation
   */
  async confirmPosition(positionId: string, txHash: string) {
    const position = await this.positionRepo.findOne({ where: { id: positionId } });
    if (!position) {
      throw new BadRequestException('Position not found');
    }

    position.status = 'active';
    position.lastTxHash = txHash;
    await this.positionRepo.save(position);

    return position;
  }

  // Helper
  private networkToChainId(network: string): number | null {
    const mapping: Record<string, number> = {
      ethereum: 1,
      polygon: 137,
      base: 8453,
      arbitrum: 42161,
      optimism: 10,
    };
    return mapping[network.toLowerCase()];
  }
}
```

---

## Step 6: Create REST Controller

**src/modules/yield/yield.controller.ts**
```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Query,
  Param,
  UseGuards,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport'; // Your existing guard
import { CurrentUser } from '../auth/decorators/current-user.decorator'; // Your decorator

import { YieldService } from './yield.service';
import { DepositDto, WithdrawDto, GetYieldsDto } from './dto';

@Controller('api/yield')
export class YieldController {
  constructor(private yieldService: YieldService) {}

  @Get('opportunities')
  async getOpportunities(@Query() dto: GetYieldsDto) {
    return await this.yieldService.getOpportunities(dto);
  }

  @Get('positions/:userAddress')
  async getPositions(@Param('userAddress') userAddress: string) {
    return await this.yieldService.getUserPositions(userAddress);
  }

  @Post('deposit')
  @UseGuards(AuthGuard('jwt'))
  async deposit(@Body() dto: DepositDto, @CurrentUser() user: any) {
    return await this.yieldService.prepareDeposit(dto, user.address);
  }

  @Post('confirm/:positionId')
  @UseGuards(AuthGuard('jwt'))
  async confirmPosition(
    @Param('positionId') positionId: string,
    @Query('txHash') txHash: string,
  ) {
    return await this.yieldService.confirmPosition(positionId, txHash);
  }
}
```

---

## Step 7: Register Module

**src/modules/yield/yield.module.ts**
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { HttpModule } from '@nestjs/axios';

import { YieldService } from './yield.service';
import { YieldController } from './yield.controller';
import { AaveAdapter } from './adapters/aave.adapter';
import { YieldPosition } from './entities/yield-position.entity';

@Module({
  imports: [
    TypeOrmModule.forFeature([YieldPosition]),
    HttpModule,
  ],
  controllers: [YieldController],
  providers: [YieldService, AaveAdapter],
  exports: [YieldService],
})
export class YieldModule {}
```

**src/app.module.ts** (Add to imports)
```typescript
import { YieldModule } from './modules/yield/yield.module';

@Module({
  imports: [
    // ... other modules
    YieldModule,
  ],
})
export class AppModule {}
```

---

## Step 8: Frontend Code Example

**frontend/hooks/useYield.ts** (React/Next.js)
```typescript
import { useState } from 'react';

export const useYield = () => {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const getOpportunities = async (protocols: string[]) => {
    try {
      setLoading(true);
      const response = await fetch(
        `/api/yield/opportunities?protocols=${protocols.join(',')}&networks=ethereum,base`,
      );
      const data = await response.json();
      return data;
    } catch (err) {
      setError(err.message);
      return null;
    } finally {
      setLoading(false);
    }
  };

  const deposit = async (protocol: string, amount: number, token: string) => {
    try {
      setLoading(true);
      const response = await fetch('/api/yield/deposit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          protocol,
          amount,
          token,
          chainId: 1, // Ethereum
        }),
      });
      return await response.json();
    } catch (err) {
      setError(err.message);
      return null;
    } finally {
      setLoading(false);
    }
  };

  return { getOpportunities, deposit, loading, error };
};
```

**frontend/components/YieldDeposit.tsx**
```typescript
import React, { useState } from 'react';
import { useYield } from '../hooks/useYield';
import { useAccount, useSendTransaction } from 'wagmi'; // Or your wallet hook
import { useTurnkey } from '@turnkey/sdk-react';

export const YieldDeposit = () => {
  const { address } = useAccount();
  const { walletClient } = useTurnkey();
  const { deposit } = useYield();
  const { sendTransaction } = useSendTransaction();

  const [amount, setAmount] = useState('100');
  const [protocol, setProtocol] = useState('aave');

  const handleDeposit = async () => {
    // 1. Get unsigned transaction
    const result = await deposit(protocol, parseFloat(amount), 'USDC');
    const { transaction, positionId } = result;

    // 2. Sign with Turnkey
    const signedTx = await walletClient?.signTransaction({
      ...transaction.data,
    });

    // 3. Broadcast
    const tx = await sendTransaction({
      to: transaction.data.to,
      data: transaction.data.data,
    });

    // 4. Confirm on backend
    if (tx) {
      await fetch(`/api/yield/confirm/${positionId}?txHash=${tx.hash}`, {
        method: 'POST',
      });
    }
  };

  return (
    <div>
      <input
        type="number"
        value={amount}
        onChange={(e) => setAmount(e.target.value)}
        placeholder="Amount"
      />
      <select value={protocol} onChange={(e) => setProtocol(e.target.value)}>
        <option value="aave">AAVE</option>
        <option value="yo">YO Protocol</option>
      </select>
      <button onClick={handleDeposit}>Deposit to {protocol}</button>
    </div>
  );
};
```

---

## Step 9: Testing

**test/yield.service.spec.ts**
```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { YieldService } from '../src/modules/yield/yield.service';
import { YieldPosition } from '../src/modules/yield/entities/yield-position.entity';
import { AaveAdapter } from '../src/modules/yield/adapters/aave.adapter';

describe('YieldService', () => {
  let service: YieldService;
  let mockPositionRepo: any;
  let mockAaveAdapter: any;

  beforeEach(async () => {
    mockPositionRepo = {
      find: jest.fn().mockResolvedValue([]),
      create: jest.fn().mockImplementation(dto => dto),
      save: jest.fn().mockResolvedValue({ id: 'test-id' }),
    };

    mockAaveAdapter = {
      getApy: jest.fn().mockResolvedValue(5.23),
      buildDepositTx: jest.fn().mockReturnValue({
        to: '0x...',
        data: '0x...',
      }),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        YieldService,
        {
          provide: getRepositoryToken(YieldPosition),
          useValue: mockPositionRepo,
        },
        {
          provide: AaveAdapter,
          useValue: mockAaveAdapter,
        },
      ],
    }).compile();

    service = module.get<YieldService>(YieldService);
  });

  it('should deposit successfully', async () => {
    const result = await service.prepareDeposit(
      {
        protocol: 'aave',
        amount: 100,
        token: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
        chainId: 1,
      },
      '0x742d35Cc6634C0532925a3b844Bc92d426B9D24e',
    );

    expect(result).toHaveProperty('positionId');
    expect(result).toHaveProperty('transaction');
  });
});
```

---

## Step 10: Environment Variables

**.env** (Add these)
```env
# Yield Module
ETHEREUM_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/YOUR_KEY
BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/YOUR_KEY

# Yield.xyz (if using)
YIELD_XYZ_API_KEY=your-api-key-here

# Turnkey (existing)
TURNKEY_API_URL=https://api.turnkey.com
TURNKEY_ORGANIZATION_ID=your-org-id
```

---

## ✅ Checklist to Run

```
□ npm install dependencies
□ Create DTOs
□ Create entity
□ Create AAVE adapter
□ Create yield service
□ Create controller
□ Add to app.module imports
□ Setup database migrations
□ Test deposit endpoint
□ Test Turnkey signing
□ Deploy to staging
```

---

## 🚀 Next Steps

1. **Test on Sepolia testnet first**
   ```bash
   ETHEREUM_RPC_URL=https://sepolia.infura.io/v3/YOUR_KEY npm run test
   ```

2. **Add YO Protocol adapter** (copy AAVE adapter pattern, change ABI)

3. **Add Yield.xyz adapter** (HTTP API based, not contract-based)

4. **Deploy to staging**
   ```bash
   npm run build
   docker build -t yield-service:latest .
   docker push your-registry/yield-service:latest
   ```

5. **Monitor** (add Prometheus metrics)

6. **Go live** (with gradual rollout)

---

## 💬 Questions?

- AAVE docs: https://docs.aave.com
- YO docs: https://docs.yo.xyz  
- Turnkey docs: https://docs.turnkey.com
- NestJS: https://docs.nestjs.com

You now have the foundation to build a production-grade yield aggregator! 🎉
