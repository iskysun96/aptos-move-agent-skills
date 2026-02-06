# Aptos Move V2 Modern Syntax Guide

**Purpose:** Guide to modern Move V2 syntax features including function values, enums, signed integers, inline
functions, lambdas, and current best practices.

**Target:** AI assistants generating Move V2 smart contracts

---

## Overview

Move V2 introduces modern syntax features that make code more concise, expressive, and safe:

- **Function values (Move 2.2+)**: First-class functions stored on-chain and passed across modules
- **Enum types (Move 2.0+)**: Variant types with pattern matching for state machines and data versioning
- **Signed integers (Move 2.3+)**: Native `i8` through `i256` types
- **Positional structs (Move 2.0+)**: Wrapper types without named fields (`struct Amount(u64)`)
- **Inline functions with lambdas**: Higher-order functions for iteration and control flow
- **Receiver-style method calls (Move 2.0+)**: Dot notation `value.func(arg)`
- **Index notation (Move 2.0+)**: `vector[i]` instead of `vector::borrow`
- **Compound assignments (Move 2.1+)**: `x += 1`, `x -= 1`
- **Modern object model**: Type-safe `Object<T>` instead of raw addresses
- **Improved error handling**: Clear error constants and abort codes

---

## Function Values (Move 2.2+)

Function values are first-class citizens in Move. Unlike inline functions, function values can cross module boundaries,
be stored on-chain, and enable dynamic dispatch.

### Function Types

```move
let f: |u64|bool;                   // Takes u64, returns bool
let g: |u64, bool|(bool, u64);     // Two params, returns tuple
let h: ||;                          // No params, no return
```

### Abilities on Function Types

```move
let f: |u64|bool has copy + drop;           // Can be used multiple times
let g: |u64|bool has copy + drop + store;   // Can be stored on-chain
```

Without abilities, a function value can only be moved and evaluated once.

### Creating Function Values

Public and entry functions are persistent by default and gain `store`:

```move
public fun is_even(x: u64): bool { x % 2 == 0 }

fun example() {
    let f: |u64|bool has copy + drop + store = is_even;
    assert!(f(4) == true, 0);
}
```

Non-public functions need `#[persistent]` to become storable:

```move
#[persistent]
fun is_odd(x: u64): bool { x % 2 == 1 }
```

Lambdas capture context variables by value:

```move
let threshold = 100;
let is_above = |x| x > threshold;
```

**Lambda restrictions:** Cannot capture references, parameters cannot be type-annotated, cannot contain `return`.

**Store ability restriction:** Lambdas that capture variables cannot have the `store` ability and cannot be stored
on-chain. Only named function references (e.g., `my_module::is_even`) or lambdas reducible to partial application of an
existing function can have `store`. Attempting to store a capturing lambda produces: _"lambda cannot be reduced to
partial application of existing function"_.

### Currying Pattern

```move
#[persistent]
fun add(x: u64, y: u64): u64 { x + y }

fun example() {
    let x = 22;
    let add_22: |u64|u64 has copy + drop + store = |y| add(x, y);
    assert!(add_22(8) == 30, 0);
}
```

### Storing Function Values On-Chain

```move
struct Plugin has key {
    handler: |u64|bool has copy + drop + store,
}

public fun register_plugin(account: &signer, handler: |u64|bool has copy + drop + store) {
    move_to(account, Plugin { handler });
}
```

### Reentrancy Safety

The VM detects module reentrancy when function values cause callbacks. Use `#[module_lock]` for strong protection:

```move
#[module_lock]
fun transfer(from: address, to: address, amount: u64, notify: |u64|) {
    account::deposit(to, amount);
    notify(amount);  // Reentrancy attempts blocked
    account::withdraw(from, amount);
}
```

See `SECURITY.md` Pattern 12 for full reentrancy safety guidance.

### Function Values vs Inline Functions

| Feature               | Inline Functions      | Function Values (2.2+)    |
| --------------------- | --------------------- | ------------------------- |
| Cross-module dispatch | No (same module only) | Yes                       |
| Stored on-chain       | No                    | Yes (with `store`)        |
| Runtime overhead      | None (inlined)        | Small (dynamic dispatch)  |
| Use case              | Iteration helpers     | Plugin systems, callbacks |

