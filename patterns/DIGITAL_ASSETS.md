# Aptos Digital Asset Standard

**Purpose:** Comprehensive guide to the Aptos Digital Asset (DA) standard for NFTs.

**Target:** AI assistants generating NFT-related Move V2 smart contracts.

---

## Overview

The **Digital Asset (DA) standard** is the modern NFT framework for Aptos. It replaces the legacy TokenV1 standard
(which has been migrated).

**Key Benefits:**

- 🚀 **50% gas reduction** compared to legacy tokens
- ✅ **Direct transfers** - no recipient opt-in required
- 🔗 **Composable NFTs** - NFTs can own other NFTs
- 🎨 **Full customization** - built on Move Objects for extensibility

**Deployed at:** `0x4` (reserved address for all Digital Asset modules)

---

## Core Concepts

### Collections

A **Collection** is a group of related NFTs with:

- Name (unique identifier)
- Description
- URI (collection metadata)
- Royalty settings
- Supply limit (fixed or unlimited)

### Tokens (Digital Assets)

A **Token** (Digital Asset) represents a unique NFT with:

- Name
- Description
- URI (links to asset metadata/image)
- Parent collection reference
- Optional properties (key-value pairs)

**Token Types:**

- **Named tokens**: Deterministic addresses (not deletable)
- **Unnamed tokens**: Non-deterministic addresses (deletable)

---

## Required Imports

For NFT contracts, always import these modules:

```move
use std::option::{Self, Option};
use std::string::{Self, String};
use std::signer;
use std::vector;

use aptos_framework::object::{Self, Object, ConstructorRef, DeleteRef};
use aptos_framework::event;

// ✅ Digital Asset standard modules
use aptos_token_objects::collection;
use aptos_token_objects::token;
use aptos_token_objects::royalty::{Self, Royalty};
use aptos_token_objects::property_map;

// For higher-level helpers (optional)
use aptos_token_objects::aptos_token::{Self, AptosToken};
```

---

## Creating Collections

### Pattern 1: Fixed Supply Collection

Use when you have a maximum number of tokens (e.g., 10,000 NFTs):

```move
module my_addr::nft_collection {
    use std::string::{Self, String};
    use std::signer;
    use std::option;
    use aptos_framework::object::{Self, Object};
    use aptos_token_objects::collection;
    use aptos_token_objects::royalty::{Self, Royalty};

    const COLLECTION_MAX_SUPPLY: u64 = 10000;

    /// Create fixed supply collection
    public entry fun create_collection(creator: &signer) {
        let royalty = royalty::create(5, 100, signer::address_of(creator)); // 5% royalty

        collection::create_fixed_collection(
            creator,
            string::utf8(b"My awesome NFT collection"),  // description
            COLLECTION_MAX_SUPPLY,                        // max_supply
            string::utf8(b"My Collection"),               // name
            option::some(royalty),                        // royalty
            string::utf8(b"https://mycollection.io"),    // uri
        );
    }
}
```

### Pattern 2: Unlimited Supply Collection

Use when tokens can be minted indefinitely:

```move
public entry fun create_unlimited_collection(creator: &signer) {
    let royalty = royalty::create(5, 100, signer::address_of(creator));

    collection::create_unlimited_collection(
        creator,
        string::utf8(b"Unlimited NFT collection"),
        string::utf8(b"Unlimited Collection"),
        option::some(royalty),
        string::utf8(b"https://unlimited.io"),
    );
}
```

### Collection with Mutability

Control what can be changed after creation:

```move
public entry fun create_mutable_collection(creator: &signer) {
    let constructor_ref = collection::create_unlimited_collection(
        creator,
        string::utf8(b"Description"),
        string::utf8(b"Collection Name"),
        option::none(), // no royalty
        string::utf8(b"https://uri.com"),
    );

    // Generate mutator refs if you need to change collection later
    let mutator_ref = collection::generate_mutator_ref(&constructor_ref);

    // Store mutator_ref in a resource if you need to mutate later
    // collection::set_description(&mutator_ref, new_description);
    // collection::set_uri(&mutator_ref, new_uri);
}
```

---

## Minting Tokens (NFTs)

### Pattern 1: Named Token (Recommended for most NFTs)

Named tokens have deterministic addresses based on collection + token name:

```move
module my_addr::nft_minting {
    use std::string::{Self, String};
    use std::signer;
    use std::option;
    use aptos_framework::object::{Self, Object};
    use aptos_framework::event;
    use aptos_token_objects::collection;
    use aptos_token_objects::token::{Self, Token};

    #[event]
    struct TokenMinted has drop, store {
        token_address: address,
        token_name: String,
        owner: address,
    }

    /// Mint named token
    public entry fun mint_nft(
        creator: &signer,
        collection_name: String,
        token_name: String,
        token_description: String,
        token_uri: String,
    ) {
        // Get collection object
        let collection_obj = collection::create_collection_address(
            &signer::address_of(creator),
            &collection_name
        );
        let collection = object::address_to_object<collection::Collection>(collection_obj);

        // Create named token
        let constructor_ref = token::create_named_token(
            creator,
            collection_name,
            token_description,
            token_name,
            option::none(), // no royalty override
            token_uri,
        );

        let token_signer = object::generate_signer(&constructor_ref);
        let token_address = object::address_from_constructor_ref(&constructor_ref);

        // Emit event
        event::emit(TokenMinted {
            token_address,
            token_name,
            owner: signer::address_of(creator),
        });
    }
}
```

### Pattern 2: Unnamed Token (For deletable NFTs)

Unnamed tokens can be burned/deleted:

```move
public entry fun mint_deletable_nft(
    creator: &signer,
    collection_name: String,
    token_description: String,
    token_name: String,
    token_uri: String,
) {
    let constructor_ref = token::create(
        creator,
        collection_name,
        token_description,
        token_name,
        option::none(),
        token_uri,
    );

    // Generate delete ref for burning later
    let delete_ref = object::generate_delete_ref(&constructor_ref);

    // Store delete_ref if you want to enable burning
    // Later: object::delete(delete_ref);
}
```

### Pattern 3: Token with Properties

Add custom metadata to tokens:

```move
use aptos_token_objects::property_map;

public entry fun mint_nft_with_properties(
    creator: &signer,
    collection_name: String,
    token_name: String,
) {
    let constructor_ref = token::create_named_token(
        creator,
        collection_name,
        string::utf8(b"Token with properties"),
        token_name,
        option::none(),
        string::utf8(b"https://token.io"),
    );

    let token_signer = object::generate_signer(&constructor_ref);

    // Add properties
    let properties = property_map::prepare_input(
        vector[string::utf8(b"level"), string::utf8(b"rarity")],
        vector[string::utf8(b"u64"), string::utf8(b"string")],
        vector[vector[5], b"Epic"],
    );

    property_map::init(&constructor_ref, properties);
}
```

---

## Using AptosToken for Simplification

The `aptos_token` module provides higher-level helpers:

```move
use aptos_token_objects::aptos_token::{Self, AptosToken};

/// Mint using aptos_token helper
public entry fun mint_simple(
    creator: &signer,
    collection: String,
    description: String,
    name: String,
    uri: String,
) {
    aptos_token::mint(
        creator,
        collection,
        description,
        name,
        uri,
        vector[], // property_keys
        vector[], // property_types
        vector[], // property_values
    );
}
```

---

## NFT Marketplace Pattern

### CRITICAL: Object Ownership Hierarchy

When creating a marketplace that mints NFTs, you MUST understand the ownership hierarchy:

**WRONG Pattern:**

```move
// ❌ WRONG: Collection creates tokens in itself
fun init_module(deployer: &signer) {
    // Create collection
    let collection_ref = collection::create_unlimited_collection(...);
    let collection_signer = object::generate_signer(&collection_ref);

    // Store collection's extend_ref
    move_to(&collection_signer, Config {
        extend_ref: object::generate_extend_ref(&collection_ref),  // ❌ Collection's extend_ref
    });
}

public entry fun mint_nft(creator: &signer) acquires Config {
    let config = borrow_global<Config>(...);
    let collection_signer = object::generate_signer_for_extending(&config.extend_ref);

    // ❌ WRONG: Collection can't create tokens in itself!
    token::create_named_token(&collection_signer, ...);
}
```

**CORRECT Pattern:**

```move
// ✅ CORRECT: Marketplace object owns collection and creates tokens
fun init_module(deployer: &signer) {
    // Create a marketplace state object
    let marketplace_ref = object::create_named_object(deployer, b"MARKETPLACE_STATE");
    let marketplace_signer = object::generate_signer(&marketplace_ref);

    // Marketplace object creates and owns the collection
    collection::create_unlimited_collection(&marketplace_signer, ...);

    // Store MARKETPLACE object's extend_ref
    move_to(&marketplace_signer, MarketplaceConfig {
        extend_ref: object::generate_extend_ref(&marketplace_ref),  // ✅ Marketplace's extend_ref
    });
}

public entry fun mint_nft(creator: &signer) acquires MarketplaceConfig {
    let config = borrow_global<MarketplaceConfig>(...);

    // ✅ CORRECT: Use marketplace signer (owner of collection)
    let marketplace_signer = object::generate_signer_for_extending(&config.extend_ref);
    token::create_named_token(&marketplace_signer, ...);  // Works!
}
```

