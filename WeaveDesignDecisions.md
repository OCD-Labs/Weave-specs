# Weave — System Design Document

**Author:** Yemi (Ikeh Chukwuka Favour) — OCD Labs

**Date:** May 30, 2026

**Status:** Implemented and deployed on Robinhood Chain Testnet (Chain ID 46630).

---

## 1. Scope

This document covers every smart contract, every interface, every backend service, every external integration, and every edge case in the Weave system. It does not cover the frontend, which is covered in a separate document. When this document says the system it means everything except the user-facing web application: the Solidity contracts deployed on Robinhood Chain, the Go backend service, the AI composition engine embedded in the backend, the Chainlink Automation setup, and the DEX router abstraction.

---

## 2. Architecture Overview

Weave has four layers.

The contract layer handles all on-chain logic: basket deployment, deposit, redemption, rebalancing, fee collection, and creator revenue distribution. Every basket is a separate contract deployed via the minimal proxy pattern pointing to a shared implementation. Every basket also has an associated creator token contract implementing ERC-7641 for revenue sharing.

The automation layer handles the continuous monitoring of rebalancing-enabled baskets. A single Chainlink Automation upkeep job monitors all active baskets and triggers rebalancing trades through the DEX router when drift exceeds a basket's configured threshold.

The backend layer handles off-chain indexing, price caching, basket performance history, the stock catalogue service, and the AI composition endpoint. It is built in Go with SQLite and exposes an HTTP API to the frontend.

The AI layer is a Go package within the backend service that receives a natural language thesis, reads the stock catalogue from the local database, calls OpenAI GPT-4.1-mini to generate a basket composition proposal, validates the response, and returns structured JSON to the frontend for human review before any on-chain action is taken.

<p align="center">
  <img src="https://i.imgur.com/fOFTeut.png" alt="weave-sequence-diagram" />
</p>

---

## 3. Contract Inventory

```
WeaveRegistry               — global config, supported assets, basket index, protocol pause
BasketFactory               — deploys basket proxies and creator tokens atomically
BasketImplementation        — shared logic contract for all basket proxies
BasketProxy                 — ERC-1167 minimal proxy, one per basket (IS the basket token)
CreatorToken                — ERC-7641 revenue share token, one per basket
WeaveAutomation             — Chainlink Automation compatible rebalancing monitor
SwapRouter                  — testnet DEX adapter priced at oracle rates with configurable spread
OracleAdapter               — testnet oracle implementing IWeaveOracle (Chainlink shape)
```

---

## 4. WeaveRegistry

The registry is the single source of truth for all protocol configuration and is deployed once. All other contracts read from it.

```solidity
contract WeaveRegistry {
    address public governance;
    address public pendingGovernance;
    address public basketFactory;
    address public automationContract;
    address public swapRouter;
    address public protocolTreasury;
    address public immutable usdg;          // settlement token on Robinhood Chain

    uint256 public managementFeeBps;        // charged on each deposit and redemption, e.g. 50 = 0.5%
    uint256 public protocolShareBps;        // protocol's cut of fee, e.g. 2000 = 20%
    uint256 public creatorShareBps;         // creator's cut of fee, e.g. 8000 = 80%
    uint256 public minAUMForAutomation;     // minimum basket AUM for protocol-funded rebalancing
    uint256 public oracleStalenessSecs;     // max age of oracle price before reverting
    uint256 public minFirstDepositUsdg;     // prevents first-depositor manipulation
    uint256 public maxConstituents;         // max stocks per basket, e.g. 20
    uint256 public minWeightBps;            // minimum weight per constituent, e.g. 100 = 1%
    uint256 public minRebalanceTradeSizeUsdg; // minimum USDG value per rebalance trade leg
    uint256 public maxSwapSlippageBps;      // per-leg slippage tolerance on constituent swaps
    bool    public paused;                  // protocol-level emergency pause

    struct AssetConfig {
        address tokenAddress;
        address oracle;         // IWeaveOracle implementor (OracleAdapter on testnet, Chainlink on mainnet)
        string symbol;
        string name;
        string sector;
        bool active;
    }

    mapping(address => AssetConfig) private _assets;
    address[] private _supportedAssets;
    mapping(address => bool) private _isBasket;
    address[] private _allBaskets;
    mapping(address => BasketMeta) private _basketMeta;

    struct BasketMeta {
        address basket;
        address creatorToken;
        address creator;
        bool active;
        uint256 createdAt;
    }

    function addAsset(AssetConfig calldata config) external onlyGovernance;
    function deactivateAsset(address token) external onlyGovernance;
    function reactivateAsset(address token) external onlyGovernance;
    function registerBasket(address basket, address creatorToken, address creator) external onlyFactory;
    function setSwapRouter(address router) external onlyGovernance;
    function setBasketFactory(address factory) external onlyGovernance;
    function setAutomationContract(address automation) external onlyGovernance;
    function setProtocolTreasury(address treasury) external onlyGovernance;
    function setManagementFee(uint256 feeBps) external onlyGovernance;
    function setFeeSplit(uint256 protocolShareBps) external onlyGovernance;
    function setMinAUM(uint256 minAUM) external onlyGovernance;
    function setOracleStaleness(uint256 secs) external onlyGovernance;
    function setMinFirstDeposit(uint256 amount) external onlyGovernance;
    function setMaxConstituents(uint256 max) external onlyGovernance;
    function setMinWeightBps(uint256 bps) external onlyGovernance;
    function setMinRebalanceTradeSize(uint256 size) external onlyGovernance;
    function setMaxSwapSlippage(uint256 bps) external onlyGovernance;
    function pauseAll() external onlyGovernance;
    function unpauseAll() external onlyGovernance;
    function nominateGovernance(address nominee) external onlyGovernance;
    function acceptGovernance() external;
    function getAllBaskets() external view returns (BasketMeta[] memory);
    function getSupportedAssets() external view returns (AssetConfig[] memory);
    function assets(address token) external view returns (AssetConfig memory);
    function isBasket(address basket) external view returns (bool);
    function basketMeta(address basket) external view returns (BasketMeta memory);
    function getAssetPrice(address token) external view returns (uint256 price, uint256 updatedAt);
}
```

