---
name: write-contracts
description:
  Generate and refactor Aptos Move V2 smart contracts following object-centric patterns, modern syntax, and security
  best practices. Use when "write move contract", "create smart contract", "build module", "refactor move code",
  "implement move function".
---

# Write Contracts Skill

## Overview

This skill guides you in writing secure, modern Aptos Move V2 smart contracts. Always follow this workflow:

1. **Search first**: Check aptos-core/move-examples for similar patterns
2. **Use objects**: Always use `Object<T>` references (never raw addresses)
3. **Security first**: Verify signers, validate inputs, protect references
4. **Modern syntax**: Use inline functions, lambdas, current object model
5. **Document clearly**: Add clear error codes and comments

## Pre-Generation Checklist ⚠️ MANDATORY

**STOP - Answer these questions before writing ANY contract code:**

1. **Does this contract involve NFTs?** (collections, marketplaces, minting, trading)
   - ✅ YES → You MUST use Digital Asset standard (see Step 2.5 below)
   - ❌ NO → Continue

2. **Does this contract involve fungible tokens?** (coins, tokens, currencies, reward points)
   - ✅ YES → You MUST use Fungible Asset standard (see Step 2.6 below)
   - ❌ NO → Continue

3. **Does this contract need global configuration?** (fees, admin settings, registry)
   - ✅ YES → You MUST use init_module() for deployment initialization
   - ❌ NO → Use public entry functions

## Core Workflow

### Step 1: Search for Examples

Before writing any new contract:

1. Search aptos-core/aptos-move/move-examples for similar functionality
2. Priority examples:
   - `hello_blockchain` - Basic module structure
   - `mint_nft` - NFT creation patterns
   - `token_objects` - Object-centric tokens
   - `fungible_asset` - Fungible token patterns
   - `dao` - Governance and voting
   - `marketplace` - Trading and escrow
3. Review official docs at https://aptos.dev/build/smart-contracts

### Step 2: Define Module Structure

```move
module my_addr::my_module {
    // ============ Imports ============
    use std::signer;
    use std::string::String;
    use std::vector;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::event;

    // ============ Events ============
    #[event]
    struct ItemCreated has drop, store {
        item: address,
        creator: address,
        name: String,
    }

    #[event]
    struct ItemTransferred has drop, store {
        item: address,
        from: address,
        to: address,
    }

    // ============ Structs ============
    // Define your data structures

    // ============ Constants ============
    // Define error codes and constants

    // ============ Init Module ============
    // REQUIRED for contract initialization on deployment
    // Called ONCE when module is first published
    fun init_module(deployer: &signer) {
        // Initialize global state, registries, etc.
        // Example: create singleton registry, set admin, etc.
    }

    // ============ Public Entry Functions ============
    // User-facing functions

    // ============ Public Functions ============
    // Composable functions

    // ============ Private Functions ============
    // Internal helpers
}
```

### Step 2.5: Identify Token Standard - Digital Assets (NFTs) ⭐ CRITICAL

**IF YOUR CONTRACT INVOLVES NFTs** (collections, marketplaces, minting, trading):

⚠️ STOP AND READ THIS SECTION CAREFULLY ⚠️

**CRITICAL TYPE RULE:**

```move
// ✅ CORRECT - Use this type for ALL NFT references
use aptos_token_objects::aptos_token::{Self, AptosToken};

public entry fun list_nft(
    seller: &signer,
    nft: Object<AptosToken>,  // ← AptosToken type!
    price: u64
) { ... }

// ❌ WRONG - DO NOT USE (causes violations)
use aptos_token_objects::token;

public entry fun list_nft(
    seller: &signer,
    nft: Object<token::Token>,  // ← Generic Token - VIOLATION!
    price: u64
) { ... }
```

**Required Imports - Copy exactly:**

```move
use aptos_token_objects::collection;
use aptos_token_objects::token;
use aptos_token_objects::aptos_token::{Self, AptosToken};  // ← Import AptosToken!
use aptos_token_objects::royalty::{Self, Royalty};
```

**NFT Marketplace Pattern:**

For NFT marketplaces, you MUST:

1. Use `init_module()` (NOT public `initialize()`)
2. Marketplace object owns the collection
3. Use extend_ref to mint as marketplace signer

```move
fun init_module(deployer: &signer) {
    // Create marketplace state object
    let marketplace_ref = object::create_named_object(deployer, b"MARKETPLACE_STATE");
    let marketplace_signer = object::generate_signer(&marketplace_ref);

    // Marketplace object creates collection
    collection::create_unlimited_collection(
        &marketplace_signer,  // ← Marketplace signer owns collection
        string::utf8(b"Collection description"),
        string::utf8(b"Collection Name"),
        option::none(),
        string::utf8(b"https://uri.com"),
    );

    // Store config with extend_ref
    move_to(&marketplace_signer, MarketplaceConfig {
        admin: signer::address_of(deployer),
        extend_ref: object::generate_extend_ref(&marketplace_ref),
    });
}

public entry fun mint_nft(creator: &signer, name: String, uri: String) acquires MarketplaceConfig {
    let config = borrow_global<MarketplaceConfig>(get_marketplace_addr());
    let marketplace_signer = object::generate_signer_for_extending(&config.extend_ref);

    // Mint with marketplace signer
    token::create_named_token(
        &marketplace_signer,  // ← Must be collection owner
        string::utf8(b"Collection Name"),
        string::utf8(b"Description"),
        name,
        option::none(),
        uri,
    );
}
```

