---
name: security-audit
description:
  Perform comprehensive security audit on Aptos Move V2 contracts before deployment. Use when "audit contract", "check
  security", "review move code", or AUTOMATICALLY before deployment.
---

# Security Audit Skill

## Overview

This skill performs systematic security audits of Move contracts using a comprehensive checklist. Every item must pass
before deployment.

**Critical:** Security is non-negotiable. User funds depend on correct implementation.

## Core Workflow

### Step 1: Run Security Checklist

Review ALL categories in order (based on [SECURITY.md](../../patterns/SECURITY.md)):

1. **Access Control** - Who can call functions?
2. **Input Validation** - Are inputs checked (including minimums)?
3. **Object Safety** - Object model used correctly?
4. **Reference Safety** - No dangerous references exposed?
5. **Arithmetic Safety** - Overflow/underflow/precision loss prevented?
6. **Resource Management** - No unbounded iterations?
7. **Generic Type Safety** - Phantom types & flash loans?
8. **Business Logic** - Front-running, oracles, IDs, pause?
9. **Randomness** - Entry functions, gas balanced? (if applicable)
10. **Testing** - 100% coverage achieved?
11. **Operations** - Key management, upgrades secure?

### Step 2: Access Control Audit

**Verify:**

- [ ] All `entry` functions verify signer authority
- [ ] Object ownership checked with `object::owner()`
- [ ] **CRITICAL**: `borrow_global_mut` scoped to signer's address (NO arbitrary address parameters)
- [ ] Admin functions check caller is admin
- [ ] Function visibility uses least-privilege (private > public(friend) > public > entry)
- [ ] No public functions modify state without authorization checks

**Check for:**

```move
// ✅ CORRECT: Signer verification
public entry fun update_config(admin: &signer, value: u64) acquires Config {
    let config = borrow_global<Config>(@my_addr);
    assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
    // Safe to proceed
}

// ❌ WRONG: No verification
public entry fun update_config(admin: &signer, value: u64) acquires Config {
    let config = borrow_global_mut<Config>(@my_addr);
    config.value = value; // Anyone can call!
}
```

**For objects:**

```move
// ✅ CORRECT: Ownership verification
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    to: address
) acquires Item {
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);
    // Safe to transfer
}

// ❌ WRONG: No ownership check
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    to: address
) acquires Item {
    // Anyone can transfer any item!
}
```

**CRITICAL - Global Storage Scoping:**

```move
// ✅ CORRECT: Scoped to signer
public entry fun update_balance(user: &signer, amount: u64) acquires Account {
    let user_addr = signer::address_of(user);
    // Can only modify signer's own account
    let account = borrow_global_mut<Account>(user_addr);
    account.balance = account.balance + amount;
}

// ❌ WRONG: Accepts arbitrary address
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

### Step 3: Input Validation Audit

**Verify:**

- [ ] Numeric inputs checked for zero: `assert!(amount > 0, E_ZERO_AMOUNT)`
- [ ] **CRITICAL**: Minimum thresholds enforced: `assert!(amount >= MIN_SIZE, E_TOO_SMALL)` (prevents fee rounding to zero)
- [ ] **CRITICAL**: Fee results validated: `let fee = calc_fee(amount); assert!(fee > 0, E_TOO_SMALL);`
- [ ] Numeric inputs within max limits: `assert!(amount <= MAX, E_AMOUNT_TOO_HIGH)`
- [ ] Vector lengths validated: `assert!(vector::length(&v) > 0, E_EMPTY_VECTOR)`
- [ ] String lengths checked: `assert!(string::length(&s) <= MAX_LENGTH, E_NAME_TOO_LONG)`
- [ ] Addresses validated: `assert!(addr != @0x0, E_ZERO_ADDRESS)`
- [ ] Enum-like values in range: `assert!(type_id < MAX_TYPES, E_INVALID_TYPE)`
- [ ] Generic types consistent (flash loan repayment matches borrowed type)

**Check for:**

```move
// ✅ CORRECT: Comprehensive validation
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    assert!(amount > 0, E_ZERO_AMOUNT);
    assert!(amount <= MAX_DEPOSIT_AMOUNT, E_AMOUNT_TOO_HIGH);

    let account = borrow_global_mut<Account>(signer::address_of(user));
    assert!(account.balance <= MAX_U64 - amount, E_OVERFLOW);

    account.balance = account.balance + amount;
}