`getAssetPrice` calls `latestRoundData()` on the oracle contract for the given token, validates that `updatedAt` is within `oracleStalenessSecs` of `block.timestamp`, and reverts with `StalePrice(address token)` if not. This function is called by every basket operation that requires a price, ensuring stale data cannot be used in any valuation.

`pauseAll()` immediately freezes all basket deposits and redemptions. Rebalancing is deliberately not paused — reducing drift during a crisis is safe and desirable. `unpauseAll()` restores normal operation.

---

## 5. BasketFactory

The factory deploys a basket proxy and a creator token atomically in a single transaction. It is the only address with permission to call `registry.registerBasket()`. All composition validation happens here so `BasketImplementation.initialize()` can trust its inputs without redundant checks.

```solidity
contract BasketFactory {
    address public immutable registry;
    address public immutable implementation;

    event BasketCreated(
        address indexed basket,
        address indexed creatorToken,
        address indexed creator,
        string name,
        string symbol,
        string thesis,
        address[] constituents,
        uint256[] targetWeightsBps,
        bool rebalancingEnabled
    );

    function createBasket(
        string    calldata name,
        string    calldata symbol,
        string    calldata thesis,
        address[] calldata constituents,
        uint256[] calldata targetWeightsBps,
        bool      rebalancingEnabled,
        uint256   driftThresholdBps,
        uint256   initialDepositUsdg
    ) external returns (address basket, address creatorToken);
}
```

On `createBasket()`:

The factory validates that `constituents.length >= 3`, that `constituents.length <= registry.maxConstituents`, that `constituents.length == targetWeightsBps.length`, that all weights sum to exactly 10,000, that every weight is at least `registry.minWeightBps`, that every weight does not exceed 5,000, that there are no duplicate constituent addresses, that every constituent address is active in the registry, and that `initialDepositUsdg >= registry.minFirstDepositUsdg`. If rebalancing is enabled, it validates that `driftThresholdBps > 0` and `driftThresholdBps <= 5,000`.

It then deploys a BasketProxy using OpenZeppelin's `Clones.clone(implementation)`, deploys a `CreatorToken` with name `"Weave Creator: {name}"` and symbol `"wCT-{symbol}"`, calls `basket.initialize(...)` with all parameters, calls `registry.registerBasket(basket, creatorToken, msg.sender)`, and routes `initialDepositUsdg` through the basket's `deposit()` function. The creator receives the initial basket tokens from that first deposit and the full supply of creator tokens.

The `BasketCreated` event emits the full basket configuration including constituent addresses, weights, and thesis — enabling the backend indexer to bootstrap basket data without follow-up RPC calls.

---

## 6. BasketImplementation

This is the logic contract that every BasketProxy delegates to. It is also the ERC-20 basket token. It handles deposits, redemptions, rebalancing, and fee collection.

**Storage layout (must never be reordered between deployments):**

```
Slot 0:  bool _initialized
Slot 1:  bool _locked (ReentrancyGuard)
ERC-20 base storage (OZ ERC20 internal)
Slot 7:  address registry
Slot 8:  address creatorToken
Slot 9:  string _thesis
Slot 10: address[] _constituents
Slot 11: uint256[] _targetWeightsBps
Slot 12: uint256[] _constituentBalances
Slot 13: bool rebalancingEnabled
Slot 14: uint256 driftThresholdBps
Slot 15: address creator
Slot 16: bool suspended
```

**Initialization:**

```solidity
function initialize(
    address _registry,
    address _creatorToken,
    string calldata name_,
    string calldata symbol_,
    string calldata thesis_,
    address[] calldata constituents_,
    uint256[] calldata targetWeightsBps_,
    bool _rebalancingEnabled,
    uint256 _driftThresholdBps,
    address _creator
) external;
```

Called exactly once immediately after proxy deployment. Sets all storage fields and marks `_initialized = true`. Reverts with `AlreadyInitialized()` on any subsequent call.

**Core functions:**