**See:** `../../patterns/DIGITAL_ASSETS.md` lines 381-589 for complete marketplace example

### Step 2.6: Identify Token Standard - Fungible Assets (Tokens/Coins) ⭐ CRITICAL

**IF YOUR CONTRACT INVOLVES FUNGIBLE TOKENS** (coins, tokens, currencies, points):

⚠️ STOP AND READ THIS SECTION CAREFULLY ⚠️

**CRITICAL TYPE RULE:**

```move
// ✅ CORRECT - Use Fungible Asset standard
use aptos_framework::fungible_asset::{Self, Metadata, FungibleAsset};

public entry fun transfer(sender: &signer, recipient: address, amount: u64) {
    let metadata = get_token_metadata();  // Returns Object<Metadata>
    primary_fungible_store::transfer(sender, metadata, recipient, amount);
}

// ❌ WRONG - DO NOT use legacy coin module for new tokens
use aptos_framework::coin;  // ← Deprecated for new tokens
```

**Required Imports:**

```move
use aptos_framework::fungible_asset::{Self, Metadata, FungibleAsset, MintRef, BurnRef};
use aptos_framework::primary_fungible_store;
use aptos_framework::object::{Self, Object};
```

**Token Creation Pattern (in init_module):**

```move
fun init_module(deployer: &signer) {
    let constructor_ref = &object::create_named_object(deployer, b"MY_TOKEN");

    primary_fungible_store::create_primary_store_enabled_fungible_asset(
        constructor_ref,
        option::none(), // max_supply (none = unlimited)
        string::utf8(b"My Token"),
        string::utf8(b"MTK"),
        8, // decimals
        string::utf8(b"https://icon.png"),
        string::utf8(b"https://project.com"),
    );

    // Store permission refs
    let mint_ref = fungible_asset::generate_mint_ref(constructor_ref);
    let burn_ref = fungible_asset::generate_burn_ref(constructor_ref);
    // ... store refs securely
}
```

**See:** `../../patterns/FUNGIBLE_ASSETS.md` for complete FA patterns

### Step 3: Implement Object Creation

**ALWAYS use this pattern for creating objects:**

```move
struct MyObject has key {
    name: String,
    // Store refs for later use
    transfer_ref: object::TransferRef,
    delete_ref: object::DeleteRef,
}

/// Create object with proper pattern
public fun create_my_object(creator: &signer, name: String): Object<MyObject> {
    // 1. Create object
    let constructor_ref = object::create_object(signer::address_of(creator));

    // 2. Generate ALL refs you'll need BEFORE constructor_ref is destroyed
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);

    // 3. Get object signer
    let object_signer = object::generate_signer(&constructor_ref);

    // 4. Store data in object
    move_to(&object_signer, MyObject {
        name,
        transfer_ref,
        delete_ref,
    });

    // 5. Return typed object reference (ConstructorRef automatically destroyed)
    object::object_from_constructor_ref<MyObject>(&constructor_ref)
}
```

### Step 4: Add Access Control

**ALWAYS verify authorization in entry functions:**

```move
const E_NOT_OWNER: u64 = 1;
const E_NOT_ADMIN: u64 = 2;

/// Entry function with ownership verification
public entry fun update_object(
    owner: &signer,
    obj: Object<MyObject>,
    new_name: String
) acquires MyObject {
    // ✅ ALWAYS: Verify ownership
    assert!(object::owner(obj) == signer::address_of(owner), E_NOT_OWNER);

    // ✅ ALWAYS: Validate inputs
    assert!(string::length(&new_name) > 0, E_INVALID_INPUT);
    assert!(string::length(&new_name) <= MAX_NAME_LENGTH, E_NAME_TOO_LONG);

    // Safe to proceed
    let obj_data = borrow_global_mut<MyObject>(object::object_address(&obj));
    obj_data.name = new_name;
}
```

### Step 5: Implement Operations

**Use modern patterns for all operations:**

```move
/// Transfer with ownership check
public entry fun transfer_object(
    owner: &signer,
    obj: Object<MyObject>,
    recipient: address
) acquires MyObject {
    // Verify ownership
    assert!(object::owner(obj) == signer::address_of(owner), E_NOT_OWNER);

    // Validate recipient
    assert!(recipient != @0x0, E_ZERO_ADDRESS);

    // Transfer using stored ref
    let obj_data = borrow_global<MyObject>(object::object_address(&obj));
    object::transfer_with_ref(
        object::generate_linear_transfer_ref(&obj_data.transfer_ref),
        recipient
    );
}

/// Delete with ownership check
public entry fun burn_object(owner: &signer, obj: Object<MyObject>) acquires MyObject {
    // Verify ownership
    assert!(object::owner(obj) == signer::address_of(owner), E_NOT_OWNER);

    // Extract data and delete
    let obj_addr = object::object_address(&obj);
    let MyObject {
        name: _,
        transfer_ref: _,
        delete_ref,
    } = move_from<MyObject>(obj_addr);

    object::delete(delete_ref);
}
```

## ALWAYS Rules

When writing Move contracts, you MUST:

### Digital Assets (NFTs) ⭐ CRITICAL

- ✅ **ALWAYS use** `Object<AptosToken>` for NFT references (NOT `Object<token::Token>`, NOT generic `Object<T>`)
- ✅ **ALWAYS import** `aptos_token_objects::aptos_token::{Self, AptosToken}`
- ✅ **ALWAYS use Digital Asset (DA) standard** for ALL NFT-related contracts
- ✅ **ALWAYS create collections** with `collection::create_fixed_collection()` or
  `collection::create_unlimited_collection()`
