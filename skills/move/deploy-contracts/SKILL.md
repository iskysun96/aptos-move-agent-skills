---
name: deploy-contracts
description: "Safely deploys Move contracts to Aptos networks (devnet, testnet, mainnet) with pre-deployment verification. Triggers on: 'deploy contract', 'publish to testnet', 'deploy to mainnet', 'how to deploy', 'publish module', 'deployment checklist', 'deploy to devnet'."
---

# Deploy Contracts Skill

## Overview

This skill guides safe deployment of Move contracts to Aptos networks. **Always deploy to testnet before mainnet.**

## Pre-Deployment Checklist

Before deploying, verify ALL items:

### Security Audit ⭐ CRITICAL - See [SECURITY.md](../../../patterns/move/SECURITY.md)
- [ ] Security audit completed (use `security-audit` skill)
- [ ] All critical vulnerabilities fixed
- [ ] All security patterns verified (arithmetic safety, storage scoping, reference safety, business logic)
- [ ] Access control verified (signer checks, object ownership)
- [ ] Input validation implemented (minimum thresholds, fee validation)
- [ ] No unbounded iterations (per-user storage, not global vectors)
- [ ] Atomic operations (no front-running opportunities)
- [ ] Randomness security (if applicable - entry functions, gas balanced)

### Testing
- [ ] 100% test coverage achieved: `aptos move test --coverage`
- [ ] All tests passing: `aptos move test`
- [ ] Coverage report shows 100.0%
- [ ] Edge cases tested

### Code Quality
- [ ] Code compiles without errors: `aptos move compile`
- [ ] No hardcoded addresses (use named addresses)
- [ ] Error codes clearly defined
- [ ] Functions properly documented

### Configuration
- [ ] Move.toml configured correctly
- [ ] Named addresses set up: `my_addr = "_"`
- [ ] Dependencies specified with correct versions
- [ ] Network URLs configured

## Object Deployment (Modern Pattern)

### CRITICAL: Use Correct Deployment Command

There are TWO ways to deploy contracts. For modern object-based contracts, use `deploy-object`:

**✅ CORRECT: Object Deployment (Modern Pattern)**
```bash
aptos move deploy-object \
    --address-name my_addr \
    --profile devnet \
    --assume-yes
```

**What this does:**
1. Creates an object to host your contract code
2. Deploys the package to that object's address
3. Returns the object address (deterministic, based on deployer + package name)
4. Object address becomes your contract address

**❌ WRONG: Using Regular Publish for Object Contracts**
```bash
# ❌ Don't use this for object-based contracts
aptos move publish \
    --named-addresses my_addr=<address>
```

**When to use each:**
- `deploy-object`: Modern contracts using objects (RECOMMENDED)
- `publish`: Legacy account-based deployment (older pattern)

**How to tell if you need object deployment:**
- Your contract creates named objects in `init_module`
- Your contract uses `object::create_named_object()`
- You want a deterministic contract address
- Documentation says "deploy as object"

### Alternative Object Deployment Commands

**Option 1: `deploy-object` (Recommended - Simplest)**
```bash
aptos move deploy-object --address-name my_addr --profile devnet
```
- Automatically creates object and deploys code
- Object address is deterministic
- Best for most use cases

**Option 2: `create-object-and-publish-package` (Advanced)**
```bash
aptos move create-object-and-publish-package \
    --address-name my_addr \
    --named-addresses my_addr=default
```
- More complex command with more options
- Use only if you need specific object configuration
- Generally not needed

**Recommendation:** Always use `deploy-object` unless you have a specific reason to use the alternative.

## Deployment Workflow

### Step 1: Test Locally

```bash
# Ensure all tests pass
aptos move test

# Verify 100% coverage
aptos move test --coverage
aptos move coverage summary
# Expected output: "coverage: 100.0%"
```

### Step 2: Compile

```bash
# Compile contract
aptos move compile --named-addresses my_addr=<your_address>

# Verify compilation succeeds
echo $?
# Expected: 0 (success)
```

### Step 3: Deploy to Devnet (Optional)

**Devnet is for quick testing and experimentation.**

```bash
# Initialize devnet account (if not already)
aptos init --network devnet --profile devnet

# Fund account
aptos account fund-with-faucet --account default --profile devnet

# Deploy as object (modern pattern)
aptos move deploy-object \
    --address-name my_addr \
    --profile devnet \
    --assume-yes

# Save the object address from output for future upgrades
# Output: "Code was successfully deployed to object address 0x..."

# Verify deployment
aptos account list --account <devnet_address> --profile devnet
```

### Step 4: Deploy to Testnet (REQUIRED)

**Testnet is for final testing before mainnet.**

