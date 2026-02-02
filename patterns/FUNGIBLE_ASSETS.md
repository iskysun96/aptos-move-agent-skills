# Aptos Fungible Asset Standard

**Purpose:** Comprehensive guide to the Aptos Fungible Asset (FA) standard for tokens and coins.

**Target:** AI assistants generating token-related Move V2 smart contracts.

---

## Overview

The **Fungible Asset (FA) standard** is the modern token framework for Aptos, replacing the legacy `coin` module.

**Key Benefits:**

- 🚀 **No recipient opt-in required** - unlike legacy coin which needs `coin::register()`
- ✅ **Object-based architecture** - leverages Move Objects for composability
- 🎨 **Custom transfer logic** - dispatchable functions for hooks
- 🔒 **Better security** - permission refs (MintRef, BurnRef, TransferRef) stored securely
- 📊 **Primary stores** - automatic per-user storage without registration

**Deployed at:** `0x1::fungible_asset` (core framework module)

---

## Core Concepts

### Metadata Object

The **Metadata Object** defines the token type (like a "coin type" in legacy):

- Token name (e.g., "My Token")
- Symbol (e.g., "MTK")
- Decimals (e.g., 8 for BTC-style, 6 for USDC-style)
- Icon URI and project URI
- Maximum supply (optional - none means unlimited)

**Type:** `Object<Metadata>`

### FungibleStore

A **FungibleStore** is where tokens are stored for each account:

- **Primary Store**: Automatically created per account (default for most use cases)
- **Secondary Stores**: Custom stores for specific use cases (escrow, vaults)

### Permission Refs

Permission refs control critical token operations:

- **MintRef**: Create new tokens
- **BurnRef**: Destroy tokens
- **TransferRef**: Control transfer logic (for non-transferable tokens)

**CRITICAL:** Store these refs securely in module resources with proper access control.

---

## Required Imports

For Fungible Asset contracts, always import these modules:

```move
use std::option::{Self, Option};
use std::string::{Self, String};
use std::signer;

use aptos_framework::object::{Self, Object, ConstructorRef, ExtendRef};
use aptos_framework::event;

// ✅ Fungible Asset standard modules
use aptos_framework::fungible_asset::{Self, Metadata, FungibleAsset, MintRef, BurnRef, TransferRef};
use aptos_framework::primary_fungible_store;
```

---

## Creating Fungible Assets

### Pattern 1: Basic FA (Unlimited Supply)

Use when tokens can be minted indefinitely (e.g., reward points, utility tokens):

```move
module my_addr::my_token {
    use std::option;
    use std::string::{Self, String};
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef, BurnRef, TransferRef};
    use aptos_framework::primary_fungible_store;

    /// Holds permission refs for the token
    struct ManagingRefs has key {
        mint_ref: MintRef,
        burn_ref: BurnRef,
        transfer_ref: TransferRef,
    }

    /// Error codes
    const E_NOT_ADMIN: u64 = 1;

    /// Initialize token on module deployment
    fun init_module(deployer: &signer) {
        // Create metadata object for the token
        let constructor = object::create_named_object(deployer, b"MY_TOKEN");
        let constructor_ref = &constructor;

        // Create primary store enabled fungible asset
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::none(), // max_supply (none = unlimited)
            string::utf8(b"My Token"),
            string::utf8(b"MTK"),
            8, // decimals
            string::utf8(b"https://mytoken.io/icon.png"),
            string::utf8(b"https://mytoken.io"),
        );

        // Generate permission refs
        let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(constructor_ref);
        let transfer_ref = fungible_asset::generate_transfer_ref(constructor_ref);

        // Store refs in metadata object
        let metadata_signer = object::generate_signer(constructor_ref);
        move_to(&metadata_signer, ManagingRefs {
            mint_ref,
            burn_ref,
            transfer_ref,
        });
    }

    /// Get metadata object address
    #[view]
    public fun get_metadata(): Object<Metadata> {
        let metadata_addr = object::create_object_address(&@my_addr, b"MY_TOKEN");
        object::address_to_object<Metadata>(metadata_addr)
    }

    /// Mint tokens (admin only)
    public entry fun mint(admin: &signer, recipient: address, amount: u64) acquires ManagingRefs {
        // Verify admin authorization
        assert!(signer::address_of(admin) == @my_addr, E_NOT_ADMIN);

        let metadata = get_metadata();
        let metadata_addr = object::object_address(&metadata);
        let refs = borrow_global<ManagingRefs>(metadata_addr);

        // Mint and deposit to recipient's primary store
        let fa = fungible_asset::mint(&refs.mint_ref, amount);
        primary_fungible_store::deposit(recipient, fa);
    }

    /// Transfer tokens
    public entry fun transfer(sender: &signer, recipient: address, amount: u64) {
        let metadata = get_metadata();
        primary_fungible_store::transfer(sender, metadata, recipient, amount);
    }

    /// Burn tokens from sender's account
    public entry fun burn(sender: &signer, amount: u64) acquires ManagingRefs {
        let metadata = get_metadata();
        let metadata_addr = object::object_address(&metadata);
        let refs = borrow_global<ManagingRefs>(metadata_addr);

        // Withdraw from sender and burn
        let fa = primary_fungible_store::withdraw(sender, metadata, amount);
        fungible_asset::burn(&refs.burn_ref, fa);
    }

    /// Get balance of an account
    #[view]
    public fun balance(account: address): u64 {
        let metadata = get_metadata();
        primary_fungible_store::balance(account, metadata)
    }
}
```

