# Aptos Move V2 Modern Syntax Guide

**Purpose:** Guide to modern Move V2 syntax features including enum types, positional structs, signed integers, inline
functions, lambdas, and current best practices.

**Target:** AI assistants generating Move V2 smart contracts

---

## Overview

Move V2 introduces modern syntax features that make code more concise, expressive, and safe:

- **Enum types (Move 2.0+)**: Variant types with pattern matching for state machines and data versioning
- **Positional structs (Move 2.0+)**: Wrapper types without named fields (`struct Amount(u64)`)
- **Signed integers (Move 2.3+)**: Native `i8` through `i256` types
- **Inline functions with lambdas**: Higher-order functions for iteration and control flow
- **Receiver-style method calls (Move 2.0+)**: Dot notation `value.func(arg)`
- **Index notation (Move 2.0+)**: `vector[i]` instead of `vector::borrow`
- **Compound assignments (Move 2.1+)**: `x += 1`, `x -= 1`
- **Modern object model**: Type-safe `Object<T>` instead of raw addresses
- **Improved error handling**: Clear error constants and abort codes

---

## Enum Types (Move 2.0+)

Enum types define different variants of data layout in a single type. They support three field styles: no fields,
positional fields, and named fields.

### Basic Enum Declaration

```move
/// Fieldless variants
enum Color has copy, drop { Red, Blue, Green }

/// Named fields
enum Shape has copy, drop {
    Circle { radius: u64 },
    Rectangle { width: u64, height: u64 },
}

/// Positional fields
enum Result<T> has copy, drop, store {
    Err(u64),
    Ok(T),
}
```

### Creating Enum Values

```move
let c = Color::Red;
let s = Shape::Circle { radius: 10 };
let r: Result<u64> = Result::Ok(42);
```

### Pattern Matching with `match`

The `match` expression destructures enums. The compiler enforces **exhaustive** matching -- all variants must be handled
or a wildcard `_` must be used.

#### Basic Match

```move
fun area(shape: &Shape): u64 {
    match (shape) {
        Shape::Circle { radius } => *radius * *radius * 314 / 100,
        Shape::Rectangle { width, height } => *width * *height,
    }
}
```

#### Mutable Reference Matching

```move
fun double_radius(shape: &mut Shape) {
    match (shape) {
        Shape::Circle { radius } => *radius = *radius * 2,
        _ => (),
    }
}
```

#### Nested Patterns with Guards

```move
fun describe(shape: &Shape): String {
    match (shape) {
        Shape::Circle { radius } if *radius == 0 => string::utf8(b"point"),
        Shape::Circle { radius } if *radius > 100 => string::utf8(b"large circle"),
        Shape::Circle { .. } => string::utf8(b"circle"),
        Shape::Rectangle { width, height } if width == height => string::utf8(b"square"),
        Shape::Rectangle { .. } => string::utf8(b"rectangle"),
    }
}
```

#### Name Resolution in Match Arms

Inside a match arm, variant field names are automatically in scope. Use `..` to ignore remaining fields:

```move
match (config) {
    Config::V1 { admin, .. } => *admin,
    Config::V2 { admin, .. } => *admin,
}
```

### Testing Variants with `is`

The `is` operator tests which variant a value holds without destructuring. Use `|` to test multiple variants:

```move
fun is_success<T>(result: &Result<T>): bool {
    result is Result::Ok
}

fun is_terminal(state: &OrderState): bool {
    state is OrderState::Completed | OrderState::Cancelled
}
```

### Direct Field Selection

You can access fields directly on an enum value using dot notation. This aborts at runtime if the value holds a
different variant.

```move
let s = Shape::Circle { radius: 10 };
let r = s.radius;  // OK: returns 10
// let w = s.width; // ABORT: Circle doesn't have 'width'
```

**Warning:** Prefer `match` over direct field selection in production code to avoid runtime aborts.

### Let Binding with Enums

You can use `let` to bind fields from a specific variant. The binding does **not** check exhaustiveness, so it aborts at
runtime if the variant doesn't match:

```move
let Shape::Circle { radius } = shape;
// Aborts if shape is not a Circle
```

### Destroying Enum Values

Enums without `drop` must be explicitly destructured to consume them:

