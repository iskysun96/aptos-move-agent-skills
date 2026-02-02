# Next Actions After Iteration 004

**Date:** 2026-02-02
**Status:** Ready for implementation

---

## Priority 0: Migrate Files to Testing Repo (FIRST!)

⚠️ **Do this before starting other work:**

See `MIGRATION_INSTRUCTIONS.md` for detailed steps, or run:

```bash
cd /Users/chriskim/Documents/2.areas/Aptos/move-agent-skills-2-3
./MOVE_TO_TESTING_REPO.sh
```

This moves all iteration-004 files to `aptos-move-agent-skills-testing/iterations/004/` for organization.

---

## Priority 1: Enhance write-contracts Skill with Data Storage Decision Tree

### Problem Identified

**Cost of mistake:** 15,661 tokens (38% of Dapp 1's total)
**Root cause:** Developers don't know when to use Table vs move_to for data storage
**Frequency:** Occurred in 1 of 2 dapps (50% error rate)

### Solution: Add Quick Data Storage Decision Tree

**Location:** `skills/write-contracts/SKILL.md`
**Placement:** After "Module Structure Template", before "Anti-patterns"
**Size:** ~30-40 lines (maintains compressed format)

### Content Structure

```markdown
## Data Storage Decision Tree

### Question 1: Are you storing data ABOUT objects or IN objects?

**ABOUT objects (external metadata):**
- ✅ Use Table<address, Data> or SimpleMap<address, Data>
- Examples: NFT staking info, user balances, voting records
- Store in your contract's global config struct

**IN objects (object owns the data):**
- ✅ Use move_to with object's signer
- Examples: Object properties, internal object state
- Pattern: Create object → get signer → move_to

### Question 2: Do you control the object's signer?

**YES - You created the object:**
```move
let constructor_ref = object::create_object(creator);
let obj_signer = object::generate_signer(&constructor_ref);
move_to(&obj_signer, MyData { ... });  // ✅ Works!
```

**NO - Object exists externally (NFTs, tokens):**
```move
// ❌ WRONG: Can't get signer for external objects
// ✅ RIGHT: Store data about the object externally
let config = borrow_global_mut<Config>(@my_addr);
table::add(&mut config.data, object_address, MyData { ... });
```

### Quick Patterns

**Pattern 1: Track data about many objects (Registry)**
```move
struct Registry has key {
    data: Table<address, ObjectData>,
}
```
Use when: Staking, voting, registries, external object tracking

**Pattern 2: Store data in your own objects**
```move
public fun create_my_object(creator: &signer): Object<MyObject> {
    let constructor_ref = object::create_object(signer::address_of(creator));
    let obj_signer = object::generate_signer(&constructor_ref);
    move_to(&obj_signer, MyObject { ... });
    object::object_from_constructor_ref(&constructor_ref)
}
```
Use when: Creating objects you control, marketplace items

**Pattern 3: Global config/state**
```move
fun init_module(deployer: &signer) {
    move_to(deployer, GlobalConfig { ... });
}
```
Use when: DAO config, protocol settings

### Common Mistake

❌ **NEVER store data IN objects you don't control**
✅ **ALWAYS store data ABOUT external objects in your contract**
```

### Expected Impact

**Prevented errors:** 50% of design mistakes (based on testing)
**Token savings:** 15,000+ tokens per prevented mistake
**Development speed:** Faster first-time-right implementations

### Implementation Steps

1. Open `skills/write-contracts/SKILL.md`
2. Locate line ~99 (after "Module Structure Template")
3. Insert the Data Storage Decision Tree section
4. Verify total file size stays under 250 lines
5. Test with a sample staking contract prompt
6. Commit with message: "Add data storage decision tree to prevent Table vs move_to confusion"

### Success Criteria

- [ ] Section added to write-contracts skill
- [ ] File size remains compressed (<250 lines)
- [ ] Decision tree covers all 3 patterns
- [ ] Examples are inline and brief
- [ ] Tested with staking contract scenario
- [ ] No design mistakes in subsequent testing

---

## Priority 2: Apply Compression to Remaining Skills

Based on iteration 004 success, apply same 80% compression pattern to:

### Skills to Compress

1. **security-audit** - Currently verbose, can compress to ~200 lines
2. **deploy-contracts** - Apply progressive disclosure
3. **analyze-gas-optimization** - Front-load critical optimizations
4. **scaffold-project** - Compress project structure guidance
5. **search-aptos-examples** - Already relatively compact, minor tweaks

### Compression Pattern (Proven Effective)

- Keep file under 250 lines
- Front-load ALWAYS/NEVER rules
- Include inline examples for key patterns
- Move verbose explanations to references/
- Add trigger-rich YAML frontmatter

---

## Priority 3: Release v1.0.0

### Prerequisites

- [x] Iteration 004 validation complete
- [ ] Priority 1 enhancement implemented
- [ ] Priority 2 compressions complete
- [ ] All skills tested

### Release Checklist

- [ ] Update CHANGELOG.md with iteration 004 findings
- [ ] Tag v1.0.0 in git
- [ ] Update README with "Production Ready" badge
- [ ] Document compression results
- [ ] Create migration guide for users of old verbose skills

---

## Reference: Iteration 004 Results

**Token Savings Achieved:**
- Skill loading: 67% reduction (13,076 tokens saved)
- Overall development: 24-59% reduction depending on complexity
- Zero critical errors maintained
- 95% average test coverage

**Key Finding:** Progressive disclosure with 80% compression is production-ready.

---

**Next Session:** Start with Priority 1 (Data Storage Decision Tree)
**Estimated Time:** 30-60 minutes to implement and test
**Expected Impact:** Prevent 50% of design mistakes, save 15k+ tokens per prevented error
