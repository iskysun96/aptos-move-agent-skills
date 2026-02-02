# Lessons Learned from NFT Marketplace Debugging Session

**Date:** 2026-01-24 **Context:** Debugging object deployment and digital assets standard implementation

---

## Summary

During the implementation and deployment of an NFT marketplace contract, several critical mistakes were made that
provide valuable lessons for future Move development. This document catalogs those mistakes, explains why they happened,
and provides clear prevention strategies.

---

## Mistake #1: Wrong Deployment Command

### What Happened

Used `aptos move create-object-and-publish-package` instead of `aptos move deploy-object` for object deployment.

### Why It Happened

- Lack of familiarity with Aptos CLI object deployment commands
- Multiple commands exist for object deployment, causing confusion
- No clear documentation reference consulted before choosing command

### The Correct Approach

```bash
# ✅ CORRECT: Simple, recommended command
aptos move deploy-object \
    --address-name marketplace_addr \
    --profile devnet \
    --assume-yes

# ❌ WRONG: More complex, not recommended for standard cases
aptos move create-object-and-publish-package \
    --address-name marketplace_addr \
    --named-addresses marketplace_addr=default
```

### Prevention Strategy

1. **Always consult aptos.dev** before using deployment commands
2. **Use `deploy-object`** for modern object-based contracts (simpler, recommended)
3. **Only use `create-object-and-publish-package`** if you need advanced object configuration
4. **Document the correct command** in your project's deployment guide

### Updated Documentation

- ✅ `skills/deploy-contracts/SKILL.md` - Added clear section on object deployment commands

---

## Mistake #2: Added Unnecessary Helper Function

### What Happened

Created `get_module_address(): address { @marketplace_addr }` helper function, thinking that `@marketplace_addr` wasn't
resolving correctly with object deployment.

### Why It Happened

- Misdiagnosed the root cause of the "object does not exist" error
- Assumed the problem was with address resolution rather than initialization logic
- Tried to "abstract" the address access without understanding the real issue

### The Correct Approach

```move
// ❌ WRONG: Unnecessary helper
fun get_module_address(): address {
    @marketplace_addr
}

// ✅ CORRECT: Use @marketplace_addr directly
let config = borrow_global<MarketplaceConfig>(@marketplace_addr);
```

### Prevention Strategy

1. **Don't add abstractions prematurely** - Understand the problem first
2. **`@marketplace_addr` works directly** - No helper needed for named addresses
3. **Debug systematically** - Trace through the code to find the real issue
4. **Test hypotheses** - Verify assumptions before implementing fixes

---

## Mistake #3: Wrong Diagnosis About dev-addresses

### What Happened

Thought that the `[dev-addresses]` section in `Move.toml` was interfering with object deployment and removed it, which
didn't fix the issue.

### Why It Happened

- Made an assumption without fully understanding the compilation process
- Didn't understand that `dev-addresses` only affects `--dev` flag compilation
- Rushed to a solution without investigating the actual cause

### The Correct Understanding

```toml
# Move.toml
[addresses]
marketplace_addr = "_"  # Placeholder for deployment

[dev-addresses]
marketplace_addr = "0xca8b..."  # Only used with --dev flag
```

**How it actually works:**

- `[dev-addresses]` only applies when using `aptos move compile --dev`
- During deployment, the CLI replaces `_` with the object address
- Removing `[dev-addresses]` doesn't affect deployment

### Prevention Strategy

1. **Understand compilation vs deployment** - They use different address resolution
2. **Read Aptos documentation** on named addresses and how they work
3. **Test systematically** - Try one change at a time and verify results
4. **Don't remove configuration** without understanding what it does

---

## Mistake #4: Stored Wrong extend_ref (Root Cause)

### What Happened

Stored the collection's `extend_ref` instead of creating a separate marketplace state object with its own `extend_ref`.

### Why It Happened

- **Critical misunderstanding of object ownership hierarchy**
- Didn't understand that collections can't create tokens in themselves
- The collection owner (not the collection itself) must create tokens
- Tried to use the collection's signer to create tokens, which doesn't work

### The Wrong Pattern

