# Contributing to Aptos Move Agent Skills

Thank you for your interest in improving the Aptos Move agent skills! This guide explains how to contribute.

## Overview

This repository provides specialized skills for AI assistants to build secure Aptos Move V2 smart contracts. We use an
**iterative testing workflow** to continuously improve these skills.

## Iterative Skills Improvement Workflow

We maintain a separate testing repository that systematically generates test dApps, validates them, and uses findings to
improve the skills.

### Testing Repository

Location: `../aptos-move-agent-skills-testing/`

The testing workflow:

1. **Generate** - Create 3-5 contract-only dApps using `create-aptos-dapp`
2. **Validate** - Run automated checks (compile/test/coverage)
3. **Review** - Perform manual security review against SECURITY.md
4. **Report** - Create findings report categorized by skill
5. **Improve** - Update skills based on findings
6. **Verify** - Re-test to confirm improvements
7. **Repeat** - Next iteration with different dApp types

### How to Run an Iteration

See the [Testing Repository README](../aptos-move-agent-skills-testing/README.md) for detailed instructions.

**Quick start:**

```bash
cd ../aptos-move-agent-skills-testing

# Generate dApps using Claude Code
# (Copy prompt from scripts/generate-dapps-prompt.md)

# Validate
./scripts/validate-dapps.sh iterations/001

# Manual security review
./scripts/create-report.sh iterations/001
# Edit iterations/001/findings.md

# Generate report
./scripts/create-report.sh iterations/001
# Edit reports/iteration-001-report.md
```

**Apply improvements:**

```bash
cd ../move-agent-skills
git checkout -b feature/iteration-NNN-improvements

# Update skills/*.md and patterns/*.md based on findings
# Add ALWAYS/NEVER rules, code examples, checklist items

git commit -m "Apply findings from iteration NNN"
# Create PR
```

## Types of Contributions

### 1. Skills Improvements (via Testing Workflow)

**Preferred method:** Use the iterative testing workflow described above.

This systematically identifies gaps and ensures improvements are:

- Based on real failure modes
- Comprehensive (fixing patterns, not one-off issues)
- Prioritized by severity
- Verified through re-testing

### 2. Direct Improvements

If you find specific issues without running the full workflow:

**For skill files (`skills/*/SKILL.md`):**

- Add missing ALWAYS/NEVER rules
- Add code examples
- Update checklists
- Clarify ambiguous guidance

**For pattern files (`patterns/*.md`):**

- Add missing security patterns
- Expand examples
- Clarify best practices
- Add anti-patterns

**For setup files:**

- Improve `setups/AGENTS.md` orchestration
- Add missing integrations
- Update tool references

### 3. New Skills

To add a new skill:

1. Create `skills/<skill-name>/SKILL.md`
2. Follow the template from existing skills
3. Include:
   - Clear purpose statement
   - Step-by-step instructions
   - ALWAYS/NEVER rules
   - Code examples
   - Common pitfalls
4. Update `CLAUDE.md` with skill reference
5. Update `setups/AGENTS.md` if it affects workflow

### 4. Bug Fixes

If you find bugs in:

- Code examples: Fix and explain the issue
- Documentation: Clarify or correct
- Scripts: Test and verify the fix

## Contribution Guidelines

### Quality Standards

**All contributions must:**

- Be based on Aptos Move V2 (not V1)
- Follow security best practices from `SECURITY.md`
- Include clear examples
- Be tested (if applicable)
- Reference official Aptos documentation where relevant

**For code examples:**

- Must compile with latest Aptos CLI
- Must follow patterns from `patterns/*.md`
- Must include inline comments explaining key concepts
- Must demonstrate secure patterns

**For security guidance:**

- Must reference specific attack vectors
- Must show both vulnerable and secure code
- Must explain _why_ the pattern is important
- Must be verifiable through testing

### Commit Message Format

Use clear, descriptive commit messages:

```
<type>: <short summary>

<detailed description>

<optional: reference to testing iteration or issue>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

**Types:**

- `feat:` - New skill or major feature
- `fix:` - Bug fix or correction
- `docs:` - Documentation improvements
- `security:` - Security pattern additions
- `test:` - Testing workflow improvements
- `refactor:` - Restructuring without behavior change

**Example:**

```
security: Add unbounded iteration pattern to write-contracts

- Add ALWAYS rule: Use per-user storage, never unbounded vectors
- Add vulnerable code example with explanation
- Add secure alternative using user-scoped tables
- Update checklist with resource management verification

Based on findings from iteration 001 where 3/5 dApps had
unbounded iterations in listing functions.

See: aptos-move-agent-skills-testing/reports/iteration-001-report.md

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
```

### Pull Request Process

1. **Create a branch:**

   ```bash
   git checkout -b feature/your-improvement
   ```

2. **Make changes:**
   - Update relevant files
   - Add examples if applicable
   - Update references in CLAUDE.md or AGENTS.md if needed

3. **Test (if applicable):**
   - Run `aptos move compile` on code examples
   - Verify links work
   - Check markdown rendering

4. **Commit:**

   ```bash
   git add .
   git commit -m "descriptive message"
   ```

5. **Push and create PR:**

   ```bash
   git push origin feature/your-improvement
   ```

6. **PR description should include:**
   - What was changed and why
   - If from testing workflow: link to iteration report
   - If fixing a bug: describe the issue and solution
   - Any breaking changes or migration needed

7. **Review process:**
   - Maintainers will review for accuracy and completeness
   - May request changes or clarifications
   - Once approved, will be merged to main

## Priority Areas

Current focus areas for improvement:

1. **Security patterns** - Expanding `SECURITY.md` with more examples
2. **Testing patterns** - Improving `TESTING.md` with comprehensive test suites
3. **Digital assets** - Enhancing `DIGITAL_ASSETS.md` with more NFT patterns
4. **DeFi patterns** - Adding DeFi-specific guidance (DEX, lending, staking)
5. **Gas optimization** - Expanding storage and computational efficiency patterns

## Questions?

- **About the skills:** Open an issue in this repository
- **About the testing workflow:** See `../aptos-move-agent-skills-testing/README.md`
- **About Aptos Move:** Check [Aptos Developer Docs](https://aptos.dev)

## Recognition

Contributors will be recognized in:

- Git commit history (use Co-Authored-By for AI assistance)
- Release notes for significant contributions
- Special recognition for systematic improvements via testing workflow

## License

By contributing, you agree that your contributions will be licensed under the same license as this project.

---

**Thank you for helping make Aptos Move development more secure and accessible!**