- ✅ **ALWAYS use init_module()** for marketplace contracts (NOT public `initialize()`)
- ✅ **ALWAYS have marketplace object own collection** (use extend_ref to mint)
- ✅ See `../../patterns/DIGITAL_ASSETS.md` for complete patterns

### Fungible Assets (Tokens/Coins) ⭐ CRITICAL

- ✅ **ALWAYS use Fungible Asset (FA) standard** for ALL token/coin contracts
- ✅ **ALWAYS use** `Object<Metadata>` for token type references
- ✅ **ALWAYS import** `aptos_framework::fungible_asset` and `primary_fungible_store`
- ✅ **ALWAYS use init_module()** to create metadata object
- ✅ **ALWAYS enable primary store** with `create_primary_store_enabled_fungible_asset()`
- ✅ **ALWAYS store permission refs** (MintRef, BurnRef, TransferRef) securely
- ✅ See `../../patterns/FUNGIBLE_ASSETS.md` for complete patterns

### Object Model

- ✅ Use `Object<T>` for all object references (NOT addresses)
- ✅ Generate all refs (TransferRef, DeleteRef) in constructor before ConstructorRef destroyed
- ✅ Return `Object<T>` from constructor functions (NEVER return ConstructorRef)
- ✅ Use `object::owner(obj)` to verify ownership
- ✅ Use `object::generate_signer(&constructor_ref)` for object signers

### Security ⭐ CRITICAL - See [SECURITY.md](../../patterns/SECURITY.md)

- ✅ **Verify signer authority** in ALL entry functions: `assert!(signer::address_of(user) == expected, E_UNAUTHORIZED)`
- ✅ **Verify object ownership**: `assert!(object::owner(obj) == signer::address_of(user), E_NOT_OWNER)`
- ✅ **Scope global storage** to signer: `borrow_global_mut<T>(signer::address_of(user))` (NEVER accept arbitrary
  address parameter)
- ✅ **Validate ALL inputs:**
  - Non-zero amounts: `assert!(amount > 0, E_ZERO_AMOUNT)`
  - Minimum thresholds: `assert!(amount >= MIN_SIZE, E_TOO_SMALL)` (prevents fee rounding to zero)
  - Fee validation: `let fee = calc_fee(amount); assert!(fee > 0, E_TOO_SMALL);`
  - Within limits: `assert!(amount <= MAX_AMOUNT, E_AMOUNT_TOO_HIGH)`
  - Non-zero addresses: `assert!(addr != @0x0, E_ZERO_ADDRESS)`
  - String lengths: `assert!(string::length(&name) <= MAX_LENGTH, E_NAME_TOO_LONG)`
- ✅ **Generic type validation**: Use `Receipt<phantom CoinType>` to ensure flash loan repayment type matches
- ✅ **Use `phantom`** for type witnesses: `struct Vault<phantom CoinType>`
- ✅ **Protect critical fields** from mem::swap attacks (never expose `&mut` publicly)
- ✅ **Re-validate after callbacks**: Check invariants before AND after calling untrusted code with `&mut` refs
- ✅ **Store each object in separate account**: Don't put multiple resources in one object account
- ✅ **Avoid unbounded iterations**: Store user data in user accounts, not global vectors
- ✅ **Use SmartTable** for scalable per-user data instead of vectors
- ✅ **Atomic operations**: Prevent front-running by combining related operations (set + evaluate)
- ✅ **Multi-oracle design**: Never use single on-chain price ratio as sole oracle
- ✅ **Collision-proof IDs**: Use object addresses, not string concatenation
- ✅ **Implement pause**: Add emergency pause mechanism for all protocols
- ✅ **Randomness security**: Make randomness functions `entry` (not `public`), balance gas across outcomes

### Error Handling

- ✅ Define clear error constants: `const E_NOT_OWNER: u64 = 1;`
- ✅ Use descriptive error names (E_NOT_OWNER, E_INSUFFICIENT_BALANCE, etc.)
- ✅ Group related errors (1-9: auth, 10-19: amounts, 20-29: validation)

### Modern Syntax

- ✅ Use inline functions for iteration: `inline fun for_each<T>(v: &vector<T>, f: |&T|)`
- ✅ Use lambdas for operations: `for_each(&items, |item| { process(item); })`
- ✅ Use proper imports: `use std::string::String;` not `use std::string;`
- ✅ Use receiver-style method calls: `obj.is_owner(user)` instead of `is_owner(obj, user)` (define first param as
  `self`)
- ✅ Use vector indexed expressions: `&mut vector[index]` instead of `vector::borrow_mut(&mut v, index)`
- ✅ Use direct named addresses: `@marketplace_addr` instead of helper functions that just return `@marketplace_addr`

### Initialization ⭐ CRITICAL

**When your contract has global configuration/registry state:**

- ✅ **ALWAYS use `init_module(deployer: &signer)` for ONE-TIME deployment initialization**
- ✅ **`init_module` is AUTOMATICALLY called on deployment** - no manual transaction required
- ✅ Put ALL deployment-time setup in `init_module`: admin config, registries, protocol parameters
- ✅ Use `create_named_object()` in init_module for deterministic addresses
- ✅ `init_module` MUST be private (no `public` keyword)
- ✅ `init_module` takes exactly one parameter: `&signer` (the deployer)

**CRITICAL: init_module can ONLY take `&signer` (the deployer) - no other parameters!**

**Decision Criteria:**