```bash
# Initialize testnet account
aptos init --network testnet --profile testnet

# Fund account
aptos account fund-with-faucet --account default --profile testnet

# Deploy to testnet as object (modern pattern)
aptos move deploy-object \
    --address-name my_addr \
    --profile testnet \
    --assume-yes

# IMPORTANT: Save the object address from output
# You'll need it for upgrades and function calls
# Output: "Code was successfully deployed to object address 0x..."

# Expected output:
# {
#   "Result": {
#     "transaction_hash": "0x...",
#     "gas_used": 1234,
#     "success": true,
#     "vm_status": "Executed successfully"
#   }
# }
```

### Step 5: Test on Testnet

```bash
# Run entry functions to verify deployment
aptos move run \
    --profile testnet \
    --function-id <testnet_address>::<module>::<function> \
    --args ...

# Test multiple scenarios
# - Happy paths
# - Error cases (should abort with correct error codes)
# - Access control
# - Edge cases

# Verify using explorer
# https://explorer.aptoslabs.com/?network=testnet
```

### Step 6: Deploy to Mainnet (After Testnet Success)

**Only deploy to mainnet after thorough testnet testing.**

```bash
# Initialize mainnet account
aptos init --network mainnet --profile mainnet

# IMPORTANT: Backup your private key securely!
# The private key is in ~/.aptos/config.yaml

# Deploy to mainnet as object (modern pattern)
aptos move deploy-object \
    --address-name my_addr \
    --profile mainnet \
    --max-gas 20000  # Optional: set gas limit

# Review prompts carefully before confirming:
# 1. Gas confirmation: Review gas costs
# 2. Object address: Note the object address for future reference

# OR use --assume-yes to auto-confirm (only if you're confident)
aptos move deploy-object \
    --address-name my_addr \
    --profile mainnet \
    --assume-yes

# SAVE THE OBJECT ADDRESS from output
# You'll need it for upgrades and documentation

# Confirm deployment
# Review transaction in explorer:
# https://explorer.aptoslabs.com/?network=mainnet
```

### Step 7: Verify Deployment

```bash
# Check module is published
aptos account list --account <mainnet_address> --profile mainnet

# Look for your module in the output
# "0x...::my_module": { ... }

# Run view function to verify
aptos move view \
    --profile mainnet \
    --function-id <mainnet_address>::<module>::<view_function> \
    --args ...
```

### Step 8: Document Deployment

Create deployment record:

```markdown
# Deployment Record

**Date:** 2026-01-23
**Network:** Mainnet
**Module:** my_module
**Address:** 0x123abc...
**Transaction:** 0x456def...

## Verification

- [x] Deployed successfully
- [x] Module visible in explorer
- [x] View functions working
- [x] Entry functions tested

## Links

- Explorer: https://explorer.aptoslabs.com/account/0x123abc...?network=mainnet
- Transaction: https://explorer.aptoslabs.com/txn/0x456def...?network=mainnet

## Notes

- All security checks passed
- 100% test coverage verified
- Tested on testnet for 1 week before mainnet
```

## Network-Specific Commands

### Devnet Deployment

```bash
aptos move publish \
    --network devnet \
    --named-addresses my_addr=<devnet_address>
```

**Devnet Details:**
- **Purpose:** Quick testing, experimentation
- **Stability:** May be reset
- **Faucet:** Available
- **URL:** https://fullnode.devnet.aptoslabs.com/v1

### Testnet Deployment

```bash
aptos move publish \
    --network testnet \
    --named-addresses my_addr=<testnet_address>
```

**Testnet Details:**
- **Purpose:** Pre-production testing
- **Stability:** More stable than devnet
- **Faucet:** Available
- **URL:** https://fullnode.testnet.aptoslabs.com/v1

### Mainnet Deployment

```bash
aptos move publish \
    --network mainnet \
    --named-addresses my_addr=<mainnet_address> \
    --max-gas 20000
```

**Mainnet Details:**
- **Purpose:** Production
- **Stability:** Permanent
- **Faucet:** Not available (real APT required)
- **URL:** https://fullnode.mainnet.aptoslabs.com/v1

## Module Upgrades

### Upgrading Existing Module

**Aptos Move modules are upgradeable by default for the account owner.**

```bash
# Deploy upgrade
aptos move publish \
    --profile mainnet \
    --named-addresses my_addr=<mainnet_address> \
    --upgrade

# Verify upgrade
aptos account list --account <mainnet_address> --profile mainnet
```

**Upgrade Compatibility Rules:**

- ✅ **CAN:** Add new functions
- ✅ **CAN:** Add new structs
- ✅ **CAN:** Add new fields to structs (with care)
- ❌ **CANNOT:** Remove existing functions (breaks compatibility)
- ❌ **CANNOT:** Change function signatures (breaks compatibility)
- ❌ **CANNOT:** Remove struct fields (breaks existing data)

### Making Modules Immutable

**To prevent future upgrades:**

