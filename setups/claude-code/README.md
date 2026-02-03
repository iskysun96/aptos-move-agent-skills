# Claude Code Integration Guide

This guide explains how to use Aptos Move V2 Agent Skills with Claude Code.

## Automatic Integration

Claude Code automatically detects and loads the `CLAUDE.md` file in your repository, which provides access to all skills
and patterns.

**No configuration needed!**

## How It Works

### 1. Skills Activate Automatically

When you work with Claude Code:

- **Creating projects:** "Create a new Move project" → `scaffold-project` skill activates
- **Writing contracts:** "Build an NFT marketplace" → `search-aptos-examples` + `write-contracts` activate
- **After writing code:** `generate-tests` auto-activates to create tests
- **Before deployment:** "Deploy this" → `security-audit` + `deploy-contracts` activate
- **On errors:** `troubleshoot-errors` auto-activates

### 2. Context-Aware Assistance

Claude Code understands the full context:

- Current files in your workspace
- Recent changes you've made
- Error messages and compilation output
- Your coding patterns and preferences

### 3. Multi-Turn Workflows

Claude Code maintains context across multiple interactions:

```
You: "Help me build an NFT collection contract"

Claude: [Searches examples] → [Generates contract] → [Creates tests] → [Runs coverage]

You: "Add a marketplace feature"

Claude: [Adapts existing contract] → [Updates tests] → [Verifies coverage]

You: "Deploy to testnet"

Claude: [Runs security audit] → [Guides deployment] → [Verifies deployment]
```

## Usage Examples

### Example 1: New NFT Project

```
You: "Create a new NFT collection project called cyber-dragons"

Claude Code:
1. ✅ Runs `aptos move init --name cyber-dragons`
2. ✅ Configures Move.toml with proper dependencies
3. ✅ Creates directory structure
4. ✅ Generates initial module template
5. ✅ Creates test template
6. ✅ Initializes git repository

Ready to start coding!
```

### Example 2: Writing Contracts

```
You: "Write a smart contract for NFT minting with 10,000 max supply"

Claude Code:
1. ✅ Searches aptos-core examples for NFT patterns
2. ✅ Generates contract with:
   - Object<NFT> references (not addresses)
   - Proper constructor pattern
   - Ownership verification
   - Supply limit validation
   - Input validation
   - Modern V2 syntax
3. ✅ Auto-generates comprehensive tests
4. ✅ Runs tests and verifies 100% coverage

All done! Your contract is secure and well-tested.
```

### Example 3: Security Review

```
You: "Review this contract for security issues"

Claude Code:
1. ✅ Runs full security checklist
2. ✅ Identifies issues:
   - Missing input validation in line 45
   - No overflow check in line 67
3. ✅ Provides fixes with explanations
4. ✅ Updates tests to cover edge cases
5. ✅ Verifies 100% coverage

Security audit complete. Ready for deployment.
```

## Available Skills

| Skill                   | Activates When                    | What It Does                          |
| ----------------------- | --------------------------------- | ------------------------------------- |
| `scaffold-project`      | "create project", "new move app"  | Initializes Move project              |
| `write-contracts`       | "write contract", "build module"  | Generates secure Move code            |
| `generate-tests`        | After writing code, "add tests"   | Creates test suite with 100% coverage |
| `security-audit`        | Before deployment, "audit"        | Runs security checklist               |
| `search-aptos-examples` | Before writing, "find example"    | Searches official examples            |
| `use-aptos-cli`         | "how to compile", "aptos command" | CLI reference                         |
| `deploy-contracts`      | "deploy", "publish"               | Guides deployment workflow            |
| `troubleshoot-errors`   | When errors occur                 | Diagnoses and fixes errors            |

## Pattern References

Claude Code has access to comprehensive pattern documentation:

- **Object Patterns** (`patterns/move/OBJECTS.md`) - How to use Object<T> correctly
- **Security Patterns** (`patterns/move/SECURITY.md`) - Security checklist and examples
- **Testing Patterns** (`patterns/move/SECURITY.md`) - Test generation with 100% coverage
- **Modern Syntax** (`patterns/move/MOVE_V2_SYNTAX.md`) - Inline functions, lambdas, V2 features

## Best Practices

### 1. Be Specific in Requests

**Good:**

```
"Create an NFT marketplace with:
- Fixed price listings
- Offer system
- 2.5% platform fee
- Admin controls"
```