**Why?** `token::create_named_token()` requires the signer to be the OWNER of the collection, not the collection itself.

**Hierarchy:**

```
Marketplace Object
  └── Owns Collection
      └── Contains Tokens (created by Marketplace signer)
```

### Marketplace with Digital Assets

```move
module marketplace_addr::nft_marketplace {
    use std::signer;
    use std::string::{Self, String};
    use std::option;
    use aptos_framework::object::{Self, Object, ExtendRef};
    use aptos_framework::coin;
    use aptos_framework::aptos_coin::AptosCoin;
    use aptos_framework::event;
    use aptos_token_objects::collection;
    use aptos_token_objects::token::{Self, Token};
    use aptos_token_objects::royalty;

    // ============ Constants ============

    const COLLECTION_NAME: vector<u8> = b"Marketplace Collection";
    const MARKETPLACE_STATE_SEED: vector<u8> = b"MARKETPLACE_STATE";

    // ============ Error Codes ============

    const E_NOT_OWNER: u64 = 1;
    const E_NOT_SELLER: u64 = 2;
    const E_ZERO_PRICE: u64 = 3;
    const E_NOT_LISTED: u64 = 4;

    // ============ Events ============

    #[event]
    struct NFTMinted has drop, store {
        nft_address: address,
        creator: address,
        name: String,
    }

    #[event]
    struct NFTListed has drop, store {
        nft_address: address,
        seller: address,
        price: u64,
    }

    // ============ Structs ============

    /// Marketplace configuration (stored in marketplace state object)
    struct MarketplaceConfig has key {
        admin: address,
        marketplace_fee_bps: u64,
        extend_ref: ExtendRef,  // For creating tokens
    }

    /// NFT-specific data
    struct NFTData has key {
        creator: address,
        royalty_bps: u64,
        extend_ref: ExtendRef,
        transfer_ref: object::TransferRef,
    }

    /// Listing (only exists when NFT is for sale)
    struct Listing has key {
        seller: address,
        price: u64,
    }

    // ============ Helper Functions ============

    /// Get marketplace state object address
    fun get_marketplace_state_address(): address {
        object::create_object_address(&@marketplace_addr, MARKETPLACE_STATE_SEED)
    }

    // ============ Init Module ============

    fun init_module(deployer: &signer) {
        // Create marketplace state object (owns everything)
        let marketplace_ref = object::create_named_object(deployer, MARKETPLACE_STATE_SEED);
        let marketplace_signer = object::generate_signer(&marketplace_ref);
        let marketplace_extend_ref = object::generate_extend_ref(&marketplace_ref);

        // Marketplace object creates and owns the collection
        collection::create_unlimited_collection(
            &marketplace_signer,  // Marketplace signer creates collection
            string::utf8(b"Collection for marketplace NFTs"),
            string::utf8(COLLECTION_NAME),
            option::none(),
            string::utf8(b"https://marketplace.io"),
        );

        // Store marketplace config
        move_to(&marketplace_signer, MarketplaceConfig {
            admin: signer::address_of(deployer),
            marketplace_fee_bps: 250,  // 2.5%
            extend_ref: marketplace_extend_ref,
        });
    }

    // ============ Public Entry Functions ============

    /// Mint NFT using marketplace signer
    public entry fun mint_nft(
        creator: &signer,
        name: String,
        description: String,
        uri: String,
        royalty_bps: u64,
    ) acquires MarketplaceConfig {
        let creator_addr = signer::address_of(creator);

        // Get marketplace config
        let marketplace_state_addr = get_marketplace_state_address();
        let config = borrow_global<MarketplaceConfig>(marketplace_state_addr);

        // Generate marketplace signer (owner of collection)
        let marketplace_signer = object::generate_signer_for_extending(&config.extend_ref);

        // Create royalty
        let royalty = royalty::create(royalty_bps, 10000, creator_addr);

        // Create token using marketplace signer (collection owner)
        let constructor_ref = token::create_named_token(
            &marketplace_signer,  // Must be collection owner
            string::utf8(COLLECTION_NAME),
            description,
            name,
            option::some(royalty),
            uri,
        );

        let token_signer = object::generate_signer(&constructor_ref);
        let extend_ref = object::generate_extend_ref(&constructor_ref);
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);

        // Disable ungated transfers
        object::disable_ungated_transfer(&transfer_ref);

        // Transfer to creator BEFORE storing NFTData
        // Must use transfer_with_ref since ungated transfers are disabled
        let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
        object::transfer_with_ref(linear_transfer_ref, creator_addr);

        // Store NFT data
        move_to(&token_signer, NFTData {
            creator: creator_addr,
            royalty_bps,
            extend_ref,
            transfer_ref,
        });

        // Emit event
        event::emit(NFTMinted {
            nft_address: signer::address_of(&token_signer),
            creator: creator_addr,
            name,
        });
    }

    /// List NFT for sale
    public entry fun list_nft<T: key>(
        seller: &signer,
        nft: Object<T>,
        price: u64,
    ) acquires NFTData {
        let seller_addr = signer::address_of(seller);
        let nft_addr = object::object_address(&nft);

        // Verify ownership
        assert!(object::is_owner(nft, seller_addr), E_NOT_OWNER);
        assert!(price > 0, E_ZERO_PRICE);

        // Get NFT data to access extend_ref
        let nft_data = borrow_global<NFTData>(nft_addr);
        let nft_signer = object::generate_signer_for_extending(&nft_data.extend_ref);

        // Create listing at NFT address
        move_to(&nft_signer, Listing {
            seller: seller_addr,
            price,
        });

        event::emit(NFTListed {
            nft_address: nft_addr,
            seller: seller_addr,
            price,
        });
    }

    /// Purchase NFT
    public entry fun purchase_nft<T: key>(
        buyer: &signer,
        nft: Object<T>,
    ) acquires NFTData, Listing, MarketplaceConfig {
        let buyer_addr = signer::address_of(buyer);
        let nft_addr = object::object_address(&nft);

        // Get listing
        assert!(exists<Listing>(nft_addr), E_NOT_LISTED);
        let Listing { seller, price } = move_from<Listing>(nft_addr);

        // Get NFT data
        let nft_data = borrow_global<NFTData>(nft_addr);

        // Calculate and transfer fees (omitted for brevity)
        coin::transfer<AptosCoin>(buyer, seller, price);

        // Transfer NFT using stored transfer_ref
        let linear_transfer_ref = object::generate_linear_transfer_ref(&nft_data.transfer_ref);
        object::transfer_with_ref(linear_transfer_ref, buyer_addr);
    }
}
```

