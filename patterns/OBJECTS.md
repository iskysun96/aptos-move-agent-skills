# Aptos Move V2 Object Model Patterns

**Purpose:** Comprehensive guide to using the Aptos object model correctly and securely.

**Target:** AI assistants generating Move V2 smart contracts

---

## Overview

The Aptos object model is the **modern, preferred way** to represent resources and NFTs. Objects provide:

- **Type safety**: Strong typing with `Object<T>` instead of raw addresses
- **Ownership tracking**: Built-in ownership verification with `object::owner()`
- **Flexible permissions**: Control transfers, deletions, and extensions
- **Composability**: Objects can own other objects, creating complex structures

**Golden Rule:** Always use `Object<T>` for object references in V2 code. Raw addresses are legacy patterns.

---

## Core Concepts

### Object vs Address

**OLD (Legacy - Avoid):**

```move
// Using raw addresses (V1 pattern)
public entry fun transfer_item(user: &signer, item_addr: address, to: address) {
    // No type safety, manual ownership checks
}
```

**NEW (Modern - Always Use):**

```move
// Using typed objects (V2 pattern)
public entry fun transfer_item(user: &signer, item: Object<Item>, to: address) {
    // Type-safe, built-in ownership tracking
    assert!(object::owner(item) == signer::address_of(user), E_NOT_OWNER);
}
```

### ConstructorRef Lifecycle

**Critical Understanding:** `ConstructorRef` is a one-time-use builder that:

1. Lets you generate refs (TransferRef, DeleteRef, ExtendRef)
2. Lets you get the object's signer to store data
3. **Automatically destroyed** when function returns
4. **NEVER RETURNED** from functions (security risk)

**Constructor Pattern Flow:**

```
ConstructorRef created
    ↓
Generate all needed refs (TransferRef, DeleteRef, etc.)
    ↓
Get object signer with generate_signer()
    ↓
Store data with move_to()
    ↓
Return Object<T> reference
    ↓
ConstructorRef automatically destroyed
```

---

## Pattern 1: Creating Objects

### Standard Object Creation

**When to use:** Creating unique objects with random addresses (NFTs, game items, user profiles)

```move
module my_addr::item {
    use std::string::String;
    use aptos_framework::object::{Self, Object, ConstructorRef};

    /// Item stored in object
    struct Item has key {
        name: String,
        owner: address,
        // Store refs for future operations
        transfer_ref: object::TransferRef,
        delete_ref: object::DeleteRef,
    }

    /// Error codes
    const E_NOT_OWNER: u64 = 1;

    /// Create item object
    /// Returns Object<Item> for type safety
    public fun create_item(creator: &signer, name: String): Object<Item> {
        let creator_addr = signer::address_of(creator);

        // 1. Create object (random address)
        let constructor_ref = object::create_object(creator_addr);

        // 2. Generate object signer (needed for move_to)
        let object_signer = object::generate_signer(&constructor_ref);

        // 3. Generate all refs BEFORE constructor_ref is destroyed
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let delete_ref = object::generate_delete_ref(&constructor_ref);

        // 4. Store data in object
        move_to(&object_signer, Item {
            name,
            owner: creator_addr,
            transfer_ref,
            delete_ref,
        });

        // 5. Return typed object reference
        // constructor_ref is automatically destroyed here
        object::object_from_constructor_ref<Item>(&constructor_ref)
    }
}
```

### Named Object Creation

**When to use:** Creating singletons or well-known objects (registries, global config, protocol state)

```move
module my_addr::registry {
    use std::string::String;
    use aptos_framework::object::{Self, Object};

    /// Global registry (singleton)
    struct Registry has key {
        items: vector<Object<Item>>,
        admin: address,
    }

    /// Create named object (deterministic address)
    /// Can be retrieved by anyone who knows the seed
    public fun create_registry(creator: &signer): Object<Registry> {
        let creator_addr = signer::address_of(creator);

        // Create named object with seed
        // Address is deterministic: hash(creator_addr, seed)
        let constructor_ref = object::create_named_object(
            creator,
            b"REGISTRY_V1" // Seed bytes
        );

        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Registry {
            items: vector::empty(),
            admin: creator_addr,
        });

        object::object_from_constructor_ref<Registry>(&constructor_ref)
    }

    /// Retrieve named object
    public fun get_registry(creator_addr: address): Object<Registry> {
        // Reconstruct address from seed
        let registry_addr = object::create_object_address(&creator_addr, b"REGISTRY_V1");
        object::address_to_object<Registry>(registry_addr)
    }
}
```