**Less Effective:**

```
"Build a marketplace"
```

### 2. Iterate Incrementally

```
Step 1: "Create basic NFT contract"
Step 2: "Add transfer restrictions"
Step 3: "Add marketplace integration"
Step 4: "Add royalty system"
```

### 3. Ask for Explanations

```
"Why did you use Object<Item> instead of address?"
"Explain the security checks in this function"
"Why is 100% test coverage important?"
```

### 4. Request Reviews

```
"Review this code for security issues"
"Check if this follows best practices"
"Verify the object pattern is correct"
```

## Workflows

### Complete Project Workflow

```
1. Initialize:
   "Create new Move project called defi-protocol"

2. Implement:
   "Build a token swap contract with constant product AMM"

3. Test:
   "Add comprehensive tests" (auto-activates)
   "Verify 100% coverage"

4. Audit:
   "Run security audit"
   "Fix any issues found"

5. Deploy:
   "Deploy to testnet"
   [Test on testnet]
   "Deploy to mainnet"

6. Document:
   "Generate documentation for this module"
```

### Quick Fix Workflow

```
1. Error occurs:
   [Claude Code detects compilation error]

2. Diagnose:
   "What's wrong with this code?"
   [troubleshoot-errors activates automatically]

3. Fix:
   [Claude provides fix with explanation]

4. Verify:
   "Run tests to confirm fix"
```

## Keyboard Shortcuts

Claude Code supports various shortcuts (check your editor for specifics):

- **Open Claude:** Usually `Cmd/Ctrl + K` or sidebar icon
- **Submit prompt:** `Enter` (or `Shift + Enter` for new line)
- **Cancel:** `Escape`
- **Copy code:** Click copy button on code blocks

## Tips & Tricks

### Tip 1: Reference Specific Files

```
"Update the NFT module in sources/nft.move to add burn functionality"
```

### Tip 2: Ask for Comparisons

```
"What's the difference between resource accounts and named objects?"
"Should I use Object<T> or address here?"
```

### Tip 3: Request Tests for Specific Scenarios

```
"Add tests for:
- Unauthorized access
- Zero amount transfers
- Max supply reached
- Empty string names"
```

### Tip 4: Get CLI Commands

```
"What's the command to test with coverage?"
"How do I deploy to testnet?"
```

### Tip 5: Verify Against Patterns

```
"Does this follow the object creation pattern from OBJECTS.md?"
"Check if security checklist items are satisfied"
```

## Troubleshooting

### Skills Not Activating

**Issue:** Skills don't seem to activate automatically

**Solution:**

1. Ensure `CLAUDE.md` exists in repository root
2. Restart Claude Code
3. Try explicit skill request: "Use write-contracts skill to..."

### Outdated Patterns

**Issue:** Code generated doesn't match latest Aptos docs

**Solution:**

1. Reference official docs: "Follow pattern from aptos.dev/build/smart-contracts/object"
2. Pull latest repository updates
3. Report issue if pattern is incorrect

### Coverage Not 100%

**Issue:** Tests don't achieve 100% coverage

**Solution:**

1. Ask: "Show me uncovered lines"
2. Request: "Add tests for uncovered paths"
3. Verify: "Run coverage report"

## Getting Help

- **In-chat:** Ask Claude Code directly: "How do I use the write-contracts skill?"
- **Documentation:** Check `setups/AGENTS.md` for complete guide
- **Patterns:** Review `patterns/` directory for detailed examples
- **Issues:** Report problems on GitHub repository

## Advanced Usage

### Custom Workflows

You can combine skills for custom workflows:

```
"Follow this workflow:
1. Search for similar examples
2. Generate contract with X features
3. Create comprehensive tests
4. Run security audit
5. Prepare for testnet deployment"
```

### Pattern Customization

Request modifications to patterns:

```
"Use the standard object creation pattern but add:
- Custom metadata fields
- Event emissions
- Extended reference storage"
```

### Integration with CI/CD

```
"Generate a GitHub Actions workflow that:
- Runs tests on every push
- Verifies 100% coverage
- Runs security checks
- Deploys to testnet on main branch"
```

## Updates

This integration works with the current version of Claude Code. Check the repository for updates to skills and patterns.

**Repository:** https://github.com/your-org/move-agent-skills

---

**Happy coding with Claude Code and Aptos Move V2!**