---

## ALWAYS Rules

When working with NFTs on Aptos:

### Digital Asset Standard

- ✅ **ALWAYS use Digital Asset (DA) standard** for ALL NFT contracts
- ✅ **ALWAYS import** `aptos_token_objects::collection` and `aptos_token_objects::token`
- ✅ **ALWAYS use** `Object<AptosToken>` for NFT references (not generic `Object<T>`)
- ✅ **ALWAYS create collections** before minting tokens
- ✅ **ALWAYS emit events** for minting, transfers, and sales

### Collection Creation

- ✅ **ALWAYS use** `collection::create_fixed_collection()` for limited supply
- ✅ **ALWAYS use** `collection::create_unlimited_collection()` for unlimited supply
- ✅ **ALWAYS set royalties** when creating collections (use `royalty::create()`)
- ✅ **ALWAYS provide** collection description, name, and URI

### Token Minting

- ✅ **ALWAYS use** `token::create_named_token()` for standard NFTs
- ✅ **ALWAYS use** `token::create()` for deletable NFTs
- ✅ **ALWAYS verify** collection exists before minting
- ✅ **ALWAYS provide** token description, name, and URI

### Marketplace Contracts

- ✅ **ALWAYS verify** token ownership before listing
- ✅ **ALWAYS use** escrow pattern (transfer to listing object)
- ✅ **ALWAYS validate** prices (non-zero, within limits)
- ✅ **ALWAYS emit events** for listings and sales

---

## NEVER Rules

### Legacy Token Standard

