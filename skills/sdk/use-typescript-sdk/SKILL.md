---
name: use-typescript-sdk
description:
  "Guides usage of @aptos-labs/ts-sdk for interacting with the Aptos blockchain from TypeScript/JavaScript applications.
  Covers client setup, account management, transaction building & submission, view functions, event queries, coin/token
  operations, wallet adapter integration, and error handling. Triggers on: 'typescript sdk', 'ts-sdk', 'aptos sdk',
  'aptos client', 'sdk setup', 'interact with contract', 'call aptos', 'aptos javascript', 'frontend integration',
  'wallet adapter', 'connect wallet'."
metadata:
  category: sdk
  tags: ["typescript", "sdk", "frontend", "wallet", "transactions"]
  priority: high
---

# Use TypeScript SDK Skill

## Core Rules

### SDK Setup

1. **ALWAYS use `@aptos-labs/ts-sdk`** (the current official SDK, NOT the deprecated `aptos` package)
2. **ALWAYS create a singleton `Aptos` client** and export it from a shared module (e.g., `lib/aptos.ts`)
3. **ALWAYS configure the network** via `AptosConfig` with environment variables
4. **ALWAYS use `Network.TESTNET`** as the default for development (NOT devnet, which resets frequently)

### Balance & Queries

5. **ALWAYS use `aptos.getBalance()`** for APT balance queries (NOT the deprecated `getAccountCoinAmount` or
   `getAccountAPTAmount`)

### Transactions

6. **ALWAYS call `waitForTransaction`** after submitting any transaction
7. **ALWAYS simulate transactions** before submitting for critical or high-value operations
8. **ALWAYS wrap blockchain calls** in try/catch with specific error handling
9. **ALWAYS use the build-sign-submit pattern**: `transaction.build.simple()` then `signAndSubmitTransaction()`

### Security

10. **NEVER hardcode private keys** in source code or frontend bundles
11. **NEVER expose private keys** in client-side code or logs
12. **NEVER store private keys** in environment variables accessible to the browser (use `VITE_` prefix only for public
    config)
13. **ALWAYS load private keys from environment variables** in server-side scripts only, using `process.env`

### Type Safety

14. **ALWAYS use `bigint`** for u128 and u256 values (JavaScript `number` loses precision)
15. **ALWAYS pass `Object<T>` references** as address strings in `functionArguments`
16. **ALWAYS use typed entry/view function wrappers** instead of raw string-based calls in production code

### Wallet Adapter

17. **ALWAYS use `@aptos-labs/wallet-adapter-react`** for frontend wallet integration
18. **ALWAYS wrap your app** with `AptosWalletAdapterProvider`
19. **ALWAYS use `useWallet()` hook** to access wallet functions in React components

## Quick Workflow

1. **Install SDK** -> `npm install @aptos-labs/ts-sdk`
2. **Create client** -> Singleton `Aptos` instance in `lib/aptos.ts`
3. **Read data** -> Use `aptos.view()` for on-chain reads
4. **Write data** -> Use `aptos.transaction.build.simple()` + `aptos.signAndSubmitTransaction()`
5. **Connect wallet** -> Use `@aptos-labs/wallet-adapter-react` for frontend dApps
6. **Handle errors** -> Wrap all calls in try/catch with error-type checks

## Key Example: Client Setup

```typescript
// lib/aptos.ts - Singleton client (create once, import everywhere)
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

export const MODULE_ADDRESS = import.meta.env.VITE_MODULE_ADDRESS;
```

## Key Example: View Functions (Read)

```typescript
// view-functions/getCount.ts
import { aptos, MODULE_ADDRESS } from "../lib/aptos";

export async function getCount(accountAddress: string): Promise<number> {
  const result = await aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::counter::get_count`,
      functionArguments: [accountAddress]
    }
  });
  return Number(result[0]);
}