```solidity
// Deposit USDG into the basket.
// Reverts if protocol is paused or basket is suspended.
// Deducts management fee first, then buys constituents in target-weight proportions.
// Mints basket tokens to receiver proportional to their contribution to total NAV.
// First depositor establishes 1 USDG = 1 basket token (6-dec → 18-dec via 1e12 scale).
// Per-leg slippage protection computed from oracle price minus registry.maxSwapSlippageBps.
function deposit(
    uint256 usdgAmount,
    uint256 minBasketTokensOut,
    address receiver
) external nonReentrant returns (uint256 basketTokensMinted);

// Redeem basket tokens for USDG.
// Reverts if protocol is paused or basket is suspended.
// Burns tokens, sells proportional holdings, deducts fee, returns net USDG.
function redeem(
    uint256 basketTokenAmount,
    uint256 minUsdgOut,
    address receiver
) external nonReentrant returns (uint256 usdgReturned);

// Rebalance the basket to restore target weights.
// Reverts if rebalancingEnabled is false or basket is suspended.
// Reverts if no constituent's drift exceeds driftThresholdBps.
// NOT blocked by protocol pause — reducing drift during a pause is safe.
// Permissionless: anyone can call. Protocol automation calls it when funded.
// Trade legs below registry.minRebalanceTradeSizeUsdg are skipped (dust protection).
// Sell phase: sells overweight constituents, accumulates USDG.
// Buy phase: uses accumulated USDG to buy underweight constituents.
function rebalance(uint256[] calldata minAmountsOut) external nonReentrant;

// Distribute any free USDG sitting in the basket to the protocol treasury
// and creator token revenue pool. Called automatically at the start of deposit
// and redeem. Can also be called externally to handle dust from rebalancing rounding.
function collectFees() external nonReentrant;

// View: current total basket value in USDG
function totalValueUsdg() external view returns (uint256);

// View: current NAV per basket token in 18-decimal USDG. Returns 0 when supply is 0.
function navPerToken() external view returns (uint256);

// View: live weight of each constituent in bps from oracle prices
function currentWeightsBps() external view returns (uint256[] memory);

// View: maximum drift in bps across all constituents
function maxDrift() external view returns (uint256);

// View: whether any constituent currently exceeds driftThresholdBps
// Always false for static baskets or suspended baskets
function needsRebalancing() external view returns (bool);

// View: constituent token addresses
function constituents() external view returns (address[] memory);

// View: target weights in bps, same order as constituents()
function targetWeightsBps() external view returns (uint256[] memory);

// View: current token balances for each constituent, same order as constituents()
function constituentBalances() external view returns (uint256[] memory);

// View: investment thesis text
function thesis() external view returns (string memory);

// View: full basket state in one call — use this to avoid multiple RPC round trips
function basketState() external view returns (
    address[] memory constituentsOut,
    uint256[] memory targetWeightsOut,
    uint256[] memory currentWeightsOut,
    uint256[] memory balancesOut,
    uint256 totalValueOut,
    uint256 navOut,
    bool rebalancingEnabledOut,
    uint256 driftThresholdBpsOut,
    uint256 maxDriftOut
);
```

---

## 7. Fee Collection Mechanics

The management fee is charged as a percentage of USDG on each deposit and each redemption rather than as a time-based AUM dilution. This keeps the fee logic transparent and predictable.

On every deposit:

```
fee_usdg = deposit_usdg * managementFeeBps / 10_000
net_deposit_usdg = deposit_usdg - fee_usdg
protocol_cut = fee_usdg * protocolShareBps / 10_000
creator_cut = fee_usdg - protocol_cut

transfer protocol_cut → registry.protocolTreasury
call creatorToken.snapshotRevenue(creator_cut) with USDG transfer
buy constituents using net_deposit_usdg only
mint basket tokens based on net_deposit_usdg contribution
```

On every redemption:

```
gross_usdg_value = basket_tokens_being_redeemed * navPerToken / 1e18
sell constituents proportional to gross_usdg_value
fee_usdg = gross_usdg_value * managementFeeBps / 10_000
net_usdg = usdg_from_sales - fee_usdg
protocol_cut = fee_usdg * protocolShareBps / 10_000
creator_cut = fee_usdg - protocol_cut

transfer protocol_cut → registry.protocolTreasury
call creatorToken.snapshotRevenue(creator_cut) with USDG transfer
transfer net_usdg → receiver
burn basket tokens
```

The fee does not compound. Every deposit and redemption pays the flat fee once on that transaction. Users who hold basket tokens without transacting pay no fee until they redeem.

---

## 8. Basket Token NAV and Minting Math

**Total basket value:**

```
totalValueUsdg = sum over all constituents:
    constituentBalances[i] * oraclePrice(constituents[i]) / PRICE_SCALE
```

Oracle prices are 8-decimal. Constituent balances are 18-decimal (standard ERC-20). `PRICE_SCALE = 1e20` produces a 6-decimal USDG result: `18 + 8 - 20 = 6`. All intermediate multiplications use `Math.mulDiv` for overflow safety.

**NAV per token:**

```
navPerToken = totalValueUsdg * 1e18 / basketToken.totalSupply()
```

Expressed in 18-decimal USDG per basket token.

**Basket tokens minted on deposit:**

```
// After fee deduction, net_deposit_usdg is available to invest.
// If total supply is zero (first depositor):
basket_tokens_minted = net_deposit_usdg * 1e12
// (converts 6-decimal USDG to 18-decimal basket token, establishing 1:1 initial price)

// If total supply is non-zero:
basket_tokens_minted = net_deposit_usdg * totalSupply / totalValueUsdg
```