- ❌ **NEVER use** the legacy TokenV1 standard (deprecated, all tokens migrated)
- ❌ **NEVER import** `aptos_token::token` (legacy module)
- ❌ **NEVER use** `token::create_token_script()` (legacy function)
- ❌ **NEVER require** recipient opt-in for transfers (DA doesn't need it)

### Bad Patterns

- ❌ **NEVER use** generic `Object<T>` for NFTs (use `Object<AptosToken>`)
- ❌ **NEVER skip** royalty configuration for collections
- ❌ **NEVER forget** to transfer NFTs to escrow in marketplaces
- ❌ **NEVER create** tokens without a parent collection
- ❌ **NEVER use** raw addresses for NFTs (use Object references)

---

## Common Patterns

### Pattern: Collection + Minting in One Contract

```move
module my_addr::nft_project {
    use std::string::{Self, String};
    use std::signer;
    use std::option;
    use aptos_framework::object::{Self, Object};
    use aptos_token_objects::collection;
    use aptos_token_objects::token;
    use aptos_token_objects::royalty;
    use aptos_token_objects::aptos_token::AptosToken;

    const COLLECTION_NAME: vector<u8> = b"My NFT Collection";
    const MAX_SUPPLY: u64 = 10000;

    struct CollectionConfig has key {
        minted: u64,
        max_supply: u64,
    }

    fun init_module(deployer: &signer) {
        // Create collection on deployment
        let royalty = royalty::create(5, 100, signer::address_of(deployer));

        collection::create_fixed_collection(
            deployer,
            string::utf8(b"Premium NFT collection"),
            MAX_SUPPLY,
            string::utf8(COLLECTION_NAME),
            option::some(royalty),
            string::utf8(b"https://collection.io"),
        );

        // Track minting
        move_to(deployer, CollectionConfig {
            minted: 0,
            max_supply: MAX_SUPPLY,
        });
    }

    public entry fun mint(
        user: &signer,
        token_name: String,
        token_uri: String,
    ) acquires CollectionConfig {
        let config = borrow_global_mut<CollectionConfig>(@my_addr);
        assert!(config.minted < config.max_supply, E_MAX_SUPPLY_REACHED);

        token::create_named_token(
            user,
            string::utf8(COLLECTION_NAME),
            string::utf8(b"NFT description"),
            token_name,
            option::none(),
            token_uri,
        );

        config.minted = config.minted + 1;
    }
}
```

### Pattern: Get Token Info

```move
use aptos_token_objects::token;

#[view]
public fun get_token_name(token: Object<AptosToken>): String {
    token::name(token)
}

#[view]
public fun get_token_uri(token: Object<AptosToken>): String {
    token::uri(token)
}

#[view]
public fun get_token_description(token: Object<AptosToken>): String {
    token::description(token)
}
```

---

## Migration from Legacy

**All legacy TokenV1 tokens have been migrated.** You should NOT support legacy tokens.

If you encounter legacy code:

- Replace `aptos_token::token` → `aptos_token_objects::token`
- Replace `TokenId` → `Object<AptosToken>`
- Remove opt-in logic (DA doesn't need it)

---

## Testing Digital Assets

```move
#[test_only]
module my_addr::nft_tests {
    use std::string;
    use std::option;
    use aptos_framework::object;
    use aptos_token_objects::collection;
    use aptos_token_objects::token;
    use aptos_token_objects::royalty;
    use aptos_token_objects::aptos_token::AptosToken;

    #[test(creator = @0x1)]
    public fun test_create_collection_and_mint(creator: &signer) {
        // Create collection
        let royalty = royalty::create(5, 100, signer::address_of(creator));
        collection::create_unlimited_collection(
            creator,
            string::utf8(b"Test collection"),
            string::utf8(b"Test"),
            option::some(royalty),
            string::utf8(b"https://test.com"),
        );

        // Mint token
        let constructor_ref = token::create_named_token(
            creator,
            string::utf8(b"Test"),
            string::utf8(b"Test token"),
            string::utf8(b"Token #1"),
            option::none(),
            string::utf8(b"https://token.com"),
        );

        let token_address = object::address_from_constructor_ref(&constructor_ref);
        let token = object::address_to_object<AptosToken>(token_address);

        // Verify token created
        assert!(object::is_owner(token, signer::address_of(creator)), 0);
    }

    #[test(seller = @0x1, buyer = @0x2)]
    public fun test_marketplace_flow(seller: &signer, buyer: &signer) {
        // Test collection creation → minting → listing → buying
        // Full marketplace flow with Digital Assets
    }
}
```

---

## References

**Official Documentation:**

- [Digital Asset Standard](https://aptos.dev/build/smart-contracts/digital-asset)
- [Digital Asset Standard (Standards)](https://aptos.dev/standards/digital-asset/)
- [Your First NFT Tutorial](https://aptos.dev/tutorials/your-first-nft/)
- [Token Objects Framework](https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/framework/aptos-token-objects)

**Related Patterns:**

- `OBJECTS.md` - Object model fundamentals
- `SECURITY.md` - NFT security considerations
- `TESTING.md` - Testing NFT contracts

---

**Remember:** Always use the Digital Asset standard for NFTs. Legacy TokenV1 is deprecated and all tokens have been
migrated.
