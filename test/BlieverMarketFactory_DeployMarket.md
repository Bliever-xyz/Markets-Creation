# BlieverMarketFactory — deployMarket Tests

**File:** `contracts/test/BlieverMarketFactory/BlieverMarketFactory_DeployMarket.t.sol`  
**Contract under test:** `BlieverMarketFactory.deployMarket(DeployParams calldata)`  
**Base:** extends `FactoryTestBase`

---

## What This File Tests

`deployMarket` is the factory's core function. It executes eight sequential steps atomically:
1. Input validation (fail-fast, no state mutation)
2. Compute LS-LMSR ε from pool parameters
3. Pull OO reward tokens from caller (if reward > 0)
4. Deploy EIP-1167 clone via CREATE2
5. Initialize the clone
6. Register clone in pool
7. Approve adapter + initialize oracle question
8. Record deployment state, emit event

All tests use `MockFactoryPool` and `MockUmaAdapter` — external call behavior is verified by spy counters, not real protocol contracts.

---

## Test Groups

### 1. Input Validation Reverts (7 tests)

Each pre-flight check in step 1 is tested in isolation:

| Test | Revert | Condition tested |
|---|---|---|
| `test_deployMarket_reverts_zeroQuestionId` | `InvalidQuestionId` | `questionId == bytes32(0)` |
| `test_deployMarket_reverts_emptyAncillaryData` | `InvalidAncillaryData` | `ancillaryData.length == 0` |
| `test_deployMarket_reverts_outcomesBelow2` | `InvalidOutcomeCount(1)` | `nOutcomes < MIN_OUTCOMES` |
| `test_deployMarket_reverts_outcomesAbove7` | `InvalidOutcomeCount(8)` | `nOutcomes > MAX_OUTCOMES` |
| `test_deployMarket_reverts_tradingDeadlineInPast` | `InvalidDeadlines` | `tradingDeadline == block.timestamp` (not strictly future) |
| `test_deployMarket_reverts_tradingDeadlineEqualToResolution` | `InvalidDeadlines` | `tradingDeadline == resolutionDeadline` |
| `test_deployMarket_reverts_tradingDeadlineAfterResolution` | `InvalidDeadlines` | `tradingDeadline > resolutionDeadline` |
| `test_deployMarket_reverts_rewardWithoutToken` | `RewardTokenRequired` | `reward > 0 && rewardToken == address(0)` |

**Why it matters:** These guards fire before any state change or external call. A missing guard could result in an unusable clone (bad ε, wrong deadlines) that is still registered in the pool and cannot be fixed post-deployment.

---

### 2. Access Control (2 tests)

| Test | What it checks |
|---|---|
| `test_deployMarket_reverts_nonOperator` | `attacker` (no role) calling `deployMarket` reverts |
| `test_deployMarket_reverts_whenPaused` | `operator` calling `deployMarket` while factory is paused reverts |

**Why it matters:** `deployMarket` must be gated by `OPERATOR_ROLE` and `whenNotPaused`. A missing guard would allow anyone to deploy arbitrary markets under any questionId.

---

### 3. Happy Path — Binary Market (n=2) (6 tests)

The canonical 2-outcome prediction market:

| Test | What it checks |
|---|---|
| `test_deployMarket_2outcomes_returnsNonZeroAddress` | Returned address is non-zero |
| `test_deployMarket_2outcomes_isDeployedMarketIsTrue` | `isDeployedMarket[market] == true` after deployment |
| `test_deployMarket_2outcomes_marketCountIncrements` | `marketCount` goes 0→1→2 across two deployments |
| `test_deployMarket_2outcomes_emitsMarketDeployed` | `MarketDeployed` event with correct indexed + non-indexed fields |
| `test_deployMarket_2outcomes_poolRegisterMarketCalled` | `pool.registerMarket` called once with `nOutcomes=2` |
| `test_deployMarket_2outcomes_adapterInitializeQuestionCalled` | `adapter.initializeQuestion` called once with correct questionId |
| `test_deployMarket_2outcomes_adapterReceivesAncillaryData` | `adapter` receives exact ancillaryData bytes |

---

### 4. Happy Path — Multi-Outcome (n=7) (2 tests)

| Test | What it checks |
|---|---|
| `test_deployMarket_7outcomes_success` | All three invariants hold: isDeployedMarket, registerCalls=1, initCalls=1 |
| `test_deployMarket_7outcomes_marketCountIncrements` | marketCount increments to 1 |

**Why it matters:** n=7 is the UMIP-183 maximum. Testing the boundary ensures the epsilon denominator computation and the pool registration both handle the largest valid input.