// ❌ WRONG: No validation
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));
    account.balance = account.balance + amount; // Can overflow!
}
```

### Step 4: Object Safety Audit

**Verify:**

- [ ] ConstructorRef never returned from public functions
- [ ] All refs (TransferRef, DeleteRef, ExtendRef) generated in constructor
- [ ] Object signer only used during construction or with ExtendRef
- [ ] Ungated transfers disabled unless explicitly needed
- [ ] DeleteRef only generated for truly burnable objects

**Check for:**

```move
// ❌ DANGEROUS: Returning ConstructorRef
public fun create_item(): ConstructorRef {
    let constructor_ref = object::create_object(@my_addr);
    constructor_ref // Caller can destroy object!
}

// ✅ CORRECT: Return Object<T>
public fun create_item(creator: &signer): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Item { transfer_ref, delete_ref });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}
```

### Step 5: Reference Safety Audit

**Verify:**

- [ ] No `&mut` references exposed in public function signatures
- [ ] Critical fields protected from `mem::swap`
- [ ] Mutable borrows minimized in scope

**Check for:**

```move
// ❌ DANGEROUS: Exposing mutable reference
public fun get_item_mut(item: Object<Item>): &mut Item acquires Item {
    borrow_global_mut<Item>(object::object_address(&item))
    // Caller can mem::swap fields!
}

// ✅ CORRECT: Controlled mutations
public entry fun update_item_name(
    owner: &signer,
    item: Object<Item>,
    new_name: String
) acquires Item {
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = new_name;
}
```

### Step 6: Arithmetic Safety Audit

**Verify:**

- [ ] Additions checked for overflow (Move aborts automatically)
- [ ] Subtractions checked for underflow (Move aborts automatically)
- [ ] Division by zero prevented: `assert!(divisor > 0, E_DIVISION_BY_ZERO)`
- [ ] **CRITICAL**: Division precision loss handled (validate `fee > 0`)
- [ ] **CRITICAL**: Left shift `<<` validated or avoided (does NOT abort on overflow!)
- [ ] Multiplication checked for overflow when needed

**Check for:**

```move
// ✅ CORRECT: Overflow protection
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));

    // Check overflow BEFORE adding
    assert!(account.balance <= MAX_U64 - amount, E_OVERFLOW);

    account.balance = account.balance + amount;
}

// ✅ CORRECT: Underflow protection
public entry fun withdraw(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));

    // Check underflow BEFORE subtracting
    assert!(account.balance >= amount, E_INSUFFICIENT_BALANCE);

    account.balance = account.balance - amount;
}

// ❌ WRONG: No overflow check
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));
    account.balance = account.balance + amount; // Can overflow!
}
```

### Step 7: Generic Type Safety Audit

**Verify:**

- [ ] Phantom types used for type witnesses: `struct Vault<phantom CoinType>`
- [ ] Generic constraints appropriate: `<T: copy + drop>`
- [ ] No type confusion possible

**Check for:**

```move
// ✅ CORRECT: Phantom type for safety
struct Vault<phantom CoinType> has key {
    balance: u64,
    // CoinType only for type safety, not stored
}

public fun deposit<CoinType>(vault: Object<Vault<CoinType>>, amount: u64) {
    // Type-safe: can't deposit BTC into USDC vault
}

// ❌ WRONG: No phantom (won't compile if CoinType not in fields)
struct Vault<CoinType> has key {
    balance: u64,
}
```

### Step 7.1: Resource Management Audit

**Verify:**

- [ ] **CRITICAL**: No unbounded iterations over global storage (gas exhaustion DOS)
- [ ] User data stored in user accounts, not global vectors
- [ ] SmartTable used for per-user scalable data
- [ ] Each independent object in separate account (not multiple resources in one)
- [ ] Module data in Objects, user data in user accounts

**Check for:**

```move
// ❌ DANGEROUS: Unbounded global iteration
struct GlobalOrders has key {
    orders: vector<Order>,  // Grows unboundedly
}

public fun process_all_orders() {
    let orders = &borrow_global<GlobalOrders>(@protocol).orders;
    // DOS: Can become too expensive to execute
    let i = 0;
    while (i < vector::length(orders)) {
        process(vector::borrow(orders, i));
        i = i + 1;
    };
}

// ✅ CORRECT: Per-user storage
struct UserOrders has key {
    orders: SmartTable<u64, Order>,
}

