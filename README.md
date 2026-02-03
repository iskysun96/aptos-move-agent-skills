# Aptos Agent Skills

**AI-powered skills for building fullstack Aptos dApps - smart contracts and frontend integration**

This repository provides specialized skills and patterns for AI assistants (Claude Code, Cursor, GitHub Copilot, future Aptos Vibe tool) to help developers build secure, well-tested Aptos dApps following best practices.

## Features

- **17+ Specialized Skills** - Context-aware skills for Move development and fullstack dApp building
- **Move Smart Contracts** - Modern Move V2 object model patterns
- **TypeScript SDK** - Frontend integration with @aptos-labs/ts-sdk
- **Wallet Integration** - @aptos-labs/wallet-adapter-react patterns
- **Security-First** - Comprehensive security checklist and audit patterns
- **100% Test Coverage** - Automated test generation with coverage requirements
- **Auto-Activation** - Skills trigger automatically based on developer actions

## Quick Start

### For Claude Code Users

1. Clone this repository into your project or alongside it:

   ```bash
   git clone https://github.com/your-org/aptos-agent-skills.git
   ```

2. The `CLAUDE.md` file will be automatically detected and loaded by Claude Code

3. Start developing - skills activate automatically:
   - "Create a new dApp" → `scaffold-project` skill activates
   - "Write an NFT contract" → `search-aptos-examples` + `write-contracts` activate
   - After writing code → `generate-tests` auto-activates
   - "Connect wallet" → `integrate-wallet-adapter` activates
   - "Call contract from frontend" → `connect-contract-to-frontend` activates

### For Other Editors (Cursor, Copilot)

1. Clone this repository
2. Reference skill files in your prompts:
   ```
   @skills/move/write-contracts/SKILL.md Help me build an NFT marketplace
   ```
3. Include `setups/AGENTS.md` in your workspace context

## Repository Structure

```
aptos-agent-skills/
├── CLAUDE.md                              # Auto-loader for Claude Code
├── README.md
├── package.json
│
├── skills/
│   │
│   ├── project/                           # Project scaffolding & setup
│   │   └── scaffold-project/              # Bootstrap from templates (degit)
│   │
│   ├── move/                              # Move smart contract development
│   │   ├── write-contracts/
│   │   ├── generate-tests/
│   │   ├── security-audit/
│   │   ├── deploy-contracts/
│   │   ├── search-aptos-examples/
│   │   ├── use-aptos-cli/
│   │   ├── troubleshoot-errors/
│   │   ├── analyze-gas-optimization/
│   │   ├── generate-move-scripts/
│   │   └── implement-upgradeable-contracts/
│   │
│   ├── sdk/                               # TypeScript SDK usage
│   │   ├── use-typescript-sdk/            # SDK client & operations
│   │   └── query-onchain-data/            # Reading blockchain state
│   │
│   ├── wallet/                            # Wallet integration
│   │   └── integrate-wallet-adapter/      # Wallet connection & management
│   │
│   ├── frontend/                          # Frontend patterns
│   │   ├── connect-contract-to-frontend/  # Entry & view functions
│   │   └── handle-transactions/           # Transaction UX
│   │
│   └── testing/                           # Testing & QA
│       └── test-fullstack-dapp/           # E2E testing patterns
│
├── patterns/
│   │
│   ├── move/                              # Move reference docs
│   │   ├── OBJECTS.md
│   │   ├── SECURITY.md
│   │   ├── DIGITAL_ASSETS.md
│   │   ├── FUNGIBLE_ASSETS.md
│   │   ├── MOVE_V2_SYNTAX.md
│   │   ├── ADVANCED_TYPES.md
│   │   └── STORAGE_OPTIMIZATION.md
│   │
│   └── fullstack/                         # Fullstack reference docs
│       ├── TYPESCRIPT_SDK.md              # Complete SDK reference
│       ├── WALLET_ADAPTER.md              # Wallet integration patterns
│       └── FRONTEND_PATTERNS.md           # React + Aptos patterns
│
└── setups/
    ├── AGENTS.md                          # Workflow orchestration
    ├── claude-code/README.md
    ├── cursor/README.md
    └── copilot/README.md
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
// Use Aptos Digital Asset standard
use aptos_token_objects::collection;
use aptos_token_objects::token;
use aptos_token_objects::aptos_token::AptosToken;

public entry fun list_nft(
    seller: &signer,
    nft: Object<AptosToken>,  // Digital Asset standard
    price: u64
)
```

