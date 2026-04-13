# BlieverMarketFactory — Developer Implementation Reference

> **Document type:** Technical implementation explanation.
> **Audience:** Developers who want to analyze, improve, debug, test, or integrate the contract.
> **Companion:** `BlieverMarketFactory.md` covers the theoretical architecture and design rationale.

---

## Contract Map

```
contracts/src/
├── BlieverMarketFactory.sol        ← This contract
├── interfaces/
│   ├── IBlieverV1Pool.sol          ← pool.registerMarket, pool.deregisterMarket, pool.alpha, pool.maxRiskPerMarket
│   └── IBlieverUmaAdapter.sol      ← adapter.initializeQuestion
└── (local, defined inline in BlieverMarketFactory.sol)
    └── IDeployableMarket           ← initialize, pause, unpause, expireUnresolved, view getters
```

`IDeployableMarket` is a factory-specific interface defined at the top of `BlieverMarketFactory.sol`. It covers the BlieverMarket functions that **only the factory calls**. It does not duplicate the adapter-facing methods (`resolve`, `questionId`, `outcomeCount`) already in `IBlieverMarket.sol`.

---

## 1. Storage Layout

`BlieverMarketFactory` is **not upgradeable** — there is no storage gap or proxy concern.

| Variable | Type | Slot (approx.) | Description |
|---|---|---|---|
| *(AccessControl internals)* | `mapping(bytes32 => RoleData)` | 0–1 | OZ role registry |
| *(Pausable internal)* | `bool _paused` | 2 | packed into slot |
| *(ReentrancyGuard status)* | `uint256 _status` | 3 | reentrancy lock |
| `isDeployedMarket` | `mapping(address => bool)` | 4 | factory origin tracking |
| `marketCount` | `uint256` | 5 | deployment counter |

**Immutables** (not in storage — embedded into bytecode at construction):

| Immutable | Type | Set in constructor |
|---|---|---|
| `implementation` | `address` | `_implementation` parameter |
| `pool` | `IBlieverV1Pool` | `_pool` parameter |
| `adapter` | `IBlieverUmaAdapter` | `_adapter` parameter |

---

## 2. Role Summary

```
DEFAULT_ADMIN_ROLE  keccak256("DEFAULT_ADMIN_ROLE") == 0x000...0 (OZ default)
OPERATOR_ROLE       keccak256("OPERATOR_ROLE")
PAUSER_ROLE         keccak256("PAUSER_ROLE")
```

Role assignment in constructor:
```solidity
_grantRole(DEFAULT_ADMIN_ROLE, admin);
_grantRole(OPERATOR_ROLE,      admin);
_grantRole(PAUSER_ROLE,        admin);
```

> In production: transfer OPERATOR_ROLE and PAUSER_ROLE to separate multisigs post-deployment. DEFAULT_ADMIN_ROLE should remain with the governance multisig (highest quorum / timelock).

---

## 3. Constants

| Name | Value | Purpose |
|---|---|---|
| `MIN_OUTCOMES` | `2` | Binary market minimum (LS-LMSR requires ≥ 2 outcomes) |
| `MAX_OUTCOMES` | `7` | UMIP-183 cap — MultiValueDecoder encodes max 7 labels per int256 price |
| `MATH_SCALE` | `1e18` | LSMath fixed-point scale (mirrors BlieverMarket.MATH_SCALE) |
| `SHARE_TO_USDC` | `1e12` | 18-dec shares → 6-dec USDC (mirrors BlieverMarket.SHARE_TO_USDC) |

---

## 4. `IDeployableMarket` — Interface Specification

Defined inline at the top of `BlieverMarketFactory.sol`. All methods are `onlyFactory` guarded in `BlieverMarket`.

```solidity
interface IDeployableMarket {
    function initialize(
        address _pool,
        bytes32 _questionId,
        uint8   _nOutcomes,
        uint256 _alpha,
        uint40  _tradingDeadline,
        uint40  _resolutionDeadline,
        uint256 _epsilon,
        address _resolver,
        address _factory
    ) external;

    function pause()   external;
    function unpause() external;
    function expireUnresolved() external;

    function resolutionDeadline() external view returns (uint40);
    function resolved()           external view returns (bool);
    function questionId()         external view returns (bytes32);
}
```

