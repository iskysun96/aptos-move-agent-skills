# Skill: implement-upgradeable-contracts

Design and implement upgradeable contract patterns for Aptos Move V2, enabling safe contract evolution while maintaining
state and security.

## When to Use This Skill

**Trigger phrases:**

- "upgradeable contract", "contract upgrade", "upgradeable pattern"
- "version management", "contract migration", "upgrade strategy"
- "backward compatibility", "contract evolution"
- "proxy pattern", "diamond pattern", "upgradeable module"

**Use cases:**

- Long-lived protocols requiring updates
- DeFi protocols with evolving features
- NFT contracts needing new functionality
- Governance-controlled upgrades
- Bug fixes without losing state

## Aptos Upgrade Model

Aptos supports **module upgrades** with strict compatibility rules:

- Can add new functions
- Can add new structs
- Cannot remove existing functions
- Cannot change existing function signatures
- Cannot change existing struct layouts
- Resources persist across upgrades

## Upgrade Patterns

### 1. Basic Module Upgrade Pattern

```move
module my_protocol::token_v1 {
    use std::string::String;
    use aptos_framework::object;

    /// Version for tracking upgrades
    const VERSION: u64 = 1;

    struct TokenConfig has key {
        version: u64,
        name: String,
        symbol: String,
        decimals: u8,
        // Cannot remove fields in upgrades!
    }

    /// Initialize module
    fun init_module(account: &signer) {
        move_to(account, TokenConfig {
            version: VERSION,
            name: string::utf8(b"MyToken"),
            symbol: string::utf8(b"MTK"),
            decimals: 8,
        });
    }

    /// Original functionality
    public fun get_decimals(token_addr: address): u8 acquires TokenConfig {
        borrow_global<TokenConfig>(token_addr).decimals
    }
}

// ===== UPGRADED VERSION =====
module my_protocol::token_v2 {
    use std::string::String;
    use aptos_framework::object;

    const VERSION: u64 = 2;

    struct TokenConfig has key {
        version: u64,
        name: String,
        symbol: String,
        decimals: u8,
        // New fields must be added at the end
        max_supply: u64,  // NEW in v2
        paused: bool,     // NEW in v2
    }

    /// Existing functions preserved
    public fun get_decimals(token_addr: address): u8 acquires TokenConfig {
        borrow_global<TokenConfig>(token_addr).decimals
    }

    /// NEW: Migration function
    public entry fun migrate_v1_to_v2(admin: &signer) acquires TokenConfig {
        let admin_addr = signer::address_of(admin);
        assert!(admin_addr == @my_protocol, E_NOT_ADMIN);

        if (exists<TokenConfig>(admin_addr)) {
            let config = borrow_global_mut<TokenConfig>(admin_addr);
            if (config.version == 1) {
                config.version = VERSION;
                // Initialize new fields with defaults
                // Note: Cannot access new fields until migration
            }
        }
    }

    /// NEW: Additional functionality
    public fun get_max_supply(token_addr: address): u64 acquires TokenConfig {
        let config = borrow_global<TokenConfig>(token_addr);
        assert!(config.version >= 2, E_NEEDS_MIGRATION);
        config.max_supply
    }
}
```

### 2. Proxy Pattern with Objects

```move
module my_protocol::proxy_token {
    use std::string::String;
    use aptos_framework::object::{Self, Object};

    /// Proxy object that delegates to implementation
    struct TokenProxy has key {
        implementation: address,
        version: u64,
    }

    /// Implementation interface
    struct TokenImpl has key {
        // Actual token logic
        total_supply: u64,
        // ... other fields
    }

    /// Create upgradeable token
    public fun create_token(
        creator: &signer,
        name: String
    ): Object<TokenProxy> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        // Deploy implementation
        let impl_addr = deploy_implementation(&object_signer);

        // Create proxy
        move_to(&object_signer, TokenProxy {
            implementation: impl_addr,
            version: 1,
        });

        object::object_from_constructor_ref<TokenProxy>(&constructor_ref)
    }

    /// Upgrade implementation (governance controlled)
    public entry fun upgrade_implementation(
        governance: &signer,
        proxy: Object<TokenProxy>,
        new_impl: address
    ) acquires TokenProxy {
        verify_governance(governance);

        let proxy_data = borrow_global_mut<TokenProxy>(
            object::object_address(&proxy)
        );

        // Migrate state if needed
        migrate_state(proxy_data.implementation, new_impl);

        // Update implementation
        proxy_data.implementation = new_impl;
        proxy_data.version = proxy_data.version + 1;
    }
}
```

