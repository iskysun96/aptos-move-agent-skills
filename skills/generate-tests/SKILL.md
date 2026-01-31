---
name: generate-tests
description:
  Generate comprehensive unit tests for Aptos Move V2 contracts with 100% coverage. Use when "write tests", "test
  contract", "add test coverage", or AUTOMATICALLY after writing any contract.
---

# Generate Tests Skill

## Overview

This skill generates comprehensive test suites for Move contracts with **100% line coverage** requirement. Tests verify:

- ✅ Happy paths (functionality works)
- ✅ Access control (unauthorized users blocked)
- ✅ Input validation (invalid inputs rejected)
- ✅ Edge cases (boundaries, limits, empty states)
- ✅ **Security patterns** (arithmetic safety, storage scoping, reference safety, business logic) ⭐ CRITICAL

**Critical Rule:** NEVER deploy without 100% test coverage and comprehensive security tests.

## Core Workflow

### Step 1: Create Test Module

```move
#[test_only]
module my_addr::my_module_tests {
    use my_addr::my_module::{Self, MyObject};
    use aptos_framework::object::{Self, Object};
    use std::string;
    use std::signer;

    // Test constants
    const ADMIN_ADDR: address = @0x100;
    const USER_ADDR: address = @0x200;
    const ATTACKER_ADDR: address = @0x300;

    // ========== Setup Helpers ==========
    // (Reusable setup functions)

    // ========== Happy Path Tests ==========
    // (Basic functionality)

    // ========== Access Control Tests ==========
    // (Unauthorized access blocked)

    // ========== Input Validation Tests ==========
    // (Invalid inputs rejected)

    // ========== Edge Case Tests ==========
    // (Boundaries and limits)
}
```

### Step 2: Write Happy Path Tests

**Test basic functionality works correctly:**

```move
#[test(creator = @0x1)]
public fun test_create_object_succeeds(creator: &signer) {
    // Execute
    let obj = my_module::create_my_object(
        creator,
        string::utf8(b"Test Object")
    );

    // Verify
    assert!(object::owner(obj) == signer::address_of(creator), 0);
}

#[test(owner = @0x1)]
public fun test_update_object_succeeds(owner: &signer) {
    // Setup
    let obj = my_module::create_my_object(owner, string::utf8(b"Old Name"));

    // Execute
    let new_name = string::utf8(b"New Name");
    my_module::update_object(owner, obj, new_name);

    // Verify (if you have view functions)
    // assert!(my_module::get_object_name(obj) == new_name, 0);
}

#[test(owner = @0x1, recipient = @0x2)]
public fun test_transfer_object_succeeds(
    owner: &signer,
    recipient: &signer
) {
    let recipient_addr = signer::address_of(recipient);

    // Setup
    let obj = my_module::create_my_object(owner, string::utf8(b"Object"));
    assert!(object::owner(obj) == signer::address_of(owner), 0);

    // Execute
    my_module::transfer_object(owner, obj, recipient_addr);

    // Verify
    assert!(object::owner(obj) == recipient_addr, 1);
}
```

### Step 2.5: Testing Time-Based Logic ⭐ NEW - For Contracts with Time Dependencies

**For contracts with time-dependent logic (staking, vesting, voting periods, timelocks):**

```move
use aptos_framework::timestamp;

#[test(framework = @0x1)]
fun test_timestamp_initialization(framework: &signer) {
    // ALWAYS: Initialize timestamp in time-based tests
    timestamp::set_time_has_started_for_testing(framework);

    // Now timestamp functions work
    let current_time = timestamp::now_seconds();
    assert!(current_time == 0, 0);
}

#[test(framework = @0x1, user = @0x200)]
fun test_voting_period(framework: &signer, user: &signer) {
    // Initialize timestamp
    timestamp::set_time_has_started_for_testing(framework);

    // Create proposal with 7-day voting period
    let proposal = create_proposal(user, b"Proposal 1", 7 * 86400);

    // Fast forward 8 days
    timestamp::fast_forward_seconds(8 * 86400);

    // Now can execute
    execute_proposal(user, proposal);
}

#[test(framework = @0x1, user = @0x200)]
fun test_staking_rewards(framework: &signer, user: &signer) {
    timestamp::set_time_has_started_for_testing(framework);

    // Stake at time 0
    stake(user, 1000);

    // Fast forward 1 year
    timestamp::fast_forward_seconds(365 * 86400);

    // Claim and verify rewards
    claim_rewards(user);
    let balance = get_balance(user);
    assert!(balance >= 1050, 0); // At least 5% APY
}
```