| Use Case                                                       | Pattern                                              | Example                                                              |
| -------------------------------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- |
| **Deployment-time config** (one-time setup)                    | `init_module(deployer: &signer)` with defaults       | Marketplace config, protocol registry, admin setup                   |
| **User-triggered creation** (multiple times, needs parameters) | `public entry fun create_X(...)`                     | `create_market(question, end_time)`, `create_auction(nft, duration)` |
| **Need custom params at deployment?**                          | `init_module` with defaults + admin update functions | init_module sets fee=250, then `update_fee()` to customize           |

**When to use init_module:**

- ✅ Global configuration that needs DEFAULT values at deployment
- ✅ Protocol-wide registry (empty initially, populated later by users)
- ✅ Shared resources (global pools, vaults, counters)

**When NOT to use init_module:**

- ❌ Creating individual instances (markets, auctions, listings) - use `create_X()` functions instead
- ❌ Functions that need multiple custom parameters - use public entry functions
- ❌ Actions users perform after deployment - use public entry functions

**Example 1 - Marketplace Config (USE init_module):**

```move
struct MarketplaceConfig has key {
    admin: address,
    fee_bps: u64,
    paused: bool,
}

// ✅ CORRECT: Automatic initialization on deployment
fun init_module(deployer: &signer) {
    let constructor_ref = object::create_named_object(
        deployer,
        b"MARKETPLACE_CONFIG_V1"
    );
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, MarketplaceConfig {
        admin: signer::address_of(deployer),
        fee_bps: 250, // 2.5% default
        paused: false,
    });
}

// If you need custom fee at deployment, provide admin update function:
public entry fun update_fee(admin: &signer, new_fee_bps: u64) acquires MarketplaceConfig {
    let config = borrow_global_mut<MarketplaceConfig>(get_config_address());
    assert!(signer::address_of(admin) == config.admin, E_NOT_ADMIN);
    assert!(new_fee_bps <= 1000, E_INVALID_FEE);
    config.fee_bps = new_fee_bps;
}
```