### Sticky Object Creation (Cannot be transferred)

**When to use:** Creating non-transferable objects (soulbound tokens, credentials, identity)

```move
/// Create non-transferable object
public fun create_soulbound_badge(recipient: &signer, badge_type: u8): Object<Badge> {
    let constructor_ref = object::create_object(signer::address_of(recipient));

    // Disable transfers by NOT generating TransferRef
    // Object becomes "sticky" - permanently bound to owner

    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Badge {
        badge_type,
        // No transfer_ref field - cannot be transferred
    });

    object::object_from_constructor_ref<Badge>(&constructor_ref)
}
```

---

## Pattern 2: Object Ownership & Verification

### Verifying Object Ownership

**Always verify ownership before operations:**

```move
public entry fun update_item(
    owner: &signer,
    item: Object<Item>,
    new_name: String
) acquires Item {
    let owner_addr = signer::address_of(owner);

    // ✅ CORRECT: Verify ownership using object::owner()
    assert!(object::owner(item) == owner_addr, E_NOT_OWNER);

    // Safe to proceed
    let item_addr = object::object_address(&item);
    let item_data = borrow_global_mut<Item>(item_addr);
    item_data.name = new_name;
}
```

### Checking if Object Exists

```move
public fun item_exists(item_addr: address): bool {
    exists<Item>(item_addr)
}

public fun is_item_owner(user_addr: address, item: Object<Item>): bool {
    object::owner(item) == user_addr
}
```

---

## Pattern 3: Object Transfers

### CRITICAL: Understanding Transfer Functions

There are TWO ways to transfer objects, and using the wrong one will cause errors:

**1. `object::transfer()` - For ungated transfers**

- Requires ungated transfers to be ENABLED
- Anyone can call this function
- Used for freely transferable objects

**2. `object::transfer_with_ref()` - For controlled transfers**

- Requires a `TransferRef` (generated during object creation)
- Works even when ungated transfers are DISABLED
- Used for controlled, permission-based transfers

### Common Mistake: Wrong Transfer Function

**❌ WRONG: Using `object::transfer()` after disabling ungated transfers**

```move
public fun create_and_transfer_item(creator: &signer) {
    let constructor_ref = object::create_object(signer::address_of(creator));
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Disable ungated transfers
    object::disable_ungated_transfer(&transfer_ref);

    let token_obj = object::object_from_constructor_ref<Token>(&constructor_ref);

    // ❌ ERROR: "The object does not have ungated transfers enabled"
    object::transfer(creator, token_obj, recipient_addr);
}
```

**✅ CORRECT: Using `object::transfer_with_ref()` when ungated transfers are disabled**

```move
public fun create_and_transfer_item(creator: &signer, recipient: address): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);

    // Disable ungated transfers
    object::disable_ungated_transfer(&transfer_ref);

    let object_signer = object::generate_signer(&constructor_ref);

    // ✅ Transfer BEFORE storing (while we still have transfer_ref)
    let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, recipient);

    // Store data with transfer_ref for future transfers
    move_to(&object_signer, Item {
        name: string::utf8(b"Item"),
        transfer_ref,  // Store for later use
    });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}
```

### Transfer with TransferRef

**When to use:** Controlled transfers that you manage (marketplaces, escrows, NFTs with disabled ungated transfers)

```move
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    recipient: address
) acquires Item {
    // Verify ownership
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    // Get item data
    let item_addr = object::object_address(&item);
    let item_data = borrow_global_mut<Item>(item_addr);

    // Transfer using stored TransferRef
    // CRITICAL: Must use transfer_with_ref when ungated transfers are disabled
    let linear_transfer_ref = object::generate_linear_transfer_ref(&item_data.transfer_ref);
    object::transfer_with_ref(linear_transfer_ref, recipient);

    // Update owner tracking
    item_data.owner = recipient;
}
```

### Ungated Transfer (Allow anyone to transfer)

**When to use:** Freely transferable objects (standard NFTs with ungated transfers, currencies)

```move
/// Enable ungated transfers in constructor
public fun create_transferable_item(creator: &signer, name: String): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Enable ungated transfers - anyone can initiate transfers
    object::enable_ungated_transfer(&constructor_ref);

    let object_signer = object::generate_signer(&constructor_ref);

    // No need to store TransferRef if using ungated transfers
    move_to(&object_signer, Item {
        name,
        owner: signer::address_of(creator),
    });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}

/// Transfer function (callable by anyone if ungated)
public entry fun transfer_ungated_item(
    sender: &signer,
    item: Object<Item>,
    recipient: address
) {
    // ✅ Can use object::transfer() because ungated transfers are enabled
    object::transfer(sender, item, recipient);
}
```

