# Error Catalog

Complete database of Aptos Move errors.

## Compilation Errors

### "Expected ';'"

**Fix:** Add semicolon at end of statement.

### "Unbound variable"

**Fix:** Declare variable before use or fix typo.

### "Ability constraint not satisfied"

**Fix:** Add required ability (key, drop, copy, store) to struct.

## Linker Errors

### "Package dependencies not resolved"

**Fix:** Add dependency to Move.toml `[dependencies]` section.

### "Named address not found"

**Fix:** Add address to Move.toml `[addresses]` section.

## Runtime Errors

### "RESOURCE_ALREADY_EXISTS"

**Fix:** Check if resource exists with `exists<T>()` before `move_to()`.

### "RESOURCE_NOT_FOUND"

**Fix:** Add `acquires T` to function signature or check exists before borrowing.

### "ARITHMETIC_ERROR"

**Fix:** Check for overflow/underflow before arithmetic operations.

## Test Errors

### "Expected failure but test passed"

**Fix:** Ensure test code actually aborts with correct error code.

### "Coverage below 100%"

**Fix:** Write tests for uncovered code paths. Use `aptos move coverage source`.

## Type Errors

### "Type mismatch"

**Fix:** Convert types correctly (e.g., address to Object<T> with `object::address_to_object<T>()`).

### "Generic type parameter not inferred"

**Fix:** Add explicit type parameter (e.g., `vector::empty<u64>()`).

### "Phantom type parameter not marked as phantom"

**Fix:** Add `phantom` keyword for generic types not stored in fields.

## Deployment Errors

### "Insufficient APT balance"

**Fix:** Fund account with `aptos account fund-with-faucet`.

### "Module bytecode verification failed"

**Fix:** Ensure code compiles, no unsafe patterns, proper ability constraints.

### "Upgrade compatibility check failed"

**Fix:** Don't remove public functions or change signatures. Add new functions instead.

## See Also

- Main SKILL.md for Object-related errors (most common)
- `error-codes.md` for framework error codes
- `debugging-guide.md` for debugging strategies