The first depositor establishes a 1:1 ratio of 1 USDG to 1 basket token. All subsequent depositors are priced against the current NAV. There is no first-depositor donation attack risk because the basket immediately deploys deposits into constituent purchases rather than holding USDG.

**Constituent purchase on deposit with per-leg slippage protection:**

```
// For each constituent i:
usdg_to_spend_on_i = net_deposit_usdg * targetWeightsBps[i] / 10_000

// Minimum acceptable tokens computed from oracle price minus maxSwapSlippageBps:
expected_tokens = usdg_to_spend_on_i * PRICE_SCALE / oraclePrice(constituents[i])
min_token_out = expected_tokens * (10_000 - maxSwapSlippageBps) / 10_000

constituent_tokens_received = swapRouter.swapExactUSDGForToken(
    constituents[i],
    usdg_to_spend_on_i,
    min_token_out,
    address(this)
)
constituentBalances[i] += constituent_tokens_received
```

**Current weight calculation:**

```
// For each constituent i:
value_i = constituentBalances[i] * oraclePrice(constituents[i]) / PRICE_SCALE
currentWeightBps[i] = value_i * 10_000 / totalValueUsdg
drift_i = abs(currentWeightBps[i] - targetWeightsBps[i])
```

**Rebalancing logic:**

```
// Identify overweight and underweight constituents
target_value_i = totalValueUsdg * targetWeightsBps[i] / 10_000
delta_i = current_value_i - target_value_i

// Skip legs where |delta_i| < minRebalanceTradeSizeUsdg (dust protection)

// Positive delta: overweight → sell delta_i USDG worth of constituent i
// Negative delta: underweight → buy |delta_i| USDG worth of constituent i

// Phase 1 (sell): sell all overweight positions, accumulate USDG
// Phase 2 (buy): use accumulated USDG to buy all underweight positions
// This avoids needing to know trade ordering in advance
```

All rounding truncates toward zero. Residual USDG from rounding differences stays in the basket and is distributed via `collectFees()` on the next interaction.

---

## 9. CreatorToken (ERC-7641)

One creator token contract is deployed per basket. The creator token is an ERC-20 with a fixed total supply of 1,000,000 units (18 decimals), all minted to the basket creator at deployment. It implements ERC-7641 to enable a revenue sharing claim mechanism against a USDG revenue pool.

Name is `"Weave Creator: {basketName}"` and symbol is `"wCT-{basketSymbol}"`.

```solidity
contract CreatorToken is ERC20, ReentrancyGuard {
    address public immutable basket;
    address public immutable usdg;

    uint256 public constant TOTAL_SUPPLY = 1_000_000 * 1e18;

    uint256 public snapshotCount;

    struct Snapshot {
        uint256 usdgAmount;   // USDG added to pool at this snapshot
        uint256 totalSupply;  // always TOTAL_SUPPLY
        uint256 blockNumber;  // block at which snapshot was recorded
    }

    // Called by the basket when it distributes creator's share of management fee.
    // Pulls USDG from the basket (must have approved this contract) and records a snapshot.
    function snapshotRevenue(uint256 usdgAmount) external onlyBasket;

    // Returns USDG claimable by account for a given snapshot.
    // Uses holder's balance at the snapshot's block number via binary-searched checkpoints.
    function claimableRevenue(address account, uint256 snapshotId)
        public view returns (uint256);

    // Claim USDG for a single snapshot.
    function claim(uint256 snapshotId) public nonReentrant;

    // Claim all unclaimed snapshots in one transaction.
    // Silently skips snapshots with nothing to claim.
    // Reverts with NothingToClaim if nothing is claimable across all snapshots.
    function claimAll() external nonReentrant;

    // Returns USDG redeemable if amount creator tokens were burned now.
    // Based on current USDG pool balance proportional to burn amount.
    function redeemableOnBurn(uint256 amount) public view returns (uint256);

    // ERC-7641 deflationary mechanism — burn creator tokens to redeem
    // a proportional share of the accumulated USDG revenue pool.
    // Permanently reduces total supply.
    function burn(uint256 amount) external nonReentrant;
}
```

The revenue sharing formula for any given snapshot:

```
claimable = snapshot.usdgAmount * balanceAtSnapshotBlock / TOTAL_SUPPLY
```

Balance at snapshot time is tracked via a block-level checkpoint system. Every transfer writes a checkpoint for sender and receiver. `claimableRevenue` binary-searches checkpoints to find the holder's balance at the snapshot's block number.

If the creator sells half their creator tokens to an investor, all subsequent snapshots distribute 50% to the creator and 50% to the investor. Past snapshots distribute based on whoever held the tokens at that specific snapshot's block. There is no economic leakage in either direction.

---

## 10. DEX Router Abstraction

All token swap operations go through a single router interface so that the underlying DEX can be changed without modifying basket contracts. On mainnet, `registry.setSwapRouter(newAdapter)` swaps it out with no other contract changes.