---

## Enum Types (Move 2.0+)

Enum types define different variants of data layout in a single type.

### Basic Enum Declaration

```move
enum Color has copy, drop { Red, Blue, Green }

enum Shape has copy, drop {
    Circle { radius: u64 },
    Rectangle { width: u64, height: u64 },
}

enum Result<T> has copy, drop, store {
    Err(u64),
    Ok(T),
}
```

### Creating and Matching

```move
let shape = Shape::Circle { radius: 10 };

fun area(shape: &Shape): u64 {
    match (shape) {
        Shape::Circle { radius } => *radius * *radius * 314 / 100,
        Shape::Rectangle { width, height } => *width * *height,
    }
}
```

The compiler enforces exhaustive matching. Use `_` for wildcard.

### Testing Variants

```move
fun is_success<T>(result: &Result<T>): bool {
    result is Result::Ok
}
```

### Data Versioning Pattern

Enums with `key` are ideal for upgradeable contract data:

```move
enum Config has key {
    V1 { admin: address, fee_bps: u64 },
    V2 { admin: address, fee_bps: u64, paused: bool },
}

fun get_admin(config: &Config): address {
    match (config) {
        Config::V1 { admin, .. } => *admin,
        Config::V2 { admin, .. } => *admin,
    }
}
```

### Testing Enums

Enum variants can only be constructed (packed) within their defining module. Test code in a separate `#[test_only]`
module cannot directly write `Shape::Circle { radius: 5 }`. Provide `#[test_only]` factory functions in the source
module:

```move
// In the source module
#[test_only]
public fun new_circle(radius: u64): Shape { Shape::Circle { radius } }

#[test_only]
public fun new_config_v1(admin: address, fee_bps: u64): Config {
    Config::V1 { admin, fee_bps }
}

// In the test module
#[test]
fun test_area() {
    let circle = my_module::new_circle(10);
    assert!(my_module::area(&circle) == 314, 0);
}
```

### Upgrade Compatibility

- Original variants must appear first in identical order
- New variants can only be added at the end
- Existing abilities cannot be removed

---

## Signed Integer Types (Move 2.3+)

Native signed integer types with full VM support:

| Type   | Size     | Range             |
| ------ | -------- | ----------------- |
| `i8`   | 1 byte   | -128 to 127       |
| `i16`  | 2 bytes  | -32,768 to 32,767 |
| `i32`  | 4 bytes  | -2^31 to 2^31-1   |
| `i64`  | 8 bytes  | -2^63 to 2^63-1   |
| `i128` | 16 bytes | -2^127 to 2^127-1 |
| `i256` | 32 bytes | -2^255 to 2^255-1 |

Builtin constants: `MAX_U64`, `MIN_I32`, `MAX_I64`, etc.

Use for price deltas, offsets, coordinates, and profit/loss calculations. Always prefer native signed integers over
custom libraries.

---

## Positional Structs (Move 2.0+)

```move
struct Amount(u64) has copy, drop, store;

fun example() {
    let amount = Amount(100);
    let Amount(value) = amount;
    assert!(value == 100, 0);
}
```

Use cases: newtype pattern, function type wrappers, simple data containers.

---

## Additional Move 2 Syntax

### Compound Assignments (Move 2.1+)

```move
x += 5; x -= 2; x *= 3;
```

### Loop Labels (Move 2.1+)

```move
'outer: loop {
    loop { if (done) break 'outer; }
}
```

### Package Visibility (Move 2.0+)

```move
package fun internal_helper(): u64 { 42 }
```

### Optional Acquires (Move 2.2+)

The compiler can infer `acquires` annotations automatically.

---

## Inline Functions & Lambdas

### Basic Inline Function

**Inline functions** are inlined at call sites, eliminating function call overhead and enabling lambda parameters.

```move
/// Inline function that applies operation to each element
inline fun for_each<T>(v: &vector<T>, f: |&T|) {
    let i = 0;
    let len = vector::length(v);
    while (i < len) {
        f(vector::borrow(v, i));
        i = i + 1;
    }
}

/// Usage with lambda
public fun print_all(numbers: &vector<u64>) {
    for_each(numbers, |x| {
        debug::print(x);
    });
}
```