**CRITICAL - Common Pitfalls:**

❌ **NEVER use** `aptos_governance::get_signer_testnet_sig()` (doesn't exist, causes compilation errors)

✅ **ALWAYS use** `#[test(framework = @0x1)]` attribute and pass framework signer to
`set_time_has_started_for_testing()`

❌ **NEVER assume** time starts at 0 without initialization

✅ **ALWAYS call** `set_time_has_started_for_testing()` first in time-based tests

### Step 3: Write Access Control Tests

**Test unauthorized access is blocked:**

```move
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = my_module::E_NOT_OWNER)]
public fun test_non_owner_cannot_update(
    owner: &signer,
    attacker: &signer
) {
    let obj = my_module::create_my_object(owner, string::utf8(b"Object"));

    // Attacker tries to update (should abort)
    my_module::update_object(attacker, obj, string::utf8(b"Hacked"));
}

#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = my_module::E_NOT_OWNER)]
public fun test_non_owner_cannot_transfer(
    owner: &signer,
    attacker: &signer
) {
    let obj = my_module::create_my_object(owner, string::utf8(b"Object"));

    // Attacker tries to transfer (should abort)
    my_module::transfer_object(attacker, obj, @0x3);
}

#[test(admin = @0x1, user = @0x2)]
#[expected_failure(abort_code = my_module::E_NOT_ADMIN)]
public fun test_non_admin_cannot_configure(
    admin: &signer,
    user: &signer
) {
    my_module::init_module(admin);

    // Regular user tries admin function (should abort)
    my_module::update_config(user, 100);
}
```

### Step 4: Write Input Validation Tests

**Test invalid inputs are rejected:**

```move
#[test(user = @0x1)]
#[expected_failure(abort_code = my_module::E_ZERO_AMOUNT)]
public fun test_zero_amount_rejected(user: &signer) {
    my_module::deposit(user, 0); // Should abort
}

#[test(user = @0x1)]
#[expected_failure(abort_code = my_module::E_AMOUNT_TOO_HIGH)]
public fun test_excessive_amount_rejected(user: &signer) {
    my_module::deposit(user, my_module::MAX_DEPOSIT_AMOUNT + 1); // Should abort
}

#[test(owner = @0x1)]
#[expected_failure(abort_code = my_module::E_EMPTY_NAME)]
public fun test_empty_string_rejected(owner: &signer) {
    let obj = my_module::create_my_object(owner, string::utf8(b"Initial"));
    my_module::update_object(owner, obj, string::utf8(b"")); // Empty - should abort
}

#[test(owner = @0x1)]
#[expected_failure(abort_code = my_module::E_NAME_TOO_LONG)]
public fun test_string_too_long_rejected(owner: &signer) {
    let obj = my_module::create_my_object(owner, string::utf8(b"Initial"));

    // String exceeding MAX_NAME_LENGTH
    let long_name = string::utf8(b"This is an extremely long name that exceeds the maximum allowed length");

    my_module::update_object(owner, obj, long_name); // Should abort
}

#[test(owner = @0x1)]
#[expected_failure(abort_code = my_module::E_ZERO_ADDRESS)]
public fun test_zero_address_rejected(owner: &signer) {
    let obj = my_module::create_my_object(owner, string::utf8(b"Object"));
    my_module::transfer_object(owner, obj, @0x0); // Should abort
}
```

### Step 5: Write Edge Case Tests

**Test boundary conditions:**

```move
#[test(user = @0x1)]
public fun test_max_amount_allowed(user: &signer) {
    my_module::init_account(user);

    // Exactly MAX_DEPOSIT_AMOUNT should work
    my_module::deposit(user, my_module::MAX_DEPOSIT_AMOUNT);

    // Verify
    assert!(my_module::get_balance(signer::address_of(user)) == my_module::MAX_DEPOSIT_AMOUNT, 0);
}

#[test(user = @0x1)]
public fun test_max_name_length_allowed(user: &signer) {
    // Create string exactly MAX_NAME_LENGTH long
    let max_name = string::utf8(b"12345678901234567890123456789012"); // 32 chars if MAX = 32

    // Should succeed
    let obj = my_module::create_my_object(user, max_name);
}

#[test(user = @0x1)]
public fun test_empty_collection_operations(user: &signer) {
    let collection = my_module::create_collection(user, string::utf8(b"Collection"));

    // Should handle empty collection gracefully
    assert!(my_module::get_collection_size(collection) == 0, 0);
}
```

### Step 6: Write Security Tests ⭐ CRITICAL

**Test security-critical patterns from [SECURITY.md](../../patterns/SECURITY.md):**

#### 6.1 Arithmetic Safety Tests

**Division Precision Loss - CRITICAL:**

```move
#[test(user = @0x1)]
#[expected_failure(abort_code = my_module::E_AMOUNT_TOO_SMALL)]
public fun test_amount_below_minimum_rejected(user: &signer) {
    // Amount too small causes fee to round to zero
    my_module::process_order(user, 100); // MIN_ORDER_SIZE is 1000
}

#[test(user = @0x1)]
public fun test_fee_calculation_non_zero(user: &signer) {
    // Test that fee is actually calculated for valid amounts
    let fee = my_module::calculate_fee(1000); // Minimum valid amount
    assert!(fee > 0, 0); // Fee must be non-zero
}

#[test(user = @0x1)]
public fun test_minimum_threshold_enforced(user: &signer) {
    // Test that minimum threshold prevents precision loss
    let amount = my_module::MIN_ORDER_SIZE;
    let fee = my_module::calculate_fee(amount);
    assert!(fee > 0, 0);
}
```

**Left Shift Overflow - CRITICAL:**

```move
#[test]
#[expected_failure(abort_code = my_module::E_OVERFLOW)]
public fun test_shift_amount_overflow_rejected() {
    // Left shift >= 64 must be rejected
    my_module::calculate_power_of_two(64); // Should abort
}

#[test]
public fun test_max_shift_amount_allowed() {
    // Maximum safe shift should work
    let result = my_module::calculate_power_of_two(63);
    // Verify result is correct
}

#[test]
#[expected_failure(abort_code = my_module::E_OVERFLOW)]
public fun test_shift_with_u8_overflow() {
    // If using u8 shift amounts, test overflow
    my_module::shift_operation(255); // u8 max
}
```

#### 6.2 Global Storage Scoping Tests - CRITICAL

**Test functions ONLY modify signer's own storage:**

```move
#[test(user1 = @0x1, user2 = @0x2)]
public fun test_update_only_affects_own_account(
    user1: &signer,
    user2: &signer
) {
    // Setup both accounts
    my_module::init_account(user1);
    my_module::init_account(user2);

    my_module::set_balance(user1, 100);
    my_module::set_balance(user2, 200);

    // User1 updates their balance
    my_module::update_balance(user1, 50);

    // Verify user1's balance changed, user2's did NOT
    assert!(my_module::get_balance(signer::address_of(user1)) == 150, 0);
    assert!(my_module::get_balance(signer::address_of(user2)) == 200, 1); // Unchanged
}

// ❌ BAD CONTRACT: If function accepts arbitrary address parameter
// #[test(user = @0x1)]
// #[expected_failure]  // This test would FAIL if contract is vulnerable
// public fun test_cannot_modify_other_account(user: &signer) {
//     // If contract accepts target_addr parameter, this is a vulnerability
//     my_module::update_balance_wrong(user, @0x999, 1000); // Should NOT be possible
// }
```

#### 6.3 Resource Management Tests

**Unbounded Iteration Prevention:**

```move
#[test(admin = @0x1)]
public fun test_user_data_stored_in_user_account(admin: &signer) {
    // Verify user data is stored in user's account, NOT global vector
    my_module::init_module(admin);

    // Check implementation doesn't use global vector
    // (This test validates architecture, not runtime behavior)
}

#[test(user = @0x1)]
public fun test_scalable_per_user_storage(user: &signer) {
    // Test that operations scale per-user (no iteration over all users)
    my_module::init_account(user);
    my_module::add_item(user, string::utf8(b"Item"));

    // Operation should be O(1), not O(n) where n = total users
    // Gas cost should be constant regardless of total system users
}
```

#### 6.4 Generic Type Validation Tests

**Flash Loan Protection:**

```move
use aptos_framework::coin::{Self, Coin};
use aptos_framework::aptos_coin::AptosCoin;

#[test(user = @0x1)]
public fun test_flash_loan_requires_matching_type(user: &signer) {
    // Borrow AptosCoin
    let receipt = my_module::flash_borrow<AptosCoin>(user, 1000);

    // Must repay with SAME type (Receipt<phantom AptosCoin> enforces this)
    let coins = coin::withdraw<AptosCoin>(user, 1000);
    my_module::flash_repay<AptosCoin>(receipt, coins); // Type must match!
}

// This should NOT compile if types don't match:
// #[test(user = @0x1)]
// public fun test_flash_loan_type_mismatch(user: &signer) {
//     let receipt = my_module::flash_borrow<AptosCoin>(user, 1000);
//     let fake_coins = coin::withdraw<FakeCoin>(user, 1000);
//     my_module::flash_repay<FakeCoin>(receipt, fake_coins); // Compile error!
// }
```

#### 6.5 Reference Safety Tests

**Callback Validation:**

```move
#[test(owner = @0x1)]
public fun test_invariants_preserved_after_callback(owner: &signer) {
    // Setup
    let obj = my_module::create_object(owner, 100);

    // Check invariant before callback
    assert!(my_module::get_value(obj) == 100, 0);

    // Execute function that calls external code with &mut ref
    my_module::process_with_callback(owner, obj);

    // CRITICAL: Re-verify invariants after callback
    // Malicious callback could have violated invariants
    assert!(my_module::get_value(obj) >= 0, 1); // Still valid
    assert!(my_module::is_initialized(obj), 2); // Still initialized
}
```

#### 6.6 Business Logic Tests

**Front-Running Prevention (Atomic Operations):**

```move
#[test(user = @0x1)]
public fun test_set_and_evaluate_atomic(user: &signer) {
    // Operation should be atomic (not split into set + evaluate)
    my_module::set_and_evaluate_price(user, 100);

    // Verify both operations happened together
    // (No way for attacker to front-run between set and evaluate)
}

#[test(attacker = @0x2, victim = @0x3)]
public fun test_cannot_front_run_price_update(
    attacker: &signer,
    victim: &signer
) {
    // Setup victim's operation
    my_module::set_and_evaluate_price(victim, 100);

    // Attacker cannot observe price before evaluation
    // (Operations are atomic in same transaction)
}
```

**Token ID Collision Prevention:**

```move
#[test(user = @0x1)]
public fun test_token_ids_use_object_addresses(user: &signer) {
    // Create two tokens with same metadata
    let token1 = my_module::create_token(user, string::utf8(b"Token"), 1);
    let token2 = my_module::create_token(user, string::utf8(b"Token"), 1);

    // IDs must be different (object addresses are unique)
    assert!(object::object_address(&token1) != object::object_address(&token2), 0);
}

// ❌ BAD: String concatenation causes collisions
// #[test(user = @0x1)]
// public fun test_string_concat_collision() {
//     // "A" + "BC" = "ABC"
//     // "AB" + "C" = "ABC"  <-- COLLISION!
// }
```

#### 6.7 Randomness Security Tests (if applicable)

**Entry Function Protection:**

```move
// ✅ CORRECT: Randomness function is `entry` (not public)
#[test(user = @0x1)]
public fun test_random_mint_is_entry_function(user: &signer) {
    // Can call directly
    my_module::random_mint(user);

    // But CANNOT be composed (prevents test-and-abort attacks)
    // my_module::try_random_mint(user); // Would not compile
}
```

**Gas Balance Testing:**

> **Note:** Aptos Move unit tests do not currently provide a built-in `estimate_gas` helper. The example below is
> **conceptual pseudo-code** showing what you want to verify. In practice, compare compiled bytecode and gas schedules,
> or measure gas usage on a localnet/testnet by sending transactions for each path and recording the gas used from the
> node/CLI output.

```move
// PSEUDO-CODE ONLY — not executable as-is.
// Goal: ensure "win" and "lose" paths of randomness have similar gas usage
// so an attacker cannot under-gas one branch to bias the outcome.
//
// Recommended approach:
// 1. Identify the "win" and "lose" code paths (e.g., helper entry functions).
// 2. Compile the module and inspect bytecode / use profiling tools, OR
// 3. Execute each path on a localnet/testnet and record gas used per tx.
// 4. Assert the absolute difference is below your chosen threshold.
//
// Example structure of the check (conceptual):
#[test(user = @0x1)]
public fun test_gas_balanced_across_outcomes(user: &signer) {
    // let gas_win = <measured gas for winning-path transaction>;
    // let gas_lose = <measured gas for losing-path transaction>;
    //
    // let diff = if (gas_win > gas_lose) { gas_win - gas_lose } else { gas_lose - gas_win };
    // assert!(diff < 1000, 0); // Within 1000 gas units (tune threshold as needed)
}
```

### Step 7: Verify Coverage

**Run tests with coverage:**

```bash
# Run all tests
aptos move test

# Run with coverage
aptos move test --coverage

# Generate detailed coverage report
aptos move coverage source --module <module_name>

# Verify 100% coverage
aptos move coverage summary
```

**Coverage report example:**

```
module: my_module
coverage: 100.0% (150/150 lines covered)
```

**If coverage < 100%:**

1. Check uncovered lines in report
2. Write tests for missing paths
3. Repeat until 100%

## Test Template Structure

```move
#[test_only]
module my_addr::module_tests {
    use my_addr::module::{Self, Type};

    // ========== Setup Helpers ==========

    fun setup_default(): Object<Type> {
        // Common setup code
    }

    // ========== Happy Path Tests ==========

    #[test(user = @0x1)]
    public fun test_basic_operation_succeeds(user: &signer) {
        // Test happy path
    }

    // ========== Access Control Tests ==========

    #[test(owner = @0x1, attacker = @0x2)]
    #[expected_failure(abort_code = E_NOT_OWNER)]
    public fun test_unauthorized_access_fails(
        owner: &signer,
        attacker: &signer
    ) {
        // Test access control
    }

    // ========== Input Validation Tests ==========

    #[test(user = @0x1)]
    #[expected_failure(abort_code = E_INVALID_INPUT)]
    public fun test_invalid_input_rejected(user: &signer) {
        // Test input validation
    }

    // ========== Edge Case Tests ==========

    #[test(user = @0x1)]
    public fun test_boundary_condition(user: &signer) {
        // Test edge cases
    }

    // ========== Security Tests ⭐ CRITICAL ==========

    #[test(user = @0x1)]
    #[expected_failure(abort_code = E_AMOUNT_TOO_SMALL)]
    public fun test_division_precision_loss_prevented(user: &signer) {
        // Test minimum threshold enforced
    }

    #[test]
    #[expected_failure(abort_code = E_OVERFLOW)]
    public fun test_left_shift_overflow_rejected() {
        // Test shift validation
    }

    #[test(user1 = @0x1, user2 = @0x2)]
    public fun test_global_storage_scoping(user1: &signer, user2: &signer) {
        // Test cross-account isolation
    }

    #[test(user = @0x1)]
    public fun test_no_unbounded_iteration(user: &signer) {
        // Test per-user storage architecture
    }

    #[test(user = @0x1)]
    public fun test_invariants_after_callback(user: &signer) {
        // Test reference safety
    }

    #[test(user = @0x1)]
    public fun test_atomic_operations(user: &signer) {
        // Test front-running prevention
    }
}
```

## Testing Checklist

For each contract, verify you have tests for:

**Happy Paths:**

- [ ] Object creation works
- [ ] State updates work
- [ ] Transfers work
- [ ] All main features work

**Access Control:**

- [ ] Non-owners cannot modify objects
- [ ] Non-admins cannot call admin functions
- [ ] Unauthorized users blocked

**Input Validation:**

- [ ] Zero amounts rejected
- [ ] Excessive amounts rejected
- [ ] Empty strings rejected
- [ ] Strings too long rejected
- [ ] Zero addresses rejected

**Edge Cases:**

- [ ] Maximum values work
- [ ] Minimum values work
- [ ] Empty states handled

**Security Tests ⭐ CRITICAL:**

- [ ] Division precision loss prevented (minimum thresholds enforced, fees > 0)
- [ ] Left shift overflow validated (shift amount < 64)
- [ ] Global storage scoped to signer (cannot modify other accounts)
- [ ] Unbounded iterations prevented (per-user storage, not global vectors)
- [ ] Generic type validation (flash loan repayment type matches)
- [ ] Reference safety (invariants preserved after callbacks)
- [ ] Front-running prevention (atomic operations tested)
- [ ] Token ID collisions prevented (object addresses used)
- [ ] Randomness security (entry functions, gas balanced) - if applicable

**Coverage:**

- [ ] 100% line coverage achieved
- [ ] All error codes tested
- [ ] All functions tested
- [ ] All security patterns tested

## ALWAYS Rules

- ✅ ALWAYS achieve 100% test coverage
- ✅ ALWAYS test error paths with `#[expected_failure(abort_code = E_CODE)]`
- ✅ ALWAYS test access control with multiple signers
- ✅ ALWAYS test input validation with invalid inputs
- ✅ ALWAYS test edge cases (boundaries, limits, empty states)
- ✅ ALWAYS use clear test names: `test_feature_scenario`
- ✅ ALWAYS verify all state changes in tests
- ✅ ALWAYS run `aptos move test --coverage` before deployment
- ✅ **ALWAYS generate minimum 10 tests:** 5 happy path, 3 error conditions, 2 access control
- ✅ **ALWAYS create all test accounts in setup function BEFORE any operations** - Never call
  `aptos_account::create_account()` mid-test if the account was already created with
  `account::create_account_for_test()`

### Security Testing ⭐ CRITICAL - See [SECURITY.md](../../patterns/SECURITY.md)

- ✅ **ALWAYS test division precision loss**: Verify minimum thresholds enforced, fees > 0
- ✅ **ALWAYS test left shift validation**: Reject shift amounts >= 64
- ✅ **ALWAYS test global storage scoping**: Multi-user tests verify isolation
- ✅ **ALWAYS test unbounded iteration prevention**: Verify per-user storage architecture
- ✅ **ALWAYS test generic type validation**: Flash loan type matching (if applicable)
- ✅ **ALWAYS test reference safety**: Verify invariants before AND after callbacks
- ✅ **ALWAYS test atomic operations**: Verify no front-running opportunities
- ✅ **ALWAYS test token ID uniqueness**: Object addresses prevent collisions
- ✅ **ALWAYS test randomness security**: Entry functions, gas balance (if applicable)

## NEVER Rules

- ❌ NEVER deploy without 100% coverage
- ❌ NEVER skip testing error paths
- ❌ NEVER skip access control tests
- ❌ NEVER use unclear test names
- ❌ NEVER batch tests without verifying each case

### Security Testing Violations ⭐ CRITICAL

- ❌ NEVER skip testing minimum thresholds (division precision loss)
- ❌ NEVER skip testing left shift validation (silent overflow)
- ❌ NEVER skip testing cross-account isolation (global storage scoping)
- ❌ NEVER skip testing scalability (unbounded iteration DOS)
- ❌ NEVER skip testing invariants after callbacks (reference safety)
- ❌ NEVER skip testing atomic operations (front-running)
- ❌ NEVER skip testing randomness functions if contract uses randomness

## References

**Pattern Documentation:**

- `../../patterns/TESTING.md` - Comprehensive testing guide
- `../../patterns/SECURITY.md` - Security testing requirements

**Official Documentation:**

- https://aptos.dev/build/smart-contracts/book/unit-testing

**Related Skills:**

- `write-contracts` - Generate code to test
- `security-audit` - Verify security after testing

---

**Remember:** 100% coverage is mandatory. Test happy paths, error paths, access control, and edge cases.