### Decision Tree: Which Transfer Function?

```
Did you call object::disable_ungated_transfer()?
├─ YES → Use object::transfer_with_ref()
│         Requires: TransferRef stored in your struct
│
└─ NO (ungated transfers enabled by default)
    └─ Use object::transfer()
       Works without TransferRef
```

---

## Pattern 4: Object Deletion

### Delete with DeleteRef

**When to use:** Objects that can be destroyed (temporary items, expired tickets)

```move
public entry fun burn_item(owner: &signer, item: Object<Item>) acquires Item {
    // Verify ownership
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    // Get item data
    let item_addr = object::object_address(&item);
    let Item {
        name: _,
        owner: _,
        transfer_ref: _,
        delete_ref,
    } = move_from<Item>(item_addr);

    // Delete object using stored DeleteRef
    object::delete(delete_ref);
}
```

### Permanent Objects (Cannot be deleted)

```move
/// Create permanent object by NOT generating DeleteRef
public fun create_permanent_record(creator: &signer, data: String): Object<Record> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Don't generate DeleteRef - object becomes permanent

    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Record {
        data,
        // No delete_ref field - cannot be deleted
    });

    object::object_from_constructor_ref<Record>(&constructor_ref)
}
```

---

## Pattern 5: Nested Objects (Objects Owning Objects)

### Parent-Child Relationships

```move
module my_addr::collection {
    use aptos_framework::object::{Self, Object};

    struct Collection has key {
        name: String,
        items: vector<Object<Item>>,
    }

    struct Item has key {
        name: String,
        parent: Object<Collection>,
    }

    /// Create collection
    public fun create_collection(creator: &signer, name: String): Object<Collection> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Collection {
            name,
            items: vector::empty(),
        });

        object::object_from_constructor_ref<Collection>(&constructor_ref)
    }

    /// Add item to collection (item becomes child of collection)
    public fun add_item(
        owner: &signer,
        collection: Object<Collection>,
        item_name: String
    ): Object<Item> acquires Collection {
        // Verify ownership of collection
        assert!(object::owner(collection) == signer::address_of(owner), E_NOT_OWNER);

        // Create item owned by collection object
        let collection_addr = object::object_address(&collection);
        let constructor_ref = object::create_object(collection_addr);
        let object_signer = object::generate_signer(&constructor_ref);

        let item_obj = object::object_from_constructor_ref<Item>(&constructor_ref);

        move_to(&object_signer, Item {
            name: item_name,
            parent: collection,
        });

        // Add to collection's item list
        let collection_data = borrow_global_mut<Collection>(collection_addr);
        vector::push_back(&mut collection_data.items, item_obj);

        item_obj
    }
}
```

---

## Pattern 6: Converting Between Object and Address

### Object → Address

```move
public fun get_item_address(item: Object<Item>): address {
    object::object_address(&item)
}
```

### Address → Object (Type-safe conversion)

```move
public fun address_to_item(item_addr: address): Object<Item> {
    // Verify that Item resource exists at address
    assert!(exists<Item>(item_addr), E_ITEM_NOT_FOUND);

    // Convert to typed object
    object::address_to_object<Item>(item_addr)
}
```

---

## Pattern 7: ExtendRef for Future Extensions

### Using ExtendRef for Dynamic Capabilities

```move
struct Item has key {
    name: String,
    extend_ref: object::ExtendRef, // Store for future extensions
}

/// Create item with extension capability
public fun create_extensible_item(creator: &signer, name: String): Object<Item> {
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Generate ExtendRef for future extensions
    let extend_ref = object::generate_extend_ref(&constructor_ref);

    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Item {
        name,
        extend_ref,
    });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}

/// Add new capability to item later
public fun add_metadata_to_item(item: Object<Item>) acquires Item {
    let item_addr = object::object_address(&item);
    let item_data = borrow_global<Item>(item_addr);

    // Generate signer from stored ExtendRef
    let object_signer = object::generate_signer_for_extending(&item_data.extend_ref);

    // Add new resource to same object
    move_to(&object_signer, ItemMetadata {
        description: string::utf8(b"Added later"),
        rarity: 1,
    });
}
```

---

## Anti-Patterns (NEVER DO THIS)

### ❌ Returning ConstructorRef

```move
// ❌ DANGEROUS: Caller can destroy object
public fun create_item_wrong(creator: &signer): ConstructorRef {
    let constructor_ref = object::create_object(signer::address_of(creator));
    // BAD: Returning constructor_ref gives caller too much power
    constructor_ref
}
```