### Lambda Syntax Variations

```move
// Single expression (no braces needed)
for_each(&numbers, |x| debug::print(x));

// Multiple statements (use braces)
for_each(&numbers, |x| {
    let doubled = *x * 2;
    debug::print(&doubled);
});

// Multiple parameters
inline fun for_each_indexed<T>(v: &vector<T>, f: |u64, &T|) {
    let i = 0;
    while (i < vector::length(v)) {
        f(i, vector::borrow(v, i));
        i = i + 1;
    }
}

// Usage
for_each_indexed(&items, |index, item| {
    debug::print(&index);
    debug::print(item);
});
```

### Common Inline Patterns

#### Map Operation

```move
/// Map function - transform each element
inline fun map<T, U>(v: &vector<T>, f: |&T| U): vector<U> {
    let result = vector::empty<U>();
    let i = 0;
    while (i < vector::length(v)) {
        vector::push_back(&mut result, f(vector::borrow(v, i)));
        i = i + 1;
    };
    result
}

/// Usage: Double all numbers
public fun double_all(numbers: &vector<u64>): vector<u64> {
    map(numbers, |x| *x * 2)
}
```

#### Filter Operation

```move
/// Filter function - keep elements matching predicate
inline fun filter<T: copy + drop>(v: &vector<T>, pred: |&T| bool): vector<T> {
    let result = vector::empty<T>();
    let i = 0;
    while (i < vector::length(v)) {
        let elem = vector::borrow(v, i);
        if (pred(elem)) {
            vector::push_back(&mut result, *elem);
        };
        i = i + 1;
    };
    result
}

/// Usage: Keep only even numbers
public fun keep_even(numbers: &vector<u64>): vector<u64> {
    filter(numbers, |x| *x % 2 == 0)
}
```

#### Fold/Reduce Operation

```move
/// Fold function - reduce vector to single value
inline fun fold<T, Acc>(v: &vector<T>, init: Acc, f: |Acc, &T| Acc): Acc {
    let acc = init;
    let i = 0;
    while (i < vector::length(v)) {
        acc = f(acc, vector::borrow(v, i));
        i = i + 1;
    };
    acc
}

/// Usage: Sum all numbers
public fun sum(numbers: &vector<u64>): u64 {
    fold(numbers, 0, |acc, x| acc + *x)
}
```

#### Any/All Operations

```move
/// Check if any element matches predicate
inline fun any<T>(v: &vector<T>, pred: |&T| bool): bool {
    let i = 0;
    while (i < vector::length(v)) {
        if (pred(vector::borrow(v, i))) {
            return true
        };
        i = i + 1;
    };
    false
}

/// Check if all elements match predicate
inline fun all<T>(v: &vector<T>, pred: |&T| bool): bool {
    let i = 0;
    while (i < vector::length(v)) {
        if (!pred(vector::borrow(v, i))) {
            return false
        };
        i = i + 1;
    };
    true
}

/// Usage
public fun has_large_number(numbers: &vector<u64>): bool {
    any(numbers, |x| *x > 1000)
}

public fun all_positive(numbers: &vector<u64>): bool {
    all(numbers, |x| *x > 0)
}
```

### Inline Functions with Mutable References

```move
/// Modify each element in place
inline fun for_each_mut<T>(v: &mut vector<T>, f: |&mut T|) {
    let i = 0;
    let len = vector::length(v);
    while (i < len) {
        f(vector::borrow_mut(v, i));
        i = i + 1;
    }
}

/// Usage: Double all numbers in place
public fun double_all_mut(numbers: &mut vector<u64>) {
    for_each_mut(numbers, |x| {
        *x = *x * 2;
    });
}
```

---

## Modern Object Operations

### Object Creation Patterns

```move
use aptos_framework::object::{Self, Object, ConstructorRef};

/// Modern object creation
public fun create_item(creator: &signer, name: String): Object<Item> {
    // Create object with modern API
    let constructor_ref = object::create_object(signer::address_of(creator));

    // Generate refs in constructor
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);

    // Get object signer
    let object_signer = object::generate_signer(&constructor_ref);

    // Store data
    move_to(&object_signer, Item {
        name,
        transfer_ref,
        delete_ref,
    });

    // Return typed object
    object::object_from_constructor_ref<Item>(&constructor_ref)
}
```