```move
// ❌ WRONG: Storing collection's extend_ref
fun init_module(deployer: &signer) {
    // Create collection directly
    let collection_ref = collection::create_unlimited_collection(deployer, ...);
    let collection_signer = object::generate_signer(&collection_ref);

    // Store COLLECTION's extend_ref (WRONG!)
    move_to(deployer, Config {
        extend_ref: object::generate_extend_ref(&collection_ref),
    });
}

public entry fun mint_nft() acquires Config {
    let config = borrow_global<Config>(...);

    // Try to use collection's signer (DOESN'T WORK!)
    let collection_signer = object::generate_signer_for_extending(&config.extend_ref);
    token::create_named_token(&collection_signer, ...);  // ❌ Fails!
}
```

**Why this fails:**

- `token::create_named_token()` requires the signer to be the OWNER of the collection
- The collection can't be its own owner
- Collections are owned by the address/object that created them

### The Correct Pattern

```move
// ✅ CORRECT: Create marketplace object that owns collection
fun init_module(deployer: &signer) {
    // 1. Create marketplace state object
    let marketplace_ref = object::create_named_object(deployer, b"MARKETPLACE_STATE");
    let marketplace_signer = object::generate_signer(&marketplace_ref);

    // 2. Marketplace object creates and owns the collection
    collection::create_unlimited_collection(&marketplace_signer, ...);

    // 3. Store MARKETPLACE object's extend_ref (CORRECT!)
    move_to(&marketplace_signer, MarketplaceConfig {
        extend_ref: object::generate_extend_ref(&marketplace_ref),
    });
}

public entry fun mint_nft() acquires MarketplaceConfig {
    let config = borrow_global<MarketplaceConfig>(...);

    // Use marketplace signer (collection owner)
    let marketplace_signer = object::generate_signer_for_extending(&config.extend_ref);
    token::create_named_token(&marketplace_signer, ...);  // ✅ Works!
}
```

**Object Ownership Hierarchy:**

```
Marketplace Object (created with create_named_object)
  └── Owns Collection (created by marketplace signer)
      └── Contains Tokens (created by marketplace signer)
```

### Prevention Strategy

1. **Understand the ownership hierarchy** - Who owns what?
2. **Collection owner creates tokens** - Not the collection itself
3. **Create a parent object** for marketplace state that owns the collection
4. **Store the parent object's extend_ref** - Not the collection's
5. **Test token creation immediately** after deployment

### Updated Documentation

- ✅ `patterns/DIGITAL_ASSETS.md` - Added "CRITICAL: Object Ownership Hierarchy" section
- ✅ `skills/troubleshoot-errors/SKILL.md` - Added "An object does not exist at this address" error

---

## Mistake #5: Used Wrong Transfer Function

### What Happened

Used `object::transfer()` instead of `object::transfer_with_ref()` after calling `object::disable_ungated_transfer()`.

### Why It Happened

- Didn't understand the difference between the two transfer functions
- Didn't know that `object::transfer()` requires ungated transfers to be enabled
- Assumed `object::transfer()` was the universal transfer function

### The Error

```
Error: "The object does not have ungated transfers enabled"
```

### The Wrong Pattern

```move
// ❌ WRONG: Using object::transfer() after disabling ungated transfers
let constructor_ref = token::create_named_token(...);
let transfer_ref = object::generate_transfer_ref(&constructor_ref);

// Disable ungated transfers
object::disable_ungated_transfer(&transfer_ref);

let token_obj = object::object_from_constructor_ref<Token>(&constructor_ref);

// ❌ FAILS: Can't use object::transfer() when ungated transfers are disabled
object::transfer(creator, token_obj, recipient);
```

### The Correct Pattern

```move
// ✅ CORRECT: Using object::transfer_with_ref() after disabling ungated transfers
let constructor_ref = token::create_named_token(...);
let transfer_ref = object::generate_transfer_ref(&constructor_ref);

// Disable ungated transfers
object::disable_ungated_transfer(&transfer_ref);

// ✅ Use transfer_with_ref instead
let linear_transfer_ref = object::generate_linear_transfer_ref(&transfer_ref);
object::transfer_with_ref(linear_transfer_ref, recipient);
```

### Understanding Transfer Functions

**Two transfer functions exist:**

1. **`object::transfer(signer, obj, to)`**
   - For ungated transfers (enabled by default)
   - Anyone can transfer if ungated is enabled
   - Does NOT require TransferRef

2. **`object::transfer_with_ref(LinearTransferRef, to)`**
   - For controlled transfers
   - Works even when ungated transfers are disabled
   - REQUIRES TransferRef