**Parameter order in `initialize` matches `BlieverMarket.sol` exactly** — any mismatch produces a silent ABI misdecode (wrong storage slots written). If `BlieverMarket.initialize` ever changes parameter order, this interface must be updated atomically.

---

## 5. `DeployParams` Struct — Calldata Layout

The struct is passed as `calldata` to `deployMarket`. On Base chain, calldata byte count directly affects L1 data-availability fees, so non-zero bytes are minimized.

```
Field              Type       Bytes  Notes
─────────────────────────────────────────────────────────
questionId         bytes32    32     keccak256(fullAncillaryData)
ancillaryData      bytes      dyn    raw JSON (≤ 8 086 bytes)
nOutcomes          uint8      1      [2, 7]
tradingDeadline    uint40     5      Unix timestamp
resolutionDeadline uint40     5      Unix timestamp > tradingDeadline
rewardToken        address    20     OO reward token (zero if reward==0)
reward             uint256    32     OO proposer reward
bond               uint256    32     OO proposer/disputer bond
liveness           uint256    32     OO liveness window (seconds)
salt               bytes32    32     CREATE2 salt
```

`ancillaryData` is a dynamic `bytes` field. When encoding off-chain (ethers.js, viem), it appears as a pointer + length in the head section of the ABI encoding.

---

## 6. `deployMarket()` — Execution Trace

### Pre-flight checks (gas before state)

All eight validation conditions are checked before any token transfer or state mutation. The recommended pattern: **revert as early as possible with the cheapest check first**.

```
questionId == bytes32(0)                  → InvalidQuestionId
nOutcomes out of [MIN, MAX]              → InvalidOutcomeCount
deadline logic fails                     → InvalidDeadlines
reward > 0 && rewardToken == address(0) → RewardTokenRequired
predicted.code.length > 0              → SaltAlreadyUsed
```

The salt-collision check (`Clones.predictDeterministicAddress`) costs approximately 500 gas (one EXTCODESIZE opcode). It is positioned last among the pure-validation checks because it makes an external call — cheaper than deploying a clone only to get a CREATE2 collision deep inside Clones.cloneDeterministic.

### Epsilon computation

```solidity
uint256 epsilon = computeEpsilon(params.nOutcomes);
```

Reads `pool.alpha()` and `pool.maxRiskPerMarket()` (two warm SLOADs on the pool proxy — ~200 gas each). Performs integer arithmetic only (no external math library). If the pool returns `alpha = 0` or `maxRiskPerMarket = 0` due to misconfiguration, epsilon may be 0 — caught by the `ZeroEpsilon` guard immediately after.

### Token pull

```solidity
IERC20(params.rewardToken).safeTransferFrom(msg.sender, address(this), params.reward);
```

Happens **before** clone deployment. If the caller's allowance is insufficient or their balance too low, the transaction reverts cleanly with no orphaned clone on-chain.

### Clone deployment

```solidity
market = Clones.cloneDeterministic(implementation, params.salt);
```

Deploys the 45-byte EIP-1167 proxy. The OZ `Clones` library uses inline assembly with the `CREATE2` opcode. The proxy bytecode contains the `implementation` address hard-coded at bytes 10–29.

After this line, `market` contains the new clone's address. The address equals the value returned by `predictMarketAddress(params.salt)`.

### Clone initialization

```solidity
IDeployableMarket(market).initialize(
    address(pool),
    params.questionId,
    params.nOutcomes,
    pool.alpha(),             // ← second pool.alpha() call; snapshot for immutability
    params.tradingDeadline,
    params.resolutionDeadline,
    epsilon,
    address(adapter),
    address(this)
);
```

