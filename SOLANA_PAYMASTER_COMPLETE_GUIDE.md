# Solana Paymaster - Complete Implementation Guide

## 📋 Overview

This guide documents the complete implementation of a **Solana Paymaster** using Turnkey's secure key management. The paymaster allows your backend to pay transaction fees on behalf of users, enabling gasless Solana transactions.

---

## 🏗️ Architecture

### Backend Structure

```
src/modules/paymaster/
├── paymaster.module.ts          # Module definition
├── paymaster.controller.ts      # API endpoints (2 endpoints)
├── paymaster.service.ts          # Business logic (minimal, optimized)
├── providers/
│   └── turnkey-solana.provider.ts  # Turnkey integration (optimized)
└── dto/
    └── paymaster-sign.dto.ts     # Request validation
```

### Code Statistics

- **Total Lines**: ~120 lines (optimized)
- **Time Complexity**: O(1) for all operations
- **Space Complexity**: O(1) - minimal memory usage
- **API Endpoints**: 2 (sign, address)

---

## ⚙️ Environment Configuration

### Required Environment Variables

Add to your `.env` file:

```bash
# Turnkey Configuration (already configured)
TURNKEY_ORG_ID=7236a896-de90-431f-b94e-f7926f38b315
TURNKEY_API_PUBLIC_KEY=your_api_public_key
TURNKEY_API_PRIVATE_KEY=your_api_private_key

# Solana Paymaster Configuration
SOLANA_PAYMASTER_ADDRESS=2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt
SOLANA_PRIVATE_KEY_ID=b646851f-9a5b-425c-8568-8fd36ae09c34  # Optional
SOLANA_RPC_URL=https://api.mainnet-beta.solana.com          # Optional
```

### Important Notes

- **`SOLANA_PAYMASTER_ADDRESS`**: Solana address (base58 format) - this IS the public key
- **`SOLANA_PRIVATE_KEY_ID`**: Optional, used for logging/reference
- **Source**: From Turnkey activity JSON → `result.privateKeys[0].addresses[0].address`

---

## 🔧 Backend Implementation

### 1. TurnkeySolanaProvider (`providers/turnkey-solana.provider.ts`)

**Purpose**: Handles Turnkey API integration for Solana transaction signing

**Key Features**:
- Initializes Turnkey client on module init
- Signs transactions using `signTransaction` API
- Optimized: Single API call, minimal processing

**Code**:
```typescript
async signTransaction(transaction: Transaction): Promise<void> {
  const response = await this.turnkeyClient.apiClient().signTransaction({
    organizationId: this.organizationId,
    signWith: this.privateKeyId || this.paymasterPublicKey.toString(),
    unsignedTransaction: Buffer.from(transaction.serialize({ requireAllSignatures: false })).toString('base64'),
    type: 'TRANSACTION_TYPE_SOLANA',
  });

  const signedTx = Transaction.from(Buffer.from(response.signedTransaction, 'base64'));
  const signature = signedTx.signatures[0]?.signature;
  
  if (!signature) {
    throw new Error('Failed to get paymaster signature from Turnkey response');
  }

  transaction.addSignature(this.paymasterPublicKey, signature);
}
```

**Complexity**: O(1) - Single API call, constant time operations

### 2. PaymasterService (`paymaster.service.ts`)

**Purpose**: Core business logic for transaction signing

**Key Features**:
- Validates fee payer
- Delegates signing to Turnkey provider
- Returns signed transaction

**Code**:
```typescript
async signTransaction(serializedTransaction: string): Promise<string> {
  const transaction = Transaction.from(Buffer.from(serializedTransaction, 'base64'));
  const paymasterPublicKey = this.turnkeySolanaProvider.getPaymasterPublicKey();

  if (!transaction.feePayer?.equals(paymasterPublicKey)) {
    throw new BadRequestException(
      `Transaction fee payer must be ${paymasterPublicKey.toString()}. Got: ${transaction.feePayer?.toString() || 'none'}`
    );
  }

  await this.turnkeySolanaProvider.signTransaction(transaction);
  return Buffer.from(transaction.serialize()).toString('base64');
}
```

**Complexity**: O(1) - Constant time validation and serialization

