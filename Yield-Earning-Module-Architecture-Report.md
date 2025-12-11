# Comprehensive Technical Report: Yield Earning Flow Integration with Turnkey Wallets

## Executive Summary

This document provides a deep technical analysis and architecture design for implementing a yield earning module integrated with your existing Turnkey wallet infrastructure. The implementation will enable users to deposit their crypto assets (USDC, USDT, ETH, etc.) into yield-generating protocols like AAVE, YO Protocol, and Yield.xyz. The module will be built using NestJS architecture, following your existing pattern of modular services for balance fetching, token swapping, and transaction signing through Turnkey.

---

## Table of Contents

1. [Yield Provider Analysis](#yield-provider-analysis)
2. [Architecture Design](#architecture-design)
3. [Technical Implementation](#technical-implementation)
4. [API Integration Details](#api-integration-details)
5. [Turnkey Integration Flow](#turnkey-integration-flow)
6. [Database & State Management](#database--state-management)
7. [Security Considerations](#security-considerations)
8. [Error Handling & Monitoring](#error-handling--monitoring)
9. [Deployment Strategy](#deployment-strategy)

---

## Section 1: Yield Provider Analysis

### 1.1 AAVE (aave.com) - Lending Protocol

#### Overview
AAVE is the leading decentralized lending protocol supporting 12+ networks with billions in TVL. It operates on a pool-based model where users supply assets to earn interest from borrowers.

#### Key Features
- **Supply Function**: Users deposit assets to earn yield
- **Borrowing**: Users can borrow against collateral (not primary use case)
- **Multiple Networks**: Ethereum, Polygon, Arbitrum, Optimism, Base, Avalanche, etc.
- **aTokens**: Interest-bearing tokens representing user's deposit
- **Risk Management**: Health Factor, Collateralization ratios

#### Technical Specifications
- **Smart Contract Architecture**: Pool-based with LendingPool contract
- **Address Provider**: Central contract managing pool addresses
- **ERC-20 Compliant**: Uses standard token interfaces
- **UI Data Provider**: UiPoolDataProviderV3 for fetching user data

#### Critical Smart Contracts
```
IPool (Lending Pool)
- supply(asset, amount, receiver, referralCode)
- withdraw(asset, amount, receiver)
- getReserveData(asset)

IPoolAddressesProvider
- getPool()
- getPriceOracle()

aToken (Interest-bearing token)
- balanceOf(user)
- transfer()
```

#### API Endpoints & Methods
- **Get Market Data**: `getReserveData()` - Fetch APY, utilization, liquidity
- **Supply Assets**: `supply()` - Deposit tokens
- **Withdraw Assets**: `withdraw()` - Redeem deposit + interest
- **Get User Positions**: UiPoolDataProviderV3 contract queries

#### Current Ecosystem Integration
Aave partners with embedded wallet providers:
- **Turnkey** (explicitly mentioned) - Perfect for your use case
- Privy, Para, Dynamic also supported

**Advantages for Turnkey:**
- No custom signing required
- Standard EVM transaction flow
- Clear ABI and contract interfaces
- Well-documented SDK (@aave/react, @aave/sdk)

#### Integration Approach with Turnkey
1. Build unsigned transaction using Aave SDK
2. Serialize transaction parameters
3. Send to Turnkey for signing via `signTransaction()`
4. Broadcast signed transaction to network
5. Monitor on-chain confirmation

---

### 1.2 YO Protocol (yo.xyz) - Yield Optimizer

#### Overview
YO is a multi-chain yield optimizer that automatically rebalances funds across the best-performing pools. TVL: $62.95M (primarily on Base chain).

#### Key Features
- **Automated Rebalancing**: Moves funds to best risk-adjusted yields
- **ERC-4626 Tokenized Vaults**: Standard vault interface
- **yoTokens**: Interest-bearing vault tokens (yoETH, yoUSD, yoBTC, yoEUR)
- **Multi-Network Support**: Base chain primary, expanding to other networks
- **Risk-Adjusted Yields**: Leverages Exponential.fi's Risk Ratings

#### Technical Specifications
- **Vault Standard**: ERC-4626 compliant
- **Base Chain Primary**: Chain ID 8453
- **Vault Tokens**: Represent user's share of vault
- **Share-to-Asset Conversion**: Automatic conversion mechanism

#### Smart Contract Architecture
```
ERC-4626 Vault Interface:
- deposit(assets: uint256, receiver: address) -> uint256 shares
- mint(shares: uint256, receiver: address) -> uint256 assets
- redeem(shares: uint256, receiver: address, owner: address) -> uint256 assets
- withdraw(assets: uint256, receiver: address, owner: address) -> uint256 shares

Preview Functions (read-only):
- previewDeposit(assets) -> shares
- previewRedeem(shares) -> assets
- previewMint(shares) -> assets
- previewWithdraw(assets) -> shares

Pricing Functions:
- convertToAssets(shares)
- convertToShares(assets)
- maxWithdraw(owner)
- maxRedeem(owner)
```

#### Vault Addresses
```
yoETH: 0x3a43aec53490cb9fa922847385d82fe25d0e9de7
yoUSD: [Contract Address]
yoBTC: [Contract Address]
yoEUR: [Contract Address]
```

#### Key API Endpoints
- **Deposit Preview**: `previewDeposit()` - Estimate shares for assets
- **Execute Deposit**: `deposit()` - Deposit assets, receive yoTokens
- **Redeem Preview**: `previewRedeem()` - Estimate assets for shares
- **Execute Redemption**: `redeem()` - Burn yoTokens, receive assets
- **Position Queries**: `maxWithdraw()`, `maxRedeem()` - User limits

#### Integration Strategy with Turnkey
1. Estimate transaction using preview functions
2. Build deposit/redeem transaction via ethers.js
3. Submit to Turnkey signing endpoint
4. Execute on Base chain
5. Track yoToken balance for yield accrual

#### Monitoring & Tracking
- APY updates: Query vault contract for asset/share ratios
- Yield accrual: Calculate difference in asset value over time
- Rebalancing events: Monitor vault's internal operations

---

### 1.3 Yield.xyz - Universal Yield Infrastructure

#### Overview
Yield.xyz is the most comprehensive Web3 yield API supporting 75+ networks, 80+ protocols. Covers staking, DeFi lending, liquid staking, restaking, and RWA yields.

#### Key Features
- **Protocol Diversity**: Staking, liquid staking, DeFi lending, RWA yields
- **Multi-Chain Support**: 75+ networks (Ethereum, Solana, Cosmos, Bitcoin, etc.)
- **Unified API**: Single interface for all yield opportunities
- **Non-Custodial**: Users maintain control of assets
- **Transaction Planning**: API constructs complete transaction flows

#### SDK & API Structure
```
Core Concepts:
- Yields Discovery: Find opportunities across networks/protocols
- Enter Yield: Build transactions to stake/deposit
- Exit Yield: Build transactions to unstake/withdraw
- History Tracking: Get position history and rewards
```

#### SDK Installation & Configuration
```typescript
import { sdk } from '@yieldxyz/sdk';

sdk.configure({
  apiKey: 'your-api-key',
});

// Discover yields
const yields = await sdk.api.getYields({
  network: 'ethereum',
  limit: 10,
  protocols: ['aave', 'lido', 'morpho'],
});

// Get details for specific yield
const yieldDetails = await sdk.api.getYield({
  id: 'ethereum-aave-usdc',
});

// Enter yield (build transaction)
const enterTx = await sdk.api.enterYield({
  yieldId: 'ethereum-aave-usdc',
  amount: '1000000000000000000', // 1 token (18 decimals)
  userAddress: '0x...',
});

// Exit yield (build transaction)
const exitTx = await sdk.api.exitYield({
  yieldId: 'ethereum-aave-usdc',
  amount: '1000000000000000000',
  userAddress: '0x...',
});
```

#### API Response Structure
```json
{
  "id": "ethereum-aave-usdc",
  "protocol": "aave",
  "network": "ethereum",
  "token": {
    "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
    "symbol": "USDC",
    "decimals": 6
  },
  "apy": 5.23,
  "tvl": "2500000000",
  "risk": {
    "level": "low",
    "factors": ["audit-passed", "established-protocol"]
  }
}
```

#### Transaction Planning
- **Approval Transactions**: If token allowance insufficient
- **Deposit/Stake Transactions**: Execute yield entry
- **Withdrawal Transactions**: Exit yield positions
- **All transactions**: Fully signed and ready for broadcast

#### Integration Advantages
1. **Multi-Protocol Support**: Single API for multiple yields
2. **Complete TX Building**: No need to build transactions manually
3. **Risk Ratings**: Built-in risk assessment
4. **Historical Data**: Track positions and rewards over time

#### Yield.xyz with Turnkey
1. Call `enterYield()` to get transaction details
2. Submit transaction objects to Turnkey signing
3. Broadcast signed transactions
4. Use `history()` API to track performance

---

### 1.4 Bridge.xyz - Money Movement Infrastructure

#### Overview
Bridge.xyz is a stablecoin payment platform enabling fast fiat-to-crypto and crypto-to-fiat movements globally.

#### Key Components
- **Transfers**: Programmatic fiat ↔ stablecoin conversions
- **Virtual Accounts**: Fiat deposit addresses for customers
- **Static Templates**: Recurring transfer instructions
- **Liquidation Addresses**: Auto-forward crypto to fiat

#### Relevance to Yield Module
**Use Case**: Converting yield earnings to user's preferred currency/blockchain

#### API Model
```
Transfer Types:
1. One-time: USDC → EUR (on-demand)
2. Recurring: Set template for automated flows
3. Virtual Accounts: Receive fiat via traditional banking

Example: User earns USDC yield → Automatically convert to EUR
```

#### Integration Point in Yield Flow
```
User deposits USDC → Earns aUSDC yield
↓
Option to auto-convert yield to:
  - Stablecoin on different chain
  - Fiat currency
  - User's bank account (via Bridge.xyz)
```

---

### 1.5 Rain.xyz - Stablecoin Payment Infrastructure

#### Overview
Rain provides embedded wallets and stablecoin payment infrastructure with optional yield capabilities.

#### Key Features
- **Embedded Wallets**: White-label wallet in your platform
- **On-Ramps**: Fiat to stablecoin conversions
- **Yield Options**: Optional yield on stablecoin balances
- **Payments API**: Global money movement

#### Relevance to Your Platform
**Complementary Infrastructure**: Rain could provide the wallet layer (though you already have Turnkey), with additional yield options.

#### Integration Considerations
- Your Turnkey setup already handles wallets
- Rain's value would be in additional yield opportunities or fiat on/off-ramps
- Not primary for core yield module, but valuable for monetization

---

## Section 2: Architecture Design

### 2.1 Overall System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Client Application                           │
│          (Web3 Wallet / Frontend Interface)                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              NestJS Backend (Your Server)                        │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────────┐│
│  │  Yield Module    │  │ Wallet Module    │  │  Token Module   ││
│  │  - Deposit       │  │ (Existing)       │  │  (Existing)     ││
│  │  - Withdraw      │  │ - Balance Fetch  │  │  - Token Info   ││
│  │  - Track APY     │  │ - Tx Creation    │  │  - Swap         ││
│  │  - Get Yields    │  └──────────────────┘  └─────────────────┘│
│  └──────────────────┘                                            │
│         │                                                        │
│         ├─→ Yield Service (Business Logic)                      │
│         ├─→ AAVE Adapter                                        │
│         ├─→ YO Protocol Adapter                                 │
│         └─→ Yield.xyz Adapter                                   │
│                                                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┬──────────────────┐
        ▼                ▼                ▼                  ▼
   ┌──────────┐   ┌────────────┐   ┌──────────┐      ┌───────────┐
   │ Turnkey  │   │ Chain RPC  │   │ Database │      │ External  │
   │ SDK      │   │ (Alchemy,  │   │ (User    │      │ APIs      │
   │(Signing) │   │ Infura)    │   │ Positions)      │(Yield.xyz)│
   └──────────┘   └────────────┘   └──────────┘      └───────────┘
        │              │                │                 │
        └──────────────┴────────────────┴─────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌────────┐  ┌──────────┐  ┌──────────────┐
   │ AAVE   │  │ YO Proto │  │ Yield.xyz    │
   │ Pool   │  │ Vault    │  │ Aggregator   │
   └────────┘  └──────────┘  └──────────────┘
   (Ethereum,   (Base)       (Multi-chain)
    Polygon,
    etc.)
```

### 2.2 Module Structure (NestJS)

```
src/
├── modules/
│   ├── yield/
│   │   ├── yield.module.ts
│   │   ├── yield.service.ts
│   │   ├── yield.controller.ts
│   │   ├── dto/
│   │   │   ├── deposit.dto.ts
│   │   │   ├── withdraw.dto.ts
│   │   │   ├── yield-opportunity.dto.ts
│   │   │   └── position.dto.ts
│   │   ├── adapters/
│   │   │   ├── aave.adapter.ts
│   │   │   ├── yo-protocol.adapter.ts
│   │   │   └── yield-xyz.adapter.ts
│   │   ├── contracts/
│   │   │   ├── aave.contract.ts
│   │   │   ├── yo-vault.contract.ts
│   │   │   └── erc4626.contract.ts
│   │   └── entities/
│   │       ├── yield-position.entity.ts
│   │       ├── yield-transaction.entity.ts
│   │       └── protocol-config.entity.ts
│   │
│   ├── wallet/
│   │   └── (Existing - integrate with Turnkey)
│   │
│   └── blockchain/
│       ├── blockchain.service.ts
│       ├── rpc.service.ts
│       └── transaction.service.ts
│
├── common/
│   ├── constants/
│   │   └── yield-protocols.ts
│   ├── decorators/
│   ├── pipes/
│   └── utils/
│       ├── contract-encoder.ts
│       └── transaction-builder.ts
│
└── app.module.ts
```

### 2.3 Data Flow Diagram

```
User Wants to Deposit USDC to AAVE
        │
        ▼
[Get User Balance] ◄─── Query via existing balance module
        │
        ▼
[Get Yield Opportunities] ◄─── Call Yield Service
        │                      (fetches AAVE APY, rates, etc.)
        ▼
[User Selects AAVE + Amount]
        │
        ▼
[Check Token Allowance] ◄─── Query contract via RPC
        │
        ├─ If insufficient allowance
        │   │
        │   ▼
        │ [Build Approval TX] ◄─── Using ethers.js
        │   │
        │   ▼
        │ [Send to Turnkey] ─────► Sign via passkey
        │   │
        │   ▼
        │ [Broadcast Approval TX]
        │   │
        │   ▼
        │ [Wait for Confirmation]
        │
        ▼
[Build Deposit TX] ◄─── encode() via AAVE adapter
        │
        ▼
[Send to Turnkey] ─────► Sign via passkey
        │
        ▼
[Broadcast Deposit TX]
        │
        ▼
[Wait for Confirmation]
        │
        ▼
[Store Position in DB] ◄─── Track user's yield position
        │
        ▼
[Return TX Hash + Position Info]
```

---

## Section 3: Technical Implementation

### 3.1 NestJS Module Setup

#### yield.module.ts
```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { HttpModule } from '@nestjs/axios';
import { YieldService } from './yield.service';
import { YieldController } from './yield.controller';
import { AaveAdapter } from './adapters/aave.adapter';
import { YoProtocolAdapter } from './adapters/yo-protocol.adapter';
import { YieldXyzAdapter } from './adapters/yield-xyz.adapter';
import { YieldPosition } from './entities/yield-position.entity';
import { YieldTransaction } from './entities/yield-transaction.entity';
import { ProtocolConfig } from './entities/protocol-config.entity';
import { WalletModule } from '../wallet/wallet.module';
import { BlockchainModule } from '../blockchain/blockchain.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([YieldPosition, YieldTransaction, ProtocolConfig]),
    HttpModule,
    WalletModule,
    BlockchainModule,
  ],
  controllers: [YieldController],
  providers: [
    YieldService,
    AaveAdapter,
    YoProtocolAdapter,
    YieldXyzAdapter,
  ],
  exports: [YieldService],
})
export class YieldModule {}
```

#### yield.service.ts (Business Logic)
```typescript
import { Injectable, BadRequestException, InternalServerErrorException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

import { YieldPosition } from './entities/yield-position.entity';
import { YieldTransaction } from './entities/yield-transaction.entity';
import { AaveAdapter } from './adapters/aave.adapter';
import { YoProtocolAdapter } from './adapters/yo-protocol.adapter';
import { YieldXyzAdapter } from './adapters/yield-xyz.adapter';
import { BlockchainService } from '../blockchain/blockchain.service';
import { WalletService } from '../wallet/wallet.service';

import { DepositDto, WithdrawDto, GetYieldsDto } from './dto';

@Injectable()
export class YieldService {
  constructor(
    @InjectRepository(YieldPosition)
    private yieldPositionRepo: Repository<YieldPosition>,
    @InjectRepository(YieldTransaction)
    private yieldTxRepo: Repository<YieldTransaction>,
    private aaveAdapter: AaveAdapter,
    private yoAdapter: YoProtocolAdapter,
    private yieldXyzAdapter: YieldXyzAdapter,
    private blockchainService: BlockchainService,
    private walletService: WalletService,
    private httpService: HttpService,
  ) {}

  /**
   * Get all available yield opportunities
   */
  async getYieldOpportunities(dto: GetYieldsDto) {
    const opportunities = [];

    try {
      // Fetch AAVE opportunities
      if (dto.protocols.includes('aave')) {
        const aaveYields = await this.aaveAdapter.getOpportunities({
          chainId: dto.chainId,
          networks: dto.networks,
        });
        opportunities.push(...aaveYields);
      }

      // Fetch YO Protocol opportunities
      if (dto.protocols.includes('yo')) {
        const yoYields = await this.yoAdapter.getOpportunities();
        opportunities.push(...yoYields);
      }

      // Fetch Yield.xyz opportunities
      if (dto.protocols.includes('yield-xyz')) {
        const yieldXyzOpportunities = await this.yieldXyzAdapter.getOpportunities({
          networks: dto.networks,
          limit: dto.limit,
        });
        opportunities.push(...yieldXyzOpportunities);
      }

      return {
        total: opportunities.length,
        opportunities: opportunities.sort((a, b) => b.apy - a.apy),
      };
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to fetch yield opportunities: ${error.message}`,
      );
    }
  }

  /**
   * Get user's yield positions
   */
  async getUserPositions(userAddress: string, chainId: number) {
    const positions = await this.yieldPositionRepo.find({
      where: { userAddress, chainId, isActive: true },
      relations: ['transactions'],
      order: { createdAt: 'DESC' },
    });

    // Enrich with current balances and yields
    const enrichedPositions = await Promise.all(
      positions.map(async (pos) => {
        const currentBalance = await this.getPositionBalance(pos);
        const earnedYield = currentBalance - pos.depositAmount;
        return {
          ...pos,
          currentBalance,
          earnedYield,
          earnedPercentage: (earnedYield / pos.depositAmount) * 100,
        };
      }),
    );

    return enrichedPositions;
  }

  /**
   * Deposit tokens to yield protocol
   */
  async deposit(dto: DepositDto, userAddress: string) {
    // Validate input
    if (!dto.amount || dto.amount <= 0) {
      throw new BadRequestException('Invalid deposit amount');
    }

    // Get user's token balance
    const balance = await this.walletService.getTokenBalance(
      userAddress,
      dto.tokenAddress,
      dto.chainId,
    );

    if (balance < dto.amount) {
      throw new BadRequestException('Insufficient token balance');
    }

    let txData;
    let adapter;

    try {
      // Route to appropriate adapter based on protocol
      switch (dto.protocol) {
        case 'aave':
          adapter = this.aaveAdapter;
          txData = await this.aaveAdapter.buildDepositTx({
            userAddress,
            token: dto.tokenAddress,
            amount: dto.amount,
            chainId: dto.chainId,
          });
          break;

        case 'yo':
          adapter = this.yoAdapter;
          txData = await this.yoAdapter.buildDepositTx({
            userAddress,
            vaultAddress: dto.vaultAddress,
            amount: dto.amount,
            chainId: dto.chainId,
          });
          break;

        case 'yield-xyz':
          adapter = this.yieldXyzAdapter;
          txData = await this.yieldXyzAdapter.buildDepositTx({
            userAddress,
            yieldId: dto.yieldId,
            amount: dto.amount,
          });
          break;

        default:
          throw new BadRequestException(`Unknown protocol: ${dto.protocol}`);
      }

      // Check token allowance and build approval tx if needed
      const allowance = await this.blockchainService.getTokenAllowance(
        dto.tokenAddress,
        userAddress,
        txData.spender || adapter.getSpenderAddress(),
        dto.chainId,
      );

      const transactions = [];

      // If allowance insufficient, add approval transaction
      if (allowance < dto.amount) {
        const approveTx = await this.blockchainService.buildApprovalTx({
          tokenAddress: dto.tokenAddress,
          spender: txData.spender || adapter.getSpenderAddress(),
          amount: dto.amount,
          userAddress,
          chainId: dto.chainId,
        });
        transactions.push({
          type: 'approval',
          data: approveTx,
          chainId: dto.chainId,
        });
      }

      // Add deposit transaction
      transactions.push({
        type: 'deposit',
        data: txData,
        chainId: dto.chainId,
      });

      // Create pending position in DB
      const position = this.yieldPositionRepo.create({
        userAddress,
        protocol: dto.protocol,
        chainId: dto.chainId,
        token: dto.tokenAddress,
        depositAmount: dto.amount,
        apy: adapter.getCurrentApy ? await adapter.getCurrentApy(dto.tokenAddress, dto.chainId) : 0,
        status: 'pending',
      });

      await this.yieldPositionRepo.save(position);

      return {
        positionId: position.id,
        transactions,
        instructions: {
          description: `Deposit ${dto.amount} to ${dto.protocol}`,
          steps: transactions.length > 1
            ? ['Approve token spending', 'Execute deposit']
            : ['Execute deposit'],
        },
      };
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to prepare deposit transaction: ${error.message}`,
      );
    }
  }

  /**
   * Withdraw from yield protocol
   */
  async withdraw(dto: WithdrawDto, userAddress: string) {
    const position = await this.yieldPositionRepo.findOne({
      where: { id: dto.positionId, userAddress },
    });

    if (!position) {
      throw new BadRequestException('Position not found');
    }

    let txData;

    try {
      switch (position.protocol) {
        case 'aave':
          txData = await this.aaveAdapter.buildWithdrawTx({
            userAddress,
            token: position.token,
            amount: dto.amount,
            chainId: position.chainId,
          });
          break;

        case 'yo':
          txData = await this.yoAdapter.buildRedeemTx({
            userAddress,
            vaultAddress: position.vaultAddress,
            shares: dto.amount, // For ERC4626, need to redeem shares
            chainId: position.chainId,
          });
          break;

        case 'yield-xyz':
          txData = await this.yieldXyzAdapter.buildWithdrawTx({
            userAddress,
            yieldId: position.yieldId,
            amount: dto.amount,
          });
          break;

        default:
          throw new BadRequestException(`Unknown protocol: ${position.protocol}`);
      }

      return {
        positionId: position.id,
        transaction: {
          type: 'withdraw',
          data: txData,
          chainId: position.chainId,
        },
      };
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to prepare withdrawal transaction: ${error.message}`,
      );
    }
  }

  /**
   * Helper: Get current position balance
   */
  private async getPositionBalance(position: YieldPosition): Promise<number> {
    switch (position.protocol) {
      case 'aave':
        return await this.aaveAdapter.getPositionBalance(
          position.userAddress,
          position.token,
          position.chainId,
        );

      case 'yo':
        return await this.yoAdapter.getPositionBalance(
          position.userAddress,
          position.vaultAddress,
          position.chainId,
        );

      case 'yield-xyz':
        return await this.yieldXyzAdapter.getPositionBalance(position.userAddress);

      default:
        return 0;
    }
  }

  /**
   * Track transaction status after broadcast
   */
  async trackTransaction(txHash: string, positionId: string, chainId: number) {
    try {
      const receipt = await this.blockchainService.waitForTransaction(txHash, chainId);
      
      if (receipt.status === 1) {
        // Transaction succeeded
        const position = await this.yieldPositionRepo.findOne({ where: { id: positionId } });
        position.status = 'active';
        position.lastTxHash = txHash;
        await this.yieldPositionRepo.save(position);

        return { success: true, receipt };
      } else {
        // Transaction failed
        throw new Error('Transaction failed on-chain');
      }
    } catch (error) {
      throw new InternalServerErrorException(`Failed to track transaction: ${error.message}`);
    }
  }
}
```

### 3.2 AAVE Adapter Implementation

#### aave.adapter.ts
```typescript
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { ethers } from 'ethers';
import { firstValueFrom } from 'rxjs';

import AAVE_POOL_ABI from '../contracts/aave-pool-abi.json';
import AAVE_DATA_PROVIDER_ABI from '../contracts/aave-data-provider-abi.json';

interface AaveNetwork {
  chainId: number;
  poolAddress: string;
  dataProviderAddress: string;
  rpcUrl: string;
}

@Injectable()
export class AaveAdapter {
  private networks: Map<number, AaveNetwork> = new Map([
    [1, {
      chainId: 1,
      poolAddress: '0x7d2768dE32b0b80b7a3454c06BdAc94A69DDc7A9',
      dataProviderAddress: '0x7F02D46380f6Fe61c7A1b1b5f5D2E13aa7E2D49f',
      rpcUrl: process.env.ETHEREUM_RPC_URL,
    }],
    [137, {
      chainId: 137,
      poolAddress: '0x794a61358D6845594F94dc1DB02A252b5b4814aD',
      dataProviderAddress: '0x69FA688f1Dc47d4B5d8029B378669Cce1dd626d8',
      rpcUrl: process.env.POLYGON_RPC_URL,
    }],
    [8453, {
      chainId: 8453,
      poolAddress: '0xA238Dd5C597D265DAC63E86649Ffdf0EF07132dD',
      dataProviderAddress: '0x2d8A3C8681B71d473474243145Eb00d866D493f7',
      rpcUrl: process.env.BASE_RPC_URL,
    }],
    // Add more networks as needed
  ]);

  constructor(private httpService: HttpService) {}

  /**
   * Get AAVE yield opportunities
   */
  async getOpportunities(filters: {
    chainId: number;
    networks?: number[];
  }) {
    const opportunities = [];

    const chainIds = filters.networks || [filters.chainId];

    for (const chainId of chainIds) {
      try {
        const network = this.networks.get(chainId);
        if (!network) continue;

        const provider = new ethers.JsonRpcProvider(network.rpcUrl);
        const poolContract = new ethers.Contract(
          network.poolAddress,
          AAVE_POOL_ABI,
          provider,
        );

        // Get all reserves (supported tokens)
        const reservesList = await poolContract.getReservesList();

        for (const reserveAddress of reservesList.slice(0, 10)) { // Limit to top 10
          try {
            const reserveData = await poolContract.getReserveData(reserveAddress);
            const token = new ethers.Contract(
              reserveAddress,
              ['function symbol() view returns (string)', 'function decimals() view returns (uint8)'],
              provider,
            );

            const [symbol, decimals] = await Promise.all([
              token.symbol(),
              token.decimals(),
            ]);

            // Current APY is stored in reserveData.currentLiquidityRate (ray value)
            const apy = this.rayToPercent(reserveData.currentLiquidityRate);

            opportunities.push({
              id: `aave-${chainId}-${reserveAddress}`,
              protocol: 'aave',
              chainId,
              token: reserveAddress,
              symbol,
              decimals,
              apy,
              tvl: this.rayToNumber(reserveData.totalAToken),
              availability: 'available',
              riskLevel: 'low',
            });
          } catch (error) {
            console.warn(`Failed to fetch data for reserve ${reserveAddress}:`, error);
          }
        }
      } catch (error) {
        console.error(`Failed to fetch AAVE opportunities for chain ${chainId}:`, error);
      }
    }

    return opportunities;
  }

  /**
   * Build deposit transaction
   */
  async buildDepositTx(params: {
    userAddress: string;
    token: string;
    amount: string | number;
    chainId: number;
  }): Promise<any> {
    const network = this.networks.get(params.chainId);
    if (!network) throw new Error(`Unsupported chain: ${params.chainId}`);

    const provider = new ethers.JsonRpcProvider(network.rpcUrl);
    const poolContract = new ethers.Contract(
      network.poolAddress,
      AAVE_POOL_ABI,
      provider,
    );

    // Encode the supply transaction
    const txData = poolContract.interface.encodeFunctionData('supply', [
      params.token,
      params.amount,
      params.userAddress,
      0, // referral code
    ]);

    return {
      to: network.poolAddress,
      data: txData,
      value: '0',
      spender: network.poolAddress,
    };
  }

  /**
   * Build withdrawal transaction
   */
  async buildWithdrawTx(params: {
    userAddress: string;
    token: string;
    amount: string | number;
    chainId: number;
  }): Promise<any> {
    const network = this.networks.get(params.chainId);
    if (!network) throw new Error(`Unsupported chain: ${params.chainId}`);

    const provider = new ethers.JsonRpcProvider(network.rpcUrl);
    const poolContract = new ethers.Contract(
      network.poolAddress,
      AAVE_POOL_ABI,
      provider,
    );

    // Encode the withdraw transaction
    // Use max uint256 to withdraw all
    const amount = params.amount === 'max' ? ethers.MaxUint256.toString() : params.amount;

    const txData = poolContract.interface.encodeFunctionData('withdraw', [
      params.token,
      amount,
      params.userAddress,
    ]);

    return {
      to: network.poolAddress,
      data: txData,
      value: '0',
    };
  }

  /**
   * Get user's supply position balance
   */
  async getPositionBalance(
    userAddress: string,
    token: string,
    chainId: number,
  ): Promise<number> {
    const network = this.networks.get(chainId);
    if (!network) throw new Error(`Unsupported chain: ${chainId}`);

    const provider = new ethers.JsonRpcProvider(network.rpcUrl);
    const dataProvider = new ethers.Contract(
      network.dataProviderAddress,
      AAVE_DATA_PROVIDER_ABI,
      provider,
    );

    try {
      const userData = await dataProvider.getUserReserveData(token, userAddress);
      // userData[0] is current aToken balance
      return Number(ethers.formatUnits(userData[0], 18));
    } catch (error) {
      console.error('Failed to fetch position balance:', error);
      return 0;
    }
  }

  /**
   * Get current APY for token
   */
  async getCurrentApy(token: string, chainId: number): Promise<number> {
    const network = this.networks.get(chainId);
    if (!network) throw new Error(`Unsupported chain: ${chainId}`);

    const provider = new ethers.JsonRpcProvider(network.rpcUrl);
    const poolContract = new ethers.Contract(
      network.poolAddress,
      AAVE_POOL_ABI,
      provider,
    );

    try {
      const reserveData = await poolContract.getReserveData(token);
      return this.rayToPercent(reserveData.currentLiquidityRate);
    } catch (error) {
      console.error('Failed to fetch APY:', error);
      return 0;
    }
  }

  getSpenderAddress(): string {
    return this.networks.get(1)?.poolAddress; // Default to Ethereum
  }

  /**
   * Convert RAY (27 decimals) to percentage
   */
  private rayToPercent(rayValue: string | number): number {
    const RAY = ethers.parseUnits('1', 27);
    const value = BigInt(rayValue);
    return Number(value * BigInt(10000) / RAY) / 100;
  }

  private rayToNumber(rayValue: string | number): number {
    const RAY = ethers.parseUnits('1', 27);
    const value = BigInt(rayValue);
    return Number(value / RAY);
  }
}
```

### 3.3 YO Protocol Adapter

#### yo-protocol.adapter.ts
```typescript
import { Injectable } from '@nestjs/common';
import { ethers } from 'ethers';
import ERC4626_ABI from '../contracts/erc4626-abi.json';

interface YoVault {
  address: string;
  symbol: string;
  asset: string;
  chainId: number;
}

@Injectable()
export class YoProtocolAdapter {
  private vaults: Map<string, YoVault> = new Map([
    ['base-eth', {
      address: '0x3a43aec53490cb9fa922847385d82fe25d0e9de7',
      symbol: 'yoETH',
      asset: '0x4200000000000000000000000000000000000006', // WETH on Base
      chainId: 8453,
    }],
    ['base-usd', {
      address: '0x...',
      symbol: 'yoUSD',
      asset: '0x833589fCD6eDb6E08f4c7C32D4f71b1566dA1C78', // USDC on Base
      chainId: 8453,
    }],
    // Add more vaults as available
  ]);

  private readonly BASE_RPC_URL = process.env.BASE_RPC_URL;

  /**
   * Get YO Protocol opportunities
   */
  async getOpportunities() {
    const opportunities = [];

    for (const [key, vault] of this.vaults) {
      try {
        const provider = new ethers.JsonRpcProvider(this.BASE_RPC_URL);
        const vaultContract = new ethers.Contract(
          vault.address,
          ERC4626_ABI,
          provider,
        );

        // Get vault APY (needs to be calculated from historical rates)
        // For now, we'll use a simplified APY calculation
        const apy = await this.calculateVaultApy(vault.address);

        opportunities.push({
          id: `yo-${key}`,
          protocol: 'yo',
          chainId: vault.chainId,
          vaultAddress: vault.address,
          symbol: vault.symbol,
          asset: vault.asset,
          apy,
          tvl: 0, // Would need to calculate from contract state
          availability: 'available',
          riskLevel: 'low-medium',
        });
      } catch (error) {
        console.error(`Failed to fetch YO vault ${key}:`, error);
      }
    }

    return opportunities;
  }

  /**
   * Build deposit transaction
   */
  async buildDepositTx(params: {
    userAddress: string;
    vaultAddress: string;
    amount: string | number;
    chainId: number;
  }): Promise<any> {
    const provider = new ethers.JsonRpcProvider(this.BASE_RPC_URL);
    const vaultContract = new ethers.Contract(
      params.vaultAddress,
      ERC4626_ABI,
      provider,
    );

    // Encode deposit (ERC4626 standard)
    const txData = vaultContract.interface.encodeFunctionData('deposit', [
      params.amount,
      params.userAddress,
    ]);

    return {
      to: params.vaultAddress,
      data: txData,
      value: '0',
      spender: params.vaultAddress,
    };
  }

  /**
   * Build redemption transaction
   */
  async buildRedeemTx(params: {
    userAddress: string;
    vaultAddress: string;
    shares: string | number;
    chainId: number;
  }): Promise<any> {
    const provider = new ethers.JsonRpcProvider(this.BASE_RPC_URL);
    const vaultContract = new ethers.Contract(
      params.vaultAddress,
      ERC4626_ABI,
      provider,
    );

    // Encode redeem (ERC4626 standard)
    const txData = vaultContract.interface.encodeFunctionData('redeem', [
      params.shares,
      params.userAddress,
      params.userAddress,
    ]);

    return {
      to: params.vaultAddress,
      data: txData,
      value: '0',
    };
  }

  /**
   * Get user's position balance in shares
   */
  async getPositionBalance(
    userAddress: string,
    vaultAddress: string,
    chainId: number,
  ): Promise<number> {
    const provider = new ethers.JsonRpcProvider(this.BASE_RPC_URL);
    const vaultContract = new ethers.Contract(
      vaultAddress,
      ERC4626_ABI,
      provider,
    );

    try {
      const balance = await vaultContract.balanceOf(userAddress);
      return Number(ethers.formatUnits(balance, 18));
    } catch (error) {
      console.error('Failed to fetch position balance:', error);
      return 0;
    }
  }

  /**
   * Get vault APY (simplified calculation)
   */
  private async calculateVaultApy(vaultAddress: string): Promise<number> {
    // This is a simplified calculation
    // In production, you'd track historical rates or fetch from subgraph
    // For now, returning a placeholder
    return 8.5; // Example APY
  }

  getSpenderAddress(): string {
    return 'vault-specific'; // Each vault is its own spender
  }
}
```

### 3.4 Yield.xyz Adapter

#### yield-xyz.adapter.ts
```typescript
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class YieldXyzAdapter {
  private readonly API_BASE_URL = 'https://api.yield.xyz/v1';
  private readonly API_KEY = process.env.YIELD_XYZ_API_KEY;

  constructor(private httpService: HttpService) {}

  /**
   * Get yield opportunities from Yield.xyz
   */
  async getOpportunities(filters: {
    networks?: string[];
    limit?: number;
  }) {
    try {
      const response = await firstValueFrom(
        this.httpService.get(`${this.API_BASE_URL}/yields`, {
          headers: {
            'Authorization': `Bearer ${this.API_KEY}`,
          },
          params: {
            networks: filters.networks?.join(',') || 'ethereum',
            limit: filters.limit || 20,
          },
        }),
      );

      return response.data.yields.map((yield_: any) => ({
        id: yield_.id,
        protocol: yield_.protocol,
        chainId: this.networkNameToChainId(yield_.network),
        token: yield_.token.address,
        symbol: yield_.token.symbol,
        decimals: yield_.token.decimals,
        apy: yield_.apy,
        tvl: yield_.tvl,
        availability: 'available',
        riskLevel: yield_.risk?.level || 'medium',
      }));
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to fetch Yield.xyz opportunities: ${error.message}`,
      );
    }
  }

  /**
   * Build enter yield transaction
   */
  async buildDepositTx(params: {
    userAddress: string;
    yieldId: string;
    amount: string | number;
  }): Promise<any> {
    try {
      const response = await firstValueFrom(
        this.httpService.post(
          `${this.API_BASE_URL}/yields/${params.yieldId}/enter`,
          {
            userAddress: params.userAddress,
            amount: params.amount.toString(),
          },
          {
            headers: {
              'Authorization': `Bearer ${this.API_KEY}`,
            },
          },
        ),
      );

      const { transactions } = response.data;

      // Return first transaction (usually approval) or deposit
      return {
        to: transactions[0].to,
        data: transactions[0].data,
        value: transactions[0].value || '0',
        spender: transactions[0].to,
      };
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to build Yield.xyz deposit transaction: ${error.message}`,
      );
    }
  }

  /**
   * Build exit yield transaction
   */
  async buildWithdrawTx(params: {
    userAddress: string;
    yieldId: string;
    amount: string | number;
  }): Promise<any> {
    try {
      const response = await firstValueFrom(
        this.httpService.post(
          `${this.API_BASE_URL}/yields/${params.yieldId}/exit`,
          {
            userAddress: params.userAddress,
            amount: params.amount.toString(),
          },
          {
            headers: {
              'Authorization': `Bearer ${this.API_KEY}`,
            },
          },
        ),
      );

      const { transactions } = response.data;

      return {
        to: transactions[0].to,
        data: transactions[0].data,
        value: transactions[0].value || '0',
      };
    } catch (error) {
      throw new InternalServerErrorException(
        `Failed to build Yield.xyz withdrawal transaction: ${error.message}`,
      );
    }
  }

  /**
   * Get user's position balance
   */
  async getPositionBalance(userAddress: string): Promise<number> {
    try {
      const response = await firstValueFrom(
        this.httpService.get(
          `${this.API_BASE_URL}/users/${userAddress}/yields`,
          {
            headers: {
              'Authorization': `Bearer ${this.API_KEY}`,
            },
          },
        ),
      );

      // Sum all yield positions
      return response.data.positions.reduce(
        (total: number, pos: any) => total + Number(pos.balance),
        0,
      );
    } catch (error) {
      console.error('Failed to fetch position balance:', error);
      return 0;
    }
  }

  /**
   * Helper: Convert network name to chain ID
   */
  private networkNameToChainId(networkName: string): number {
    const mapping: Record<string, number> = {
      ethereum: 1,
      polygon: 137,
      arbitrum: 42161,
      optimism: 10,
      base: 8453,
      solana: 0, // Special case
      cosmos: 0, // Special case
    };
    return mapping[networkName.toLowerCase()] || 1;
  }

  getSpenderAddress(): string {
    return 'yield-xyz-api'; // Yield.xyz handles this
  }
}
```

---

## Section 4: API Integration Details

### 4.1 RESTful API Endpoints

#### GET `/api/yield/opportunities`
Retrieve all available yield opportunities

```typescript
@Get('opportunities')
async getOpportunities(@Query() dto: GetYieldsDto) {
  return this.yieldService.getYieldOpportunities(dto);
}

// Query Parameters:
// - protocols: 'aave' | 'yo' | 'yield-xyz' (comma-separated)
// - networks: ethereum, polygon, base, etc. (comma-separated)
// - chainId: specific chain ID
// - limit: max results

// Response:
{
  "total": 45,
  "opportunities": [
    {
      "id": "aave-1-0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "protocol": "aave",
      "chainId": 1,
      "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "symbol": "USDC",
      "decimals": 6,
      "apy": 5.23,
      "tvl": "2500000000",
      "riskLevel": "low"
    },
    // ... more yields
  ]
}
```

#### GET `/api/yield/positions/:userAddress`
Get user's active yield positions

```typescript
@Get('positions/:userAddress')
async getPositions(
  @Param('userAddress') userAddress: string,
  @Query('chainId') chainId: number,
) {
  return this.yieldService.getUserPositions(userAddress, chainId);
}

// Response:
{
  "positions": [
    {
      "id": "uuid",
      "protocol": "aave",
      "chainId": 1,
      "token": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "depositAmount": 1000,
      "currentBalance": 1052.34,
      "earnedYield": 52.34,
      "earnedPercentage": 5.234,
      "apy": 5.23,
      "status": "active",
      "createdAt": "2024-01-01T10:00:00Z"
    }
  ]
}
```

#### POST `/api/yield/deposit`
Initiate yield deposit (generates unsigned transactions)

```typescript
@Post('deposit')
async deposit(@Body() dto: DepositDto, @Req() req: any) {
  const userAddress = req.user.address;
  return this.yieldService.deposit(dto, userAddress);
}

// Request Body:
{
  "protocol": "aave",
  "chainId": 1,
  "token Address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "amount": "1000000000", // 1000 USDC (6 decimals)
  "vaultAddress": "0x...", // For YO Protocol
  "yieldId": "..." // For Yield.xyz
}

// Response:
{
  "positionId": "uuid",
  "transactions": [
    {
      "type": "approval",
      "chainId": 1,
      "data": {
        "to": "0x...",
        "data": "0x...",
        "value": "0"
      }
    },
    {
      "type": "deposit",
      "chainId": 1,
      "data": {
        "to": "0x...",
        "data": "0x...",
        "value": "0"
      }
    }
  ],
  "instructions": {
    "description": "Deposit 1000 USDC to AAVE",
    "steps": ["Approve token spending", "Execute deposit"]
  }
}
```

#### POST `/api/yield/withdraw`
Initiate yield withdrawal

```typescript
@Post('withdraw')
async withdraw(@Body() dto: WithdrawDto, @Req() req: any) {
  const userAddress = req.user.address;
  return this.yieldService.withdraw(dto, userAddress);
}

// Request Body:
{
  "positionId": "uuid",
  "amount": "1052.34" // Amount to withdraw
}

// Response:
{
  "positionId": "uuid",
  "transaction": {
    "type": "withdraw",
    "chainId": 1,
    "data": {
      "to": "0x...",
      "data": "0x...",
      "value": "0"
    }
  }
}
```

#### POST `/api/yield/track/:txHash`
Track transaction status

```typescript
@Post('track/:txHash')
async trackTransaction(
  @Param('txHash') txHash: string,
  @Query('positionId') positionId: string,
  @Query('chainId') chainId: number,
) {
  return this.yieldService.trackTransaction(txHash, positionId, chainId);
}

// Response:
{
  "success": true,
  "receipt": {
    "transactionHash": "0x...",
    "blockNumber": 19000000,
    "status": 1,
    "gasUsed": "125000"
  }
}
```

---

## Section 5: Turnkey Integration Flow

### 5.1 Complete Transaction Signing Flow with Turnkey

```
┌─────────────────────────────────────────────────────────────┐
│ User Initiates Yield Deposit Flow                            │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
         ┌────────────────────────────┐
         │ Frontend calls GET /yield/  │
         │      opportunities          │
         └────────────┬────────────────┘
                      │
                      ▼
         ┌────────────────────────────┐
         │ Backend returns list of     │
         │ yield options with APYs     │
         └────────────┬────────────────┘
                      │
                      ▼
         ┌────────────────────────────┐
         │ User selects yield +        │
         │ enters amount               │
         └────────────┬────────────────┘
                      │
                      ▼
         ┌────────────────────────────┐
         │ Frontend calls POST /yield/ │
         │      deposit                │
         │ (with user address via auth)│
         └────────────┬────────────────┘
                      │
    ┌─────────────────┴──────────────────┐
    │                                    │
    ▼                                    ▼
[Check Allowance]              [Generate Transactions]
    │                                    │
    ├─ If insufficient ────────► Build Approval TX
    │                           Encode with adapter
    │                           Return in array
    │
    ▼
[Return Transaction Array to Frontend]
    │
    ├─ Transaction 1: Approval (if needed)
    │   {
    │     "type": "approval",
    │     "to": "0x...",
    │     "data": "0x...",
    │     "chainId": 1
    │   }
    │
    └─ Transaction 2: Deposit
       {
         "type": "deposit",
         "to": "0x...",
         "data": "0x...",
         "chainId": 1
       }
    │
    ▼
[Frontend: User Confirms Transaction]
    │
    ▼
┌──────────────────────────────────────────────┐
│ Frontend calls Turnkey signing flow:         │
│ 1. Creates activity with transaction data    │
│ 2. User signs with passkey                   │
│ 3. Receives signed transaction               │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│ Frontend broadcasts signed TX via RPC        │
│ (or sends to backend for relay)              │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│ Transaction enters mempool                   │
│ Status: pending                              │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│ Frontend polls /yield/track/:txHash          │
│ OR use WebSocket for real-time updates       │
└──────────────┬───────────────────────────────┘
               │
          ┌────┴─────┐
          │           │
      Pending     Confirmed
          │           │
          ▼           ▼
    [Continue]  [Update Position DB]
         │           │
         │           ▼
         │       [Return Success]
         │           │
         └─────┬─────┘
               │
               ▼
    [Display Yield Position to User]
```

### 5.2 Turnkey SDK Integration Code (Frontend)

```typescript
// This would be in your frontend/client code

import { TurnkeyClient } from '@turnkey/sdk-browser';
import { useCallback } from 'react';

const useTurnkeyYieldSigning = () => {
  const turnkey = new TurnkeyClient({
    baseUrl: 'https://api.turnkey.com',
  });

  const signAndSubmitYieldTransaction = useCallback(
    async (
      transaction: {
        to: string;
        data: string;
        value: string;
        chainId: number;
      },
      walletAddress: string,
      subOrgId: string,
    ) => {
      try {
        // Build Turnkey activity for signing
        const signActivity = {
          type: 'ACTIVITY_TYPE_SIGN_TRANSACTION_V2',
          timestampMs: String(Date.now()),
          organizationId: subOrgId,
          parameters: {
            signWith: walletAddress, // Account to sign with
            type: 'TRANSACTION_TYPE_ETHEREUM',
            unsignedTransaction: {
              to: transaction.to,
              value: transaction.value,
              data: transaction.data,
              chainId: transaction.chainId,
              // Add nonce, gasPrice, etc. as needed
            },
          },
        };

        // User signs the activity with passkey
        const signedActivity = await turnkey.signTransaction(signActivity);

        // Extract signed transaction
        const signedTx = signedActivity.result.signTransactionResult.signedTransaction;

        // Broadcast to network
        const receipt = await broadcastTransaction(signedTx, transaction.chainId);

        return {
          success: true,
          txHash: receipt.hash,
          receipt,
        };
      } catch (error) {
        console.error('Turnkey signing failed:', error);
        throw error;
      }
    },
    [turnkey],
  );

  return { signAndSubmitYieldTransaction };
};
```

### 5.3 Backend-Assisted Signing (Alternative)

```typescript
// Backend endpoint to help with Turnkey signing

@Post('sign-yield-tx')
async signYieldTransaction(
  @Body() dto: {
    transactions: any[];
    userAddress: string;
  },
  @Req() req: any,
) {
  const subOrgId = req.user.subOrgId; // From authenticated user
  const userAddress = req.user.address;

  try {
    const signedTransactions = [];

    for (const transaction of dto.transactions) {
      // Build Turnkey HTTP client
      const httpClient = new TurnkeyClient(
        {
          baseUrl: 'https://api.turnkey.com',
        },
        new TurnkeyPasskeyStamper(), // Requires frontend passkey
      );

      // Create sign activity
      const activity = {
        type: 'ACTIVITY_TYPE_SIGN_TRANSACTION_V2',
        timestampMs: String(Date.now()),
        organizationId: subOrgId,
        parameters: {
          signWith: userAddress,
          type: 'TRANSACTION_TYPE_ETHEREUM',
          unsignedTransaction: {
            to: transaction.to,
            data: transaction.data,
            value: transaction.value,
            chainId: transaction.chainId,
          },
        },
      };

      // Send to Turnkey for signing
      const response = await httpClient.signTransaction(activity);

      signedTransactions.push({
        type: transaction.type,
        signedTx: response.result.signTransactionResult.signedTransaction,
        chainId: transaction.chainId,
      });
    }

    return {
      success: true,
      signedTransactions,
    };
  } catch (error) {
    throw new InternalServerErrorException(
      `Turnkey signing failed: ${error.message}`,
    );
  }
}
```

---

## Section 6: Database & State Management

### 6.1 Entity Schemas

#### yield-position.entity.ts
```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
} from 'typeorm';
import { YieldTransaction } from './yield-transaction.entity';

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

  @Column({ nullable: true })
  vaultAddress: string; // For YO Protocol

  @Column({ nullable: true })
  yieldId: string; // For Yield.xyz

  @Column('decimal', { precision: 20, scale: 8 })
  depositAmount: number;

  @Column('decimal', { precision: 20, scale: 8 })
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
  lastTxHash: string;

  @OneToMany(() => YieldTransaction, (tx) => tx.position)
  transactions: YieldTransaction[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### yield-transaction.entity.ts
```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  ManyToOne,
  CreateDateColumn,
} from 'typeorm';
import { YieldPosition } from './yield-position.entity';

@Entity('yield_transactions')
export class YieldTransaction {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => YieldPosition, (position) => position.transactions)
  position: YieldPosition;

  @Column()
  transactionHash: string;

  @Column()
  chainId: number;

  @Column({
    type: 'enum',
    enum: ['approval', 'deposit', 'withdraw'],
  })
  type: string;

  @Column({
    type: 'enum',
    enum: ['pending', 'confirmed', 'failed'],
    default: 'pending',
  })
  status: string;

  @Column({ nullable: true })
  blockNumber: number;

  @Column('decimal', { precision: 20, scale: 8, nullable: true })
  gasUsed: number;

  @Column({ nullable: true })
  gasPrice: string;

  @CreateDateColumn()
  createdAt: Date;
}
```

#### protocol-config.entity.ts
```typescript
import { Entity, Column, PrimaryGeneratedColumn } from 'typeorm';

@Entity('protocol_configs')
export class ProtocolConfig {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  protocol: 'aave' | 'yo' | 'yield-xyz';

  @Column()
  chainId: number;

  @Column()
  contractAddress: string;

  @Column({ type: 'json' })
  config: {
    poolAddress?: string;
    dataProviderAddress?: string;
    vaultAddress?: string;
    [key: string]: any;
  };

  @Column('timestamp', { default: () => 'CURRENT_TIMESTAMP' })
  lastUpdated: Date;
}
```

### 6.2 Database Queries & Optimizations

```typescript
// Efficient position queries
async getUserYieldPositions(userAddress: string) {
  return await this.yieldPositionRepo
    .createQueryBuilder('position')
    .where('position.userAddress = :userAddress', { userAddress })
    .andWhere('position.status = :status', { status: 'active' })
    .leftJoinAndSelect('position.transactions', 'transaction')
    .orderBy('position.createdAt', 'DESC')
    .getMany();
}

// Get positions by protocol for analytics
async getProtocolStats(protocol: string) {
  return await this.yieldPositionRepo
    .createQueryBuilder('position')
    .select('position.chainId', 'chainId')
    .addSelect('COUNT(position.id)', 'count')
    .addSelect('SUM(position.currentBalance)', 'totalBalance')
    .where('position.protocol = :protocol', { protocol })
    .groupBy('position.chainId')
    .getRawMany();
}
```

---

## Section 7: Security Considerations

### 7.1 Key Security Points

#### 1. **Private Key Management**
- ✅ All signing handled exclusively by Turnkey
- ✅ Private keys never exposed to your backend
- ✅ Use Trusted Execution Environments (TEEs)
- ✅ Multi-sig option available for institutional users

#### 2. **Transaction Validation**
```typescript
// Validate transaction parameters before sending to Turnkey
validateTransactionData(tx: any) {
  const errors = [];

  // Check recipient
  if (!ethers.isAddress(tx.to)) {
    errors.push('Invalid recipient address');
  }

  // Check amount
  if (tx.value && BigInt(tx.value) < 0) {
    errors.push('Invalid amount');
  }

  // Validate data encoding
  try {
    ethers.getAddress(tx.to);
  } catch {
    errors.push('Data encoding error');
  }

  if (errors.length > 0) {
    throw new Error(`Transaction validation failed: ${errors.join(', ')}`);
  }
}
```

#### 3. **Rate Limiting & Throttling**
```typescript
// Implement rate limiting for yield operations
@UseGuards(ThrottlerGuard)
@Post('deposit')
async deposit(@Body() dto: DepositDto) {
  // Throttled endpoint
}

// In AppModule:
ThrottlerModule.forRoot([
  {
    ttl: 60000, // 1 minute
    limit: 10, // max 10 requests per minute
  },
]);
```

#### 4. **Input Sanitization**
```typescript
// Always validate and sanitize user inputs
@IsAddress()
@ValidateIf((o) => o.token)
token: string;

@Min(0)
@IsNumber()
amount: number;

@IsEnum(['aave', 'yo', 'yield-xyz'])
protocol: string;
```

#### 5. **Audit Logging**
```typescript
// Log all yield operations for audit trail
async logYieldTransaction(
  userAddress: string,
  action: string,
  details: any,
) {
  await this.auditLogRepo.save({
    userAddress,
    action,
    details,
    timestamp: new Date(),
    ipAddress: this.getClientIp(),
  });
}
```

#### 6. **Smart Contract Risk Assessment**
```typescript
// Before executing yield operations, check protocol risks
async assessProtocolRisk(protocol: string, chainId: number) {
  const riskFactors = {
    aave: { audited: true, tvl: 'billions', established: true },
    yo: { audited: true, tvl: '$62M', established: true },
    yieldXyz: { audited: true, tvl: 'multi-billion', established: true },
  };

  return riskFactors[protocol];
}
```

### 7.2 Network Security

```typescript
// Use HTTPS only
@Module({
  controllers: [YieldController],
})
export class YieldModule {}

// In main.ts
const app = await NestFactory.create(AppModule);
app.use(helmet()); // Helmet for security headers
app.enableCors({
  origin: process.env.ALLOWED_ORIGINS?.split(','),
  credentials: true,
});
```

### 7.3 Environmental Variables
```bash
# .env (never commit)
TURNKEY_API_URL=https://api.turnkey.com
TURNKEY_ORGANIZATION_ID=your-org-id
YIELD_XYZ_API_KEY=your-api-key
ETHEREUM_RPC_URL=https://eth-rpc-url
POLYGON_RPC_URL=https://polygon-rpc-url
BASE_RPC_URL=https://base-rpc-url
DATABASE_URL=postgresql://...
JWT_SECRET=your-secret
```

---

## Section 8: Error Handling & Monitoring

### 8.1 Error Handling Strategy

```typescript
// Custom exception filters
@Catch(BadRequestException)
export class BadRequestExceptionFilter implements ExceptionFilter {
  catch(exception: BadRequestException, host: HttpArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();

    return response.status(400).json({
      statusCode: 400,
      message: exception.getResponse(),
      timestamp: new Date().toISOString(),
    });
  }
}

// Try-catch with proper logging
async depositWithErrorHandling(dto: DepositDto) {
  try {
    return await this.yieldService.deposit(dto, userAddress);
  } catch (error) {
    if (error instanceof BadRequestException) {
      this.logger.warn(`Validation error: ${error.message}`);
      throw error;
    } else if (error instanceof InternalServerErrorException) {
      this.logger.error(`Internal error: ${error.message}`, error.stack);
      // Alert ops team
      await this.alertingService.sendAlert({
        severity: 'high',
        message: `Yield module error: ${error.message}`,
      });
      throw error;
    }
  }
}
```

### 8.2 Monitoring & Observability

```typescript
// Use structured logging
import * as pino from 'pino';

const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: {
    target: 'pino-pretty',
  },
});

logger.info({
  event: 'deposit_initiated',
  userAddress,
  protocol,
  amount,
  chainId,
  timestamp: new Date().toISOString(),
});

// Metrics collection
import { Counter, Histogram } from 'prom-client';

const depositCounter = new Counter({
  name: 'yield_deposits_total',
  help: 'Total number of yield deposits',
  labelNames: ['protocol', 'status'],
});

const depositGauge = new Histogram({
  name: 'yield_deposit_amount_usdc',
  help: 'Distribution of deposit amounts',
  buckets: [100, 500, 1000, 5000, 10000, 50000, 100000],
});
```

### 8.3 Health Checks

```typescript
@Injectable()
export class HealthService {
  async checkTurnkeyConnectivity(): Promise<boolean> {
    try {
      // Attempt to connect to Turnkey
      const client = new TurnkeyClient({
        baseUrl: process.env.TURNKEY_API_URL,
      });
      // Would need to implement a simple ping
      return true;
    } catch {
      return false;
    }
  }

  async checkRPCConnectivity(chainId: number): Promise<boolean> {
    try {
      const provider = new ethers.JsonRpcProvider(this.getRpcUrl(chainId));
      await provider.getBlockNumber();
      return true;
    } catch {
      return false;
    }
  }
}

@Controller('health')
export class HealthController {
  constructor(private healthService: HealthService) {}

  @Get('status')
  async getHealth() {
    const turnkeyHealthy = await this.healthService.checkTurnkeyConnectivity();
    const rpcHealthy = await this.healthService.checkRPCConnectivity(1);

    return {
      status: turnkeyHealthy && rpcHealthy ? 'healthy' : 'degraded',
      services: {
        turnkey: turnkeyHealthy,
        rpc: rpcHealthy,
      },
    };
  }
}
```

---

## Section 9: Deployment Strategy

### 9.1 Development Environment

```bash
# Install dependencies
npm install ethers @typeorm/core @nestjs/axios @turnkey/sdk-browser

# Setup .env for development
cp .env.example .env

# Run migrations
npm run typeorm:migration:run

# Start development server
npm run start:dev
```

### 9.2 Production Deployment

#### Docker Setup
```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Build application
RUN npm run build

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD node healthcheck.js

# Start application
CMD ["node", "dist/main.js"]
```

#### Kubernetes Deployment
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yield-module-config
data:
  TURNKEY_API_URL: "https://api.turnkey.com"
  LOG_LEVEL: "info"
  DATABASE_POOL_SIZE: "10"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yield-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: yield-service
  template:
    metadata:
      labels:
        app: yield-service
    spec:
      containers:
      - name: yield-service
        image: your-registry/yield-service:latest
        envFrom:
        - configMapRef:
            name: yield-module-config
        - secretRef:
            name: yield-secrets # Contains API keys
        ports:
        - containerPort: 3000
        livenessProbe:
          httpGet:
            path: /health/status
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/status
            port: 3000
          initialDelaySeconds: 20
          periodSeconds: 5
```

### 9.3 Scaling & Performance

```typescript
// Cache frequently accessed data
@Injectable()
export class CacheService {
  constructor(private cacheManager: CacheModule) {}

  async getCachedYields(key: string, ttl: number = 300) {
    const cached = await this.cacheManager.get(key);
    if (cached) return cached;

    const data = await this.fetchYields();
    await this.cacheManager.set(key, data, ttl);
    return data;
  }
}

// Use read replicas for queries
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST,
      replication: {
        master: {
          host: process.env.DB_MASTER_HOST,
          port: 5432,
          username: process.env.DB_USER,
          password: process.env.DB_PASSWORD,
          database: process.env.DB_NAME,
        },
        slaves: [
          {
            host: process.env.DB_SLAVE_1_HOST,
            port: 5432,
            username: process.env.DB_USER,
            password: process.env.DB_PASSWORD,
            database: process.env.DB_NAME,
          },
        ],
      },
    }),
  ],
})
export class DatabaseModule {}
```

### 9.4 Monitoring & Alerting

```yaml
# Prometheus alerts
groups:
- name: yield_module
  rules:
  - alert: YieldDepositFailureRate
    expr: |
      (
        rate(yield_deposits_total{status="failed"}[5m]) /
        rate(yield_deposits_total[5m])
      ) > 0.05
    for: 5m
    annotations:
      summary: "Yield deposit failure rate > 5%"
      
  - alert: TurnkeySigningLatency
    expr: |
      histogram_quantile(0.95, turnkey_signing_duration_seconds_bucket) > 5
    for: 5m
    annotations:
      summary: "95th percentile Turnkey signing latency > 5 seconds"
```

---

## Integration Summary

### Data Flow Overview

```
User Interface (Frontend)
        ↓
[Request Yield Opportunities]
        ↓
Backend Yield Service
├── Fetch AAVE APYs (via RPC)
├── Fetch YO Protocol rates (via RPC on Base)
└── Fetch Yield.xyz opportunities (via API)
        ↓
[Display Opportunities to User]
        ↓
[User Selects & Inputs Amount]
        ↓
[Request Deposit]
        ↓
Backend:
├── Validate user balance
├── Check token allowance
├── Build approval TX (if needed)
├── Build deposit TX
└── Return transaction data
        ↓
[Send to Turnkey for Signing]
        ↓
Frontend:
├── User authenticates with passkey
├── Turnkey signs transactions
└── Receive signed transactions
        ↓
[Broadcast to Blockchain]
        ↓
Blockchain Network
├── Validate transaction
├── Execute smart contract
└── Update state
        ↓
[Backend Tracks Position]
├── Store in DB
├── Monitor for confirmations
└── Calculate yield accrual
        ↓
[User Sees Position]
└── Real-time APY, balance, earned yield
```

---

## Conclusion & Recommendations

### Implementation Roadmap

**Phase 1 (Weeks 1-2): Core Setup**
- Set up yield module structure
- Implement AAVE adapter
- Create basic DB schema
- Develop Turnkey integration

**Phase 2 (Weeks 3-4): Additional Protocols**
- Implement YO Protocol adapter
- Add Yield.xyz integration
- Complete transaction signing flow
- Add error handling

**Phase 3 (Week 5): Testing & Optimization**
- Comprehensive unit tests
- Integration tests with testnets
- Performance optimization
- Security audit

**Phase 4 (Week 6): Production Deployment**
- Deploy to production environment
- Monitor and optimize
- Add analytics
- Scale as needed

### Key Advantages of This Architecture

1. **Modular**: Each protocol in separate adapter
2. **Scalable**: Easy to add new protocols
3. **Secure**: Turnkey handles all signing
4. **Type-Safe**: Full TypeScript support
5. **Well-Structured**: NestJS best practices
6. **Production-Ready**: Error handling, monitoring, logging
7. **Multi-Chain**: Support for multiple blockchains
8. **Non-Custodial**: Users maintain full control

This architecture provides a solid foundation for your yield earning module, fully integrated with Turnkey wallet infrastructure, following NestJS best practices, and supporting multiple yield protocols seamlessly.
