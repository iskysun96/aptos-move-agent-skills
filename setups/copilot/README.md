# GitHub Copilot Integration Guide

This guide explains how to use Aptos Move V2 Agent Skills with GitHub Copilot.

## Setup

### Step 1: Clone Repository

Clone this repository into your workspace:

```bash
cd your-move-project
git clone https://github.com/your-org/move-agent-skills.git
```

### Step 2: Reference in Comments

GitHub Copilot uses comments and context to generate code. Reference patterns in your comments:

```move
// Following pattern from move-agent-skills/patterns/OBJECTS.md
// Create NFT using Object<T> with proper constructor pattern
module my_addr::nft {
    // Copilot will suggest code based on the pattern
}
```

## Usage Patterns

### Pattern 1: Inline Comments

Use specific comments to guide Copilot:

```move
// Pattern: move-agent-skills/patterns/OBJECTS.md - Object Creation
// Create NFT object with TransferRef and DeleteRef
public fun create_nft(creator: &signer, name: String): Object<NFT> {
    // Copilot suggests: proper object creation pattern
}
```

### Pattern 2: Function-Level Documentation

Document what you want at the function level:

```move
/// Create NFT collection using modern Object<T> pattern
/// Security: Verify signer authority, validate inputs
/// Pattern: move-agent-skills/skills/write-contracts/SKILL.md
public entry fun create_collection(
    creator: &signer,
    name: String,
    description: String
) {
    // Copilot suggests implementation
}
```

### Pattern 3: File-Level Instructions

Add instructions at the top of files:

```move
/// Module: NFT Marketplace
/// Patterns:
/// - move-agent-skills/patterns/OBJECTS.md for object handling
/// - move-agent-skills/patterns/SECURITY.md for security checks
/// - Always use Object<T>, never addresses
/// - Verify signer authority in all entry functions
/// - Validate all inputs

module my_addr::marketplace {
    // Copilot follows these guidelines
}
```

## Copilot Chat Integration

### Using Copilot Chat

Open Copilot Chat and reference patterns:

```
Reference: move-agent-skills/skills/write-contracts/SKILL.md

Generate an NFT marketplace contract with:
- Fixed price listings
- Offer system
- Platform fees
- Object<T> references
```

### Multi-Turn Conversations

Have iterative conversations:

```
You: Based on move-agent-skills/patterns/OBJECTS.md,
     create an NFT contract

Copilot: [Generates contract]

You: Add transfer functionality following security patterns
     from move-agent-skills/patterns/SECURITY.md

Copilot: [Adds secure transfer function]

You: Generate tests per move-agent-skills/patterns/TESTING.md

Copilot: [Generates comprehensive tests]
```

## Best Practices

### 1. Reference Patterns in Comments

Always reference relevant patterns:

```move
// PATTERN: move-agent-skills/patterns/OBJECTS.md
// SECURITY: move-agent-skills/patterns/SECURITY.md
// Generate all refs in constructor before ConstructorRef destroyed
```

### 2. Use TODO Comments with Patterns

Create structured TODOs:

```move
// TODO: Implement following move-agent-skills/patterns/OBJECTS.md
// TODO: Add security checks from move-agent-skills/patterns/SECURITY.md
// TODO: Generate tests per move-agent-skills/patterns/TESTING.md
```

### 3. Document Expected Behavior

Be specific about what you want:

```move
// Create NFT object
// MUST: Use Object<NFT> return type (not address)
// MUST: Generate TransferRef and DeleteRef in constructor
// MUST: Verify signer::address_of(creator) == expected
// PATTERN: move-agent-skills/patterns/OBJECTS.md
```

### 4. Reference Skills for Complete Functions

```move
// SKILL: move-agent-skills/skills/write-contracts/SKILL.md
// Generate secure transfer function with:
// - Ownership verification
// - Input validation
// - Proper error codes
```

## Code Examples

### Example 1: Object Creation