// With type arguments
export async function getCoinBalance(accountAddress: string, coinType: string): Promise<bigint> {
  const result = await aptos.view({
    payload: {
      function: "0x1::coin::balance",
      typeArguments: [coinType],
      functionArguments: [accountAddress]
    }
  });
  return BigInt(result[0] as string);
}

// Multiple return values
// Move: public fun get_listing(addr): (address, u64, bool)
export async function getListing(nftAddress: string): Promise<{ seller: string; price: number; isActive: boolean }> {
  const [seller, price, isActive] = await aptos.view({
    payload: {
      function: `${MODULE_ADDRESS}::marketplace::get_listing`,
      functionArguments: [nftAddress]
    }
  });
  return {
    seller: seller as string,
    price: Number(price),
    isActive: isActive as boolean
  };
}
```

## Key Example: Entry Functions (Write) - Server/Script

```typescript
import { Account, Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const config = new AptosConfig({ network: Network.TESTNET });
const aptos = new Aptos(config);

// Build transaction
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: []
  }
});

// Sign and submit
const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction
});

// ALWAYS wait for confirmation
const committedTx = await aptos.waitForTransaction({
  transactionHash: pendingTx.hash
});

console.log("Success:", committedTx.success);
```

## Key Example: Entry Functions (Write) - Frontend with Wallet

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

// Component usage
import { useWallet } from "@aptos-labs/wallet-adapter-react";
import { aptos } from "../lib/aptos";
import { buildIncrementPayload } from "../entry-functions/increment";

function IncrementButton() {
  const { signAndSubmitTransaction } = useWallet();

  const handleClick = async () => {
    try {
      const response = await signAndSubmitTransaction(
        buildIncrementPayload(),
      );
      await aptos.waitForTransaction({
        transactionHash: response.hash,
      });
    } catch (error) {
      console.error("Transaction failed:", error);
    }
  };

  return <button onClick={handleClick}>Increment</button>;
}
```

## Key Example: Wallet Adapter Setup

```typescript
// main.tsx or App.tsx
import { AptosWalletAdapterProvider } from "@aptos-labs/wallet-adapter-react";
import { Network } from "@aptos-labs/ts-sdk";

function App() {
  return (
    <AptosWalletAdapterProvider
      autoConnect={true}
      dappConfig={{
        network: Network.TESTNET,
      }}
      onError={(error) => console.error("Wallet error:", error)}
    >
      <YourApp />
    </AptosWalletAdapterProvider>
  );
}

// In any child component
import { useWallet } from "@aptos-labs/wallet-adapter-react";

function WalletInfo() {
  const { account, connected, connect, disconnect, wallet, wallets } =
    useWallet();

  if (!connected) {
    return (
      <div>
        {wallets.map((w) => (
          <button key={w.name} onClick={() => connect(w.name)}>
            Connect {w.name}
          </button>
        ))}
      </div>
    );
  }

  return (
    <div>
      <p>Connected: {account?.address}</p>
      <p>Wallet: {wallet?.name}</p>
      <button onClick={disconnect}>Disconnect</button>
    </div>
  );
}
```

## Key Example: Sponsored Transactions (Fee Payer)

```typescript
import { Account, Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

const aptos = new Aptos(new AptosConfig({ network: Network.TESTNET }));

// 1. Build with fee payer flag
const transaction = await aptos.transaction.build.simple({
  sender: sender.accountAddress,
  withFeePayer: true,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: []
  }
});

// 2. Sender signs
const senderAuth = aptos.transaction.sign({
  signer: sender,
  transaction
});

// 3. Fee payer signs (different method!)
const feePayerAuth = aptos.transaction.signAsFeePayer({
  signer: feePayer,
  transaction
});

// 4. Submit with both signatures
const pendingTx = await aptos.transaction.submit.simple({
  transaction,
  senderAuthenticator: senderAuth,
  feePayerAuthenticator: feePayerAuth
});

await aptos.waitForTransaction({ transactionHash: pendingTx.hash });
```