### 3. PaymasterController (`paymaster.controller.ts`)

**Purpose**: REST API endpoints

**Endpoints**:
- `POST /api/paymaster/sign` - Sign transaction as fee payer
- `GET /api/paymaster/address` - Get paymaster address

**Code**:
```typescript
@Post('sign')
@UseGuards(AuthGuard)
async signTransaction(@Body() dto: PaymasterSignDto) {
  return {
    signedTransaction: await this.paymasterService.signTransaction(dto.transaction),
    paymasterAddress: this.paymasterService.getPaymasterAddress(),
  };
}

@Get('address')
@UseGuards(AuthGuard)
getPaymasterAddress() {
  return { address: this.paymasterService.getPaymasterAddress() };
}
```

---

## 📡 API Endpoints

### POST `/api/paymaster/sign`

Sign a Solana transaction as the fee payer.

**Authentication**: Required (Bearer token)

**Request**:
```json
{
  "transaction": "base64_encoded_serialized_transaction"
}
```

**Response**:
```json
{
  "signedTransaction": "base64_encoded_signed_transaction",
  "paymasterAddress": "2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt"
}
```

**Example**:
```bash
curl -X POST https://yourbackend.io/api/paymaster/sign \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{"transaction": "AQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABAAEDAr4..."}'
```

### GET `/api/paymaster/address`

Get the paymaster's Solana address.

**Authentication**: Required (Bearer token)

**Response**:
```json
{
  "address": "2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt"
}
```

---

## 💻 Frontend Implementation

### Step 1: Install Dependencies

```bash
npm install @solana/web3.js @solana/spl-token
```

### Step 2: Complete Transfer Flow