### Pattern 2: Fixed Supply FA

Use when you have a maximum token supply (e.g., fixed supply coins):

```move
module my_addr::fixed_token {
    use std::option;
    use std::string;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef, BurnRef};
    use aptos_framework::primary_fungible_store;

    const MAX_SUPPLY: u64 = 1_000_000; // 1 million tokens
    const DECIMALS: u8 = 8;

    /// Error codes
    const E_NOT_ADMIN: u64 = 1;

    struct ManagingRefs has key {
        mint_ref: MintRef,
        burn_ref: BurnRef,
    }

    fun init_module(deployer: &signer) {
        let constructor = object::create_named_object(deployer, b"FIXED_TOKEN");
        let constructor_ref = &constructor;

        // Create with max supply
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::some(MAX_SUPPLY * pow(10, (DECIMALS as u64))), // max_supply with decimals
            string::utf8(b"Fixed Supply Token"),
            string::utf8(b"FST"),
            DECIMALS,
            string::utf8(b"https://fixed.io/icon.png"),
            string::utf8(b"https://fixed.io"),
        );

        let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(constructor_ref);

        let metadata_signer = object::generate_signer(constructor_ref);
        move_to(&metadata_signer, ManagingRefs { mint_ref, burn_ref });
    }

    /// Mint will abort if exceeds max supply
    public entry fun mint(admin: &signer, recipient: address, amount: u64) acquires ManagingRefs {
        assert!(signer::address_of(admin) == @my_addr, E_NOT_ADMIN);

        let metadata = get_metadata();
        let refs = borrow_global<ManagingRefs>(object::object_address(&metadata));

        // This will abort with EMAX_SUPPLY_EXCEEDED if exceeds max
        let fa = fungible_asset::mint(&refs.mint_ref, amount);
        primary_fungible_store::deposit(recipient, fa);
    }

    #[view]
    public fun get_metadata(): Object<Metadata> {
        let metadata_addr = object::create_object_address(&@my_addr, b"FIXED_TOKEN");
        object::address_to_object<Metadata>(metadata_addr)
    }

    fun pow(base: u64, exp: u64): u64 {
        let result = 1;
        let i = 0;
        while (i < exp) {
            result = result * base;
            i = i + 1;
        };
        result
    }
}
```

### Pattern 3: Dispatchable FA (Custom Transfer Hooks)

Use when you need custom logic on transfers (e.g., pausable tokens, whitelisted transfers):