## Key Example: Multi-Agent Transactions

```typescript
// 1. Build multi-agent transaction
const transaction = await aptos.transaction.build.multiAgent({
  sender: alice.accountAddress,
  secondarySignerAddresses: [bob.accountAddress],
  data: {
    function: `${MODULE_ADDRESS}::escrow::exchange`,
    functionArguments: [itemAddress, paymentAmount]
  }
});

// 2. Each agent signs
const aliceAuth = aptos.transaction.sign({
  signer: alice,
  transaction
});
const bobAuth = aptos.transaction.sign({
  signer: bob,
  transaction
});

// 3. Submit with all signatures
const pendingTx = await aptos.transaction.submit.multiAgent({
  transaction,
  senderAuthenticator: aliceAuth,
  additionalSignersAuthenticators: [bobAuth]
});

await aptos.waitForTransaction({ transactionHash: pendingTx.hash });
```

## Type Mappings: Move to TypeScript

| Move Type    | TypeScript Type        | Example                                             |
| ------------ | ---------------------- | --------------------------------------------------- |
| `u8`         | `number`               | `255`                                               |
| `u16`        | `number`               | `65535`                                             |
| `u32`        | `number`               | `4294967295`                                        |
| `u64`        | `number \| bigint`     | `1000000`                                           |
| `u128`       | `bigint`               | `BigInt("340282366920938463463374607431768211455")` |
| `u256`       | `bigint`               | `BigInt("...")`                                     |
| `i8`         | `number`               | `-128` (Move 2.3+)                                  |
| `i16`        | `number`               | `-32768` (Move 2.3+)                                |
| `i32`        | `number`               | `-2147483648` (Move 2.3+)                           |
| `i64`        | `number \| bigint`     | Use `bigint` for large values (Move 2.3+)           |
| `i128`       | `bigint`               | `BigInt("-170141183460469231731687303715884105728")` |
| `i256`       | `bigint`               | `BigInt("...")` (Move 2.3+)                         |
| `bool`       | `boolean`              | `true`                                              |
| `address`    | `string`               | `"0x1"`                                             |
| `String`     | `string`               | `"hello"`                                           |
| `vector<u8>` | `Uint8Array \| string` | `new Uint8Array([1,2,3])` or hex string             |
| `vector<T>`  | `T[]`                  | `[1, 2, 3]` for `vector<u64>`                       |
| `Object<T>`  | `string`               | Object address as hex string                        |
| `Option<T>`  | `T \| null`            | Value or `null`                                     |

## Transaction Simulation

```typescript
// Build transaction
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: []
  }
});

// Simulate to check for errors and estimate gas
const [simResult] = await aptos.transaction.simulate.simple({
  signerPublicKey: account.publicKey,
  transaction
});

if (!simResult.success) {
  throw new Error(`Simulation failed: ${simResult.vm_status}`);
}

console.log("Gas estimate:", simResult.gas_used);
```

## Gas Profiling

```typescript
// Profile gas usage of a transaction (useful for optimization)
const gasProfile = await aptos.gasProfile({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::module::function_name`,
    functionArguments: [],
  },
});

