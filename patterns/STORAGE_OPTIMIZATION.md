# Storage Optimization Patterns for Aptos Move V2

Comprehensive guide for optimizing on-chain storage to minimize costs and maximize performance in Aptos Move contracts.

## Storage Cost Model

In Aptos, storage costs are based on:

- **Bytes stored**: Each byte costs gas to store
- **Number of items**: More storage items = more cost
- **Access patterns**: Reading/writing costs gas
- **Storage refunds**: Deleting storage can refund gas

## Storage Optimization Strategies

### 1. Data Structure Selection

#### Use Tables for Large Collections

```move
module my_addr::storage_optimization {
    use std::table::{Self, Table};
    use aptos_std::smart_table::{Self, SmartTable};

    // BAD: Vector for large collections - O(n) operations
    struct BadRegistry has key {
        users: vector<User>,  // Linear search, expensive to grow
    }

    // GOOD: Table for key-value lookups - O(1) operations
    struct GoodRegistry has key {
        users: Table<address, User>,
        user_count: u64,
    }

    // BETTER: SmartTable for very large collections
    struct BetterRegistry has key {
        users: SmartTable<address, User>,  // Optimized for large datasets
        user_count: u64,
    }
}
```

#### Choose Appropriate Integer Sizes

```move
// BAD: Wasting space with oversized integers
struct WastefulData has store {
    level: u64,      // Max value: 100 (wastes 7 bytes)
    lives: u64,      // Max value: 3 (wastes 7 bytes)
    score: u64,      // Actually needs u64
    active: u64,     // Boolean (wastes 7 bytes)
}

// GOOD: Right-sized integers
struct EfficientData has store {
    level: u8,       // 1 byte for 0-255
    lives: u8,       // 1 byte for 0-255
    score: u64,      // 8 bytes when needed
    flags: u8,       // Bit flags for booleans
}
```

### 2. Struct Packing

#### Pack Related Fields

```move
// BAD: 32 bytes total
struct UnpackedOrder has store {
    price: u64,        // 8 bytes
    quantity: u64,     // 8 bytes
    is_buy: bool,      // 8 bytes (!)
    is_active: bool,   // 8 bytes (!)
}

// GOOD: 16 bytes total (50% reduction)
struct PackedOrder has store {
    price: u64,        // 8 bytes
    data: u64,         // 8 bytes: quantity (56 bits) + flags (8 bits)
}

const FLAG_IS_BUY: u8 = 1;
const FLAG_IS_ACTIVE: u8 = 2;

public fun unpack_order(order: &PackedOrder): (u64, u64, bool, bool) {
    let quantity = order.data >> 8;
    let flags = (order.data & 0xFF) as u8;
    (
        order.price,
        quantity,
        (flags & FLAG_IS_BUY) != 0,
        (flags & FLAG_IS_ACTIVE) != 0
    )
}
```

#### Bit Packing Pattern

```move
module my_addr::bit_packing {
    /// Pack multiple values into single u128
    struct PackedData has store {
        // Layout: [amount: 64][timestamp: 32][user_type: 8][status: 8][reserved: 16]
        packed: u128,
    }

    const SHIFT_TIMESTAMP: u8 = 64;
    const SHIFT_USER_TYPE: u8 = 96;
    const SHIFT_STATUS: u8 = 104;

    const MASK_AMOUNT: u128 = 0xFFFFFFFFFFFFFFFF;                    // 64 bits
    const MASK_TIMESTAMP: u128 = 0xFFFFFFFF << 64;                   // 32 bits
    const MASK_USER_TYPE: u128 = 0xFF << 96;                         // 8 bits
    const MASK_STATUS: u128 = 0xFF << 104;                           // 8 bits

    public fun pack(amount: u64, timestamp: u32, user_type: u8, status: u8): PackedData {
        PackedData {
            packed: (amount as u128) |
                    ((timestamp as u128) << SHIFT_TIMESTAMP) |
                    ((user_type as u128) << SHIFT_USER_TYPE) |
                    ((status as u128) << SHIFT_STATUS)
        }
    }

    public fun get_amount(data: &PackedData): u64 {
        (data.packed & MASK_AMOUNT) as u64
    }

    public fun get_timestamp(data: &PackedData): u32 {
        ((data.packed & MASK_TIMESTAMP) >> SHIFT_TIMESTAMP) as u32
    }
}
```

### 3. Storage Patterns

#### Lazy Storage Pattern

