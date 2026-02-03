# Aptos Agent Skills

This repository provides specialized skills for AI assistants to build secure, modern Aptos dApps - both Move smart contracts and fullstack applications.

**Main AI Instructions:** See [`setups/AGENTS.md`](setups/AGENTS.md) for complete guidance.

## Quick Links

### Move Development
- **[AI Assistant Guide](setups/AGENTS.md)** - Main orchestration file with workflows
- **[Digital Assets](patterns/move/DIGITAL_ASSETS.md)** - NFT standard (CRITICAL for NFTs)
- **[Fungible Assets](patterns/move/FUNGIBLE_ASSETS.md)** - Token standard (CRITICAL for tokens/coins)
- **[Object Patterns](patterns/move/OBJECTS.md)** - Object model reference
- **[Security Guide](patterns/move/SECURITY.md)** - Security checklist
- **[Modern Syntax](patterns/move/MOVE_V2_SYNTAX.md)** - V2 syntax guide
- **[Advanced Types](patterns/move/ADVANCED_TYPES.md)** - Advanced type patterns
- **[Storage Optimization](patterns/move/STORAGE_OPTIMIZATION.md)** - Storage cost reduction

### Fullstack Development
- **[TypeScript SDK](patterns/fullstack/TYPESCRIPT_SDK.md)** - SDK client & operations
- **[Wallet Adapter](patterns/fullstack/WALLET_ADAPTER.md)** - Wallet integration patterns
- **[Frontend Patterns](patterns/fullstack/FRONTEND_PATTERNS.md)** - React + Aptos patterns

## Skills

### Project Scaffolding
- **[scaffold-project](skills/project/scaffold-project/SKILL.md)** - Bootstrap from templates (degit)

### Move Smart Contracts
- **[write-contracts](skills/move/write-contracts/SKILL.md)** - Generate secure Move contracts
- **[generate-tests](skills/move/generate-tests/SKILL.md)** - Create comprehensive test suites
- **[security-audit](skills/move/security-audit/SKILL.md)** - Audit contracts before deployment
- **[deploy-contracts](skills/move/deploy-contracts/SKILL.md)** - Deploy to networks
- **[search-aptos-examples](skills/move/search-aptos-examples/SKILL.md)** - Find example patterns
- **[use-aptos-cli](skills/move/use-aptos-cli/SKILL.md)** - CLI command reference
- **[troubleshoot-errors](skills/move/troubleshoot-errors/SKILL.md)** - Debug common errors
- **[analyze-gas-optimization](skills/move/analyze-gas-optimization/SKILL.md)** - Optimize gas usage
- **[generate-move-scripts](skills/move/generate-move-scripts/SKILL.md)** - Create atomic scripts
- **[implement-upgradeable-contracts](skills/move/implement-upgradeable-contracts/SKILL.md)** - Contract upgrades

### TypeScript SDK
- **[use-typescript-sdk](skills/sdk/use-typescript-sdk/SKILL.md)** - SDK client & operations
- **[query-onchain-data](skills/sdk/query-onchain-data/SKILL.md)** - Reading blockchain state

### Wallet Integration
- **[integrate-wallet-adapter](skills/wallet/integrate-wallet-adapter/SKILL.md)** - Wallet connection & management

### Frontend Development
- **[connect-contract-to-frontend](skills/frontend/connect-contract-to-frontend/SKILL.md)** - Entry & view functions
- **[handle-transactions](skills/frontend/handle-transactions/SKILL.md)** - Transaction UX

### Testing
- **[test-fullstack-dapp](skills/testing/test-fullstack-dapp/SKILL.md)** - E2E testing patterns

## Core Principles

1. **Search first** - Check aptos-core examples before writing
2. **Use Digital Asset standard** - For ALL NFT contracts (use Object<AptosToken>)
3. **Use Fungible Asset standard** - For ALL token/coin contracts (use Object<Metadata>)
4. **Use objects** - Always use `Object<T>` references (never addresses)
5. **Security first** - Verify signers, validate inputs, protect references
6. **Test everything** - 100% coverage required
7. **Modern syntax** - Use inline functions, lambdas, V2 patterns
8. **Format with prettier** - Always run `npx prettier --write <files>` before committing

## Git Workflow Best Practices

**ALWAYS follow these steps when starting work:**

```bash
# 1. Pull latest changes from main BEFORE starting work
git checkout main
git pull origin main

# 2. Create feature branch from updated main
git checkout -b feature/your-feature-name

# 3. If using worktrees, run npm install to get all dependencies
npm install

# 4. Do your work...

# 5. Before committing, ensure you're up to date
git fetch origin main
git rebase origin/main  # Or merge if preferred
```

**Key principles:**

1. **Always pull from main first** - Prevents merge conflicts and ensures you have latest changes
2. **Run npm install in new worktrees** - Ensures all dependencies (prettier, etc.) are available
3. **Keep feature branches up to date** - Regularly sync with main to avoid large merge conflicts
4. **Use descriptive branch names** - `feature/`, `fix/`, `docs/` prefixes

## Code Formatting

**ALWAYS format markdown files with prettier before committing:**

```bash
# Format specific files
npx prettier --write CONTRIBUTING.md skills/**/*.md patterns/**/*.md

# Format all markdown files
npx prettier --write "**/*.md"

# Check formatting
npx prettier --check "**/*.md"
```

**Prettier configuration** (`.prettierrc`):

- `printWidth: 120` - Max line length
- `proseWrap: always` - Wrap markdown prose
- `tabWidth: 2` - 2 space indentation

This ensures consistent formatting across all documentation files.

## Integration

**Claude Code:** This file is automatically loaded when detected in the repository.

**Other Editors:** Include `setups/AGENTS.md` in your workspace context.

---

For detailed instructions, see [`setups/AGENTS.md`](setups/AGENTS.md).