`pool.alpha()` is called a second time here (separately from `computeEpsilon`) to guarantee the clone receives the identical alpha value that ε was computed for. If `pool.alpha()` were to change between the two calls (governance transaction in the same block) there would be a mismatch. In practice this is not possible on Base (single-threaded block execution), but using a cached local variable would be slightly more explicit. The current design prioritizes readability over micro-optimization here.

`initialize()` internally calls:
- `LSMath.liquidityParameter(initQ, _alpha)` — validates ε produces a non-degenerate market.
- `LSMath.costFunction(initQ, _alpha)` — caches `_initialCost = C(q⁰)` for use in liability tracking.

Both calls are pure (no storage writes to the pool). If they revert (e.g., ε too small for the liquidity formula), the entire `deployMarket` transaction reverts.

### Pool registration

```solidity
pool.registerMarket(market, uint32(params.nOutcomes));
```

`registerMarket` does the following inside `BlieverV1Pool`:
1. Validates: market is a contract, nOutcomes ∈ [2, 100], not already registered, active market cap not exceeded.
2. Creates `MarketInfo` with `riskBudget = pool.maxRiskPerMarket` and `currentLiability = riskBudget`.
3. Adds `riskBudget` to `totalLiability`.
4. Increments `activeMarketCount`.
5. Grants `MARKET_ROLE` to `market`.

The `uint32` cast on `params.nOutcomes` is safe: `nOutcomes ∈ [2, 7]` ⊆ `uint32` range.

Factory must hold `MARKET_MANAGER_ROLE` on the pool for this call to succeed.

### Adapter question initialization

```solidity
if (params.reward > 0) {
    IERC20(params.rewardToken).forceApprove(address(adapter), params.reward);
}
adapter.initializeQuestion(
    params.questionId, market, params.ancillaryData,
    params.rewardToken, params.reward, params.bond, params.liveness
);
```

`forceApprove` (SafeERC20) resets any existing allowance to zero first, then sets it to `params.reward`. This protects against ERC-20 tokens that reject non-zero-to-non-zero approval updates (e.g., legacy USDT).

Inside `adapter.initializeQuestion`:
1. Validates `FACTORY_ROLE` of caller.
2. Appends `,initializer:<factory_hex>` to `ancillaryData` via `AncillaryDataLib`.
3. Derives `fullQuestionId = keccak256(fullAncillaryData)` and verifies it equals `params.questionId`.
4. Verifies `market.questionId() == params.questionId` (set in `initialize` step).
5. Writes `QuestionData` to storage.
6. Submits a `MULTIPLE_VALUES` price request to the UMA Optimistic Oracle.
7. Pulls the `reward` tokens from the factory (using the allowance set above).

### State recording

```solidity
isDeployedMarket[market] = true;
unchecked { ++marketCount; }
```

State updates are placed last (post-interactions). This is intentional and safe because:
- `nonReentrant` is applied at the function level — no reentrancy vector exists.
- The pool and adapter are trusted protocol contracts; neither re-enters `deployMarket`.
- The clone cannot call back to the factory during its own `initialize`.

`unchecked` on `marketCount` increment: `uint256` exhaustion is physically infeasible (2^256 markets cannot be deployed).

---

## 7. `computeEpsilon()` — Math Implementation

```solidity
uint256 R18         = maxRisk * SHARE_TO_USDC;
uint256 lnN         = _lnLookup(nOutcomes);
uint256 denominator = MATH_SCALE + (alpha_ * nOutcomes * lnN) / MATH_SCALE;
uint256 epsilon     = (R18 * MATH_SCALE) / denominator;
```

### Step-by-step with example values

Pool parameters: `alpha = 3e16` (3%), `maxRiskPerMarket = 1_000_000` (= $1 USDC, 6-dec), `nOutcomes = 2`.

```
R18         = 1_000_000 * 1e12                    = 1e18
lnN         = _lnLookup(2)                         = 693_147_180_559_945_309
alpha*n*lnN = 3e16 * 2 * 693_147_180_559_945_309  = 4.158...e34
÷ MATH_SCALE= 4.158...e34 / 1e18                  = 41_588_830_833_596_718
denominator = 1e18 + 41_588_830_833_596_718       = 1_041_588_830_833_596_718
R18*SCALE   = 1e18 * 1e18                         = 1e36
epsilon     = 1e36 / 1_041_588_830_833_596_718    ≈ 9.601e17
```