```move
// Pattern: move-agent-skills/patterns/OBJECTS.md - Standard Object Creation
// Create item object with proper refs
public fun create_item(creator: &signer, name: String): Object<Item> {
    // Copilot suggests:
    let constructor_ref = object::create_object(signer::address_of(creator));
    let transfer_ref = object::generate_transfer_ref(&constructor_ref);
    let delete_ref = object::generate_delete_ref(&constructor_ref);
    let object_signer = object::generate_signer(&constructor_ref);

    move_to(&object_signer, Item {
        name,
        transfer_ref,
        delete_ref,
    });

    object::object_from_constructor_ref<Item>(&constructor_ref)
}
```

### Example 2: Security Checks

```move
// Security: move-agent-skills/patterns/SECURITY.md - Access Control
// Verify ownership before transfer
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,
    recipient: address
) acquires Item {
    // Copilot suggests security checks:
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);
    assert!(recipient != @0x0, E_ZERO_ADDRESS);

    // Transfer logic...
}
```

### Example 3: Test Generation

```move
// Testing: move-agent-skills/patterns/TESTING.md
// Test unauthorized access (should fail)
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_unauthorized_transfer_fails(
    owner: &signer,
    attacker: &signer
) {
    // Copilot suggests test implementation
}
```

## Copilot Labs Features

### Explain Code

1. Select code
2. Open Copilot Labs
3. Click "Explain"
4. Add context: "Explain based on move-agent-skills/patterns/OBJECTS.md"

### Translate to Tests

1. Select function
2. Open Copilot Labs
3. Click "Test"
4. Add: "Follow move-agent-skills/patterns/TESTING.md patterns"

### Fix Bug

1. Select problematic code
2. Copilot Labs → "Fix"
3. Reference: "Fix per move-agent-skills/skills/troubleshoot-errors/SKILL.md"

## VS Code Integration

### Settings

Add to `.vscode/settings.json`:

```json
{
  "github.copilot.advanced": {
    "authProvider": "github",
    "inlineSuggest.enable": true
  },
  "github.copilot.enable": {
    "*": true,
    "yaml": true,
    "plaintext": true,
    "markdown": true,
    "move": true
  }
}
```

### Workspace Snippets

Create `.vscode/move.code-snippets`:

```json
{
  "Aptos Object Creation": {
    "prefix": "aptos-object",
    "body": [
      "// Pattern: move-agent-skills/patterns/OBJECTS.md",
      "public fun create_${1:object}(creator: &signer) : Object<${2:Type}> {",
      "    let constructor_ref = object::create_object(signer::address_of(creator));",
      "    let transfer_ref = object::generate_transfer_ref(&constructor_ref);",
      "    let delete_ref = object::generate_delete_ref(&constructor_ref);",
      "    let object_signer = object::generate_signer(&constructor_ref);",
      "    ",
      "    move_to(&object_signer, ${2:Type} {",
      "        $0",
      "        transfer_ref,",
      "        delete_ref,",
      "    });",
      "    ",
      "    object::object_from_constructor_ref<${2:Type}>(&constructor_ref)",
      "}"
    ],
    "description": "Create Aptos object following patterns"
  }
}
```

## Prompt Engineering Tips

### Tip 1: Be Explicit About Patterns

```move
// EXPLICIT: Use Object<NFT>, not address
// EXPLICIT: Generate refs in constructor
// PATTERN: move-agent-skills/patterns/OBJECTS.md
```

### Tip 2: List Requirements

```move
// Requirements from move-agent-skills/patterns/SECURITY.md:
// 1. Verify signer authority
// 2. Validate all inputs (amount > 0, amount <= MAX)
// 3. Check ownership: object::owner(obj) == user
// 4. Use clear error codes (E_NOT_OWNER, E_INVALID_AMOUNT)
```

### Tip 3: Reference Examples

```move
// Example: move-agent-skills/patterns/OBJECTS.md - Pattern 2
// Similar to: aptos-core/move-examples/token_objects/sources/token.move
```

### Tip 4: Specify What NOT to Do

```move
// NEVER: Return ConstructorRef from this function
// NEVER: Use raw address for item (use Object<Item>)
// NEVER: Skip signer verification
// REF: move-agent-skills/skills/write-contracts/SKILL.md - NEVER Rules
```