console.log("Gas profile:", gasProfile);
```

## Anti-patterns

1. **NEVER use the deprecated `aptos` npm package** - use `@aptos-labs/ts-sdk` instead
2. **NEVER skip `waitForTransaction`** after submitting - transaction may not be committed yet
3. **NEVER hardcode module addresses** - use environment variables (`VITE_MODULE_ADDRESS`)
4. **NEVER use `number` for u128/u256 values** - JavaScript loses precision above 2^53; use `bigint`
5. **NEVER create multiple `Aptos` client instances** - create one singleton and share it
6. **NEVER ignore transaction simulation results** for high-value operations
7. **NEVER hardcode network selection** - use environment-based configuration
8. **NEVER store private keys in browser-accessible env vars** (e.g., `VITE_PRIVATE_KEY`)
9. **NEVER use `Account.generate()` in frontend code** for real users - use wallet adapter instead
10. **NEVER use raw `aptos.signAndSubmitTransaction` in React** - use the wallet adapter's `signAndSubmitTransaction`
11. **NEVER use `scriptComposer`** - it was removed in v6.0; use batch transactions or separate calls instead
12. **NEVER use `getAccountCoinAmount` or `getAccountAPTAmount`** - deprecated; use `getBalance()` instead

## Edge Cases to Handle

| Scenario                | Check                                            | Action                                       |
| ----------------------- | ------------------------------------------------ | -------------------------------------------- |
| Resource not found      | `error.message.includes("RESOURCE_NOT_FOUND")`   | Return default value or null                 |
| Module not deployed     | `error.message.includes("MODULE_NOT_FOUND")`     | Show "contract not deployed" message         |
| Function not found      | `error.message.includes("FUNCTION_NOT_FOUND")`   | Check function name and module address       |
| Move abort              | `error.message.includes("ABORTED")`              | Parse abort code, map to user-friendly error |
| Out of gas              | `error.message.includes("OUT_OF_GAS")`           | Increase `maxGasAmount` and retry            |
| Sequence number error   | `error.message.includes("SEQUENCE_NUMBER")`      | Retry after fetching fresh sequence number   |
| Network timeout         | `error.message.includes("timeout")`              | Retry with exponential backoff               |
| Account does not exist  | `error.message.includes("ACCOUNT_NOT_FOUND")`    | Fund account or prompt user to create one    |
| Insufficient balance    | `error.message.includes("INSUFFICIENT_BALANCE")` | Show balance and required amount             |
| User rejected in wallet | Wallet-specific rejection error                  | Show "transaction cancelled" message         |

## Error Handling Pattern

```typescript
async function submitTransaction(
  aptos: Aptos,
  signer: Account,
  payload: InputGenerateTransactionPayloadData
): Promise<string> {
  try {
    const transaction = await aptos.transaction.build.simple({
      sender: signer.accountAddress,
      data: payload
    });

    const pendingTx = await aptos.signAndSubmitTransaction({
      signer,
      transaction
    });

    const committed = await aptos.waitForTransaction({
      transactionHash: pendingTx.hash
    });

    if (!committed.success) {
      throw new Error(`Transaction failed: ${committed.vm_status}`);
    }

    return pendingTx.hash;
  } catch (error) {
    if (error instanceof Error) {
      if (error.message.includes("RESOURCE_NOT_FOUND")) {
        throw new Error("Resource does not exist at the specified address");
      }
      if (error.message.includes("MODULE_NOT_FOUND")) {
        throw new Error("Contract is not deployed at the specified address");
      }
      if (error.message.includes("ABORTED")) {
        const match = error.message.match(/code: (\d+)/);
        const code = match ? match[1] : "unknown";
        throw new Error(`Contract error (code ${code})`);
      }
    }
    throw error;
  }
}
```

## File Organization Pattern

```
src/
  lib/
    aptos.ts                    # Singleton Aptos client + MODULE_ADDRESS
  view-functions/
    getCount.ts                 # One file per view function
    getListing.ts
  entry-functions/
    increment.ts                # One file per entry function
    createListing.ts
  hooks/
    useCounter.ts               # React hooks wrapping view functions
    useListing.ts
  components/
    WalletProvider.tsx           # AptosWalletAdapterProvider wrapper
    IncrementButton.tsx          # Components calling entry functions
```

## SDK Version Notes (v5.2+ / v6.0)

### Balance Queries (v5.1+)

```typescript
// CORRECT (v5.1+)
const balance = await aptos.getBalance({
  accountAddress: account.accountAddress,
});
// Returns bigint in octas (1 APT = 100_000_000 octas)

