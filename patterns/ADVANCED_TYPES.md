# Advanced Types in Aptos Move V2

This guide covers advanced type system features including phantom types, generic programming, type constraints, and
complex struct patterns.

## Phantom Types

Phantom types are type parameters that don't appear in the struct's fields but provide compile-time type safety.

### Basic Phantom Type Pattern

```move
module my_addr::phantom_example {
    /// Phantom type for compile-time guarantees
    struct Vault<phantom T> has key {
        // T doesn't appear in fields
        balance: u64,
    }

    /// Type witnesses for phantom types
    struct USD {}
    struct EUR {}

    /// Create typed vault
    public fun create_vault<T>(creator: &signer): Object<Vault<T>> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Vault<T> {
            balance: 0,
        });

        object::object_from_constructor_ref<Vault<T>>(&constructor_ref)
    }

    /// Type-safe operations
    public fun deposit<T>(vault: Object<Vault<T>>, amount: u64) acquires Vault {
        let vault_data = borrow_global_mut<Vault<T>>(
            object::object_address(&vault)
        );
        vault_data.balance = vault_data.balance + amount;
    }
}
```

### Advanced Phantom Type Uses

#### 1. State Machine Type Safety

```move
module my_addr::state_machine {
    /// Phantom types for states
    struct Created {}
    struct Active {}
    struct Completed {}

    struct Order<phantom State> has key {
        id: u64,
        amount: u64,
    }

    /// Can only create in Created state
    public fun create_order(creator: &signer, id: u64): Object<Order<Created>> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Order<Created> {
            id,
            amount: 0,
        });

        object::object_from_constructor_ref<Order<Created>>(&constructor_ref)
    }

    /// State transition: Created -> Active
    public fun activate_order(
        order: Object<Order<Created>>,
        amount: u64
    ): Object<Order<Active>> acquires Order {
        let order_addr = object::object_address(&order);
        let Order { id, amount: _ } = move_from<Order<Created>>(order_addr);

        // Create new order with Active state
        let constructor_ref = object::create_object(order_addr);
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Order<Active> {
            id,
            amount,
        });

        object::object_from_constructor_ref<Order<Active>>(&constructor_ref)
    }

    /// Can only complete Active orders
    public fun complete_order(
        order: Object<Order<Active>>
    ): Object<Order<Completed>> acquires Order {
        // Type system ensures only Active orders can be completed
        transition_to_completed(order)
    }
}
```

#### 2. Permission Types

```move
module my_addr::permissions {
    /// Phantom types for permissions
    struct ReadOnly {}
    struct ReadWrite {}
    struct Admin {}

    struct Resource<phantom Perm> has key {
        data: vector<u8>,
        owner: address,
    }

    /// Read allowed for all permission types
    public fun read<Perm>(
        resource: &Resource<Perm>
    ): &vector<u8> {
        &resource.data
    }

    /// Write only for ReadWrite and Admin
    public fun write<Perm>(
        resource: &mut Resource<Perm>,
        data: vector<u8>
    ) {
        // Runtime check for permission type
        assert!(
            type_of<Perm>() == type_of<ReadWrite>() ||
            type_of<Perm>() == type_of<Admin>(),
            E_INSUFFICIENT_PERMISSION
        );
        resource.data = data;
    }
}
```

## Generic Programming

### Type Parameters and Constraints

```move
module my_addr::generics {
    use std::signer;
    use aptos_framework::coin::{Self, Coin};

    /// Multiple type parameters with constraints
    struct Pool<phantom X, phantom Y> has key {
        reserve_x: Coin<X>,
        reserve_y: Coin<Y>,
        lp_supply: u64,
    }

    /// Generic function with constraints
    public fun create_pool<X, Y>(
        creator: &signer,
        initial_x: Coin<X>,
        initial_y: Coin<Y>
    ): Object<Pool<X, Y>> {
        // Type parameters X and Y must be valid Coin types
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Pool<X, Y> {
            reserve_x: initial_x,
            reserve_y: initial_y,
            lp_supply: 0,
        });

        object::object_from_constructor_ref<Pool<X, Y>>(&constructor_ref)
    }

    /// Swap with type inference
    public fun swap<X, Y>(
        pool: Object<Pool<X, Y>>,
        coin_in: Coin<X>
    ): Coin<Y> acquires Pool {
        let pool_data = borrow_global_mut<Pool<X, Y>>(
            object::object_address(&pool)
        );

        // Type system ensures correct coin types
        let amount_in = coin::value(&coin_in);
        coin::merge(&mut pool_data.reserve_x, coin_in);

        // Calculate output
        let output_amount = calculate_output<X, Y>(pool_data, amount_in);
        coin::extract(&mut pool_data.reserve_y, output_amount)
    }
}
```

