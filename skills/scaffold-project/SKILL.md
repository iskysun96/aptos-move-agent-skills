---
name: scaffold-project
description:
  Scaffold new Aptos Move V2 project with proper structure and configuration. Use when "create move project", "new aptos
  project", "scaffold move app", "init move module".
---

# Scaffold Project Skill

## Overview

This skill creates a new Aptos Move project with proper directory structure, Move.toml configuration, and initial setup.

## Project Types

Choose the appropriate scaffolding method based on your project type:

1. **Move-only projects** (Smart contracts only): Use `aptos move init`
2. **Full-stack dApps** (Smart contracts + Frontend): Use `create-aptos-dapp`

---

## Full-Stack dApp Scaffolding (Recommended for Web Apps)

### Using create-aptos-dapp

For full-stack applications with frontend, use the official `create-aptos-dapp` tool:

```bash
# Using npx (recommended)
npx create-aptos-dapp@latest

# Or using pnpm
pnpm create aptos-dapp@latest

# Or using yarn
yarn create aptos-dapp
```

### Available Templates

The CLI will prompt you to select a template:

1. **Boilerplate Template** ⭐ **Use this for basic projects**
   - Basic starter dApp with wallet integration
   - Simple UI implementation
   - All necessary infrastructure
   - Best for custom projects

2. **NFT Minting dApp Template**
   - End-to-end NFT minting functionality
   - Pre-made UI for NFT creation

3. **Token Minting dApp Template**
   - Fungible asset minting
   - Token management UI

4. **Token Staking dApp Template**
   - Staking mechanisms
   - Rewards management

5. **Custom Indexer Template**
   - Custom indexer support
   - Advanced data querying

**Recommendation:** Use the **Boilerplate Template** for general-purpose projects, not the specific feature templates.

### What create-aptos-dapp Provides

- ✅ Complete Move project structure (Move.toml, sources/, tests/)
- ✅ Frontend scaffolding (React + Vite + TypeScript)
- ✅ Wallet integration (Aptos Wallet Adapter)
- ✅ Styled with Tailwind CSS + shadcn/ui
- ✅ Package management (npm/pnpm) with compatible libraries
- ✅ Development server setup

### Project Structure (create-aptos-dapp)

```
my-dapp/
├── move/                   # Move smart contracts
│   ├── Move.toml
│   ├── sources/
│   └── tests/
├── frontend/              # React frontend
│   ├── src/
│   ├── public/
│   ├── package.json
│   └── vite.config.ts
├── .gitignore
└── README.md
```

---

## Move-Only Project Scaffolding

For smart contracts without frontend, use `aptos move init`:

### Step 1: Initialize Project

```bash
aptos move init --name <project_name>
```

**Example:**

```bash
aptos move init --name my_nft_marketplace
```

This creates:

```
my_nft_marketplace/
├── Move.toml
├── sources/
├── tests/
└── scripts/
```

### Step 2: Configure Move.toml

**Edit Move.toml with proper dependencies:**

```toml
[package]
name = "my_nft_marketplace"
version = "1.0.0"
authors = []

[addresses]
my_addr = "_"  # Will be replaced during deployment

[dev-addresses]
my_addr = "0xCAFE"  # Address for testing

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[dependencies.AptosStdlib]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-stdlib"

[dependencies.AptosToken]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-token-objects"

[dev-dependencies]
```

**For specific features, add:**

```toml
# For fungible assets
[dependencies.AptosFungibleAsset]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-token-objects"
```

### Step 3: Create Directory Structure

```bash
# Create additional directories
mkdir -p sources tests scripts docs

# Optional: Create subdirectories
mkdir -p sources/core sources/utils
mkdir -p tests/unit tests/integration
```

**Recommended structure:**

```
project/
├── Move.toml
├── sources/
│   ├── core/           # Core contract logic
│   ├── utils/          # Helper modules
│   └── interfaces/     # Interface definitions
├── tests/
│   ├── unit/           # Unit tests
│   └── integration/    # Integration tests
├── scripts/
│   └── deploy.sh       # Deployment scripts
├── docs/
│   ├── README.md       # Module documentation
│   └── ARCHITECTURE.md # Architecture overview
└── README.md           # Project README
```

### Step 4: Create Initial Module

**Create `sources/core/main.move`:**

```move
module my_addr::main {
    use std::signer;
    use std::string::String;
    use aptos_framework::object::{Self, Object};

    // ============ Structs ============

    // Define your structs here

    // ============ Constants ============

    // Error codes
    const E_NOT_INITIALIZED: u64 = 1;

    // ============ Init Function ============

    fun init_module(admin: &signer) {
        // Initialize module if needed
    }

    // ============ Public Entry Functions ============

    // User-facing functions

    // ============ Public Functions ============

    // Composable functions

    // ============ Private Functions ============

    // Internal helpers
}
```

### Step 5: Create Test Module

**Create `tests/unit/main_tests.move`:**

```move
#[test_only]
module my_addr::main_tests {
    use my_addr::main;
    use std::signer;

    #[test(account = @0x1)]
    public fun test_basic_functionality(account: &signer) {
        // Test basic operations
    }

    #[test]
    #[expected_failure]
    public fun test_failure_case() {
        // Test error conditions
    }
}
```