## Keyboard Shortcuts

- **Accept Suggestion:** `Tab`
- **Dismiss Suggestion:** `Esc`
- **Next Suggestion:** `Alt/Option + ]`
- **Previous Suggestion:** `Alt/Option + [`
- **Open Copilot Chat:** `Ctrl + Shift + I` (Windows/Linux) or `Cmd + Shift + I` (Mac)

## Limitations

### Context Window

Copilot has limited context window. For best results:

1. Keep relevant patterns in open files
2. Reference patterns in comments near code
3. Use Copilot Chat for longer conversations

### Pattern Recognition

Copilot learns from:

- Your current file
- Open files in editor
- Comments and documentation
- Your coding history

Improve recognition:

- Keep pattern files open
- Add explicit comments
- Use consistent naming

### Move Language Support

Move is relatively new, so:

- Be more explicit than with popular languages
- Verify generated code against patterns
- Test thoroughly (100% coverage!)

## Troubleshooting

### Suggestions Not Following Patterns

**Solution:**

1. Add more explicit comments referencing patterns
2. Keep pattern files open in editor
3. Use Copilot Chat with explicit references

### Incorrect Object Pattern

**Solution:**

```move
// CRITICAL: Return Object<T>, NOT ConstructorRef
// REF: move-agent-skills/patterns/OBJECTS.md - Anti-Patterns
// WRONG: return constructor_ref;
// RIGHT: return object::object_from_constructor_ref(&constructor_ref);
```

### Missing Security Checks

**Solution:**

```move
// REQUIRED SECURITY CHECKS (move-agent-skills/patterns/SECURITY.md):
// 1. assert!(signer::address_of(user) == expected, E_UNAUTHORIZED);
// 2. assert!(object::owner(obj) == user_addr, E_NOT_OWNER);
// 3. assert!(amount > 0, E_ZERO_AMOUNT);
```

## Example Workflow

```move
// FILE: sources/marketplace.move
// Pattern References:
// - move-agent-skills/patterns/OBJECTS.md (object handling)
// - move-agent-skills/patterns/SECURITY.md (security checks)
// - move-agent-skills/skills/write-contracts/SKILL.md (complete examples)

module my_addr::marketplace {
    use std::signer;
    use std::string::String;
    use aptos_framework::object::{Self, Object};

    // Error codes from move-agent-skills/patterns/SECURITY.md
    const E_NOT_OWNER: u64 = 1;
    const E_ZERO_AMOUNT: u64 = 10;

    struct Listing has key {
        item: Object<Item>,
        price: u64,
        seller: address,
    }

    // Pattern: move-agent-skills/patterns/OBJECTS.md - Object Creation
    // Create listing with proper object pattern
    public entry fun create_listing(
        seller: &signer,
        item: Object<Item>,
        price: u64
    ) {
        // Security: move-agent-skills/patterns/SECURITY.md
        // Copilot suggests ownership and validation checks
        assert!(object::owner(item) == signer::address_of(seller), E_NOT_OWNER);
        assert!(price > 0, E_ZERO_AMOUNT);

        // Copilot suggests proper implementation
    }
}

// FILE: tests/marketplace_tests.move
// Testing: move-agent-skills/patterns/TESTING.md

#[test_only]
module my_addr::marketplace_tests {
    // Pattern: move-agent-skills/patterns/TESTING.md - Access Control Tests
    // Test unauthorized access (should fail)
    #[test(owner = @0x1, attacker = @0x2)]
    #[expected_failure(abort_code = E_NOT_OWNER)]
    public fun test_unauthorized_listing(
        owner: &signer,
        attacker: &signer
    ) {
        // Copilot suggests test implementation
    }
}
```

## Getting Help

- **Copilot Docs:** https://docs.github.com/copilot
- **Pattern Docs:** `move-agent-skills/setups/AGENTS.md`
- **VS Code:** Press `F1` → "GitHub Copilot: Open Documentation"

## Updates

Keep patterns updated:

```bash
cd move-agent-skills
git pull origin main
```

---

**Happy coding with GitHub Copilot and Aptos Move V2!**