```move
module my_addr::pausable_token {
    use std::option;
    use std::string;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef, BurnRef, TransferRef, FungibleAsset};
    use aptos_framework::primary_fungible_store;
    use aptos_framework::function_info;
    use aptos_framework::dispatchable_fungible_asset;

    struct ManagingRefs has key {
        mint_ref: MintRef,
        burn_ref: BurnRef,
        transfer_ref: TransferRef,
    }

    struct PauseState has key {
        paused: bool,
        admin: address,
    }

    const E_PAUSED: u64 = 1;
    const E_NOT_ADMIN: u64 = 2;

    fun init_module(deployer: &signer) {
        let constructor = object::create_named_object(deployer, b"PAUSABLE_TOKEN");
        let constructor_ref = &constructor;

        // Create primary store enabled fungible asset
        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::none(),
            string::utf8(b"Pausable Token"),
            string::utf8(b"PST"),
            8,
            string::utf8(b"https://pausable.io/icon.png"),
            string::utf8(b"https://pausable.io"),
        );

        // Register dispatch functions for custom transfer logic
        let withdraw_function = function_info::new_function_info(
            deployer,
            string::utf8(b"pausable_token"),
            string::utf8(b"withdraw"),
        );

        let deposit_function = function_info::new_function_info(
            deployer,
            string::utf8(b"pausable_token"),
            string::utf8(b"deposit"),
        );

        dispatchable_fungible_asset::register_dispatch_functions(
            constructor_ref,
            option::some(withdraw_function),
            option::some(deposit_function),
            option::none(), // No custom derived balance
        );

        let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(constructor_ref);
        let transfer_ref = fungible_asset::generate_transfer_ref(constructor_ref);

        let metadata_signer = object::generate_signer(constructor_ref);
        move_to(&metadata_signer, ManagingRefs { mint_ref, burn_ref, transfer_ref });
        move_to(&metadata_signer, PauseState {
            paused: false,
            admin: signer::address_of(deployer),
        });
    }

    /// Custom withdraw hook - checks if paused
    public fun withdraw<T: key>(
        store: Object<T>,
        amount: u64,
        transfer_ref: &TransferRef,
    ): FungibleAsset acquires PauseState {
        let metadata = get_metadata();
        let pause_state = borrow_global<PauseState>(object::object_address(&metadata));
        assert!(!pause_state.paused, E_PAUSED);

        fungible_asset::withdraw_with_ref(transfer_ref, store, amount)
    }

    /// Custom deposit hook - checks if paused
    public fun deposit<T: key>(
        store: Object<T>,
        fa: FungibleAsset,
        transfer_ref: &TransferRef,
    ) acquires PauseState {
        let metadata = get_metadata();
        let pause_state = borrow_global<PauseState>(object::object_address(&metadata));
        assert!(!pause_state.paused, E_PAUSED);

        fungible_asset::deposit_with_ref(transfer_ref, store, fa);
    }

    /// Pause transfers
    public entry fun pause(admin: &signer) acquires PauseState {
        let metadata = get_metadata();
        let pause_state = borrow_global_mut<PauseState>(object::object_address(&metadata));
        assert!(signer::address_of(admin) == pause_state.admin, E_NOT_ADMIN);
        pause_state.paused = true;
    }

    /// Unpause transfers
    public entry fun unpause(admin: &signer) acquires PauseState {
        let metadata = get_metadata();
        let pause_state = borrow_global_mut<PauseState>(object::object_address(&metadata));
        assert!(signer::address_of(admin) == pause_state.admin, E_NOT_ADMIN);
        pause_state.paused = false;
    }

    #[view]
    public fun get_metadata(): Object<Metadata> {
        let metadata_addr = object::create_object_address(&@my_addr, b"PAUSABLE_TOKEN");
        object::address_to_object<Metadata>(metadata_addr)
    }
}
```

---

## Common Operations

### Transfer Tokens

```move
/// Transfer from sender to recipient
public entry fun transfer(sender: &signer, recipient: address, amount: u64) {
    let metadata = get_metadata(); // Get your token's metadata object
    primary_fungible_store::transfer(sender, metadata, recipient, amount);
}
```

### Check Balance

```move
#[view]
public fun balance(account: address): u64 {
    let metadata = get_metadata();
    primary_fungible_store::balance(account, metadata)
}
```

### Mint Tokens

```move
public entry fun mint(admin: &signer, recipient: address, amount: u64) acquires ManagingRefs {
    // Authorization check
    assert!(is_admin(signer::address_of(admin)), E_NOT_ADMIN);

    let metadata = get_metadata();
    let refs = borrow_global<ManagingRefs>(object::object_address(&metadata));

    // Mint and deposit
    let fa = fungible_asset::mint(&refs.mint_ref, amount);
    primary_fungible_store::deposit(recipient, fa);
}
```

### Burn Tokens

```move
public entry fun burn_from_sender(sender: &signer, amount: u64) acquires ManagingRefs {
    let metadata = get_metadata();
    let refs = borrow_global<ManagingRefs>(object::object_address(&metadata));

    // Withdraw from sender
    let fa = primary_fungible_store::withdraw(sender, metadata, amount);

    // Burn
    fungible_asset::burn(&refs.burn_ref, fa);
}
```

### Withdraw and Deposit (For custom logic)