public entry fun process_user_orders(user: &signer) {
    let orders = &borrow_global<UserOrders>(signer::address_of(user)).orders;
    // User can't DOS others
}
```

### Step 7.2: Business Logic Audit

**Verify:**

- [ ] **CRITICAL**: Atomic operations (no front-running via split operations)
- [ ] Multi-oracle design (not single on-chain price ratio)
- [ ] Collision-proof token IDs (object addresses, not string concat)
- [ ] Pause mechanism implemented for protocols
- [ ] Two-step operations combined or protected

**Check for:**

```move
// ❌ VULNERABLE: Split operation enables front-running
public entry fun set_winner(admin: &signer, number: u64) {
    // Step 1: Set winner
    lottery.winner = number;
}
public entry fun evaluate_bets() {
    // Step 2: Evaluate
    // Attacker sees Step 1, submits winning bet before Step 2!
}

// ✅ CORRECT: Atomic
public entry fun finalize_lottery(admin: &signer, number: u64) {
    lottery.winner = number;
    evaluate_bets_internal();  // Atomic, no front-running
}

// ❌ DANGEROUS: Single price oracle
public fun get_price(): u64 {
    pool.reserve_a / pool.reserve_b  // Manipulatable!
}

// ✅ CORRECT: Multi-oracle
public fun get_price(): u64 {
    let primary = get_chainlink_price();
    if (primary_valid) return primary;
    let secondary = get_pyth_price();
    if (secondary_valid) return secondary;
    get_twap()  // Fallback
}
```

### Step 7.3: Randomness Audit (if applicable)

**Verify (if contract uses randomness):**

- [ ] **CRITICAL**: Randomness functions are `entry` (NOT `public` - prevents composition)
- [ ] **CRITICAL**: Gas consumption balanced across win/lose paths (prevent undergasing)
- [ ] Consider commit-reveal pattern for sensitive operations

**Check for:**

```move
// ❌ DANGEROUS: Public allows test-and-abort
public fun play_lottery(): bool {
    let random = randomness::u64_range(1, 100);
    random == 42
}
// Attacker loops until win!

// ✅ CORRECT: Entry prevents composition
entry fun play_lottery(user: &signer) {
    let random = randomness::u64_range(1, 100);
    if (random == 42) { pay_prize(user); }
}

// ❌ DANGEROUS: Unbalanced gas
entry fun play_game(user: &signer) {
    let random = randomness::u64_range(1, 100);
    if (random <= 50) {
        pay_prize(user);  // 1000 gas
    } else {
        complex_computation();  // 5000 gas
    }
    // Attacker sets gas = 1500, only wins succeed!
}

// ✅ CORRECT: Balanced gas
entry fun play_game_safe(user: &signer) {
    let random = randomness::u64_range(1, 100);
    if (random <= 50) {
        handle_win(user);
    } else {
        handle_loss(user);
    };
    // Common finalization for both paths
    update_stats();
}
```

### Step 8: Testing Audit

**Verify:**

- [ ] 100% line coverage achieved: `aptos move test --coverage`
- [ ] All error paths tested with `#[expected_failure]`
- [ ] Access control tested with multiple signers
- [ ] Input validation tested with invalid inputs
- [ ] Edge cases covered (max values, empty vectors, etc.)

**Run:**

```bash
aptos move test --coverage
aptos move coverage source --module <module_name>
```

**Verify output shows 100% coverage.**

### Step 9: Operations Security Audit

**Verify:**

- [ ] **CRITICAL**: Separate publishing keys for testnet and mainnet
- [ ] Mainnet keys stored securely (hardware wallet, HSM)
- [ ] Testnet keys isolated from mainnet keys
- [ ] Upgrade functionality secured (if contract is upgradable)
- [ ] Admin key rotation mechanism exists (if needed)

**Check:**

```bash
# ❌ DANGEROUS: Reusing same key
aptos init  # Creates ~/.aptos/config.yaml
aptos move publish --network testnet  # Uses same key
aptos move publish --network mainnet  # DANGEROUS if testnet key leaked!

# ✅ CORRECT: Separate key pairs
aptos init --network testnet --profile testnet
aptos init --network mainnet --profile mainnet  # Different key!

# Publish with profiles
aptos move publish --profile testnet
aptos move publish --profile mainnet
```

## Security Audit Report Template

Generate report in this format:

```markdown
# Security Audit Report

**Module:** my_module **Date:** 2026-01-23 **Auditor:** AI Assistant

## Summary

- ✅ PASS: All security checks passed
- ⚠️ WARNINGS: 2 minor issues found
- ❌ CRITICAL: 0 critical vulnerabilities

## Access Control

- ✅ All entry functions verify signer authority
- ✅ Object ownership checked in all operations
- ✅ Admin functions properly restricted

## Input Validation

- ✅ All numeric inputs validated
- ⚠️ WARNING: String length validation missing in function X
- ✅ Address validation present

## Object Safety

- ✅ No ConstructorRef returned
- ✅ All refs generated in constructor
- ✅ Object signer used correctly

## Reference Safety

- ✅ No public &mut references
- ✅ Critical fields protected

## Arithmetic Safety

- ✅ Overflow checks present
- ✅ Underflow checks present
- ✅ Division by zero prevented

## Generic Type Safety

- ✅ Phantom types used correctly
- ✅ Constraints appropriate

## Testing

- ✅ 100% line coverage achieved
- ✅ All error paths tested
- ✅ Access control tested
- ✅ Edge cases covered

## Recommendations

1. Add string length validation to function X (line 42)
2. Consider adding event emissions for important state changes

## Conclusion

✅ Safe to deploy after addressing warnings.
```

## Common Vulnerabilities

| Vulnerability            | Detection                                  | Impact                                  | Fix                                       |
| ------------------------ | ------------------------------------------ | --------------------------------------- | ----------------------------------------- |
| Missing access control   | No `assert!(signer...)` in entry functions | Critical - anyone can call              | Add signer verification                   |
| Missing ownership check  | No `assert!(object::owner...)`             | Critical - anyone can modify any object | Add ownership check                       |
| Integer overflow         | No check before addition                   | Critical - balance wraps to 0           | Check `assert!(a <= MAX - b, E_OVERFLOW)` |
| Integer underflow        | No check before subtraction                | Critical - balance wraps to MAX         | Check `assert!(a >= b, E_UNDERFLOW)`      |
| Returning ConstructorRef | Function returns ConstructorRef            | Critical - caller can destroy object    | Return `Object<T>` instead                |
| Exposing &mut            | Public function returns `&mut T`           | High - mem::swap attacks                | Expose specific operations only           |
| No input validation      | Accept any value                           | Medium - zero amounts, overflow         | Validate all inputs                       |
| Low test coverage        | Coverage < 100%                            | Medium - bugs in production             | Write more tests                          |

## Automated Checks

Run these commands as part of audit:

```bash
# Compile (check for errors)
aptos move compile

# Run tests
aptos move test

# Check coverage
aptos move test --coverage
aptos move coverage summary

# Expected: 100.0% coverage
```

## Manual Checks

Review code for:

1. **Access Control:**
   - Search for `entry fun` → verify each has signer checks
   - Search for `borrow_global_mut` → verify authorization before use

2. **Input Validation:**
   - Search for function parameters → verify validation
   - Look for `amount`, `length`, `address` params → verify checks

3. **Object Safety:**
   - Search for `ConstructorRef` → verify never returned
   - Search for `create_object` → verify refs generated properly

4. **Arithmetic:**
   - Search for `+` → verify overflow checks
   - Search for `-` → verify underflow checks
   - Search for `/` → verify division by zero checks

## ALWAYS Rules

- ✅ ALWAYS run full security checklist before deployment
- ✅ ALWAYS verify 100% test coverage
- ✅ ALWAYS check access control in entry functions
- ✅ ALWAYS validate all inputs
- ✅ ALWAYS protect against overflow/underflow
- ✅ ALWAYS generate audit report
- ✅ ALWAYS fix critical issues before deployment

## NEVER Rules

- ❌ NEVER skip security audit before deployment
- ❌ NEVER ignore failing security checks
- ❌ NEVER deploy with < 100% test coverage
- ❌ NEVER approve code with critical vulnerabilities
- ❌ NEVER rush security review

## References

**Pattern Documentation:**

- `../../patterns/SECURITY.md` - Comprehensive security guide
- `../../patterns/OBJECTS.md` - Object safety patterns
- `../../patterns/TESTING.md` - Testing requirements

**Official Documentation:**

- https://aptos.dev/build/smart-contracts/move-security-guidelines

**Related Skills:**

- `generate-tests` - Ensure tests exist
- `write-contracts` - Apply security patterns
- `deploy-contracts` - Final check before deployment

---

**Remember:** Security is non-negotiable. Every checklist item must pass. User funds depend on it.