### 3. Diamond Pattern (Multi-Facet)

```move
module my_protocol::diamond {
    use std::table::{Self, Table};
    use aptos_framework::object::{Self, Object};

    /// Diamond proxy with multiple facets
    struct Diamond has key {
        /// Function selector -> implementation address
        facets: Table<vector<u8>, address>,
        version: u64,
    }

    /// Facet registry
    struct FacetInfo has store, drop {
        address: address,
        selectors: vector<vector<u8>>,
    }

    /// Initialize diamond
    public fun create_diamond(creator: &signer): Object<Diamond> {
        let constructor_ref = object::create_object(signer::address_of(creator));
        let object_signer = object::generate_signer(&constructor_ref);

        move_to(&object_signer, Diamond {
            facets: table::new(),
            version: 1,
        });

        object::object_from_constructor_ref<Diamond>(&constructor_ref)
    }

    /// Add or update facet
    public fun update_facet(
        admin: &signer,
        diamond: Object<Diamond>,
        facet_addr: address,
        selectors: vector<vector<u8>>
    ) acquires Diamond {
        verify_admin(admin);

        let diamond_data = borrow_global_mut<Diamond>(
            object::object_address(&diamond)
        );

        // Update selectors
        let i = 0;
        while (i < vector::length(&selectors)) {
            let selector = vector::borrow(&selectors, i);
            table::upsert(&mut diamond_data.facets, *selector, facet_addr);
            i = i + 1;
        };

        diamond_data.version = diamond_data.version + 1;
    }
}
```

### 4. Version Manager Pattern

```move
module my_protocol::version_manager {
    use std::table::{Self, Table};
    use std::vector;

    /// Central version management
    struct VersionManager has key {
        current_version: u64,
        /// Version -> module address mapping
        versions: Table<u64, address>,
        /// Feature flags by version
        features: Table<u64, vector<u8>>,
        migration_status: Table<address, u64>,
    }

    /// Initialize version manager
    fun init_module(account: &signer) {
        move_to(account, VersionManager {
            current_version: 1,
            versions: table::new(),
            features: table::new(),
            migration_status: table::new(),
        });
    }

    /// Register new version
    public fun register_version(
        admin: &signer,
        version: u64,
        module_addr: address,
        features: vector<u8>
    ) acquires VersionManager {
        verify_admin(admin);

        let manager = borrow_global_mut<VersionManager>(@my_protocol);
        table::add(&mut manager.versions, version, module_addr);
        table::add(&mut manager.features, version, features);

        if (version > manager.current_version) {
            manager.current_version = version;
        }
    }

    /// Check if user needs migration
    public fun needs_migration(user_addr: address): bool acquires VersionManager {
        let manager = borrow_global<VersionManager>(@my_protocol);
        if (!table::contains(&manager.migration_status, user_addr)) {
            return true
        };

        let user_version = *table::borrow(&manager.migration_status, user_addr);
        user_version < manager.current_version
    }
}
```

## Upgrade Strategies

### 1. Gradual Migration Strategy

