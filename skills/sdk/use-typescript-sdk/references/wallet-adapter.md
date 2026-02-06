# Wallet Adapter Reference

## Installation

```bash
npm install @aptos-labs/wallet-adapter-react
```

Modern AIP-62 standard wallets (Petra, Nightly, etc.) are autodetected and do NOT require additional packages. Legacy
wallets need their plugin package installed separately.

---

## Provider Setup

Wrap your app root with `AptosWalletAdapterProvider`:

```tsx
// main.tsx
import { AptosWalletAdapterProvider } from "@aptos-labs/wallet-adapter-react";
import { Network } from "@aptos-labs/ts-sdk";

function App() {
  return (
    <AptosWalletAdapterProvider
      autoConnect={true}
      dappConfig={{
        network: Network.TESTNET
      }}
      onError={(error) => console.error("Wallet error:", error)}
    >
      <YourApp />
    </AptosWalletAdapterProvider>
  );
}
```

---

## useWallet Hook

```tsx
import { useWallet } from "@aptos-labs/wallet-adapter-react";

function MyComponent() {
  const {
    account, // Current connected account { address, publicKey }
    connected, // Boolean: is wallet connected?
    wallet, // Current wallet info { name, icon, url }
    wallets, // Array of available wallets
    connect, // (walletName) => Promise<void>
    disconnect, // () => Promise<void>
    signAndSubmitTransaction, // Submit entry function calls
    signTransaction, // Sign without submitting
    submitTransaction, // Submit a signed transaction
    signMessage, // Sign an arbitrary message
    signMessageAndVerify, // Sign and verify a message
    changeNetwork, // Switch networks (not all wallets support this)
    network // Current network info
  } = useWallet();
}
```

---

## Submitting Transactions

```tsx
import { useWallet } from "@aptos-labs/wallet-adapter-react";
import { aptos } from "../lib/aptos";

function TransferButton() {
  const { signAndSubmitTransaction } = useWallet();

  const handleTransfer = async () => {
    try {
      const response = await signAndSubmitTransaction({
        data: {
          function: "0x1::aptos_account::transfer",
          functionArguments: [recipientAddress, amount]
        }
      });

      // ALWAYS wait for transaction confirmation
      await aptos.waitForTransaction({
        transactionHash: response.hash
      });
    } catch (error) {
      console.error("Transfer failed:", error);
    }
  };

  return <button onClick={handleTransfer}>Transfer</button>;
}
```

---

## Common Mistakes

| Mistake                                       | Fix                                                     |
| --------------------------------------------- | ------------------------------------------------------- |
| Missing `AptosWalletAdapterProvider` wrapper  | Wrap app root with the provider                         |
| Not waiting for transaction after submit      | Always call `aptos.waitForTransaction()`                |
| Using SDK `signAndSubmitTransaction` in React | Use the wallet adapter's `signAndSubmitTransaction`     |
| Not handling user rejection                   | Catch and check for rejection-related error messages    |
| Hardcoding wallet names                       | Use `wallets` array from `useWallet()` for dynamic list |
