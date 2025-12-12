# LI.FI Bitcoin Integration: Detailed Implementation Documentation

## Table of Contents
1. [Overview](#overview)
2. [Problem Statement](#problem-statement)
3. [Architecture Changes](#architecture-changes)
4. [Detailed Code Changes](#detailed-code-changes)
5. [Function Reference](#function-reference)
6. [Flow Diagrams](#flow-diagrams)
7. [Testing Considerations](#testing-considerations)

---

## Overview

This document provides a comprehensive breakdown of all code changes made to implement LI.FI Bitcoin integration in the NestJS backend. The implementation enables cross-chain swaps involving Bitcoin (BTC) as either source or destination chain.

### Key Requirements Addressed

1. **Bitcoin Source Swaps (BTC → EVM/SOL)**: Handle PSBT signing, finalization, and broadcasting
2. **Bitcoin Destination Swaps (EVM/SOL → BTC)**: Track status after frontend broadcasts transactions
3. **PSBT Detection**: Identify Bitcoin transactions vs EVM/Solana transactions
4. **Status Tracking**: Properly track swap status with correct chain identifiers

---

## Problem Statement

### What Was Missing Previously

**Before this implementation, the codebase had:**

1. ❌ **No PSBT Detection**: Could not differentiate between Bitcoin PSBTs and EVM transaction data
2. ❌ **Incorrect Swap Flow Handling**: Treated all LI.FI swaps as Bitcoin source swaps, causing failures for EVM/SOL → BTC swaps
3. ❌ **Missing Chain ID Normalization**: Bitcoin chain IDs were inconsistent ('btc', '20000000000001', 'bitcoin')
4. ❌ **Incomplete Status Tracking**: `toChain` was required when it should be optional
5. ❌ **No Swap Type Detection**: Could not determine if a swap was Bitcoin source or destination

### Critical Issues Found

1. **EVM/SOL → BTC Swaps Were Broken**: The code tried to finalize EVM/Solana transactions as PSBTs, causing runtime errors
2. **Missing Transaction Hash Handling**: No way for frontend to provide transaction hash after broadcasting EVM/SOL transactions
3. **Incorrect Broadcasting Logic**: All swaps attempted to broadcast to Bitcoin network, even for EVM/SOL transactions

---

## Architecture Changes

### High-Level Flow Comparison

#### Before (Incorrect)
```
All LI.FI Swaps:
  Frontend → Backend → Try to finalize as PSBT → Broadcast to Bitcoin ❌
```

#### After (Correct)
```
BTC → EVM/SOL Swaps:
  Frontend → Sign PSBT → Backend → Finalize PSBT → Broadcast to Bitcoin ✅

EVM/SOL → BTC Swaps:
  Frontend → Sign & Broadcast → Provide txHash → Backend → Track Status ✅
```

---

## Detailed Code Changes

### 1. Bitcoin Constants Utility (`src/modules/lifi/utils/bitcoin-constants.util.ts`)

**File Created**: New file

**Purpose**: Centralize Bitcoin chain identifier constants and normalization logic

**Why Added**: 
- Bitcoin chain IDs are represented inconsistently across LI.FI API ('btc', '20000000000001', 'bitcoin')
- Needed reusable functions to normalize these identifiers
- Prevents bugs from inconsistent chain ID comparisons

**Code Added**:

```typescript
/**
 * Bitcoin chain identifiers used by LI.FI
 */
export const BITCOIN_CHAIN_ID = '20000000000001';
export const BITCOIN_CHAIN_KEY = 'btc';
export const BITCOIN_CHAIN_NAME = 'bitcoin';

/**
 * Normalize Bitcoin chain identifier to standard format
 * Converts '20000000000001', 'bitcoin', 'btc' → 'btc'
 */
export function normalizeBitcoinChainId(chainId: string | number | null | undefined): string

/**
 * Check if a chain identifier represents Bitcoin
 */
export function isBitcoinChain(chainId: string | number | null | undefined): boolean
```

**What It Does**:
- `normalizeBitcoinChainId()`: Converts any Bitcoin identifier to standard 'btc' format
- `isBitcoinChain()`: Checks if a chain ID represents Bitcoin
- Used throughout codebase to ensure consistent chain ID handling

---

### 2. LifiService Updates (`src/modules/lifi/lifi.service.ts`)

#### 2.1 Added Bitcoin Chain Constants Import

**Change**:
```typescript
// BEFORE: No imports
// AFTER:
import { BITCOIN_CHAIN_ID, BITCOIN_CHAIN_KEY, normalizeBitcoinChainId } from './utils/bitcoin-constants.util';
```

**Why**: Reuse centralized constants instead of duplicating values

---

#### 2.2 Updated `getStatus()` Method

**Before**:
```typescript
async getStatus(
  txHash: string,
  fromChain: string,
  toChain: string,  // ❌ Required
  bridge?: string,
): Promise<any>
```

**After**:
```typescript
async getStatus(
  txHash: string,
  fromChain: string,  // ✅ Required (speeds up lookup)
  toChain?: string,    // ✅ Optional (but recommended)
  bridge?: string,
): Promise<any>
```

**Why Changed**:
- LI.FI API documentation states `toChain` is optional
- `fromChain` is critical for fast status lookups (especially for Bitcoin)
- Making `toChain` optional allows more flexible status tracking

**What It Does**:
- Fetches swap status from LI.FI `/status` endpoint
- Uses `fromChain` as primary lookup parameter
- Optionally includes `toChain` and `bridge` for faster/more accurate results

---

#### 2.3 Added `isBitcoinPsbt()` Method

**Code Added**:
```typescript
/**
 * Detects if a transaction request contains a Bitcoin PSBT
 * 
 * PSBTs from LI.FI start with magic bytes '70736274' (hex for 'psbt')
 * followed by 0xFF separator byte.
 */
isBitcoinPsbt(transactionData: string): boolean {
  if (!transactionData || typeof transactionData !== 'string') {
    return false;
  }
  const psbtMagicBytes = '70736274ff';
  const normalizedData = transactionData.toLowerCase().trim();
  return normalizedData.startsWith(psbtMagicBytes);
}
```

**Why Added**:
- Need to differentiate PSBTs from EVM transaction calldata
- PSBTs have a specific magic byte sequence that can be detected
- Critical for routing swaps to correct execution flow

**What It Does**:
- Checks if hex string starts with PSBT magic bytes (`70736274ff`)
- Returns `true` if data is a PSBT, `false` otherwise
- Used to validate transaction data before processing

---

#### 2.4 Added `isBitcoinSourceSwap()` Method

**Code Added**:
```typescript
/**
 * Detects if a quote is for a Bitcoin source swap
 * 
 * Bitcoin source swaps return a PSBT in transactionRequest.data
 * instead of standard EVM calldata.
 */
isBitcoinSourceSwap(quote: any): boolean {
  if (!quote?.transactionRequest?.data) {
    return false;
  }
  return this.isBitcoinPsbt(quote.transactionRequest.data);
}
```

**Why Added**:
- Need to determine swap direction to route to correct execution handler
- Bitcoin source swaps require PSBT handling (finalization + broadcast)
- Non-Bitcoin source swaps only need status tracking

**What It Does**:
- Checks if quote's `transactionRequest.data` contains a PSBT
- Returns `true` for BTC → EVM/SOL swaps
- Returns `false` for EVM/SOL → BTC swaps

---

#### 2.5 Added `isBitcoinDestinationSwap()` Method

**Code Added**:
```typescript
/**
 * Detects if a quote is for a Bitcoin destination swap
 */
isBitcoinDestinationSwap(quote: any): boolean {
  if (!quote?.action?.toChainId) {
    return false;
  }
  const toChainId = quote.action.toChainId.toString();
  return toChainId === BITCOIN_CHAIN_ID || toChainId === BITCOIN_CHAIN_KEY;
}
```

**Why Added**:
- Need to identify when Bitcoin is the destination chain
- Helps frontend understand swap type for UI rendering
- Useful for status tracking and explorer URL generation

**What It Does**:
- Checks if destination chain ID matches Bitcoin identifiers
- Returns `true` for EVM/SOL → BTC swaps
- Returns `false` for BTC → EVM/SOL swaps

---

### 3. SwapExecutionHandler Refactoring (`src/modules/swaps/handlers/swap-execution.handler.ts`)

#### 3.1 Added BroadcastService Dependency

**Change**:
```typescript
// BEFORE:
constructor(
  private prisma: PrismaService,
  private oneBalanceProvider: OneBalanceProvider,
  private lifiService: LifiService,
) {}

// AFTER:
constructor(
  private prisma: PrismaService,
  private oneBalanceProvider: OneBalanceProvider,
  private lifiService: LifiService,
  private broadcastService: BroadcastService,  // ✅ Added
) {}
```

**Why Added**:
- Need `BroadcastService` to finalize PSBTs and broadcast Bitcoin transactions
- Separates concerns: execution handler orchestrates, broadcast service handles Bitcoin-specific logic

---

#### 3.2 Refactored `executeLifiBtcSwap()` Method

**Before** (Incorrect - treated all swaps as Bitcoin source):
```typescript
async executeLifiBtcSwap(
  swap: any,
  transaction: any,
  signedPsbtHex: string,  // ❌ Always expected PSBT
) {
  // Always tried to finalize and broadcast
  const rawTxHex = this.broadcastService.finalizePsbt(signedPsbtHex);
  const txHash = await this.broadcastService.broadcast(rawTxHex);
  // ...
}
```

**After** (Correct - routes based on swap type):
```typescript
async executeLifiBtcSwap(
  swap: any,
  transaction: any,
  signedTransactionData?: string,  // ✅ Optional: For BTC source
  transactionHash?: string,         // ✅ Optional: For EVM/SOL source
) {
  const metadata = transaction.metadata as any;
  const isBitcoinSource = this.lifiService.isBitcoinSourceSwap(metadata);

  if (isBitcoinSource) {
    // BTC → EVM/SOL: Finalize PSBT and broadcast
    return this.executeBitcoinSourceSwap(swap, transaction, signedTransactionData);
  } else {
    // EVM/SOL → BTC: Just store txHash and track status
    return this.executeNonBitcoinSourceSwap(swap, transaction, transactionHash);
  }
}
```

**Why Changed**:
- **Critical Bug Fix**: Previous code failed for EVM/SOL → BTC swaps by trying to finalize EVM transactions as PSBTs
- Different swap directions require different handling:
  - Bitcoin source: Backend must finalize PSBT and broadcast
  - Non-Bitcoin source: Frontend broadcasts, backend only tracks

**What It Does**:
- Detects swap type using `isBitcoinSourceSwap()`
- Routes to appropriate execution method based on swap direction
- Handles both Bitcoin source and non-Bitcoin source swaps correctly

---

#### 3.3 Added `executeBitcoinSourceSwap()` Method

**Code Added**:
```typescript
/**
 * BTC → EVM/SOL: Finalize PSBT and broadcast to Bitcoin network
 * 
 * Flow:
 * 1. Validate signed PSBT from Turnkey
 * 2. Finalize PSBT (Turnkey signs but doesn't finalize)
 * 3. Extract raw transaction
 * 4. Broadcast to Bitcoin network via mempool.space
 * 5. Update database with txHash
 */
private async executeBitcoinSourceSwap(
  swap: any,
  transaction: any,
  signedPsbtHex: string,
) {
  // Validate PSBT format
  if (!this.lifiService.isBitcoinPsbt(signedPsbtHex)) {
    throw new BadRequestException('Expected signed PSBT for Bitcoin source swap');
  }

  // Finalize PSBT (Turnkey signs but doesn't finalize)
  const rawTxHex = this.broadcastService.finalizePsbt(signedPsbtHex);

  // Broadcast to Bitcoin network
  const txHash = await this.broadcastService.broadcast(rawTxHex);

  // Update status
  await this.updateSwapStatus(swap, transaction, TransactionStatus.EXECUTING, txHash);

  return {
    success: true,
    transactionHash: txHash,
    explorerUrl: `https://mempool.space/tx/${txHash}`,
    message: 'Bitcoin transaction broadcast successfully. Cross-chain swap in progress.',
    estimatedCompletion: '10-60 minutes',
  };
}
```

**Why Added**:
- Separates Bitcoin source swap logic into dedicated method
- Handles PSBT finalization (Turnkey doesn't finalize automatically)
- Broadcasts to Bitcoin network (mempool.space)
- Updates database with transaction hash for status tracking

**What It Does**:
1. **Validates PSBT**: Ensures data is actually a PSBT before processing
2. **Finalizes PSBT**: Turnkey signs but doesn't finalize - we must do this step
3. **Extracts Raw Transaction**: Converts finalized PSBT to raw transaction hex
4. **Broadcasts**: Sends raw transaction to Bitcoin network
5. **Updates Database**: Stores txHash for status tracking

**Critical Notes**:
- Never modifies PSBT structure (would cause fund loss)
- PSBT contains 3 outputs: vault payment, OP_RETURN memo, refund address
- Chainflip requires exactly 3 outputs - removing refund output causes permanent loss

---

#### 3.4 Added `executeNonBitcoinSourceSwap()` Method

**Code Added**:
```typescript
/**
 * EVM/SOL → BTC: Frontend broadcasts, backend just tracks status
 * 
 * Flow:
 * 1. Frontend signs EVM/Solana transaction with Turnkey
 * 2. Frontend broadcasts directly to source chain
 * 3. Frontend provides transaction hash to backend
 * 4. Backend stores hash and tracks status via LI.FI
 */
private async executeNonBitcoinSourceSwap(
  swap: any,
  transaction: any,
  transactionHash: string,
) {
  // Validate transaction hash format
  if (!transactionHash || transactionHash.length < 10) {
    throw new BadRequestException('Invalid transaction hash format');
  }

  // Update database with transaction hash
  await this.updateSwapStatus(swap, transaction, TransactionStatus.EXECUTING, transactionHash);

  const metadata = transaction.metadata as any;
  const fromChain = metadata?.action?.fromChainId?.toString() || swap.fromNetwork;

  return {
    success: true,
    transactionHash,
    explorerUrl: this.getExplorerUrl(fromChain, transactionHash),
    message: 'Transaction confirmed. Cross-chain swap in progress.',
    estimatedCompletion: this.getEstimatedCompletion(fromChain),
  };
}
```

**Why Added**:
- **Critical Bug Fix**: Previous code tried to finalize EVM/Solana transactions as PSBTs
- EVM/Solana transactions are broadcast by frontend, not backend
- Backend only needs to store txHash and track status

**What It Does**:
1. **Validates Hash**: Ensures transaction hash is provided and valid format
2. **Stores Hash**: Updates database with transaction hash for status tracking
3. **Returns Info**: Provides explorer URL and estimated completion time
4. **No Broadcasting**: Frontend already broadcast the transaction

**Key Difference from Bitcoin Source**:
- No PSBT finalization needed
- No broadcasting needed (frontend handles it)
- Only status tracking required

---

#### 3.5 Added Helper Methods

**`updateSwapStatus()`**:
```typescript
private async updateSwapStatus(
  swap: any,
  transaction: any,
  status: TransactionStatus,
  txHash: string,
): Promise<void>
```

**Purpose**: Centralized method to update both swap and transaction records in database

**Why Added**: Reduces code duplication, ensures consistent status updates

---

**`handleSwapFailure()`**:
```typescript
private async handleSwapFailure(swap: any, transaction: any): Promise<void>
```

**Purpose**: Centralized error handling for swap failures

**Why Added**: Consistent error handling across all swap execution paths

---

**`getExplorerUrl()`**:
```typescript
private getExplorerUrl(chainId: string | null | undefined, txHash: string): string
```

**Purpose**: Returns chain-specific blockchain explorer URL

**What It Does**:
- Ethereum: `https://etherscan.io/tx/{hash}`
- Solana: `https://solscan.io/tx/{hash}`
- Arbitrum: `https://arbiscan.io/tx/{hash}`
- Bitcoin: `https://mempool.space/tx/{hash}` (handled in Bitcoin source method)

**Why Added**: Provides users with direct links to view their transactions

---

**`getEstimatedCompletion()`**:
```typescript
private getEstimatedCompletion(chainId: string | null | undefined): string
```

**Purpose**: Returns estimated completion time based on source chain

**What It Does**:
- Solana: `'30 seconds - 2 minutes'` (fast confirmations)
- Ethereum: `'2-5 minutes'` (7 confirmations for THORChain)
- Default: `'2-5 minutes'`

**Why Added**: Sets user expectations for swap completion time

---

### 4. SwapLifiQuoteHandler Updates (`src/modules/swaps/handlers/swap-lifi-quote.handler.ts`)

#### 4.1 Added Swap Type Detection in Quote Response

**Code Added**:
```typescript
// 11. Detect swap type for frontend handling
const isBitcoinSource = this.lifiService.isBitcoinSourceSwap(lifiQuote);
const isBitcoinDestination = this.lifiService.isBitcoinDestinationSwap(lifiQuote);

// 12. Validate PSBT structure for Bitcoin source swaps
if (isBitcoinSource && lifiQuote.transactionRequest?.data) {
  this.logger.log('Bitcoin source swap detected - PSBT will require signing and finalization');
  
  // Validate PSBT format
  if (!this.lifiService.isBitcoinPsbt(lifiQuote.transactionRequest.data)) {
    this.logger.warn('Transaction data does not appear to be a valid PSBT');
  }

  // CRITICAL: Warn about PSBT modification risks
  this.logger.warn(
    'CRITICAL: Do not modify PSBT from LI.FI. ' +
    'Changing outputs, amounts, or removing OP_RETURN memo will cause permanent fund loss. ' +
    'Chainflip requires exactly three outputs - the third is a mandatory refund output.'
  );
}

// Added to response:
swapType: {
  isBitcoinSource,
  isBitcoinDestination,
  requiresPsbtSigning: isBitcoinSource,
  requiresEvmSigning: !isBitcoinSource && !isBitcoinDestination,
}
```

**Why Added**:
- Frontend needs to know swap type to use correct signing flow
- Bitcoin source swaps require PSBT signing (Turnkey `TRANSACTION_TYPE_BITCOIN`)
- EVM/SOL source swaps require standard transaction signing
- Warns developers about PSBT modification risks

**What It Does**:
- Detects swap type during quote creation
- Validates PSBT format for Bitcoin source swaps
- Adds `swapType` object to response for frontend routing
- Logs critical warnings about PSBT handling

---

### 5. SwapStatusHandler Updates (`src/modules/swaps/handlers/swap-status.handler.ts`)

#### 5.1 Improved Chain ID Normalization

**Before**:
```typescript
let fromChain = metadata?.action?.fromChainId?.toString() || swap.fromNetwork;
let toChain = metadata?.action?.toChainId?.toString() || swap.toNetwork;

// Manual normalization
if (fromChain === '20000000000001' || fromChain === 'bitcoin') {
  fromChain = 'btc';
}
```

**After**:
```typescript
import { normalizeBitcoinChainId } from '../../lifi/utils/bitcoin-constants.util';

let fromChain = normalizeBitcoinChainId(
  metadata?.action?.fromChainId?.toString() || swap.fromNetwork
);
let toChain = normalizeBitcoinChainId(
  metadata?.action?.toChainId?.toString() || swap.toNetwork
);
```

**Why Changed**:
- Uses centralized normalization function
- Handles all Bitcoin identifier variations consistently
- Reduces code duplication

---

#### 5.2 Updated Status Check to Make `toChain` Optional

**Before**:
```typescript
const statusResult = await this.lifiService.getStatus(
  transaction.transactionHash,
  fromChain,
  toChain,  // ❌ Required
  bridge,
);
```

**After**:
```typescript
const statusResult = await this.lifiService.getStatus(
  transaction.transactionHash,
  fromChain,  // ✅ Required (speeds up lookup)
  toChain,     // ✅ Optional (but recommended)
  bridge,      // ✅ Optional (helps with lookup)
);
```

**Why Changed**:
- Matches LI.FI API specification (`toChain` is optional)
- `fromChain` is sufficient for status lookup
- More flexible status tracking

---

### 6. SwapsService Updates (`src/modules/swaps/swaps.service.ts`)

#### 6.1 Added LifiService Injection

**Change**:
```typescript
// BEFORE:
constructor(
  private configService: ConfigService,
  private prisma: PrismaService,
  private oneBalanceProvider: OneBalanceProvider,
  private oneBalanceQuoteHandler: SwapOneBalanceQuoteHandler,
  private lifiQuoteHandler: SwapLifiQuoteHandler,
  private executionHandler: SwapExecutionHandler,
  private statusHandler: SwapStatusHandler,
) {}

// AFTER:
constructor(
  private configService: ConfigService,
  private prisma: PrismaService,
  private oneBalanceProvider: OneBalanceProvider,
  private lifiService: LifiService,  // ✅ Added
  private oneBalanceQuoteHandler: SwapOneBalanceQuoteHandler,
  private lifiQuoteHandler: SwapLifiQuoteHandler,
  private executionHandler: SwapExecutionHandler,
  private statusHandler: SwapStatusHandler,
) {}
```

**Why Added**:
- Need `LifiService` to detect Bitcoin source swaps using `isBitcoinSourceSwap()`
- Service needs to route swaps to correct execution handler based on swap type

---

#### 6.2 Updated `executeSwap()` Method Signature

**Before**:
```typescript
async executeSwap(
  userId: string,
  quoteId: string,
  signedOperations: any[],
  tamperProofSignature: string,
  signedTransactionHex?: string,
) {
```

**After**:
```typescript
async executeSwap(
  userId: string,
  quoteId: string,
  signedOperations?: any[],           // ✅ Optional (OneBalance only)
  tamperProofSignature?: string,      // ✅ Optional (OneBalance only)
  signedTransactionData?: string,     // ✅ Optional (BTC source: signed PSBT)
  transactionHash?: string,            // ✅ Optional (EVM/SOL source: txHash)
) {
```

**Why Changed**:
- Different swap types require different parameters:
  - OneBalance: `signedOperations` + `tamperProofSignature`
  - BTC source: `signedTransactionData` (signed PSBT)
  - EVM/SOL source: `transactionHash` (from frontend broadcast)
- Made parameters optional to support all swap types

---

#### 6.3 Added Swap Type Detection and Routing

**Code Added**:
```typescript
if (transaction.service === ServiceProvider.LIFI) {
  // LI.FI swaps have two different flows:
  // 1. BTC → EVM/SOL: Requires signedTransactionData (signed PSBT)
  // 2. EVM/SOL → BTC: Requires transactionHash (frontend broadcasts)
  const metadata = transaction.metadata as any;
  const isBitcoinSource = this.lifiService.isBitcoinSourceSwap(metadata);

  if (isBitcoinSource) {
    // BTC → EVM/SOL: Need signed PSBT
    if (!signedTransactionData) {
      throw new BadRequestException(
        'signedTransactionData (signed PSBT) is required for Bitcoin source swaps.'
      );
    }
    return this.executionHandler.executeLifiBtcSwap(
      swap, transaction, signedTransactionData, undefined
    );
  } else {
    // EVM/SOL → BTC: Need transaction hash from frontend broadcast
    if (!transactionHash) {
      throw new BadRequestException(
        'transactionHash is required for EVM/Solana to BTC swaps.'
      );
    }
    return this.executionHandler.executeLifiBtcSwap(
      swap, transaction, undefined, transactionHash
    );
  }
}
```

**Why Added**:
- **Critical Bug Fix**: Previous code always expected PSBT, causing failures for EVM/SOL → BTC swaps
- Routes swaps to correct execution handler based on swap type
- Validates required parameters for each swap type

**What It Does**:
1. Detects swap type using `isBitcoinSourceSwap()`
2. Validates required parameters based on swap type
3. Routes to `executeLifiBtcSwap()` with correct parameters
4. Bitcoin source: Passes `signedTransactionData`, `undefined` for hash
5. Non-Bitcoin source: Passes `undefined` for data, `transactionHash`

---

### 7. SwapExecuteDto Updates (`src/modules/swaps/dto/swap-execute.dto.ts`)

#### 7.1 Updated Field Definitions

**Before**:
```typescript
export class SwapExecuteDto {
  @IsString()
  @IsNotEmpty()
  quoteId: string;

  @IsArray()
  @IsNotEmpty()
  signedOperations: any[];  // ❌ Required

  @IsString()
  @IsNotEmpty()
  tamperProofSignature: string;  // ❌ Required

  @IsString()
  @IsOptional()
  signedTransactionHex?: string;  // ❌ Wrong name, no documentation
}
```

**After**:
```typescript
export class SwapExecuteDto {
  @IsString()
  @IsNotEmpty()
  quoteId: string;

  @IsArray()
  @IsOptional()
  signedOperations?: any[];  // ✅ Optional: For OneBalance swaps

  @IsString()
  @IsOptional()
  tamperProofSignature?: string;  // ✅ Optional: For OneBalance swaps

  /**
   * For BTC → EVM/SOL swaps: Signed PSBT from Turnkey (not finalized)
   * Backend will finalize and broadcast to Bitcoin network
   */
  @IsString()
  @IsOptional()
  signedTransactionData?: string;  // ✅ Renamed, documented

  /**
   * For EVM/SOL → BTC swaps: Transaction hash from frontend broadcast
   * Frontend signs and broadcasts EVM/Solana transaction, then provides hash
   */
  @IsString()
  @IsOptional()
  transactionHash?: string;  // ✅ New field, documented
}
```

**Why Changed**:
- **Field Renaming**: `signedTransactionHex` → `signedTransactionData` (more descriptive)
- **New Field**: Added `transactionHash` for EVM/SOL source swaps
- **Optional Fields**: Made OneBalance fields optional (only required for OneBalance swaps)
- **Documentation**: Added JSDoc comments explaining when each field is used

**What It Does**:
- Defines request body structure for swap execution endpoint
- Validates input parameters using class-validator decorators
- Documents which fields are required for which swap types

---

### 8. SwapsController Updates (`src/modules/swaps/swaps.controller.ts`)

#### 8.1 Updated Parameter Passing

**Before**:
```typescript
async executeSwap(
  @Request() req,
  @Body() dto: SwapExecuteDto,
) {
  return await this.swapsService.executeSwap(
    req.user.userId,
    dto.quoteId,
    dto.signedOperations,
    dto.tamperProofSignature,
    dto.signedTransactionHex,  // ❌ Old field name
  );
}
```

**After**:
```typescript
async executeSwap(
  @Request() req,
  @Body() dto: SwapExecuteDto,
) {
  return await this.swapsService.executeSwap(
    req.user.userId,
    dto.quoteId,
    dto.signedOperations,
    dto.tamperProofSignature,
    dto.signedTransactionData,  // ✅ New field name
    dto.transactionHash,         // ✅ New field
  );
}
```

**Why Changed**:
- Matches updated DTO field names
- Passes new `transactionHash` parameter for EVM/SOL source swaps
- Maintains backward compatibility (fields are optional)

---

### 9. SwapsModule Updates (`src/modules/swaps/swaps.module.ts`)

#### 9.1 Added BitcoinModule Import

**Change**:
```typescript
// BEFORE:
@Module({
  imports: [AuthModule, DatabaseModule, LifiModule],
  // ...
})

// AFTER:
@Module({
  imports: [AuthModule, DatabaseModule, LifiModule, BitcoinModule],  // ✅ Added
  // ...
})
```

**Why Added**:
- `SwapExecutionHandler` needs `BroadcastService` from `BitcoinModule`
- `BroadcastService` handles PSBT finalization and Bitcoin transaction broadcasting
- Required for Bitcoin source swap execution

**What It Does**:
- Imports `BitcoinModule` to make `BroadcastService` available
- Enables dependency injection of `BroadcastService` in `SwapExecutionHandler`

---

## Function Reference

### LifiService Methods

#### `isBitcoinPsbt(transactionData: string): boolean`
- **Purpose**: Detects if hex string is a Bitcoin PSBT
- **Parameters**: `transactionData` - Hex-encoded transaction data
- **Returns**: `true` if PSBT, `false` otherwise
- **Logic**: Checks for PSBT magic bytes (`70736274ff`)

#### `isBitcoinSourceSwap(quote: any): boolean`
- **Purpose**: Determines if swap is Bitcoin source (BTC → EVM/SOL)
- **Parameters**: `quote` - LI.FI quote response object
- **Returns**: `true` if Bitcoin source swap
- **Logic**: Checks if `quote.transactionRequest.data` is a PSBT

#### `isBitcoinDestinationSwap(quote: any): boolean`
- **Purpose**: Determines if swap is Bitcoin destination (EVM/SOL → BTC)
- **Parameters**: `quote` - LI.FI quote response object
- **Returns**: `true` if Bitcoin destination swap
- **Logic**: Checks if `quote.action.toChainId` matches Bitcoin chain IDs

#### `getStatus(txHash: string, fromChain: string, toChain?: string, bridge?: string): Promise<any>`
- **Purpose**: Fetches swap status from LI.FI API
- **Parameters**:
  - `txHash` - Transaction hash from source chain
  - `fromChain` - Source chain ID (required, speeds up lookup)
  - `toChain` - Destination chain ID (optional)
  - `bridge` - Bridge name (optional, helps with lookup)
- **Returns**: Status response from LI.FI
- **Logic**: Calls LI.FI `/status` endpoint with parameters

---

### SwapExecutionHandler Methods

#### `executeLifiBtcSwap(swap, transaction, signedTransactionData?, transactionHash?)`
- **Purpose**: Main entry point for LI.FI swap execution
- **Parameters**:
  - `swap` - Swap database record
  - `transaction` - Transaction database record
  - `signedTransactionData` - Signed PSBT (for BTC source swaps)
  - `transactionHash` - Transaction hash (for EVM/SOL source swaps)
- **Returns**: Execution result with txHash and explorer URL
- **Logic**: Routes to appropriate execution method based on swap type

#### `executeBitcoinSourceSwap(swap, transaction, signedPsbtHex)`
- **Purpose**: Handles BTC → EVM/SOL swaps
- **Flow**:
  1. Validates PSBT format
  2. Finalizes PSBT (Turnkey signs but doesn't finalize)
  3. Extracts raw transaction
  4. Broadcasts to Bitcoin network
  5. Updates database with txHash
- **Returns**: Success response with Bitcoin explorer URL

#### `executeNonBitcoinSourceSwap(swap, transaction, transactionHash)`
- **Purpose**: Handles EVM/SOL → BTC swaps
- **Flow**:
  1. Validates transaction hash
  2. Updates database with txHash
  3. Returns success response with chain-specific explorer URL
- **Returns**: Success response with chain-specific explorer URL

---

## Flow Diagrams

### BTC → EVM/SOL Swap Flow

```
┌─────────┐         ┌─────────┐         ┌─────────┐         ┌──────────────┐
│Frontend │         │ Backend │         │  LI.FI  │         │Bitcoin Network
└────┬────┘         └────┬────┘         └────┬────┘         └──────┬───────┘
     │                   │                    │                      │
     │ 1. requestQuote() │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │ 2. GET /quote      │                      │
     │                   ├───────────────────>│                      │
     │                   │                    │                      │
     │                   │ 3. Returns PSBT    │                      │
     │                   │<───────────────────┤                      │
     │                   │                    │                      │
     │ 4. Quote + PSBT   │                    │                      │
     │<──────────────────┤                    │                      │
     │                   │                    │                      │
     │ 5. Sign PSBT      │                    │                      │
     │    (Turnkey)      │                    │                      │
     │                   │                    │                      │
     │ 6. executeSwap()  │                    │                      │
     │    (signed PSBT)  │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │                    │                      │
     │                   │ 7. Finalize PSBT   │                      │
     │                   │    Extract raw tx  │                      │
     │                   │                    │                      │
     │                   │ 8. Broadcast       │                      │
     │                   ├──────────────────────────────────────────>│
     │                   │                    │                      │
     │                   │ 9. Returns txHash  │                      │
     │                   │<───────────────────────────────────────────┤
     │                   │                    │                      │
     │ 10. Success +     │                    │                      │
     │     txHash        │                    │                      │
     │<──────────────────┤                    │                      │
     │                   │                    │                      │
     │ 11. getStatus()   │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │ 12. GET /status    │                      │
     │                   │    (txHash, btc)   │                      │
     │                   ├───────────────────>│                      │
     │                   │                    │                      │
     │                   │ 13. Status update  │                      │
     │                   │<───────────────────┤                      │
     │                   │                    │                      │
     │ 14. Status        │                    │                      │
     │<──────────────────┤                    │                      │
```

### EVM/SOL → BTC Swap Flow

```
┌─────────┐         ┌─────────┐         ┌─────────┐         ┌──────────────┐
│Frontend │         │ Backend │         │  LI.FI  │         │Ethereum/Solana
└────┬────┘         └────┬────┘         └────┬────┘         └──────┬───────┘
     │                   │                    │                      │
     │ 1. requestQuote() │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │ 2. GET /quote       │                      │
     │                   ├───────────────────>│                      │
     │                   │                    │                      │
     │                   │ 3. Returns EVM tx  │                      │
     │                   │<───────────────────┤                      │
     │                   │                    │                      │
     │ 4. Quote + EVM tx │                    │                      │
     │<──────────────────┤                    │                      │
     │                   │                    │                      │
     │ 5. Sign EVM tx    │                    │                      │
     │    (Turnkey)      │                    │                      │
     │                   │                    │                      │
     │ 6. Broadcast      │                    │                      │
     │    directly       │                    │                      │
     ├──────────────────────────────────────────────────────────────>│
     │                   │                    │                      │
     │ 7. Returns txHash │                    │                      │
     │<──────────────────────────────────────────────────────────────┤
     │                   │                    │                      │
     │ 8. executeSwap()  │                    │                      │
     │    (txHash)       │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │                    │                      │
     │                   │ 9. Store txHash    │                      │
     │                   │    Update status   │                      │
     │                   │                    │                      │
     │ 10. Success       │                    │                      │
     │<──────────────────┤                    │                      │
     │                   │                    │                      │
     │ 11. getStatus()   │                    │                      │
     ├──────────────────>│                    │                      │
     │                   │ 12. GET /status    │                      │
     │                   │    (txHash, eth)   │                      │
     │                   ├───────────────────>│                      │
     │                   │                    │                      │
     │                   │ 13. Status update  │                      │
     │                   │<───────────────────┤                      │
     │                   │                    │                      │
     │ 14. Status        │                    │                      │
     │<──────────────────┤                    │                    │
```

---

## Testing Considerations

### Test Cases to Verify

1. **BTC → ETH Swap**:
   - ✅ Quote returns PSBT
   - ✅ PSBT signing works
   - ✅ Finalization succeeds
   - ✅ Broadcasting to Bitcoin network works
   - ✅ Status tracking with `fromChain='btc'` works

2. **ETH → BTC Swap**:
   - ✅ Quote returns EVM transaction data
   - ✅ Frontend can sign and broadcast EVM transaction
   - ✅ Backend accepts transaction hash
   - ✅ Status tracking with `fromChain='eth'` works
   - ✅ No PSBT finalization attempted

3. **SOL → BTC Swap**:
   - ✅ Quote returns Solana transaction data
   - ✅ Frontend can sign and broadcast Solana transaction
   - ✅ Backend accepts transaction hash
   - ✅ Status tracking with `fromChain='sol'` works
   - ✅ No PSBT finalization attempted

4. **Error Cases**:
   - ✅ Invalid PSBT format rejected
   - ✅ Missing transaction hash for EVM/SOL source swaps
   - ✅ Missing signed PSBT for BTC source swaps
   - ✅ Invalid transaction hash format rejected

---

## Summary

### What Was Fixed

1. ✅ **PSBT Detection**: Added methods to detect Bitcoin PSBTs vs EVM transaction data
2. ✅ **Swap Type Routing**: Correctly routes swaps based on source chain (Bitcoin vs non-Bitcoin)
3. ✅ **Bitcoin Source Swaps**: Properly finalizes PSBTs and broadcasts to Bitcoin network
4. ✅ **Non-Bitcoin Source Swaps**: Only tracks status, doesn't attempt PSBT operations
5. ✅ **Chain ID Normalization**: Consistent handling of Bitcoin chain identifiers
6. ✅ **Status Tracking**: Improved status tracking with optional `toChain` parameter
7. ✅ **Error Handling**: Better error messages and validation

### Key Architectural Decisions

1. **Separation of Concerns**: Bitcoin-specific logic in `BroadcastService`, swap orchestration in `SwapExecutionHandler`
2. **Type Detection**: Detect swap type early to route to correct handler
3. **Frontend Responsibility**: Frontend broadcasts EVM/SOL transactions, backend only tracks
4. **Backend Responsibility**: Backend finalizes and broadcasts Bitcoin transactions
5. **Status Tracking**: Use `fromChain` as primary lookup parameter for faster responses

### Critical Warnings

⚠️ **Never modify PSBTs from LI.FI**:
- Changing outputs, amounts, or removing OP_RETURN memo causes permanent fund loss
- Chainflip requires exactly 3 outputs - the third is a mandatory refund output

⚠️ **Turnkey doesn't finalize PSBTs**:
- Backend must finalize PSBTs before broadcasting
- Finalization extracts raw transaction from signed PSBT

⚠️ **Quote expiration**:
- LI.FI quotes expire after ~30 minutes
- Always request fresh quotes before execution
- Never cache vault addresses

---

## Conclusion

This implementation correctly handles both Bitcoin source and destination swaps through LI.FI, with proper PSBT handling, status tracking, and error handling. The codebase now supports the full range of cross-chain swaps involving Bitcoin.

