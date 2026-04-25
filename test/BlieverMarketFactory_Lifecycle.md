# BlieverMarketFactory — Lifecycle Tests

**File:** `contracts/test/BlieverMarketFactory/BlieverMarketFactory_Lifecycle.t.sol`  
**Contract under test:** `expireUnresolved`, `pauseMarket`, `unpauseMarket`, `deregisterMarket`, `pause`, `unpause`  
**Base:** extends `FactoryTestBase`

---

## What This File Tests

All post-deployment operations that the factory performs on behalf of the protocol. Each operation either:
- **Forwards a call** into a deployed BlieverMarket clone (expireUnresolved, pauseMarket, unpauseMarket)
- **Forwards a call** into the pool (deregisterMarket)
- **Controls the factory itself** (global pause / unpause)

Every function that forwards into a clone first calls `_assertDeployedMarket`, which is the factory's security perimeter: it rejects any address not created by this factory instance.

Each test uses `_freshMarket(label)` — a helper that deploys a 2-outcome binary market with a unique questionId per test, so tests are fully isolated.

---

## Test Groups

### 1. expireUnresolved — Happy Path (5 tests)

| Test | What it checks |
|---|---|
| `test_expireUnresolved_success_emitsEvent` | `MarketExpiredByFactory(market, qId, timestamp)` is emitted |
| `test_expireUnresolved_callsPoolSettleWithZeroPayout` | `pool.settleMarket` called once with `totalPayout = 0` |
| `test_expireUnresolved_setsMarketResolved` | `IDeployableMarket(market).resolved() == true` after expiry |
| `test_expireUnresolved_isPermissionless` | `attacker` (no role) can trigger expiry — function is open to anyone |
| `test_expireUnresolved_exactlyOnDeadlinePlusOne` | Warping to `resolutionDeadline + 1` is the minimum valid timestamp |

**Setup pattern:** Deploy market → `vm.warp(block.timestamp + T_RESOLUTION + 1)` → call expireUnresolved.

**Why it matters:** Expiry is a safety hatch for oracle failures. If it were role-gated, a single compromised key could hold trader funds hostage. The permissionless design ensures the protocol can always self-heal.

---

### 2. expireUnresolved — Reverts (5 tests)

| Test | Revert | Condition |
|---|---|---|
| `test_expireUnresolved_reverts_beforeDeadline` | `ResolutionDeadlineNotPassed(deadline, now)` | Warp to exactly `resolutionDeadline` (boundary, still reverts) |
| `test_expireUnresolved_reverts_alreadyResolved` | `MarketAlreadyResolved(market)` | Market resolved normally before deadline; expire attempted after |
| `test_expireUnresolved_reverts_notDeployedMarket` | `NotDeployedMarket(random)` | Arbitrary address (not a factory clone) |
| `test_expireUnresolved_reverts_calledTwice` | `MarketAlreadyResolved(market)` | Second call after successful first expiry |

**Key timing detail for `reverts_alreadyResolved`:**
1. Deploy market
2. `vm.warp(T_TRADING + 1)` → after trading closes, before resolution deadline
3. `_resolveMarket(market, 0)` via mockAdapter (resolver = mockAdapter address)
4. `vm.warp(T_RESOLUTION + 1)` → past resolution deadline
5. `factory.expireUnresolved(market)` → factory checks `m.resolved()` first → reverts immediately

**Why boundary at exactly `resolutionDeadline` matters:** The factory uses `currentTime <= deadline` (strictly-not-greater) so the exact deadline second is still too early. Confirming this prevents an off-by-one that could allow premature expiry.

---

### 3. pauseMarket (4 tests)

| Test | What it checks |
|---|---|
| `test_pauseMarket_success_emitsEvent` | `MarketPausedByFactory(market)` emitted |
| `test_pauseMarket_cloneIsPaused` | `BlieverMarket(market).paused() == true` |
| `test_pauseMarket_reverts_notPauser` | `attacker` cannot pause (missing PAUSER_ROLE) |
| `test_pauseMarket_reverts_notDeployedMarket` | Arbitrary address rejected by `_assertDeployedMarket` |

**Why it matters:** BlieverMarket.pause() is `onlyFactory` guarded. The factory is the sole entry point. Testing the pause state from the clone side confirms the delegation chain works end-to-end.

---

### 4. unpauseMarket (4 tests)