So ε ≈ 9.601e17 (18-dec), representing approximately 0.96 shares per outcome. This means `C([0.96, 0.96]) / 1e12 ≈ $1.00 USDC = maxRiskPerMarket`. ✓

### Overflow bounds verification

| Expression | Max value | Comparison |
|---|---|---|
| `alpha_ * nOutcomes * lnN` | `2e17 * 7 * 1.946e18 ≈ 2.72e36` | `< 2^256 ≈ 1.16e77` ✓ |
| `R18 * MATH_SCALE` (with maxRisk = 1e12 USDC) | `1e12 * 1e12 * 1e18 = 1e42` | `< 2^256` ✓ |

### `_lnLookup` dispatch

The function uses linear `if` chains over 6 integers. The Solidity compiler typically converts this to a `JUMPI` chain. For n ∈ [2, 7], the maximum is 5 conditional branches (n=7 falls through to the final `return`).

---

## 8. `expireUnresolved()` — Execution Trace

```
1. _assertDeployedMarket(market)      ← mapping check; reverts NotDeployedMarket if false
2. m.resolved()                       ← warm SLOAD on clone; reverts MarketAlreadyResolved if true
3. m.resolutionDeadline()             ← warm SLOAD on clone
4. block.timestamp <= deadline         ← reverts ResolutionDeadlineNotPassed
5. m.expireUnresolved()               ← clone calls pool.settleMarket(0)
6. emit MarketExpiredByFactory(...)   ← includes m.questionId() (warm SLOAD)
```

This function is **permissionless** — any EOA may call it after the deadline. The `_assertDeployedMarket` guard ensures only factory-deployed markets can be expired, preventing the factory from being used as a generic lifecycle controller for arbitrary contracts.

Inside `BlieverMarket.expireUnresolved()`:
- Validates: `onlyFactory` modifier (checks `msg.sender == factory`).
- Calls `IBlieverV1Pool(pool).settleMarket(0)` — settledPayout = 0, MARKET_ROLE revoked.
- Emits `MarketExpired`.

Inside `BlieverV1Pool.settleMarket(0)`:
- Subtracts `currentLiability` from `totalLiability`.
- Decrements `activeMarketCount`.
- Records `settledPayout = 0`.
- Revokes `MARKET_ROLE` from the market.
- Emits `MarketSettled(market, 0, riskBudget)` — entire riskBudget becomes LP profit.

---

## 9. Modifier and Guard Summary

| Function | Modifiers / Guards | Notes |
|---|---|---|
| `deployMarket` | `nonReentrant`, `whenNotPaused`, `onlyRole(OPERATOR_ROLE)` | Atomic 5-step pipeline |
| `expireUnresolved` | `nonReentrant`, `_assertDeployedMarket` | Permissionless after deadline |
| `pauseMarket` | `onlyRole(PAUSER_ROLE)`, `_assertDeployedMarket` | Forwarded to clone |
| `unpauseMarket` | `onlyRole(PAUSER_ROLE)`, `_assertDeployedMarket` | Forwarded to clone |
| `deregisterMarket` | `onlyRole(DEFAULT_ADMIN_ROLE)`, `_assertDeployedMarket` | Pool enforces hasTrades check |
| `pause` (factory) | `onlyRole(PAUSER_ROLE)` | Blocks deployMarket only |
| `unpause` (factory) | `onlyRole(PAUSER_ROLE)` | Resumes deployMarket |
| `predictMarketAddress` | — | Pure view, no state |
| `computeEpsilon` | — | `pool.alpha()` + `pool.maxRiskPerMarket()` reads |

---

## 10. External Role Dependencies

The factory requires two external role grants from other protocol admins before `deployMarket` can succeed:

```
pool.grantRole(pool.MARKET_MANAGER_ROLE(), address(factory))
```
Required for: `pool.registerMarket()` (step 6 in deployMarket), `pool.deregisterMarket()` (in factory.deregisterMarket).

