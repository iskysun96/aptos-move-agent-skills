# Skill: generate-move-scripts

Generate Move scripts for complex, atomic multi-operation transactions that execute multiple contract calls in a single
transaction.

## When to Use This Skill

**Trigger phrases:**

- "create move script", "write script", "atomic transaction"
- "multiple operations in one transaction", "batch operations"
- "complex transaction", "multi-step transaction"
- "atomic swap", "flash loan pattern"

**Use cases:**

- Atomic swaps between multiple tokens
- Complex DeFi operations (deposit + stake + claim)
- Multi-step NFT operations
- Flash loan implementations
- Batch updates across multiple contracts

## Move Scripts Overview

Move scripts are executable transaction payloads that:

- Run atomically (all-or-nothing execution)
- Can call multiple module functions
- Have access to signer capabilities
- Cannot be stored on-chain
- Perfect for complex, one-time operations

## Script Structure

### Basic Script Template

```move
script {
    use aptos_framework::coin;
    use aptos_framework::aptos_account;
    use 0x1::signer;

    fun main(account: &signer) {
        // Script logic here
        // All operations execute atomically
    }
}
```

### Multi-Signer Script

```move
script {
    use std::vector;

    fun main(
        account1: &signer,
        account2: &signer,
        account3: &signer,
    ) {
        // Multi-party operations
        // All signers must approve transaction
    }
}
```

## Common Script Patterns

### 1. Atomic Token Swap

```move
script {
    use aptos_framework::coin;
    use aptos_framework::aptos_account;
    use dex::swap_module;

    const E_INSUFFICIENT_OUTPUT: u64 = 1;

    fun atomic_swap<CoinIn, CoinOut>(
        trader: &signer,
        amount_in: u64,
        min_amount_out: u64,
        pool_address: address
    ) {
        // Extract coins from trader
        let coins_in = coin::withdraw<CoinIn>(trader, amount_in);

        // Perform swap
        let coins_out = swap_module::swap<CoinIn, CoinOut>(
            pool_address,
            coins_in
        );

        // Verify minimum output
        let output_amount = coin::value(&coins_out);
        assert!(output_amount >= min_amount_out, E_INSUFFICIENT_OUTPUT);

        // Deposit output to trader
        coin::deposit(signer::address_of(trader), coins_out);
    }
}
```

### 2. DeFi Composition

```move
script {
    use lending::lending_pool;
    use staking::stake_module;
    use farming::yield_farm;

    /// Deposit, stake, and farm in one transaction
    fun defi_combo<Asset>(
        user: &signer,
        amount: u64,
        pool_id: u64
    ) {
        // 1. Deposit to lending pool
        let receipt = lending_pool::deposit<Asset>(user, amount);

        // 2. Stake the receipt tokens
        let staked_receipt = stake_module::stake_receipt(user, receipt);

        // 3. Farm with staked tokens
        yield_farm::enter_farm(user, pool_id, staked_receipt);

        // All operations succeed or all revert
    }
}
```

### 3. NFT Batch Operations

```move
script {
    use aptos_token_objects::token;
    use marketplace::listing;
    use std::vector;

    /// List multiple NFTs with same price
    fun batch_list_nfts(
        seller: &signer,
        nfts: vector<Object<Token>>,
        price: u64,
        marketplace_addr: address
    ) {
        let i = 0;
        let len = vector::length(&nfts);

        while (i < len) {
            let nft = vector::borrow(&nfts, i);
            listing::create_listing(
                seller,
                *nft,
                price,
                marketplace_addr
            );
            i = i + 1;
        }
    }
}
```

### 4. Flash Loan Pattern

