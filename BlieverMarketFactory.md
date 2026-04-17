# BlieverMarketFactory — Architecture & Design Reference

> **Document type:** Theoretical architecture and system-design rationale.
> **Audience:** Anyone who wants to understand **what** the contract does and **why** it is designed this way — analysts, auditors, protocol researchers, and non-developer stakeholders.
> **Companion:** `BlieverMarketFactory_DEV.md` covers the technical implementation detail for developers who need to analyze, improve, debug, or test the contract.

---

## 1. Role in the Bliever Protocol

The Bliever prediction-market stack is composed of four discrete components. Each one has a single, tightly scoped responsibility:

```
┌──────────────────────────────────────────────────────────────────┐
│  BlieverV1Pool      Global USDC vault (LS-LMSR liability hub)    │
│  BlieverMarket      LS-LMSR AMM per prediction event (clone)     │
│  BlieverUmaAdapter  UMA oracle resolution bridge (UUPS proxy)    │
│  BlieverMarketFactory  ← you are here                            │
└──────────────────────────────────────────────────────────────────┘
```

The factory is the **sole authorized entry point** for creating new prediction markets. Its job is to:

1. Deploy an EIP-1167 minimal proxy clone of the `BlieverMarket` master implementation.
2. Initialize the clone with its market-specific configuration.
3. Register the clone in `BlieverV1Pool` (reserving the LS-LMSR risk budget).
4. Initialize the UMA oracle question in `BlieverUmaAdapter` (submitting the price request).

These four steps execute atomically inside a single `deployMarket()` transaction. Either all succeed or the entire transaction reverts — there is no "partially deployed" market state possible.

---

## 2. The Market Curation Model

The factory enforces a **team-curated market model** inspired by Polymarket:

> *"While users cannot directly create their own markets, they are encouraged to suggest ideas for new markets. To give your proposal the best chance of being listed, include as much information as possible — via X or other social channels. After review, the Bliever team creates and lists the market."*

Only addresses holding `OPERATOR_ROLE` may call `deployMarket()`. This prevents spam markets, poorly constructed oracle questions, and adversarial use of LP capital. End-users engage with the protocol as traders; the team curates the market catalog.

---

## 3. Why EIP-1167 Minimal Proxy?

### The problem with full deployment
Deploying a full `BlieverMarket` contract for every prediction event (elections, sports results, macroeconomic indicators) would cost approximately **500 000 gas per market** due to the 24 KB maximum bytecode size. At $0.01 per gas unit on Base, this is $5 per market — infeasible for a platform targeting thousands of simultaneous markets.

### The EIP-1167 solution
The EIP-1167 standard deploys a **45-byte proxy shell** that delegates all logic execution to a single shared `implementation` contract. This shell costs approximately **41 000 gas** to deploy — more than a 90% reduction.

```
Proxy shell bytecode:
  363d3d373d3d3d363d73<ImplementationAddress>5af43d82803e903d91602b57fd5bf3
                        ↑ 20-byte master address embedded at offset 10
```

When a trader calls `buy()` on a deployed market clone:
1. The clone receives the calldata.
2. Its 45-byte body executes `DELEGATECALL` to the master `implementation`.
3. The master's logic runs, but **all state changes write to the clone's storage**, not the master's.

This means every cloned market has:
- **Shared code** — zero bytecode duplication.
- **Isolated state** — its own `questionId`, `q-vector`, trader share ledger, deadlines, and resolution status.

### Why the master is immutable (non-upgradeable)
Individual market clones are **immutable by design**. Traders require a cryptographic guarantee that the mathematical rules and resolution parameters of a specific market cannot be altered mid-trade by an admin key. Adding UUPS upgrade logic to the master would allow a compromised admin to silently change the AMM math for every deployed market simultaneously — a catastrophic trust violation. Immutability is a security feature, not a limitation.

---

## 4. Deterministic Addressing via CREATE2

The factory deploys every clone using the `CREATE2` opcode instead of the standard `CREATE`. The key property: **the clone's address is computable before the transaction is broadcast**.

```
salt           = keccak256(abi.encode(questionId))
market_address = keccak256(0xff ++ factory_address ++ salt ++ keccak256(proxy_bytecode))[12:]
```

The salt is derived **on-chain inside `deployMarket`** from `params.questionId`. Callers do not supply a salt. This means:

- **Cryptographically enforced bijection**: Each unique oracle question maps to exactly one deterministic market address. No operator can accidentally deploy two markets for the same question, and no duplicate or mismatched salt is possible.
- **Simplified off-chain integration**: Frontends and indexers call `predictMarketAddress(questionId)` directly — no separate salt computation step is required.
- **Lazy deployment / pre-routing**: Off-chain order-matching engines can route limit orders and display market data at the pre-computed address *before* the deployment transaction lands on-chain.
- **Predictable integration**: Indexers and analytics dashboards can subscribe to market events at addresses known before block confirmation.
- **Duplicate prevention**: A repeated `questionId` produces the same salt and therefore the same CREATE2 address. The factory pre-checks for this before spending any gas on token transfers or clone initialization, reverting with `QuestionAlreadyDeployed(questionId, existing)`.

---

## 5. The Epsilon (ε) Initialization Formula

Every prediction market's AMM must be seeded with an initial quantity vector to ensure the market maker can immediately handle both buy and sell orders. The factory computes this seed value **on-chain** from the pool's current parameters.

### The LS-LMSR seed formula

The initial quantity vector is set to `q⁰ = [ε, ε, ..., ε]` — a uniform prior across all outcomes. The value ε is chosen so that the LS-LMSR cost function evaluates to exactly the vault's worst-case loss budget per market:

```
C(q⁰) = pool.maxRiskPerMarket   (the LS-LMSR "R" from Proposition 4.9)
```

For an n-outcome LS-LMSR with liquidity parameter `b = α · Σq_i = α · n · ε`:

```
C([ε,...,ε]) = b · ln(n · exp(ε/b))
             = α·n·ε · (ln n + 1/(α·n))
             = ε · (1 + α·n·ln n)
```

Solving for ε:

```
                    R
ε  =  ─────────────────────────────
          1  +  α · n · ln(n)
```

Where:
- `R = pool.maxRiskPerMarket × 1e12` — the risk budget converted from 6-dec USDC to 18-dec shares.
- `α = pool.alpha` — the LS-LMSR spread/commission parameter (18-dec).
- `n = nOutcomes` — the number of mutually exclusive outcomes.
- `ln(n)` — precomputed via a lookup table for `n ∈ {2, 3, 4, 5, 6, 7}` in 18-dec fixed-point.

### Why ε is computed on-chain (not caller-supplied)

Allowing the operator to pass an arbitrary ε would break the LS-LMSR solvency invariant if the value is wrong. By computing it deterministically from the pool's own parameters, the factory guarantees that `C(q⁰) = R` holds at every market launch — no operator error can produce an under-collateralized market.

`pool.alpha()` and `pool.maxRiskPerMarket()` are read **once** at the start of `deployMarket` and cached in local variables. The internal pure helper `_computeEpsilon` receives the cached values to produce ε. The same `alpha_` value is forwarded directly to `market.initialize()` — guaranteeing that the seed and the clone's stored alpha are always derived from the same read, with no possibility of drift between the two uses.

The public `computeEpsilon(uint8 nOutcomes)` view delegates to the same `_computeEpsilon` helper, reading pool parameters fresh on each external call so operators can verify the value off-chain before constructing a `DeployParams`.

### The lookup table approach

Because the maximum outcome count is 7 (UMIP-183), `ln(n)` only needs to be correct for integers 2 through 7. A precomputed lookup table provides exact 18-decimal precision at negligible gas cost (~6 conditional branches) — no Solidity math library is required.

---

## 6. The Atomic Deployment Pipeline

Every `deployMarket()` call executes these steps in one transaction:

```
Operator calls deployMarket(params)
    │
    ├─ [1] Validate inputs            ← reverts with descriptive errors if invalid
    │       (incl. ancillaryData non-empty guard and questionId duplicate check)
    │
    ├─ [2] Read pool.alpha + pool.maxRiskPerMarket once; compute ε
    │       (cached alpha_ forwarded to both _computeEpsilon and market.initialize)
    │
    ├─ [3] Pull reward tokens         ← safeTransferFrom(caller → factory)
    │
    ├─ [4] Derive salt = keccak256(abi.encode(questionId));
    │       deploy EIP-1167 clone via CREATE2
    │
    ├─ [5] market.initialize(...)     ← q-vector seeded, config pinned
    │
    ├─ [6] pool.registerMarket(...)   ← riskBudget reserved, MARKET_ROLE granted
    │
    ├─ [7] adapter.initializeQuestion(...)  ← UMA OO price request submitted
    │
    └─ [8] isDeployedMarket[market] = true; emit MarketDeployed;
```

If **any step reverts**, the EVM rolls back the entire transaction. There is no "half-initialized" market — the clone either exists in a fully wired protocol state or does not exist at all.

---

## 7. The Oracle Question Identity Chain