```move
enum Token {
    Fungible { amount: u64 },
    NonFungible { id: u64 },
}

fun consume_token(token: Token) {
    match (token) {
        Token::Fungible { amount: _ } => (),
        Token::NonFungible { id: _ } => (),
    }
}
```

### Recursive Enums

Enums can be recursive using `Box` for heap allocation:

```move
enum Expr has drop {
    Literal(u64),
    Add(Box<Expr>, Box<Expr>),
    Mul(Box<Expr>, Box<Expr>),
}

fun eval(expr: &Expr): u64 {
    match (expr) {
        Expr::Literal(value) => *value,
        Expr::Add(left, right) => eval(left) + eval(right),
        Expr::Mul(left, right) => eval(left) * eval(right),
    }
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

fun is_paused(config: &Config): bool {
    match (config) {
        Config::V1 { .. } => false,
        Config::V2 { paused, .. } => *paused,
    }
}
```

### Upgrade Compatibility Rules

When upgrading a module that declares an enum:

- **Original variants must appear first** in identical order
- **New variants can only be added at the end**
- **Existing abilities cannot be removed**
- **Runtime handling of missing variants:** If code adds a variant in an upgrade, existing `match` expressions from
  older code will not cover the new variant. Use a wildcard `_` arm to safely handle unknown future variants.

---

## Positional Structs (Move 2.0+)

Positional structs use unnamed fields accessed by position, ideal for newtype wrappers:

```move
/// Newtype wrappers for type safety
struct UserId(u64) has copy, drop, store;
struct OrderId(u64) has copy, drop, store;
struct Amount(u64) has copy, drop, store;

/// Can't accidentally mix up IDs
fun get_order(user: UserId, order: OrderId): Order {
    // Type system prevents passing OrderId as UserId
}
```

### Positional vs Named Fields

| Feature       | Positional (`struct Foo(u64)`)  | Named (`struct Foo { val: u64 }`) |
| ------------- | ------------------------------- | --------------------------------- |
| Access style  | `foo.0`                         | `foo.val`                         |
| Readability   | Brief, for simple wrappers      | Clear, for multi-field structs    |
| Destructuring | `let Foo(x) = foo`              | `let Foo { val: x } = foo`        |
| Best for      | Newtypes, single-field wrappers | Domain objects, configs           |

Positional fields also work in enum variants:

```move
enum Result<T> has copy, drop {
    Ok(T),
    Err(u64),
}
```

---

## Signed Integer Types (Move 2.3+)

Move 2.3 introduces native signed integer types:

| Type   | Size    | Range                           |
| ------ | ------- | ------------------------------- |
| `i8`   | 8-bit   | -128 to 127                     |
| `i16`  | 16-bit  | -32,768 to 32,767               |
| `i32`  | 32-bit  | -2,147,483,648 to 2,147,483,647 |
| `i64`  | 64-bit  | -9.2e18 to 9.2e18               |
| `i128` | 128-bit | -1.7e38 to 1.7e38               |
| `i256` | 256-bit | -5.8e76 to 5.8e76               |

### Builtin Constants

```move
let max = MAX_U64; // 18446744073709551615
let min = MIN_I32; // -2147483648
let max_signed = MAX_I64;
```

### Use Cases

```move
/// Price deltas that can be negative
struct PriceDelta has copy, drop, store {
    delta_bps: i64,
}

/// Accounting with credits and debits
fun apply_adjustment(balance: u64, adjustment: i64): u64 {
    if (adjustment >= 0) {
        balance + (adjustment as u64)
    } else {
        let abs = (0 - adjustment) as u64;
        assert!(balance >= abs, E_INSUFFICIENT_BALANCE);
        balance - abs
    }
}
```

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
// MODERN: Type-safe object references
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

// OLD: Raw addresses (avoid)
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

// V2: Use index notation for reading
public fun get_item(registry: &Registry, index: u64): u64 {
    *&registry.items[index]  // Clean and readable
}

// V2: Use index notation for writing
public fun update_item(registry: &mut Registry, index: u64, value: u64) {
    *&mut registry.items[index] = value;  // Clean mutation
}

// V2: Iterate with index notation
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
// OLD: vector::borrow syntax
public fun get_item_old(registry: &Registry, index: u64): u64 {
    *vector::borrow(&registry.items, index)  // More verbose
}

