---
name: troubleshoot-errors
description:
  "Diagnoses and fixes Aptos Move compilation, runtime, and deployment errors. Triggers on: 'error', 'fix this',
  'debug', 'troubleshoot', 'why is this failing', error codes like 'EOBJECT_DOES_NOT_EXIST', 'ABORTED',
  'RESOURCE_NOT_FOUND', 'Type mismatch', 'ability constraint'."
metadata:
  category: move
  tags: ["debugging", "errors", "troubleshooting", "fixes"]
  priority: medium
---

# Troubleshoot Errors Skill

## Quick Triage Workflow

### Step 1: Identify Error Category

- **Compilation error** → Check syntax, types, abilities
- **Linker error** → Check dependencies, named addresses
- **Runtime abort (ABORTED)** → Check error code, find failed assertion
- **Object error** → See Object-Related Errors section below (CRITICAL)
- **Test error** → Check assertions, expected failures
- **Type error** → Check generic types, type conversions

### Step 2: Top 10 Common Errors (Quick Fixes)

1. **"object does not exist"** → Verify seed/creator address for named objects
2. **"RESOURCE_NOT_FOUND"** → Add `acquires` clause to function
3. **"Type mismatch"** → Use `object::address_to_object<T>()` to convert address to Object
4. **"Ability constraint not satisfied"** → Add required ability (key, drop, copy, store)
5. **"unbound variable"** → Declare variable before use or fix typo
6. **"missing acquires"** → Add `acquires ResourceType` to function signature
7. **"Named address not found"** → Add address to Move.toml `[addresses]` section
8. **"Package dependencies not resolved"** → Add dependency to Move.toml
9. **"Expected semicolon"** → Add semicolon at end of statement
10. **"ungated transfers disabled"** → Use `object::transfer_with_ref()` instead of `object::transfer()`

See `references/error-catalog.md` for complete error database.

## Object-Related Errors ⭐ CRITICAL

These are the most common and complex errors in Aptos Move V2.

### Error: "An object does not exist at this address"

**Cause:** Trying to access a resource at an object address that hasn't been created yet.

**Common Scenarios:**

#### Scenario 1: Collection owner can't create tokens

```move
// ❌ WRONG: Storing collection's extend_ref
fun init_module(deployer: &signer) {
    let collection_ref = collection::create_unlimited_collection(...);
    let collection_signer = object::generate_signer(&collection_ref);

    move_to(&collection_signer, Config {
        extend_ref: object::generate_extend_ref(&collection_ref),  // Wrong ref!
    });
}

public entry fun mint_nft() acquires Config {
    let config = borrow_global<Config>(...);
    let signer = object::generate_signer_for_extending(&config.extend_ref);

    // ❌ ERROR: Collection can't create tokens in itself
    token::create_named_token(&signer, ...);
}
```

```move
// ✅ CORRECT: Create marketplace object that OWNS the collection
fun init_module(deployer: &signer) {
    // Create marketplace state object
    let marketplace_ref = object::create_named_object(deployer, b"MARKETPLACE_STATE");
    let marketplace_signer = object::generate_signer(&marketplace_ref);

    // Marketplace creates collection (marketplace is the owner)
    collection::create_unlimited_collection(&marketplace_signer, ...);

    // Store MARKETPLACE object's extend_ref
    move_to(&marketplace_signer, MarketplaceConfig {
        extend_ref: object::generate_extend_ref(&marketplace_ref),
    });
}

public entry fun mint_nft() acquires MarketplaceConfig {
    let config = borrow_global<MarketplaceConfig>(...);

    // Use marketplace signer (collection owner)
    let marketplace_signer = object::generate_signer_for_extending(&config.extend_ref);
    token::create_named_token(&marketplace_signer, ...);  // ✅ Works!
}
```

**Key lesson:** When minting tokens into a collection, use the **collection owner's** signer, not the collection's
signer.

#### Scenario 2: init_module never ran

```move
// During deployment, init_module failed an assertion
fun init_module(deployer: &signer) {
    assert!(signer::address_of(deployer) == @marketplace_addr, E_NOT_AUTHORIZED);
    // If this fails, MarketplaceConfig is never created
    move_to(deployer, MarketplaceConfig { ... });
}
```

**Fix:**

- Verify `@marketplace_addr` resolves to correct address during deployment
- For object deployment, use `aptos move deploy-object` not `publish`
- Check deployment transaction succeeded

#### Scenario 3: Wrong address calculation

```move
// ❌ WRONG: Incorrect seed or creator address
let obj_addr = object::create_object_address(&wrong_creator, b"SEED");
let config = borrow_global<Config>(obj_addr);  // Doesn't exist at this address
```

```move
// ✅ CORRECT: Use correct creator address and seed
let obj_addr = object::create_object_address(&@marketplace_addr, b"MARKETPLACE_STATE");
let config = borrow_global<MarketplaceConfig>(obj_addr);
```

### Error: "The object does not have ungated transfers enabled"

**Cause:** Trying to use `object::transfer()` on an object with disabled ungated transfers.