```move
/// Withdraw tokens to FungibleAsset
public fun withdraw_fa(account: &signer, amount: u64): FungibleAsset {
    let metadata = get_metadata();
    primary_fungible_store::withdraw(account, metadata, amount)
}

/// Deposit FungibleAsset to account
public fun deposit_fa(account: address, fa: FungibleAsset) {
    primary_fungible_store::deposit(account, fa);
}
```

---

## ALWAYS Rules

When working with Fungible Assets, you MUST:

### Token Creation

- ✅ **ALWAYS use** `Object<Metadata>` for token type references
- ✅ **ALWAYS create metadata in init_module()** for deployment-time token setup
- ✅ **ALWAYS use** `primary_fungible_store::create_primary_store_enabled_fungible_asset()` for standard tokens
- ✅ **ALWAYS use named objects** for deterministic metadata addresses:
  `object::create_named_object(deployer, b"TOKEN_NAME")`
- ✅ **ALWAYS specify decimals** appropriately (8 for BTC-style, 6 for USDC-style, 18 for ETH-style)

### Permission Refs

- ✅ **ALWAYS store permission refs** (MintRef, BurnRef, TransferRef) in module resources
- ✅ **ALWAYS protect mint/burn operations** with proper authorization checks
- ✅ **ALWAYS store refs in the metadata object** account (use `object::generate_signer(constructor_ref)`)
- ✅ **NEVER expose refs publicly** - store them in private structs with `has key`

### Transfers

