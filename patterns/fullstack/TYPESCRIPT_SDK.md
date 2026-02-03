# Aptos TypeScript SDK Reference

**Purpose:** Comprehensive reference for `@aptos-labs/ts-sdk` - the official TypeScript SDK for Aptos blockchain.

**Package:** `@aptos-labs/ts-sdk`

---

## Installation

```bash
npm install @aptos-labs/ts-sdk
# or
pnpm add @aptos-labs/ts-sdk
# or
yarn add @aptos-labs/ts-sdk
```

---

## Client Configuration

### Basic Setup

```typescript
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

// Testnet (recommended for development)
const config = new AptosConfig({ network: Network.TESTNET });
const aptos = new Aptos(config);
```

### All Network Options

```typescript
// Devnet - Frequent resets, latest features
const devnetConfig = new AptosConfig({ network: Network.DEVNET });

// Testnet - Stable test environment
const testnetConfig = new AptosConfig({ network: Network.TESTNET });

// Mainnet - Production
const mainnetConfig = new AptosConfig({ network: Network.MAINNET });

// Custom endpoint
const customConfig = new AptosConfig({
  network: Network.CUSTOM,
  fullnode: "https://your-fullnode.example.com/v1",
  indexer: "https://your-indexer.example.com/v1/graphql", // Optional
  faucet: "https://your-faucet.example.com", // Optional
});
```

### Environment-Based Configuration

```typescript
// lib/aptos.ts
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

function getNetwork(): Network {
  const network = import.meta.env.VITE_APP_NETWORK;
  switch (network) {
    case "mainnet":
      return Network.MAINNET;
    case "testnet":
      return Network.TESTNET;
    case "devnet":
      return Network.DEVNET;
    default:
      return Network.TESTNET;
  }
}

const config = new AptosConfig({ network: getNetwork() });
export const aptos = new Aptos(config);

// Export module address from environment
export const MODULE_ADDRESS = import.meta.env.VITE_MODULE_ADDRESS;
```

---

## Account Management

### Generate New Account

```typescript
import { Account, Ed25519PrivateKey } from "@aptos-labs/ts-sdk";

// Generate random account
const account = Account.generate();

console.log("Address:", account.accountAddress.toString());
console.log("Public Key:", account.publicKey.toString());
console.log("Private Key:", account.privateKey.toString()); // Keep secret!
```

### Load from Private Key

```typescript
import { Account, Ed25519PrivateKey } from "@aptos-labs/ts-sdk";

// From hex string
const privateKey = new Ed25519PrivateKey("0x...");
const account = Account.fromPrivateKey({ privateKey });

// From bytes
const privateKeyBytes = new Uint8Array([...]);
const privateKey2 = new Ed25519PrivateKey(privateKeyBytes);
const account2 = Account.fromPrivateKey({ privateKey: privateKey2 });
```

### Fund Account (Devnet/Testnet Only)

```typescript
// Request 1 APT from faucet
await aptos.fundAccount({
  accountAddress: account.accountAddress,
  amount: 100_000_000, // 1 APT = 100,000,000 octas
});
```

### Account Queries

```typescript
// Get APT balance
const balance = await aptos.getAccountAPTAmount({
  accountAddress: account.accountAddress,
});

// Get all resources
const resources = await aptos.getAccountResources({
  accountAddress: account.accountAddress,
});

// Get specific resource
const resource = await aptos.getAccountResource({
  accountAddress: account.accountAddress,
  resourceType: `${MODULE_ADDRESS}::counter::Counter`,
});

// Get account info (sequence number, auth key)
const info = await aptos.getAccountInfo({
  accountAddress: account.accountAddress,
});

// Get account modules (deployed code)
const modules = await aptos.getAccountModules({
  accountAddress: account.accountAddress,
});
```

---

## View Functions (Read-Only Queries)

View functions read blockchain state without creating transactions.

### Basic View Function

```typescript
const result = await aptos.view({
  payload: {
    function: `${MODULE_ADDRESS}::counter::get_count`,
    functionArguments: [accountAddress],
  },
});

// result is an array of return values
const count = result[0]; // First return value
```

### With Type Arguments

```typescript
// Generic function: get_balance<CoinType>(account: address): u64
const result = await aptos.view({
  payload: {
    function: "0x1::coin::balance",
    typeArguments: ["0x1::aptos_coin::AptosCoin"],
    functionArguments: [accountAddress],
  },
});
```

### Multiple Return Values

```typescript
// Move: public fun get_stats(): (u64, u64, bool)
const [total, active, isPaused] = await aptos.view({
  payload: {
    function: `${MODULE_ADDRESS}::stats::get_stats`,
    functionArguments: [],
  },
});
```

