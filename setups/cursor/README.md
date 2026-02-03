# Cursor Integration Guide

This guide explains how to use Aptos Move V2 Agent Skills with Cursor AI.

## Setup

### Step 1: Clone Repository

Clone this repository into your workspace:

```bash
cd your-move-project
git clone https://github.com/your-org/move-agent-skills.git
```

Or clone it alongside your project:

```bash
cd parent-directory
git clone https://github.com/your-org/move-agent-skills.git
```

### Step 2: Configure Cursor Rules

Create or update `.cursorrules` in your project root:

```
# Aptos Move V2 Development Rules

When working with Aptos Move contracts:

## Core Principles
1. Always search aptos-core examples before writing new contracts
2. Use Object<T> references, never raw addresses (unless legacy integration)
3. Verify signer authority in all entry functions
4. Validate all inputs (amounts, addresses, strings, vectors)
5. Achieve 100% test coverage before deployment

## Reference Documentation
- Main guide: @move-agent-skills/setups/AGENTS.md
- Object patterns: @move-agent-skills/patterns/move/OBJECTS.md
- Security checklist: @move-agent-skills/patterns/move/SECURITY.md
- Testing guide: @move-agent-skills/patterns/move/SECURITY.md
- Modern syntax: @move-agent-skills/patterns/move/MOVE_V2_SYNTAX.md

## Skills
Use these skills explicitly when needed:
- @move-agent-skills/skills/move/write-contracts/SKILL.md - Generate contracts
- @move-agent-skills/skills/move/generate-tests/SKILL.md - Create tests
- @move-agent-skills/skills/move/security-audit/SKILL.md - Audit security
- @move-agent-skills/skills/project/scaffold-project/SKILL.md - Init projects
- @move-agent-skills/skills/move/search-aptos-examples/SKILL.md - Find examples
- @move-agent-skills/skills/move/deploy-contracts/SKILL.md - Deploy guide
- @move-agent-skills/skills/move/troubleshoot-errors/SKILL.md - Debug errors

## ALWAYS Rules
- Use Object<T> for all object references
- Verify signer authority: assert!(signer::address_of(user) == expected, E_UNAUTHORIZED)
- Verify object ownership: assert!(object::owner(obj) == signer::address_of(user), E_NOT_OWNER)
- Validate inputs: assert!(amount > 0, E_ZERO_AMOUNT)
- Generate all refs in constructor before ConstructorRef destroyed
- Return Object<T>, never ConstructorRef

## NEVER Rules
- Never use raw addresses for objects (use Object<T>)
- Never return ConstructorRef from functions
- Never expose &mut in public functions
- Never skip signer verification
- Never deploy without 100% test coverage
- Never use resource accounts (legacy pattern)
```

### Step 3: Use @ References

In Cursor, you can reference files using `@` syntax:

```
@move-agent-skills/patterns/move/OBJECTS.md Show me how to create objects correctly
```

## Usage

### Explicit Skill References

Reference skills explicitly in your prompts:

```
@move-agent-skills/skills/move/write-contracts/SKILL.md
Help me build an NFT marketplace with these features:
- Fixed price listings
- Offer system
- Platform fees
```

### Pattern References

Reference patterns for specific guidance:

```
@move-agent-skills/patterns/move/SECURITY.md
Review this contract for security issues
```

### Multiple References

Combine multiple references:

```
@move-agent-skills/patterns/move/OBJECTS.md
@move-agent-skills/patterns/move/SECURITY.md
Write a secure NFT contract using proper object patterns
```

## Workflows

### Workflow 1: New Contract

```
1. Search examples:
   @move-agent-skills/skills/move/search-aptos-examples/SKILL.md
   Find examples of token swap contracts

2. Write contract:
   @move-agent-skills/skills/move/write-contracts/SKILL.md
   Generate a token swap contract with constant product formula

3. Generate tests:
   @move-agent-skills/skills/move/generate-tests/SKILL.md
   Create comprehensive tests for the swap contract

4. Security audit:
   @move-agent-skills/skills/move/security-audit/SKILL.md
   Audit the swap contract for security issues
```

### Workflow 2: Quick Fix

```
1. Diagnose:
   @move-agent-skills/skills/move/troubleshoot-errors/SKILL.md
   What's wrong with this compilation error? [paste error]

2. Fix:
   [Apply suggested fix]

3. Verify:
   Run tests to confirm fix works
```

### Workflow 3: Deployment

```
1. Pre-deployment check:
   @move-agent-skills/skills/move/security-audit/SKILL.md
   Run full security checklist before deployment

2. Deploy:
   @move-agent-skills/skills/move/deploy-contracts/SKILL.md
   Guide me through testnet deployment

3. Verify:
   Help me test the deployed contract on testnet
```

## Cursor-Specific Features

### Composer Mode

Use Composer for multi-file operations:

