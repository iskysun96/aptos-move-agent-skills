---
name: troubleshoot-errors
description:
  Diagnose and fix common Aptos Move errors. Use when "fix error", "debug move", "compilation failed", "test failed", or
  AUTOMATICALLY when errors detected.
---

# Troubleshoot Errors Skill

## Overview

This skill helps diagnose and fix common errors in Aptos Move development.

## Error Categories

1. **Compilation Errors** - Code won't compile
2. **Linker Errors** - Dependencies not resolved
3. **Runtime Errors** - Aborts during execution
4. **Test Errors** - Tests failing
5. **Deployment Errors** - Publishing failed
6. **Type Errors** - Type mismatches
7. **Security-Critical Errors** ⭐ - Vulnerabilities that can be exploited

---

## Compilation Errors

### Error: "Expected ';'"

**Cause:** Missing semicolon at end of statement

**Example:**

```move
let x = 5  // Missing semicolon
```

**Fix:**

```move
let x = 5;  // ✅ Added semicolon
```

### Error: "Unbound variable"

**Cause:** Using variable before declaration or typo

**Example:**

```move
let result = value + 10;  // 'value' not declared
```

**Fix:**

```move
let value = 5;
let result = value + 10;  // ✅ Declared first
```

### Error: "Invalid function signature"

**Cause:** Function parameters or return type incorrect

**Example:**

```move
public fun transfer(item: address) { }  // Should be Object<Item>
```

**Fix:**

```move
public fun transfer(item: Object<Item>) { }  // ✅ Correct type
```

### Error: "Ability constraint not satisfied"

**Cause:** Type doesn't have required abilities

**Example:**

```move
struct Item { }  // No 'key' ability
fun init(account: &signer) {
    move_to(account, Item {});  // Error: Item needs 'key'
}
```

**Fix:**

```move
struct Item has key { }  // ✅ Added 'key' ability
```

---

## Linker Errors

### Error: "Package dependencies not resolved"

**Cause:** Missing dependencies in Move.toml

**Example Error:**

```
LINKER_ERROR: Unable to resolve address 'aptos_framework'
```

**Fix:**

```toml
# Add to Move.toml
[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"
```

### Error: "Unable to find module"

**Cause:** Module imported but not in dependencies or sources/

**Example:**

```move
use my_addr::missing_module;  // Module doesn't exist
```

**Fix:**

```move
// 1. Create the module file in sources/
// OR
// 2. Remove the unused import
```

### Error: "Named address not found"

**Cause:** Named address used but not defined in Move.toml

**Example Error:**

```
Address 'my_addr' is not defined
```

**Fix:**

```toml
# Add to Move.toml
[addresses]
my_addr = "_"

[dev-addresses]
my_addr = "0xCAFE"
```

---

## Runtime Errors (Aborts)

### Error: "ABORTED: 0x1"

**Cause:** Assertion failed with error code 1

**Example:**

```move
const E_NOT_OWNER: u64 = 1;
assert!(object::owner(obj) == user, E_NOT_OWNER);  // Failed
```

**Fix:**

1. Identify which assertion failed (error code 1 = E_NOT_OWNER)
2. Verify the condition (is user actually the owner?)
3. Fix the logic or provide correct parameters

**Debug:**

```move
// Add debug prints before assertion
debug::print(&object::owner(obj));
debug::print(&user);
assert!(object::owner(obj) == user, E_NOT_OWNER);
```

### Error: "RESOURCE_ALREADY_EXISTS"

**Cause:** Trying to move_to resource that already exists

**Example:**

```move
move_to(account, Counter { value: 0 });  // Already exists at this address
```

**Fix:**

```move
// Check if resource exists first
if (!exists<Counter>(signer::address_of(account))) {
    move_to(account, Counter { value: 0 });
};
```

### Error: "RESOURCE_NOT_FOUND"

**Cause:** Trying to borrow resource that doesn't exist

**Example:**

```move
let counter = borrow_global<Counter>(@0x1);  // Doesn't exist
```

**Fix:**

```move
// Verify resource exists first
assert!(exists<Counter>(@0x1), E_NOT_INITIALIZED);
let counter = borrow_global<Counter>(@0x1);
```

