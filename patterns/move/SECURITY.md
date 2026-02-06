# Aptos Move V2 Security Patterns

**Purpose:** Comprehensive security guidelines for secure Aptos Move V2 smart contracts.

**Based on:**
[Official Aptos Move Security Guidelines](https://aptos.dev/build/smart-contracts/move-security-guidelines)

**Target:** AI assistants generating Move V2 smart contracts

**Critical Note:** Security is non-negotiable. Every pattern here must be followed. User funds depend on it.

---

## Security Checklist

Before deploying any Move module, verify ALL items in this checklist:

### Access Control

- [ ] All `entry` functions verify signer authority with `assert!`
- [ ] Object ownership verified with `object::owner()` before operations
- [ ] Use `borrow_global_mut<T>(signer::address_of(user))` to scope operations to signer's storage
- [ ] Admin-only functions check caller is admin address
- [ ] Function visibility uses least-privilege (private > public(friend) > public > entry)
- [ ] No public functions that modify state without authorization checks

### Input Validation

- [ ] **CRITICAL:** User-facing functions use explicit amount parameters (never implicit balance queries)
- [ ] Numeric inputs checked for zero where appropriate: `assert!(amount > 0, E_ZERO_AMOUNT)`
- [ ] Numeric inputs checked for minimum thresholds to prevent rounding to zero
- [ ] Vector lengths validated: `assert!(vector::length(&items) > 0, E_EMPTY_VECTOR)`
- [ ] String lengths checked: `assert!(string::length(&name) <= MAX_LENGTH, E_NAME_TOO_LONG)`
- [ ] Addresses validated as non-zero: `assert!(addr != @0x0, E_ZERO_ADDRESS)`
- [ ] Generic type parameters validated for consistency (especially in flash loans)

### Object Safety

- [ ] ConstructorRef never returned from public functions
- [ ] All refs (TransferRef, DeleteRef, ExtendRef) generated in constructor before ConstructorRef destroyed
- [ ] Object signer only used during construction or with ExtendRef
- [ ] Logically independent resources stored in separate object accounts
- [ ] Ungated transfers disabled unless explicitly needed
- [ ] DeleteRef only generated for truly burnable objects

### Reference Safety

- [ ] No `&mut` references exposed in public function signatures
- [ ] Critical fields protected from `mem::swap` attacks
- [ ] Mutable references to untrusted code validated before AND after mutations
- [ ] Mutable borrows minimized in scope

### Arithmetic Safety

- [ ] All additions checked for overflow or use Move's abort-on-overflow
- [ ] All subtractions checked for underflow or use Move's abort-on-underflow
- [ ] Division by zero prevented: `assert!(divisor > 0, E_DIVISION_BY_ZERO)`
- [ ] **CRITICAL:** Division results validated for non-zero when fees/increments expected
- [ ] **CRITICAL:** Multi-step calculations restructured to prevent intermediate overflow
- [ ] Left shift operations avoided or results validated (does NOT abort on overflow)
- [ ] Multiplication checked for overflow when needed

### Resource Management

- [ ] No unbounded iterations over publicly mutable structures
- [ ] User-specific data stored in user accounts, not global storage
- [ ] Use `SmartTable` or similar for efficient per-user scoped data
- [ ] Module data in Objects, user data in user accounts

### Generic Type Safety

- [ ] Use `phantom` for generic types not stored in fields: `struct Vault<phantom CoinType>`
- [ ] Type witnesses used for authorization where appropriate
- [ ] Generic function constraints appropriate (`drop`, `copy`, `store`, `key`)
- [ ] Flash loan repayment types match borrowed types

### Business Logic

- [ ] Front-running prevented through atomic operations
- [ ] Price oracles use multiple independent sources (never single on-chain ratio)
- [ ] Token identifiers use deterministic, collision-proof schemes (object addresses)
- [ ] Pause functionality implemented for emergency response

### DeFi-Specific (if applicable)

- [ ] **CRITICAL:** Never assume 1:1 price ratios - always use price oracles
- [ ] Multi-oracle design with fallback sources
- [ ] Price staleness checks implemented (max age limits)
- [ ] Price deviation limits between oracles
- [ ] Slippage protection parameters for swaps
- [ ] Deadline timestamps for time-sensitive operations
- [ ] Partial liquidation support (not all-or-nothing)
- [ ] Liquidation incentives properly bounded

### Governance (if applicable)

- [ ] All proposals have expiration times
- [ ] Owner/member removal validates threshold requirements
- [ ] Pending approvals handled on owner removal
- [ ] Threshold changes validated against owner count

### Randomness (if applicable)

- [ ] Randomness-using functions are `entry` (not `public`) to prevent composition
- [ ] Gas consumption balanced across win/lose paths to prevent undergasing
- [ ] Consider commit-reveal pattern for sensitive randomness

### Function Values / Reentrancy (if using Move 2.2+ function values)

- [ ] Functions accepting callbacks that modify state use `#[module_lock]`
- [ ] Stored function values only accepted from trusted/authorized sources
- [ ] Non-public functions stored on-chain are marked `#[persistent]`
- [ ] Reentrancy implications considered for all function value callbacks

### Testing

- [ ] 100% line coverage achieved: `aptos move test --coverage`
- [ ] All error paths tested with `#[expected_failure(abort_code = E_CODE)]`
- [ ] Access control tested with multiple signers (owner, attacker)
- [ ] Input validation tested with invalid inputs (zero, overflow, empty)
- [ ] Edge cases covered (empty vectors, max values, boundary conditions)

### Operations

- [ ] Publishing keys separated by network (testnet vs mainnet)
- [ ] Emergency pause mechanism implemented and tested
- [ ] Upgrade functionality secured if contract is upgradable

---

## Pattern 1: Access Control

### 1.1 Verifying Signer Authority

**CORRECT - Always verify signer matches expected address:**

```move
const E_UNAUTHORIZED: u64 = 1;

/// Update global config (admin only)
public entry fun update_config(admin: &signer, new_value: u64) acquires Config {
    let config = borrow_global_mut<Config>(@my_addr);

    // ✅ CORRECT: Verify signer is admin
    assert!(signer::address_of(admin) == config.admin, E_UNAUTHORIZED);

    config.value = new_value;
}
```

**INCORRECT - No verification:**

```move
// ❌ DANGEROUS: Anyone can call this
public entry fun update_config(admin: &signer, new_value: u64) acquires Config {
    let config = borrow_global_mut<Config>(@my_addr);
    config.value = new_value; // No check!
}
```

### 1.2 Verifying Object Ownership

**CORRECT - Check object ownership:**

```move
const E_NOT_OWNER: u64 = 2;

/// Transfer item (owner only)
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    recipient: address
) acquires Item {
    // ✅ CORRECT: Verify signer owns object
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    // Safe to transfer
    let item_data = borrow_global<Item>(object::object_address(&item));
    object::transfer_with_ref(
        object::generate_linear_transfer_ref(&item_data.transfer_ref),
        recipient
    );
}
```

**INCORRECT - No ownership check:**

```move
// ❌ DANGEROUS: Anyone can transfer anyone's items
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    recipient: address
) acquires Item {
    // No ownership verification!
    let item_data = borrow_global<Item>(object::object_address(&item));
    object::transfer_with_ref(
        object::generate_linear_transfer_ref(&item_data.transfer_ref),
        recipient
    );
}
```

### 1.3 Global Storage Access Control

**CRITICAL: Always scope `borrow_global_mut` to signer's address**

```move
// ✅ CORRECT: Operations scoped to signer's own storage
public entry fun update_balance(user: &signer, amount: u64) acquires Account {
    let user_addr = signer::address_of(user);

    // SAFE: Can only modify the signer's own account
    let account = borrow_global_mut<Account>(user_addr);
    account.balance = account.balance + amount;
}

// ❌ DANGEROUS: Accepts arbitrary address parameter
public entry fun update_balance_wrong(
    user: &signer,
    target_addr: address,  // DANGEROUS
    amount: u64
) acquires Account {
    // User can modify ANY account, not just their own!
    let account = borrow_global_mut<Account>(target_addr);
    account.balance = account.balance + amount;
}
```

### 1.4 Function Visibility Hierarchy

**Principle of Least Privilege - Start private, escalate only as needed:**

```move
// Private: Only this module
fun internal_helper() { }

// Public(friend): Only friend modules
public(friend) fun restricted_operation() { }

// Public: Any module can call, but not CLI/SDK
public fun library_function() { }

// Entry: Can be called from CLI/SDK
public entry fun user_facing_action(user: &signer) { }

// View: Read-only, for queries
#[view]
public fun get_balance(addr: address): u64 { }
```

### 1.5 Role-Based Access Control

```move
module my_addr::marketplace {
    use std::signer;
    use std::vector;

    struct Marketplace has key {
        admin: address,
        operators: vector<address>,
        paused: bool,
    }

    const E_NOT_ADMIN: u64 = 1;
    const E_NOT_OPERATOR: u64 = 2;
    const E_PAUSED: u64 = 3;

    /// Admin-only: Add operator
    public entry fun add_operator(
        admin: &signer,
        operator: address
    ) acquires Marketplace {
        let marketplace = borrow_global_mut<Marketplace>(@my_addr);
        assert!(signer::address_of(admin) == marketplace.admin, E_NOT_ADMIN);
        vector::push_back(&mut marketplace.operators, operator);
    }

    /// Operator-only: Pause marketplace
    public entry fun pause_marketplace(operator: &signer) acquires Marketplace {
        let marketplace = borrow_global_mut<Marketplace>(@my_addr);
        let operator_addr = signer::address_of(operator);
        assert!(
            vector::contains(&marketplace.operators, &operator_addr),
            E_NOT_OPERATOR
        );
        marketplace.paused = true;
    }

    /// Internal: Check if paused
    fun assert_not_paused() acquires Marketplace {
        let marketplace = borrow_global<Marketplace>(@my_addr);
        assert!(!marketplace.paused, E_PAUSED);
    }
}
```

---

## Pattern 2: Input Validation - Implicit Amounts (CRITICAL)

### 2.1 Never Use Implicit Amounts

**CRITICAL RULE:** Never use balance queries as implicit transfer amounts.

❌ **WRONG - Dangerous implicit amount:**

```move
public entry fun place_bid(bidder: &signer, auction: Object<Auction>) {
    let bid_amount = coin::balance<AptosCoin>(signer::address_of(bidder));
    // User loses ENTIRE balance!
    coin::transfer<AptosCoin>(bidder, auction_addr, bid_amount);
}
```

✅ **CORRECT - Explicit amount parameter:**

```move
public entry fun place_bid(
    bidder: &signer,
    auction: Object<Auction>,
    bid_amount: u64  // User specifies exact amount
) {
    assert!(bid_amount > 0, E_ZERO_AMOUNT);
    let user_balance = coin::balance<AptosCoin>(signer::address_of(bidder));
    assert!(user_balance >= bid_amount, E_INSUFFICIENT_BALANCE);
    coin::transfer<AptosCoin>(bidder, auction_addr, bid_amount);
}
```

**Why This Matters:**

- User consent: Users must explicitly approve exact amounts
- Safety: Prevents draining entire balance accidentally
- Predictability: Users know exactly what they're spending

**Applies To:**

- `coin::transfer` and `coin::withdraw`
- `primary_fungible_store::transfer` and `withdraw`
- Any user-facing token operation

**Exception:** Internal/admin functions can query balances, but user-facing `entry` functions MUST have explicit amount
parameters.

### 2.2 Numeric Validation

```move
const E_ZERO_AMOUNT: u64 = 10;
const E_AMOUNT_TOO_SMALL: u64 = 11;
const E_INSUFFICIENT_BALANCE: u64 = 12;
const E_ZERO_ADDRESS: u64 = 13;

const MIN_TRANSFER_AMOUNT: u64 = 100; // Prevent rounding to zero
const PROTOCOL_FEE_BPS: u64 = 30; // 0.3%

/// Transfer tokens with comprehensive validation
public entry fun transfer(
    from: &signer,
    to: address,
    amount: u64
) acquires Account {
    // ✅ Validate amount is non-zero
    assert!(amount > 0, E_ZERO_AMOUNT);

    // ✅ Validate amount meets minimum (prevents fee rounding to zero)
    assert!(amount >= MIN_TRANSFER_AMOUNT, E_AMOUNT_TOO_SMALL);

    // ✅ Calculate and validate fee is non-zero
    let fee = (amount * PROTOCOL_FEE_BPS) / 10000;
    assert!(fee > 0, E_ZERO_AMOUNT); // Ensure fee doesn't round to zero

    // ✅ Validate sender has sufficient balance
    let from_account = borrow_global<Account>(signer::address_of(from));
    assert!(from_account.balance >= amount, E_INSUFFICIENT_BALANCE);

    // ✅ Validate recipient is not zero address
    assert!(to != @0x0, E_ZERO_ADDRESS);

    // Safe to proceed
    // ... transfer logic
}
```

### 2.3 String Validation

```move
const E_EMPTY_NAME: u64 = 20;
const E_NAME_TOO_LONG: u64 = 21;
const MAX_NAME_LENGTH: u64 = 32;

/// Set item name with validation
public entry fun set_name(
    owner: &signer,
    item: Object<Item>,
    name: String
) acquires Item {
    // ✅ Validate name is not empty
    assert!(string::length(&name) > 0, E_EMPTY_NAME);

    // ✅ Validate name length
    assert!(string::length(&name) <= MAX_NAME_LENGTH, E_NAME_TOO_LONG);

    // ✅ Validate ownership
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = name;
}
```

### 2.4 Vector Validation

```move
const E_EMPTY_VECTOR: u64 = 30;
const E_TOO_MANY_ITEMS: u64 = 31;
const MAX_ITEMS: u64 = 100;

/// Add items with validation
public entry fun add_items(
    owner: &signer,
    collection: Object<Collection>,
    items: vector<Object<Item>>
) acquires Collection {
    // ✅ Validate vector is not empty
    assert!(vector::length(&items) > 0, E_EMPTY_VECTOR);

    // ✅ Validate total count within bounds
    let collection_data = borrow_global_mut<Collection>(object::object_address(&collection));
    let new_total = vector::length(&collection_data.items) + vector::length(&items);
    assert!(new_total <= MAX_ITEMS, E_TOO_MANY_ITEMS);

    // Safe to add items
    vector::append(&mut collection_data.items, items);
}
```

### 2.5 Generic Type Validation

**CRITICAL: Prevent flash loan token type mismatch**

```move
// ✅ CORRECT: Type parameter ensures borrowed and repaid types match
struct Receipt<phantom T> has drop {
    amount: u64,
}

public fun borrow<CoinType>(user: &signer, amount: u64): Receipt<CoinType> {
    // ... lending logic
    Receipt<CoinType> { amount }
}

public fun repay<CoinType>(user: &signer, receipt: Receipt<CoinType>) {
    let Receipt { amount } = receipt;
    // Type system guarantees CoinType matches borrowed type
    // ... repayment logic
}

// ❌ DANGEROUS: No type validation
struct Receipt has drop {
    amount: u64,
}

// Can borrow BTC, repay with worthless token!
```

---

## Pattern 3: Arithmetic Safety

### 3.1 Overflow/Underflow (Move aborts automatically)

```move
const E_INSUFFICIENT_BALANCE: u64 = 40;

/// Deposit - Move aborts on overflow automatically
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));
    // Move runtime aborts if this overflows
    account.balance = account.balance + amount;
}

/// Withdraw - Explicit underflow check for better error message
public entry fun withdraw(user: &signer, amount: u64) acquires Account {
    let account = borrow_global_mut<Account>(signer::address_of(user));

    // ✅ Explicit check gives clearer error
    assert!(account.balance >= amount, E_INSUFFICIENT_BALANCE);

    account.balance = account.balance - amount; // Safe
}
```

### 3.2 Division Precision Loss

**CRITICAL: Integer division rounds down - validate results**

```move
const E_DIVISION_BY_ZERO: u64 = 42;
const E_AMOUNT_TOO_SMALL: u64 = 43;
const MIN_ORDER_SIZE: u64 = 1000; // Minimum to ensure non-zero fees

const PROTOCOL_FEE_BPS: u64 = 30; // 0.3% = 30 basis points

/// Calculate fee with precision loss protection
public fun calculate_fee(amount: u64): u64 {
    // Method 1: Enforce minimum threshold
    assert!(amount >= MIN_ORDER_SIZE, E_AMOUNT_TOO_SMALL);

    let fee = (amount * PROTOCOL_FEE_BPS) / 10000;

    // Method 2: Validate non-zero result
    assert!(fee > 0, E_AMOUNT_TOO_SMALL);

    fee
}

// Example:
// - amount = 100, fee = (100 * 30) / 10000 = 0 ❌ Rounds to zero!
// - amount = 1000, fee = (1000 * 30) / 10000 = 3 ✅ Non-zero
```

### 3.3 Division Result Validation

**CRITICAL:** Division can round to zero for small amounts, breaking fee logic and minimum increments.

❌ **WRONG - Can round to zero:**

```move
let fee = amount / 100;  // Rounds to 0 if amount < 100!
coin::transfer(user, platform, fee);  // Fee bypassed!

let min_bid = highest_bid + (highest_bid / 20);  // 5% increment
// For highest_bid < 20, increment = 0!
```

✅ **CORRECT - Validate non-zero or use minimum:**

```move
// Option 1: Minimum fee
const MIN_FEE: u64 = 1;
let fee = amount / 100;
if (fee == 0 && amount > 0) {
    fee = MIN_FEE;
};

// Option 2: Minimum amount to prevent rounding
const MIN_AMOUNT_FOR_FEES: u64 = 100;
assert!(amount >= MIN_AMOUNT_FOR_FEES, E_AMOUNT_TOO_SMALL);
let fee = amount / 100;

// Option 3: Check result non-zero
let increment = highest_bid / 20;
if (increment == 0) {
    increment = 1;  // Minimum increment
};
```

**When to Apply:**

- Fee calculations (platform fees, protocol fees)
- Percentage-based increments (auction minimum bids)
- Any division where zero result breaks logic

### 3.4 Multi-Step Calculation Overflow

**Intermediate results can overflow even if final result fits.**

❌ **WRONG - Risk of overflow:**

```move
// Each value might be safe individually, but a * b could overflow!
let result = (a * b * c) / d;

// Interest calculation example:
let interest = (principal * rate * time_elapsed) / (PRECISION * SECONDS_PER_YEAR);
// principal * rate * time_elapsed might overflow!
```

✅ **CORRECT - Restructure to reduce intermediate size:**

```move
// Option 1: Divide early to keep intermediates smaller
let result = ((a / d) * b * c);

// Option 2: Break into steps with overflow checks
let temp = (a as u128) * (b as u128);  // Use larger type
let result = ((temp * (c as u128)) / (d as u128)) as u64;

// Option 3: For interest, restructure calculation
let rate_per_second = ANNUAL_INTEREST_RATE / SECONDS_PER_YEAR;
let interest = ((principal / PRECISION) * rate_per_second * time_elapsed);

// Option 4: Add maximum amount checks
const MAX_SAFE_PRINCIPAL: u128 = 1_000_000_000_000_000;  // Calculate based on formula
assert!(principal <= MAX_SAFE_PRINCIPAL, E_AMOUNT_TOO_LARGE);
let interest = (principal * rate * time_elapsed) / (PRECISION * SECONDS_PER_YEAR);
```

**When to Apply:**

- Interest calculations over time
- Complex fee formulas
- Any formula with multiple multiplications

### 3.5 Left Shift Overflow (Does NOT abort!)

**CRITICAL: Left shift can produce incorrect values without aborting**

```move
// ❌ DANGEROUS: Left shift doesn't abort on overflow
public fun calculate_power_of_two(exponent: u8): u64 {
    // If exponent >= 64, this silently produces wrong result!
    1 << exponent
}

// Error codes
const E_OVERFLOW: u64 = 1;
// ✅ CORRECT: Validate shift amount
public fun calculate_power_of_two_safe(exponent: u8): u64 {
    assert!(exponent < 64, E_OVERFLOW);
    1 << exponent
}

// ✅ BETTER: Avoid left shift for critical calculations
public fun calculate_power_of_two_safer(exponent: u8): u64 {
    let result = 1u64;
    let i = 0;
    while (i < exponent) {
        result = result * 2; // Move aborts on overflow
        i = i + 1;
    };
    result
}
```

### 3.6 Division by Zero

```move
const E_DIVISION_BY_ZERO: u64 = 44;

/// Safe division
public fun safe_divide(numerator: u64, denominator: u64): u64 {
    assert!(denominator > 0, E_DIVISION_BY_ZERO);
    numerator / denominator
}
```

---

## Pattern 4: Object Security

### 4.1 Never Return ConstructorRef

**CRITICAL: Returned ConstructorRef can be used to reclaim ownership**

```move
// ❌ DANGEROUS: Exposes ConstructorRef
public fun create_item_wrong(creator: &signer): ConstructorRef {
    let constructor_ref = object::create_object(signer::address_of(creator));
    // DANGEROUS: Caller can:
    // 1. Delete the object
    // 2. Generate arbitrary refs
    // 3. Reclaim ownership after transfer
    constructor_ref
}

// ✅ CORRECT: Return typed Object reference
public fun create_item_correct(creator: &signer, name: String): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Generate all needed refs
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);

    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Item {
        name,
        transfer_ref,
        delete_ref,
    });

    // ✅ Return safe typed reference
    object::object_from_constructor_ref<Item>(&constructor_ref)
}
```

### 4.2 Object Account Isolation

**CRITICAL: Separate objects = separate accounts**

```move
// ✅ CORRECT: Each NFT in separate account
public fun mint_nft(creator: &signer, name: String): Object<NFT> {
    // Each NFT gets its own object account
    let constructor_ref = object::create_object(signer::address_of(creator));

    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, NFT {
        name,
        transfer_ref,
    });

    object::object_from_constructor_ref<NFT>(&constructor_ref)
}

// ❌ WRONG: Multiple resources in one account
public fun mint_nft_wrong(creator: &signer) {
    // Reusing same object account for multiple NFTs
    let constructor_ref = object::create_object(signer::address_of(creator));
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, NFT { name: string::utf8(b"NFT1") });
    move_to(&object_signer, Metadata { rarity: 1 });
    // Transferring object transfers BOTH resources!
}
```

### 4.3 Generate All Refs in Constructor

```move
// ✅ CORRECT: All refs generated before ConstructorRef destroyed
public fun create_item(creator: &signer): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Generate ALL refs you'll need NOW
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);
    let extend_ref = object::generate_extend_ref(&constructor_ref);

    let object_signer = object::generate_signer(&constructor_ref);

    // Store refs for later use
    move_to(&object_signer, Item {
        name: string::utf8(b""),
        transfer_ref,
        delete_ref,
        extend_ref,
    });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}

// ❌ WRONG: Can't generate refs later (ConstructorRef destroyed)
public fun add_delete_capability_later(item: Object<Item>) {
    // ERROR: Can't do this - ConstructorRef is gone!
    // let delete_ref = object::generate_delete_ref(&???);
}
```

---

## Pattern 5: Reference Safety

### 5.1 No Public Mutable References

```move
// ❌ DANGEROUS: Exposes mutable reference
public fun get_item_mut(item: Object<Item>): &mut Item acquires Item {
    // Allows caller to use mem::swap, breaking invariants
    borrow_global_mut<Item>(object::object_address(&item))
}

// ✅ CORRECT: Expose specific operations
public entry fun update_item_name(
    owner: &signer,
    item: Object<Item>,
    new_name: String
) acquires Item {
    // Verify ownership
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    // Controlled mutation
    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = new_name;
}
```

### 5.2 Protect Against mem::swap

```move
use std::mem;

struct Item has key {
    name: String,
    creator: address,  // CRITICAL: Must never change
    transfer_ref: object::TransferRef,
}

// ❌ DANGEROUS: If attacker gets &mut Item:
fun attacker_swap(item1: &mut Item, item2: &mut Item) {
    // Swaps ALL fields including creator and transfer_ref
    mem::swap(item1, item2);
    // Now ownership is confused!
}

// ✅ PROTECTION: Never expose &mut publicly
// Always use internal functions with access control
```

### 5.3 Mutable References in Callbacks

**CRITICAL: Re-validate after untrusted mutations**

```move
struct FungibleAsset has key {
    metadata: Metadata,
    amount: u64,
}

// ❌ DANGEROUS: Callback can swap asset
public fun process_with_hook(
    asset: &mut FungibleAsset,
    callback: |&mut FungibleAsset|
) {
    // Asset is valid here
    callback(asset);
    // Asset might be swapped with worthless asset!
}

// ✅ CORRECT: Validate before AND after
public fun process_with_hook_safe(
    asset: &mut FungibleAsset,
    callback: |&mut FungibleAsset|
) {
    let expected_metadata = asset.metadata;

    // Validate before
    check_metadata(&asset);

    // Call untrusted code
    callback(asset);

    // Re-validate after
    assert!(asset.metadata == expected_metadata, E_INVALID_ASSET);
    check_metadata(&asset);
}
```

---

## Pattern 6: Generic Type Safety

### 6.1 Phantom Types

```move
// ✅ CORRECT: Phantom type parameter
struct Vault<phantom CoinType> has key {
    balance: u64,
    // CoinType is only for type safety, not stored
}

/// Deposit specific coin type
public fun deposit<CoinType>(
    user: &signer,
    vault: Object<Vault<CoinType>>,
    amount: u64
) acquires Vault {
    // Type safety: can't deposit USDC into BTC vault
    let vault_data = borrow_global_mut<Vault<CoinType>>(
        object::object_address(&vault)
    );
    vault_data.balance = vault_data.balance + amount;
}

// ❌ WRONG: Not using phantom
struct Vault<CoinType> has key {
    balance: u64, // Compiler error: CoinType not used
}
```

### 6.2 Type Witness Pattern

```move
/// Type witness for authorization
struct AdminCapability has drop {}

/// Only callable with AdminCapability
public fun privileged_operation(
    _witness: AdminCapability,
    value: u64
) acquires Config {
    // Caller must possess AdminCapability
    let config = borrow_global_mut<Config>(@my_addr);
    config.value = value;
}

/// Get admin capability (restricted)
public fun get_admin_capability(admin: &signer): AdminCapability acquires Config {
    let config = borrow_global<Config>(@my_addr);
    assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
    AdminCapability {}
}
```

---

## Pattern 7: DeFi-Specific Security

### 7.1 Price Oracles (CRITICAL for DeFi)

**CRITICAL RULE:** Never assume 1:1 price ratios between different assets.

❌ **WRONG - Broken economics:**

```move
// Assumes all tokens have same value!
let collateral_value = collateral_amount;
let borrowed_value = borrowed_amount;
let ratio = (collateral_value * 100) / borrowed_value;
```

**Impact:** Protocol can be drained by depositing worthless tokens and borrowing valuable ones.

✅ **CORRECT - Use price oracles:**

```move
// Use Pyth, Switchboard, or other oracle
let collateral_price = oracle::get_price(collateral_metadata);
let borrow_price = oracle::get_price(borrow_metadata);

let collateral_value = (collateral_amount as u128) * collateral_price;
let borrowed_value = (borrowed_amount as u128) * borrow_price;

let ratio = (collateral_value * 100) / borrowed_value;
```

**Requirements for Production DeFi:**

1. **Multi-Oracle Design:**
   - Use at least 2 independent oracle sources
   - Implement fallback oracles
   - Add circuit breaker if oracles disagree significantly

2. **Price Staleness Checks:**

   ```move
   const MAX_PRICE_AGE: u64 = 300; // 5 minutes
   let price_timestamp = oracle::get_price_timestamp(asset);
   let current_time = timestamp::now_seconds();
   assert!(current_time - price_timestamp <= MAX_PRICE_AGE, E_STALE_PRICE);
   ```

3. **Price Deviation Limits:**

   ```move
   const MAX_PRICE_DEVIATION_BPS: u128 = 1000; // 10%
   let oracle1_price = oracle1::get_price(asset);
   let oracle2_price = oracle2::get_price(asset);
   let diff = if (oracle1_price > oracle2_price) {
       oracle1_price - oracle2_price
   } else {
       oracle2_price - oracle1_price
   };
   let max_deviation = (oracle1_price * MAX_PRICE_DEVIATION_BPS) / 10000;
   assert!(diff <= max_deviation, E_ORACLE_DEVIATION_TOO_HIGH);
   ```

4. **Never Use Single On-Chain Ratios:**
   - Vulnerable to flash loan manipulation
   - Use time-weighted average prices (TWAP)
   - Add minimum liquidity requirements

### 7.2 MEV Protection

**Front-running Prevention:**

- Use atomic operations (all-or-nothing)
- Add slippage protection parameters
- Implement deadline timestamps for time-sensitive operations

```move
public entry fun swap(
    user: &signer,
    amount_in: u64,
    min_amount_out: u64,  // Slippage protection
    deadline: u64         // MEV protection
) {
    let current_time = timestamp::now_seconds();
    assert!(current_time <= deadline, E_DEADLINE_EXCEEDED);
    // ... swap logic
    assert!(amount_out >= min_amount_out, E_SLIPPAGE_TOO_HIGH);
}
```

### 7.3 Liquidation Safety

For lending/borrowing protocols:

1. **Partial Liquidation:**
   - Don't liquidate entire position at once
   - Allow liquidators to specify amount
   - Prevents excessive slippage

2. **Liquidation Incentive:**
   - Reward liquidators (e.g., 5-10% bonus)
   - But cap maximum bonus to prevent abuse

3. **Minimum Collateralization:**
   - Enforce buffer between liquidation threshold and minimum ratio
   - Example: 150% minimum, 120% liquidation threshold
   - Gives borrowers time to add collateral

---

## Pattern 8: Governance and Multi-Sig Security

### 8.1 Proposal Expiration

**All governance proposals MUST have expiration times.**

❌ **WRONG - No expiration:**

```move
struct Proposal has key {
    description: String,
    approvals: vector<address>,
    executed: bool,
}
// Can be executed years later in changed circumstances!
```

✅ **CORRECT - With expiration:**

```move
struct Proposal has key {
    description: String,
    approvals: vector<address>,
    executed: bool,
    expiration_time: u64,  // Add this
}

public entry fun execute_proposal(executor: &signer, proposal: Object<Proposal>) {
    // ... approval checks ...
    let current_time = timestamp::now_seconds();
    assert!(current_time <= proposal_data.expiration_time, E_PROPOSAL_EXPIRED);
    // ... execute
}
```

**Recommended Expiration Times:**

- Standard proposals: 7 days
- Urgent proposals: 3 days
- Parameter changes: 14 days

### 8.2 Owner/Member Removal Safety

**When removing owners/members with voting power:**

```move
public entry fun propose_remove_owner(
    proposer: &signer,
    wallet: Object<MultiSigWallet>,
    owner_to_remove: address
) {
    // 1. Check threshold still met after removal
    let wallet_data = borrow_global<MultiSigWallet>(object_address);
    let remaining_owners = vector::length(&wallet_data.owners) - 1;
    assert!(wallet_data.threshold <= remaining_owners, E_THRESHOLD_TOO_HIGH);

    // 2. Handle pending approvals (choose one):
    // Option A: Reject removal if owner has approved pending proposals
    assert!(!has_pending_approvals(owner_to_remove), E_HAS_PENDING_APPROVALS);

    // Option B: Automatically revoke approvals from removed owner
    revoke_all_approvals(owner_to_remove);
}
```

### 8.3 Threshold Management

**Never allow threshold to exceed number of owners:**

```move
public entry fun change_threshold(
    wallet: Object<MultiSigWallet>,
    new_threshold: u64
) {
    let wallet_data = borrow_global_mut<MultiSigWallet>(object_address);
    let num_owners = vector::length(&wallet_data.owners);

    // Validate bounds
    assert!(new_threshold >= 1, E_THRESHOLD_TOO_LOW);
    assert!(new_threshold <= num_owners, E_THRESHOLD_TOO_HIGH);

    wallet_data.threshold = new_threshold;
}
```

---

## Pattern 9: Resource Management

### 9.1 Avoid Unbounded Iterations

**CRITICAL: Gas exhaustion attacks via unbounded loops**

```move
// ❌ DANGEROUS: Unbounded iteration over global storage
struct GlobalOrders has key {
    orders: vector<Order>,  // Anyone can add, grows unboundedly
}

public fun process_all_orders() acquires GlobalOrders {
    let orders = &borrow_global<GlobalOrders>(@protocol).orders;
    let i = 0;
    // Gas exhaustion: loop can become too expensive to execute
    while (i < vector::length(orders)) {
        process_order(vector::borrow(orders, i));
        i = i + 1;
    };
}

// ✅ CORRECT: Per-user storage, scoped operations
struct UserOrders has key {
    orders: vector<Order>,  // User controls their own data size
}

public entry fun process_user_orders(user: &signer) acquires UserOrders {
    let user_addr = signer::address_of(user);
    let orders = &borrow_global<UserOrders>(user_addr).orders;
    let i = 0;
    // Safe: User can't DOS others, only themselves
    while (i < vector::length(orders)) {
        process_order(vector::borrow(orders, i));
        i = i + 1;
    };
}

// ✅ BETTER: Direct lookup, no iteration (using SmartTable)
struct UserOrdersTable has key {
    orders_table: smart_table::SmartTable<u64, Order>,
}

public fun get_order_by_id(
    user: address,
    order_id: u64
): Order acquires UserOrdersTable {
    let user_orders = borrow_global<UserOrdersTable>(user);
    // Direct access, no loop
    *smart_table::borrow(&user_orders.orders_table, order_id)
}
```

### 9.2 Storage Structure Best Practices

```move
// ✅ CORRECT: Module data in Objects, user data in user accounts
module my_addr::marketplace {
    // Protocol-level data in an Object
    struct Marketplace has key {
        admin: address,
        fee_bps: u64,
        paused: bool,
    }

    // User-specific data in user accounts
    struct UserProfile has key {
        orders: smart_table::SmartTable<u64, Order>,
        total_trades: u64,
    }

    public entry fun create_order(user: &signer, price: u64) acquires UserProfile {
        let user_addr = signer::address_of(user);
        // Each user manages their own data
        let profile = borrow_global_mut<UserProfile>(user_addr);
        // ... create order
    }
}
```

---

## Pattern 10: Business Logic Security

### 10.1 Front-Running Prevention

**CRITICAL: Atomic finalization prevents front-running**

```move
// ❌ VULNERABLE: Two-step process allows front-running
public entry fun set_winner_number(admin: &signer, number: u64) acquires Lottery {
    let lottery = borrow_global_mut<Lottery>(@protocol);
    assert!(signer::address_of(admin) == lottery.admin, E_NOT_ADMIN);
    lottery.winner_number = number;
    // Attacker sees this transaction, submits winning bet before next step!
}

public entry fun evaluate_bets(admin: &signer) acquires Lottery {
    // Process bets against winner_number
    // Too late - attacker already submitted winning bet
}

// ✅ CORRECT: Atomic operation
public entry fun finalize_lottery(admin: &signer, number: u64) acquires Lottery {
    let lottery = borrow_global_mut<Lottery>(@protocol);
    assert!(signer::address_of(admin) == lottery.admin, E_NOT_ADMIN);

    // Atomic: No opportunity for front-running
    lottery.winner_number = number;
    evaluate_bets_internal(lottery);
}

fun evaluate_bets_internal(lottery: &mut Lottery) {
    // Process bets atomically
}
```

### 10.2 Price Oracle Manipulation

**CRITICAL: Never use single on-chain ratio as sole price oracle**

```move
// ❌ DANGEROUS: Single price source
public fun get_price(): u64 acquires Pool {
    let pool = borrow_global<Pool>(@protocol);
    // Attacker can manipulate via large swap
    pool.token_a_reserve / pool.token_b_reserve
}

// ✅ CORRECT: Multi-oracle design
public fun get_price(): u64 acquires OracleConfig {
    let config = borrow_global<OracleConfig>(@protocol);

    // Try primary oracle
    let (primary_price, primary_valid) = get_primary_oracle_price();
    if (primary_valid) {
        return primary_price
    };

    // Fallback to secondary oracle
    let (secondary_price, secondary_valid) = get_secondary_oracle_price();
    if (secondary_valid) {
        return secondary_price
    };

    // Emergency: Use time-weighted average
    get_twap_price()
}
```

### 10.3 Token Identifier Collision

**CRITICAL: Use collision-proof identifiers**

```move
// ❌ DANGEROUS: String concatenation can collide
public fun get_lp_token_symbol(token_a: String, token_b: String): String {
    let result = string::utf8(b"LP-");
    string::append(&mut result, token_a);
    string::append(&mut result, string::utf8(b"-"));
    string::append(&mut result, token_b);
    // "AB" + "C" == "A" + "BC" - COLLISION!
    result
}

// ✅ CORRECT: Use object addresses (guaranteed unique)
public fun get_lp_token_id<TokenA, TokenB>(): vector<u8> {
    // Object addresses are deterministic and collision-proof
    let token_a_addr = type_info::type_of<TokenA>();
    let token_b_addr = type_info::type_of<TokenB>();

    let id = vector::empty<u8>();
    vector::append(&mut id, bcs::to_bytes(&token_a_addr));
    vector::append(&mut id, bcs::to_bytes(&token_b_addr));
    id
}
```

### 10.4 Pause Functionality

**REQUIRED: Emergency pause mechanism**

```move
struct Protocol has key {
    admin: address,
    is_paused: bool,
}

const E_NOT_ADMIN: u64 = 1;
const E_PAUSED: u64 = 2;

/// Emergency pause (admin only)
public entry fun pause_protocol(admin: &signer) acquires Protocol {
    let protocol = borrow_global_mut<Protocol>(@protocol_address);
    assert!(signer::address_of(admin) == protocol.admin, E_NOT_ADMIN);
    protocol.is_paused = true;
}

/// Unpause after fixing issues
public entry fun unpause_protocol(admin: &signer) acquires Protocol {
    let protocol = borrow_global_mut<Protocol>(@protocol_address);
    assert!(signer::address_of(admin) == protocol.admin, E_NOT_ADMIN);
    protocol.is_paused = false;
}

/// Check not paused (called by all critical functions)
fun assert_not_paused() acquires Protocol {
    let protocol = borrow_global<Protocol>(@protocol_address);
    assert!(!protocol.is_paused, E_PAUSED);
}

/// Example usage
public entry fun critical_operation(user: &signer) acquires Protocol {
    // First check: protocol not paused
    assert_not_paused();

    // Proceed with operation
    // ...
}
```

---

## Pattern 11: Randomness Security

### 11.1 Prevent Test-and-Abort

**CRITICAL: Randomness functions must be `entry` (not `public`)**

```move
// ❌ DANGEROUS: Public function can be composed
public fun play_lottery(user: &signer): bool acquires Lottery {
    let random_value = aptos_framework::randomness::u64_range(1, 100);
    random_value == 42
}

// Attacker can do this:
fun attack() {
    loop {
        if (play_lottery(&user)) {
            break // Win!
        };
        // Abort and retry until win
    }
}

// ✅ CORRECT: Entry function prevents composition
entry fun play_lottery(user: &signer) acquires Lottery {
    let random_value = aptos_framework::randomness::u64_range(1, 100);
    if (random_value == 42) {
        pay_prize(user);
    }
    // Attacker can't loop/compose this
}
```

### 11.2 Prevent Undergasing

**CRITICAL: Balance gas consumption across outcomes**

```move
// ❌ DANGEROUS: Win path uses less gas than lose path
entry fun play_game(user: &signer) acquires Game {
    let random_value = aptos_framework::randomness::u64_range(1, 100);

    if (random_value <= 50) {
        // Win: 1000 gas
        pay_prize(user);
    } else {
        // Lose: 5000 gas
        complex_computation();
        update_statistics();
        emit_event();
    }
    // Attacker sets max gas = 1500, guarantees wins only
}

// ✅ CORRECT: Balanced gas consumption
entry fun play_game_safe(user: &signer) acquires Game {
    let random_value = aptos_framework::randomness::u64_range(1, 100);

    // Process result in separate step (equal gas)
    if (random_value <= 50) {
        handle_win(user);
    } else {
        handle_loss(user);
    };

    // Common finalization (equal gas for both paths)
    update_statistics();
    emit_event();
}

// ✅ ALTERNATIVE: Commit-reveal pattern
entry fun commit_game(user: &signer) {
    // Commit to random value (no result yet)
    let random_seed = aptos_framework::randomness::u64_range(1, 1000000);
    save_commitment(user, random_seed);
}

entry fun reveal_game(user: &signer) acquires Game {
    // Reveal in separate transaction (gas normalized)
    let commitment = get_commitment(user);
    let result = commitment % 100;
    // Process result
}
```

---

## Pattern 12: Function Value Reentrancy (Move 2.2+)

Function values (Move 2.2+) enable dynamic dispatch, which introduces reentrancy risks.

### 12.1 Understanding Reentrancy

When a function value passed to another module calls back into the originating module, resources from that module are
locked by the VM:

```move
module 0x42::lending {
    struct Pool has key { balance: u64 }

    public fun process_with_callback(pool: &mut Pool, callback: |u64|) {
        let amount = pool.balance;
        callback(amount);
        // If callback re-enters this module and modifies Pool,
        // the VM will abort with a reentrancy error
    }
}
```

### 12.2 Using #[module_lock]

Use `#[module_lock]` for strong reentrancy protection. While this function runs, ALL calls re-entering the module will
abort:

```move
#[module_lock]
public fun transfer_with_notification(
    from: address,
    to: address,
    amount: u64,
    on_transfer: |u64|
) acquires Account {
    let from_account = borrow_global_mut<Account>(from);
    from_account.balance = from_account.balance - amount;

    on_transfer(amount);  // Cannot re-enter this module

    let to_account = borrow_global_mut<Account>(to);
    to_account.balance = to_account.balance + amount;
}
```

### 12.3 When to Use #[module_lock]

| Scenario                                      | Recommendation                  |
| --------------------------------------------- | ------------------------------- |
| Function accepts callback that modifies state | Use `#[module_lock]`            |
| Function accepts callback for read-only use   | Default protection usually fine |
| DeFi protocol with external hooks             | Use `#[module_lock]`            |
| Internal-only function values                 | Default protection usually fine |

### 12.4 Stored Function Values Security

- Only store function values from trusted/authorized sources
- Always use `#[module_lock]` when executing stored function values that may modify state
- Non-public functions need `#[persistent]` attribute to gain `store` ability

---

## Pattern 13: Move Abilities

### 13.1 Ability Restrictions

```move
// Token type - NO copy (prevents double-spending)
struct Token has key {
    value: u64,
}

// Loan receipt - NO drop (prevents escaping repayment)
struct LoanReceipt has key {
    amount: u64,
    borrower: address,
}

// Hot potato pattern - NO drop, NO store
struct HotPotato {
    // Must be consumed in same transaction
}

// ✅ CORRECT: Conservative abilities
struct NFT has key {
    // key: Can be stored in global storage
    // NO copy: Can't duplicate
    // NO drop: Can't destroy arbitrarily
    name: String,
}
```

---

## Pattern 14: Publishing Key Management

**CRITICAL: Separate keys by network**

```bash
# ❌ DANGEROUS: Reusing same key
# testnet: Uses key from ~/.aptos/config.yaml
aptos move publish --network testnet

# mainnet: Same key (RISKY if testnet key leaked)
aptos move publish --network mainnet

# ✅ CORRECT: Separate key pairs
# Generate testnet key
aptos init --network testnet --profile testnet

# Generate separate mainnet key (store securely!)
aptos init --network mainnet --profile mainnet

# Publish with separate profiles
aptos move publish --profile testnet
aptos move publish --profile mainnet
```

---

## Common Vulnerabilities Table

| Vulnerability              | Example                             | Impact                        | Fix                                                                        |
| -------------------------- | ----------------------------------- | ----------------------------- | -------------------------------------------------------------------------- |
| Missing access control     | No signer verification              | Anyone calls admin functions  | Add `assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN)`      |
| Missing ownership check    | No `object::owner()` check          | Anyone modifies any object    | Add `assert!(object::owner(obj) == signer::address_of(user), E_NOT_OWNER)` |
| Cross-account manipulation | Accepts `target_addr` parameter     | User modifies others' storage | Use `borrow_global_mut<T>(signer::address_of(user))`                       |
| **Implicit amounts**       | **`coin::balance()` as amount**     | **User loses entire balance** | **Add explicit `amount: u64` parameter**                                   |
| Generic type mismatch      | No flash loan type validation       | Borrow BTC, repay junk        | Use `Receipt<phantom T>` pattern                                           |
| Unbounded iteration        | Loop over global vector             | Gas exhaustion DOS            | Store per-user, use direct lookup                                          |
| Division rounds to zero    | `(100 * 30) / 10000` = 0            | Fee-free transactions         | Enforce `MIN_ORDER_SIZE`, validate `fee > 0`                               |
| **Multi-step overflow**    | **`(a * b * c) / d`**               | **Intermediate overflow**     | **Restructure or use u128**                                                |
| Left shift overflow        | `1 << 64` (wrong value)             | Incorrect calculations        | Validate shift `< 64` or avoid                                             |
| Returning ConstructorRef   | `return constructor_ref`            | Caller destroys object        | Return `Object<T>` instead                                                 |
| Exposed &mut               | `public fun get_mut()`              | mem::swap attacks             | Expose specific operations                                                 |
| **No price oracles**       | **Assume 1:1 ratio**                | **Protocol drained**          | **Use multi-oracle design**                                                |
| Single price oracle        | Use one pool ratio                  | Price manipulation            | Multi-oracle with fallbacks                                                |
| **Stale prices**           | **No timestamp check**              | **Old prices exploited**      | **Add `MAX_PRICE_AGE` validation**                                         |
| Front-running              | Two-step operations                 | Attacker inserts between      | Atomic finalization                                                        |
| **No slippage protection** | **No `min_amount_out`**             | **MEV/sandwich attacks**      | **Add slippage parameters**                                                |
| Token ID collision         | String concatenation                | Same ID, different tokens     | Use object addresses                                                       |
| No pause mechanism         | No emergency stop                   | Can't respond to exploits     | Implement pause functionality                                              |
| **No proposal expiration** | **Proposals live forever**          | **Stale proposals executed**  | **Add expiration timestamps**                                              |
| **Threshold > owners**     | **Can't reach threshold**           | **Wallet locked**             | **Validate threshold <= owner count**                                      |
| Public randomness          | Composable lottery                  | Test-and-abort attacks        | Use `entry` visibility                                                     |
| Undergasing                | Win uses less gas                   | Attacker guarantees wins      | Balance gas across paths                                                   |
| **Callback reentrancy**    | **Function value re-enters module** | **State corruption**          | **Use `#[module_lock]` attribute**                                         |
| Untrusted stored functions | Store function from any caller      | Malicious code execution      | Only accept function values from authorized sources                        |

---

## Security Testing Requirements

### Test Coverage Checklist

```move
// Access Control Tests
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
fun test_unauthorized_access_fails(owner: &signer, attacker: &signer) { }

// Input Validation Tests
#[test(user = @0x1)]
#[expected_failure(abort_code = E_ZERO_AMOUNT)]
fun test_zero_amount_rejected(user: &signer) { }

#[test(user = @0x1)]
#[expected_failure(abort_code = E_AMOUNT_TOO_SMALL)]
fun test_small_amount_rejected(user: &signer) { }

// Arithmetic Tests
#[test(user = @0x1)]
#[expected_failure(arithmetic_error, location = Self)]
fun test_overflow_aborts(user: &signer) { }

// Edge Case Tests
#[test(user = @0x1)]
fun test_max_value_allowed(user: &signer) { }

#[test(user = @0x1)]
fun test_boundary_conditions(user: &signer) { }
```

**Required: 100% line coverage**

```bash
# Generate coverage
aptos move test --coverage

# View summary
aptos move coverage summary

# Expected: 100.0% coverage for all modules
```

---

## Additional Resources

**Official Documentation:**

- [Aptos Move Security Guidelines](https://aptos.dev/build/smart-contracts/move-security-guidelines) ⭐
- [Object Model](https://aptos.dev/build/smart-contracts/object)
- [Move Book](https://aptos.dev/build/smart-contracts/book)

**Related Patterns:**

- `OBJECTS.md` - Object model best practices
- `TESTING.md` - Security testing patterns
- `DIGITAL_ASSETS.md` - NFT security considerations

---

**Remember:** Security is not optional. Every checklist item must pass before deployment. User funds depend on your
code's correctness. When in doubt, be more restrictive.