```move
module my_addr::lazy_storage {
    /// Don't store computed values
    struct Pool has key {
        total_shares: u64,
        total_assets: u64,
        // DON'T store share_price - compute when needed
    }

    /// Compute on demand
    public fun get_share_price(pool: &Pool): u64 {
        if (pool.total_shares == 0) {
            INITIAL_SHARE_PRICE
        } else {
            pool.total_assets * PRECISION / pool.total_shares
        }
    }
}
```

#### Storage Recycling Pattern

```move
module my_addr::storage_recycling {
    use std::table::{Self, Table};

    struct ObjectPool has key {
        available: vector<Object<Item>>,
        in_use: Table<address, Object<Item>>,
    }

    /// Reuse objects instead of creating new ones
    public fun acquire_item(
        pool: &mut ObjectPool,
        user: address
    ): Object<Item> {
        let item = if (vector::length(&pool.available) > 0) {
            // Reuse existing object
            vector::pop_back(&mut pool.available)
        } else {
            // Only create new if none available
            create_new_item()
        };

        table::add(&mut pool.in_use, user, item);
        item
    }

    /// Return to pool for reuse
    public fun release_item(
        pool: &mut ObjectPool,
        user: address
    ) {
        let item = table::remove(&mut pool.in_use, user);
        reset_item(&item);
        vector::push_back(&mut pool.available, item);
    }
}
```

### 4. Off-Chain Storage

#### Hash-Based Verification

```move
module my_addr::offchain_storage {
    use std::hash;
    use std::bcs;

    struct Document has key {
        // Store only hash on-chain
        content_hash: vector<u8>,
        timestamp: u64,
        verified: bool,
    }

    /// Store document hash only
    public fun register_document(
        account: &signer,
        content_hash: vector<u8>
    ) {
        move_to(account, Document {
            content_hash,
            timestamp: timestamp::now_seconds(),
            verified: false,
        });
    }

    /// Verify off-chain content matches
    public fun verify_content(
        document: &Document,
        content: vector<u8>
    ): bool {
        let computed_hash = hash::sha3_256(content);
        computed_hash == document.content_hash
    }
}
```

#### IPFS Integration Pattern

```move
module my_addr::ipfs_storage {
    struct NFTMetadata has store {
        // Store only IPFS CID on-chain
        ipfs_cid: String,  // 46 bytes for CIDv1
        // Don't store actual metadata on-chain
    }

    struct Collection has key {
        name: String,
        // Store collection metadata IPFS hash
        metadata_cid: String,
        base_uri: String,  // "ipfs://"
    }

    public fun get_token_uri(token_id: u64, metadata: &NFTMetadata): String {
        // Construct URI for off-chain resolution
        string::append(&mut string::utf8(b"ipfs://"), metadata.ipfs_cid)
    }
}
```

### 5. Event-Based Storage

#### Use Events Instead of Storage

```move
module my_addr::event_storage {
    // BAD: Storing history on-chain
    struct BadAuction has key {
        bids: vector<Bid>,  // Grows unbounded!
    }

    // GOOD: Use events for history
    struct GoodAuction has key {
        highest_bid: u64,
        highest_bidder: address,
        // No bid history stored
    }

    struct BidEvent has drop, store {
        auction_id: u64,
        bidder: address,
        amount: u64,
        timestamp: u64,
    }

    public fun place_bid(auction: &mut GoodAuction, bidder: address, amount: u64) {
        // Update only current state
        auction.highest_bid = amount;
        auction.highest_bidder = bidder;

        // Emit event for history
        event::emit(BidEvent {
            auction_id: object::object_address(auction),
            bidder,
            amount,
            timestamp: timestamp::now_microseconds(),
        });
    }
}
```

### 6. Compression Techniques

#### String Compression

```move
module my_addr::compression {
    /// Use enums instead of strings
    const STATUS_PENDING: u8 = 0;
    const STATUS_ACTIVE: u8 = 1;
    const STATUS_COMPLETED: u8 = 2;

    // BAD: 8+ bytes per status
    struct BadOrder has store {
        status: String,  // "PENDING", "ACTIVE", "COMPLETED"
    }

    // GOOD: 1 byte per status
    struct GoodOrder has store {
        status: u8,  // 0, 1, 2
    }

    public fun status_to_string(status: u8): String {
        if (status == STATUS_PENDING) {
            string::utf8(b"PENDING")
        } else if (status == STATUS_ACTIVE) {
            string::utf8(b"ACTIVE")
        } else {
            string::utf8(b"COMPLETED")
        }
    }
}
```

#### Address Compression