### Error: "ARITHMETIC_ERROR" (Overflow/Underflow)

**Cause:** Integer overflow or underflow

**Example:**

```move
let result = MAX_U64 + 1;  // Overflow
let result = 5 - 10;  // Underflow
```

**Fix:**

```move
// Check before operation
assert!(MAX_U64 - amount > 0, E_OVERFLOW);
let result = amount + 1;
```

---

## Security-Critical Errors ⭐ See [SECURITY.md](../../patterns/SECURITY.md)

### Error: "Fee calculated as zero" (Division Precision Loss)

**Cause:** Integer division rounds down, causing fees to be zero for small amounts

**Example:**
```move
const PROTOCOL_FEE_BPS: u64 = 30; // 0.3% = 30 basis points

public fun process_order(amount: u64) {
    // Amount = 500, fee = (500 * 30) / 10000 = 15000 / 10000 = 1 ✅
    // Amount = 100, fee = (100 * 30) / 10000 = 3000 / 10000 = 0 ❌ VULNERABILITY!
    let fee = (amount * PROTOCOL_FEE_BPS) / 10000;
    // ... charge fee
}
```

**Fix:**
```move
const MIN_ORDER_SIZE: u64 = 1000; // Minimum to ensure non-zero fees
const PROTOCOL_FEE_BPS: u64 = 30;

public fun process_order(amount: u64) {
    // Method 1: Enforce minimum threshold
    assert!(amount >= MIN_ORDER_SIZE, E_AMOUNT_TOO_SMALL);
    let fee = (amount * PROTOCOL_FEE_BPS) / 10000;

    // Method 2: Validate non-zero result
    assert!(fee > 0, E_AMOUNT_TOO_SMALL);
    // ... charge fee
}
```

**Security Impact:** CRITICAL - Users can bypass protocol fees

### Error: "Incorrect calculation result" (Left Shift Overflow)

**Cause:** Left shift (<<) does NOT abort on overflow - produces silent incorrect results

**Example:**
```move
public fun calculate_power_of_two(exponent: u8): u64 {
    // If exponent = 64, this produces 0 (wraps around)
    // If exponent = 65, this produces incorrect result
    // NO ABORT - SILENT FAILURE!
    1 << exponent
}
```

**Fix:**
```move
const E_OVERFLOW: u64 = 100;

public fun calculate_power_of_two(exponent: u8): u64 {
    // MUST validate shift amount manually
    assert!(exponent < 64, E_OVERFLOW);
    1 << exponent  // Safe after validation
}
```

**Security Impact:** CRITICAL - Can cause incorrect calculations in financial logic

### Error: "User modified other user's account" (Global Storage Scoping)

**Cause:** Function accepts arbitrary address parameter for `borrow_global_mut`, allowing cross-account manipulation

**Example:**
```move
// ❌ VULNERABLE CODE
public entry fun update_balance(
    user: &signer,
    target_addr: address,  // DANGEROUS!
    amount: u64
) acquires Account {
    // User can modify ANY account!
    let account = borrow_global_mut<Account>(target_addr);
    account.balance = account.balance + amount;
}
```

**Fix:**
```move
// ✅ SECURE CODE
public entry fun update_balance(
    user: &signer,
    amount: u64
) acquires Account {
    // Can ONLY modify signer's own account
    let user_addr = signer::address_of(user);
    let account = borrow_global_mut<Account>(user_addr);
    account.balance = account.balance + amount;
}
```

**Security Impact:** CRITICAL - Complete account takeover vulnerability

### Error: "Transaction out of gas" (Unbounded Iteration DOS)

**Cause:** Iterating over global storage (all users) causes gas exhaustion as system grows

**Example:**
```move
// ❌ VULNERABLE CODE
struct Registry has key {
    all_users: vector<address>,  // Grows unbounded!
}

public entry fun distribute_rewards(admin: &signer) acquires Registry {
    let registry = borrow_global<Registry>(@admin);

    // DOS: Gas cost grows with every user!
    let i = 0;
    while (i < vector::length(&registry.all_users)) {
        let user_addr = *vector::borrow(&registry.all_users, i);
        // ... give reward
        i = i + 1;
    };
}
```