### Handling Errors

```typescript
async function safeViewCall<T>(
  functionId: string,
  args: any[],
  typeArgs: string[] = []
): Promise<T | null> {
  try {
    const result = await aptos.view({
      payload: {
        function: functionId,
        typeArguments: typeArgs,
        functionArguments: args,
      },
    });
    return result[0] as T;
  } catch (error) {
    if (error instanceof Error) {
      if (error.message.includes("RESOURCE_NOT_FOUND")) {
        return null; // Resource doesn't exist
      }
    }
    throw error;
  }
}

// Usage
const count = await safeViewCall<number>(
  `${MODULE_ADDRESS}::counter::get_count`,
  [accountAddress]
);
```

---

## Entry Functions (Transactions)

Entry functions modify blockchain state.

### Simple Transaction

```typescript
// Build transaction
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

console.log("Transaction successful:", committedTx.success);
```

### Transaction with Arguments

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::marketplace::create_listing`,
    functionArguments: [
      itemAddress,    // address (Object<T>)
      1000000,        // u64 (price in octas)
      86400,          // u64 (duration in seconds)
    ],
  },
});
```

### Transaction with Type Arguments

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::vault::deposit`,
    typeArguments: ["0x1::aptos_coin::AptosCoin"],
    functionArguments: [
      1000000, // u64 (amount)
    ],
  },
});
```

### Custom Gas Settings

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: [],
  },
  options: {
    maxGasAmount: 10000,
    gasUnitPrice: 100,
    expireTimestamp: Math.floor(Date.now() / 1000) + 600, // 10 min
  },
});
```

### Transaction Simulation

```typescript
// Build transaction
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: [],
  },
});

// Simulate to check for errors and estimate gas
const [simulationResult] = await aptos.transaction.simulate.simple({
  signerPublicKey: account.publicKey,
  transaction,
});

if (!simulationResult.success) {
  throw new Error(`Simulation failed: ${simulationResult.vm_status}`);
}

console.log("Estimated gas:", simulationResult.gas_used);
console.log("Changes:", simulationResult.changes);

// If simulation passes, submit
const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction,
});
```

---

## Type Mappings

### Move Types to TypeScript

| Move Type | TypeScript Type | Example |
|-----------|-----------------|---------|
| `u8` | `number` | `255` |
| `u16` | `number` | `65535` |
| `u32` | `number` | `4294967295` |
| `u64` | `number \| bigint` | `1000000` or `BigInt(1000000)` |
| `u128` | `bigint` | `BigInt("340282366920938463463374607431768211455")` |
| `u256` | `bigint` | `BigInt("...")` |
| `bool` | `boolean` | `true`, `false` |
| `address` | `string` | `"0x1"` |
| `String` | `string` | `"hello"` |
| `vector<u8>` | `Uint8Array \| string` | `new Uint8Array([1,2,3])` or hex string |
| `vector<T>` | `T[]` | `[1, 2, 3]` for `vector<u64>` |
| `Object<T>` | `string` | Object address as string |
| `Option<T>` | `T \| null` | Value or null |

### Encoding Examples

```typescript
// String
functionArguments: ["Hello World"]

// Numbers (u8, u16, u32, u64 - small values)
functionArguments: [42, 1000, 1000000]

// Large numbers (u64 large values, u128, u256)
functionArguments: [BigInt("1000000000000000000")]

// Address
functionArguments: ["0x1abc...def"]

// Object<T> - just the object's address
functionArguments: ["0x1234...object_address"]

// Boolean
functionArguments: [true, false]

// Bytes (vector<u8>)
functionArguments: [new Uint8Array([1, 2, 3, 4])]

// Vector of numbers
functionArguments: [[1, 2, 3, 4, 5]]

// Vector of addresses
functionArguments: [["0x1", "0x2", "0x3"]]

// Vector of strings
functionArguments: [["alice", "bob", "charlie"]]
```

---

## Events

### Query Events by Type

```typescript
// Get all events of a type
const events = await aptos.getAccountEventsByEventType({
  accountAddress: MODULE_ADDRESS,
  eventType: `${MODULE_ADDRESS}::counter::IncrementEvent`,
});

// With pagination
const events = await aptos.getAccountEventsByEventType({
  accountAddress: MODULE_ADDRESS,
  eventType: `${MODULE_ADDRESS}::counter::IncrementEvent`,
  options: {
    offset: 0,
    limit: 25,
  },
});
```

### Event Structure

```typescript
interface AptosEvent {
  guid: {
    creation_number: string;
    account_address: string;
  };
  sequence_number: string;
  type: string;
  data: any; // Event-specific data
}
```

---

## Resource Queries

### Get All Resources

```typescript
const resources = await aptos.getAccountResources({
  accountAddress: account.accountAddress,
});

