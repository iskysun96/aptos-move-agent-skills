# Aptos Move V2 Testing Patterns

**Purpose:** Comprehensive guide to testing Aptos Move V2 smart contracts with 100% coverage.

**Target:** AI assistants generating tests for Move V2 contracts

**Critical Note:** 100% test coverage is mandatory. Never deploy without complete test suite.

---

## Testing Philosophy

**Core Principles:**

1. **100% Coverage Required**: Every line of code must be tested
2. **Test All Paths**: Happy paths AND error paths (with #[expected_failure])
3. **Test Access Control**: Verify unauthorized access is blocked
4. **Test Input Validation**: Invalid inputs must abort with correct error codes
5. **Test Edge Cases**: Boundary conditions, empty vectors, max values

**When to Write Tests:**

- ALWAYS after writing any contract code
- BEFORE deploying to any network
- After fixing bugs (add regression test)
- When refactoring (ensure behavior unchanged)

---

## Test Structure

### Basic Test Anatomy

```move
#[test_only]
module my_addr::item_tests {
    use my_addr::item::{Self, Item};
    use aptos_framework::object::{Self, Object};
    use std::string;

    /// Test happy path - basic functionality works
    #[test(creator = @0x1)]
    public fun test_create_item(creator: &signer) {
        // Setup
        let name = string::utf8(b"Test Item");

        // Execute
        let item = item::create_item(creator, name);

        // Verify
        assert!(object::owner(item) == signer::address_of(creator), 0);
    }

    /// Test error path - unauthorized access rejected
    #[test(owner = @0x1, attacker = @0x2)]
    #[expected_failure(abort_code = item::E_NOT_OWNER)]
    public fun test_unauthorized_transfer_fails(
        owner: &signer,
        attacker: &signer
    ) {
        // Setup
        let item = item::create_item(owner, string::utf8(b"Test"));

        // Execute & Verify - should abort
        item::transfer_item(attacker, item, @0x3);
    }
}
```

### Test Module Organization

```move
#[test_only]
module my_addr::marketplace_tests {
    // Organize tests by feature/function

    // ========== Setup Helpers ==========
    fun setup_marketplace(admin: &signer): Object<Marketplace> { ... }
    fun setup_item(creator: &signer): Object<Item> { ... }

    // ========== Creation Tests ==========
    #[test]
    public fun test_create_marketplace() { ... }

    #[test]
    public fun test_create_item() { ... }

    // ========== Transfer Tests ==========
    #[test]
    public fun test_successful_transfer() { ... }

    #[test]
    #[expected_failure(abort_code = E_NOT_OWNER)]
    public fun test_unauthorized_transfer_fails() { ... }

    // ========== Access Control Tests ==========
    #[test]
    #[expected_failure(abort_code = E_NOT_ADMIN)]
    public fun test_non_admin_cannot_update_config() { ... }

    // ========== Input Validation Tests ==========
    #[test]
    #[expected_failure(abort_code = E_ZERO_AMOUNT)]
    public fun test_zero_amount_rejected() { ... }

    // ========== Edge Case Tests ==========
    #[test]
    public fun test_max_amount_allowed() { ... }

    #[test]
    public fun test_empty_vector_handling() { ... }
}
```

---

## Pattern 1: Happy Path Tests

**Purpose:** Verify basic functionality works correctly

### Creating Objects

```move
#[test(creator = @0x1)]
public fun test_create_item(creator: &signer) {
    // Create item
    let name = string::utf8(b"Sword");
    let item = create_item(creator, name);

    // Verify ownership
    assert!(object::owner(item) == signer::address_of(creator), 0);

    // Verify state (if you have getter functions)
    // let item_name = get_item_name(item);
    // assert!(item_name == name, 1);
}
```

### State Mutations

```move
#[test(owner = @0x1)]
public fun test_update_item_name(owner: &signer) {
    // Setup: Create item
    let item = create_item(owner, string::utf8(b"Old Name"));

    // Execute: Update name
    let new_name = string::utf8(b"New Name");
    update_item_name(owner, item, new_name);

    // Verify: Name changed (if you have getters)
    // assert!(get_item_name(item) == new_name, 0);
}
```

### Transfers

```move
#[test(owner = @0x1, recipient = @0x2)]
public fun test_transfer_item(owner: &signer, recipient: &signer) {
    let recipient_addr = signer::address_of(recipient);

    // Setup: Create item owned by owner
    let item = create_item(owner, string::utf8(b"Item"));
    assert!(object::owner(item) == signer::address_of(owner), 0);

    // Execute: Transfer to recipient
    transfer_item(owner, item, recipient_addr);

    // Verify: Ownership changed
    assert!(object::owner(item) == recipient_addr, 1);
}
```

### Multiple Operations

```move
#[test(user = @0x1)]
public fun test_deposit_and_withdraw(user: &signer) {
    let user_addr = signer::address_of(user);

    // Setup: Initialize account
    init_account(user);

    // Test deposit
    deposit(user, 1000);
    assert!(get_balance(user_addr) == 1000, 0);

    // Test withdraw
    withdraw(user, 300);
    assert!(get_balance(user_addr) == 700, 1);

    // Test another deposit
    deposit(user, 500);
    assert!(get_balance(user_addr) == 1200, 2);
}
```

---

## Pattern 2: Access Control Tests

**Purpose:** Verify unauthorized access is blocked

### Object Ownership Tests

```move
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_non_owner_cannot_update_item(
    owner: &signer,
    attacker: &signer
) {
    // Setup: Owner creates item
    let item = create_item(owner, string::utf8(b"Item"));

    // Attempt: Attacker tries to update (should fail)
    update_item_name(attacker, item, string::utf8(b"Hacked"));
}

#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_non_owner_cannot_transfer_item(
    owner: &signer,
    attacker: &signer
) {
    let item = create_item(owner, string::utf8(b"Item"));

    // Attacker tries to transfer owner's item (should fail)
    transfer_item(attacker, item, @0x3);
}
```

### Admin/Role Tests

```move
#[test(admin = @0x1, user = @0x2)]
#[expected_failure(abort_code = E_NOT_ADMIN)]
public fun test_non_admin_cannot_update_config(
    admin: &signer,
    user: &signer
) {
    // Setup: Admin initializes marketplace
    init_marketplace(admin);

    // Attempt: Regular user tries admin function (should fail)
    update_marketplace_config(user, 100);
}

#[test(admin = @0x1, operator = @0x2, user = @0x3)]
#[expected_failure(abort_code = E_NOT_OPERATOR)]
public fun test_non_operator_cannot_pause(
    admin: &signer,
    operator: &signer,
    user: &signer
) {
    // Setup: Admin creates marketplace and adds operator
    init_marketplace(admin);
    add_operator(admin, signer::address_of(operator));

    // Attempt: Regular user tries operator function (should fail)
    pause_marketplace(user);
}
```

---

## Pattern 3: Input Validation Tests

**Purpose:** Verify invalid inputs are rejected

### Numeric Validation

```move
#[test(user = @0x1)]
#[expected_failure(abort_code = E_ZERO_AMOUNT)]
public fun test_zero_amount_rejected(user: &signer) {
    init_account(user);
    deposit(user, 0); // Should abort
}

#[test(user = @0x1)]
#[expected_failure(abort_code = E_AMOUNT_TOO_HIGH)]
public fun test_excessive_amount_rejected(user: &signer) {
    init_account(user);
    deposit(user, MAX_DEPOSIT_AMOUNT + 1); // Should abort
}

#[test(user = @0x1)]
#[expected_failure(abort_code = E_INSUFFICIENT_BALANCE)]
public fun test_insufficient_balance_rejected(user: &signer) {
    init_account(user);
    deposit(user, 100);
    withdraw(user, 200); // Should abort - trying to withdraw more than balance
}
```

### String Validation

```move
#[test(owner = @0x1)]
#[expected_failure(abort_code = E_EMPTY_NAME)]
public fun test_empty_name_rejected(owner: &signer) {
    let item = create_item(owner, string::utf8(b"Initial"));
    update_item_name(owner, item, string::utf8(b"")); // Empty string should abort
}

#[test(owner = @0x1)]
#[expected_failure(abort_code = E_NAME_TOO_LONG)]
public fun test_name_too_long_rejected(owner: &signer) {
    let item = create_item(owner, string::utf8(b"Initial"));

    // Create name longer than MAX_NAME_LENGTH
    let long_name = string::utf8(b"This is an extremely long name that exceeds the maximum allowed length for item names in this contract");

    update_item_name(owner, item, long_name); // Should abort
}
```

### Vector Validation

```move
#[test(user = @0x1)]
#[expected_failure(abort_code = E_EMPTY_VECTOR)]
public fun test_empty_vector_rejected(user: &signer) {
    let collection = create_collection(user);
    let empty_items = vector::empty<Object<Item>>();

    add_items(user, collection, empty_items); // Should abort
}

#[test(user = @0x1)]
#[expected_failure(abort_code = E_TOO_MANY_ITEMS)]
public fun test_too_many_items_rejected(user: &signer) {
    let collection = create_collection(user);

    // Create vector with MAX_ITEMS + 1 items
    let items = vector::empty<Object<Item>>();
    let i = 0;
    while (i <= MAX_ITEMS) {
        vector::push_back(&mut items, create_item(user, string::utf8(b"Item")));
        i = i + 1;
    };

    add_items(user, collection, items); // Should abort
}
```

### Address Validation

```move
#[test(owner = @0x1)]
#[expected_failure(abort_code = E_ZERO_ADDRESS)]
public fun test_zero_address_rejected(owner: &signer) {
    let item = create_item(owner, string::utf8(b"Item"));
    transfer_item(owner, item, @0x0); // Transferring to zero address should abort
}
```

---

## Pattern 4: Edge Case Tests

**Purpose:** Verify boundary conditions and special cases

### Boundary Values

```move
#[test(user = @0x1)]
public fun test_max_amount_allowed(user: &signer) {
    init_account(user);

    // Should succeed with MAX_DEPOSIT_AMOUNT
    deposit(user, MAX_DEPOSIT_AMOUNT);

    assert!(get_balance(signer::address_of(user)) == MAX_DEPOSIT_AMOUNT, 0);
}

#[test(user = @0x1)]
public fun test_single_item_allowed(user: &signer) {
    let collection = create_collection(user);

    // Single item should work
    let items = vector::empty<Object<Item>>();
    vector::push_back(&mut items, create_item(user, string::utf8(b"Item")));

    add_items(user, collection, items);
}

#[test(user = @0x1)]
public fun test_max_items_allowed(user: &signer) {
    let collection = create_collection(user);

    // Exactly MAX_ITEMS should work
    let items = vector::empty<Object<Item>>();
    let i = 0;
    while (i < MAX_ITEMS) {
        vector::push_back(&mut items, create_item(user, string::utf8(b"Item")));
        i = i + 1;
    };

    add_items(user, collection, items); // Should succeed
}
```

### Empty States

```move
#[test(user = @0x1)]
public fun test_operations_on_empty_collection(user: &signer) {
    let collection = create_collection(user);

    // Verify collection starts empty
    assert!(get_collection_size(collection) == 0, 0);

    // Operations on empty collection should handle gracefully
    // (depending on your design - might return empty, might abort, etc.)
}
```

### Sequential Operations

```move
#[test(user = @0x1)]
public fun test_multiple_transfers(user: &signer) {
    let item = create_item(user, string::utf8(b"Item"));

    // Transfer through multiple owners
    transfer_item(user, item, @0x2);
    assert!(object::owner(item) == @0x2, 0);

    // Note: Would need @0x2 signer to continue, or use test helpers
}

#[test(user = @0x1)]
public fun test_multiple_updates(user: &signer) {
    let item = create_item(user, string::utf8(b"Name1"));

    update_item_name(user, item, string::utf8(b"Name2"));
    update_item_name(user, item, string::utf8(b"Name3"));
    update_item_name(user, item, string::utf8(b"Name4"));

    // Should handle multiple updates without issues
}
```

---

## Pattern 5: Test Helpers

**Purpose:** Reusable setup code for tests

### Setup Functions

```move
#[test_only]
module my_addr::test_helpers {
    use my_addr::marketplace::{Self, Marketplace};
    use my_addr::item::{Self, Item};
    use aptos_framework::object::Object;
    use std::string;

    /// Create marketplace with default settings
    public fun setup_marketplace(admin: &signer): Object<Marketplace> {
        marketplace::init_marketplace(admin);
        marketplace::get_marketplace(signer::address_of(admin))
    }

    /// Create item with default name
    public fun create_test_item(creator: &signer): Object<Item> {
        item::create_item(creator, string::utf8(b"Test Item"))
    }

    /// Create item with specific name
    public fun create_named_item(creator: &signer, name: vector<u8>): Object<Item> {
        item::create_item(creator, string::utf8(name))
    }

    /// Setup account with initial balance
    public fun setup_account_with_balance(user: &signer, balance: u64) {
        marketplace::init_account(user);
        marketplace::deposit(user, balance);
    }
}
```

### Using Test Helpers

```move
#[test(user = @0x1)]
public fun test_with_helper(user: &signer) {
    use my_addr::test_helpers;

    // Setup using helper
    let item = test_helpers::create_test_item(user);
    test_helpers::setup_account_with_balance(user, 1000);

    // Test logic
    marketplace::purchase_item(user, item, 500);

    // Verify
    assert!(object::owner(item) == signer::address_of(user), 0);
}
```

---

## Pattern 6: Coverage Verification

### Running Tests with Coverage

```bash
# Run all tests
aptos move test

# Run with coverage
aptos move test --coverage

# Generate coverage report
aptos move coverage source --module <module_name>

# Generate coverage summary
aptos move coverage summary
```

### Coverage Report Analysis

```move
// Example coverage report
// module: my_module
// coverage: 95.5% (86/90 lines covered)
//
// Uncovered lines:
// - my_module.move:45 (error path not tested)
// - my_module.move:67 (edge case not tested)
// - my_module.move:89 (helper function not called)
```

**Action:** Write tests to cover missing lines until 100%.

### Coverage Requirements

**Target: 100% line coverage**

Must cover:

- ✅ All happy paths (main functionality)
- ✅ All error paths (with #[expected_failure])
- ✅ All branches (if/else, match arms)
- ✅ All public functions
- ✅ All entry functions

Can skip:

- Test-only functions (marked #[test_only])
- Helper functions if they're tested indirectly

---

## Pattern 7: Testing Different Scenarios

### Multi-User Tests

```move
#[test(user1 = @0x1, user2 = @0x2, user3 = @0x3)]
public fun test_multiple_users_interact(
    user1: &signer,
    user2: &signer,
    user3: &signer
) {
    // User1 creates marketplace
    init_marketplace(user1);

    // User2 creates item
    let item = create_item(user2, string::utf8(b"Item"));

    // User2 lists item on User1's marketplace
    list_item(user2, item, 1000);

    // User3 purchases item
    purchase_item(user3, item);

    // Verify: User3 owns item, User2 received payment
    assert!(object::owner(item) == signer::address_of(user3), 0);
}
```

### State Transitions

```move
#[test(admin = @0x1, user = @0x2)]
public fun test_marketplace_pause_unpause(
    admin: &signer,
    user: &signer
) {
    // Setup
    init_marketplace(admin);
    let item = create_item(user, string::utf8(b"Item"));
    list_item(user, item, 1000);

    // Pause marketplace
    pause_marketplace(admin);

    // Operations should fail when paused
    // (test this with #[expected_failure] in separate test)

    // Unpause marketplace
    unpause_marketplace(admin);

    // Operations should work again
    purchase_item(user, item); // Should succeed
}
```

### Complex Workflows

```move
#[test(creator = @0x1, buyer = @0x2)]
public fun test_complete_purchase_workflow(
    creator: &signer,
    buyer: &signer
) {
    // 1. Setup accounts
    setup_account_with_balance(creator, 0);
    setup_account_with_balance(buyer, 10000);

    // 2. Creator creates and lists item
    let item = create_item(creator, string::utf8(b"Rare Sword"));
    let price = 5000;
    list_item(creator, item, price);

    // 3. Buyer purchases item
    purchase_item(buyer, item);

    // 4. Verify complete state
    assert!(object::owner(item) == signer::address_of(buyer), 0);
    assert!(get_balance(signer::address_of(creator)) == price, 1);
    assert!(get_balance(signer::address_of(buyer)) == 10000 - price, 2);
}
```

---

## Testing Checklist

For each contract, verify you have tests for:

**Happy Paths:**

- [ ] Object creation works correctly
- [ ] State updates work correctly
- [ ] Transfers work correctly
- [ ] All main features work as expected

**Access Control:**

- [ ] Non-owners cannot modify objects
- [ ] Non-admins cannot call admin functions
- [ ] Unauthorized users are blocked

**Input Validation:**

- [ ] Zero amounts rejected
- [ ] Excessive amounts rejected
- [ ] Empty strings rejected
- [ ] Strings too long rejected
- [ ] Empty vectors rejected
- [ ] Vectors too large rejected
- [ ] Zero addresses rejected

**Edge Cases:**

- [ ] Maximum allowed values work
- [ ] Minimum allowed values work
- [ ] Boundary conditions handled
- [ ] Empty states handled

**Error Paths:**

- [ ] All error codes tested with #[expected_failure]
- [ ] Abort messages are clear

**Coverage:**

- [ ] 100% line coverage achieved
- [ ] All branches covered
- [ ] All functions tested

---

## Common Testing Mistakes

| Mistake                                | Impact                       | Solution                                                      |
| -------------------------------------- | ---------------------------- | ------------------------------------------------------------- |
| Not testing error paths                | Bugs in error handling       | Add #[expected_failure] tests                                 |
| Testing only happy paths               | Miss edge cases              | Test boundaries and invalid inputs                            |
| Not testing access control             | Security vulnerabilities     | Test unauthorized access attempts                             |
| Low coverage                           | Untested code has bugs       | Achieve 100% coverage                                         |
| Unclear test names                     | Hard to understand failures  | Use descriptive names like `test_unauthorized_transfer_fails` |
| No test organization                   | Hard to maintain             | Group tests by feature                                        |
| Duplicate setup code                   | Hard to maintain             | Use test helper functions                                     |
| Not verifying all state changes        | Incomplete tests             | Assert all relevant state after operations                    |
| Accessing struct fields across modules | Compilation error            | Add public view accessor functions (see Pattern 8)            |
| Wrong error expectation after escrow   | Test fails unexpectedly      | Track ownership changes - after escrow, seller no longer owns |
| Unnecessary `acquires` annotations     | Compiler warnings/errors     | Only list resources YOUR code borrows, not framework calls    |

---

## Pattern 8: Cross-Module Testing Patterns

**Purpose:** Handle common issues when test modules access main contract data.

### Struct Field Visibility

**Problem:** Test modules cannot directly access struct fields from other modules.

```move
// ❌ WRONG - This will NOT compile
#[test(seller = @0x1)]
public fun test_listing_price(seller: &signer) {
    let nft = mint_test_nft(seller);
    list_nft(seller, nft, 1000);

    let listing = marketplace::get_listing(nft_addr);
    // ERROR: Invalid operation: access of field 'price' on type
    // 'marketplace::ListingInfo' can only be done within the defining module
    assert!(listing.price == 1000, 0);
}
```

**Solution:** Add public view accessor functions in the main module.

```move
// In main module (marketplace.move)
/// Get listing details (seller, price, listed_at)
#[view]
public fun get_listing_details(nft_addr: address): (address, u64, u64) acquires Listings {
    let marketplace_addr = get_marketplace_address();
    let listings = borrow_global<Listings>(marketplace_addr);
    assert!(table::contains(&listings.items, nft_addr), E_NOT_LISTED);
    let listing = table::borrow(&listings.items, nft_addr);
    (listing.seller, listing.price, listing.listed_at)
}

// In test module
#[test(seller = @0x1)]
public fun test_listing_price(seller: &signer) {
    let nft = mint_test_nft(seller);
    list_nft(seller, nft, 1000);

    // ✅ CORRECT - Use accessor function
    let (listing_seller, listing_price, _) = marketplace::get_listing_details(nft_addr);
    assert!(listing_price == 1000, 0);
}
```

### Testing Escrow Patterns

**Problem:** After escrowing an asset, ownership changes. Tests expecting "already listed" errors may fail
because the original owner no longer owns the asset.

```move
// ❌ WRONG expectation
#[test(seller = @0x1)]
#[expected_failure(abort_code = marketplace::E_ALREADY_LISTED)]
public fun test_cannot_list_twice(seller: &signer) {
    let nft = mint_test_nft(seller);
    list_nft(seller, nft, 1000);  // NFT transfers to marketplace (escrow)
    list_nft(seller, nft, 2000);  // Fails with E_NOT_OWNER, not E_ALREADY_LISTED!
}
```

**Solution:** Understand the order of checks. After escrow, seller no longer owns the NFT:

```move
// ✅ CORRECT expectation
#[test(seller = @0x1)]
#[expected_failure(abort_code = marketplace::E_NOT_OWNER)]
public fun test_cannot_list_twice(seller: &signer) {
    let nft = mint_test_nft(seller);
    list_nft(seller, nft, 1000);  // NFT transfers to marketplace
    list_nft(seller, nft, 2000);  // Seller doesn't own it anymore -> E_NOT_OWNER
}
```

### Acquires Annotations

**Problem:** Adding unnecessary `acquires` annotations causes compiler errors.

```move
// ❌ WRONG - StakeTokenRefs not borrowed by YOUR code
public entry fun stake(user: &signer, amount: u64) acquires VaultConfig, Stakes, StakeTokenRefs {
    // ...
    // primary_fungible_store::transfer() handles its own borrows internally
    primary_fungible_store::transfer(user, stake_metadata, vault_addr, amount);
}
```

**Solution:** Only list resources that YOUR code directly borrows with `borrow_global` or `borrow_global_mut`:

```move
// ✅ CORRECT - Only list what YOU borrow
public entry fun stake(user: &signer, amount: u64) acquires VaultConfig, Stakes {
    // VaultConfig: your code borrows it
    let config = borrow_global<VaultConfig>(vault_addr);
    // Stakes: your code borrows it
    let stakes = borrow_global_mut<Stakes>(vault_addr);
    // Framework functions handle their own acquires - don't list them
    primary_fungible_store::transfer(user, stake_metadata, vault_addr, amount);
}
```

---

## Additional Resources

**Official Testing Documentation:**

- https://aptos.dev/build/smart-contracts/book/unit-testing
- https://aptos.dev/build/cli (for coverage commands)

**Related Patterns:**

- `SECURITY.md` - Security testing requirements
- `OBJECTS.md` - Testing object operations

---

**Remember:** 100% test coverage is mandatory. Never deploy without comprehensive tests. Tests are your contract's proof
of correctness.
