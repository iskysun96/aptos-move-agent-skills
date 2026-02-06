---
name: scaffold-project
description:
  "Initializes new Aptos dApp projects using degit to bootstrap from official templates. Triggers on: 'create project',
  'scaffold project', 'new dApp', 'new Move project', 'initialize project', 'setup project', 'start new contract', 'init
  aptos project', 'create fullstack dapp'."
metadata:
  category: project
  tags: ["scaffolding", "templates", "project-setup", "dapp"]
  priority: high
---

# Scaffold Project Skill

## Overview

This skill creates new Aptos dApp projects by bootstrapping directly from official templates using `degit`. This
approach provides clean copies of production-ready templates without git history.

## Project Types

| Type               | Template                        | Use Case                         |
| ------------------ | ------------------------------- | -------------------------------- |
| **Fullstack dApp** | `boilerplate-template`          | Frontend + smart contracts       |
| **Contract-only**  | `contract-boilerplate-template` | Smart contracts without frontend |

---

## Fullstack dApp Scaffolding

### Step 1: Bootstrap with degit

```bash
# Bootstrap fullstack template (no git history)
npx degit aptos-labs/create-aptos-dapp/templates/boilerplate-template my-dapp

cd my-dapp
```

> **Note:** The degit command references a specific template path in the aptos-labs/create-aptos-dapp repository. If you
> encounter errors, verify the template path exists at
> https://github.com/aptos-labs/create-aptos-dapp/tree/main/templates

### Step 2: Configure Environment

**Create `.env` file** with the following variables:

```bash
# Create .env file
cat > .env << 'EOF'
PROJECT_NAME=my-dapp
VITE_APP_NETWORK=devnet
VITE_APTOS_API_KEY=""
VITE_MODULE_PUBLISHER_ACCOUNT_ADDRESS=
# This is the module publisher account's private key.
# Be cautious about who you share it with, and ensure it is not exposed when deploying your dApp.
VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY=
EOF
```

**Configure the values:**

- `PROJECT_NAME` - Your project name
- `VITE_APP_NETWORK` - Network to use (`devnet`, `testnet`, or `mainnet`)
- `VITE_APTOS_API_KEY` - Optional API key from Aptos Labs
- `VITE_MODULE_PUBLISHER_ACCOUNT_ADDRESS` - Your deployer account address (set after `aptos init`)
- `VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY` - Your deployer private key (from `~/.aptos/config.yaml`)

**⚠️ CRITICAL: Ensure `.env` is in `.gitignore`:**

```bash
# Verify .env is gitignored (should already be there)
grep -q "^\.env$" .gitignore || echo ".env" >> .gitignore
```

### Step 3: Update Move.toml

Edit `contract/Move.toml` with your project name:

```toml
[package]
name = "my_dapp"  # Your project name
version = "1.0.0"
authors = []

[addresses]
my_dapp_addr = "_"  # Will be set during deployment

[dev-addresses]
my_dapp_addr = "0xCAFE"  # For testing

[dependencies]
AptosFramework = { git = "https://github.com/aptos-labs/aptos-framework.git", rev = "mainnet", subdir = "aptos-framework" }
```

### Step 4: Install Dependencies

```bash
npm install
```

### Step 5: Initialize Git

```bash
git init
git add .
git commit -m "Initial commit: Bootstrap Aptos dApp from boilerplate template"
```

### Step 6: Verify Setup

```bash
# Compile Move contracts
npm run move:compile

# Run Move tests
npm run move:test

# Start frontend development server
npm run dev
```

### Fullstack Project Structure

```
my-dapp/
├── frontend/
│   ├── components/           # React UI components
│   ├── entry-functions/      # Write operations (transactions)
│   ├── view-functions/       # Read operations (queries)
│   ├── lib/                  # Shared libraries (wallet, aptos client)
│   ├── utils/                # Helpers
│   ├── App.tsx
│   ├── constants.ts
│   └── main.tsx
├── contract/
│   ├── sources/              # Move modules
│   ├── tests/                # Move tests
│   └── Move.toml
├── scripts/move/             # Deployment scripts
├── package.json              # npm scripts for move:compile, move:test, etc.
├── .env                      # Environment variables (NEVER commit!)
├── .gitignore                # Must include .env
└── [config files]            # vite, tailwind, typescript, etc.
```

### Key Directories Explained

| Directory                   | Purpose                                   |
| --------------------------- | ----------------------------------------- |
| `frontend/entry-functions/` | Transaction payloads for write operations |
| `frontend/view-functions/`  | Queries for read operations               |
| `frontend/lib/`             | Aptos client and wallet provider setup    |
| `contract/sources/`         | Move smart contract modules               |
| `scripts/move/`             | Deployment and utility scripts            |

---

## Contract-Only Scaffolding

### Step 1: Bootstrap with degit

```bash
# Bootstrap contract-only template
npx degit aptos-labs/create-aptos-dapp/templates/contract-boilerplate-template my-contract

cd my-contract
```

### Step 2: Configure Environment

**Create `.env` file** with the following variables:

```bash
# Create .env file
cat > .env << 'EOF'
PROJECT_NAME=my-contract
VITE_APP_NETWORK=devnet
VITE_APTOS_API_KEY=""
VITE_MODULE_PUBLISHER_ACCOUNT_ADDRESS=
# This is the module publisher account's private key.
# Be cautious about who you share it with, and ensure it is not exposed when deploying your dApp.
VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY=
EOF

# Ensure .env is gitignored
grep -q "^\.env$" .gitignore || echo ".env" >> .gitignore
```

### Step 3: Update Move.toml

Edit `contract/Move.toml`:

```toml
[package]
name = "my_contract"
version = "1.0.0"

[addresses]
my_contract_addr = "_"

[dev-addresses]
my_contract_addr = "0xCAFE"

[dependencies]
AptosFramework = { git = "https://github.com/aptos-labs/aptos-framework.git", rev = "mainnet", subdir = "aptos-framework" }
```

### Step 4: Install & Verify

```bash
npm install

# Compile
npm run move:compile

# Test
npm run move:test
```

### Step 5: Initialize Git

```bash
git init
git add .
git commit -m "Initial commit: Bootstrap Aptos contract from template"
```

### Contract-Only Project Structure

```
my-contract/
├── contract/
│   ├── sources/              # Move modules
│   ├── tests/                # Move tests
│   └── Move.toml
├── scripts/move/             # Deployment scripts
├── package.json              # npm scripts
├── .env                      # Environment variables (NEVER commit!)
└── .gitignore                # Must include .env
```

---

## Available npm Scripts

Both templates include these npm scripts:

```bash
# Move development
npm run move:compile    # Compile Move contracts
npm run move:test       # Run Move tests
npm run move:publish    # Publish to network (uses .env)

# Fullstack only
npm run dev             # Start frontend dev server
npm run build           # Build for production
```

---

## Alternative: Manual Move-Only Setup

For pure Move development without the npm wrapper, use `aptos move init`:

```bash
# Initialize Move project
aptos move init --name my_module

# Configure Move.toml manually
# Create sources/ and tests/ directories
```

See the "Move-Only Reference" section below for detailed manual setup.

---

## Quick Reference Commands

### Fullstack dApp (Recommended)

```bash
# Bootstrap and setup
npx degit aptos-labs/create-aptos-dapp/templates/boilerplate-template my-dapp
cd my-dapp
npm install
git init

# Then create .env manually (see Step 2 above) - NEVER commit .env!
```

### Contract-Only

```bash
# Bootstrap and setup
npx degit aptos-labs/create-aptos-dapp/templates/contract-boilerplate-template my-contract
cd my-contract
npm install
git init

# Then create .env manually (see Step 2 above) - NEVER commit .env!
```

---

## Post-Scaffolding Checklist

After bootstrapping, complete these steps:

- [ ] Create `.env` file with required variables (see Step 2)
- [ ] Verify `.env` is in `.gitignore`
- [ ] Update `Move.toml` with project name and address alias
- [ ] Run `aptos init` to create deployer account
- [ ] Add account address and private key to `.env`
- [ ] Verify compilation: `npm run move:compile`
- [ ] Verify tests pass: `npm run move:test`
- [ ] Initialize git repository
- [ ] (Fullstack) Verify frontend runs: `npm run dev`

**⚠️ Before committing:** Double-check that `.env` is NOT staged (`git status`)

---

## Move-Only Reference (Manual Setup)

For cases where you need manual Move setup without templates:

### Initialize

```bash
aptos move init --name my_module
```

### Configure Move.toml

```toml
[package]
name = "my_module"
version = "1.0.0"

[addresses]
my_addr = "_"

[dev-addresses]
my_addr = "0xCAFE"

[dependencies]
AptosFramework = { git = "https://github.com/aptos-labs/aptos-framework.git", rev = "mainnet", subdir = "aptos-framework" }
```

### Create Module

```move
// sources/main.move
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
```

### Verify

```bash
aptos move compile
aptos move test
```

---

## ALWAYS Rules

- ✅ ALWAYS use `degit` for bootstrapping (clean copy, no git history)
- ✅ ALWAYS use the **boilerplate-template** for fullstack dApps
- ✅ ALWAYS update Move.toml with your project name and address alias
- ✅ ALWAYS create `.env` file with required environment variables
- ✅ ALWAYS ensure `.env` is listed in `.gitignore`
- ✅ ALWAYS run `npm install` after bootstrapping
- ✅ ALWAYS verify compilation and tests pass
- ✅ ALWAYS initialize git after setup
- ✅ ALWAYS use named addresses (my_addr = "\_")

## NEVER Rules

- ❌ NEVER commit `.env` to git (contains private keys!)
- ❌ NEVER push `.env` to GitHub or any remote repository
- ❌ NEVER share your `VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY`
- ❌ NEVER skip Move.toml configuration
- ❌ NEVER use hardcoded addresses in code
- ❌ NEVER skip verifying compilation after scaffolding
- ❌ NEVER read `.env` files after creation — to verify existence, use `ls -la .env` not `cat .env`
- ❌ NEVER display `VITE_MODULE_PUBLISHER_ACCOUNT_PRIVATE_KEY` values in responses
- ❌ NEVER run `git add .` or `git add -A` until `.gitignore` contains `.env` — always verify first

---

## Template Sources

| Template      | GitHub URL                                                                                        |
| ------------- | ------------------------------------------------------------------------------------------------- |
| Fullstack     | https://github.com/aptos-labs/create-aptos-dapp/tree/main/templates/boilerplate-template          |
| Contract-only | https://github.com/aptos-labs/create-aptos-dapp/tree/main/templates/contract-boilerplate-template |

---

## References

**Official Documentation:**

- CLI Reference: https://aptos.dev/build/cli
- Move.toml: https://aptos.dev/build/cli/working-with-move-contracts
- TypeScript SDK: https://aptos.dev/sdks/ts-sdk

**Related Skills:**

- `write-contracts` - Write Move modules after scaffolding
- `generate-tests` - Create test suite
- `connect-contract-to-frontend` - Wire up frontend to contracts
- `integrate-wallet-adapter` - Add wallet connection

---

**Remember:** Use `degit` for clean bootstrapping. The boilerplate template provides the best starting point for custom
dApps.