```move
module my_protocol::gradual_migration {
    /// Support multiple versions simultaneously
    struct UserData has key {
        version: u64,
        // Common fields
        balance: u64,
    }

    struct UserDataV2 has key {
        // Extended data for v2 users
        rewards: u64,
        tier: u8,
    }

    /// Get balance works for all versions
    public fun get_balance(user: address): u64 acquires UserData {
        borrow_global<UserData>(user).balance
    }

    /// V2 features require migration
    public fun get_rewards(user: address): u64 acquires UserData, UserDataV2 {
        let user_data = borrow_global<UserData>(user);
        assert!(user_data.version >= 2, E_NEEDS_MIGRATION);

        borrow_global<UserDataV2>(user).rewards
    }

    /// Opt-in migration
    public entry fun migrate_to_v2(user: &signer) acquires UserData {
        let user_addr = signer::address_of(user);
        let user_data = borrow_global_mut<UserData>(user_addr);

        if (user_data.version == 1) {
            user_data.version = 2;

            // Initialize V2 data
            move_to(user, UserDataV2 {
                rewards: 0,
                tier: 0,
            });

            // Migration bonus
            user_data.balance = user_data.balance + MIGRATION_BONUS;
        }
    }
}
```

### 2. Emergency Pause and Upgrade

```move
module my_protocol::emergency_upgrade {
    struct ProtocolState has key {
        paused: bool,
        emergency_admin: address,
        upgrade_delay: u64,
        proposed_upgrade: Option<UpgradeProposal>,
    }

    struct UpgradeProposal has store, drop {
        new_impl: address,
        timestamp: u64,
        executed: bool,
    }

    /// Emergency pause
    public entry fun emergency_pause(admin: &signer) acquires ProtocolState {
        let state = borrow_global_mut<ProtocolState>(@my_protocol);
        assert!(signer::address_of(admin) == state.emergency_admin, E_NOT_ADMIN);
        state.paused = true;
    }

    /// Propose upgrade (with timelock)
    public entry fun propose_upgrade(
        admin: &signer,
        new_impl: address
    ) acquires ProtocolState {
        let state = borrow_global_mut<ProtocolState>(@my_protocol);
        verify_admin(admin);

        state.proposed_upgrade = option::some(UpgradeProposal {
            new_impl,
            timestamp: timestamp::now_seconds(),
            executed: false,
        });
    }

    /// Execute upgrade after delay
    public entry fun execute_upgrade(admin: &signer) acquires ProtocolState {
        let state = borrow_global_mut<ProtocolState>(@my_protocol);
        verify_admin(admin);

        let proposal = option::borrow_mut(&mut state.proposed_upgrade);
        assert!(!proposal.executed, E_ALREADY_EXECUTED);
        assert!(
            timestamp::now_seconds() >= proposal.timestamp + state.upgrade_delay,
            E_TIMELOCK_ACTIVE
        );

        // Execute upgrade
        perform_upgrade(proposal.new_impl);
        proposal.executed = true;

        // Resume operations
        state.paused = false;
    }
}
```

## Compatibility Rules

### What CAN Be Changed

- ✅ Add new functions
- ✅ Add new structs/resources
- ✅ Add new constants
- ✅ Add new error codes
- ✅ Change function implementations
- ✅ Add generic type parameters to new functions

### What CANNOT Be Changed

- ❌ Remove functions
- ❌ Change function signatures
- ❌ Remove or reorder struct fields
- ❌ Change struct abilities
- ❌ Remove structs/resources
- ❌ Change public constant values

### Safe Struct Evolution

```move
// Original struct
struct Token has key {
    name: String,
    symbol: String,
    total_supply: u64,
}

// INVALID: Cannot remove fields
struct Token has key {
    name: String,
    // symbol removed - BREAKS COMPATIBILITY
    total_supply: u64,
}

// INVALID: Cannot reorder fields
struct Token has key {
    symbol: String,  // Was second, now first - BREAKS
    name: String,
    total_supply: u64,
}

// VALID: Can add fields at end
struct Token has key {
    name: String,
    symbol: String,
    total_supply: u64,
    max_supply: u64,     // NEW - OK
    paused: bool,        // NEW - OK
}
```

## Testing Upgrades

### Upgrade Test Pattern