**Fix:**
```move
// ✅ SECURE CODE - Per-user storage
struct UserReward has key {
    amount: u64,
}

public entry fun claim_reward(user: &signer) acquires UserReward {
    // O(1) operation - no iteration!
    let reward = borrow_global_mut<UserReward>(signer::address_of(user));
    // ... claim logic
}
```

**Security Impact:** HIGH - Denial of service attack vector

### Error: "Invariant violation after callback" (Reference Safety)

**Cause:** Passing `&mut` reference to external code that violates invariants

**Example:**
```move
// ❌ VULNERABLE CODE
public fun process_with_callback(
    obj: &mut MyObject,
    callback: |&mut MyObject|
) {
    // Check invariant before
    assert!(obj.value > 0, E_INVALID);

    // Call external code with mutable reference
    callback(obj);  // External code could violate invariants!

    // DON'T re-check after callback - VULNERABILITY!
    obj.value = obj.value + 100;
}
```

**Fix:**
```move
// ✅ SECURE CODE
public fun process_with_callback(
    obj: &mut MyObject,
    callback: |&mut MyObject|
) {
    // Check invariant before
    assert!(obj.value > 0, E_INVALID);

    // Call external code
    callback(obj);

    // CRITICAL: Re-validate invariants after callback!
    assert!(obj.value > 0, E_INVALID);
    assert!(obj.is_initialized, E_INVALID);

    obj.value = obj.value + 100;  // Safe now
}
```

**Security Impact:** CRITICAL - Invariant violations, corrupted state

### Error: "Front-running attack successful" (Non-Atomic Operations)

**Cause:** Operations split across multiple transactions allow front-running

**Example:**
```move
// ❌ VULNERABLE CODE - Two separate calls
public entry fun set_price(user: &signer, price: u64) acquires PriceOracle {
    let oracle = borrow_global_mut<PriceOracle>(@oracle_addr);
    oracle.current_price = price;
}

public entry fun evaluate_position(user: &signer) acquires PriceOracle {
    let oracle = borrow_global<PriceOracle>(@oracle_addr);
    // VULNERABLE: Attacker can front-run between set_price and evaluate_position!
    let price = oracle.current_price;
    // ... evaluate using price
}
```

**Fix:**
```move
// ✅ SECURE CODE - Atomic operation
public entry fun set_and_evaluate_price(
    user: &signer,
    price: u64
) acquires PriceOracle {
    // Both operations in same transaction - atomic!
    let oracle = borrow_global_mut<PriceOracle>(@oracle_addr);
    oracle.current_price = price;

    // Evaluate immediately - no front-running opportunity
    let result = evaluate_internal(price);
    // ... use result
}
```

**Security Impact:** HIGH - MEV attacks, sandwich attacks, price manipulation

---

## Test Errors

### Error: "Expected failure but test passed"

**Cause:** Test marked #[expected_failure] but didn't abort

**Example:**

```move
#[test]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_should_fail() {
    // Test didn't abort!
}
```

**Fix:**

```move
#[test]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_should_fail() {
    // Add code that actually aborts
    assert!(false, E_NOT_OWNER);
}
```

### Error: "Test failed: assertion failed"

**Cause:** Assertion in test failed

**Example:**

```move
#[test]
public fun test_value() {
    let x = get_value();
    assert!(x == 10, 0);  // Failed: x was 5, not 10
}
```

**Fix:**

```move
// Debug the issue
#[test]
public fun test_value() {
    let x = get_value();
    debug::print(&x);  // See actual value
    assert!(x == 5, 0);  // ✅ Corrected expected value
}
```

### Error: "Coverage below 100%"

**Cause:** Some code paths not tested

**Example:**

```
module: my_module
coverage: 85.5% (94/110 lines covered)

Uncovered lines:
- my_module.move:45
- my_module.move:67
- my_module.move:89
```

**Fix:**

```bash
# 1. View coverage report
aptos move coverage source --module my_module

# 2. Identify uncovered lines
# 3. Write tests for those paths
# 4. Verify 100% coverage
aptos move test --coverage
```

---

## Type Errors

### Error: "Type mismatch"

**Cause:** Wrong type provided

**Example:**

```move
public fun transfer(item: Object<Item>) { }

// Called with:
transfer(@0x123);  // Wrong: passing address, not Object<Item>
```