**Why it's wrong:** Caller can delete the object, extract object signer, or otherwise manipulate it unsafely.

**Correct version:** Return `Object<T>` instead (see Pattern 1).

### ❌ Using Addresses Instead of Objects

```move
// ❌ LEGACY: No type safety
public fun transfer_item(owner: &signer, item_addr: address, to: address) {
    // Have to manually verify resource type, ownership, etc.
}
```

**Why it's wrong:** No type safety, manual checks required, error-prone.

**Correct version:** Use `Object<Item>` parameter (see Pattern 2).

### ❌ Storing Object Signer

```move
// ❌ DANGEROUS: Storing object signer gives permanent control
struct Item has key {
    object_signer: signer, // Can't even do this - signer doesn't have `store` ability
}
```

**Why it's wrong:** Object signer should only be used during construction or with ExtendRef.

**Correct version:** Store ExtendRef if you need future signer access (see Pattern 7).

### ❌ Not Verifying Ownership

```move
// ❌ INSECURE: Anyone can update any item
public entry fun update_item(item: Object<Item>, new_name: String) acquires Item {
    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = new_name; // No ownership check!
}
```

**Why it's wrong:** Anyone can modify any object without permission.

**Correct version:** Always verify ownership (see Pattern 2).

---

## Reference Table: Object Functions

| Function                                                  | Purpose                             | When to Use                            |
| --------------------------------------------------------- | ----------------------------------- | -------------------------------------- |
| `object::create_object(owner_addr)`                       | Create random-address object        | NFTs, items, unique entities           |
| `object::create_named_object(creator, seed)`              | Create deterministic-address object | Singletons, registries, protocol state |
| `object::generate_signer(&ConstructorRef)`                | Get object's signer                 | During construction to store data      |
| `object::generate_transfer_ref(&ConstructorRef)`          | Generate transfer capability        | When you need controlled transfers     |
| `object::generate_delete_ref(&ConstructorRef)`            | Generate delete capability          | When object can be destroyed           |
| `object::generate_extend_ref(&ConstructorRef)`            | Generate extension capability       | When you need future extensions        |
| `object::object_from_constructor_ref<T>(&ConstructorRef)` | Convert ConstructorRef to Object<T> | Return from constructor function       |
| `object::owner(obj)`                                      | Get object owner address            | Verify ownership                       |
| `object::object_address(&obj)`                            | Get object's address                | Access stored data                     |
| `object::address_to_object<T>(addr)`                      | Convert address to typed object     | When you have address, need Object<T>  |
| `object::transfer_with_ref(LinearTransferRef, to)`        | Transfer with TransferRef           | Controlled transfers                   |
| `object::enable_ungated_transfer(&ConstructorRef)`        | Allow free transfers                | Standard NFTs                          |
| `object::delete(DeleteRef)`                               | Delete object                       | Destroy burnable objects               |

---

## Checklist for Object Implementation

When implementing objects, verify:

- [ ] Using `Object<T>` in function signatures (not `address`)
- [ ] ConstructorRef is never returned from functions
- [ ] All needed refs generated in constructor before ConstructorRef destroyed
- [ ] Ownership verified with `object::owner()` in all entry functions
- [ ] Object signer only used during construction (or with ExtendRef)
- [ ] Refs stored in struct for later use (TransferRef, DeleteRef, ExtendRef)
- [ ] Type parameter correct: `Object<MyType>` matches stored resource
- [ ] Named objects use unique seeds to avoid collisions
- [ ] Ungated transfers only enabled when appropriate
- [ ] DeleteRef only generated for truly burnable objects

---

## Additional Resources

**Official Documentation:**

- Object Model: https://aptos.dev/build/smart-contracts/object
- Creating Objects: https://aptos.dev/build/smart-contracts/object/creating-objects
- Using Objects: https://aptos.dev/en/build/smart-contracts/object/using-objects
- Configurable Properties: https://aptos.dev/build/smart-contracts/object/configuring-objects

**Example Code:**

- aptos-core/aptos-move/move-examples/token_objects
- aptos-core/aptos-move/move-examples/mint_nft
- aptos-core/aptos-move/move-examples/fungible_asset

**Related Patterns:**

- `SECURITY.md` - Security implications of object usage
- `TESTING.md` - Testing object operations
- `MOVE_V2_SYNTAX.md` - Modern syntax with objects

---

**Remember:** Objects are the modern, secure, type-safe way to handle resources in Aptos Move V2. Always prefer objects
over raw addresses.