### Type-Safe Object References

```move
// ✅ MODERN: Type-safe object references
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,  // Type-safe!
    recipient: address
) acquires Item {
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    object::transfer_with_ref(
        object::generate_linear_transfer_ref(&item_data.transfer_ref),
        recipient
    );
}

// ❌ OLD: Raw addresses (avoid)
public entry fun transfer_item_old(
    owner: &signer,
    item_addr: address,  // No type safety
    recipient: address
) {
    // Manual type checking required
}
```

---

## Error Handling Patterns

### Clear Error Constants

```move
/// Modern error codes - descriptive constants
module my_addr::marketplace {
    // Group related errors
    const E_NOT_OWNER: u64 = 1;
    const E_NOT_ADMIN: u64 = 2;
    const E_UNAUTHORIZED: u64 = 3;

    const E_ZERO_AMOUNT: u64 = 10;
    const E_AMOUNT_TOO_HIGH: u64 = 11;
    const E_INSUFFICIENT_BALANCE: u64 = 12;

    const E_EMPTY_NAME: u64 = 20;
    const E_NAME_TOO_LONG: u64 = 21;

    const E_MARKETPLACE_PAUSED: u64 = 30;
    const E_ITEM_NOT_LISTED: u64 = 31;
    const E_ITEM_ALREADY_SOLD: u64 = 32;

    /// Use clear error messages
    public entry fun purchase_item(
        buyer: &signer,
        item: Object<Item>
    ) acquires Marketplace, Item {
        let marketplace = borrow_global<Marketplace>(@my_addr);

        // Clear error codes
        assert!(!marketplace.paused, E_MARKETPLACE_PAUSED);
        assert!(is_listed(item), E_ITEM_NOT_LISTED);
        assert!(!is_sold(item), E_ITEM_ALREADY_SOLD);

        // ... purchase logic
    }
}
```

---

## String Operations

### Modern String Handling

```move
use std::string::{Self, String};

/// Create strings
public fun create_strings() {
    // From bytes
    let s1 = string::utf8(b"Hello");

    // Append strings
    let s2 = string::utf8(b" World");
    string::append(&mut s1, s2);
    // s1 is now "Hello World"

    // Get length
    let len = string::length(&s1);

    // Compare strings
    assert!(s1 == string::utf8(b"Hello World"), 0);
}

/// Validate string
public fun validate_name(name: &String): bool {
    let len = string::length(name);
    len > 0 && len <= 32
}
```

---

## Vector Operations

### Modern Vector Patterns

```move
use std::vector;

/// Comprehensive vector operations
public fun vector_examples() {
    // Create empty vector
    let v = vector::empty<u64>();

    // Push elements
    vector::push_back(&mut v, 10);
    vector::push_back(&mut v, 20);
    vector::push_back(&mut v, 30);

    // Access elements
    let first = *vector::borrow(&v, 0);
    assert!(first == 10, 0);

    // Modify elements
    let elem = vector::borrow_mut(&mut v, 1);
    *elem = 25;

    // Pop element
    let last = vector::pop_back(&mut v);
    assert!(last == 30, 1);

    // Length
    let len = vector::length(&v);
    assert!(len == 2, 2);

    // Check contains
    assert!(vector::contains(&v, &10), 3);

    // Find index
    let (found, index) = vector::index_of(&v, &25);
    assert!(found && index == 1, 4);

    // Remove element
    let removed = vector::remove(&mut v, 0);
    assert!(removed == 10, 5);
}
```

### Vector with Lambdas

```move
/// Modern iteration patterns
public fun process_vector(numbers: &mut vector<u64>) {
    // Filter with lambda
    let evens = filter(numbers, |x| *x % 2 == 0);

    // Transform with lambda
    let doubled = map(&evens, |x| *x * 2);

    // Sum with lambda
    let total = fold(&doubled, 0, |acc, x| acc + *x);

    debug::print(&total);
}
```

### Vector Index Notation (Move 2)