**Fix:**

```move
// Convert address to Object<Item>
let item = object::address_to_object<Item>(@0x123);
transfer(item);  // ✅ Correct type
```

### Error: "Generic type parameter not inferred"

**Cause:** Compiler can't determine generic type

**Example:**

```move
let empty = vector::empty();  // What type?
```

**Fix:**

```move
let empty = vector::empty<u64>();  // ✅ Explicit type
```

### Error: "Phantom type parameter not marked as phantom"

**Cause:** Generic type not stored in fields but not marked phantom

**Example:**

```move
struct Vault<CoinType> has key {  // CoinType not in fields
    balance: u64,
}
```

**Fix:**

```move
struct Vault<phantom CoinType> has key {  // ✅ Added phantom
    balance: u64,
}
```

---

## Object-Related Errors

### Error: "An object does not exist at this address"

**Cause:** Trying to access a resource at an object address that hasn't been created yet

**Common Scenarios:**

**Scenario 1: Collection owner can't create tokens**

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

**Fix:**

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

**Scenario 2: init_module never ran**

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

**Scenario 3: Wrong address calculation**

```move
// ❌ WRONG: Incorrect seed or creator address
let obj_addr = object::create_object_address(&wrong_creator, b"SEED");
let config = borrow_global<Config>(obj_addr);  // Doesn't exist at this address
```

**Fix:**

```move
// ✅ Use correct creator address and seed
let obj_addr = object::create_object_address(&@marketplace_addr, b"MARKETPLACE_STATE");
let config = borrow_global<MarketplaceConfig>(obj_addr);
```

### Error: "The object does not have ungated transfers enabled"

**Cause:** Trying to use `object::transfer()` on an object with disabled ungated transfers

**Example:**

```move
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

**Fix:**

```move
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

**Cause:** Named object address doesn't match expected address

**Example:**

```move
// Created object with one seed
let obj = object::create_named_object(creator, b"SEED_V1");

// Later, trying to access with different seed
let obj_addr = object::create_object_address(&creator_addr, b"SEED_V2");  // Wrong seed!
let data = borrow_global<Data>(obj_addr);  // Doesn't exist
```

**Fix:**

```move
// ✅ Use consistent seeds
const SEED: vector<u8> = b"MY_OBJECT_SEED";

// Creation
let obj = object::create_named_object(creator, SEED);

// Access
let obj_addr = object::create_object_address(&creator_addr, SEED);
let data = borrow_global<Data>(obj_addr);  // ✅ Correct
```

## Deployment Errors

### Error: "Insufficient APT balance"

**Cause:** Account doesn't have enough APT to pay gas

**Fix:**

```bash
# Testnet/Devnet: Use faucet
aptos account fund-with-faucet --profile testnet

# Mainnet: Transfer APT to account
```

### Error: "Module bytecode verification failed"

**Cause:** Module doesn't pass bytecode verifier

**Fix:**

```bash
# 1. Ensure code compiles locally
aptos move compile

# 2. Run Move prover (if available)
aptos move prove

# 3. Check for unsafe patterns
# - No unsafe code
# - No deprecated patterns
# - Proper ability constraints
```

### Error: "Upgrade compatibility check failed"

**Cause:** Upgrade breaks compatibility

**Example:**

```
Cannot remove public function 'old_function'
Cannot change signature of 'existing_function'
```

**Fix:**

```move
// ❌ Don't remove public functions
// ❌ Don't change function signatures

// ✅ Add new functions instead
public fun new_function() { }

// ✅ Deprecate old functions (keep them)
/// DEPRECATED: Use new_function() instead
public fun old_function() { }
```

---

## Common Error Patterns

### Pattern 1: Object Access Errors

**Problem:**

```move
let item = borrow_global<Item>(item_obj);  // Wrong: passing Object<Item>, need address
```

**Solution:**

```move
let item_addr = object::object_address(&item_obj);
let item = borrow_global<Item>(item_addr);  // ✅ Correct
```

### Pattern 2: Missing acquires

**Problem:**

```move
public fun get_balance(addr: address): u64 {  // Missing 'acquires'
    let account = borrow_global<Account>(addr);
    account.balance
}
```

