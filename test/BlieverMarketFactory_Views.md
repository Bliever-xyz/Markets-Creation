# BlieverMarketFactory — View & Pure Function Tests

**File:** `contracts/test/BlieverMarketFactory/BlieverMarketFactory_Views.t.sol`  
**Contract under test:** `computeEpsilon(uint8)` and `predictMarketAddress(bytes32)`  
**Base:** extends `FactoryTestBase`

---

## What This File Tests

Two read-only functions that operators and frontends call before submitting a `deployMarket` transaction:

- **`computeEpsilon`** — computes the LS-LMSR initial quantity seed ε from live pool parameters. Exposed publicly so operators can verify the ε that `deployMarket` will use without executing a transaction.
- **`predictMarketAddress`** — returns the deterministic CREATE2 address a clone will receive. Used by frontends and order engines to route limit orders to a market address before the deployment transaction is broadcast.

Both are pure/view — no state mutation occurs. Tests focus on mathematical correctness, boundary enforcement, and determinism.

---

## Internal Helper: `_manualEpsilon`

A Solidity re-implementation of the factory's `_computeEpsilon` private function, used to cross-check every `computeEpsilon` assertion:

```
ε = (maxRisk × SHARE_TO_USDC × MATH_SCALE) / (MATH_SCALE + (alpha × n × ln(n)) / MATH_SCALE)
```

Where:
- `SHARE_TO_USDC = 1e12` (6-dec → 18-dec conversion)
- `MATH_SCALE = 1e18`
- `ln(n)` values come from `_manualLn` — a copy of the factory's `_lnLookup` lookup table

This helper makes every exact-value test verifiable without relying on the contract under test as its own oracle.

---

## Test Groups

### 1. computeEpsilon — Exact Value Per n (7 tests)

Each valid outcome count gets its own exact-value assertion:

| Test | Outcome count | Approximate expected ε |
|---|---|---|
| `test_computeEpsilon_n2_exactValue` | n=2 | ~480e18 |
| `test_computeEpsilon_n3_exactValue` | n=3 | ~443e18 |
| `test_computeEpsilon_n4_exactValue` | n=4 | ~418e18 |
| `test_computeEpsilon_n5_exactValue` | n=5 | ~399e18 |
| `test_computeEpsilon_n6_exactValue` | n=6 | ~385e18 |
| `test_computeEpsilon_n7_exactValue` | n=7 | ~355e18 |
| `test_computeEpsilon_strictlyDecreasingWithN` | n=2..7 | `ε(n) > ε(n+1)` for all n |

For n=2 and n=7 (the two boundaries), tests additionally call `assertApproxEqRel` against the documented approximate values from `BlieverMarketBase.t.sol` as a sanity cross-check (±0.5% tolerance).

**Why it matters:** ε seeds every outcome slot of the AMM's initial quantity vector q⁰. An incorrect ε means the vault's worst-case loss bound `C(q⁰) = maxRiskPerMarket` is violated from block zero — the pool's risk guarantee fails silently.

**Why monotone-decreasing matters:** With a fixed risk budget R, more outcomes means each outcome gets a smaller seed. If ε increased with n, the total initial cost would exceed R, breaking the vault's liability accounting.

---

### 2. computeEpsilon — Bounds Reverts (4 tests)

| Test | n value | Revert |
|---|---|---|
| `test_computeEpsilon_reverts_n0` | 0 | `InvalidOutcomeCount(0)` |
| `test_computeEpsilon_reverts_n1` | 1 | `InvalidOutcomeCount(1)` |
| `test_computeEpsilon_reverts_n8` | 8 | `InvalidOutcomeCount(8)` |
| `test_computeEpsilon_reverts_maxUint8` | 255 | `InvalidOutcomeCount(255)` |

**Why it matters:** The `_lnLookup` function has no default case for n < 2 or n > 7 — it returns the n=7 value as a silent fallthrough. If the bounds check in `computeEpsilon` were absent or wrong, n=0 or n=255 would silently compute a nonsense epsilon and feed it to `deployMarket`.

---

### 3. computeEpsilon — Pool Parameter Sensitivity (2 tests)

