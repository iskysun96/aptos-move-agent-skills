# Aptos Move V2 Agent Skills - AI Assistant Guide

**Version:** 1.0 **Target:** AI assistants (Claude Code, Cursor, GitHub Copilot, future Aptos Vibe tool) **Purpose:**
Build secure, modern Aptos Move V2 smart contracts using object-centric patterns

## Quick Start for AI Assistants

When helping developers with Aptos Move V2:

1. **Search first**: Check `aptos-core/move-examples` and official docs before writing code
2. **Token standards**:
   - Digital Assets for NFTs (Object<AptosToken>)
   - Fungible Assets for tokens/coins (Object<Metadata>)
3. **Use objects**: Always use `Object<T>` references, never raw addresses (unless legacy integration)
4. **Security first**: Verify signer authority, validate inputs, protect references
5. **Test everything**: Generate comprehensive tests with 100% coverage
6. **Modern syntax**: Use inline functions, lambdas, current object model

## Core Workflows

### Workflow 1: Create New Project

**Trigger:** User says "create move project", "new aptos project", "scaffold move app"

**Steps:**

1. Activate `scaffold-project` skill
2. Run `aptos move init --name <project>`
3. Configure Move.toml with proper dependencies
4. Create directory structure (sources/, tests/, scripts/)
5. Initialize git repository
6. Create basic README

**Output:** Ready-to-build Move project with proper structure

### Workflow 2: Write Smart Contract

**Trigger:** User says "write move contract", "create smart contract", "build module", "implement move code"

**Steps:**

1. Activate `search-aptos-examples` skill → find similar examples
2. Activate `write-contracts` skill → generate code following patterns
3. **Critical requirements:**
   - Use `Object<T>` for all object references
   - Implement proper constructor pattern (ConstructorRef → generate refs → move_to)
   - Add signer verification for all entry functions
   - Validate all inputs (amounts, lengths, types)
   - Use inline functions and lambdas for modern code
4. Reference `patterns/move/OBJECTS.md` for object patterns
5. Reference `patterns/move/SECURITY.md` for security checklist
6. Reference `patterns/move/MOVE_V2_SYNTAX.md` for syntax

**Output:** Secure, well-structured Move module

### Workflow 3: Generate Tests

**Trigger:** User says "write tests", "test this contract", "add test coverage", or AUTOMATICALLY after writing any
contract

**Steps:**