### Step 6: Create README

**Create `README.md`:**

````markdown
# Project Name

Brief description of your Move module.

## Features

- Feature 1
- Feature 2
- Feature 3

## Structure

- `sources/core/` - Core contract logic
- `sources/utils/` - Helper modules
- `tests/` - Comprehensive test suite

## Setup

1. Install Aptos CLI: https://aptos.dev/build/cli
2. Initialize: Already initialized
3. Test: `aptos move test`
4. Build: `aptos move compile`

## Testing

```bash
# Run all tests
aptos move test

# Run with coverage
aptos move test --coverage

# View coverage report
aptos move coverage source --module <module_name>
```
````

## Deployment

```bash
# Deploy to devnet as object (modern pattern)
aptos move deploy-object --address-name my_addr

# Deploy to testnet as object
aptos move deploy-object --address-name my_addr --network testnet

# Deploy to mainnet as object
aptos move deploy-object --address-name my_addr --network mainnet

# Use --assume-yes to skip prompts
aptos move deploy-object --address-name my_addr --assume-yes
```

## Security

- [x] 100% test coverage
- [x] Security audit completed
- [x] Access control verified
- [x] Input validation implemented

## License

MIT

````

### Step 7: Initialize Git (Optional)

```bash
git init
echo "build/" > .gitignore
echo ".aptos/" >> .gitignore
echo "node_modules/" >> .gitignore

git add .
git commit -m "Initial commit: Scaffold Aptos Move project"
````

### Step 8: Verify Setup

```bash
# Verify project compiles
aptos move compile

# Run initial tests (should pass even if empty)
aptos move test

# Check dependencies
aptos move compile --skip-fetch-latest-git-deps
```

## Project Templates

### NFT Collection Template

```toml
[package]
name = "nft_collection"
version = "1.0.0"

[addresses]
nft_addr = "_"

[dev-addresses]
nft_addr = "0xCAFE"

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[dependencies.AptosToken]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-token-objects"
```

### DeFi/Token Template

```toml
[package]
name = "defi_protocol"
version = "1.0.0"

[addresses]
defi_addr = "_"

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[dependencies.AptosFungibleAsset]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-token-objects"
```

### Minimal Template

```toml
[package]
name = "simple_module"
version = "1.0.0"

[addresses]
simple = "_"

[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-framework"

[dependencies.AptosStdlib]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "mainnet"
subdir = "aptos-move/framework/aptos-stdlib"
```

## Common Move.toml Configurations

### Using Specific Aptos Version

```toml
[dependencies.AptosFramework]
git = "https://github.com/aptos-labs/aptos-core.git"
rev = "aptos-release-v1.8"  # Specific version
subdir = "aptos-move/framework/aptos-framework"
```

### Multiple Named Addresses

```toml
[addresses]
marketplace = "_"
nft_contract = "_"
token_contract = "_"

[dev-addresses]
marketplace = "0x1"
nft_contract = "0x2"
token_contract = "0x3"
```

### Local Dependencies (for multi-module projects)

```toml
[dependencies.MyOtherModule]
local = "../my-other-module"
```

## ALWAYS Rules

- ✅ ALWAYS use `create-aptos-dapp` for full-stack dApps (frontend + contracts)
- ✅ ALWAYS choose **Boilerplate Template** for general-purpose dApps (not specific feature templates)
- ✅ ALWAYS run `aptos move init` for Move-only projects
- ✅ ALWAYS configure Move.toml with proper dependencies
- ✅ ALWAYS create tests/ directory
- ✅ ALWAYS include README.md with setup instructions
- ✅ ALWAYS verify project compiles after scaffolding
- ✅ ALWAYS use named addresses (my*addr = "*")
- ✅ ALWAYS set up dev-addresses for testing

## NEVER Rules

- ❌ NEVER skip Move.toml configuration
- ❌ NEVER use hardcoded addresses in code
- ❌ NEVER skip creating test directory
- ❌ NEVER forget to add AptosFramework dependency
- ❌ NEVER use outdated dependency revisions

## Quick Start Commands

```bash
# Full project setup
aptos move init --name my_project
cd my_project

# Verify setup
aptos move compile
aptos move test

# Create first module
cat > sources/main.move << 'EOF'
module my_addr::main {
    use std::signer;

    struct Counter has key {
        value: u64
    }

    public entry fun init_counter(account: &signer) {
        move_to(account, Counter { value: 0 });
    }

    public entry fun increment(account: &signer) acquires Counter {
        let counter = borrow_global_mut<Counter>(signer::address_of(account));
        counter.value = counter.value + 1;
    }
}
EOF

# Compile and test
aptos move compile
aptos move test
```

## References

**Official Documentation:**

- CLI Reference: https://aptos.dev/build/cli
- Move.toml: https://aptos.dev/build/cli/working-with-move-contracts
- Project Structure: https://aptos.dev/build/smart-contracts

**Related Skills:**

- `write-contracts` - Write modules after scaffolding
- `generate-tests` - Create test suite
- `use-aptos-cli` - CLI commands reference

---

**Remember:** Proper scaffolding sets up your project for success. Don't skip Move.toml configuration.