```move
script {
    use flash_loan::pool;
    use arbitrage::strategy;

    fun flash_loan_arbitrage<CoinType>(
        arbitrageur: &signer,
        loan_amount: u64,
        pool_address: address
    ) {
        // 1. Borrow funds
        let (loaned_coins, repayment_amount) =
            pool::borrow<CoinType>(pool_address, loan_amount);

        // 2. Execute arbitrage
        let profit_coins = strategy::execute_arbitrage<CoinType>(
            arbitrageur,
            loaned_coins
        );

        // 3. Repay loan with profit
        let repayment = coin::extract(&mut profit_coins, repayment_amount);
        pool::repay<CoinType>(pool_address, repayment);

        // 4. Keep remaining profit
        coin::deposit(signer::address_of(arbitrageur), profit_coins);
    }
}
```

### 5. Multi-Sig Treasury Operation

```move
script {
    use multisig::treasury;
    use aptos_framework::coin;

    fun execute_treasury_proposal(
        executor1: &signer,
        executor2: &signer,
        executor3: &signer,
        proposal_id: u64,
        treasury_address: address
    ) {
        // All signers must be valid executors
        treasury::verify_executors(
            treasury_address,
            vector[
                signer::address_of(executor1),
                signer::address_of(executor2),
                signer::address_of(executor3)
            ]
        );

        // Execute the proposal
        treasury::execute_proposal(
            treasury_address,
            proposal_id,
            vector[executor1, executor2, executor3]
        );
    }
}
```

## Script Development Workflow

### 1. Create Script File

```bash
# Create script file
touch scripts/my_script.move

# Basic structure
cat > scripts/my_script.move << 'EOF'
script {
    use std::signer;

    fun main(account: &signer) {
        // Your logic here
    }
}
EOF
```

### 2. Compile Script

```bash
# Compile script to bytecode
aptos move compile --save-metadata \
    --package-dir . \
    --named-addresses my_addr=0x123

# Verify compilation
ls build/MyPackage/scripts/
```

### 3. Run Script

```bash
# Execute script
aptos move run-script \
    --compiled-script-path build/MyPackage/scripts/my_script.mv \
    --args ... \
    --profile default
```

## Script Arguments

### Supported Argument Types

```move
script {
    fun main(
        // Signers (always first)
        account: &signer,

        // Primitive types
        amount: u64,
        flag: bool,
        small_value: u8,

        // Vectors
        addresses: vector<address>,
        amounts: vector<u64>,

        // Strings (as vectors)
        name_bytes: vector<u8>,
    ) {
        // Use arguments
    }
}
```

### Passing Arguments via CLI

```bash
aptos move run-script \
    --compiled-script-path script.mv \
    --args \
        u64:1000000 \
        bool:true \
        u8:5 \
        "address:[0x1,0x2,0x3]" \
        "u64:[100,200,300]" \
        "u8:[72,101,108,108,111]"  # "Hello" in bytes
```

## Advanced Script Patterns

### 1. Conditional Execution

```move
script {
    fun conditional_operation(
        user: &signer,
        condition: bool,
        amount: u64
    ) {
        if (condition) {
            // Path A: Stake tokens
            staking::stake(user, amount);
        } else {
            // Path B: Provide liquidity
            amm::add_liquidity(user, amount);
        }
    }
}
```

### 2. Error Recovery

```move
script {
    use std::error;

    const E_PARTIAL_FAILURE: u64 = 1;

    fun safe_batch_operation(
        user: &signer,
        operations: vector<u8>
    ) {
        let successes = vector::empty<bool>();
        let i = 0;

        while (i < vector::length(&operations)) {
            let op = *vector::borrow(&operations, i);

            // Try operation, track success
            if (try_operation(user, op)) {
                vector::push_back(&mut successes, true);
            } else {
                vector::push_back(&mut successes, false);
            }
            i = i + 1;
        }

        // Ensure at least one success
        assert!(vector::contains(&successes, &true), E_PARTIAL_FAILURE);
    }
}
```

### 3. Gas-Optimized Batch

```move
script {
    /// Process in chunks to manage gas
    fun chunked_batch_process(
        admin: &signer,
        items: vector<address>,
        chunk_size: u64
    ) {
        let i = 0;
        let len = vector::length(&items);

        while (i < len) {
            let chunk_end = i + chunk_size;
            if (chunk_end > len) {
                chunk_end = len;
            };

            // Process chunk
            process_chunk(admin, &items, i, chunk_end);

            i = chunk_end;
        }
    }
}
```

