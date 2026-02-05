# Installation Guide

## Quick Install (All Skills)

```bash
npx skills add iskysun96/aptos-agent-skills
```

## Selective Installation

### Core Move Skills (Recommended)

```bash
npx skills add iskysun96/aptos-agent-skills \
  --skill write-contracts \
  --skill generate-tests \
  --skill security-audit \
  --skill deploy-contracts
```

### Full Move Development

```bash
npx skills add iskysun96/aptos-agent-skills \
  --skill scaffold-project \
  --skill write-contracts \
  --skill generate-tests \
  --skill security-audit \
  --skill deploy-contracts \
  --skill search-aptos-examples \
  --skill troubleshoot-errors
```

### All Move Skills

```bash
npx skills add iskysun96/aptos-agent-skills \
  --skill scaffold-project \
  --skill write-contracts \
  --skill generate-tests \
  --skill security-audit \
  --skill deploy-contracts \
  --skill search-aptos-examples \
  --skill use-aptos-cli \
  --skill troubleshoot-errors \
  --skill analyze-gas-optimization \
  --skill generate-move-scripts \
  --skill implement-upgradeable-contracts
```

## Agent-Specific Installation

```bash
# For Claude Code
npx skills add iskysun96/aptos-agent-skills -a claude-code

# For Cursor
npx skills add iskysun96/aptos-agent-skills -a cursor

# For GitHub Copilot
npx skills add iskysun96/aptos-agent-skills -a copilot
```

## Claude Code Plugin

```bash
/plugin marketplace add iskysun96/aptos-agent-skills
```

## Available Skills

| Skill                             | Category | Description                             |
| --------------------------------- | -------- | --------------------------------------- |
| `scaffold-project`                | project  | Bootstrap Aptos dApp from templates     |
| `write-contracts`                 | move     | Generate secure Move V2 smart contracts |
| `generate-tests`                  | move     | Create comprehensive test suites        |
| `security-audit`                  | move     | Audit contracts before deployment       |
| `deploy-contracts`                | move     | Deploy to devnet/testnet/mainnet        |
| `search-aptos-examples`           | move     | Find patterns from aptos-core           |
| `use-aptos-cli`                   | move     | CLI command reference                   |
| `troubleshoot-errors`             | move     | Debug common errors                     |
| `analyze-gas-optimization`        | move     | Optimize gas usage                      |
| `generate-move-scripts`           | move     | Create atomic transaction scripts       |
| `implement-upgradeable-contracts` | move     | Contract upgrade patterns               |

## Verifying Installation

After installation, verify the skills are available:

```bash
# Check installed skills
ls ~/.claude/skills/

# Or for Cursor
ls ~/.cursor/skills/
```

## Uninstalling

```bash
npx skills remove iskysun96/aptos-agent-skills
```

## Manual Installation

If you prefer manual installation:

1. Clone the repository:

   ```bash
   git clone https://github.com/iskysun96/aptos-agent-skills.git
   ```

2. Copy to your agent's skills directory:

   ```bash
   # For Claude Code
   cp -r aptos-agent-skills/skills/* ~/.claude/skills/
   cp -r aptos-agent-skills/patterns/* ~/.claude/patterns/
   ```

3. Reference in your project by including `AGENTS.md` in your workspace context.
