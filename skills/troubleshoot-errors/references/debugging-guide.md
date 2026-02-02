# Debugging Guide

Advanced debugging techniques for Aptos Move contracts.

## Strategy 1: Add Debug Prints

Use `std::debug::print()` to output values during execution:

```move
use std::debug;

public fun my_function(x: u64, y: u64) {
    debug::print(&x);
    debug::print(&y);
    let result = x + y;
    debug::print(&result);
}
```

## Strategy 2: Simplify Code

Break complex expressions into simple steps:

```move
// Complex (hard to debug)
let result = complex_calc(func1(x), func2(y), func3(z));

// Simplified (easier to debug)
let a = func1(x);
debug::print(&a);

let b = func2(y);
debug::print(&b);

let c = func3(z);
debug::print(&c);

let result = complex_calc(a, b, c);
debug::print(&result);
```

## Strategy 3: Test Incrementally

Write separate tests for each step of complex logic:

```move
#[test]
public fun test_step1() {
    // Test first part
}

#[test]
public fun test_step2() {
    // Test second part
}

#[test]
public fun test_full_flow() {
    // Test everything together
}
```

## Strategy 4: Check Error Codes

When you see an abort like `ABORTED: 0x1`, find the constant with that value:

```move
// Error code 1 = E_NOT_OWNER
const E_NOT_OWNER: u64 = 1;
```

## See Also

- Main SKILL.md for common debugging patterns
- `error-catalog.md` for specific error solutions