```move
// In your module
fun init_module(account: &signer) {
    // After deployment, burn upgrade capability
    // (implementation depends on your setup)
}
```

## Cost Estimation

### Gas Costs

```bash
# Simulate deployment to estimate gas
aptos move publish \
    --named-addresses my_addr=<address> \
    --simulate

# Typical costs:
# - Small module: ~500-1000 gas units
# - Medium module: ~1000-5000 gas units
# - Large module: ~5000-20000 gas units
```

### Mainnet Costs

**Gas costs are paid in APT:**
- Gas units × Gas price = Total cost
- Example: 5000 gas units × 100 Octas/gas = 500,000 Octas = 0.005 APT

## Multi-Module Deployment

### Deploying Multiple Modules

**Option 1: Single package**

```
project/
├── Move.toml
└── sources/
    ├── module1.move
    ├── module2.move
    └── module3.move
```

```bash
# Deploys all modules at once
aptos move publish --named-addresses my_addr=<address>
```

**Option 2: Separate packages**

```bash
# Deploy dependency first
cd dependency-package
aptos move publish --named-addresses dep_addr=<address>

# Then deploy main package
cd ../main-package
aptos move publish --named-addresses main_addr=<address>,dep_addr=<dep_address>
```

## Troubleshooting Deployment

### "Insufficient APT balance"

```bash
# Testnet/Devnet: Use faucet
aptos account fund-with-faucet --profile testnet

# Mainnet: Transfer APT to your account
```

### "Module already exists"

```bash
# Use --upgrade flag
aptos move publish --named-addresses my_addr=<address> --upgrade
```

### "Address verification failed"

```bash
# Ensure named address matches your account
aptos move publish --named-addresses my_addr=$(aptos account list --profile mainnet --account default | grep "account" | cut -d'"' -f4)
```

### "Compilation failed"

```bash
# Fix compilation errors first
aptos move compile --named-addresses my_addr=<address>
# Fix all errors, then retry deployment
```

### "Gas limit exceeded"

```bash
# Increase max gas
aptos move publish \
    --named-addresses my_addr=<address> \
    --max-gas 30000
```

## Deployment Checklist

**Before Deployment:**
- [ ] Security audit passed
- [ ] 100% test coverage
- [ ] All tests passing
- [ ] Code compiles successfully
- [ ] Named addresses configured
- [ ] Target network selected (testnet first!)

**During Deployment:**
- [ ] Correct network selected
- [ ] Correct address specified
- [ ] Transaction submitted
- [ ] Transaction hash recorded

**After Deployment:**
- [ ] Module visible in explorer
- [ ] View functions work
- [ ] Entry functions tested
- [ ] Deployment documented
- [ ] Team notified

## ALWAYS Rules

- ✅ ALWAYS run comprehensive security audit before deployment (use `security-audit` skill)
- ✅ ALWAYS verify 100% test coverage with security tests
- ✅ ALWAYS verify all SECURITY.md patterns (arithmetic, storage scoping, reference safety, business logic)
- ✅ ALWAYS deploy to testnet before mainnet
- ✅ ALWAYS test on testnet thoroughly (happy paths, error cases, security scenarios)
- ✅ ALWAYS backup private keys securely
- ✅ ALWAYS document deployment addresses
- ✅ ALWAYS verify deployment in explorer
- ✅ ALWAYS test functions after deployment
- ✅ ALWAYS use separate keys for testnet and mainnet (SECURITY.md - Operations)

## NEVER Rules

- ❌ NEVER deploy without comprehensive security audit
- ❌ NEVER deploy with < 100% test coverage
- ❌ NEVER deploy without security test verification
- ❌ NEVER deploy directly to mainnet without testnet testing
- ❌ NEVER deploy without testing on testnet first
- ❌ NEVER commit private keys to version control
- ❌ NEVER skip post-deployment verification
- ❌ NEVER rush mainnet deployment
- ❌ NEVER reuse publishing keys between testnet and mainnet (security risk)

## References

**Official Documentation:**
- CLI Publishing: https://aptos.dev/build/cli/working-with-move-contracts
- Network Endpoints: https://aptos.dev/nodes/networks
- Gas and Fees: https://aptos.dev/concepts/gas-txn-fee

**Explorers:**
- Mainnet: https://explorer.aptoslabs.com/?network=mainnet
- Testnet: https://explorer.aptoslabs.com/?network=testnet
- Devnet: https://explorer.aptoslabs.com/?network=devnet

**Related Skills:**
- `security-audit` - Audit before deployment
- `generate-tests` - Ensure tests exist
- `use-aptos-cli` - CLI command reference
- `troubleshoot-errors` - Fix deployment errors

---

**Remember:** Security audit → 100% tests → Testnet → Thorough testing → Mainnet. Never skip testnet.
