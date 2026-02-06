# Transaction Patterns Reference

## Standard Transaction Flow

Every transaction follows the build-sign-submit-wait pattern:

```typescript
// 1. Build
const transaction = await aptos.transaction.build.simple({
  sender: account.accountAddress,
  data: {
    function: `${MODULE_ADDRESS}::module::function_name`,
    functionArguments: [arg1, arg2]
  }
});

// 2. Sign and Submit
const pendingTx = await aptos.signAndSubmitTransaction({
  signer: account,
  transaction
});

// 3. Wait for confirmation
const committedTx = await aptos.waitForTransaction({
  transactionHash: pendingTx.hash
});
```

---

## Transaction Simulation

```typescript
const [simResult] = await aptos.transaction.simulate.simple({
  signerPublicKey: account.publicKey,
  transaction
});

if (!simResult.success) {
  console.error("Would fail:", simResult.vm_status);
  return;
}

console.log("Estimated gas:", simResult.gas_used);
```

---

## Sponsored Transactions (Fee Payer)

A third party pays gas fees on behalf of the sender.

```typescript
// 1. Build with fee payer flag
const transaction = await aptos.transaction.build.simple({
  sender: sender.accountAddress,
  withFeePayer: true,
  data: {
    function: `${MODULE_ADDRESS}::module::function_name`,
    functionArguments: []
  }
});

// 2. Sender signs normally
const senderAuth = aptos.transaction.sign({
  signer: sender,
  transaction
});

// 3. Fee payer signs with different method
const feePayerAuth = aptos.transaction.signAsFeePayer({
  signer: feePayer,
  transaction
});

// 4. Submit with both authenticators
const pendingTx = await aptos.transaction.submit.simple({
  transaction,
  senderAuthenticator: senderAuth,
  feePayerAuthenticator: feePayerAuth
});

await aptos.waitForTransaction({ transactionHash: pendingTx.hash });
```

**Key rules:**

- Use `withFeePayer: true` when building
- Sender uses `.sign()`, fee payer uses `.signAsFeePayer()` (different methods)
- Fee payer must have enough APT to cover the gas fee

---

## Multi-Agent Transactions

Multiple accounts participate in a single transaction.

```typescript
// 1. Build with secondary signers
const transaction = await aptos.transaction.build.multiAgent({
  sender: alice.accountAddress,
  secondarySignerAddresses: [bob.accountAddress],
  data: {
    function: `${MODULE_ADDRESS}::escrow::exchange`,
    functionArguments: [itemAddress, paymentAmount]
  }
});

// 2. Each participant signs
const aliceAuth = aptos.transaction.sign({ signer: alice, transaction });
const bobAuth = aptos.transaction.sign({ signer: bob, transaction });

// 3. Submit with all signatures
const pendingTx = await aptos.transaction.submit.multiAgent({
  transaction,
  senderAuthenticator: aliceAuth,
  additionalSignersAuthenticators: [bobAuth]
});

await aptos.waitForTransaction({ transactionHash: pendingTx.hash });
```

---

## Deprecated APIs

The following APIs are deprecated and should NOT be used:

| Deprecated API                                    | Replacement                                 | Since |
| ------------------------------------------------- | ------------------------------------------- | ----- |
| `aptos.getAccountCoinAmount()`                    | `aptos.getBalance()`                        | v5.1  |
| `aptos.getAccountAPTAmount()`                     | `aptos.getBalance()`                        | v5.1  |
| `scriptComposer`                                  | Batch transactions or separate calls        | v6.0  |
| `aptos.deriveAccountFromPrivateKey()`             | `Account.fromPrivateKey()`                  | v3.1  |