## Script Testing

### Unit Testing Scripts

```move
#[test_only]
module my_addr::script_tests {
    use std::signer;

    #[test]
    fun test_script_logic() {
        // Create test signers
        let account = account::create_account_for_test(@0x123);

        // Test script logic directly
        my_script::main(&account, 1000, true);

        // Verify results
        assert!(some_condition(), 0);
    }

    #[test(account1 = @0x123, account2 = @0x456)]
    fun test_multi_signer_script(
        account1: &signer,
        account2: &signer
    ) {
        // Test multi-signer scripts
        multi_sig_script::main(account1, account2);
    }
}
```

## Script vs Entry Function

### When to Use Scripts

- Complex atomic operations across multiple modules
- One-time operations (migrations, setup)
- Multi-signer coordination
- Operations requiring intermediate calculations
- Flash loan patterns

### When to Use Entry Functions

- Simple, reusable operations
- Single module operations
- Standard user interactions
- Operations needing minimal client logic

## Security Considerations

### Script Security Checklist

- [ ] Validate all inputs
- [ ] Check signer permissions
- [ ] Handle arithmetic overflow
- [ ] Verify external contract states
- [ ] Test all execution paths
- [ ] Consider reentrancy risks
- [ ] Validate type arguments

### Common Script Vulnerabilities

```move
script {
    // VULNERABLE: No validation
    fun unsafe_script(user: &signer, amount: u64) {
        // Missing: amount validation
        // Missing: permission checks
        transfer_tokens(user, @admin, amount);
    }

    // SECURE: Proper validation
    fun safe_script(user: &signer, amount: u64) {
        assert!(amount > 0 && amount <= MAX_AMOUNT, E_INVALID_AMOUNT);
        assert!(is_authorized(signer::address_of(user)), E_UNAUTHORIZED);
        transfer_tokens(user, @admin, amount);
    }
}
```

## Integration Examples

### DeFi Protocol Integration

```move
script {
    use uniswap::router;
    use compound::ctoken;
    use aave::lending_pool;

    /// Swap, lend, and stake in one transaction
    fun defi_strategy<TokenA, TokenB>(
        user: &signer,
        amount_a: u64,
        min_b: u64
    ) {
        // 1. Swap TokenA for TokenB
        let token_b = router::swap_exact_tokens_for_tokens<TokenA, TokenB>(
            user,
            amount_a,
            min_b,
            vector[@user],
            timestamp::now_microseconds() + 300 // 5 min deadline
        );

        // 2. Deposit half to Compound
        let half = coin::value(&token_b) / 2;
        let compound_deposit = coin::extract(&mut token_b, half);
        ctoken::mint<TokenB>(user, compound_deposit);

        // 3. Deposit remaining to Aave
        aave::deposit<TokenB>(user, token_b, signer::address_of(user), 0);
    }
}
```

## Best Practices

1. **Keep Scripts Simple**: Complex logic belongs in modules
2. **Validate Early**: Check all preconditions at start
3. **Use Type Parameters**: Make scripts generic when possible
4. **Document Arguments**: Clear parameter documentation
5. **Test Thoroughly**: Scripts can't be upgraded
6. **Gas Awareness**: Scripts run in single transaction
7. **Error Messages**: Provide clear abort codes

## Usage Notes

- Scripts are compiled to bytecode before execution
- Cannot import private functions from modules
- All operations must complete in single transaction
- Gas limit applies to entire script execution
- Scripts cannot store data on-chain
- Perfect for migrations and one-time operations

## References

- Move Book Scripts: https://move-language.github.io/move/modules-and-scripts.html
- Aptos Script Examples: Check `aptos-core/aptos-move/move-examples/scripts/`
- Script Compilation: https://aptos.dev/build/smart-contracts/scripts/