**Decision Tree:**

```
Did you call object::disable_ungated_transfer()?
├─ YES → Use object::transfer_with_ref()
│         Must have TransferRef stored
│
└─ NO (ungated transfers enabled by default)
    └─ Use object::transfer()
       Works without TransferRef
```

### Prevention Strategy

1. **Know the two transfer functions** and when to use each
2. **If ungated transfers disabled** → MUST use `transfer_with_ref()`
3. **Store TransferRef** in your struct for future transfers
4. **Do initial transfer BEFORE storing** the struct (while you still have constructor_ref)
5. **Test transfers immediately** after minting

### Updated Documentation

- ✅ `patterns/OBJECTS.md` - Added "CRITICAL: Understanding Transfer Functions" section
- ✅ `skills/troubleshoot-errors/SKILL.md` - Added "The object does not have ungated transfers enabled" error

---

## Summary of All Mistakes

| #   | Mistake                     | Root Cause                     | Prevention                                                       |
| --- | --------------------------- | ------------------------------ | ---------------------------------------------------------------- |
| 1   | Wrong deployment command    | Lack of CLI knowledge          | Always use `deploy-object` for objects; consult docs first       |
| 2   | Unnecessary helper function | Premature abstraction          | Use `@marketplace_addr` directly; debug systematically           |
| 3   | Removed dev-addresses       | Misdiagnosis of issue          | Understand compilation vs deployment; test one change at a time  |
| 4   | Stored wrong extend_ref     | Misunderstood object ownership | Create parent object that owns collection; parent creates tokens |
| 5   | Wrong transfer function     | Didn't know two functions      | Use `transfer_with_ref()` when ungated transfers disabled        |

---

## Key Learnings

### 1. Object Ownership Hierarchy is Critical

- Collections can't create tokens in themselves
- The OWNER of the collection creates tokens
- For marketplaces: Create a parent "marketplace state" object that owns the collection

### 2. Transfer Functions Are Not Interchangeable

- `object::transfer()` = ungated transfers enabled
- `object::transfer_with_ref()` = ungated transfers disabled
- Using the wrong one causes runtime errors

### 3. Always Consult Documentation First

- Don't guess at CLI commands - look them up
- aptos.dev has comprehensive documentation
- Test commands on devnet first

### 4. Debug Systematically

- Don't jump to conclusions
- Trace through the code step by step
- Test one hypothesis at a time
- Verify assumptions before implementing fixes

### 5. Understand Before Abstracting

- Don't add helper functions prematurely
- Understand the problem fully first
- Direct access is often clearer than abstraction

---

## Best Practices Going Forward

### For Object-Based Contracts

1. **Create a parent state object** that owns collections
2. **Store the parent's extend_ref** for creating tokens later
3. **Use marketplace signer** (not collection signer) to create tokens
4. **Always use `deploy-object`** for deployment
5. **Test token creation immediately** after deployment

### For NFT Transfers

1. **Decide on transfer model early**: ungated or controlled?
2. **If controlled** (recommended for marketplaces):
   - Call `object::disable_ungated_transfer()`
   - Always use `object::transfer_with_ref()`
   - Store `TransferRef` in your NFT data struct
3. **Do initial transfer BEFORE storing** the struct

### For Deployment

1. **Always use correct command**: `aptos move deploy-object`
2. **Test on devnet first** before testnet/mainnet
3. **Verify deployment** by calling view functions
4. **Document the object address** for future reference
5. **Test all entry functions** after deployment

---

## Conclusion

These mistakes highlight the importance of:

- Understanding Aptos object model deeply
- Knowing the correct CLI commands
- Debugging systematically rather than guessing
- Consulting documentation before implementing
- Testing immediately after changes

The updated skill files now contain all these learnings to prevent future mistakes.

**Files Updated:**

- ✅ `patterns/DIGITAL_ASSETS.md` - Object ownership hierarchy
- ✅ `patterns/OBJECTS.md` - Transfer function decision tree
- ✅ `skills/deploy-contracts/SKILL.md` - Correct deployment commands
- ✅ `skills/troubleshoot-errors/SKILL.md` - Common object/transfer errors
- ✅ `LESSONS_LEARNED.md` - This document

**All future AI agents working with Aptos Move will benefit from these documented lessons.**