| Test | What it checks |
|---|---|
| `test_computeEpsilon_doublingMaxRisk_doublesEpsilon` | ε is linear in R: `2×maxRisk → 2×ε` (verified with `assertApproxEqAbs(..., 1)` for integer rounding) |
| `test_computeEpsilon_higherAlpha_yieldsLowerEpsilon` | ε is inversely related to α: `2×alpha → ε_new < ε_old` |

**Setup:** Uses `mockPool.setAlpha()` and `mockPool.setMaxRisk()` — the factory reads pool parameters live on every `computeEpsilon` call (no caching at the view level).

**Why it matters:** These tests confirm the factory reads current pool state, not stale values. If pool parameters are adjusted post-factory-deployment (e.g., governance changes alpha), `computeEpsilon` must reflect the new values immediately.

---

### 4. computeEpsilon — Fuzz Tests (2 tests)

| Test | Fuzz input | Invariant |
|---|---|---|
| `testFuzz_computeEpsilon_alwaysMatchesFormula(n)` | `n ∈ [2,7]` | `computeEpsilon(n) == _manualEpsilon(n, ALPHA, MAX_RISK_USDC)` (exact) |
| `testFuzz_computeEpsilon_alwaysLessThanR(n)` | `n ∈ [2,7]` | `computeEpsilon(n) < R18` where `R18 = MAX_RISK_USDC × 1e12` |

The second fuzz test encodes the mathematical property that ε < R strictly: the denominator `(1 + α·n·ln n)` is always > 1 for any α > 0 and n ≥ 2, so ε < R always.

---

### 5. predictMarketAddress (5 tests)

| Test | What it checks |
|---|---|
| `test_predictMarketAddress_matchesDeployedAddress` | Predicted address equals actual deployed clone address |
| `test_predictMarketAddress_deterministicForSameQuestionId` | Calling twice with same questionId returns same address |
| `test_predictMarketAddress_differentForDifferentQuestionIds` | Two distinct questionIds → two distinct addresses |
| `test_predictMarketAddress_noBytecodeBeforeDeployment` | Predicted address has `code.length == 0` before deployment |
| `test_predictMarketAddress_hasBytecodeAfterDeployment` | After deployment, predicted address has `code.length == 45` (EIP-1167 fixed size) |

**Why the code-length check matters:** The factory uses `predicted.code.length > 0` as its duplicate-deployment guard inside `deployMarket`. Confirming that an undeployed address is empty and a deployed clone is exactly 45 bytes validates both the guard mechanism and the EIP-1167 proxy format.

---

### 6. predictMarketAddress — Fuzz Test (1 test)

| Test | Fuzz inputs | Invariant |
|---|---|---|
| `testFuzz_predictMarketAddress_uniquePerQuestionId(qId1, qId2)` | Any two distinct `bytes32` values | `predictMarketAddress(qId1) != predictMarketAddress(qId2)` |

Uses `vm.assume(qId1 != qId2)`. Confirms the CREATE2 salt derivation `keccak256(abi.encode(questionId))` produces unique salts for all distinct inputs — no hash collision at the Solidity level.

---

## What Is NOT Tested Here

- Internal `_lnLookup` directly (it is private and tested indirectly via `computeEpsilon`)
- `computeEpsilon` with zero alpha (if `alpha = 0`, denominator = 1, ε = R; the factory does not prevent this — add a test if pool governance allows alpha=0)
- `predictMarketAddress` for a factory deployed at a different address (address is factory-specific by CREATE2 design)

---

## Improvement / Debugging Notes

- If `_lnLookup` values are ever updated (e.g., for higher precision), update `_manualLn` in this test file to match and re-run the exact-value tests. The fuzz tests will catch formula drift automatically.
- `test_computeEpsilon_doublingMaxRisk_doublesEpsilon` uses `assertApproxEqAbs(..., 1)` to allow for one wei of integer rounding. If the formula ever changes to include a non-linear R term, this test will fail — which is the intended behavior.
- The 45-byte EIP-1167 proxy length check in `test_predictMarketAddress_hasBytecodeAfterDeployment` is a constant defined by the EIP itself. If OpenZeppelin ever changes the clone bytecode format, this test will catch it.
- Alpha and maxRisk sensitivity tests (`test_computeEpsilon_doublingMaxRisk_doublesEpsilon`, `test_computeEpsilon_higherAlpha_yieldsLowerEpsilon`) reset pool state after each test via `setUp()`. They do not restore pool state mid-test — safe because each test function gets a fresh `setUp()`.
