# BlieverMarketFactory — Constructor Tests

**File:** `contracts/test/BlieverMarketFactory/BlieverMarketFactory_Constructor.t.sol`  
**Contract under test:** `BlieverMarketFactory` constructor  
**Base:** extends `FactoryTestBase`

---

## What This File Tests

The constructor is the factory's single setup point. It must correctly bind three immutable dependencies, grant three roles to one admin address, and enforce four early-rejection guards before any state is committed. No market deployment is involved.

---

## Test Groups

### 1. Immutables & Initial State (5 tests)

| Test | What it checks |
|---|---|
| `test_constructor_implementation_isSet` | `factory.implementation()` equals the BlieverMarket impl address passed to constructor |
| `test_constructor_pool_isSet` | `factory.pool()` equals the MockFactoryPool address |
| `test_constructor_adapter_isSet` | `factory.adapter()` equals the MockUmaAdapter address |
| `test_constructor_marketCount_isZero` | `marketCount` starts at 0 — no markets deployed yet |
| `test_constructor_paused_isFalse` | Factory is not paused at construction — `deployMarket` must be available immediately |

**Why it matters:** Immutables are baked into bytecode. Any mismatch means every downstream call (registerMarket, initializeQuestion, clone deploy) will target wrong contracts permanently.

---

### 2. Role Grants (4 tests)

| Test | What it checks |
|---|---|
| `test_constructor_admin_hasDefaultAdminRole` | `admin` holds `DEFAULT_ADMIN_ROLE` |
| `test_constructor_admin_hasOperatorRole` | `admin` holds `OPERATOR_ROLE` |
| `test_constructor_admin_hasPauserRole` | `admin` holds `PAUSER_ROLE` |
| `test_constructor_strangerHasNoRoles` | `attacker` holds none of the three roles |

**Why it matters:** OZ `AccessControl` starts with no roles — every role must be explicitly granted. A missing grant means the protocol is permanently broken at deployment (admin can never create markets or pause them).

---

### 3. Zero-Address Reverts (4 tests)

Each of the four constructor parameters is independently tested to confirm `BlieverMarketFactory__ZeroAddress` is emitted when `address(0)` is passed:

- `test_constructor_reverts_zeroImplementation`
- `test_constructor_reverts_zeroPool`
- `test_constructor_reverts_zeroAdapter`
- `test_constructor_reverts_zeroAdmin`

**Why it matters:** A zero-address implementation would silently succeed at clone creation but revert on every `initialize()` call, producing orphaned contracts. A zero admin would mean all role-gated functions are permanently locked.

---

### 4. Implementation Must Have Code (2 tests)

| Test | What it checks |
|---|---|
| `test_constructor_reverts_implementationIsEOA` | Passing a plain EOA address reverts with `BlieverMarketFactory__NotAContract(eoa)` |
| `test_constructor_acceptsAnyContractAsImpl` | Any address _with_ code passes the guard — the check is code-length only, not type-safety |

**Why it matters:** EIP-1167 proxies DELEGATECALL into `implementation` on every call. An EOA as implementation means every call silently returns empty calldata rather than executing BlieverMarket logic — difficult to detect after deployment.

---

### 5. Constants Sanity (4 tests)

| Test | What it checks |
|---|---|
| `test_constants_MIN_OUTCOMES` | `MIN_OUTCOMES == 2` |
| `test_constants_MAX_OUTCOMES` | `MAX_OUTCOMES == 7` |
| `test_constants_OPERATOR_ROLE_selector` | `OPERATOR_ROLE == keccak256("OPERATOR_ROLE")` |
| `test_constants_PAUSER_ROLE_selector` | `PAUSER_ROLE == keccak256("PAUSER_ROLE")` |

**Why it matters:** These constants govern the outcome-count validation gate and the AccessControl selectors used across the entire test suite. Confirming them in isolation prevents silent off-by-one bugs in later tests.

---

## What Is NOT Tested Here

- `deployMarket` logic (covered in `BlieverMarketFactory_DeployMarket.t.sol`)
- Lifecycle operations (covered in `BlieverMarketFactory_Lifecycle.t.sol`)
- Epsilon / address prediction math (covered in `BlieverMarketFactory_Views.t.sol`)

---