### Generic Collections

```move
module my_addr::generic_collections {
    use std::vector;
    use std::table::{Self, Table};

    /// Generic registry pattern
    struct Registry<phantom T> has key {
        items: Table<u64, T>,
        next_id: u64,
    }

    /// Initialize generic registry
    public fun create_registry<T: store>(
        creator: &signer
    ): Object<Registry<T>> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Registry<T> {
            items: table::new(),
            next_id: 0,
        });

        object::object_from_constructor_ref<Registry<T>>(&constructor_ref)
    }

    /// Add item with auto-incrementing ID
    public fun add_item<T: store>(
        registry: Object<Registry<T>>,
        item: T
    ): u64 acquires Registry {
        let registry_data = borrow_global_mut<Registry<T>>(
            object::object_address(&registry)
        );

        let id = registry_data.next_id;
        table::add(&mut registry_data.items, id, item);
        registry_data.next_id = id + 1;

        id
    }

    /// Generic iteration helper
    inline fun for_each_item<T: store>(
        registry: &Registry<T>,
        f: |u64, &T|
    ) {
        let i = 0;
        while (i < registry.next_id) {
            if (table::contains(&registry.items, i)) {
                f(i, table::borrow(&registry.items, i));
            };
            i = i + 1;
        }
    }
}
```

## Advanced Struct Patterns

### 1. Wrapper Types with Abilities

```move
module my_addr::wrappers {
    /// Wrapper that adds abilities
    struct Droppable<T: store> has drop, store {
        inner: T,
    }

    /// Wrapper that removes abilities
    struct NoDrop<T> has store {
        inner: T,
    }

    /// Create droppable wrapper
    public fun make_droppable<T: store>(value: T): Droppable<T> {
        Droppable { inner: value }
    }

    /// Extract from wrapper
    public fun unwrap_droppable<T: store>(wrapper: Droppable<T>): T {
        let Droppable { inner } = wrapper;
        inner
    }
}
```

### 2. Recursive Types (via Objects)

```move
module my_addr::recursive_types {
    use aptos_framework::object::{Self, Object};

    /// Tree node with recursive structure
    struct TreeNode<T: store> has key {
        value: T,
        children: vector<Object<TreeNode<T>>>,
    }

    /// Create tree node
    public fun create_node<T: store>(
        creator: &signer,
        value: T
    ): Object<TreeNode<T>> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, TreeNode<T> {
            value,
            children: vector::empty(),
        });

        object::object_from_constructor_ref<TreeNode<T>>(&constructor_ref)
    }

    /// Add child to node
    public fun add_child<T: store>(
        parent: Object<TreeNode<T>>,
        child: Object<TreeNode<T>>
    ) acquires TreeNode {
        let parent_data = borrow_global_mut<TreeNode<T>>(
            object::object_address(&parent)
        );
        vector::push_back(&mut parent_data.children, child);
    }
}
```

### 3. Type Witnesses

```move
module my_addr::type_witnesses {
    /// One-time witness pattern
    struct INIT has drop {}

    /// Type witness for capability
    struct MintCap<phantom CoinType> has key, store {
        /// Ensures one capability per coin type
    }

    /// Initialize with witness
    public fun initialize<CoinType>(
        account: &signer,
        _witness: INIT  // Consumes witness
    ) {
        // INIT can only be created in init_module
        // This ensures one-time initialization
        move_to(account, MintCap<CoinType> {});
    }

    fun init_module(account: &signer) {
        // Create and use witness
        initialize<MyCoin>(account, INIT {});
    }
}
```

## Type-Level Programming

### 1. Type Equality Checks

```move
module my_addr::type_equality {
    use std::type_info;

    /// Check if two types are equal
    public fun types_equal<T1, T2>(): bool {
        type_info::type_of<T1>() == type_info::type_of<T2>()
    }

    /// Conditional logic based on types
    public fun process<Input, Output>(data: Input): Option<Output> {
        if (types_equal<Input, Output>()) {
            // Safe cast when types are equal
            option::some(unsafe_cast<Input, Output>(data))
        } else {
            option::none()
        }
    }
}
```