// DEPRECATED - do NOT use
// await aptos.getAccountCoinAmount(...)
// await aptos.getAccountAPTAmount(...)
```

### AIP-80 Private Key Format (v2.0+)

Ed25519 and Secp256k1 private keys now use an AIP-80 prefixed format when serialized with `toString()`:

```typescript
const key = new Ed25519PrivateKey("0x...");
key.toString(); // Returns AIP-80 prefixed format, NOT raw hex
```

### AccountAddress Parsing (v1.32+)

`AccountAddress.fromString()` now only accepts SHORT format (60-64 hex chars) by default. Use
`AccountAddress.from()` for flexible parsing:

```typescript
// CORRECT
const addr = AccountAddress.from("0x1"); // Accepts any format
const addr2 = AccountAddress.fromString(
  "0x000000000000000000000000000000000000000000000000000000000000001",
); // SHORT format only

// MAY FAIL in v1.32+
// AccountAddress.fromString("0x1") -- too short for SHORT format
```

### Fungible Asset Transfers (v1.39+)

```typescript
// Transfer between FA stores directly
await aptos.transferFungibleAssetBetweenStores({
  sender: account,
  fungibleAssetMetadataAddress: metadataAddr,
  senderStoreAddress: fromStore,
  recipientStoreAddress: toStore,
  amount: 1000n,
});
```

### Bun Runtime Compatibility

When using Bun instead of Node.js, disable HTTP/2 in the client config:

```typescript
const config = new AptosConfig({
  network: Network.TESTNET,
  clientConfig: { http2: false },
});
```

### Account Abstraction (v1.34+, AIP-104)

The `aptos.abstraction` namespace provides APIs for custom authentication:

```typescript
// Check if AA is enabled for an account
const isEnabled =
  await aptos.abstraction.isAccountAbstractionEnabled({
    accountAddress: "0x...",
    authenticationFunction: `${MODULE_ADDRESS}::auth::authenticate`,
  });

// Enable AA on an account
const enableTxn =
  await aptos.abstraction.enableAccountAbstractionTransaction({
    accountAddress: account.accountAddress,
    authenticationFunction: `${MODULE_ADDRESS}::auth::authenticate`,
  });

// Disable AA
const disableTxn =
  await aptos.abstraction.disableAccountAbstractionTransaction({
    accountAddress: account.accountAddress,
    authenticationFunction: `${MODULE_ADDRESS}::auth::authenticate`,
  });

// Use AbstractedAccount for signing with custom auth logic
import { AbstractedAccount } from "@aptos-labs/ts-sdk";
```

## References

**Detailed Patterns (references/ folder):**

- `references/wallet-adapter.md` - Wallet adapter setup and patterns
- `references/transaction-patterns.md` - Advanced transaction patterns (sponsored, multi-agent, simulation)
- `references/type-mappings.md` - Complete Move-to-TypeScript type reference

**Pattern Documentation (patterns/ folder):**

- `../../../patterns/fullstack/TYPESCRIPT_SDK.md` - Complete SDK API reference

**Official Documentation:**

- TypeScript SDK: https://aptos.dev/build/sdks/ts-sdk
- SDK Quickstart: https://aptos.dev/build/sdks/ts-sdk/quickstart
- API Reference: https://aptos-labs.github.io/aptos-ts-sdk/
- Wallet Adapter: https://aptos.dev/build/sdks/wallet-adapter/dapp
- Sponsored Transactions: https://aptos.dev/build/sdks/ts-sdk/building-transactions/sponsoring-transactions
- Multi-Agent Transactions: https://aptos.dev/build/sdks/ts-sdk/building-transactions/multi-agent-transactions
- GitHub: https://github.com/aptos-labs/aptos-ts-sdk

**Related Skills:**

- `write-contracts` - Write the Move contracts that this SDK interacts with
- `deploy-contracts` - Deploy contracts before calling them from TypeScript
- `scaffold-project` - Bootstrap a fullstack project with SDK already configured