1. Activate `generate-tests` skill
2. Create test module in tests/ directory
3. **Required test categories:**
   - Happy path tests (#[test])
   - Access control tests (unauthorized signers)
   - Input validation tests (zero amounts, overflows, invalid data)
   - Edge cases (empty vectors, max limits, boundary conditions)
4. Use `#[expected_failure(abort_code = E_CODE)]` for error paths
5. Run `aptos move test --coverage` to verify 100% coverage
6. Reference `patterns/move/SECURITY.md` for patterns

**Output:** Comprehensive test suite with 100% line coverage

### Workflow 4: Security Audit

**Trigger:** User says "audit contract", "check security", "review move code", or AUTOMATICALLY before deployment

**Steps:**

1. Activate `security-audit` skill
2. Reference `patterns/move/SECURITY.md` checklist
3. **Verify each category:**
   - **Access Control:** All entry functions verify signer authority
   - **Input Validation:** All inputs checked (non-zero, no overflow, within limits)
   - **Object Safety:** ConstructorRef never returned, all refs generated in constructor
   - **Reference Safety:** No &mut exposed publicly, fields protected from mem::swap
   - **Testing:** 100% coverage including all error paths
4. Generate report with findings and fixes
5. Provide code changes for any issues found

**Output:** Security report with fixes applied

### Workflow 5: Deploy Contract

**Trigger:** User says "deploy contract", "publish module", "deploy to testnet/mainnet"

**Steps:**

1. Activate `security-audit` skill → ensure security checklist passes
2. Activate `generate-tests` skill → ensure 100% coverage
3. Run `aptos move compile` to verify compilation
4. Activate `deploy-contracts` skill
5. Choose network (devnet, testnet, mainnet)
6. Run `aptos move publish --named-addresses <name>=<address>`
7. Verify deployment with `aptos account list --account <address>`
8. Document deployed addresses

**Output:** Contract deployed with verification

### Workflow 6: Optimize Gas Usage

**Trigger:** User says "optimize gas", "reduce gas costs", "make contract cheaper", "gas efficiency"

**Steps:**

1. Activate `analyze-gas-optimization` skill
2. Analyze current contract for expensive operations
3. Identify optimization opportunities:
   - Storage packing
   - Batch operations
   - Efficient data structures (Tables vs Vectors)
   - Lazy evaluation patterns
4. Apply optimizations while maintaining functionality
5. Measure gas improvements
6. Run tests to ensure correctness

**Output:** Optimized contract with gas savings report

## Skill Activation Table

| Skill                      | Activates When                                                 | Priority | Auto-Active           |
| -------------------------- | -------------------------------------------------------------- | -------- | --------------------- |
| `write-contracts`          | "write move contract", "create smart contract", "build module" | CRITICAL | No                    |
| `generate-tests`           | "write tests", "add coverage", after any contract written      | CRITICAL | Yes (after contracts) |
| `security-audit`           | "audit", "check security", before deployment                   | CRITICAL | Yes (before deploy)   |
| `scaffold-project`         | "create project", "new move app", "scaffold"                   | High     | No                    |
| `search-aptos-examples`    | "find example", "search aptos", before writing contracts       | High     | Yes (before writing)  |
| `analyze-gas-optimization` | "optimize gas", "reduce gas costs", "gas efficiency"           | High     | No                    |
| `use-aptos-cli`            | "run aptos", "compile", "test", "deploy"                       | Medium   | No                    |
| `deploy-contracts`         | "deploy", "publish"                                            | Medium   | No                    |
| `troubleshoot-errors`      | Error messages detected, "fix error", "debug"                  | Medium   | Yes (on errors)       |

**Auto-activation rules:**

- Always search examples before writing new contracts
- Always generate tests after writing contracts
- Always run security audit before deployment
- Always activate troubleshooting when errors occur

## ALWAYS Rules (Non-Negotiable)

When working with Aptos Move V2, you MUST:

**Object Model:**

- ✅ Use `Object<T>` for object references (modern)
- ✅ Generate all refs (TransferRef, DeleteRef) in constructor BEFORE ConstructorRef is destroyed
- ✅ Use `object::owner(obj)` to verify ownership
- ✅ Use `object::generate_signer(&constructor_ref)` for object signers
- ✅ Use named objects for singletons: `object::create_named_object(creator, seed)`

**Security:**

- ✅ Verify signer authority: `assert!(signer::address_of(user) == expected, E_UNAUTHORIZED)`
- ✅ Check object ownership: `assert!(object::owner(obj) == signer::address_of(user), E_NOT_OWNER)`
- ✅ Validate all numeric inputs: `assert!(amount > 0 && amount <= MAX, E_INVALID_AMOUNT)`
- ✅ Use `phantom` for generic type safety: `struct Vault<phantom CoinType>`
- ✅ Protect critical fields from `mem::swap` attacks

**Testing:**

- ✅ Achieve 100% line coverage: `aptos move test --coverage`
- ✅ Test all error paths with `#[expected_failure(abort_code = E_CODE)]`
- ✅ Test access control with multiple signers
- ✅ Test input validation with invalid data

**Syntax:**

- ✅ Use inline functions with lambdas for iteration
- ✅ Use modern object functions: `object::address_to_object<T>(addr)`
- ✅ Use proper error constants: `const E_NOT_OWNER: u64 = 1;`

## NEVER Rules (Strictly Prohibited)

When working with Aptos Move V2, you MUST NEVER:

**Legacy Patterns:**

- ❌ NEVER use resource accounts (legacy pattern, use objects instead)
- ❌ NEVER use raw addresses for objects (use `Object<T>`)
- ❌ NEVER use `account::create_resource_account()` (replaced by named objects)

**Security Violations:**

- ❌ NEVER return ConstructorRef from functions (caller can destroy object)
- ❌ NEVER expose `&mut` in public functions (allows mem::swap attacks)
- ❌ NEVER skip signer verification in entry functions
- ❌ NEVER trust caller addresses without verification
- ❌ NEVER allow ungated transfers without good reason

**Bad Practices:**

- ❌ NEVER deploy without 100% test coverage
- ❌ NEVER skip input validation
- ❌ NEVER ignore security checklist
- ❌ NEVER copy code without understanding security implications
- ❌ NEVER use old examples without verifying they're V2-compatible

## Pattern Examples

### Object Creation (CORRECT)

```move
module my_addr::nft {
    use std::string::String;
    use aptos_framework::object::{Self, Object, ConstructorRef};
    use aptos_token_objects::token;

    struct MyNFT has key {
        name: String,
        // Store refs for later use
        transfer_ref: object::TransferRef,
        delete_ref: object::DeleteRef,
    }

    /// Create NFT using object pattern
    public fun create_nft(creator: &signer, name: String): Object<MyNFT> {
        // Create object
        let constructor_ref = object::create_object(signer::address_of(creator));

        // Generate signer for object
        let object_signer = object::generate_signer(&constructor_ref);

        // Generate all refs BEFORE constructor_ref is destroyed
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let delete_ref = object::generate_delete_ref(&constructor_ref);

        // Store data in object
        move_to(&object_signer, MyNFT {
            name,
            transfer_ref,
            delete_ref,
        });

        // Return object reference (constructor_ref is automatically destroyed)
        object::object_from_constructor_ref<MyNFT>(&constructor_ref)
    }

    /// Transfer with ownership verification
    public entry fun transfer_nft(
        owner: &signer,
        nft: Object<MyNFT>,
        recipient: address
    ) acquires MyNFT {
        // Verify ownership
        assert!(object::owner(nft) == signer::address_of(owner), E_NOT_OWNER);

        // Safe to transfer
        let nft_data = borrow_global<MyNFT>(object::object_address(&nft));
        object::transfer_with_ref(
            object::generate_linear_transfer_ref(&nft_data.transfer_ref),
            recipient
        );
    }
}
```

### Security Verification (CORRECT)

```move
/// Entry function with proper security
public entry fun update_item(
    owner: &signer,
    item: Object<Item>,
    new_name: String
) acquires Item {
    // 1. Verify signer authority
    let owner_addr = signer::address_of(owner);

    // 2. Verify object ownership
    assert!(object::owner(item) == owner_addr, E_NOT_OWNER);

    // 3. Validate inputs
    assert!(string::length(&new_name) > 0, E_INVALID_NAME);
    assert!(string::length(&new_name) <= MAX_NAME_LENGTH, E_NAME_TOO_LONG);

    // 4. Safe to proceed
    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = new_name;
}
```

### Modern Syntax with Inline Functions (CORRECT)

```move
use std::vector;

/// Process each element using inline function and lambda
inline fun for_each<T>(v: &vector<T>, f: |&T|) {
    let i = 0;
    let len = vector::length(v);
    while (i < len) {
        f(vector::borrow(v, i));
        i = i + 1;
    }
}

/// Usage
public fun sum_amounts(amounts: &vector<u64>): u64 {
    let total = 0;
    for_each(amounts, |amount| {
        total = total + *amount;
    });
    total
}
```

## Common Mistakes to Avoid

| Mistake                                  | Why It's Wrong                 | Correct Pattern                                                 |
| ---------------------------------------- | ------------------------------ | --------------------------------------------------------------- |
| `public fun create(): ConstructorRef`    | Caller can destroy object      | Return `Object<T>` instead                                      |
| `public fun update(item: &mut T)`        | Allows mem::swap attacks       | Use `acquires`, borrow internally                               |
| `entry fun transfer(item_addr: address)` | Legacy pattern, no type safety | Use `Object<Item>`                                              |
| `let obj = object::create(...); obj`     | ConstructorRef returned        | Store data, return `object::object_from_constructor_ref()`      |
| No signer verification                   | Anyone can call function       | `assert!(signer::address_of(user) == expected, E_UNAUTHORIZED)` |
| No input validation                      | Overflow, zero amounts, etc.   | `assert!(amount > 0 && amount <= MAX, E_INVALID)`               |
| Skipping tests                           | Bugs in production             | Write tests with 100% coverage                                  |
| Using resource accounts                  | Deprecated in V2               | Use named objects instead                                       |

## Error Handling

**Define clear error constants:**

```move
/// Error codes
const E_NOT_OWNER: u64 = 1;
const E_INVALID_AMOUNT: u64 = 2;
const E_INSUFFICIENT_BALANCE: u64 = 3;
const E_ALREADY_EXISTS: u64 = 4;
const E_NOT_FOUND: u64 = 5;
```

**Test all error paths:**

```move
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_unauthorized_transfer(owner: &signer, attacker: &signer) {
    let item = create_item(owner);
    transfer_item(attacker, item, @0x3); // Should abort
}
```

## Documentation References

**Official Aptos Docs:**

- Object Model: https://aptos.dev/build/smart-contracts/object
- Security Guidelines: https://aptos.dev/build/smart-contracts/move-security-guidelines
- Move Book: https://aptos.dev/build/smart-contracts/book
- CLI Reference: https://aptos.dev/build/cli
- Testing: https://aptos.dev/build/smart-contracts/book/unit-testing

**Example Repositories:**

- aptos-core/aptos-move/move-examples (53 official examples)
- Priority: hello_blockchain, mint_nft, token_objects, fungible_asset, dao, marketplace

**Pattern Documentation (Local):**

- `patterns/move/OBJECTS.md` - Comprehensive object model guide
- `patterns/move/SECURITY.md` - Security checklist and patterns
- `patterns/move/MOVE_V2_SYNTAX.md` - Modern syntax examples

## Troubleshooting Guide

**Common Error: "LINKER_ERROR"**

- **Cause:** Missing dependency in Move.toml
- **Fix:** Add required package (AptosFramework, AptosStdlib, AptosToken)

**Common Error: "ABORTED at 0x1"**

- **Cause:** Assertion failed, no error code provided
- **Fix:** Use `const E_CODE: u64 = N;` and `assert!(..., E_CODE)`

**Common Error: "Type mismatch"**

- **Cause:** Using `address` where `Object<T>` expected
- **Fix:** Update function signature to use `Object<T>`

**Common Error: "Coverage below 100%"**

- **Cause:** Missing tests for some code paths
- **Fix:** Run `aptos move coverage source --module <name>` to see uncovered lines

**Common Error: "Resource already exists"**

- **Cause:** Trying to `move_to` twice for same resource
- **Fix:** Check if resource exists with `exists<T>(addr)` first

## Integration Notes

**For Claude Code:**

- Skills activate automatically based on context
- Use `/commit` after significant changes
- Use `/review-pr` before submitting code
- This file is loaded when CLAUDE.md is detected

**For Cursor/Copilot:**

- Include this file in workspace context
- Reference skill files explicitly: `@skills/write-contracts/SKILL.md`
- Use inline comments to trigger patterns: `// PATTERN: object-creation`

**For Future Aptos Vibe Tool:**

- Skills designed for automatic activation
- Follows Aptos Labs coding standards
- Compatible with Aptos Developer Console workflows

## Version History

**v1.0 (2026-01-23):**

- Initial release
- 9 core skills implemented
- Object-centric patterns enforced
- Security-first approach
- 100% test coverage requirement

---

**Next Steps for AI Assistant:**

When user starts a task:

1. Identify which workflow applies (create/write/test/audit/deploy)
2. Activate relevant skills in order
3. Follow ALWAYS rules strictly
4. Avoid NEVER rules completely
5. Reference pattern files for detailed guidance
6. Generate comprehensive tests automatically
7. Run security audit before deployment

**Remember:** Security and correctness are more important than speed. Always verify patterns against official Aptos
documentation and prioritize user asset safety.