### 2. Type Tags for Runtime Decisions

```move
module my_addr::type_tags {
    struct TypedEvent<phantom T> has drop, store {
        type_name: String,
        data: vector<u8>,
    }

    /// Emit typed event
    public fun emit_typed<T>(data: vector<u8>) {
        event::emit(TypedEvent<T> {
            type_name: type_info::type_name<T>(),
            data,
        });
    }

    /// Type-based routing
    public fun route_by_type<T>(input: vector<u8>) {
        if (type_info::type_name<T>() == string::utf8(b"0x1::coin::Coin")) {
            handle_coin(input);
        } else if (type_info::type_name<T>() == string::utf8(b"0x1::nft::Token")) {
            handle_nft(input);
        } else {
            handle_generic<T>(input);
        }
    }
}
```

## Ability Combinations

### Complex Ability Patterns

```move
module my_addr::abilities {
    /// No abilities - can only exist in global storage
    struct Singleton {
        value: u64,
    }

    /// Store only - can be stored in other structs
    struct Component has store {
        data: vector<u8>,
    }

    /// Drop + Store - flexible value type
    struct Value has drop, store {
        amount: u64,
    }

    /// Copy + Drop + Store - fully copyable value
    struct Config has copy, drop, store {
        enabled: bool,
        threshold: u64,
    }

    /// Key - can exist in global storage
    struct Account has key {
        // Often contains non-droppable resources
        balance: Coin<AptosCoin>,
        components: vector<Component>,
    }

    /// Key + Store - can be stored and be a resource
    struct FlexibleResource has key, store {
        // Can be moved between accounts
        owner: address,
        data: Value,
    }
}
```

## Type Safety Patterns

### 1. Newtype Pattern

```move
module my_addr::newtype {
    /// Wrap primitive for type safety
    struct UserId has copy, drop, store {
        value: u64,
    }

    struct OrderId has copy, drop, store {
        value: u64,
    }

    /// Can't mix up UserId and OrderId
    public fun get_order(user: UserId, order: OrderId): Order {
        // Type system prevents passing OrderId as UserId
    }
}
```

### 2. Builder Pattern with Types

```move
module my_addr::builder {
    struct Builder<phantom T> {
        config: T,
    }

    struct Incomplete has drop {
        field1: Option<u64>,
        field2: Option<String>,
    }

    struct Complete has drop {
        field1: u64,
        field2: String,
    }

    public fun new(): Builder<Incomplete> {
        Builder {
            config: Incomplete {
                field1: option::none(),
                field2: option::none(),
            }
        }
    }

    public fun set_field1(
        builder: Builder<Incomplete>,
        value: u64
    ): Builder<Incomplete> {
        let Builder { config } = builder;
        Builder {
            config: Incomplete {
                field1: option::some(value),
                field2: config.field2,
            }
        }
    }

    public fun build(builder: Builder<Incomplete>): Option<Complete> {
        let Builder { config } = builder;
        if (option::is_some(&config.field1) && option::is_some(&config.field2)) {
            option::some(Complete {
                field1: option::extract(&mut config.field1),
                field2: option::extract(&mut config.field2),
            })
        } else {
            option::none()
        }
    }
}
```

## Best Practices

1. **Use phantom types for compile-time guarantees**
2. **Leverage type witnesses for one-time operations**
3. **Apply appropriate abilities based on use case**
4. **Use newtype pattern for domain modeling**
5. **Prefer generic solutions for reusability**
6. **Document type constraints clearly**
7. **Test type safety with negative tests**

## Common Patterns

### Generic Factory

```move
public fun create<T: key + store>(
    creator: &signer,
    initial_value: T
): Object<Container<T>> {
    // Generic object creation
}
```

### Type-Safe State Machine

```move
struct State<phantom S> has key {
    data: Data,
}

public fun transition<From, To>(
    state: Object<State<From>>
): Object<State<To>> {
    // Type-safe state transitions
}
```

### Polymorphic Collections

```move
struct Collection<T: store> has key {
    items: Table<u64, T>,
}

public fun add<T: store>(
    collection: &mut Collection<T>,
    item: T
) {
    // Works with any storable type
}
```