---

### 5. Clone State After Initialization (4 tests)

Verifies that the deployed clone has correctly stored fields from `DeployParams`:

| Test | Field checked |
|---|---|
| `test_deployMarket_clone_questionIdMatches` | `IDeployableMarket(market).questionId()` |
| `test_deployMarket_clone_resolvedIsFalse` | `IDeployableMarket(market).resolved()` — must be false at birth |
| `test_deployMarket_clone_resolutionDeadlineIsSet` | `IDeployableMarket(market).resolutionDeadline()` matches `DeployParams.resolutionDeadline` |
| `test_deployMarket_clone_outcomeCountMatches` | `IDeployableMarket(market).outcomeCount()` — tested for both n=2 and n=7 |

---

### 6. CREATE2 Address Prediction (2 tests)

| Test | What it checks |
|---|---|
| `test_deployMarket_matchesPredictedAddress` | `predictMarketAddress(qId)` called before deploy equals the deployed address |
| `test_deployMarket_differentQuestionIds_differentAddresses` | Two distinct questionIds predict two distinct addresses |

**Why it matters:** Frontends and order engines pre-compute the clone address before deployment. A mismatch would point them to the wrong contract.

---

### 7. Duplicate Question Guard (2 tests)

| Test | What it checks |
|---|---|
| `test_deployMarket_reverts_duplicateQuestionId` | Second deploy with same questionId reverts with `QuestionAlreadyDeployed(qId, existing)` |
| `test_deployMarket_duplicate_doesNotIncrementMarketCount` | `marketCount` stays at 1 after a reverted duplicate attempt |

**Why it matters:** One questionId must map to exactly one market. A duplicate would break oracle question uniqueness and potentially allow a malicious operator to override an existing market's resolution.

---

### 8. Reward Token Flow (5 tests)

Tests the ERC-20 pull and approval path when `params.reward > 0`:

| Test | What it checks |
|---|---|
| `test_deployMarket_withReward_pullsTokensFromOperator` | Operator's token balance decreases by `reward` |
| `test_deployMarket_withReward_factoryHoldsTokens` | Factory balance equals `reward` (mock adapter does not pull) |
| `test_deployMarket_withReward_approvedAdapterForReward` | `rewardToken.allowance(factory, adapter) == reward` after deploy |
| `test_deployMarket_withReward_adapterReceivesRewardParams` | Adapter spy records correct `rewardToken` and `reward` values |
| `test_deployMarket_withReward_reverts_insufficientAllowance` | Missing approval → ERC-20 reverts → whole tx reverts (no orphaned clone) |
| `test_deployMarket_zeroReward_doesNotPullTokens` | With `reward=0`, no ERC-20 interactions occur |

---

### 9. Pool Registration Args (2 tests)

| Test | What it checks |
|---|---|
| `test_deployMarket_poolReceivesCloneAddress` | `pool.lastRegisteredMarket` equals the predicted clone address |
| `test_deployMarket_poolReceivesNOutcomes` | `pool.lastRegisteredOutcomes` equals 7 when n=7 |

---

### 10. Fuzz Tests (3 tests)

| Test | Fuzz inputs | Invariant |
|---|---|---|
| `testFuzz_deployMarket_allValidOutcomes(n)` | `n ∈ [2,7]` | Deployment succeeds, `isDeployedMarket == true`, `registerCalls == 1` |
| `testFuzz_deployMarket_marketCountIsMonotone(count)` | `count ∈ [1,7]` | After `count` deployments, `marketCount == count` |
| `testFuzz_deployMarket_deadlineValidation(tOffset, rOffset)` | `tOffset ∈ [1, 365d]`, `rOffset ∈ [tOffset+1, 730d]` | Valid deadline pairs always succeed |

---

## What Is NOT Tested Here

- Constructor arguments (→ Constructor tests)
- expireUnresolved / pauseMarket / deregisterMarket (→ Lifecycle tests)
- computeEpsilon math precision (→ Views tests)

---

## Improvement / Debugging Notes

- If `_computeEpsilon` returns 0 (e.g., pool returns maxRisk=0), deployMarket reverts with `ZeroEpsilon`. Add a test for this by calling `mockPool.setMaxRisk(0)` before deploy.
- Reward token tests use `MockRewardToken` (18-dec OZ ERC-20). Tests for 6-dec reward tokens (e.g., USDC as reward) may expose decimal-precision edge cases worth testing separately.
- The `test_deployMarket_reverts_whenPaused` test confirms `whenNotPaused` but not `nonReentrant`. Reentrancy on deployMarket would require a malicious pool or adapter mock — add if threat model requires it.
