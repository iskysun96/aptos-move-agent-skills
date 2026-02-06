# Aptos TypeScript SDK Reference

**Purpose:** Comprehensive reference for `@aptos-labs/ts-sdk` - the official TypeScript SDK for Aptos blockchain.

**Package:** `@aptos-labs/ts-sdk` (NOT the deprecated `aptos` package)

---

## Installation

```bash
npm install @aptos-labs/ts-sdk
```

---

## Client Configuration

```typescript
import { Aptos, AptosConfig, Network } from "@aptos-labs/ts-sdk";

// Testnet (recommended for development)
const config = new AptosConfig({ network: Network.TESTNET });
const aptos = new Aptos(config);
```

### All Network Options

```typescript
const devnetConfig = new AptosConfig({ network: Network.DEVNET });
const testnetConfig = new AptosConfig({ network: Network.TESTNET });
const mainnetConfig = new AptosConfig({ network: Network.MAINNET });

// Custom endpoint
const customConfig = new AptosConfig({
  network: Network.CUSTOM,
  fullnode: "https://your-fullnode.example.com/v1",
  indexer: "https://your-indexer.example.com/v1/graphql",
  faucet: "https://your-faucet.example.com"
});
```

---

## Account Management

```typescript
import { Account, Ed25519PrivateKey } from "@aptos-labs/ts-sdk";

// Generate random account (server-side scripts only!)
const account = Account.generate();

// From private key (server-side only - load from env var)
const privateKey = new Ed25519PrivateKey(process.env.PRIVATE_KEY!);
const account2 = Account.fromPrivateKey({ privateKey });

// Fund account (devnet/testnet only)
await aptos.fundAccount({
  accountAddress: account.accountAddress,
  amount: 100_000_000 // 1 APT = 100,000,000 octas
});

// Account queries
const balance = await aptos.getAccountAPTAmount({
  accountAddress: account.accountAddress
});
const resources = await aptos.getAccountResources({
  accountAddress: account.accountAddress
});
const resource = await aptos.getAccountResource({
  accountAddress: account.accountAddress,
  resourceType: `${MODULE_ADDRESS}::counter::Counter`
});
```

---

## View Functions (Read-Only)

```typescript
const result = await aptos.view({
  payload: {
    function: `${MODULE_ADDRESS}::counter::get_count`,
    functionArguments: [accountAddress]
  }
});
const count = Number(result[0]);

// With type arguments
const result2 = await aptos.view({
  payload: {
    function: "0x1::coin::balance",
    typeArguments: ["0x1::aptos_coin::AptosCoin"],
    functionArguments: [accountAddress]
  }
});
```

---

## Entry Functions (Transactions)

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: []
  }
});

const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction
});

const committedTx = await aptos.waitForTransaction({
  transactionHash: pendingTx.hash
});
```

---

## Sponsored Transactions

```typescript
const transaction = await aptos.transaction.build.simple({
  sender: sender.accountAddress,
  withFeePayer: true,
  data: {
    function: `${MODULE_ADDRESS}::counter::increment`,
    functionArguments: []
  }
});

const senderAuth = aptos.transaction.sign({ signer: sender, transaction });
const feePayerAuth = aptos.transaction.signAsFeePayer({
  signer: feePayer,
  transaction
});

const pendingTx = await aptos.transaction.submit.simple({
  transaction,
  senderAuthenticator: senderAuth,
  feePayerAuthenticator: feePayerAuth
});
```

---

## Multi-Agent Transactions

```typescript
const transaction = await aptos.transaction.build.multiAgent({
  sender: alice.accountAddress,
  secondarySignerAddresses: [bob.accountAddress],
  data: {
    function: `${MODULE_ADDRESS}::escrow::exchange`,
    functionArguments: []
  }
});

const aliceAuth = aptos.transaction.sign({ signer: alice, transaction });
const bobAuth = aptos.transaction.sign({ signer: bob, transaction });

const pendingTx = await aptos.transaction.submit.multiAgent({
  transaction,
  senderAuthenticator: aliceAuth,
  additionalSignersAuthenticators: [bobAuth]
});
```

---

## Type Mappings: Move to TypeScript

| Move Type    | TypeScript Type        | Example                      |
| ------------ | ---------------------- | ---------------------------- |
| `u8`         | `number`               | `255`                        |
| `u16`        | `number`               | `65535`                      |
| `u32`        | `number`               | `4294967295`                 |
| `u64`        | `number \| bigint`     | `1000000`                    |
| `u128`       | `bigint`               | `BigInt("...")`              |
| `u256`       | `bigint`               | `BigInt("...")`              |
| `bool`       | `boolean`              | `true`                       |
| `address`    | `string`               | `"0x1"`                      |
| `String`     | `string`               | `"hello"`                    |
| `vector<u8>` | `Uint8Array \| string` | `new Uint8Array([1,2,3])`    |
| `vector<T>`  | `T[]`                  | `[1, 2, 3]`                  |
| `Object<T>`  | `string`               | Object address as hex string |
| `Option<T>`  | `T \| null`            | Value or `null`              |

---

## Events

```typescript
const events = await aptos.getAccountEventsByEventType({
  accountAddress: MODULE_ADDRESS,
  eventType: `${MODULE_ADDRESS}::counter::IncrementEvent`,
  options: { offset: 0, limit: 25 }
});
```

---

## Error Handling

```typescript
try {
  const result = await aptos.view({ payload: { ... } });
} catch (error) {
  if (error instanceof Error) {
    const msg = error.message;
    if (msg.includes("RESOURCE_NOT_FOUND")) { /* Resource doesn't exist */ }
    else if (msg.includes("MODULE_NOT_FOUND")) { /* Module not published */ }
    else if (msg.includes("ABORTED")) { /* Move abort */ }
    else if (msg.includes("OUT_OF_GAS")) { /* Ran out of gas */ }
  }
}
```

---

## References

- TypeScript SDK: https://aptos.dev/build/sdks/ts-sdk
- API Reference: https://aptos-labs.github.io/aptos-ts-sdk/
- GitHub: https://github.com/aptos-labs/aptos-ts-sdk
- Wallet Adapter: https://aptos.dev/build/sdks/wallet-adapter/dapp