```move
// ❌ WRONG
public entry fun mint_and_transfer(creator: &signer, recipient: address) {
    let constructor_ref = token::create_named_token(...);
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Disable ungated transfers
    object::disable_ungated_transfer(&transfer_ref);

    let token_obj = object::object_from_constructor_ref<Token>(&constructor_ref);

    // ❌ ERROR: ungated transfers are disabled!
    object::transfer(creator, token_obj, recipient);
}
```

```move
// ✅ CORRECT
public entry fun mint_and_transfer(creator: &signer, recipient: address) {
    let constructor_ref = token::create_named_token(...);
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Disable ungated transfers
    object::disable_ungated_transfer(&transfer_ref);

    // ✅ Use transfer_with_ref instead
    let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, recipient);

    // Store transfer_ref for future transfers
    let token_signer = object::generate_signer(&constructor_ref);
    move_to(&token_signer, TokenData { transfer_ref, ... });
}
```

**Rule:**

- `object::disable_ungated_transfer()` called? → Use `object::transfer_with_ref()`
- Ungated transfers enabled (default)? → Use `object::transfer()`

### Error: "Object address derivation mismatch"

**Cause:** Named object address doesn't match expected address.

```move
// ❌ WRONG: Inconsistent seeds
let obj = object::create_named_object(creator, b"SEED_V1");

// Later, trying to access with different seed
let obj_addr = object::create_object_address(&creator_addr, b"SEED_V2");  // Wrong seed!
```

```move
// ✅ CORRECT: Use consistent seeds
const SEED: vector<u8> = b"MY_OBJECT_SEED";

// Creation
let obj = object::create_named_object(creator, SEED);

// Access
let obj_addr = object::create_object_address(&creator_addr, SEED);
```

## Common Error Patterns

### Pattern 1: Object Access

```move
// ❌ WRONG
let item = borrow_global<Item>(item_obj);  // Passing Object<Item>, need address

// ✅ CORRECT
let item_addr = object::object_address(&item_obj);
let item = borrow_global<Item>(item_addr);
```

### Pattern 2: Missing acquires

```move
// ❌ WRONG
public fun get_balance(addr: address): u64 {  // Missing 'acquires'
    let account = borrow_global<Account>(addr);
    account.balance
}

// ✅ CORRECT
public fun get_balance(addr: address): u64 acquires Account {
    let account = borrow_global<Account>(addr);
    account.balance
}
```

### Pattern 3: Incorrect Error Codes

```move
// ❌ WRONG
assert!(condition, 0);  // Using 0 - unclear what failed

// ✅ CORRECT
const E_CONDITION_FAILED: u64 = 1;
assert!(condition, E_CONDITION_FAILED);
```

## Debugging Strategies

### 1. Add Debug Prints

```move
use std::debug;

public fun my_function(x: u64, y: u64) {
    debug::print(&x);
    debug::print(&y);
    let result = x + y;
    debug::print(&result);
}
```

### 2. Simplify Code

Break complex expressions into simple steps with debug prints between each step.

### 3. Test Incrementally

Write separate tests for each step of complex logic.

### 4. Check Error Codes

```move
// When you see: ABORTED: 0x1 (error code 1)
// Find the constant with value 1:
const E_NOT_OWNER: u64 = 1;  // This is what failed
```

## Custom Error Code Structure

```move
// Access control: 1-9
const E_NOT_OWNER: u64 = 1;
const E_NOT_ADMIN: u64 = 2;
const E_UNAUTHORIZED: u64 = 3;

// Input validation: 10-19
const E_ZERO_AMOUNT: u64 = 10;
const E_AMOUNT_TOO_HIGH: u64 = 11;
const E_INVALID_ADDRESS: u64 = 12;

// State errors: 20-29
const E_NOT_INITIALIZED: u64 = 20;
const E_ALREADY_INITIALIZED: u64 = 21;
const E_PAUSED: u64 = 22;

// Business logic: 30+
const E_INSUFFICIENT_BALANCE: u64 = 30;
```

## ALWAYS Rules

1. **Read error messages completely** - Don't guess from partial info
2. **Check error codes** - Identify which assertion failed
3. **Use debug::print** - Add debugging output systematically
4. **Test incrementally** - Don't test everything at once
5. **Define clear error constants** - No magic numbers
6. **Verify fixes with tests** - Ensure error doesn't recur

## NEVER Rules

1. **Never ignore compiler warnings** - They often indicate real bugs
2. **Never use generic error codes** - Always define descriptive constants
3. **Never skip testing after fixing** - Regression tests are critical
4. **Never deploy with known errors** - Fix all errors before deployment
5. **Never assume error location** - Verify with debug prints
6. **Never read `~/.aptos/config.yaml` or `.env` to debug issues** — these contain private keys; use `aptos account list` to verify account config instead
7. **Never run `echo $VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY`** or `env | grep KEY` — these expose private keys
8. **Never display private key values** that appear in error output or user messages

## References

**Official Documentation:**

- Move Book: https://aptos.dev/build/smart-contracts/book
- Error Codes: https://aptos.dev/build/smart-contracts/book/abort-and-assert

**Related Skills:**

- `write-contracts` - Write correct code to avoid errors
- `generate-tests` - Test for errors proactively
- `security-audit` - Find potential issues before deployment

---

**Remember:** Object errors are the most common in V2. Check seeds, creator addresses, and which signer you're using.