```move
module my_addr::address_compression {
    use std::table::{Self, Table};

    /// Map addresses to smaller IDs
    struct AddressRegistry has key {
        address_to_id: Table<address, u32>,
        id_to_address: Table<u32, address>,
        next_id: u32,
    }

    /// Use u32 (4 bytes) instead of address (32 bytes)
    struct CompressedData has store {
        user_id: u32,      // 4 bytes vs 32 bytes
        referred_by_id: u32,  // 4 bytes vs 32 bytes
    }

    public fun register_address(
        registry: &mut AddressRegistry,
        addr: address
    ): u32 {
        if (table::contains(&registry.address_to_id, addr)) {
            *table::borrow(&registry.address_to_id, addr)
        } else {
            let id = registry.next_id;
            table::add(&mut registry.address_to_id, addr, id);
            table::add(&mut registry.id_to_address, id, addr);
            registry.next_id = id + 1;
            id
        }
    }
}
```

### 7. Dynamic vs Static Storage

#### Separate Hot and Cold Data

```move
module my_addr::data_separation {
    /// Frequently accessed data
    struct HotData has key {
        balance: u64,
        nonce: u64,
        last_update: u64,
    }

    /// Rarely accessed data
    struct ColdData has key {
        created_at: u64,
        metadata: String,
        preferences: vector<u8>,
    }

    /// Access hot data without loading cold data
    public fun get_balance(user: address): u64 acquires HotData {
        borrow_global<HotData>(user).balance
    }

    /// Load cold data only when needed
    public fun get_metadata(user: address): String acquires ColdData {
        borrow_global<ColdData>(user).metadata
    }
}
```

### 8. Computational Complexity Limits

#### Avoid Unbounded Loops

**CRITICAL:** User-controlled input that affects loop iterations can cause gas exhaustion.

❌ **WRONG - O(n²) complexity on user input:**

```move
// Checking for duplicates in owner list
while (i < num_owners) {
    let owner = *vector::borrow(&owners, i);
    let j = i + 1;
    while (j < num_owners) {
        assert!(*vector::borrow(&owners, j) != owner, E_DUPLICATE_OWNER);
        j = j + 1;
    };
    i = i + 1;
};
// For 100 owners, this does 10,000 iterations!
```

✅ **CORRECT - Add maximum size limits:**

```move
const MAX_OWNERS: u64 = 50;
const MAX_BATCH_SIZE: u64 = 100;

// Validate input size FIRST
assert!(vector::length(&owners) <= MAX_OWNERS, E_TOO_MANY_OWNERS);

// Then process (still O(n²), but bounded)
while (i < num_owners) {
    // ... duplicate check
};
```

**Better Solution - Use SmartTable for O(1) lookups:**

```move
use aptos_std::smart_table::{Self, SmartTable};

// Check duplicates in O(n) time
let seen = smart_table::new<address, bool>();
let i = 0;
while (i < num_owners) {
    let owner = *vector::borrow(&owners, i);
    assert!(!smart_table::contains(&seen, &owner), E_DUPLICATE_OWNER);
    smart_table::add(&mut seen, owner, true);
    i = i + 1;
};
```

#### Recommended Maximum Limits

Based on gas cost analysis:

| Operation                      | Recommended Max | Rationale            |
| ------------------------------ | --------------- | -------------------- |
| Vector iteration (single loop) | 1000 items      | ~100K gas            |
| Nested loops (O(n²))           | 50 items        | 2500 iterations max  |
| Batch operations               | 100 items       | User experience      |
| Owner/member lists             | 50 members      | Multi-sig complexity |
| Token batch mints              | 100 recipients  | Transaction size     |

**Always Add Size Constants:**

```move
/// Maximum number of items in a batch operation
const MAX_BATCH_SIZE: u64 = 100;

/// Maximum number of owners in a multi-sig wallet
const MAX_OWNERS: u64 = 50;

/// Maximum number of proposals in a single query
const MAX_PROPOSALS_PER_PAGE: u64 = 100;
```

#### Pagination Pattern

For large datasets, implement pagination:

```move
#[view]
public fun get_proposals_paginated(
    wallet: Object<MultiSigWallet>,
    start_index: u64,
    limit: u64
): vector<ProposalInfo> {
    assert!(limit <= MAX_PROPOSALS_PER_PAGE, E_LIMIT_TOO_HIGH);

    let wallet_data = borrow_global<MultiSigWallet>(object_address);
    let total = vector::length(&wallet_data.proposal_ids);

    let end_index = min(start_index + limit, total);
    let results = vector::empty<ProposalInfo>();

    let i = start_index;
    while (i < end_index) {
        let proposal_id = *vector::borrow(&wallet_data.proposal_ids, i);
        vector::push_back(&mut results, get_proposal_info(proposal_id));
        i = i + 1;
    };

    results
}
```