**Example 2 - Prediction Market (DON'T use init_module for create_market):**

```move
// ❌ WRONG: Don't use init_module for user-triggered creation with parameters
// init_module can't take multiple parameters!

// ✅ CORRECT: Use public entry function for creating individual markets
public entry fun create_market(
    creator: &signer,
    question: String,
    end_time: u64,
) {
    // Users call this AFTER deployment to create individual markets
    // This can be called multiple times with different parameters
    // This is NOT deployment initialization

    let constructor_ref = object::create_object(signer::address_of(creator));
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Market {
        question,
        end_time,
        creator: signer::address_of(creator),
        // ... more fields
    });
}

// If prediction market needs marketplace-level config, THAT would use init_module:
fun init_module(deployer: &signer) {
    // Optional: Set up marketplace-level config (fees, admin, etc.)
    // But individual market creation happens in create_market()
}
```

**Example 3 - NFT Marketplace (How to handle multiple parameters):**

```move
// ❌ WRONG: Trying to pass custom parameters to init_module
// fun init_module(deployer: &signer, fee_bps: u64) { ... }  // ERROR: Can't do this!

// ❌ WRONG: Public initialize with custom parameters
public entry fun initialize(admin: &signer, fee_bps: u64) { ... }  // Requires manual call

// ✅ CORRECT Option A: init_module with defaults + admin update
fun init_module(deployer: &signer) {
    // Set reasonable defaults
    create_config(deployer, 250); // 2.5% default
}

public entry fun update_marketplace_config(admin: &signer, new_fee: u64) {
    // Admin can customize after deployment
}

// ✅ CORRECT Option B: No global config needed, per-listing fees
// If each listing can have its own fee, no init_module needed at all!
public entry fun create_listing(seller: &signer, nft: Object<Token>, fee_bps: u64) {
    // Each listing has its own configuration
}
```

### Events

- ✅ Define events with `#[event]` attribute and `has drop, store` abilities
- ✅ Emit events for ALL significant activities (create, transfer, update, delete)
- ✅ Use `event::emit<EventType>(event_instance)` to emit events
- ✅ Include relevant context in events (addresses, amounts, IDs)

## NEVER Rules

When writing Move contracts, you MUST NEVER:

### Digital Assets (NFTs) ⭐ CRITICAL

- ❌ **NEVER use** `Object<token::Token>` for NFTs (use `Object<AptosToken>` specifically)
- ❌ **NEVER use** generic `Object<T>` for NFTs (use `Object<AptosToken>`)
- ❌ **NEVER use legacy TokenV1** (deprecated - use Digital Asset standard)
- ❌ **NEVER import** `aptos_token::token` (legacy - use `aptos_token_objects::token`)
- ❌ **NEVER create tokens** without a parent collection
- ❌ **NEVER use public `initialize()`** for marketplace setup (use `init_module()`)

### Fungible Assets (Tokens/Coins) ⭐ CRITICAL

- ❌ **NEVER use legacy coin module** for new token contracts
- ❌ **NEVER import** `aptos_framework::coin` for new tokens (use FA instead)
- ❌ **NEVER require recipient opt-in** (FA doesn't need `coin::register()`)
- ❌ **NEVER expose MintRef/BurnRef** publicly without authorization

### Legacy Patterns

- ❌ NEVER use resource accounts (use named objects instead)
- ❌ NEVER use raw addresses for objects (use `Object<T>`)
- ❌ NEVER use `account::create_resource_account()` (deprecated)

### Security Violations ⭐ CRITICAL

- ❌ NEVER return ConstructorRef from public functions (can reclaim ownership after transfer)
- ❌ NEVER expose `&mut` references in public functions (mem::swap attacks)
- ❌ NEVER skip signer verification in entry functions
- ❌ NEVER accept arbitrary `address` parameter for `borrow_global_mut` (cross-account manipulation)
- ❌ NEVER trust caller addresses without verification
- ❌ NEVER allow ungated transfers without good reason
- ❌ NEVER use left shift `<<` without validation (does NOT abort on overflow)
- ❌ NEVER iterate over unbounded global storage (gas exhaustion DOS)
- ❌ NEVER store multiple unrelated resources in one object account
- ❌ NEVER use single on-chain price ratio as sole oracle (manipulation)
- ❌ NEVER use string concatenation for token IDs (collisions)
- ❌ NEVER make randomness functions `public` (test-and-abort attacks)
- ❌ NEVER skip pause mechanism in protocols holding user funds
- ❌ NEVER reuse publishing keys between testnet and mainnet

### Bad Practices

- ❌ NEVER skip input validation (especially minimum thresholds for fees)
- ❌ NEVER use magic numbers for errors
- ❌ NEVER ignore division precision loss (validate fees > 0)
- ❌ NEVER deploy without 100% test coverage
- ❌ NEVER create helper functions that just return named addresses (use `@addr` directly)
- ❌ NEVER forget to emit events for significant activities
- ❌ NEVER use old syntax when V2 syntax is available (vector::borrow vs vector[i])
- ❌ NEVER split atomic operations (enables front-running)
- ❌ NEVER pass `&mut` to untrusted callbacks without re-validation

### Initialization Anti-Patterns ⭐ CRITICAL

- ❌ **NEVER use public entry `initialize()` function for DEPLOYMENT-TIME config setup** (use private `init_module()`
  with defaults instead)
- ❌ **NEVER skip `init_module` when your contract has global config/registry** (admin, fees, protocol state) that needs
  DEFAULT values at deployment
- ❌ **NEVER try to pass multiple parameters to init_module** (it can ONLY take `&signer` - use defaults + admin update
  functions)
- ❌ NEVER make init_module public (it's automatically called by the VM, must be private)
- ❌ NEVER put user-specific initialization in init_module (that belongs in per-user entry functions)
- ❌ **NEVER confuse deployment initialization with user actions:**
  - `init_module` = deployment-time config (marketplace admin, default fees) - ONE TIME
  - `create_market()` = user creates individual market - MULTIPLE TIMES
  - `create_listing()` = user creates listing - MULTIPLE TIMES

## Common Patterns

### Pattern 1: Named Objects (Singletons)

```move
/// Create singleton registry
public fun create_registry(admin: &signer): Object<Registry> {
    let constructor_ref = object::create_named_object(
        admin,
        b"REGISTRY_V1"  // Unique seed
    );

    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Registry {
        admin: signer::address_of(admin),
        items: vector::empty(),
    });

    object::object_from_constructor_ref<Registry>(&constructor_ref)
}

/// Retrieve registry by reconstructing address
public fun get_registry(creator_addr: address): Object<Registry> {
    let registry_addr = object::create_object_address(&creator_addr, b"REGISTRY_V1");
    object::address_to_object<Registry>(registry_addr)
}
```

### Pattern 2: Collections (Objects Owning Objects)

```move
struct Collection has key {
    name: String,
    items: vector<Object<Item>>,
}

struct Item has key {
    name: String,
    parent: Object<Collection>,
}

/// Add item to collection
public entry fun add_item_to_collection(
    owner: &signer,
    collection: Object<Collection>,
    item_name: String
) acquires Collection {
    // Verify ownership
    assert!(object::owner(collection) == signer::address_of(owner), E_NOT_OWNER);

    // Create item owned by collection
    let collection_addr = object::object_address(&collection);
    let constructor_ref = object::create_object(collection_addr);

    let item_obj = object::object_from_constructor_ref<Item>(&constructor_ref);
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Item {
        name: item_name,
        parent: collection,
    });

    // Add to collection
    let collection_data = borrow_global_mut<Collection>(collection_addr);
    vector::push_back(&mut collection_data.items, item_obj);
}
```

### Pattern 3: Role-Based Access Control

```move
struct Marketplace has key {
    admin: address,
    operators: vector<address>,
    paused: bool,
}

const E_NOT_ADMIN: u64 = 1;
const E_NOT_OPERATOR: u64 = 2;

/// Admin-only function
public entry fun add_operator(
    admin: &signer,
    operator_addr: address
) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(@my_addr);
    assert!(signer::address_of(admin) == marketplace.admin, E_NOT_ADMIN);

    vector::push_back(&mut marketplace.operators, operator_addr);
}

/// Operator-only function
public entry fun pause(operator: &signer) acquires Marketplace {
    let marketplace = borrow_global_mut<Marketplace>(@my_addr);
    let operator_addr = signer::address_of(operator);

    assert!(
        vector::contains(&marketplace.operators, &operator_addr),
        E_NOT_OPERATOR
    );

    marketplace.paused = true;
}
```

### Pattern 4: Safe Arithmetic

```move
const E_OVERFLOW: u64 = 10;
const E_UNDERFLOW: u64 = 11;
const E_INSUFFICIENT_BALANCE: u64 = 12;

const MAX_U64: u64 = 18446744073709551615;

/// Safe deposit
public entry fun deposit(user: &signer, amount: u64) acquires Account {
    assert!(amount > 0, E_ZERO_AMOUNT);

    let account = borrow_global_mut<Account>(signer::address_of(user));

    // Check overflow before adding
    assert!(account.balance <= MAX_U64 - amount, E_OVERFLOW);

    account.balance = account.balance + amount;
}

/// Safe withdrawal
public entry fun withdraw(user: &signer, amount: u64) acquires Account {
    assert!(amount > 0, E_ZERO_AMOUNT);

    let account = borrow_global_mut<Account>(signer::address_of(user));

    // Check underflow before subtracting
    assert!(account.balance >= amount, E_INSUFFICIENT_BALANCE);

    account.balance = account.balance - amount;
}
```

### Pattern 5: Inline Functions with Lambdas

```move
/// Filter items by predicate
inline fun filter_items<T: copy + drop>(
    items: &vector<T>,
    pred: |&T| bool
): vector<T> {
    let result = vector::empty<T>();
    let i = 0;
    while (i < vector::length(items)) {
        let item = vector::borrow(items, i);
        if (pred(item)) {
            vector::push_back(&mut result, *item);
        };
        i = i + 1;
    };
    result
}

/// Get expensive items (price > 1000)
public fun get_expensive_items(
    marketplace: Object<Marketplace>
): vector<Object<Item>> acquires Marketplace {
    let marketplace_data = borrow_global<Marketplace>(
        object::object_address(&marketplace)
    );

    filter_items(&marketplace_data.items, |item| {
        get_item_price(*item) > 1000
    })
}
```

### Pattern 6: Optional Values

```move
use std::option::{Self, Option};

/// Safe lookup that returns Option
public fun find_item_by_name(
    collection: Object<Collection>,
    name: String
): Option<Object<Item>> acquires Collection {
    let collection_data = borrow_global<Collection>(
        object::object_address(&collection)
    );

    let i = 0;
    while (i < vector::length(&collection_data.items)) {
        let item = *vector::borrow(&collection_data.items, i);
        // Assuming we have get_item_name function
        if (get_item_name(item) == name) {
            return option::some(item)
        };
        i = i + 1;
    };

    option::none()
}

/// Use optional value safely
public fun process_item_if_exists(
    collection: Object<Collection>,
    name: String
) acquires Collection {
    let item_opt = find_item_by_name(collection, name);

    if (option::is_some(&item_opt)) {
        let item = option::extract(&mut item_opt);
        // Process item
    } else {
        // Handle not found
    }
}
```

### Pattern 7: init_module for Initialization

**CORRECT:** Use init_module for contract initialization

```move
module marketplace_addr::marketplace {
    use std::signer;
    use aptos_framework::object::{Self, Object};

    struct MarketplaceConfig has key {
        admin: address,
        fee_percentage: u64,
        paused: bool,
    }

    // ✅ CORRECT: Private init_module for initialization
    fun init_module(deployer: &signer) {
        // Initialize global config on deployment
        let constructor_ref = object::create_named_object(
            deployer,
            b"MARKETPLACE_CONFIG_V1"
        );
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, MarketplaceConfig {
            admin: signer::address_of(deployer),
            fee_percentage: 250, // 2.5%
            paused: false,
        });
    }

    // Access config directly using named address
    public fun get_config(): Object<MarketplaceConfig> {
        let config_addr = object::create_object_address(&@marketplace_addr, b"MARKETPLACE_CONFIG_V1");
        object::address_to_object<MarketplaceConfig>(config_addr)
    }
}
```

**INCORRECT:** Don't create separate init functions or helper functions for named addresses

```move
// ❌ WRONG: Public entry initialize function instead of init_module
public entry fun initialize(admin: &signer, fee_bps: u64) {
    // PROBLEMS with this approach:
    // 1. Requires manual call after deployment (extra transaction, costs gas)
    // 2. User must remember to call it (risk of forgetting)
    // 3. Can be called by anyone unless protected (security risk)
    // 4. Can potentially be called multiple times (need extra protection)
    // 5. Contract is non-functional until initialized

    let constructor_ref = object::create_object(signer::address_of(admin));
    // ... initialization code
}

// ✅ CORRECT: Use private init_module instead
fun init_module(deployer: &signer) {
    // Automatically called on deployment - no manual step!
    // Private - can't be called by users
    // Guaranteed one-time execution
    // Contract is immediately functional after deployment

    let constructor_ref = object::create_named_object(deployer, b"CONFIG");
    // ... initialization code
}

// ❌ WRONG: Unnecessary helper function for named address
fun get_marketplace_address(): address {
    @marketplace_addr  // Just use @marketplace_addr directly in code!
}
```

### Pattern 8: Event Emission

**CORRECT:** Emit events for all significant activities

```move
module my_addr::nft_marketplace {
    use std::signer;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::event;

    // ✅ Define events with #[event] attribute
    #[event]
    struct ListingCreated has drop, store {
        listing_id: address,
        seller: address,
        nft: address,
        price: u64,
        timestamp: u64,
    }

    #[event]
    struct ListingCancelled has drop, store {
        listing_id: address,
        seller: address,
        timestamp: u64,
    }

    #[event]
    struct ItemSold has drop, store {
        listing_id: address,
        seller: address,
        buyer: address,
        nft: address,
        price: u64,
        timestamp: u64,
    }

    struct Listing has key {
        nft: Object<NFT>,
        seller: address,
        price: u64,
    }

    /// Create listing with event emission
    public entry fun create_listing(
        seller: &signer,
        nft: Object<NFT>,
        price: u64
    ) {
        let seller_addr = signer::address_of(seller);

        // Verify ownership
        assert!(object::owner(nft) == seller_addr, E_NOT_OWNER);
        assert!(price > 0, E_INVALID_PRICE);

        // Create listing object
        let constructor_ref = object::create_object(seller_addr);
        let listing_addr = object::address_from_constructor_ref(&constructor_ref);
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Listing {
            nft,
            seller: seller_addr,
            price,
        });

        // ✅ ALWAYS emit event for significant activities
        event::emit(ListingCreated {
            listing_id: listing_addr,
            seller: seller_addr,
            nft: object::object_address(&nft),
            price,
            timestamp: aptos_framework::timestamp::now_seconds(),
        });
    }

    /// Cancel listing with event
    public entry fun cancel_listing(
        seller: &signer,
        listing: Object<Listing>
    ) acquires Listing {
        let seller_addr = signer::address_of(seller);
        let listing_addr = object::object_address(&listing);

        // Verify seller owns listing
        assert!(object::owner(listing) == seller_addr, E_NOT_OWNER);

        // Remove listing
        let Listing { nft: _, seller: _, price: _ } = move_from<Listing>(listing_addr);

        // ✅ Emit cancellation event
        event::emit(ListingCancelled {
            listing_id: listing_addr,
            seller: seller_addr,
            timestamp: aptos_framework::timestamp::now_seconds(),
        });
    }
}
```

**INCORRECT:** Don't skip events

```move
// ❌ WRONG: No events emitted
public entry fun create_listing(
    seller: &signer,
    nft: Object<NFT>,
    price: u64
) {
    // ... create listing logic ...
    // Missing event emission!
}
```

### Pattern 9: V2 Syntax - Method Calls

**CORRECT:** Use receiver-style method calls with `self`

```move
module my_addr::items {
    use std::signer;
    use aptos_framework::object::{Self, Object};

    struct Item has key {
        owner: address,
        value: u64,
    }

    // ✅ CORRECT: Use 'self' as first parameter name
    public fun is_owner(self: &Object<Item>, user: &signer): bool acquires Item {
        let item_data = borrow_global<Item>(object::object_address(self));
        item_data.owner == signer::address_of(user)
    }

    public fun get_value(self: &Object<Item>): u64 acquires Item {
        let item_data = borrow_global<Item>(object::object_address(self));
        item_data.value
    }

    // ✅ CORRECT: Call with dot notation
    public entry fun update_if_owner(
        user: &signer,
        item: Object<Item>,
        new_value: u64
    ) acquires Item {
        // Receiver-style call (syntactic sugar for is_owner(&item, user))
        assert!(item.is_owner(user), E_NOT_OWNER);

        let item_data = borrow_global_mut<Item>(object::object_address(&item));
        item_data.value = new_value;
    }
}
```

**INCORRECT:** Old function call style (still works but less modern)

```move
// ❌ LESS MODERN: Not using 'self' parameter
public fun is_owner(item: &Object<Item>, user: &signer): bool acquires Item {
    // ...
}

// Must call with function-style syntax
assert!(is_owner(&item, user), E_NOT_OWNER);  // Old style
```

### Pattern 10: V2 Syntax - Vector Indexing

**CORRECT:** Use index notation for vectors

```move
module my_addr::registry {
    use std::vector;

    struct Registry has key {
        items: vector<u64>,
    }

    // ✅ CORRECT: Use vector[index] syntax
    public fun get_item(registry: &Registry, index: u64): u64 {
        *&registry.items[index]  // V2 index notation
    }

    public fun update_item(registry: &mut Registry, index: u64, value: u64) {
        *&mut registry.items[index] = value;  // V2 mutable index notation
    }

    // ✅ CORRECT: Iterate with index notation
    public fun sum_all(registry: &Registry): u64 {
        let sum = 0;
        let i = 0;
        let len = vector::length(&registry.items);

        while (i < len) {
            sum = sum + registry.items[i];  // Clean V2 syntax
            i = i + 1;
        };

        sum
    }
}
```

**INCORRECT:** Old vector borrow syntax (still works but less modern)

```move
// ❌ LESS MODERN: Using vector::borrow
public fun get_item(registry: &Registry, index: u64): u64 {
    *vector::borrow(&registry.items, index)  // Old style
}

// ❌ LESS MODERN: Using vector::borrow_mut
public fun update_item(registry: &mut Registry, index: u64, value: u64) {
    *vector::borrow_mut(&mut registry.items, index) = value;  // Old style
}
```

## Edge Cases to Handle

| Scenario            | Check                                                   | Error Code         |
| ------------------- | ------------------------------------------------------- | ------------------ |
| Zero amounts        | `assert!(amount > 0, E_ZERO_AMOUNT)`                    | E_ZERO_AMOUNT      |
| Excessive amounts   | `assert!(amount <= MAX, E_AMOUNT_TOO_HIGH)`             | E_AMOUNT_TOO_HIGH  |
| Empty vectors       | `assert!(vector::length(&v) > 0, E_EMPTY_VECTOR)`       | E_EMPTY_VECTOR     |
| Empty strings       | `assert!(string::length(&s) > 0, E_EMPTY_STRING)`       | E_EMPTY_STRING     |
| Strings too long    | `assert!(string::length(&s) <= MAX, E_STRING_TOO_LONG)` | E_STRING_TOO_LONG  |
| Zero address        | `assert!(addr != @0x0, E_ZERO_ADDRESS)`                 | E_ZERO_ADDRESS     |
| Overflow            | `assert!(a <= MAX_U64 - b, E_OVERFLOW)`                 | E_OVERFLOW         |
| Underflow           | `assert!(a >= b, E_UNDERFLOW)`                          | E_UNDERFLOW        |
| Division by zero    | `assert!(divisor > 0, E_DIVISION_BY_ZERO)`              | E_DIVISION_BY_ZERO |
| Unauthorized access | `assert!(signer == expected, E_UNAUTHORIZED)`           | E_UNAUTHORIZED     |
| Not object owner    | `assert!(object::owner(obj) == user, E_NOT_OWNER)`      | E_NOT_OWNER        |

## Complete Example: NFT Collection

```move
module my_addr::nft_collection {
    use std::string::String;
    use std::signer;
    use std::vector;
    use aptos_framework::object::{Self, Object};

    // ============ Structs ============

    struct Collection has key {
        name: String,
        description: String,
        creator: address,
        nfts: vector<Object<NFT>>,
    }

    struct NFT has key {
        name: String,
        token_id: u64,
        collection: Object<Collection>,
        transfer_ref: object::TransferRef,
    }

    // ============ Constants ============

    const E_NOT_OWNER: u64 = 1;
    const E_NOT_CREATOR: u64 = 2;
    const E_EMPTY_NAME: u64 = 10;
    const E_NAME_TOO_LONG: u64 = 11;
    const E_ZERO_ADDRESS: u64 = 20;

    const MAX_NAME_LENGTH: u64 = 64;

    // ============ Public Entry Functions ============

    /// Create NFT collection
    public entry fun create_collection(
        creator: &signer,
        name: String,
        description: String
    ) {
        // Validate inputs
        assert!(string::length(&name) > 0, E_EMPTY_NAME);
        assert!(string::length(&name) <= MAX_NAME_LENGTH, E_NAME_TOO_LONG);

        let constructor_ref = object::create_named_object(
            creator,
            *string::bytes(&name)
        );

        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Collection {
            name,
            description,
            creator: signer::address_of(creator),
            nfts: vector::empty(),
        });
    }

    /// Mint NFT into collection
    public entry fun mint_nft(
        creator: &signer,
        collection: Object<Collection>,
        nft_name: String,
        token_id: u64
    ) acquires Collection {
        // Verify creator owns collection
        let creator_addr = signer::address_of(creator);
        assert!(object::owner(collection) == creator_addr, E_NOT_CREATOR);

        // Validate input
        assert!(string::length(&nft_name) > 0, E_EMPTY_NAME);

        // Create NFT owned by collection
        let collection_addr = object::object_address(&collection);
        let constructor_ref = object::create_object(collection_addr);

        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let object_signer = object::generate_signer(&constructor_ref);

        let nft_obj = object::object_from_constructor_ref<NFT>(&constructor_ref);

        move_to(&object_signer, NFT {
            name: nft_name,
            token_id,
            collection,
            transfer_ref,
        });

        // Add to collection
        let collection_data = borrow_global_mut<Collection>(collection_addr);
        vector::push_back(&mut collection_data.nfts, nft_obj);
    }

    /// Transfer NFT
    public entry fun transfer_nft(
        owner: &signer,
        nft: Object<NFT>,
        recipient: address
    ) acquires NFT {
        // Verify ownership
        assert!(object::owner(nft) == signer::address_of(owner), E_NOT_OWNER);

        // Validate recipient
        assert!(recipient != @0x0, E_ZERO_ADDRESS);

        // Transfer
        let nft_data = borrow_global<NFT>(object::object_address(&nft));
        object::transfer_with_ref(
            object::generate_linear_transfer_ref(&nft_data.transfer_ref),
            recipient
        );
    }

    // ============ Public View Functions ============

    #[view]
    public fun get_collection_size(collection: Object<Collection>): u64 acquires Collection {
        let collection_data = borrow_global<Collection>(
            object::object_address(&collection)
        );
        vector::length(&collection_data.nfts)
    }

    #[view]
    public fun get_nft_name(nft: Object<NFT>): String acquires NFT {
        let nft_data = borrow_global<NFT>(object::object_address(&nft));
        nft_data.name
    }
}
```

## References

**Official Documentation:**

- Digital Asset Standard: https://aptos.dev/build/smart-contracts/digital-asset
- Digital Asset (Standards): https://aptos.dev/standards/digital-asset/
- Your First NFT Tutorial: https://aptos.dev/tutorials/your-first-nft/
- Object Model: https://aptos.dev/build/smart-contracts/object
- Security Guidelines: https://aptos.dev/build/smart-contracts/move-security-guidelines
- Move Book: https://aptos.dev/build/smart-contracts/book

**Example Repositories:**

- aptos-core/aptos-move/move-examples/
- aptos-core/aptos-move/framework/aptos-token-objects/

**Pattern Documentation (Local):**

- `../../patterns/DIGITAL_ASSETS.md` - ⭐ Digital Asset (NFT) standard - CRITICAL for NFTs
- `../../patterns/OBJECTS.md` - Comprehensive object model guide
- `../../patterns/SECURITY.md` - Security checklist and patterns
- `../../patterns/MOVE_V2_SYNTAX.md` - Modern syntax examples

**Related Skills:**

- `generate-tests` - Write tests for contracts (use AFTER writing contracts)
- `security-audit` - Audit contracts before deployment
- `search-aptos-examples` - Find similar examples (use BEFORE writing)

---

**Remember:** Search examples first, use objects always, verify security, validate inputs, test everything.
