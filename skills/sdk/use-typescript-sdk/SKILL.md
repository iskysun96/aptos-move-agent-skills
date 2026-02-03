---
name: use-typescript-sdk
description: "Guides usage of @aptos-labs/ts-sdk for interacting with the Aptos blockchain from TypeScript/JavaScript. Triggers on: 'typescript sdk', 'ts-sdk', 'aptos sdk', 'aptos client', 'sdk setup', 'interact with contract', 'call aptos', 'aptos javascript'."
---

# Use TypeScript SDK Skill

## Overview

This skill covers using `@aptos-labs/ts-sdk` to interact with the Aptos blockchain from TypeScript/JavaScript applications. The SDK provides methods for account management, transaction building, view function calls, and more.

## Quick Setup

### Installation

```bash
npm install @aptos-labs/ts-sdk
```

### Basic Client Setup

```typescript
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

// Configure for specific network
const config = new AptosConfig({ network: Network.TESTNET });
const aptos = new Aptos(config);
```

### Network Options

```typescript
// Devnet
const config = new AptosConfig({ network: Network.DEVNET });

// Testnet
const config = new AptosConfig({ network: Network.TESTNET });

// Mainnet
const config = new AptosConfig({ network: Network.MAINNET });

// Custom endpoint
const config = new AptosConfig({
  network: Network.CUSTOM,
  fullnode: "https://your-fullnode.com/v1",
});
```

---

## Core Operations

### 1. View Functions (Read-Only)

View functions read blockchain state without transactions.

```typescript
// Call a view function
const result = await aptos.view({
  payload: {
    function: `${MODULE_ADDRESS}::counter::get_count`,
    functionArguments: [accountAddress],
  },
});

// Result is an array of return values
const count = result[0];
```

**With Type Arguments:**

```typescript
const balance = await aptos.view({
  payload: {
    function: "0x1::coin::balance",
    typeArguments: ["0x1::aptos_coin::AptosCoin"],
    functionArguments: [accountAddress],
  },
});
```

### 2. Entry Functions (Transactions)

Entry functions modify blockchain state via transactions.

```typescript
import { Account } from "@aptos-labs/ts-sdk";

// Create or load account
const account = Account.generate();
// Or from private key:
// const account = Account.fromPrivateKey({ privateKey: new Ed25519PrivateKey(key) });

// Build and submit transaction
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: [],
  },
});

// Sign and submit
const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction,
});

// Wait for confirmation
const committedTx = await aptos.waitForTransaction({
  transactionHash: pendingTx.hash,
});
```

### 3. Transaction with Arguments

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::marketplace::list_item`,
    functionArguments: [
      itemObjectAddress,  // address
      1000000,            // u64 (price in octas)
    ],
  },
});
```

**Type Arguments:**

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::vault::deposit`,
    typeArguments: ["0x1::aptos_coin::AptosCoin"],
    functionArguments: [amount],
  },
});
```

### 4. Account Operations

```typescript
// Get account balance
const balance = await aptos.getAccountAPTAmount({
  accountAddress: account.accountAddress,
});

// Get account resources
const resources = await aptos.getAccountResources({
  accountAddress: account.accountAddress,
});

// Get specific resource
const resource = await aptos.getAccountResource({
  accountAddress: account.accountAddress,
  resourceType: `${MODULE_ADDRESS}::counter::Counter`,
});

// Check if account exists
const accountInfo = await aptos.getAccountInfo({
  accountAddress: account.accountAddress,
});
```

### 5. Fund Account (Devnet/Testnet)

```typescript
// Fund from faucet (devnet/testnet only)
await aptos.fundAccount({
  accountAddress: account.accountAddress,
  amount: 100_000_000, // 1 APT = 100,000,000 octas
});
```

---

## Type Mappings

### Move to TypeScript

| Move Type | TypeScript | Example |
|-----------|------------|---------|
| `u8` | `number` | `255` |
| `u64` | `number \| bigint` | `1000000` |
| `u128` | `bigint` | `BigInt("1000000000000")` |
| `u256` | `bigint` | `BigInt("...")` |
| `bool` | `boolean` | `true` |
| `address` | `string` | `"0x1"` |
| `vector<u8>` | `Uint8Array` | `new Uint8Array([1,2,3])` |
| `String` | `string` | `"hello"` |
| `Object<T>` | `string` (address) | `"0xabc..."` |

### Argument Encoding

```typescript
// String
functionArguments: ["Hello World"]

// Numbers (u8, u64)
functionArguments: [42, 1000000]

// Large numbers (u128, u256)
functionArguments: [BigInt("1000000000000000000")]

// Address
functionArguments: ["0x1abc..."]

// Boolean
functionArguments: [true]

// Vector<u8> (bytes)
functionArguments: [new Uint8Array([1, 2, 3, 4])]

// Vector<address>
functionArguments: [["0x1", "0x2", "0x3"]]
```

---

## Common Patterns

### Pattern 1: Frontend dApp Setup