Move 2 introduces cleaner syntax for vector access using index notation instead of `vector::borrow`.

**MODERN (V2 Syntax):**

```move
use std::vector;

struct Registry has key {
    items: vector<u64>,
}

// ✅ V2: Use index notation for reading
public fun get_item(registry: &Registry, index: u64): u64 {
    *&registry.items[index]  // Clean and readable
}

// ✅ V2: Use index notation for writing
public fun update_item(registry: &mut Registry, index: u64, value: u64) {
    *&mut registry.items[index] = value;  // Clean mutation
}

// ✅ V2: Iterate with index notation
public fun sum_all(registry: &Registry): u64 {
    let sum = 0;
    let i = 0;
    let len = vector::length(&registry.items);

    while (i < len) {
        sum = sum + registry.items[i];  // Much cleaner!
        i = i + 1;
    };

    sum
}
```

**OLD (Pre-V2 Syntax):**

```move
// ❌ OLD: vector::borrow syntax
public fun get_item_old(registry: &Registry, index: u64): u64 {
    *vector::borrow(&registry.items, index)  // More verbose
}

// ❌ OLD: vector::borrow_mut syntax
public fun update_item_old(registry: &mut Registry, index: u64, value: u64) {
    *vector::borrow_mut(&mut registry.items, index) = value;
}

// ❌ OLD: Iteration with borrow
public fun sum_all_old(registry: &Registry): u64 {
    let sum = 0;
    let i = 0;
    let len = vector::length(&registry.items);

    while (i < len) {
        sum = sum + *vector::borrow(&registry.items, i);  // Verbose
        i = i + 1;
    };

    sum
}
```

---

## Receiver-Style Method Calls (Move 2)

Move 2 introduced receiver-style function calls that allow using dot notation `value.func(arg)` instead of
`func(&value, arg)`.

### Defining Receiver Functions

Use `self` as the first parameter name to enable dot notation:

```move
module my_addr::items {
    use std::signer;
    use aptos_framework::object::{Self, Object};

    struct Item has key {
        owner: address,
        value: u64,
        name: String,
    }

    // ✅ MODERN: Use 'self' as first parameter
    public fun is_owner(self: &Object<Item>, user: &signer): bool acquires Item {
        let item_data = borrow_global<Item>(object::object_address(self));
        item_data.owner == signer::address_of(user)
    }

    public fun get_value(self: &Object<Item>): u64 acquires Item {
        let item_data = borrow_global<Item>(object::object_address(self));
        item_data.value
    }

    public fun get_name(self: &Object<Item>): String acquires Item {
        let item_data = borrow_global<Item>(object::object_address(self));
        item_data.name
    }

    // Mutable receiver
    public fun set_value(self: &Object<Item>, new_value: u64) acquires Item {
        let item_data = borrow_global_mut<Item>(object::object_address(self));
        item_data.value = new_value;
    }
}
```

### Using Receiver-Style Calls

```move
module my_addr::marketplace {
    use my_addr::items;

    // ✅ MODERN: Call with dot notation
    public entry fun update_item_if_owner(
        user: &signer,
        item: Object<Item>,
        new_value: u64
    ) {
        // Receiver-style call - reads much more naturally!
        assert!(item.is_owner(user), E_NOT_OWNER);

        // Can chain multiple calls
        let current = item.get_value();
        assert!(new_value > current, E_VALUE_TOO_LOW);

        item.set_value(new_value);
    }

    // ❌ OLD: Traditional function call style
    public entry fun update_item_old_style(
        user: &signer,
        item: Object<Item>,
        new_value: u64
    ) {
        // Less readable function call syntax
        assert!(items::is_owner(&item, user), E_NOT_OWNER);
        let current = items::get_value(&item);
        items::set_value(&item, new_value);
    }
}
```

### Automatic Discovery

The compiler automatically discovers receiver functions - no need to import them explicitly:

```move
// Receiver functions are discovered automatically based on type
public fun use_item(item: Object<Item>) {
    // Compiler finds is_owner, get_value, etc. automatically
    // No need to: use my_addr::items::is_owner;
    let value = item.get_value();  // Works!
}
```

### Benefits of Receiver Style