// Find specific resource
const counterResource = resources.find(
  (r) => r.type === `${MODULE_ADDRESS}::counter::Counter`
);
```

### Get Specific Resource

```typescript
const resource = await aptos.getAccountResource({
  accountAddress: account.accountAddress,
  resourceType: `${MODULE_ADDRESS}::counter::Counter`,
});

// resource.data contains the resource fields
console.log("Count:", resource.data.count);
```

### Check Resource Existence

```typescript
async function resourceExists(
  address: string,
  resourceType: string
): Promise<boolean> {
  try {
    await aptos.getAccountResource({
      accountAddress: address,
      resourceType,
    });
    return true;
  } catch {
    return false;
  }
}
```

---

## Coin Operations

### Check APT Balance

```typescript
const balance = await aptos.getAccountAPTAmount({
  accountAddress: account.accountAddress,
});

// Convert to APT (from octas)
const aptBalance = balance / 100_000_000;
```

### Check Custom Coin Balance

```typescript
const result = await aptos.view({
  payload: {
    function: "0x1::coin::balance",
    typeArguments: [`${COIN_MODULE}::my_coin::MyCoin`],
    functionArguments: [accountAddress],
  },
});
const balance = result[0];
```

### Transfer APT

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: sender.accountAddress,
  data: {
    function: "0x1::aptos_account::transfer",
    functionArguments: [
      recipientAddress,
      1000000, // 0.01 APT
    ],
  },
});

const pendingTx = await aptos.signAndSubmitTransaction({
  signer: sender,
  transaction,
});

await aptos.waitForTransaction({ transactionHash: pendingTx.hash });
```

---

## Error Handling

### Common Error Types

```typescript
try {
  const result = await aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::module::function`,
      functionArguments: [],
    },
  });
} catch (error) {
  if (error instanceof Error) {
    const message = error.message;

    if (message.includes("RESOURCE_NOT_FOUND")) {
      // Resource doesn't exist at address
    } else if (message.includes("MODULE_NOT_FOUND")) {
      // Module not published at address
    } else if (message.includes("FUNCTION_NOT_FOUND")) {
      // Function doesn't exist in module
    } else if (message.includes("ABORTED")) {
      // Move abort - extract code from message
      const match = message.match(/ABORTED.*code: (\d+)/);
      const abortCode = match ? parseInt(match[1]) : null;
    } else if (message.includes("OUT_OF_GAS")) {
      // Transaction ran out of gas
    } else if (message.includes("SEQUENCE_NUMBER")) {
      // Sequence number mismatch
    }
  }
}
```

### Retry Pattern

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts: number = 3,
  delay: number = 1000
): Promise<T> {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxAttempts) throw error;
      await new Promise((resolve) => setTimeout(resolve, delay * attempt));
    }
  }
  throw new Error("Unreachable");
}

// Usage
const result = await withRetry(() =>
  aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::counter::get_count`,
      functionArguments: [address],
    },
  })
);
```

---

## Best Practices

### 1. Singleton Client

```typescript
// lib/aptos.ts - Create once, export everywhere
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const config = new AptosConfig({ network: Network.TESTNET });
export const aptos = new Aptos(config);
```

### 2. Environment Variables

```typescript
// .env
VITE_APP_NETWORK=testnet
VITE_MODULE_ADDRESS=0x123...

// Usage
const network = import.meta.env.VITE_APP_NETWORK;
const moduleAddress = import.meta.env.VITE_MODULE_ADDRESS;
```

### 3. Type-Safe View Functions

```typescript
// view-functions/getCount.ts
import { aptos, MODULE_ADDRESS } from "../lib/aptos";

export async function getCount(accountAddress: string): Promise<number> {
  const result = await aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::counter::get_count`,
      functionArguments: [accountAddress],
    },
  });
  return Number(result[0]);
}
```

### 4. Type-Safe Entry Functions

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
```

---

## References

**Official Documentation:**
- TypeScript SDK: https://aptos.dev/sdks/ts-sdk
- API Reference: https://aptos-labs.github.io/aptos-ts-sdk/
- GitHub: https://github.com/aptos-labs/aptos-ts-sdk

**Related Patterns:**
- `WALLET_ADAPTER.md` - Wallet integration
- `FRONTEND_PATTERNS.md` - React patterns

**Related Skills:**
- `query-onchain-data` - Query patterns
- `connect-contract-to-frontend` - Frontend integration
- `handle-transactions` - Transaction UX

---

**Remember:** Always handle errors, wait for transaction confirmation, and use environment variables for configuration.