```
adapter.grantRole(adapter.FACTORY_ROLE(), address(factory))
```
Required for: `adapter.initializeQuestion()` (step 7 in deployMarket).

Without either grant, `deployMarket` reverts with an `AccessControl` error at the respective step. Because the token pull (step 3) occurs before the pool call (step 6), a missing `MARKET_MANAGER_ROLE` means the reward tokens are temporarily pulled into the factory — but the EVM rolls back the entire transaction, so no tokens are permanently stuck.

**Recommended pre-deployment test:**
1. Deploy factory.
2. Grant both roles.
3. Call `factory.deployMarket(params)` on a fork with `vm.prank`.
4. Verify `pool.isActiveMarket(market) == true`.
5. Verify `adapter.isInitialized(params.questionId) == true`.
6. Verify `market.resolved() == false`.

---

## 11. Key Error Conditions and Debug Guide

| Error | Cause | Fix |
|---|---|---|
| `InvalidQuestionId` | `params.questionId == bytes32(0)` | Pre-compute questionId off-chain correctly |
| `InvalidOutcomeCount(n)` | `n < 2` or `n > 7` | Use nOutcomes ∈ [2, 7]; n > 7 is an adapter cap |
| `InvalidDeadlines` | tradingDeadline ≥ resolutionDeadline OR tradingDeadline in past | Check deadline ordering; ensure tradingDeadline > block.timestamp |
| `RewardTokenRequired` | `reward > 0 && rewardToken == address(0)` | Provide a valid rewardToken address |
| `SaltAlreadyUsed(salt, existing)` | CREATE2 collision; same salt used before | Derive salt from questionId: `keccak256(abi.encode(questionId))` |
| `ZeroEpsilon` | Pool misconfiguration (alpha=0 or maxRisk=0) | Check pool state; contact pool admin |
| `NotDeployedMarket(market)` | `isDeployedMarket[market] == false` | Only markets deployed by this factory can be managed |
| `MarketAlreadyResolved(market)` | `expireUnresolved` called on a resolved market | Check `market.resolved()` first |
| `ResolutionDeadlineNotPassed` | `expireUnresolved` called too early | Wait until `block.timestamp > market.resolutionDeadline()` |
| `NotAContract(_implementation)` | implementation address has no bytecode | Deploy `BlieverMarket` master first |
| Pool: `AccessControl` error | Factory missing `MARKET_MANAGER_ROLE` on pool | `pool.grantRole(pool.MARKET_MANAGER_ROLE(), factory)` |
| Adapter: `AccessControl` error | Factory missing `FACTORY_ROLE` on adapter | `adapter.grantRole(adapter.FACTORY_ROLE(), factory)` |
| Adapter: `QuestionIdMismatch` | Off-chain questionId computation is wrong | Re-derive: `keccak256(rawData ++ ",initializer:" ++ lowerCaseHex(factory))` |
| Adapter: `InvalidOutcomeCount` | nOutcomes > 7 (UMIP-183 cap) | Redundant guard — factory caps at 7 before calling adapter |
| Market: `ZeroEpsilon` | `_epsilon` param to initialize was 0 | Factory-computed epsilon rounds to 0 due to extreme pool params |
| Market: `InvalidAlpha` | `pool.alpha()` is outside `[LSMath.MIN_ALPHA, LSMath.MAX_ALPHA]` | Pool admin must set alpha in valid range before deploying markets |

---

## 12. Gas Profile (Base Chain, L2 execution only)

| Step | Approx. gas | Dominant cost |
|---|---|---|
| Pre-flight checks | ~5 000 | EXTCODESIZE for salt collision check |
| `computeEpsilon` | ~3 000 | 2× warm SLOADs on pool, integer math |
| Token pull (`safeTransferFrom`) | ~30 000 | ERC-20 transfer + allowance check |
| `cloneDeterministic` | ~41 000 | CREATE2 opcode, 45-byte bytecode init |
| `market.initialize()` | ~120 000 | n SSTORE slots (q-vector) + Pausable init |
| `pool.registerMarket()` | ~60 000 | MarketInfo struct SSTORE + MARKET_ROLE grant |
| `adapter.initializeQuestion()` | ~250 000 | ancillaryData concat + OO price request (dominant) |
| Factory state update + event | ~3 000 | 1 SSTORE + 1 LOG |
| **Total** | **~512 000** | Adapter step is the bottleneck |