1. **More readable**: `item.is_owner(user)` reads like English
2. **Familiar syntax**: Similar to methods in other languages
3. **Chainable**: `item.get_value() * 2` is cleaner
4. **Auto-discovery**: Compiler finds functions automatically

### Complete Example

```move
module marketplace_addr::listings {
    use std::signer;
    use aptos_framework::object::{Self, Object};

    struct Listing has key {
        seller: address,
        price: u64,
        active: bool,
    }

    // ✅ Define receiver-style functions with 'self'
    public fun is_active(self: &Object<Listing>): bool acquires Listing {
        let listing = borrow_global<Listing>(object::object_address(self));
        listing.active
    }

    public fun get_price(self: &Object<Listing>): u64 acquires Listing {
        let listing = borrow_global<Listing>(object::object_address(self));
        listing.price
    }

    public fun is_seller(self: &Object<Listing>, user: &signer): bool acquires Listing {
        let listing = borrow_global<Listing>(object::object_address(self));
        listing.seller == signer::address_of(user)
    }

    public fun deactivate(self: &Object<Listing>) acquires Listing {
        let listing = borrow_global_mut<Listing>(object::object_address(self));
        listing.active = false;
    }

    // ✅ Use receiver style in other functions
    public entry fun cancel_listing(
        seller: &signer,
        listing: Object<Listing>
    ) acquires Listing {
        // Beautiful, readable code!
        assert!(listing.is_active(), E_ALREADY_INACTIVE);
        assert!(listing.is_seller(seller), E_NOT_SELLER);

        listing.deactivate();
    }

    public entry fun purchase_listing(
        buyer: &signer,
        listing: Object<Listing>
    ) acquires Listing {
        // Chain checks naturally
        assert!(listing.is_active(), E_NOT_ACTIVE);

        let price = listing.get_price();
        // ... payment logic ...

        listing.deactivate();
    }
}
```

---

## Option Type

### Working with Option<T>

```move
use std::option::{Self, Option};

/// Option patterns
public fun option_examples() {
    // Create Some
    let some_value = option::some(42);

    // Create None
    let none_value = option::none<u64>();

    // Check if some
    assert!(option::is_some(&some_value), 0);
    assert!(option::is_none(&none_value), 1);

    // Extract value (aborts if None)
    let value = option::extract(&mut some_value);
    assert!(value == 42, 2);

    // Borrow value (aborts if None)
    let some_value2 = option::some(100);
    let borrowed = option::borrow(&some_value2);
    assert!(*borrowed == 100, 3);
}

/// Safe option handling
public fun safe_divide(a: u64, b: u64): Option<u64> {
    if (b == 0) {
        option::none()
    } else {
        option::some(a / b)
    }
}

/// Using option in functions
public fun calculate_with_option(a: u64, b: u64): u64 {
    let result = safe_divide(a, b);

    if (option::is_some(&result)) {
        option::extract(&mut result)
    } else {
        0 // Default value
    }
}
```

---

## Abilities and Constraints

### Modern Ability Usage

```move
/// Phantom types for type safety
struct Vault<phantom CoinType> has key {
    balance: u64,
}

/// Copy + Drop for value types
struct Point has copy, drop {
    x: u64,
    y: u64,
}

/// Store for nested resources
struct Container has key, store {
    value: u64,
}

/// Key for global storage
struct GlobalConfig has key {
    admin: address,
    value: u64,
}

/// Generic with ability constraints
public fun process_copyable<T: copy + drop>(item: T) {
    let copy_of_item = item;
    // Both item and copy_of_item can be used
}
```

---

## Module Structure

### Modern Module Organization

