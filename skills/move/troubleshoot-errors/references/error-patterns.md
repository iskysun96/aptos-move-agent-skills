# Error Patterns and Anti-patterns

Common error patterns and how to fix them.

## Pattern 1: Missing acquires Clause

**Problem:**

```move
// ❌ WRONG: Missing 'acquires'
public fun get_balance(addr: address): u64 {
    let account = borrow_global<Account>(addr);
    account.balance
}
```

**Solution:**

```move
// ✅ CORRECT: Add 'acquires Account'
public fun get_balance(addr: address): u64 acquires Account {
    let account = borrow_global<Account>(addr);
    account.balance
}
```

## Pattern 2: Object Address Confusion

**Problem:**

```move
// ❌ WRONG: Passing Object<T> where address expected
let item = borrow_global<Item>(item_obj);
```

**Solution:**

```move
// ✅ CORRECT: Convert Object to address first
let item_addr = object::object_address(&item_obj);
let item = borrow_global<Item>(item_addr);
```

## Pattern 3: Incorrect Error Codes

**Problem:**

```move
// ❌ WRONG: Using 0 - unclear what failed
assert!(condition, 0);
```

**Solution:**

```move
// ✅ CORRECT: Use descriptive error constant
const E_CONDITION_FAILED: u64 = 1;
assert!(condition, E_CONDITION_FAILED);
```

## Pattern 4: Resource Already Exists

**Problem:**

```move
// ❌ WRONG: Not checking if resource exists
move_to(account, Counter { value: 0 });
```

**Solution:**

```move
// ✅ CORRECT: Check before creating
if (!exists<Counter>(signer::address_of(account))) {
    move_to(account, Counter { value: 0 });
}
```

## Pattern 5: Named Object Seed Mismatch

**Problem:**

```move
// Created with one seed
let obj = object::create_named_object(creator, b"SEED_V1");

// ❌ WRONG: Accessing with different seed
let obj_addr = object::create_object_address(&creator_addr, b"SEED_V2");
```

**Solution:**

```move
// ✅ CORRECT: Use consistent seeds
const SEED: vector<u8> = b"MY_OBJECT_SEED";

// Creation
let obj = object::create_named_object(creator, SEED);

// Access
let obj_addr = object::create_object_address(&creator_addr, SEED);
```

## See Also

- Main SKILL.md for Object-related error patterns
- `error-catalog.md` for specific error solutions
- `debugging-guide.md` for debugging strategies