### 9. Multi-Asset Storage Patterns

#### Problem: Supporting Multiple Different Assets

When users need to hold multiple different assets (tokens, NFTs, etc.), avoid hardcoding single asset fields.

❌ **WRONG - Limited to single asset per user:**

```move
struct UserPosition has key {
    collateral_amount: u128,
    collateral_asset: address,  // Only ONE collateral type!
    borrowed_amount: u128,
    borrowed_asset: address,    // Only ONE borrowed type!
    last_update_time: u64,
}
```

**Problems:**

- User can only have one collateral asset
- Cannot diversify collateral
- Requires closing position to switch assets
- Breaking change to upgrade later

❌ **WRONG - Hardcoded asset fields:**

```move
struct Portfolio has key {
    usdc_amount: u64,
    apt_amount: u64,
    btc_amount: u64,
    // What about new tokens?
}
```

**Problems:**

- Fixed set of assets
- Cannot add new tokens without upgrade
- Wastes storage for unused assets

#### Solution 1: SmartTable for Asset Mapping

✅ **CORRECT - Flexible multi-asset support:**

```move
use aptos_std::smart_table::{Self, SmartTable};

struct UserPosition has key {
    /// Maps asset address -> amount held
    collateral: SmartTable<address, u128>,

    /// Maps asset address -> amount borrowed
    borrowed: SmartTable<address, u128>,

    /// Maps asset address -> interest accrued
    interest_accrued: SmartTable<address, u128>,

    /// Last interest update time per asset
    last_update_time: SmartTable<address, u64>,
}
```

**Benefits:**

- Unlimited asset types
- Pay storage only for assets used
- Can add new assets without upgrade
- User can diversify collateral

**Usage Example:**

```move
public entry fun deposit_collateral(
    user: &signer,
    asset: Object<Metadata>,
    amount: u64
) acquires UserPosition {
    let user_addr = signer::address_of(user);

    if (!exists<UserPosition>(user_addr)) {
        // Initialize position with empty tables
        move_to(user, UserPosition {
            collateral: smart_table::new(),
            borrowed: smart_table::new(),
            interest_accrued: smart_table::new(),
            last_update_time: smart_table::new(),
        });
    };

    let position = borrow_global_mut<UserPosition>(user_addr);
    let asset_addr = object::object_address(&asset);

    // Update or add collateral for this asset
    if (smart_table::contains(&position.collateral, &asset_addr)) {
        let current = smart_table::borrow_mut(&mut position.collateral, &asset_addr);
        *current = *current + (amount as u128);
    } else {
        smart_table::add(&mut position.collateral, asset_addr, amount as u128);
    };

    // Transfer tokens
    primary_fungible_store::transfer(user, asset, protocol_address, amount);
}
```

#### Solution 2: Vector of Structs (for small sets)

For a small, bounded number of assets (< 10), you can use a vector:

✅ **CORRECT - For small asset counts:**

```move
struct AssetBalance has store, drop {
    asset: address,
    amount: u64,
}

const MAX_ASSETS_PER_USER: u64 = 10;

struct Portfolio has key {
    assets: vector<AssetBalance>,
}

public entry fun add_asset(
    user: &signer,
    asset: address,
    amount: u64
) acquires Portfolio {
    let portfolio = borrow_global_mut<Portfolio>(signer::address_of(user));

    // Linear search for existing asset
    let i = 0;
    let len = vector::length(&portfolio.assets);
    let found = false;

    while (i < len) {
        let balance = vector::borrow_mut(&mut portfolio.assets, i);
        if (balance.asset == asset) {
            balance.amount = balance.amount + amount;
            found = true;
            break
        };
        i = i + 1;
    };

    if (!found) {
        assert!(len < MAX_ASSETS_PER_USER, E_TOO_MANY_ASSETS);
        vector::push_back(&mut portfolio.assets, AssetBalance { asset, amount });
    };
}
```

**When to Use:**

- Small, bounded number of assets (< 10)
- Simple iteration patterns
- Simpler than SmartTable for beginners

**When NOT to Use:**

- Unbounded asset types
- DeFi protocols (use SmartTable)
- High-frequency trading

#### Solution 3: Multiple Tables (for different asset categories)

For complex protocols with different asset categories:

```move
use aptos_std::smart_table::{Self, SmartTable};

struct DeFiPosition has key {
    /// Fungible assets as collateral
    collateral_tokens: SmartTable<address, u128>,

    /// NFTs held as collateral
    collateral_nfts: SmartTable<address, vector<address>>,  // token -> nft IDs

    /// Borrowed fungible assets
    borrowed_tokens: SmartTable<address, u128>,

    /// Reward tokens earned
    rewards: SmartTable<address, u128>,
}
```