The `questionId` is the cryptographic identity linking the market, the factory, and the oracle:

```
fullAncillaryData = rawAncillaryData ++ ",initializer:" ++ lowerCaseHex(factoryAddress)
questionId = keccak256(fullAncillaryData)
```

The factory address embedded in the ancillary data suffix attributes every market to a specific factory instance on the UMA DVM voter UI, allowing UMA voters to distinguish protocol-sanctioned markets from spam. The adapter re-derives and verifies this on-chain inside `initializeQuestion()`.

The operator must pre-compute `questionId` off-chain using this formula and supply it as a `deployMarket` parameter. A mismatch causes the adapter to revert with `QuestionIdMismatch`.

---

## 8. Access Control Architecture

The factory uses OpenZeppelin's `AccessControl` (non-upgradeable version) with three roles:

| Role | Holder (Production) | Capabilities |
|---|---|---|
| `DEFAULT_ADMIN_ROLE` | Governance multisig (highest quorum) | Grant / revoke roles; `deregisterMarket` |
| `OPERATOR_ROLE` | Operations multisig | `deployMarket` |
| `PAUSER_ROLE` | Fast-response multisig (fewer signers) | `pause` / `unpause` factory; `pauseMarket` / `unpauseMarket` |

The separation between `OPERATOR_ROLE` and `PAUSER_ROLE` is intentional: emergency circuit-breakers (pausing) require faster response than new market deployments, so the PAUSER multisig can be configured with a lower signing threshold.

`DEFAULT_ADMIN_ROLE` is the only role that can call `deregisterMarket` because deregistration is irreversible once trading has started — the pool enforces this separately by reverting on any market that has ever had a trade.

---

## 9. Factory Non-Upgradeability — The Trust Model

The factory is intentionally **not a UUPS proxy** and has no upgrade mechanism:

- BlieverMarket clones pin `factory = address(this)` at initialization. Only this factory can pause, unpause, or expire those clones.
- If the factory is replaced (new version deployed), old markets retain their original factory address and remain fully functional — they simply use the old factory for lifecycle control.
- The new factory receives fresh role grants on the pool and adapter. It deploys new markets going forward.
- There is no risk that a factory upgrade compromises the integrity of already-deployed markets.

This design enforces a clean separation: **upgradeable infrastructure** (pool, adapter) coexists with **immutable market instances**.

---

## 10. Market Lifecycle Summary

```
                    ┌─── deployMarket() ───────────────── Factory deploys clone ──────┐
                    │                                                                   │
                    ▼                                                                   │
          [ACTIVE — trading open]                                                       │
                    │                                                                   │
         ┌──────────┴──────────┐                                                       │
         │                     │                                                       │
   tradingDeadline           PAUSER pauses                                             │
         │                     │                                                       │
         ▼                     ▼                                                       │
  [TRADING CLOSED]        [PAUSED]                                                     │
         │                     │                                                       │
         │              PAUSER unpauses                                                │
         │                     │                                                       │
         ▼                     ▼                                                       │
  adapter.resolve()     Adapter resolves                                               │
         │                                                                             │
         ▼                                                                             │
   [RESOLVED] ──── winners claim USDC ──── pool.claimWinnings() ────                 │
                                                                                       │
   OR if resolutionDeadline passes:                                                    │
         │                                                                             │
         ▼                                                                             │
  factory.expireUnresolved()                                                           │
         │                                                                             │
         ▼                                                                             │
   [EXPIRED — zero payout, riskBudget returned to LP NAV] ─────────────────────────── ┘
```

---

## 11. Security Properties

| Property | Mechanism |
|---|---|
| No orphaned clones | Token pull precedes clone deploy; any revert rolls back everything atomically |
| No invalid seeds | ε computed on-chain from cached pool params; caller cannot supply arbitrary ε |
| No alpha drift | `pool.alpha()` read once and forwarded to both `_computeEpsilon` and `market.initialize()` |
| No wrong oracle question | Adapter validates keccak256(fullAncillaryData) == questionId on-chain |
| No duplicate markets | Salt auto-derived from questionId; factory pre-checks CREATE2 address before spending gas |
| No empty oracle payload | `ancillaryData.length == 0` reverts before any state mutation or token movement |
| No unauthorized lifecycle control | `isDeployedMarket` mapping prevents acting on non-factory markets |
| No market upgrade vector | Clones are non-upgradeable; master is pinned as an immutable |
| No reentrancy | `nonReentrant` modifier on `deployMarket` and `expireUnresolved` |
| No stale ERC-20 approval | `forceApprove` resets then sets allowance atomically |