- ✅ **ALWAYS use** `primary_fungible_store::transfer()` for standard transfers
- ✅ **ALWAYS check balance** before withdrawing: `primary_fungible_store::balance(account, metadata) >= amount`
- ✅ **ALWAYS deposit FungibleAsset** after withdrawing (don't leave it dangling)

### Queries

- ✅ **ALWAYS provide a get_metadata() view function** for clients to query token metadata
- ✅ **ALWAYS provide a balance() view function** for easy balance queries
- ✅ **ALWAYS use** `#[view]` attribute for read-only functions

---

## NEVER Rules

When working with Fungible Assets, you MUST NEVER:

### Legacy Patterns

- ❌ **NEVER use legacy coin module** for new token contracts (use FA instead)
- ❌ **NEVER import** `aptos_framework::coin` for new tokens
- ❌ **NEVER require recipient opt-in** (FA doesn't need `coin::register()`)
- ❌ **NEVER use** `coin::transfer()` or `coin::deposit()` for FA tokens

### Security Violations

- ❌ **NEVER expose MintRef/BurnRef publicly** without authorization
- ❌ **NEVER allow arbitrary minting** without proper admin checks
- ❌ **NEVER store refs in user accounts** (store in metadata object account)
- ❌ **NEVER forget to burn withdrawn FungibleAsset** if not depositing elsewhere (causes asset loss)

### Bad Practices

- ❌ **NEVER create tokens without decimals** (always specify decimals explicitly)
- ❌ **NEVER use raw addresses** for metadata (use `Object<Metadata>`)
- ❌ **NEVER skip max supply** for fixed supply tokens (prevents infinite minting)
- ❌ **NEVER mutate supply** outside of mint/burn refs (supply is tracked automatically)

---

## Common Patterns

### Pattern: Token with Admin Controls

```move
module my_addr::admin_token {
    use std::option;
    use std::string;
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef, BurnRef};
    use aptos_framework::primary_fungible_store;

    struct TokenRefs has key {
        mint_ref: MintRef,
        burn_ref: BurnRef,
    }

    struct AdminConfig has key {
        admin: address,
    }

    const E_NOT_ADMIN: u64 = 1;

    fun init_module(deployer: &signer) {
        let constructor = object::create_named_object(deployer, b"ADMIN_TOKEN");
        let constructor_ref = &constructor;

        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::none(),
            string::utf8(b"Admin Token"),
            string::utf8(b"ADM"),
            8,
            string::utf8(b"https://admin.io/icon.png"),
            string::utf8(b"https://admin.io"),
        );

        let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
        let burn_ref = fungible_asset::generate_burn_ref(constructor_ref);

        let metadata_signer = object::generate_signer(constructor_ref);
        move_to(&metadata_signer, TokenRefs { mint_ref, burn_ref });
        move_to(&metadata_signer, AdminConfig {
            admin: signer::address_of(deployer),
        });
    }

    /// Admin-only mint
    public entry fun mint(admin: &signer, recipient: address, amount: u64) acquires TokenRefs, AdminConfig {
        let metadata = get_metadata();
        let metadata_addr = object::object_address(&metadata);

        // Verify admin
        let config = borrow_global<AdminConfig>(metadata_addr);
        assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);

        // Mint
        let refs = borrow_global<TokenRefs>(metadata_addr);
        let fa = fungible_asset::mint(&refs.mint_ref, amount);
        primary_fungible_store::deposit(recipient, fa);
    }

    /// Transfer admin rights
    public entry fun transfer_admin(current_admin: &signer, new_admin: address) acquires AdminConfig {
        let metadata = get_metadata();
        let config = borrow_global_mut<AdminConfig>(object::object_address(&metadata));
        assert!(signer::address_of(current_admin) == config.admin, E_NOT_ADMIN);
        config.admin = new_admin;
    }

    #[view]
    public fun get_metadata(): Object<Metadata> {
        let metadata_addr = object::create_object_address(&@my_addr, b"ADMIN_TOKEN");
        object::address_to_object<Metadata>(metadata_addr)
    }
}
```

### Pattern: Get Token Metadata Info

```move
#[view]
public fun get_token_info(): (String, String, u8, Option<u128>) {
    let metadata = get_metadata();
    let name = fungible_asset::name(metadata);
    let symbol = fungible_asset::symbol(metadata);
    let decimals = fungible_asset::decimals(metadata);
    let max_supply = fungible_asset::maximum(metadata);

    (name, symbol, decimals, max_supply)
}

#[view]
public fun get_supply(): Option<u128> {
    let metadata = get_metadata();
    fungible_asset::supply(metadata)
}
```

### Pattern: Airdrop Tokens

```move
public entry fun airdrop(
    admin: &signer,
    recipients: vector<address>,
    amounts: vector<u64>
) acquires ManagingRefs {
    assert!(vector::length(&recipients) == vector::length(&amounts), E_LENGTH_MISMATCH);
    assert!(is_admin(signer::address_of(admin)), E_NOT_ADMIN);

    let metadata = get_metadata();
    let refs = borrow_global<ManagingRefs>(object::object_address(&metadata));

    let i = 0;
    let len = vector::length(&recipients);
    while (i < len) {
        let recipient = *vector::borrow(&recipients, i);
        let amount = *vector::borrow(&amounts, i);

        let fa = fungible_asset::mint(&refs.mint_ref, amount);
        primary_fungible_store::deposit(recipient, fa);

        i = i + 1;
    };
}
```

---

## Migration from Coin

If you have legacy coin contracts, migrate to FA for better UX:

### Legacy Coin Pattern

```move
// ❌ OLD - Legacy coin (requires recipient registration)
module my_addr::old_coin {
    use aptos_framework::coin;

    struct MyCoin {}

    fun init_module(account: &signer) {
        let (burn_cap, freeze_cap, mint_cap) = coin::initialize<MyCoin>(
            account,
            string::utf8(b"My Coin"),
            string::utf8(b"MC"),
            8,
            true,
        );
        // ... store capabilities
    }

    // ❌ Recipient must register first!
    public entry fun transfer(from: &signer, to: address, amount: u64) {
        coin::transfer<MyCoin>(from, to, amount);
    }
}
```

### Migrated FA Pattern

```move
// ✅ NEW - Fungible Asset (no registration needed)
module my_addr::new_token {
    use aptos_framework::fungible_asset::{Self, Metadata, MintRef};
    use aptos_framework::primary_fungible_store;
    use aptos_framework::object::{Self, Object};

    struct TokenRefs has key {
        mint_ref: MintRef,
    }

    fun init_module(deployer: &signer) {
        let constructor = object::create_named_object(deployer, b"MY_TOKEN");
        let constructor_ref = &constructor;

        primary_fungible_store::create_primary_store_enabled_fungible_asset(
            constructor_ref,
            option::none(),
            string::utf8(b"My Token"),
            string::utf8(b"MT"),
            8,
            string::utf8(b"https://icon.png"),
            string::utf8(b"https://project.io"),
        );

        let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
        let metadata_signer = object::generate_signer(constructor_ref);
        move_to(&metadata_signer, TokenRefs { mint_ref });
    }

    // ✅ No registration needed!
    public entry fun transfer(sender: &signer, recipient: address, amount: u64) {
        let metadata = get_metadata();
        primary_fungible_store::transfer(sender, metadata, recipient, amount);
    }

    #[view]
    public fun get_metadata(): Object<Metadata> {
        let metadata_addr = object::create_object_address(&@my_addr, b"MY_TOKEN");
        object::address_to_object<Metadata>(metadata_addr)
    }
}
```

---

## Testing Fungible Assets

### Basic Operation Tests

```move
#[test_only]
module my_addr::token_tests {
    use std::signer;
    use aptos_framework::primary_fungible_store;
    use my_addr::my_token;

    #[test(deployer = @my_addr, user1 = @0x100, user2 = @0x200)]
    public fun test_mint_transfer_burn(
        deployer: &signer,
        user1: &signer,
        user2: &signer
    ) {
        // Initialize
        // Initialize token (use test-only wrapper since init_module is private)
        // Inside my_token module, define: #[test_only] public fun init_for_test(deployer: &signer) { init_module(deployer); }
        my_token::init_for_test(deployer);

        let user1_addr = signer::address_of(user1);
        let user2_addr = signer::address_of(user2);
        let metadata = my_token::get_metadata();

        // Test mint
        my_token::mint(deployer, user1_addr, 1000);
        assert!(primary_fungible_store::balance(user1_addr, metadata) == 1000, 0);

        // Test transfer
        my_token::transfer(user1, user2_addr, 400);
        assert!(primary_fungible_store::balance(user1_addr, metadata) == 600, 1);
        assert!(primary_fungible_store::balance(user2_addr, metadata) == 400, 2);

        // Test burn
        my_token::burn(user1, 100);
        assert!(primary_fungible_store::balance(user1_addr, metadata) == 500, 3);
    }

    #[test(deployer = @my_addr)]
    #[expected_failure]
    public fun test_zero_transfer_fails(deployer: &signer) {
        // Initialize token (use test-only wrapper since init_module is private)
        // Inside my_token module, define: #[test_only] public fun init_for_test(deployer: &signer) { init_module(deployer); }
        my_token::init_for_test(deployer);
        my_token::transfer(deployer, @0x100, 0); // Should fail
    }

    #[test(deployer = @my_addr, attacker = @0x999)]
    #[expected_failure(abort_code = my_token::E_NOT_ADMIN)]
    public fun test_unauthorized_mint_fails(deployer: &signer, attacker: &signer) {
        // Initialize token (use test-only wrapper since init_module is private)
        // Inside my_token module, define: #[test_only] public fun init_for_test(deployer: &signer) { init_module(deployer); }
        my_token::init_for_test(deployer);
        my_token::mint(attacker, @0x100, 1000); // Should fail
    }
}
```

### Max Supply Tests

```move
#[test(deployer = @my_addr)]
#[expected_failure(abort_code = aptos_framework::fungible_asset::EMAX_SUPPLY_EXCEEDED)]
public fun test_cannot_exceed_max_supply(deployer: &signer) {
    // Initialize token (use test-only wrapper since init_module is private)
    // Inside my_token module, define: #[test_only] public fun init_for_test(deployer: &signer) { init_module(deployer); }
    my_token::init_for_test(deployer);

    // Assuming max supply is 1,000,000 with 8 decimals
    let max_plus_one = 1_000_001 * 100_000_000;
    my_token::mint(deployer, signer::address_of(deployer), max_plus_one);
}
```

### Pausable Token Tests

```move
#[test(deployer = @my_addr, user = @0x100)]
#[expected_failure(abort_code = E_PAUSED)]
public fun test_paused_transfer_fails(deployer: &signer, user: &signer) {
    // Initialize token (use test-only wrapper since init_module is private)
    // Inside pausable_token module, define: #[test_only] public fun init_for_test(deployer: &signer) { init_module(deployer); }
    pausable_token::init_for_test(deployer);

    // Mint some tokens
    pausable_token::mint(deployer, signer::address_of(user), 1000);

    // Pause
    pausable_token::pause(deployer);

    // Try transfer (should fail)
    pausable_token::transfer(user, @0x200, 100);
}
```

---

## References

- **Official Docs:** [Aptos Fungible Asset](https://aptos.dev/standards/fungible-asset/)
- **Framework Code:**
  [aptos-core/aptos-move/framework/aptos-framework/sources/fungible_asset.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/fungible_asset.move)
- **Examples:**
  [aptos-core/aptos-move/move-examples/fungible_asset](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples/fungible_asset)
- **Primary Store:**
  [aptos-core/aptos-move/framework/aptos-framework/sources/primary_fungible_store.move](https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/aptos-framework/sources/primary_fungible_store.move)