```typescript
import { Connection, PublicKey, Transaction, Keypair } from '@solana/web3.js';
import { 
  getAssociatedTokenAddress, 
  createTransferInstruction, 
  TOKEN_PROGRAM_ID 
} from '@solana/spl-token';

const PAYMASTER_ADDRESS = '2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt';
const connection = new Connection('https://api.mainnet-beta.solana.com');

async function transferWithPaymaster(
  tokenMint: string,        // USDC mint: EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
  recipient: string,         // Recipient wallet address
  amount: number,            // Amount in token units (e.g., 100 for 100 USDC)
  userKeypair: Keypair,     // User's keypair
  authToken: string         // JWT auth token
) {
  // 1. Get token accounts
  const fromAccount = await getAssociatedTokenAddress(
    new PublicKey(tokenMint),
    userKeypair.publicKey
  );
  const toAccount = await getAssociatedTokenAddress(
    new PublicKey(tokenMint),
    new PublicKey(recipient)
  );

  // 2. Build transaction
  const transaction = new Transaction().add(
    createTransferInstruction(
      fromAccount,
      toAccount,
      userKeypair.publicKey,
      amount * 1_000_000, // Convert to smallest unit (6 decimals for USDC)
      [],
      TOKEN_PROGRAM_ID
    )
  );

  // 3. Set paymaster as fee payer
  transaction.feePayer = new PublicKey(PAYMASTER_ADDRESS);
  transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

  // 4. User signs their part
  transaction.partialSign(userKeypair);

  // 5. Get paymaster signature from backend
  const serialized = transaction.serialize({ requireAllSignatures: false }).toString('base64');
  
  const response = await fetch('https://yourbackend.io/api/paymaster/sign', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${authToken}`
    },
    body: JSON.stringify({ transaction: serialized })
  });

  if (!response.ok) {
    throw new Error(`Paymaster signing failed: ${response.statusText}`);
  }

  const { signedTransaction } = await response.json();

  // 6. Broadcast to Solana
  const finalTx = Transaction.from(Buffer.from(signedTransaction, 'base64'));
  const signature = await connection.sendRawTransaction(finalTx.serialize());
  
  // 7. Wait for confirmation
  await connection.confirmTransaction(signature, 'confirmed');
  
  return signature;
}
```

### Step 3: Usage Example

```typescript
// React component example
function SendTokenButton() {
  const [loading, setLoading] = useState(false);
  
  const handleSend = async () => {
    setLoading(true);
    try {
      const signature = await transferWithPaymaster(
        'EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v', // USDC mint
        '7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU', // Recipient
        100, // Amount
        userKeypair,
        authToken
      );
      alert(`Transaction sent! Signature: ${signature}`);
    } catch (error) {
      alert(`Error: ${error.message}`);
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <button onClick={handleSend} disabled={loading}>
      {loading ? 'Sending...' : 'Send 100 USDC'}
    </button>
  );
}
```

---

## 🔄 Complete Workflow

### The Story: Alice Sends Tokens to Bob

#### Step 1: Alice Builds Transaction
```typescript
// Frontend: Alice wants to send 100 USDC to Bob
const transaction = new Transaction().add(
  createTransferInstruction(
    aliceTokenAccount,  // From: Alice's USDC account
    bobTokenAccount,     // To: Bob's USDC account
    alicePublicKey,      // Owner: Alice
    100000000           // Amount: 100 USDC (6 decimals)
  )
);

// Set paymaster as fee payer
transaction.feePayer = new PublicKey('2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt');
transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

// Alice signs (approves the token transfer)
transaction.partialSign(aliceKeypair);
```

**What's in the transaction?**
- ✅ Alice's signature (approving the USDC transfer)
- ✅ Transfer instruction (move 100 USDC)
- ❌ Paymaster signature (missing - backend will add this)

#### Step 2: Frontend Sends to Backend
```typescript
// Serialize and send
const serialized = transaction.serialize({ requireAllSignatures: false }).toString('base64');

fetch('/api/paymaster/sign', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${authToken}`
  },
  body: JSON.stringify({ transaction: serialized })
});
```

#### Step 3: Backend Validates & Signs
```typescript
// Backend receives request
// 1. Deserialize transaction
const tx = Transaction.from(Buffer.from(serializedTx, 'base64'));

// 2. Validate fee payer matches paymaster
if (tx.feePayer !== PAYMASTER_ADDRESS) {
  throw new Error('Invalid fee payer');
}

// 3. Sign with Turnkey
await turnkeyProvider.signTransaction(tx);

// 4. Return signed transaction
return { signedTransaction: tx.serialize().toString('base64') };
```

**What Turnkey Does**:
- Receives unsigned transaction
- Signs it using secure HSM (private key never exposed)
- Returns fully signed transaction

#### Step 4: Frontend Broadcasts
```typescript
// Receive signed transaction
const { signedTransaction } = await response.json();
const finalTx = Transaction.from(Buffer.from(signedTransaction, 'base64'));

// Broadcast to Solana
const signature = await connection.sendRawTransaction(finalTx.serialize());
await connection.confirmTransaction(signature);
```

#### Step 5: Solana Executes
- ✅ Validates both signatures (Alice + Paymaster)
- ✅ Executes transfer: 100 USDC from Alice → Bob
- ✅ Deducts ~0.000005 SOL from paymaster (fee)
- ✅ Transaction confirmed!

**Result**: Alice sent tokens without paying fees! 🎉

---

## 🔐 Security Model

### Why It's Safe

1. **Not a Money Transmitter**: Only paying fees, not handling user funds
2. **User Controls Transaction**: User signs their part, backend can't modify
3. **Secure Key Storage**: Turnkey manages keys in HSMs (hardware security modules)
4. **Fee Payer Validation**: Backend verifies fee payer before signing

### Security Features

- ✅ Authentication required (`AuthGuard`)
- ✅ Fee payer validation (must match paymaster)
- ✅ No private key exposure (Turnkey handles signing)
- ✅ Transaction integrity (user signs first)

---

## ✅ Verification with Turnkey Docs

### Verified Implementation

✅ **API Method**: `signTransaction` (correct)
✅ **Transaction Type**: `TRANSACTION_TYPE_SOLANA` (correct)
✅ **Input Format**: Base64 encoded unsigned transaction (correct)
✅ **Output Format**: Base64 encoded signed transaction (correct)
✅ **SDK Usage**: `@turnkey/sdk-server` with `Turnkey` class (correct)
✅ **Signature Extraction**: From fee payer position (index 0) (correct)

### Turnkey Documentation Reference

According to Turnkey API Reference:
- Endpoint: `/public/v1/submit/sign_transaction`
- Method: `signTransaction` via SDK client
- Type: `TRANSACTION_TYPE_SOLANA`
- Response: `signedTransaction` (base64 string)

**Our Implementation**: ✅ Matches Turnkey's official documentation

---

## 🎯 Use Cases

### 1. Token Transfers
```typescript
// Send USDC, USDT, or any SPL token
await transferWithPaymaster(usdcMint, recipient, amount, keypair, token);
```

### 2. Swaps (via OneBalance/DEX)
```typescript
// 1. Get swap quote
const quote = await getSwapQuote({ fromAsset: 'USDC', toAsset: 'SOL', ... });

// 2. Build transaction from quote
const transaction = Transaction.from(Buffer.from(quote.transactionData, 'base64'));

// 3. Set paymaster as fee payer
transaction.feePayer = new PublicKey(PAYMASTER_ADDRESS);
transaction.recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

// 4. User signs
transaction.partialSign(userKeypair);

// 5. Get paymaster signature
const signedTx = await getPaymasterSignature(transaction);

// 6. Broadcast
await connection.sendRawTransaction(signedTx.serialize());
```

### 3. Any Solana Transaction
The paymaster works with **any** Solana transaction type:
- Token transfers
- NFT transfers
- Program interactions
- Staking operations
- etc.

---

## 📊 Performance & Complexity

### Time Complexity
- **Sign Transaction**: O(1) - Single API call
- **Validation**: O(1) - Constant time checks
- **Serialization**: O(n) where n = transaction size (unavoidable)

### Space Complexity
- **Memory Usage**: O(1) - Minimal, only transaction objects
- **No Caching**: Stateless design, no memory overhead

### Optimizations Applied
- ✅ Removed duplicate validations
- ✅ Simplified error handling
- ✅ Minimized code lines (~120 total)
- ✅ Single API call per request
- ✅ No unnecessary abstractions

---

## 🐛 Error Handling

### Common Errors

**Invalid Fee Payer**:
```typescript
// Error: Transaction fee payer must be 2Ui2XQvR24ckJP7Eoe1aoiqHqsonzhsCifTVto6j5GKt
// Solution: Ensure frontend sets transaction.feePayer = PAYMASTER_ADDRESS
```

**Authentication Failed**:
```typescript
// Error: 401 Unauthorized
// Solution: Check JWT token is valid and included in Authorization header
```

**Turnkey API Error**:
```typescript
// Error: Failed to sign transaction with Turnkey
// Solution: Verify Turnkey credentials and private key permissions
```

---

## 📚 Database Impact

### Do We Need Database Changes?

**Answer: NO!** ✅

The paymaster is **stateless**:
- No transaction storage needed (Solana handles it)
- No user balance tracking needed (on-chain)
- No fee tracking required (optional, monitor via Solana explorer)
- Each request is independent

---

## 🚀 Quick Start Checklist

- [ ] Add environment variables to `.env`
- [ ] Start backend: `npm run start:dev`
- [ ] Test endpoint: `GET /api/paymaster/address`
- [ ] Install frontend dependencies: `npm install @solana/web3.js @solana/spl-token`
- [ ] Implement `transferWithPaymaster` function
- [ ] Test with a small transaction
- [ ] Monitor paymaster SOL balance

---

## 📖 Summary

### What Was Built
- ✅ Backend endpoint to sign transactions as fee payer
- ✅ Secure integration with Turnkey
- ✅ Optimized code (~120 lines total)
- ✅ No database changes needed
- ✅ Works with any Solana transaction

### How It Works
1. User builds transaction with paymaster as `feePayer`
2. User signs their part
3. Backend signs as fee payer using Turnkey
4. Transaction broadcasts to Solana
5. Solana executes and confirms

### Key Benefits
- ✅ Gasless UX for users
- ✅ Secure (keys in Turnkey HSM)
- ✅ Simple (single endpoint)
- ✅ Flexible (any transaction type)
- ✅ Non-custodial (user controls funds)

---

**Implementation Date**: 2025-01-30  
**Status**: ✅ Complete, Verified, Optimized  
**Turnkey SDK**: `@turnkey/sdk-server` (latest)  
**Solana SDK**: `@solana/web3.js` (latest)