// OLD: vector::borrow_mut syntax
public fun update_item_old(registry: &mut Registry, index: u64, value: u64) {
    *vector::borrow_mut(&mut registry.items, index) = value;
}

// OLD: Iteration with borrow
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

    // MODERN: Use 'self' as first parameter
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

    // MODERN: Call with dot notation
    public entry fun update_item_if_owner(
        user: &signer,
        item: Object<Item>,
        new_value: u64
    ) {
        // Receiver-style call - reads much more naturally
        assert!(item.is_owner(user), E_NOT_OWNER);

        // Can chain multiple calls
        let current = item.get_value();
        assert!(new_value > current, E_VALUE_TOO_LOW);

        item.set_value(new_value);
    }

    // OLD: Traditional function call style
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

    // Define receiver-style functions with 'self'
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

    // Use receiver style in other functions
    public entry fun cancel_listing(
        seller: &signer,
        listing: Object<Listing>
    ) acquires Listing {
        assert!(listing.is_active(), E_ALREADY_INACTIVE);
        assert!(listing.is_seller(seller), E_NOT_SELLER);

        listing.deactivate();
    }

    public entry fun purchase_listing(
        buyer: &signer,
        listing: Object<Listing>
    ) acquires Listing {
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

## Additional Move 2 Syntax Features

### Compound Assignments (Move 2.1+)

```move
x += 1;
x -= amount;
x *= 2;
x /= divisor;
x %= modulus;
```

### Loop Labels (Move 2.1+)

```move
'outer: loop {
    let i = 0;
    while (i < len) {
        if (found) break 'outer;
        i += 1;
    };
};
```

### Dot-Dot Patterns (Move 2.1+)

Ignore remaining fields in struct destructuring:

```move
let Config::V2 { admin, .. } = config;
```

### Package Visibility (Move 2.1+)

```move
public(package) fun internal_helper() {
    // Visible to other modules in same package, but not to external packages
}
```

---

## Best Practices Summary

**DO:**

- Use enum types for data versioning and state machines
- Use positional structs for newtype wrappers (`struct Amount(u64)`)
- Use signed integers for deltas, offsets, and accounting
- Use `match` expressions for exhaustive enum handling
- Add wildcard `_` arms in `match` for future upgrade compatibility
- Use inline functions for iteration logic
- Use lambdas for concise operation definitions
- Use `Object<T>` for type-safe object references
- Define clear error constants with descriptive names
- Group related functionality in modules
- Use phantom types for type witnesses
- Use Option<T> for optional values
- Use `init_module(deployer: &signer)` for contract initialization
- Emit events with `#[event]` attribute and `event::emit()`
- Use receiver-style method calls with `self` parameter
- Use vector index notation `vector[i]` instead of `vector::borrow`
- Use direct named addresses `@addr` instead of helper functions

**DON'T:**

- Use raw addresses instead of `Object<T>`
- Use magic numbers for errors (use named constants)
- Ignore ability constraints on generics
- Mix old and new patterns in same codebase
- Create helper functions that just return named addresses
- Skip event emission for significant activities
- Use old syntax (`vector::borrow`) when V2 syntax (`vector[i]`) is available
- Skip `init_module` when contracts need initialization
- Use direct field selection on enums in production code (prefer `match`)
- Use structs for state machines when enums are a better fit
- Reorder enum variants during upgrades (breaks compatibility)

---

## Additional Resources

**Official Documentation:**

- Move Book: https://aptos.dev/build/smart-contracts/book
- Move 2 Release Notes: https://aptos.dev/build/smart-contracts/book/move-2
- Enums: https://aptos.dev/build/smart-contracts/book/enums
- Functions: https://aptos.dev/build/smart-contracts/book/functions
- Object Model: https://aptos.dev/build/smart-contracts/object
- Generics: https://aptos.dev/build/smart-contracts/book/generics

**Related Patterns:**

- `OBJECTS.md` - Detailed object patterns
- `SECURITY.md` - Security with modern syntax
- `TESTING.md` - Testing modern code
- `ADVANCED_TYPES.md` - Advanced enum patterns, phantom types, generics

---

**Remember:** Use modern Move V2 syntax for cleaner, safer, more maintainable code. Embrace enum types, inline
functions, lambdas, and type-safe objects.
