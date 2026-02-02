# Aptos Move V2 Agent Skills

**AI-powered skills for building secure, modern Aptos Move V2 smart contracts**

This repository provides specialized skills and patterns for AI assistants (Claude Code, Cursor, GitHub Copilot, future
Aptos Vibe tool) to help developers build secure, well-tested Aptos Move V2 smart contracts following best practices.

## Features

- **11 Specialized Skills** - Context-aware skills for common Move development tasks
- **Object-Centric Patterns** - Modern V2 object model patterns (no legacy addresses)
- **Security-First** - Comprehensive security checklist and audit patterns
- **100% Test Coverage** - Automated test generation with coverage requirements
- **Modern Syntax** - Inline functions, lambdas, and current Move V2 features
- **Auto-Activation** - Skills trigger automatically based on developer actions

## Quick Start

### For Claude Code Users

1. Clone this repository into your project or alongside it:

   ```bash
   git clone https://github.com/your-org/move-agent-skills.git
   ```

2. The `CLAUDE.md` file will be automatically detected and loaded by Claude Code

3. Start developing - skills activate automatically:
   - "Create a new Move project" → `scaffold-project` skill activates
   - "Write an NFT contract" → `search-aptos-examples` + `write-contracts` activate
   - After writing code → `generate-tests` auto-activates
   - "Deploy this contract" → `security-audit` + `deploy-contracts` activate

### For Other Editors (Cursor, Copilot)

1. Clone this repository
2. Reference skill files in your prompts:
   ```
   @skills/write-contracts/SKILL.md Help me build an NFT marketplace
   ```
3. Include `setups/AGENTS.md` in your workspace context

## Repository Structure

```
move-agent-skills/
├── skills/                           # 11 skill modules
│   ├── scaffold-project/            # Initialize new projects
│   ├── write-contracts/             # Generate Move contracts ⭐
│   ├── generate-tests/              # Create test suites
│   ├── security-audit/              # Security auditing
│   ├── search-aptos-examples/       # Find example patterns
│   ├── use-aptos-cli/               # CLI command reference
│   ├── deploy-contracts/            # Deploy to networks
│   ├── troubleshoot-errors/         # Debug common errors
│   ├── analyze-gas-optimization/    # Optimize gas usage 🆕
│   ├── generate-move-scripts/       # Create atomic scripts 🆕
│   └── implement-upgradeable-contracts/ # Contract upgrades 🆕
├── patterns/                         # Reference documentation
│   ├── DIGITAL_ASSETS.md            # NFT standard (Digital Assets) ⭐
│   ├── OBJECTS.md                   # Object model patterns ⭐
│   ├── SECURITY.md                  # Security checklist ⭐
│   ├── TESTING.md                   # Test patterns
│   ├── MOVE_V2_SYNTAX.md           # Modern syntax guide
│   ├── ADVANCED_TYPES.md            # Advanced type patterns 🆕
│   └── STORAGE_OPTIMIZATION.md      # Storage optimization 🆕
├── setups/
│   ├── AGENTS.md                    # Main AI instruction file ⭐
│   ├── claude-code/README.md       # Claude Code integration
│   ├── cursor/README.md            # Cursor integration
│   └── copilot/README.md           # Copilot integration
├── examples/                        # Example projects (TODO)
│   ├── simple-nft/
│   ├── token-swap/
│   └── dao-voting/
├── CLAUDE.md                        # Auto-loaded by Claude Code
├── README.md                        # This file
└── LICENSE
```

## Core Principles

### 1. Always Search Examples First

```
"I want to build an NFT marketplace"
→ search-aptos-examples skill finds relevant patterns
→ Adapts from official aptos-core examples
```

### 2. Digital Asset Standard for NFTs

```move
// ✅ MODERN: Aptos Digital Asset standard
use aptos_token_objects::collection;
use aptos_token_objects::token;
use aptos_token_objects::aptos_token::AptosToken;

public entry fun list_nft(
    seller: &signer,
    nft: Object<AptosToken>,  // Digital Asset standard
    price: u64
)

// ❌ LEGACY: TokenV1 (deprecated, all migrated)
use aptos_token::token;  // Don't use this module
```

### 3. Object-Centric Development

```move
// ✅ MODERN (V2): Type-safe objects
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,  // Type-safe!
    recipient: address
)

// ❌ LEGACY (V1): Raw addresses
public entry fun transfer_item(
    owner: &signer,
    item_addr: address,  // No type safety
    recipient: address
)
```

### 3. Security-First

```move
// Every entry function verifies authorization
public entry fun update_item(
    owner: &signer,
    item: Object<Item>,
    new_name: String
) acquires Item {
    // ✅ Verify ownership
    assert!(object::owner(item) == signer::address_of(owner), E_NOT_OWNER);

    // ✅ Validate input
    assert!(string::length(&new_name) > 0, E_EMPTY_NAME);

    // Safe to proceed
    let item_data = borrow_global_mut<Item>(object::object_address(&item));
    item_data.name = new_name;
}
```