```typescript
// lib/aptos.ts
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const NETWORK = (import.meta.env.VITE_APP_NETWORK as Network) || Network.TESTNET;

const config = new AptosConfig({ network: NETWORK });
export const aptos = new Aptos(config);

export const MODULE_ADDRESS = import.meta.env.VITE_MODULE_ADDRESS;
```

### Pattern 2: View Function Hook

```typescript
// hooks/useCounter.ts
import { useState, useEffect } from "react";
import { aptos, MODULE_ADDRESS } from "../lib/aptos";

export function useCounter(accountAddress: string) {
  const [count, setCount] = useState<number | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);

  useEffect(() => {
    async function fetchCount() {
      try {
        setLoading(true);
        const result = await aptos.view({
          payload: {
            function: `${MODULE_ADDRESS}::counter::get_count`,
            functionArguments: [accountAddress],
          },
        });
        setCount(Number(result[0]));
      } catch (err) {
        setError(err as Error);
      } finally {
        setLoading(false);
      }
    }

    if (accountAddress) {
      fetchCount();
    }
  }, [accountAddress]);

  return { count, loading, error };
}
```

### Pattern 3: Transaction Builder

```typescript
// entry-functions/increment.ts
import { InputTransactionData } from "@aptos-labs/wallet-adapter-react";
import { MODULE_ADDRESS } from "../lib/aptos";

export function buildIncrementPayload(): InputTransactionData {
  return {
    data: {
      function: `${MODULE_ADDRESS}::counter::increment`,
      functionArguments: [],
    },
  };
}

// Usage with wallet adapter
const { signAndSubmitTransaction } = useWallet();
await signAndSubmitTransaction(buildIncrementPayload());
```

### Pattern 4: Error Handling

```typescript
try {
  const result = await aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::counter::get_count`,
      functionArguments: [accountAddress],
    },
  });
  return result[0];
} catch (error) {
  if (error instanceof Error) {
    // Check for specific error types
    if (error.message.includes("RESOURCE_NOT_FOUND")) {
      // Resource doesn't exist yet
      return 0;
    }
    if (error.message.includes("MODULE_NOT_FOUND")) {
      // Contract not deployed
      throw new Error("Contract not deployed");
    }
  }
  throw error;
}
```

---

## Transaction Simulation

Simulate before submitting to check for errors:

```typescript
// Build transaction
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: [],
  },
});

// Simulate to check for errors
const [simulationResult] = await aptos.transaction.simulate.simple({
  signerPublicKey: account.publicKey,
  transaction,
});

if (!simulationResult.success) {
  console.error("Simulation failed:", simulationResult.vm_status);
  throw new Error(`Transaction would fail: ${simulationResult.vm_status}`);
}

// If simulation passes, submit
const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction,
});
```

---

## Gas Estimation

```typescript
// Get gas estimate from simulation
const [simulationResult] = await aptos.transaction.simulate.simple({
  signerPublicKey: account.publicKey,
  transaction,
});

const estimatedGas = simulationResult.gas_used;
const maxGasAmount = Math.ceil(Number(estimatedGas) * 1.2); // 20% buffer

// Build with custom gas settings
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: [],
  },
  options: {
    maxGasAmount,
    gasUnitPrice: 100, // octas per gas unit
  },
});
```

---

## Event Queries

```typescript
// Get events by event handle
const events = await aptos.getAccountEventsByEventType({
  accountAddress: MODULE_ADDRESS,
  eventType: `${MODULE_ADDRESS}::counter::IncrementEvent`,
});

// Get events with pagination
const events = await aptos.getAccountEventsByEventType({
  accountAddress: MODULE_ADDRESS,
  eventType: `${MODULE_ADDRESS}::counter::IncrementEvent`,
  options: {
    offset: 0,
    limit: 25,
  },
});
```

---

## ALWAYS Rules

- ✅ ALWAYS use `AptosConfig` to configure the network
- ✅ ALWAYS handle errors appropriately (network, transaction failures)
- ✅ ALWAYS use `waitForTransaction` after submitting transactions
- ✅ ALWAYS simulate transactions before submitting for critical operations
- ✅ ALWAYS use environment variables for network and module addresses
- ✅ ALWAYS convert large numbers to `bigint` for u128/u256 types

## NEVER Rules

- ❌ NEVER hardcode private keys in frontend code
- ❌ NEVER skip error handling for blockchain operations
- ❌ NEVER assume transactions succeed without waiting
- ❌ NEVER use synchronous patterns for async SDK operations
- ❌ NEVER expose private keys in client-side code

---

## References

**Official Documentation:**
- TypeScript SDK: https://aptos.dev/sdks/ts-sdk
- SDK Reference: https://aptos-labs.github.io/aptos-ts-sdk/

**Related Skills:**
- `query-onchain-data` - Detailed query patterns
- `connect-contract-to-frontend` - Frontend integration
- `handle-transactions` - Transaction UX patterns

**Pattern Documentation:**
- `patterns/fullstack/TYPESCRIPT_SDK.md` - Complete SDK reference

---

**Remember:** Always handle errors and wait for transaction confirmation. Use environment variables for configuration.