**Solution:**

```move
public fun get_balance(addr: address): u64 acquires Account {  // ✅ Added
    let account = borrow_global<Account>(addr);
    account.balance
}
```

### Pattern 3: Incorrect Error Codes

**Problem:**

```move
assert!(condition, 0);  // Using 0 - unclear what failed
```

**Solution:**

```move
const E_CONDITION_FAILED: u64 = 1;
assert!(condition, E_CONDITION_FAILED);  // ✅ Clear error code
```

---

## Debugging Strategies

### Strategy 1: Add Debug Prints

```move
use std::debug;

public fun my_function(x: u64, y: u64) {
    debug::print(&x);
    debug::print(&y);

    let result = x + y;
    debug::print(&result);
}
```

### Strategy 2: Simplify Code

```move
// Complex (hard to debug)
let result = complex_calc(func1(x), func2(y), func3(z));

// Simplified (easier to debug)
let a = func1(x);
debug::print(&a);

let b = func2(y);
debug::print(&b);

let c = func3(z);
debug::print(&c);

let result = complex_calc(a, b, c);
debug::print(&result);
```

### Strategy 3: Test Incrementally

```move
#[test]
public fun test_step1() {
    // Test first part
}

#[test]
public fun test_step2() {
    // Test second part
}

#[test]
public fun test_step3() {
    // Test third part
}

#[test]
public fun test_full_flow() {
    // Test everything together
}
```

---

## Error Code Reference

### Aptos Framework Error Codes

Common framework error codes:

| Code  | Error             | Meaning                     |
| ----- | ----------------- | --------------------------- |
| `0x1` | INVALID_ARGUMENT  | Invalid function argument   |
| `0x2` | OUT_OF_RANGE      | Value out of valid range    |
| `0x3` | INVALID_STATE     | Invalid state for operation |
| `0x5` | NOT_FOUND         | Resource not found          |
| `0x6` | ALREADY_EXISTS    | Resource already exists     |
| `0x7` | PERMISSION_DENIED | Caller not authorized       |
| `0x8` | ABORTED           | Operation aborted           |

### Custom Error Codes

**Recommended structure:**

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
const E_ITEM_NOT_AVAILABLE: u64 = 31;
```

---

## ALWAYS Rules

- ✅ ALWAYS read error messages completely
- ✅ ALWAYS check error codes to identify which assertion failed
- ✅ ALWAYS use debug::print for debugging
- ✅ ALWAYS test incrementally
- ✅ ALWAYS define clear error constants
- ✅ ALWAYS verify fixes with tests

### Security Error Awareness ⭐ See [SECURITY.md](../../patterns/SECURITY.md)
- ✅ ALWAYS check for division precision loss (fees = 0)
- ✅ ALWAYS validate left shift amounts (< 64 for u64)
- ✅ ALWAYS scope global storage to signer (no arbitrary address parameters)
- ✅ ALWAYS avoid unbounded iterations (use per-user storage)
- ✅ ALWAYS re-validate after callbacks (reference safety)
- ✅ ALWAYS use atomic operations (prevent front-running)

## NEVER Rules

- ❌ NEVER ignore compiler warnings
- ❌ NEVER use generic error codes (0, 1, 2 without constants)
- ❌ NEVER skip testing after fixing
- ❌ NEVER deploy with known errors
- ❌ NEVER assume error location without verification

### Security Error Violations ⭐ CRITICAL
- ❌ NEVER ignore security-critical errors (fee bypass, overflow, cross-account access)
- ❌ NEVER deploy code with security vulnerabilities
- ❌ NEVER assume division always produces non-zero results
- ❌ NEVER assume left shift will abort on overflow (it doesn't!)
- ❌ NEVER accept arbitrary address parameters for borrow_global_mut

## References

**Official Documentation:**

- Move Book: https://aptos.dev/build/smart-contracts/book
- Error Codes: https://aptos.dev/build/smart-contracts/book/abort-and-assert

**Related Skills:**

- `write-contracts` - Write correct code
- `generate-tests` - Test for errors
- `security-audit` - Find potential issues
- `use-aptos-cli` - CLI error solutions

---

**Remember:** Read errors carefully, debug systematically, test thoroughly, verify fixes.