### 4. 100% Test Coverage

```bash
# Required before deployment
aptos move test --coverage
aptos move coverage summary
# Expected: coverage: 100.0%
```

### 5. Modern Syntax

```move
// Inline functions with lambdas
inline fun for_each<T>(v: &vector<T>, f: |&T|) {
    let i = 0;
    while (i < vector::length(v)) {
        f(vector::borrow(v, i));
        i = i + 1;
    }
}

// Usage
for_each(&items, |item| {
    process(item);
});
```

## Skills Overview

### Critical Skills

#### write-contracts ⭐

**Most important skill** - Generates secure Move V2 contracts following object-centric patterns.

**Activates when:** "write move contract", "create smart contract", "build module"

**Key features:**

- Uses `Object<T>` references (never addresses)
- Implements proper constructor patterns
- Adds access control automatically
- Validates all inputs
- Uses modern V2 syntax

#### generate-tests

Creates comprehensive test suites with 100% coverage requirement.

**Activates when:** "write tests", "add coverage", or automatically after writing contracts

**Test categories:**

- Happy path tests
- Access control tests
- Input validation tests
- Edge case tests

#### security-audit

Performs systematic security audits using comprehensive checklist.

**Activates when:** "audit contract", "check security", or automatically before deployment

**Checks:**

- Access control verification
- Input validation
- Object safety
- Reference safety
- Arithmetic safety
- 100% test coverage

### Supporting Skills

#### scaffold-project

Initializes new Move projects with proper structure and configuration.

**Command:** "create move project", "scaffold new app"

#### search-aptos-examples

Searches aptos-core/move-examples for similar patterns before writing code.

**Auto-activates:** Before writing new contracts

#### use-aptos-cli

Comprehensive CLI command reference for Move development.

**Command:** "how do I compile", "aptos command help"

#### deploy-contracts

Guides safe deployment to devnet, testnet, and mainnet.

**Command:** "deploy contract", "publish to testnet"

**Workflow:** Local tests → Devnet → Testnet → Mainnet

#### troubleshoot-errors

Diagnoses and fixes common Move errors.

**Auto-activates:** When errors detected

## Example Workflows

### Workflow 1: Create NFT Collection

```
User: "Help me create an NFT collection contract"

AI Assistant:
1. ✅ Activates search-aptos-examples → finds token_objects example
2. ✅ Activates write-contracts → generates secure NFT contract
   - Uses Object<NFT> references
   - Implements proper constructor pattern
   - Adds ownership verification
   - Validates all inputs
3. ✅ Activates generate-tests → creates comprehensive test suite
   - Tests NFT creation
   - Tests unauthorized access (should fail)
   - Tests transfers
   - Achieves 100% coverage
4. ✅ Ready to deploy!
```

### Workflow 2: Deploy to Production

```
User: "Deploy my marketplace contract"

AI Assistant:
1. ✅ Activates security-audit → runs full security checklist
   - Verifies access control
   - Checks input validation
   - Confirms 100% test coverage
2. ✅ Activates deploy-contracts → guides deployment
   - Step 1: Deploy to testnet
   - Step 2: Test on testnet thoroughly
   - Step 3: Deploy to mainnet (after testnet success)
3. ✅ Verifies deployment in explorer
4. ✅ Provides deployment documentation
```

### Workflow 3: Fix Compilation Error

```
User: (Compilation error detected)

AI Assistant:
1. ✅ Activates troubleshoot-errors automatically
2. ✅ Analyzes error message
3. ✅ Identifies issue (e.g., "missing semicolon at line 42")
4. ✅ Provides fix with explanation
5. ✅ Verifies fix compiles
```

## Pattern Examples

### Object Creation Pattern

```move
module my_addr::nft {
    use std::string::String;
    use aptos_framework::object::{Self, Object};

    struct NFT has key {
        name: String,
        transfer_ref: object::TransferRef,
        delete_ref: object::DeleteRef,
    }

    /// Create NFT using object pattern
    public fun create_nft(creator: &signer, name: String): Object<NFT> {
        // 1. Create object
        let constructor_ref = object::create_object(signer::address_of(creator));

        // 2. Generate all refs BEFORE constructor_ref destroyed
        let transfer_ref = object::generate_transfer_ref(&constructor_ref);
        let delete_ref = object::generate_delete_ref(&constructor_ref);

        // 3. Get object signer
        let object_signer = object::generate_signer(&constructor_ref);

        // 4. Store data
        move_to(&object_signer, NFT {
            name,
            transfer_ref,
            delete_ref,
        });

        // 5. Return typed object reference
        object::object_from_constructor_ref<NFT>(&constructor_ref)
    }
}
```

### Security Pattern

```move
const E_NOT_OWNER: u64 = 1;
const E_ZERO_AMOUNT: u64 = 2;

public entry fun transfer_with_fee(
    owner: &signer,
    nft: Object<NFT>,
    recipient: address,
    fee: u64
) acquires NFT {
    // ✅ Verify ownership
    assert!(object::owner(nft) == signer::address_of(owner), E_NOT_OWNER);

    // ✅ Validate inputs
    assert!(recipient != @0x0, E_ZERO_ADDRESS);
    assert!(fee > 0, E_ZERO_AMOUNT);

    // Safe to proceed
    // ... transfer logic
}
```