| Test | What it checks |
|---|---|
| `test_unpauseMarket_success_emitsEvent` | `MarketUnpausedByFactory(market)` emitted |
| `test_unpauseMarket_cloneIsNotPaused` | `BlieverMarket(market).paused() == false` after unpause |
| `test_unpauseMarket_reverts_notPauser` | Missing PAUSER_ROLE → revert |
| `test_unpauseMarket_reverts_notDeployedMarket` | Not-a-factory-market → `NotDeployedMarket` revert |

---

### 5. deregisterMarket (4 tests)

| Test | What it checks |
|---|---|
| `test_deregisterMarket_success_emitsEvent` | `MarketDeregisteredByFactory(market)` emitted |
| `test_deregisterMarket_callsPoolDeregister` | `pool.deregisterMarket(market)` called with correct address |
| `test_deregisterMarket_reverts_notAdmin` | `operator` (OPERATOR_ROLE but NOT DEFAULT_ADMIN_ROLE) is rejected |
| `test_deregisterMarket_reverts_notPauser` | `pauser` (PAUSER_ROLE but NOT DEFAULT_ADMIN_ROLE) is rejected |
| `test_deregisterMarket_reverts_notDeployedMarket` | Stranger address rejected |

**Role boundary note:** `deregisterMarket` requires `DEFAULT_ADMIN_ROLE` — the strongest role. Both `test_deregisterMarket_reverts_notAdmin` and `test_deregisterMarket_reverts_notPauser` confirm that lower roles (even OPERATOR which can deploy markets) cannot deregister.

---

### 6. Factory-Level Pause (6 tests)

| Test | What it checks |
|---|---|
| `test_factoryPause_blocksDeployMarket` | Paused factory reverts on `deployMarket` |
| `test_factoryUnpause_allowsDeployMarket` | After unpause, `deployMarket` succeeds |
| `test_factoryPause_reverts_notPauser` | `attacker` cannot call `factory.pause()` |
| `test_factoryUnpause_reverts_notPauser` | `attacker` cannot call `factory.unpause()` |
| `test_factoryPause_doesNotBlockExpireUnresolved` | Factory pause does NOT affect `expireUnresolved` |
| `test_factoryPause_doesNotBlockPauseMarket` | Factory pause does NOT affect `pauseMarket` |
| `test_factoryPause_doesNotBlockDeregisterMarket` | Factory pause does NOT affect `deregisterMarket` |

**Critical design assertion:** Only `deployMarket` is `whenNotPaused` guarded. The three lifecycle function tests (`doesNotBlock*`) are regression canaries — if someone accidentally adds `whenNotPaused` to expireUnresolved, these tests will catch it.

---

### 7. _assertDeployedMarket Guard (1 test)

| Test | What it checks |
|---|---|
| `test_assertDeployedMarket_rejectsExternalClone` | A clone deployed via `Clones.clone(impl)` directly (bypassing factory) is rejected by `pauseMarket` |

**Why it matters:** Without this guard, an attacker could deploy a malicious contract that looks like a BlieverMarket clone and trick the factory into calling `pause()` / `expireUnresolved()` / `unpause()` on it, exploiting the `onlyFactory` gate on any contract the attacker controls.

---

## What Is NOT Tested Here

- `deployMarket` inputs and atomicity (→ DeployMarket tests)
- Epsilon math (→ Views tests)
- Constructor (→ Constructor tests)
- Real oracle resolution (UMA adapter is mocked — only initializeQuestion is spied on)

---

## Improvement / Debugging Notes

- `test_expireUnresolved_reverts_alreadyResolved` resolves via `mockAdapter` prank. If the resolver address ever changes (e.g., a new adapter deployment), update `_resolveMarket` in `FactoryTestBase`.
- The lifecycle tests do not test reentrancy on `expireUnresolved`. Add a malicious pool mock that re-enters the factory if reentrancy protection needs explicit validation.
- `test_deregisterMarket_reverts_notAdmin` validates operator role cannot deregister. If a future version adds an `OPERATOR_DEREGISTER_ROLE`, this test becomes a regression guard — update it intentionally.
- Factory-level `pause` and clone-level `pauseMarket` are independent mechanisms. A test that pauses both and verifies the interaction (e.g., unpauseMarket works while factory is paused) could be added under the "combined state" scenario.
