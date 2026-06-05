# BlieverMarketFactory — Test Suite Overview

**Directory:** `contracts/test/BlieverMarketFactory/`  
**Contract under test:** `contracts/src/BlieverMarketFactory.sol`  
**Chain:** Base (Solidity 0.8.31, Foundry)

---

## What Is Tested

| File | Function(s) tested | Test count |
|---|---|---|
| `Constructor` | `constructor(impl, pool, adapter, admin)` | 19 |
| `DeployMarket` | `deployMarket(DeployParams)` | 28 |
| `Lifecycle` | `expireUnresolved`, `pauseMarket`, `unpauseMarket`, `deregisterMarket`, `pause`, `unpause` | 24 |
| `Views` | `computeEpsilon(uint8)`, `predictMarketAddress(bytes32)` | 17 |
| **Total** | | **~88** |

---

## Mock Contracts (defined in BlieverMarketFactoryBase.t.sol)

| Mock | Replaces | Key features |
|---|---|---|
| `MockFactoryPool` | `BlieverV1Pool` | Configurable `alpha` / `maxRiskPerMarket`; spy counters for `registerMarket` / `deregisterMarket` / `settleMarket`; fault-injection `revertOnRegister` flag |
| `MockUmaAdapter` | `BlieverUmaAdapter` | Spy for `initializeQuestion` — records all 7 args; no oracle logic |
| `MockRewardToken` | Any ERC-20 OO reward | Standard OZ ERC-20 with free `mint` |
| `MockUSDC` | USDC (pool asset) | 6-decimal minimal ERC-20 |

**Design rule:** Mocks capture *what was called and with what arguments*, not *whether the call is correct semantically*. The factory's correctness is tested by asserting spy values, not by running real pool/oracle logic.

---

## Actor Addresses

| Name | Role | Used for |
|---|---|---|
| `admin` | DEFAULT_ADMIN_ROLE | deregisterMarket, role grants |
| `operator` | OPERATOR_ROLE | deployMarket |
| `pauser` | PAUSER_ROLE | pause/unpause factory and markets |
| `attacker` | none | all unauthorized-caller tests |
| `anyone` | none | permissionless tests (expireUnresolved) |

---

## Running the Tests

```bash
# All factory tests
forge test --match-path "test/BlieverMarketFactory/*" -vv

# Specific file
forge test --match-contract "BlieverMarketFactory_DeployMarket" -vvv

# Fuzz with more runs
forge test --match-test "testFuzz" --fuzz-runs 10000 -vv

# Gas snapshot
forge snapshot --match-path "test/BlieverMarketFactory/*"
```

---

## Key Design Decisions

**Why no fork tests?**  
The factory integrates with `BlieverV1Pool` and `BlieverUmaAdapter`, both of which have their own test suites. Fork tests for the factory would duplicate pool and oracle behavior testing. Mock-based tests are faster, deterministic, and isolate factory logic cleanly.

**Why is `MockFactoryPool` not the same as `MockPool` from `BlieverMarketBase.t.sol`?**  
`BlieverMarketBase.MockPool` was designed to test market trading (buy/sell/claim flows with real token transfers). `MockFactoryPool` is designed to test the factory's interaction surface: `alpha()`, `maxRiskPerMarket()`, `registerMarket()`, and `deregisterMarket()`. Merging them would add unneeded complexity to both.

**Why does `_resolveMarket` prank as `mockAdapter`?**  
`BlieverMarket.resolve()` is `onlyResolver` guarded. The resolver is set in `market.initialize()` as `address(adapter)` — i.e., `address(mockAdapter)`. Therefore tests that need a resolved market must prank the MockUmaAdapter address.

---

## Coverage Goals

| Metric | Target |
|---|---|
| Line coverage | ≥ 95 % |
| Branch coverage | ≥ 90 % |

```bash
forge coverage --match-path "test/BlieverMarketFactory/*" --report summary
```

Branches not covered by unit tests that may require additional effort:
- `epsilon == 0` revert path in `deployMarket` (requires pool returning `maxRisk = 0`)
- `pool.registerMarket` hard-revert path (use `mockPool.setRevertOnRegister(true)`)