```move
module my_addr::marketplace {
    // ============ Imports ============
    use std::signer;
    use std::string::String;
    use std::vector;
    use aptos_framework::object::{Self, Object};

    // ============ Structs ============
    struct Marketplace has key {
        items: vector<Object<Item>>,
        admin: address,
        paused: bool,
    }

    struct Item has key {
        name: String,
        price: u64,
        seller: address,
        transfer_ref: object::TransferRef,
    }

    // ============ Constants ============
    const E_NOT_ADMIN: u64 = 1;
    const E_NOT_OWNER: u64 = 2;
    const E_MARKETPLACE_PAUSED: u64 = 3;

    const MAX_ITEMS: u64 = 1000;

    // ============ Init Functions ============
    fun init_module(admin: &signer) {
        move_to(admin, Marketplace {
            items: vector::empty(),
            admin: signer::address_of(admin),
            paused: false,
        });
    }

    // ============ Public Entry Functions ============
    public entry fun list_item(
        seller: &signer,
        item: Object<Item>,
        price: u64
    ) acquires Marketplace {
        // ... implementation
    }

    // ============ Public Functions ============
    public fun get_item_price(item: Object<Item>): u64 acquires Item {
        let item_data = borrow_global<Item>(object::object_address(&item));
        item_data.price
    }

    // ============ Private Functions ============
    fun assert_not_paused() acquires Marketplace {
        let marketplace = borrow_global<Marketplace>(@my_addr);
        assert!(!marketplace.paused, E_MARKETPLACE_PAUSED);
    }

    // ============ Inline Helpers ============
    inline fun for_each_item(f: |&Object<Item>|) acquires Marketplace {
        let marketplace = borrow_global<Marketplace>(@my_addr);
        let i = 0;
        while (i < vector::length(&marketplace.items)) {
            f(vector::borrow(&marketplace.items, i));
            i = i + 1;
        }
    }

    // ============ Test Module ============
    #[test_only]
    module marketplace_tests {
        // ... tests
    }
}
```

---

## Best Practices Summary

**DO:**

- ✅ Use function values for cross-module callbacks and plugin systems (Move 2.2+)
- ✅ Use enum types for variant data and state machines (Move 2.0+)
- ✅ Use signed integers (`i8`-`i256`) for deltas and offsets (Move 2.3+)
- ✅ Use `#[module_lock]` when function value callbacks must not re-enter your module
- ✅ Use `match` expressions for exhaustive enum handling
- ✅ Use inline functions for iteration logic within a module
- ✅ Use lambdas for concise operation definitions
- ✅ Use `Object<T>` for type-safe object references
- ✅ Define clear error constants with descriptive names
- ✅ Group related functionality in modules
- ✅ Use phantom types for type witnesses
- ✅ Use Option<T> for optional values
- ✅ Use `init_module(deployer: &signer)` for contract initialization
- ✅ Emit events with `#[event]` attribute and `event::emit()`
- ✅ Use receiver-style method calls with `self` parameter
- ✅ Use vector index notation `vector[i]` instead of `vector::borrow`
- ✅ Use direct named addresses `@addr` instead of helper functions

**DON'T:**

- ❌ Use raw addresses instead of `Object<T>`
- ❌ Use magic numbers for errors (use named constants)
- ❌ Ignore ability constraints on generics
- ❌ Mix old and new patterns in same codebase
- ❌ Create helper functions that just return named addresses
- ❌ Skip event emission for significant activities
- ❌ Use old syntax (`vector::borrow`) when V2 syntax (`vector[i]`) is available
- ❌ Skip `init_module` when contracts need initialization
- ❌ Use custom signed integer libraries when native `i8`-`i256` types are available
- ❌ Store function values without considering reentrancy implications
- ❌ Reorder enum variants during upgrades (breaks compatibility)

---

## Additional Resources

**Official Documentation:**

- Move Book: https://aptos.dev/build/smart-contracts/book
- Move 2 Release Notes: https://aptos.dev/build/smart-contracts/book/move-2
- Functions (incl. function values): https://aptos.dev/build/smart-contracts/book/functions
- Enums: https://aptos.dev/build/smart-contracts/book/enums
- Object Model: https://aptos.dev/build/smart-contracts/object
- Generics: https://aptos.dev/build/smart-contracts/book/generics

**Related Patterns:**

- `OBJECTS.md` - Detailed object patterns
- `SECURITY.md` - Security with modern syntax (incl. reentrancy safety)
- `ADVANCED_TYPES.md` - Advanced type patterns including enums
- `TESTING.md` - Testing modern code

---

**Remember:** Use modern Move V2 syntax for cleaner, safer, more maintainable code. Embrace function values, enums,
inline functions, lambdas, and type-safe objects.