```move
#[test_only]
module my_protocol::upgrade_tests {
    use my_protocol::token_v1;
    use my_protocol::token_v2;

    #[test]
    fun test_upgrade_compatibility() {
        let admin = account::create_account_for_test(@my_protocol);

        // Deploy V1
        token_v1::init_module(&admin);

        // Use V1 features
        let decimals = token_v1::get_decimals(@my_protocol);
        assert!(decimals == 8, 0);

        // Upgrade to V2 (simulated)
        token_v2::migrate_v1_to_v2(&admin);

        // V1 features still work
        let decimals_v2 = token_v2::get_decimals(@my_protocol);
        assert!(decimals_v2 == 8, 1);

        // V2 features available
        let max_supply = token_v2::get_max_supply(@my_protocol);
        assert!(max_supply > 0, 2);
    }

    #[test]
    fun test_gradual_migration() {
        // Test that both V1 and V2 users can coexist
        let user1 = account::create_account_for_test(@0x1);
        let user2 = account::create_account_for_test(@0x2);

        // User1 stays on V1
        create_v1_user(&user1);

        // User2 migrates to V2
        create_v1_user(&user2);
        migrate_to_v2(&user2);

        // Both can use basic features
        assert!(get_balance(@0x1) > 0, 0);
        assert!(get_balance(@0x2) > 0, 1);

        // Only V2 user can use new features
        assert!(can_use_rewards(@0x2), 2);
    }
}
```

## Deployment Process

### 1. Prepare Upgrade

```bash
# 1. Test upgrade compatibility
aptos move test --filter upgrade

# 2. Compile new version
aptos move compile --save-metadata

# 3. Verify bytecode compatibility
aptos move verify-upgrade \
    --old-bytecode old_build/package.mv \
    --new-bytecode build/package.mv
```

### 2. Deploy Upgrade

```bash
# Deploy with upgrade capability
aptos move publish \
    --profile mainnet \
    --upgrade-policy compatible \
    --max-gas 200000

# For object-based deployment
aptos move deploy-object \
    --profile mainnet \
    --upgrade-policy compatible
```

### 3. Execute Migration

```bash
# Run migration function
aptos move run-function \
    --profile mainnet \
    --function-id 0x1::token_v2::migrate_v1_to_v2
```

## Best Practices

### 1. Version Everything

```move
const VERSION: u64 = 2;
const FEATURE_STAKING: u8 = 1;
const FEATURE_GOVERNANCE: u8 = 2;

struct Config has key {
    version: u64,
    enabled_features: vector<u8>,
}
```

### 2. Migration Functions

```move
/// Always provide migration path
public entry fun migrate_from_v1(user: &signer) acquires OldData {
    assert!(!has_migrated(signer::address_of(user)), E_ALREADY_MIGRATED);

    // Read old data
    let old_data = move_from<OldData>(signer::address_of(user));

    // Create new data structure
    move_to(user, NewData {
        // Map old fields
        ...old_data,
        // Initialize new fields
        new_field: default_value(),
    });
}
```

### 3. Feature Flags

```move
/// Use feature flags for gradual rollout
public fun use_new_feature(user: address) acquires Config {
    let config = borrow_global<Config>(@my_protocol);
    assert!(
        vector::contains(&config.enabled_features, &FEATURE_NEW),
        E_FEATURE_DISABLED
    );
    // Feature logic
}
```

### 4. Backward Compatibility

```move
/// Maintain old interfaces
public fun old_function(user: &signer) {
    // Delegate to new implementation
    new_function_with_defaults(user, DEFAULT_PARAM);
}
```

## Common Pitfalls

1. **Struct Layout Changes**: Never modify existing struct fields
2. **Missing Migrations**: Always provide migration paths
3. **Incompatible Upgrades**: Test with `verify-upgrade`
4. **State Loss**: Ensure all state is preserved or migrated
5. **Access Control**: Secure upgrade functions properly

## Security Considerations

- Use timelock for upgrades
- Implement multi-sig for upgrade approval
- Test extensively on testnet
- Provide rollback mechanisms
- Audit upgrade code thoroughly
- Document all changes clearly

## References

- Aptos Upgrade Guide: https://aptos.dev/build/smart-contracts/upgrade-guide
- Object Code Deployment: https://aptos.dev/build/smart-contracts/object-code-deploy
- Move Compatibility: https://github.com/aptos-labs/aptos-core/blob/main/aptos-move/framework/COMPATIBILITY.md