```solidity
interface IWeaveRouter {
    function swapExactUSDGForToken(address token, uint256 usdgIn, uint256 minTokenOut, address recipient)
        external returns (uint256 tokenOut);

    function swapExactTokenForUSDG(address token, uint256 tokenIn, uint256 minUsdgOut, address recipient)
        external returns (uint256 usdgOut);

    function swapExactTokenForToken(address tokenIn, address tokenOut, uint256 amountIn, uint256 minOut, address recipient)
        external returns (uint256 amountOut);

    function quoteUSDGForToken(address token, uint256 usdgIn) external view returns (uint256 tokenOut);
    function quoteTokenForUSDG(address token, uint256 tokenIn) external view returns (uint256 usdgOut);
}
```

**SwapRouter for testnet.** Since Robinhood Chain testnet has no DEX with real liquidity, the `SwapRouter` simulates swaps at oracle prices with a configurable spread (default 30 bps = 0.3%, matching Uniswap v3's standard fee tier). It holds a treasury of test USDG and test stock tokens funded by the team. The spread makes NAV calculations realistic — a deposit does not return exactly 100 USDG worth of tokens for 100 USDG in, which no real DEX would ever do. `InsufficientLiquidity(address token, uint256 needed, uint256 available)` is returned when the treasury runs dry. The registry swap router address is updated to the real DEX adapter on mainnet with no contract changes required.

---

## 11. WeaveAutomation

Chainlink Automation compatible contract that monitors all rebalancing-enabled baskets.

```solidity
contract WeaveAutomation is AutomationCompatibleInterface {
    address public immutable registry;
    address public immutable governance;
    uint256 public batchSize;               // max baskets per upkeep call, default 50
    uint256 public maxRebalanceSlippageBps; // slippage tolerance on sell legs, default 50 bps

    // checkUpkeep runs off-chain on Chainlink keeper nodes — no gas cost to protocol.
    // checkData encodes (uint256 startIndex) for pagination across multiple upkeep jobs.
    function checkUpkeep(bytes calldata checkData)
        external view override
        returns (bool upkeepNeeded, bytes memory performData);

    // performUpkeep runs on-chain only when checkUpkeep returned true.
    // Computes per-constituent minAmountsOut from oracle prices minus maxRebalanceSlippageBps.
    // Double-checks conditions on-chain before executing — stale performData protection.
    function performUpkeep(bytes calldata performData) external override;

    function setBatchSize(uint256 newSize) external; // governance only
    function setMaxRebalanceSlippage(uint256 bps) external; // governance only, max 500 bps
}
```

`performUpkeep` computes meaningful `minAmountsOut` for sell legs from oracle prices minus `maxRebalanceSlippageBps`, protecting automation-triggered rebalances against sandwich attacks on mainnet. Buy legs are protected by `BasketImplementation._buyConstituents` using `registry.maxSwapSlippageBps`.

The Chainlink Automation upkeep is registered once by the protocol and funded from the protocol treasury. The `batchSize` parameter prevents `checkUpkeep` from exceeding Chainlink's check gas limit. As the basket count grows, additional upkeep jobs can be registered each covering a different index range via `checkData`.

Baskets below `minAUMForAutomation` are excluded from the automated job but can still be rebalanced by any caller directly.

---

## 12. OracleAdapter

Implements `IWeaveOracle` — the full Chainlink `AggregatorV3Interface` five-field return shape. On mainnet, a thin wrapper around the real Chainlink aggregator implementing the same interface is a drop-in replacement with zero contract changes required.

```solidity
contract OracleAdapter is IWeaveOracle {
    // Set price manually to simulate market movement on testnet.
    // Each call increments roundId. On mainnet this function does not exist.
    function setPrice(int256 newPrice) external onlyOwner;

    // Returns full Chainlink-compatible round data.
    function latestRoundData() external view returns (
        uint80 roundId,
        int256 answer,    // price in 8-decimal USD
        uint256 startedAt,
        uint256 updatedAt,
        uint80 answeredInRound
    );

    function decimals() external view returns (uint8);     // always 8
    function description() external view returns (string memory); // e.g. "TSLA / USD"
}
```

---

## 13. Edge Cases and Their Handling

**Constituent becomes inactive.** Governance calls `registry.deactivateAsset(token)`. The basket is flagged as `suspended = true` by the next interaction that reads the asset config via `_checkConstituentsActive()`. Suspended baskets cannot accept new deposits or rebalance. Existing holders can still redeem. Governance can restore the asset via `registry.reactivateAsset(token)`, after which the basket's suspension is resolved on the next interaction.

**Protocol is paused.** Governance calls `registry.pauseAll()`. All `deposit()` and `redeem()` calls on all baskets revert with `ProtocolPaused()`. Rebalancing is not affected — reducing drift during a crisis is safe. Governance calls `registry.unpauseAll()` to resume.

**Oracle price is stale.** Any operation reading the stale feed's price reverts with `StalePrice(address token)`. The specific token address is included in the error. This affects any valuation of total basket value since all feeds are read.

**Deposit when DEX has insufficient liquidity.** The swap router reverts with `InsufficientLiquidity(address token, uint256 needed, uint256 available)`. The entire deposit transaction reverts. The user's USDG is returned. The basket state is unchanged. The user should retry with a smaller deposit or wait for the router treasury to be refunded.

**Rebalancing trades result in dust.** After rebalancing, rounding means the basket may hold tiny USDG residuals that could not be fully deployed. These accumulate in the basket. `collectFees()` distributes them on the next deposit or redemption, or can be called directly by anyone.

**Rebalance trade leg below minimum size.** Legs where `|delta_i| < minRebalanceTradeSizeUsdg` are skipped entirely. This avoids spending more gas than the trade is worth on dust positions.

**Creator transfers or sells creator token.** The checkpoint system tracks balances at snapshot block numbers. The new holder earns revenue from all future snapshots. Past unclaimed snapshots are claimable by whoever held the tokens at that snapshot's block. There is no economic leakage.

**Two baskets with identical compositions.** Allowed. Both compete for capital on their own merits with different addresses, histories, and creator reputations.

**First depositor minimum not met.** The factory reverts with `InitialDepositTooLow(uint256 got, uint256 min)`. The creator must fund the basket above the minimum at launch.

**Rebalancing-enabled basket falls below minAUM.** It falls out of the automated Chainlink job. Rebalancing remains possible permissionlessly but is no longer triggered automatically. If AUM recovers, the basket is automatically re-included.

**Stock split.** If token supply doubles, the basket's constituent balance doubles (split credits all holders). The oracle price halves simultaneously. The product `balance * price` is constant, so `totalValueUsdg` and NAV are unaffected. No special handling needed.

**Zero drift threshold.** The factory rejects this with `InvalidDriftThreshold()`. Zero threshold means any movement triggers rebalancing, which is operationally impossible.

**Fee collection when basket has zero AUM.** `collectFees()` reads zero USDG balance and returns immediately. No transfer occurs.

**Reentrancy.** All state-mutating basket functions use `ReentrancyGuard`. All external calls follow checks-effects-interactions strictly.

---

## 14. Backend Service Architecture

The backend is a Go service with SQLite storage. It holds no private keys, submits no transactions, and does not interact with any protocol contract in a write capacity.

### 14.1 Database Schema

```sql
CREATE TABLE baskets (
    address TEXT PRIMARY KEY,
    creator_token_address TEXT NOT NULL,
    creator_address TEXT NOT NULL,
    name TEXT NOT NULL,
    symbol TEXT NOT NULL,
    thesis TEXT NOT NULL,
    rebalancing_enabled INTEGER NOT NULL,
    drift_threshold_bps INTEGER,
    created_at INTEGER NOT NULL,
    created_tx TEXT NOT NULL,
    suspended INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE basket_constituents (
    basket_address TEXT NOT NULL,
    stock_address TEXT NOT NULL,
    symbol TEXT NOT NULL,
    target_weight_bps INTEGER NOT NULL,
    display_order INTEGER NOT NULL,
    PRIMARY KEY (basket_address, stock_address)
);

CREATE TABLE deposits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    basket_address TEXT NOT NULL,
    investor_address TEXT NOT NULL,
    usdg_amount TEXT NOT NULL,
    basket_tokens_minted TEXT NOT NULL,
    fee_usdg TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    tx_hash TEXT NOT NULL
);

CREATE TABLE redemptions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    basket_address TEXT NOT NULL,
    investor_address TEXT NOT NULL,
    basket_tokens_burned TEXT NOT NULL,
    usdg_returned TEXT NOT NULL,
    fee_usdg TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    tx_hash TEXT NOT NULL
);

CREATE TABLE rebalances (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    basket_address TEXT NOT NULL,
    triggered_by TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    tx_hash TEXT NOT NULL
);

CREATE TABLE fee_snapshots (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    basket_address TEXT NOT NULL,
    snapshot_id INTEGER NOT NULL,
    usdg_amount TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    tx_hash TEXT NOT NULL
);

CREATE TABLE supported_assets (
    address TEXT PRIMARY KEY,
    symbol TEXT NOT NULL,
    name TEXT NOT NULL,
    sector TEXT NOT NULL,
    oracle_address TEXT NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1,
    added_at INTEGER NOT NULL
);

CREATE TABLE price_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    stock_address TEXT NOT NULL,
    price_usdg TEXT NOT NULL,
    timestamp INTEGER NOT NULL
);

CREATE TABLE nav_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    basket_address TEXT NOT NULL,
    nav_per_token TEXT NOT NULL,
    total_value_usdg TEXT NOT NULL,
    timestamp INTEGER NOT NULL
);

CREATE TABLE sync_cursors (
    key TEXT PRIMARY KEY,
    block_num INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE basket_state_cache (
    basket_address TEXT PRIMARY KEY,
    constituents_json TEXT NOT NULL,
    nav_per_token TEXT NOT NULL,
    total_value_usdg TEXT NOT NULL,
    max_drift_bps INTEGER NOT NULL,
    needs_rebalancing INTEGER NOT NULL,
    cached_at INTEGER NOT NULL
);

CREATE TABLE creator_claimable_cache (
    wallet_address TEXT NOT NULL,
    snapshot_id INTEGER NOT NULL,
    basket_address TEXT NOT NULL,
    claimable_usdg TEXT NOT NULL,
    cached_at INTEGER NOT NULL,
    PRIMARY KEY (wallet_address, snapshot_id, basket_address)
);
```

### 14.2 Indexer

The indexer syncs chain state on startup and then subscribes to live events.

**Startup sync:** Calls `registry.getSupportedAssets()` and `registry.getAllBaskets()` directly via RPC to upsert the complete current state into SQLite. This eliminates missed-block issues for the asset and basket lists. For any basket with no constituent rows, calls `basketState()` plus `name()`, `symbol()`, `thesis()` on the proxy to seed full metadata. A paginated worker pool of 5 goroutines processes 50 baskets at a time. Constituent writes are atomic database transactions — all constituents for a basket are written or none.

**Transactional event scanning:** Scans historical blocks from `DEPLOY_BLOCK` forward for deposit, redemption, rebalance, and fee snapshot events using a cursor stored in `sync_cursors`. The cursor only advances after all events in a chunk are successfully written. On restart, scanning resumes from the cursor rather than from the deploy block.

**Live subscription:** WebSocket subscription via Alchemy for all new events from the registry and all known basket addresses. `BasketCreated` and `AssetAdded` events are enriched — they carry full data in their logs so no follow-up RPC calls are needed.

Events indexed:

```
BasketCreated(address indexed basket, address indexed creatorToken, address indexed creator,
              string name, string symbol, string thesis, address[] constituents,
              uint256[] targetWeightsBps, bool rebalancingEnabled)
Deposited(address indexed investor, uint256 usdgAmount, uint256 basketTokensMinted, uint256 feeUsdg)
Redeemed(address indexed investor, uint256 basketTokensBurned, uint256 usdgReturned, uint256 feeUsdg)
Rebalanced(address indexed triggeredBy)
RevenueSnapshoted(uint256 indexed snapshotId, uint256 usdgAmount, uint256 blockNumber)
AssetAdded(address indexed token, string symbol, string name, string sector, address oracle)
AssetDeactivated(address indexed token)
Suspended()
```

A price polling goroutine reads oracle prices every 60 seconds for assets that are constituents of at least one active basket — assets not held by any basket are not polled. A NAV computation goroutine polls `navPerToken()` and `totalValueUsdg()` every 5 minutes for all non-suspended baskets using a 5-worker pool. Both are best-effort background processes.

### 14.3 API Endpoints

```
GET /baskets
  Returns all baskets with current NAV, AUM, constituent summary, and 24h NAV change.

GET /baskets/:address
  Returns full basket detail: constituents with live weights (30s cache from basketState()),
  NAV changes (24h, 7d, 30d), deposit history, rebalance history, performance history.

GET /baskets/:address/performance
  Returns full NAV history time series ordered by timestamp ascending.

GET /baskets/:address/positions/:wallet
  Returns the investor's basket token balance, current USDG value, cost basis,
  and unrealised PnL computed from deposit/redemption event history.

GET /catalogue
  Returns all active supported assets with symbol, name, sector, oracle address,
  current oracle price (8-decimal), and 24h price change.

GET /catalogue/:address
  Returns single asset metadata. Price data not included — use GET /catalogue.

GET /prices
  Returns latest oracle prices for assets held by at least one active basket.

GET /positions/:wallet
  Returns portfolio summary across all baskets: total value, total deposited,
  unrealised PnL, and per-basket position breakdown.

GET /creator/:wallet
  Returns all baskets created by the wallet, unclaimed revenue snapshots,
  claimable amounts (60s cache from claimableRevenue() contract reads),
  and revenue history.

GET /creator-tokens/:address
  Returns all revenue snapshots for a creator token contract.

POST /ai/compose
  Body: { "thesis": string (min 20 chars) }
  Returns: { "constituents": [...], "overallRationale": string, "riskNotes": string, "provider": string }
  Calls OpenAI GPT-4.1-mini, validates the proposal, returns structured JSON.
  The on-chain basket is NOT created here. The frontend uses this response to populate
  the basket creation form for human review before submission.
```

---

## 15. AI Composition Engine

The AI engine is a Go package (`server/ai`) within the backend service. There is no separate service.

**Flow:**

1. Frontend sends `POST /ai/compose` with the thesis string to the Go backend
2. Go backend fetches the full active catalogue from its local SQLite database
3. Go backend calls OpenAI `gpt-4.1-mini` via `api.openai.com/v1/chat/completions` with the thesis and catalogue
4. Go backend validates the response: weights sum to 10,000, all addresses in catalogue, 3–12 constituents, each weight 100–5,000 bps, no duplicates, symbol matches
5. If the response is invalid JSON or fails validation, retries once with a stricter prompt
6. Validated proposal is returned to the frontend

**System prompt:**

```
You are a financial analyst building thematic equity baskets from a specific catalogue of
tokenized stocks. You will receive a natural language investment thesis and a JSON catalogue
of available stocks with their symbols, names, sectors, and current prices.

Your task is to select 3 to 12 stocks from the catalogue that best represent the thesis,
assign each a target weight in basis points that sum to exactly 10,000, and provide a
one-sentence rationale for each inclusion.

Rules:
- Only select stocks that appear in the provided catalogue
- Weights must be integers and must sum to exactly 10,000
- No single weight may be less than 100 (1%) or more than 5000 (50%)
- Do not include more than 12 constituents
- Do not include fewer than 3 constituents
- Prefer direct plays over indirect beneficiaries unless the thesis specifically calls for breadth
- Weight by conviction and relevance to the thesis, not by market cap alone

Return ONLY valid JSON matching this schema, no preamble or explanation outside the JSON:
{
  "constituents": [
    {
      "address": "0x...",
      "symbol": "AAPL",
      "weightBps": 2000,
      "rationale": "one sentence explaining why this stock fits the thesis"
    }
  ],
  "overallRationale": "two to three sentences explaining the basket's overall construction logic",
  "riskNotes": "one to two sentences noting the key risks or concentration exposures"
}
```

---

## 16. External Integrations

**Oracle price feeds.** Every supported stock has an oracle contract address stored in `registry.assets[token].oracle`. On testnet this is an `OracleAdapter` contract with manually set prices. On mainnet this is a thin wrapper around the real Chainlink aggregator. Both implement `IWeaveOracle` returning the full five-field `AggregatorV3Interface` shape from `latestRoundData()`. The `answer` field is the price in 8-decimal USD. The `updatedAt` field is validated against `block.timestamp - registry.oracleStalenessSecs` before any value is used.

**Alchemy.** Used exclusively for the WebSocket live event subscription. HTTP RPC calls for state reads and block scanning use the public Robinhood Chain RPC endpoint (`https://rpc.testnet.chain.robinhood.com`) which has no rate limits on `eth_getLogs` block range.

**OpenAI API.** The AI composition engine calls `gpt-4.1-mini` via the OpenAI REST API. The `OPENAI_API_KEY` environment variable is stored server-side and never exposed to the frontend. The model was selected for its instruction-following performance and cost relative to alternatives.

**Chainlink Automation.** The WeaveAutomation contract is registered as a custom logic upkeep on Chainlink Automation. The upkeep is funded with LINK from the protocol treasury. Custom logic upkeeps support `checkData` for parameterising which baskets each job monitors, enabling pagination across multiple jobs as the basket count grows.

---

## 17. Deployment Order

```
1.  WeaveRegistry(governance, usdg, protocolTreasury, managementFeeBps,
                  protocolShareBps, minAUMForAutomation, oracleStalenessSecs,
                  minFirstDepositUsdg, maxConstituents, minWeightBps,
                  minRebalanceTradeSizeUsdg, maxSwapSlippageBps)
2.  For each supported stock: OracleAdapter("SYMBOL / USD", initialPrice)
3.  SwapRouter(registry)
4.  BasketImplementation()
5.  BasketFactory(registry, implementation)
6.  WeaveAutomation(registry, governance)
7.  Registry.setBasketFactory(basketFactory)
8.  Registry.setSwapRouter(swapRouter)
9.  Registry.setAutomationContract(weaveAutomation)
10. For each supported asset:
    Registry.addAsset(AssetConfig { tokenAddress, oracle, symbol, name, sector, active: true })
11. For each stock token: approve SwapRouter, fund SwapRouter treasury
12. Approve USDG to BasketFactory for initial deposit
13. BasketFactory.createBasket(...) — first basket, protocol-seeded to demonstrate the flow
14. Register WeaveAutomation as a Chainlink Automation upkeep
15. Fund the upkeep with LINK from protocol treasury
```

---

## 18. Security Considerations

**Reentrancy.** Every state-mutating basket and creator token function uses `ReentrancyGuard`. All external calls follow checks-effects-interactions strictly.

**Oracle manipulation.** The protocol reads from oracle adapters (testnet) or direct Chainlink aggregators (mainnet). Oracle prices cannot be manipulated in a single block. Stale price protection reverts any operation where the price has not been updated within `oracleStalenessSecs`.

**Slippage protection.** All DEX swaps are protected at two levels: per-leg slippage on constituent purchases enforced by `registry.maxSwapSlippageBps` inside `BasketImplementation._buyConstituents`, and per-leg slippage on automation-triggered rebalance sell legs enforced by `WeaveAutomation.maxRebalanceSlippageBps`.

**Weight sum validation.** The factory enforces that weights sum to exactly 10,000 bps at basket creation. Weights are immutable after deployment. Drift over time is by design and governed by the drift threshold.

**Protocol pause.** Governance can freeze all deposits and redemptions instantly via `registry.pauseAll()`. Rebalancing is explicitly not paused. Two-step governance transfer prevents a typo from permanently locking the protocol.

**Creator token economics.** Total supply is fixed at `1,000,000 * 1e18` and set in the constructor. No minting after creation is possible. Governance has no power over creator token balances or revenue distributions.

**No upgradeability for existing baskets.** Basket proxies are not upgradeable. Governance can deploy a new `BasketImplementation` and new baskets will use it, while existing proxies continue operating safely on the old implementation.

**Dust protection.** Rebalance trade legs below `minRebalanceTradeSizeUsdg` are skipped, preventing gas-wasteful trades that cost more than they contribute to weight restoration.

---

## 19. Open Questions

**Real Chainlink feed addresses on Robinhood Chain testnet.** Chainlink is a confirmed infrastructure partner but specific feed addresses for Robinhood Chain testnet are not yet publicly documented. On mainnet, OracleAdapter contracts are replaced by thin wrappers around real Chainlink aggregators — the `IWeaveOracle` interface is identical.

**Mainnet DEX availability.** The SwapRouter handles testnet swaps at oracle prices with a 30 bps spread. The mainnet DEX with meaningful liquidity for tokenized stocks is not yet confirmed. A production DEX adapter implementing `IWeaveRouter` is deployed and activated via `registry.setSwapRouter(newAdapter)` with no other contract changes required.