### Testing Pattern

```move
#[test(owner = @0x1, attacker = @0x2)]
#[expected_failure(abort_code = E_NOT_OWNER)]
public fun test_unauthorized_transfer_fails(
    owner: &signer,
    attacker: &signer
) {
    let nft = create_nft(owner, string::utf8(b"Test NFT"));

    // Attacker tries to transfer (should abort)
    transfer_nft(attacker, nft, @0x3);
}
```

## Integration Guides

### Claude Code

1. Clone this repo into your workspace
2. `CLAUDE.md` is automatically detected
3. Skills activate based on context
4. No manual configuration needed

### Cursor

1. Clone this repo
2. Add to `.cursorrules`:
   ```
   When working with Aptos Move:
   - Reference @move-agent-skills/setups/AGENTS.md for guidance
   - Follow patterns in @move-agent-skills/patterns/
   - Use skills from @move-agent-skills/skills/
   ```

### GitHub Copilot

1. Clone this repo
2. Reference files in comments:
   ```move
   // Following pattern from move-agent-skills/patterns/OBJECTS.md
   module my_addr::my_module {
       // ... implementation
   }
   ```

## Formatting

This project uses Prettier for consistent markdown formatting. To format all markdown files:

```bash
# Format all markdown files
npm run format

# Check formatting without making changes
npm run format:check
```

## Configuration

### For AI Assistants

The main configuration is in [`setups/AGENTS.md`](setups/AGENTS.md), which includes:

- **5 Core Workflows** (create, write, test, audit, deploy)
- **Skill Activation Rules** (when each skill triggers)
- **ALWAYS Rules** (required patterns)
- **NEVER Rules** (prohibited patterns)
- **Pattern References** (detailed guides)

### For Developers

No configuration needed. Just clone the repo and start building.

## Documentation

### Pattern Guides (patterns/)

- **[OBJECTS.md](patterns/OBJECTS.md)** - Comprehensive object model guide with examples
- **[SECURITY.md](patterns/SECURITY.md)** - Security checklist and vulnerability patterns
- **[TESTING.md](patterns/TESTING.md)** - Test generation and coverage patterns
- **[MOVE_V2_SYNTAX.md](patterns/MOVE_V2_SYNTAX.md)** - Modern Move V2 syntax guide

### Skill Reference (skills/)

Each skill has detailed documentation in `skills/<skill-name>/SKILL.md`:

- Overview and when to use
- Step-by-step workflows
- Code examples
- Common patterns
- ALWAYS/NEVER rules
- References

## Contributing

We welcome contributions! Areas to contribute:

1. **Example Projects** - Add complete example projects to `examples/`
2. **Additional Skills** - Propose new skills for common tasks
3. **Pattern Improvements** - Enhance existing pattern documentation
4. **Bug Fixes** - Fix issues in skill logic or examples
5. **Integration Guides** - Add guides for other editors/tools

### Contribution Guidelines

1. Follow existing documentation structure
2. Include code examples for all patterns
3. Add tests for example projects
4. Ensure security patterns are correct
5. Reference official Aptos documentation

## Resources

### Official Aptos Documentation

- **Object Model:** https://aptos.dev/build/smart-contracts/object
- **Security Guidelines:** https://aptos.dev/build/smart-contracts/move-security-guidelines
- **Move Book:** https://aptos.dev/build/smart-contracts/book
- **CLI Reference:** https://aptos.dev/build/cli
- **Testing:** https://aptos.dev/build/smart-contracts/book/unit-testing

### Example Code

- **aptos-core Examples:** https://github.com/aptos-labs/aptos-core/tree/main/aptos-move/move-examples
- Priority examples: `token_objects`, `fungible_asset`, `dao`, `marketplace`

### Inspired By

This project is modeled after the [Algorand Agent Skills](https://github.com/algorand-devrel/algorand-agent-skills)
repository structure.

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Support

- **Issues:** Report bugs or request features via GitHub Issues
- **Discussions:** Join discussions about patterns and best practices
- **Documentation:** All documentation is in this repository

## Roadmap

- [x] Core infrastructure (patterns, skills, setup)
- [x] 8 essential skills implemented
- [ ] Example projects (NFT collection, token swap, DAO)
- [ ] MCP server integration (optional)
- [ ] Video tutorials and demos
- [ ] Integration with Aptos Vibe tool (when available)
- [ ] Community-contributed skills and patterns

## Version

**v1.0** (2026-01-23)

- Initial release
- 8 core skills
- 4 comprehensive pattern guides
- Object-centric patterns enforced
- Security-first approach
- 100% test coverage requirement

---

**Built for secure, modern Aptos Move V2 development with AI assistance.**

Start building: `git clone https://github.com/your-org/move-agent-skills.git`
