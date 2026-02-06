# Advanced Types in Aptos Move V2

This guide covers advanced type system features including enum patterns, phantom types, generic programming, type
constraints, and complex struct patterns.

## Advanced Enum Patterns

Enums (Move 2.0+) enable powerful type-safe patterns beyond basic variant matching. See `MOVE_V2_SYNTAX.md` for basic
enum syntax.

### 1. State Machine with Enums

Enums are often better than phantom types for state machines when you need to store different data per state:

```move
module my_addr::order_state_machine {
    use std::string::String;

    /// Order states with variant-specific data
    enum OrderState has key, store {
        Pending { created_at: u64 },
        Active { created_at: u64, amount: u64 },
        Completed { created_at: u64, amount: u64, completed_at: u64 },
        Cancelled { reason: String },
    }

    struct Order has key {
        id: u64,
        owner: address,
        state: OrderState,
    }

    const E_INVALID_TRANSITION: u64 = 1;
    const E_NOT_OWNER: u64 = 2;

    /// Check if order is in a terminal state
    fun is_terminal(state: &OrderState): bool {
        state is OrderState::Completed | OrderState::Cancelled
    }

    /// Transition: Pending -> Active
    public entry fun activate_order(
        owner: &signer,
        order_obj: Object<Order>,
        amount: u64
    ) acquires Order {
        let order = borrow_global_mut<Order>(object::object_address(&order_obj));
        assert!(order.owner == signer::address_of(owner), E_NOT_OWNER);

        match (&order.state) {
            OrderState::Pending { created_at } => {
                order.state = OrderState::Active {
                    created_at: *created_at,
                    amount,
                };
            },
            _ => abort E_INVALID_TRANSITION,
        };
    }

    /// Transition: Active -> Completed
    public entry fun complete_order(
        owner: &signer,
        order_obj: Object<Order>,
        timestamp: u64
    ) acquires Order {
        let order = borrow_global_mut<Order>(object::object_address(&order_obj));
        assert!(order.owner == signer::address_of(owner), E_NOT_OWNER);

        match (&order.state) {
            OrderState::Active { created_at, amount } => {
                order.state = OrderState::Completed {
                    created_at: *created_at,
                    amount: *amount,
                    completed_at: timestamp,
                };
            },
            _ => abort E_INVALID_TRANSITION,
        };
    }
}
```

### 2. Recursive Data Structures

Enums with `Box` enable tree-like data structures:

```move
module my_addr::expression_tree {
    /// Arithmetic expression tree
    enum Expr has drop {
        Literal(u64),
        Add(Box<Expr>, Box<Expr>),
        Mul(Box<Expr>, Box<Expr>),
        Neg(Box<Expr>),
    }

    /// Evaluate expression recursively
    fun eval(expr: &Expr): u64 {
        match (expr) {
            Expr::Literal(val) => *val,
            Expr::Add(left, right) => eval(left) + eval(right),
            Expr::Mul(left, right) => eval(left) * eval(right),
            Expr::Neg(inner) => {
                let val = eval(inner);
                // Saturating negate for unsigned
                0
            },
        }
    }

    /// Build expression: (2 + 3) * 4
    fun example(): u64 {
        let expr = Expr::Mul(
            Box::new(Expr::Add(
                Box::new(Expr::Literal(2)),
                Box::new(Expr::Literal(3)),
            )),
            Box::new(Expr::Literal(4)),
        );
        eval(&expr) // Returns 20
    }
}
```

### 3. Resource Versioning with Migration

Enums are ideal for on-chain data that evolves across upgrades:

```move
module my_addr::versioned_config {
    use std::signer;

    enum Config has key {
        V1 { admin: address, fee_bps: u64 },
        V2 { admin: address, fee_bps: u64, paused: bool },
    }

    const E_NOT_ADMIN: u64 = 1;
    const E_ALREADY_V2: u64 = 2;

    /// Read admin from any version
    public fun get_admin(config: &Config): address {
        match (config) {
            Config::V1 { admin, .. } => *admin,
            Config::V2 { admin, .. } => *admin,
        }
    }

    /// Check paused status (V1 defaults to false)
    public fun is_paused(config: &Config): bool {
        match (config) {
            Config::V1 { .. } => false,
            Config::V2 { paused, .. } => *paused,
        }
    }

    /// Migrate V1 -> V2 (admin-only)
    public entry fun migrate_to_v2(admin: &signer) acquires Config {
        let addr = signer::address_of(admin);
        let config = borrow_global_mut<Config>(addr);

        match (config) {
            Config::V1 { admin: config_admin, fee_bps } => {
                assert!(*config_admin == addr, E_NOT_ADMIN);
                *config = Config::V2 {
                    admin: *config_admin,
                    fee_bps: *fee_bps,
                    paused: false,
                };
            },
            Config::V2 { .. } => abort E_ALREADY_V2,
        };
    }
}
```

### 4. Enum vs Phantom Types

Choose the right pattern for your use case:

| Criteria                     | Enum                                | Phantom Type                         |
| ---------------------------- | ----------------------------------- | ------------------------------------ |
| Different data per state     | Natural fit                         | Requires separate structs per state  |
| Compile-time state guarantee | No (runtime matching)               | Yes (type system enforced)           |
| State stored on-chain        | Single resource, enum field         | Separate resource per state          |
| State transitions            | Mutate enum variant in place        | Destroy old struct, create new       |
| Upgrade-friendly             | Add variants at the end             | Add new phantom type structs         |
| Best for                     | Data versioning, multi-state values | Permission systems, factory patterns |

**Rule of thumb:** Use enums when states carry different data or you need upgrade-safe versioning. Use phantom types
when you want the compiler to enforce state constraints at the type level.

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

With positional structs (Move 2.0+), the newtype pattern is more concise:

```move
module my_addr::newtype {
    /// Positional struct newtypes
    struct UserId(u64) has copy, drop, store;
    struct OrderId(u64) has copy, drop, store;

    /// Can't mix up UserId and OrderId
    public fun get_order(user: UserId, order: OrderId): Order {
        // Type system prevents passing OrderId as UserId
    }

    /// Named field newtypes (pre-Move 2.0 style, still valid)
    struct Amount has copy, drop, store {
        value: u64,
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

1. **Use enums for data versioning and state machines with variant-specific data**
2. **Use phantom types for compile-time state guarantees**
3. **Use positional structs for concise newtype wrappers**
4. **Leverage type witnesses for one-time operations**
5. **Apply appropriate abilities based on use case**
6. **Use newtype pattern for domain modeling**
7. **Prefer generic solutions for reusability**
8. **Document type constraints clearly**
9. **Test type safety with negative tests**

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