1. Open Composer (`Cmd/Ctrl + I`)
2. Reference multiple files:

   ```
   @move-agent-skills/skills/move/write-contracts/SKILL.md
   @sources/nft.move
   @tests/nft_tests.move

   Add burn functionality to the NFT contract and update tests
   ```

### Inline Chat

Use inline chat (`Cmd/Ctrl + K`) for quick edits:

```
@move-agent-skills/patterns/move/SECURITY.md
Add input validation to this function
```

### CMD+K Fix Mode

When you see an error:

1. Select the error
2. Press `Cmd/Ctrl + K`
3. Type: `@move-agent-skills/skills/move/troubleshoot-errors/SKILL.md fix this`

## Best Practices

### 1. Start with Main Guide

Begin with the main orchestration file:

```
@move-agent-skills/setups/AGENTS.md
I'm building [describe project]. What's the recommended workflow?
```

### 2. Reference Patterns for Details

For implementation details, reference pattern files:

```
@move-agent-skills/patterns/move/OBJECTS.md
How should I implement object creation for my NFT?
```

### 3. Use Skills for Complete Tasks

For complete tasks, reference skill files:

```
@move-agent-skills/skills/move/generate-tests/SKILL.md
Generate comprehensive tests for sources/marketplace.move
```

### 4. Combine with Context

Combine skill references with your code:

```
@move-agent-skills/skills/move/write-contracts/SKILL.md
@sources/token.move

Refactor this contract to use modern Object<T> patterns
```

## Tips & Tricks

### Tip 1: Save Common References

Create shortcuts for common patterns in your `.cursorrules`:

```
# Quick references
@objects = @move-agent-skills/patterns/move/OBJECTS.md
@security = @move-agent-skills/patterns/move/SECURITY.md
@testing = @move-agent-skills/patterns/move/SECURITY.md
```

### Tip 2: Use Tab Autocomplete

Type `@move-agent-skills/` and use tab completion to explore available files.

### Tip 3: Multi-Step Conversations

Cursor maintains context across messages, so you can have multi-step conversations:

```
You: @write-contracts Generate NFT contract
Cursor: [Generates contract]

You: Now add marketplace integration
Cursor: [Adds marketplace features]

You: Generate tests for everything
Cursor: [Generates comprehensive tests]
```

### Tip 4: Code Actions

Right-click code and use Cursor's code actions:

- "Ask Cursor" with `@move-agent-skills/` reference
- "Edit with Cursor" for refactoring
- "Fix with Cursor" for errors

## Keyboard Shortcuts

- **Composer:** `Cmd/Ctrl + I`
- **Inline Chat:** `Cmd/Ctrl + K`
- **Chat Panel:** `Cmd/Ctrl + L`
- **Accept Suggestion:** `Tab`
- **Reject Suggestion:** `Esc`

## Example Sessions

### Session 1: NFT Collection

```
You: @move-agent-skills/setups/AGENTS.md
     I want to create an NFT collection. What's the workflow?

Cursor: [Explains workflow]

You: @move-agent-skills/skills/project/scaffold-project/SKILL.md
     Create project structure

Cursor: [Creates project]

You: @move-agent-skills/skills/move/write-contracts/SKILL.md
     Generate NFT collection contract with 10k max supply

Cursor: [Generates contract]

You: @move-agent-skills/skills/move/generate-tests/SKILL.md
     Create tests

Cursor: [Creates tests, verifies 100% coverage]
```

### Session 2: Security Review

```
You: @move-agent-skills/skills/move/security-audit/SKILL.md
     @sources/marketplace.move
     Review this contract

Cursor: [Runs security checklist]
        Found 2 issues:
        1. Missing input validation in list_item (line 45)
        2. No overflow check in calculate_fee (line 89)

You: Fix these issues

Cursor: [Applies fixes, updates tests]

You: Re-run security audit

Cursor: [Verifies all checks pass]
```

## Troubleshooting

### @ References Not Working

**Issue:** `@move-agent-skills/...` not resolving

**Solution:**

1. Ensure repository is cloned in accessible location
2. Try absolute path: `@/path/to/move-agent-skills/...`
3. Restart Cursor

### Context Too Large

**Issue:** "Context too large" error with many references

**Solution:**

1. Reference specific skill files, not entire repository
2. Use smaller, focused references
3. Break into multiple conversations

### Skills Not Applied

**Issue:** Cursor doesn't follow skill patterns

**Solution:**

1. Be more explicit: "Follow the pattern in @move-agent-skills/patterns/move/OBJECTS.md exactly"
2. Include relevant code context
3. Reference `.cursorrules` to ensure it's configured

## Getting Help

- **Documentation:** Check `@move-agent-skills/setups/AGENTS.md`
- **Patterns:** Review `@move-agent-skills/patterns/` directory
- **Issues:** Report on GitHub repository
- **Cursor Docs:** https://cursor.com/docs

## Updates

Keep the repository updated:

```bash
cd move-agent-skills
git pull origin main
```

---

**Happy coding with Cursor and Aptos Move V2!**