**Benefits:**

- Type safety (separate tables for tokens vs NFTs)
- Clear categorization
- Efficient queries per category

#### Querying Multi-Asset Positions

**Get all collateral assets:**

```move
#[view]
public fun get_all_collateral(user_addr: address): vector<AssetInfo> acquires UserPosition {
    let position = borrow_global<UserPosition>(user_addr);
    let results = vector::empty<AssetInfo>();

    // SmartTable iteration
    smart_table::for_each_ref(&position.collateral, |asset_addr, amount| {
        vector::push_back(&mut results, AssetInfo {
            asset: *asset_addr,
            amount: *amount,
        });
    });

    results
}
```

**Get collateral value (summed across all assets):**

```move
#[view]
public fun get_total_collateral_value(user_addr: address): u128 acquires UserPosition {
    let position = borrow_global<UserPosition>(user_addr);
    let total_value = 0u128;

    smart_table::for_each_ref(&position.collateral, |asset_addr, amount| {
        let price = oracle::get_price(*asset_addr);
        let value = (*amount) * price;
        total_value = total_value + value;
    });

    total_value
}
```

#### Best Practices Summary

1. **Use SmartTable for unlimited asset types** (DeFi protocols)
2. **Use Vector<Struct> for small, bounded sets** (< 10 assets)
3. **Add MAX_ASSETS constants** even with SmartTable (sanity check)
4. **Separate tables by asset category** for type safety
5. **Provide view functions** to query all user assets
6. **Never hardcode specific token addresses** in user structs

## Storage Calculation

### Size Estimation

```move
module my_addr::size_calc {
    /// Calculate struct size
    /// Base types:
    /// - bool: 1 byte
    /// - u8: 1 byte
    /// - u16: 2 bytes
    /// - u32: 4 bytes
    /// - u64: 8 bytes
    /// - u128: 16 bytes
    /// - u256: 32 bytes
    /// - address: 32 bytes
    /// - Object<T>: 32 bytes

    struct Example has store {
        id: u64,           // 8 bytes
        owner: address,    // 32 bytes
        data: vector<u8>,  // 1 byte per element + overhead
        name: String,      // 1 byte per char + overhead
    }

    /// Vector overhead: ~40 bytes + elements
    /// String overhead: ~40 bytes + characters
    /// Table overhead: ~100 bytes + entries
}
```

## Best Practices Checklist

### Design Phase

- [ ] Choose minimal integer sizes
- [ ] Pack related boolean flags
- [ ] Consider off-chain storage for large data
- [ ] Use events for historical data
- [ ] Design for storage reuse

### Implementation Phase

- [ ] Bit-pack where appropriate
- [ ] Use Tables for key-value data
- [ ] Implement lazy computation
- [ ] Separate hot and cold data
- [ ] Compress repetitive data

### Optimization Phase

- [ ] Profile storage usage
- [ ] Remove redundant fields
- [ ] Consolidate related data
- [ ] Review access patterns
- [ ] Test storage costs

## Common Anti-Patterns

### 1. Storing Computed Values

```move
// BAD
struct BadPool has key {
    total_x: u64,
    total_y: u64,
    k: u128,  // Can be computed: total_x * total_y
}

// GOOD
struct GoodPool has key {
    total_x: u64,
    total_y: u64,
    // Compute k when needed
}
```

### 2. Unbounded Collections

```move
// BAD
struct BadDAO has key {
    all_proposals: vector<Proposal>,  // Grows forever!
}

// GOOD
struct GoodDAO has key {
    active_proposals: Table<u64, Proposal>,
    next_proposal_id: u64,
}
```

### 3. Redundant Storage

```move
// BAD
struct BadUser has key {
    address: address,     // Redundant - already in global storage key
    balances: Table<address, u64>,
    balance_count: u64,   // Redundant - can use table::length
}

// GOOD
struct GoodUser has key {
    balances: Table<address, u64>,
}
```

## Measurement and Monitoring

```move
#[test]
public fun test_storage_size() {
    let account = account::create_account_for_test(@0x1);

    // Measure before
    let size_before = account::get_storage_usage(&account);

    // Create storage
    create_large_struct(&account);

    // Measure after
    let size_after = account::get_storage_usage(&account);

    // Calculate cost
    let bytes_used = size_after - size_before;
    let storage_cost = bytes_used * STORAGE_PRICE_PER_BYTE;

    assert!(bytes_used < MAX_ACCEPTABLE_SIZE, E_TOO_LARGE);
}
```
