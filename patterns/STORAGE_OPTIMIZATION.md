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