### 3. Object-Centric Development

```move
// MODERN (V2): Type-safe objects
public entry fun transfer_item(
    owner: &signer,
    item: Object<Item>,  // Type-safe!
    recipient: address
)
```

### 4. Frontend Integration

```typescript
// Entry function (write)
const payload = {
  function: `${MODULE_ADDRESS}::counter::increment`,
  functionArguments: [],
};
await signAndSubmitTransaction({ data: payload });

// View function (read)
const [count] = await aptos.view({
  payload: {
    function: `${MODULE_ADDRESS}::counter::get_count`,
    functionArguments: [accountAddress],
  },
});
```

## Skills Overview

### Project Scaffolding
- **scaffold-project** - Bootstrap from templates using degit (fullstack or contract-only)

### Move Smart Contracts
- **write-contracts** - Generate secure Move V2 contracts
- **generate-tests** - Create comprehensive test suites (100% coverage required)
- **security-audit** - Security auditing before deployment
- **deploy-contracts** - Deploy to devnet/testnet/mainnet
- **search-aptos-examples** - Find patterns from aptos-core
- **use-aptos-cli** - CLI command reference
- **troubleshoot-errors** - Debug common errors
- **analyze-gas-optimization** - Optimize gas usage
- **generate-move-scripts** - Create atomic scripts
- **implement-upgradeable-contracts** - Contract upgrade patterns

### TypeScript SDK
- **use-typescript-sdk** - @aptos-labs/ts-sdk patterns
- **query-onchain-data** - View functions, resource queries, events

### Wallet Integration
- **integrate-wallet-adapter** - WalletProvider, useWallet, connection states

### Frontend Development
- **connect-contract-to-frontend** - Entry functions, view functions, type encoding
- **handle-transactions** - Transaction UX, loading states, error handling

### Testing
- **test-fullstack-dapp** - E2E testing patterns

## Example Workflows

### Workflow: Build Fullstack dApp

1. `scaffold-project` → Bootstrap fullstack template
2. `write-contracts` → Write Move modules
3. `generate-tests` → Create Move tests
4. `security-audit` → Audit before deployment
5. `deploy-contracts` → Deploy to devnet
6. `connect-contract-to-frontend` → Wire up entry/view functions
7. `integrate-wallet-adapter` → Configure wallet connection
8. `handle-transactions` → Polish transaction UX
9. `test-fullstack-dapp` → E2E testing

### Workflow: Contract Only

1. `scaffold-project` → Bootstrap contract-only template
2. `write-contracts` → Write Move modules
3. `generate-tests` → Create Move tests
4. `security-audit` → Audit before deployment
5. `deploy-contracts` → Deploy to network

## Formatting

This project uses Prettier for consistent markdown formatting:

```bash
# Format all markdown files
npm run format

# Check formatting without making changes
npm run format:check
```

## Resources

### Official Aptos Documentation

- **Object Model:** https://aptos.dev/build/smart-contracts/object
- **Security Guidelines:** https://aptos.dev/build/smart-contracts/move-security-guidelines
- **Move Book:** https://aptos.dev/build/smart-contracts/book
- **TypeScript SDK:** https://aptos.dev/sdks/ts-sdk
- **Wallet Adapter:** https://aptos.dev/sdks/wallet-adapter

### Template Sources

- **Fullstack Template:** https://github.com/aptos-labs/create-aptos-dapp/tree/main/templates/boilerplate-template
- **Contract-only Template:** https://github.com/aptos-labs/create-aptos-dapp/tree/main/templates/contract-boilerplate-template

## Contributing

We welcome contributions! Areas to contribute:

1. **Skills** - Add new skills or improve existing ones
2. **Patterns** - Enhance pattern documentation
3. **Examples** - Add working examples
4. **Bug Fixes** - Fix issues in skill logic or examples

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Roadmap

- [x] Core Move skills (11 skills)
- [x] Repository restructure for fullstack support
- [ ] TypeScript SDK skills
- [ ] Wallet integration skills
- [ ] Frontend integration skills
- [ ] E2E testing skills
- [ ] Example projects

---

**Built for secure, modern Aptos dApp development with AI assistance.**