L1 data-availability fee (Base rollup) depends on calldata byte count. For typical ancillaryData (~500 bytes):
- Non-zero calldata bytes ≈ 400 × 16 gas/byte ≈ 6 400 L2-equiv gas at the L1 calldata cost.
- Using `DeployParams` as a calldata struct avoids redundant ABI encoding overhead from individual arguments.

---

## 13. Off-Chain Integration Guide

### Computing questionId (TypeScript / ethers.js)

```typescript
import { utils } from "ethers";

const rawAncillaryData = utils.toUtf8Bytes(jsonString);  // raw JSON
const suffix = utils.toUtf8Bytes(
  ",initializer:" + factoryAddress.slice(2).toLowerCase()
);
const fullData   = utils.concat([rawAncillaryData, suffix]);
const questionId = utils.keccak256(fullData);
```

### Computing the CREATE2 salt

```typescript
const salt = utils.keccak256(utils.defaultAbiCoder.encode(["bytes32"], [questionId]));
```

### Pre-computing market address

```typescript
const market = await factory.predictMarketAddress(salt);
// market address is deterministic — usable before deployment tx lands
```

### Pre-computing epsilon (off-chain cross-check)

```typescript
const epsilon = await factory.computeEpsilon(nOutcomes);
// Compare with your off-chain calculation of R / (1 + α·n·ln n) for sanity check
```

### Approving reward tokens before deployMarket

```typescript
await rewardToken.approve(factory.address, params.reward);
// THEN call deployMarket
const tx = await factory.deployMarket(params);
const receipt = await tx.wait();
// Parse MarketDeployed event from receipt to get clone address
```

### Reading the MarketDeployed event

```typescript
const iface = new utils.Interface([
  "event MarketDeployed(address indexed market, bytes32 indexed questionId, uint8 nOutcomes, uint40 tradingDeadline, uint40 resolutionDeadline, bytes32 salt)"
]);
const log   = receipt.logs.find(l => l.topics[0] === iface.getEventTopic("MarketDeployed"));
const event = iface.parseLog(log);
console.log("Deployed market:", event.args.market);
```

---

## 14. UUPS Upgrade Notes (Pool and Adapter)

The factory stores `pool` and `adapter` as immutables pointing at the **proxy addresses** of those contracts. When the pool or adapter is upgraded via UUPS:
- The proxy addresses remain unchanged.
- The implementation logic changes transparently.
- The factory continues calling the same proxy addresses.
- No factory changes are required.

If the **interface** of pool or adapter changes (function signatures or selectors), the factory must be redeployed.

---

## 15. Deployment Checklist

Before going live on Base mainnet:

- [ ] Deploy `BlieverMarket` master implementation → note `_implementation` address.
- [ ] Deploy `BlieverV1Pool` proxy (UUPS, with correct USDC address and initial params).
- [ ] Deploy `BlieverUmaAdapter` proxy (UUPS, with OO address and reward defaults).
- [ ] Deploy `BlieverMarketFactory(implementation, pool, adapter, adminMultisig)`.
- [ ] Pool admin: `pool.grantRole(pool.MARKET_MANAGER_ROLE(), factory)`.
- [ ] Adapter admin: `adapter.grantRole(adapter.FACTORY_ROLE(), factory)`.
- [ ] Factory admin: transfer `OPERATOR_ROLE` to operator multisig.
- [ ] Factory admin: transfer `PAUSER_ROLE` to fast-response multisig.
- [ ] Run a fork test deploying one market end-to-end and verifying all state invariants.
- [ ] Obtain a professional security audit before handling real LP funds.
