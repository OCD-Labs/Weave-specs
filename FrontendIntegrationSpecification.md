# Weave — Frontend Integration Specification

**Project:** Weave — Onchain Index Protocol for Tokenized Equities

**Author:** Yemi (Ikeh Chukwuka Favour) — OCD Labs

**Date:** May 30, 2026

**Backend:** https://weave.up.railway.app

**Chain:** Robinhood Chain Testnet (Chain ID: 46630)

This document is the authoritative reference for frontend integration. It is written from the actual deployed backend. Every response shape, field name, and data type is verified against the live API.

---

## 1. Global Setup

### 1.1 Environment Variables

```
NEXT_PUBLIC_BACKEND_URL=https://weave.up.railway.app
NEXT_PUBLIC_CHAIN_ID=46630
NEXT_PUBLIC_REGISTRY_ADDRESS=0x19Ab3408af6503a7D4BeC255b064f8B02A345D04
NEXT_PUBLIC_BASKET_FACTORY_ADDRESS=0xE9854c4734cd4A9dbC5086398A11df3c11f40b21
NEXT_PUBLIC_USDG_ADDRESS=0x7E955252E15c84f5768B83c41a71F9eba181802F
```

**Note on USDC vs USDG:** The settlement token on Robinhood Chain is USDG, not USDC. Every field in the API that references a monetary amount uses the suffix `Usdg`.

### 1.2 Wallet Connection

```typescript
const robinhoodChainTestnet = {
  id: 46630,
  name: 'Robinhood Chain Testnet',
  network: 'robinhood-testnet',
  nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
  rpcUrls: {
    default: { http: ['https://rpc.testnet.chain.robinhood.com'] },
    public:  { http: ['https://rpc.testnet.chain.robinhood.com'] },
  },
}
```

### 1.3 Contract ABIs Required

```
WeaveRegistry.json          — for reading fee config and min deposit
BasketProxy.json            — same ABI as BasketImplementation
BasketFactory.json          — for createBasket
CreatorToken.json           — for claimAll and claimableRevenue
ERC20.json                  — standard, for USDG approve and allowance
```

### 1.4 Token Decimals

| Token | Decimals | Notes |
|-------|----------|-------|
| USDG | 6 | All `*Usdg` fields in the API are 6-decimal integer strings |
| Basket tokens | 18 | `basketTokensMinted`, `basketTokenBalance`, `basketTokensBurned` |
| Oracle prices | 8 | `currentPriceUsdg` in `/catalogue` and `/prices` is 8-decimal. Divide by 1e8 for display |

### 1.5 Error Handling

Every non-200 response returns:
```json
{ "error": "description string" }
```

Show a toast notification. All API calls should have loading states.

---

## 2. Endpoints

### 2.1 GET /catalogue

Returns all supported tokenized equity assets.

**Wallet required:** No

```typescript
interface CatalogueAsset {
  address: string            // ERC-20 token contract address, lowercase
  symbol: string             // e.g. "TSLA"
  name: string               // e.g. "Tesla Inc"
  sector: string             // e.g. "Consumer Discretionary"
  oracle: string             // OracleAdapter contract address, lowercase
  isActive: boolean
  currentPriceUsdg: string   // 8-decimal integer string e.g. "41555000000" = $415.55
  priceChange24hPct: string  // e.g. "2.45" or "-1.20" or "0.00"
}

Response: CatalogueAsset[]  // sorted alphabetically by symbol
```

**Price display:** `parseFloat(currentPriceUsdg) / 1e8` gives the dollar price.

**Sample response:**
```json
[
  {
    "address": "0x71178bac73cbeb415514eb542a8995b82669778d",
    "symbol": "AMD",
    "name": "Advanced Micro Devices Inc",
    "sector": "Technology",
    "oracle": "0xDaf7e6168A748A0348e8392d31377B486D9278Ab",
    "isActive": true,
    "currentPriceUsdg": "49701000000",
    "priceChange24hPct": "0.00"
  }
]
```

---

### 2.2 GET /catalogue/:address

Returns a single asset by its token contract address. Address matching is case-insensitive.

**Wallet required:** No

```typescript
interface CatalogueAssetDetail {
  address: string
  symbol: string
  name: string
  sector: string
  oracle: string
  isActive: boolean
  currentPriceUsdg: string   // 8-decimal integer string. "0" before first price poll
  priceChange24hPct: string  // "0.00" when insufficient history
}
```

Returns 404 `{ "error": "asset not found" }` when address has no match.

---

### 2.3 GET /prices

Returns latest price history entries for assets that are constituents of at least one active basket. Assets not held by any basket are not returned.

**Wallet required:** No

```typescript
interface PriceEntry {
  address: string           // token contract address, lowercase
  symbol: string
  priceUsdg: string         // 8-decimal integer string
  priceChange24hPct: string
  timestamp: number         // Unix timestamp of this price reading
}

Response: PriceEntry[]
```

Returns `[]` when no prices have been polled yet (e.g. immediately after deployment before the first poll cycle).

---

### 2.4 GET /baskets

Returns all baskets with their latest NAV and constituent summary.

**Wallet required:** No

```typescript
interface BasketSummary {
  address: string              // basket proxy contract address, lowercase
  creatorToken: string         // creator token contract address, lowercase
  creator: string              // creator wallet address, lowercase
  name: string                 // e.g. "AI Infrastructure"
  symbol: string               // e.g. "AIIB"
  thesis: string               // full thesis text
  rebalancingEnabled: boolean
  driftThresholdBps: number    // 0 when rebalancingEnabled is false
  createdAt: number            // Unix timestamp
  suspended: boolean
  navPerToken: string          // 18-decimal integer string
  totalValueUsdg: string       // 6-decimal integer string e.g. "9920147" = $9.92
  navChange24hPct: string      // "0.00" when insufficient history
  constituentCount: number
  constituents: {
    address: string
    symbol: string
    targetWeightBps: number    // integer e.g. 5000 = 50%
    sector: string
  }[]
}

Response: BasketSummary[]
```

---

### 2.5 GET /baskets/:address

Returns full basket detail including live state read from the contract.

**Wallet required:** No for viewing. Yes for deposit/redeem actions.

```typescript
interface BasketDetail {
  address: string
  creatorToken: string
  creator: string
  name: string
  symbol: string
  thesis: string
  rebalancingEnabled: boolean
  driftThresholdBps: number
  createdAt: number
  suspended: boolean
  navPerToken: string          // 18-decimal integer string
  totalValueUsdg: string       // 6-decimal integer string
  navChange24hPct: string
  navChange7dPct: string
  navChange30dPct: string
  maxDriftBps: number          // current maximum drift across all constituents
  needsRebalancing: boolean
  constituents: {
    address: string            // token contract address
    symbol: string
    name: string               // full company name e.g. "Tesla Inc"
    sector: string
    targetWeightBps: string    // string not number e.g. "5000"
    currentWeightBps: string   // string not number e.g. "4998"
    balanceRaw: string         // 18-decimal token balance string
    priceUsdg: string          // 8-decimal oracle price e.g. "41555000000" = $415.55. "0" before first price poll
    valueUsdg: string          // 6-decimal USDG value of this constituent = balanceRaw * priceUsdg / 1e20. "0" before first price poll
    priceChange24hPct: string  // formatted percentage e.g. "2.45" or "-1.20". "0.00" when insufficient history
  }[]
  performanceHistory: {
    navPerToken: string
    totalValueUsdg: string
    timestamp: number
  }[]
  rebalanceHistory: {
    timestamp: number
    txHash: string
    triggeredBy: string
  }[]
  depositHistory: {
    investor: string
    usdgAmount: string
    basketTokensMinted: string
    timestamp: number
    txHash: string
  }[]
}
```

Returns 404 `{ "error": "basket not found" }` for unknown addresses.

**On constituent weight fields:** `targetWeightBps` and `currentWeightBps` are returned as strings. Parse with `parseInt()` before arithmetic.

**On constituent price fields:** `priceUsdg` and `valueUsdg` are "0" before the first price poll cycle completes (60 seconds after startup). `priceChange24hPct` is "0.00" when fewer than 24 hours of price history exist for a constituent.

---

### 2.6 GET /baskets/:address/performance

Returns the full NAV history time series for a basket.

**Wallet required:** No

```typescript
interface PerformancePoint {
  navPerToken: string
  totalValueUsdg: string
  timestamp: number
}

Response: PerformancePoint[]  // ordered by timestamp ascending
```

Returns `[]` when no NAV history exists yet (basket was just created and the nav poller hasn't run).

---

### 2.7 GET /baskets/:address/positions/:wallet

Returns a single wallet's position in a specific basket.

**Wallet required:** Yes (pass connected wallet address as :wallet)

```typescript
interface InvestorPosition {
  basketAddress: string
  walletAddress: string
  basketName: string
  basketSymbol: string
  basketNavPerToken: string  // 18-decimal
  basketTokenBalance: string  // 18-decimal string e.g. "9950000000000000000"
  currentValueUsdg: string    // 6-decimal string e.g. "9920140"
  totalDepositedUsdg: string  // 6-decimal string, sum of all deposits
  unrealisedPnlUsdg: string   // 6-decimal string, may be negative e.g. "-79860"
  unrealisedPnlPct: string    // e.g. "-0.80"
  constituents: {
    address: string
    symbol: string
    targetWeightBps: number
    sector: string
  }[]
}
```

`basketTokenBalance` here is computed from the deposit/redemption event history in the database. Cross-check with a live contract `balanceOf` call for the redemption form.

---

### 2.8 GET /positions/:wallet

Returns a portfolio summary across all baskets for a wallet.

**Wallet required:** Yes

```typescript
interface PortfolioSummary {
  walletAddress: string
  totalValueUsdg: string
  totalDepositedUsdg: string
  totalUnrealisedPnlUsdg: string
  totalUnrealisedPnlPct: string
  positions: {
    basketAddress: string
    basketName: string
    basketSymbol: string
    basketNavPerToken: string
    rebalancingEnabled: boolean
    suspended: boolean
    basketTokenBalance: string
    currentValueUsdg: string
    totalDepositedUsdg: string
    unrealisedPnlUsdg: string
    unrealisedPnlPct: string
    constituents: {
      address: string
      symbol: string
      targetWeightBps: number
      sector: string
    }[]
  }[]
}
```

Returns a valid response with empty `positions[]` and zero totals when the wallet has no deposit history. Does not return 404.

---

### 2.9 GET /creator/:wallet

Returns creator dashboard data for all baskets created by the wallet.

**Wallet required:** Yes

```typescript
interface CreatorDashboard {
  walletAddress: string
  totalClaimableUsdg: string    // sum of claimableByWallet across all unclaimed snapshots
  baskets: {
    basketAddress: string
    basketName: string
    basketSymbol: string
    creatorTokenAddress: string
    totalValueUsdg: string
    totalClaimableUsdg: string  // claimable across all unclaimed snapshots for this basket
    unclaimedSnapshots: {
      snapshotId: number
      usdgAmount: string        // total fee amount in this snapshot
      timestamp: number
      claimableByWallet: string // wallet's proportional share based on creator token balance
      txHash: string
    }[]
    revenueHistory: {
      snapshotId: number
      usdgAmount: string
      timestamp: number
      txHash: string  // transaction hash — link to block explorer
    }[]
  }[]
}
```

Returns `{ "walletAddress": "0x...", "totalClaimableUsdg": "0", "baskets": [] }` when the wallet has created no baskets.

**Fields NOT returned by the backend — read from contract directly:**

`creatorTokenBalance`, `totalCreatorTokenSupply`, and `ownershipPct` are not in the API response. Read them from the contract:

```typescript
// Creator token balance for the connected wallet
const balance = await readContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'balanceOf',
  args: [walletAddress],
})
// Total supply is always 1_000_000 * 1e18
const totalSupply = 1_000_000n * 10n ** 18n
const ownershipPct = (balance * 10000n / totalSupply) // in bps
```

**Why these are not in the API:** These are live ERC-20 state values that change every time a creator token transfer occurs. Caching them in the backend would require indexing every Transfer event on every creator token contract. The contract read is instant and always accurate.

---

### 2.10 GET /creator-tokens/:address

Returns snapshot and revenue data for a specific creator token contract address.

**Wallet required:** No

```typescript
interface CreatorTokenDetail {
  creatorTokenAddress: string
  snapshots: {
    snapshotId: number
    usdgAmount: string
    timestamp: number
    txHash: string
  }[]
  totalRevenueUsdg: string
}
```

Returns `{ "creatorTokenAddress": "0x...", "snapshots": [], "totalRevenueUsdg": "0" }` when no snapshots exist.

---

### 2.11 POST /ai/compose

Generates a basket composition proposal from a natural language investment thesis. The AI model (OpenAI gpt-4.1-mini) runs server-side. No API key is needed on the frontend.

**Wallet required:** No

```typescript
// Request
interface ComposeRequest {
  thesis: string   // minimum 20 characters
}

// Response 200
interface AIProposal {
  constituents: {
    address: string          // token contract address from the catalogue
    symbol: string
    name: string
    sector: string
    weightBps: number        // integer e.g. 5000 = 50%
    rationale: string        // why this asset was selected
    currentPriceUsdg: string // 8-decimal string
  }[]
  overallRationale: string
  riskNotes: string
  provider: string           // e.g. "openai/gpt-4.1-mini"
}

// Response 400 — thesis too short or invalid JSON body
{ "error": "thesis must be at least 20 characters" }

// Response 503 — fewer than 3 active assets in catalogue (cannot compose)
{ "error": "not enough active assets in catalogue" }

// Response 502 — OpenAI API call failed
{ "error": "openai call: ..." }
```

The proposal always contains 3–12 constituents with weights summing to exactly 10,000 bps. All addresses are from the active catalogue. No individual weight is below 100 bps or above 5,000 bps.

Note: the AI caps proposals at 12 constituents for focus and coherence. The
contract itself supports up to `maxConstituents` (currently 20), so users can add more
constituents manually in the review step before deploying.

---

## 3. Contract Reads

These are `eth_call` — free, no gas, no wallet signature required. Call these for live on-chain state that supplements or cross-checks the backend API.

Where possible, batch multiple reads into a single multicall using viem's `useContractReads` hook to reduce RPC round trips.

### 3.1 WeaveRegistry reads

```typescript
// Management fee rate applied on every deposit and redemption
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'managementFeeBps' })
// returns: bigint — e.g. 50n = 0.5%

// Protocol's share of the management fee
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'protocolShareBps' })
// returns: bigint — e.g. 2000n = 20%

// Creator's share of the management fee (always 10000n - protocolShareBps)
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'creatorShareBps' })
// returns: bigint — e.g. 8000n = 80%

// Minimum initial deposit required to create a basket
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'minFirstDepositUsdg' })
// returns: bigint — 6-decimal USDG, e.g. 10000000n = $10

// Maximum number of constituents per basket
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'maxConstituents' })
// returns: bigint — currently 20n

// Minimum weight per constituent in basis points
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'minWeightBps' })
// returns: bigint — currently 100n = 1%

// Minimum rebalance trade leg size — legs below this are skipped (avoids dust gas waste)
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'minRebalanceTradeSizeUsdg' })
// returns: bigint — 6-decimal USDG, currently 1000000n = $1

// Maximum per-leg swap slippage on constituent purchases — enforced automatically by contracts
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'maxSwapSlippageBps' })
// returns: bigint — e.g. 100n = 1%

// Oracle staleness threshold in seconds — prices older than this revert
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'oracleStalenessSecs' })
// returns: bigint — currently 86400n = 24 hours

// Whether the protocol is globally paused — check before showing deposit/create forms
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'paused' })
// returns: boolean
// When true: disable deposit button, disable create basket flow, show a protocol-wide
// pause banner. Poll this on every page that has a deposit or create action.

// USDG token address (immutable — never changes)
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'usdg' })
// returns: address

// Protocol treasury address
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'protocolTreasury' })
// returns: address

// Whether an address is a registered basket
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'isBasket', args: [address] })
// returns: boolean

// Asset config for a token address
readContract({ address: REGISTRY_ADDRESS, abi: WeaveRegistryABI, functionName: 'assets', args: [tokenAddress] })
// returns: AssetConfig { tokenAddress, oracle, symbol, name, sector, active }
```

### 3.2 BasketProxy reads

Each basket is a proxy at its own address. Use `BasketImplementation` ABI — the proxy delegates to the shared implementation.

```typescript
// Basket token balance for a wallet — source of truth for redemption form max
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'balanceOf', args: [walletAddress] })
// returns: bigint — 18-decimal basket tokens

// Total basket token supply outstanding
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'totalSupply' })
// returns: bigint — 18-decimal

// NAV per basket token in USDG — cross-check against backend before showing estimates
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'navPerToken' })
// returns: bigint — 18-decimal USDG per basket token. Returns 0 when totalSupply is 0.

// Total AUM of the basket in USDG
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'totalValueUsdg' })
// returns: bigint — 6-decimal USDG

// Whether any constituent currently exceeds the drift threshold — always read live, never cache
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'needsRebalancing' })
// returns: boolean. Always false for static baskets or suspended baskets.

// Current maximum drift in basis points across all constituents
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'maxDrift' })
// returns: bigint — basis points

// Current weights of all constituents in bps computed from live oracle prices
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'currentWeightsBps' })
// returns: bigint[] — same order as constituents()

// Whether this basket has auto-rebalancing enabled
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'rebalancingEnabled' })
// returns: boolean

// Drift threshold in bps — rebalancing triggers when any constituent exceeds this
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'driftThresholdBps' })
// returns: bigint — 0 for static baskets

// Whether this basket is suspended (a constituent was deactivated by governance)
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'suspended' })
// returns: boolean

// Constituent token addresses — needed to build minAmountsOut for rebalance call
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'constituents' })
// returns: address[]

// Target weight in bps for each constituent — same order as constituents()
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'targetWeightsBps' })
// returns: bigint[]

// Current token balance of each constituent held by the basket
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'constituentBalances' })
// returns: bigint[] — 18-decimal token amounts, same order as constituents()

// Investment thesis text
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'thesis' })
// returns: string

// Creator wallet address
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'creator' })
// returns: address

// Creator token contract address for this basket
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'creatorToken' })
// returns: address

// Registry contract address
readContract({ address: basketAddress, abi: BasketProxyABI, functionName: 'registry' })
// returns: address

// Full basket state in one call — use this instead of multiple individual reads
const [
  constituents,
  targetWeights,
  currentWeights,
  balances,
  totalValue,
  nav,
  rebalancingEnabled,
  driftThresholdBps,
  maxDrift,
] = await readContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'basketState',
})
// constituents:        address[]  — constituent token addresses
// targetWeights:       bigint[]   — target weights in bps
// currentWeights:      bigint[]   — live weights computed from oracle prices
// balances:            bigint[]   — constituent token balances (18-decimal each)
// totalValue:          bigint     — total AUM in 6-decimal USDG
// nav:                 bigint     — NAV per token in 18-decimal USDG. 0 when supply is 0.
// rebalancingEnabled:  boolean
// driftThresholdBps:   bigint     — drift trigger threshold in bps
// maxDrift:            bigint     — current maximum drift in bps
```

### 3.3 CreatorToken reads

```typescript
// Creator token balance for a wallet
readContract({ address: creatorTokenAddress, abi: CreatorTokenABI, functionName: 'balanceOf', args: [walletAddress] })
// returns: bigint — 18-decimal creator tokens
// TOTAL_SUPPLY = 1_000_000n * 10n**18n (constant, never changes)
// ownershipPct in bps = balance * 10000n / (1_000_000n * 10n**18n)

// Total creator token supply (constant — always 1_000_000 * 1e18)
readContract({ address: creatorTokenAddress, abi: CreatorTokenABI, functionName: 'totalSupply' })
// returns: bigint — always 1_000_000_000_000_000_000_000_000n unless tokens have been burned

// Total number of revenue snapshots recorded so far
readContract({ address: creatorTokenAddress, abi: CreatorTokenABI, functionName: 'snapshotCount' })
// returns: bigint — snapshot IDs run from 1 to snapshotCount inclusive

// USDG claimable by a wallet for a specific snapshot ID
readContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'claimableRevenue',
  args: [walletAddress, snapshotId],  // snapshotId: bigint
})
// returns: bigint — 6-decimal USDG. Returns 0 if already claimed or snapshot does not exist.

// USDG that would be received if a wallet burned a given amount of creator tokens (ERC-7641)
readContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'redeemableOnBurn',
  args: [burnAmount],  // bigint — 18-decimal creator tokens
})
// returns: bigint — 6-decimal USDG
// Based on the current USDG pool balance held by the creator token contract
// Show this as a preview before the user calls burn()

// Number of balance checkpoints recorded for an account (off-chain tooling / debugging)
readContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'checkpointCount',
  args: [walletAddress],
})
// returns: bigint

// Basket address this creator token belongs to (immutable)
readContract({ address: creatorTokenAddress, abi: CreatorTokenABI, functionName: 'basket' })
// returns: address

// USDG token address (immutable)
readContract({ address: creatorTokenAddress, abi: CreatorTokenABI, functionName: 'usdg' })
// returns: address
```

### 3.4 ERC-20 reads

```typescript
// USDG allowance for a spender — check before deposit or basket creation
readContract({
  address: USDG_ADDRESS,
  abi: ERC20ABI,
  functionName: 'allowance',
  args: [walletAddress, spenderAddress],
  // spenderAddress = basketAddress for deposits
  // spenderAddress = BASKET_FACTORY_ADDRESS for basket creation
})
// returns: bigint — 6-decimal USDG
```

---

## 4. Contract Writes

These require a connected wallet and gas. Always show a transaction status toast with pending → confirmed → error states. Always `waitForTransactionReceipt` before updating UI state or navigating — a transaction hash does not mean the transaction succeeded.

### 4.1 USDG Approval

Check `allowance()` first — do not always re-approve.

```typescript
const current = await readContract({
  address: USDG_ADDRESS,
  abi: ERC20ABI,
  functionName: 'allowance',
  args: [walletAddress, spenderAddress],
})

if (current < requiredAmount) {
  writeContract({
    address: USDG_ADDRESS,
    abi: ERC20ABI,
    functionName: 'approve',
    args: [
      spenderAddress,  // basketAddress for deposits, BASKET_FACTORY_ADDRESS for creation
      requiredAmount,
    ],
  })
  // Wait for approval confirmation before submitting the actual transaction
}
```

### 4.2 createBasket

Deploys a basket proxy and creator token atomically. Validate all inputs client-side before submitting.

```typescript
writeContract({
  address: BASKET_FACTORY_ADDRESS,
  abi: BasketFactoryABI,
  functionName: 'createBasket',
  args: [
    name,                // string — basket display name
    symbol,              // string — basket token symbol (becomes "wCT-{symbol}" for creator token)
    thesis,              // string — investment thesis
    constituents,        // address[] — constituent token addresses from catalogue
    targetWeightsBps,    // uint256[] — must sum to exactly 10000
    rebalancingEnabled,  // bool
    driftThresholdBps,   // uint256 — 0 if static; 1–5000 if rebalancing enabled
    initialDepositUsdg,  // uint256 — 6-decimal USDG
  ],
})

// Validation rules enforced by BasketFactory (revert if violated):
//   constituents.length >= 3                         (TooFewConstituents)
//   constituents.length <= maxConstituents            (TooManyConstituents)
//   constituents.length === targetWeightsBps.length   (ArrayLengthMismatch)
//   no duplicate addresses                            (DuplicateConstituent)
//   every weight >= minWeightBps (currently 100)      (WeightTooLow)
//   every weight <= 5000                              (WeightTooHigh)
//   every address active in catalogue                 (AssetNotActive)
//   weights sum to exactly 10000                      (WeightSumInvalid)
//   initialDepositUsdg >= minFirstDepositUsdg         (InitialDepositTooLow)
//   if rebalancingEnabled: driftThresholdBps in 1–5000 (InvalidDriftThreshold)
//
// Prerequisite: USDG approved to BASKET_FACTORY_ADDRESS for at least initialDepositUsdg
//
// After confirmation: listen for BasketCreated event (see section 12) to get basket address
```

### 4.3 deposit

```typescript
writeContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'deposit',
  args: [
    usdgAmount,          // uint256 — 6-decimal USDG
    minBasketTokensOut,  // uint256 — slippage protection (see section 5.1)
    receiverAddress,     // address — who receives the basket tokens (usually walletAddress)
  ],
})
// Reverts: ZeroAmount, BasketSuspended, ProtocolPaused, InsufficientSlippage
// Prerequisite: USDG approved to basketAddress for at least usdgAmount
// Also check: paused() on registry before showing deposit form
```

### 4.4 redeem

```typescript
writeContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'redeem',
  args: [
    basketTokenAmount,   // uint256 — 18-decimal basket tokens
    minUsdgOut,          // uint256 — slippage protection (see section 5.2)
    receiverAddress,     // address — who receives the USDG (usually walletAddress)
  ],
})
// Reverts: ZeroAmount, BasketSuspended, ProtocolPaused, InsufficientSlippage
// No approval needed — the basket burns the caller's own tokens
```

### 4.5 rebalance

Permissionless — any wallet can call this. Only works when `rebalancingEnabled === true`
and `needsRebalancing()` returns true. Rebalancing is NOT blocked by protocol pause.
Always check `needsRebalancing()` live immediately before showing the button.

```typescript
const constituentsArr = await readContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'constituents',
})

// Passing all zeros tells the basket to use its own oracle-based slippage calculation
// (registry.maxSwapSlippageBps) for buy legs. Sell legs use the passed minAmountsOut.
// For user-initiated rebalance from the UI, zeros are appropriate.
writeContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'rebalance',
  args: [new Array(constituentsArr.length).fill(0n)],
})
// Reverts: RebalancingNotEnabled, BasketSuspended, DriftThresholdNotMet
```

### 4.6 collectFees

Permissionless — anyone can call. The basket calls this automatically at the start of
every deposit and redemption. Manual calls handle dust USDG accumulation from rebalancing rounding.

```typescript
writeContract({
  address: basketAddress,
  abi: BasketProxyABI,
  functionName: 'collectFees',
})
// No arguments. Distributes any free USDG sitting in the basket to the
// protocol treasury and creator token revenue pool.
// Safe to call at any time, no-ops if basket USDG balance is zero.
```

### 4.7 claimAll (creator token)

Claims all unclaimed revenue snapshots for a creator token in one transaction.
Silently skips snapshots with nothing to claim (zero balance at snapshot block).
Reverts with `NothingToClaim` if there is genuinely nothing across all snapshots.

```typescript
writeContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'claimAll',
})
// If the creator has multiple baskets, call claimAll on each creator token separately
// Show progress: "Claiming from basket 1 of N..."
```

### 4.8 claim (single snapshot)

```typescript
writeContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'claim',
  args: [snapshotId],  // uint256 — from 1 to snapshotCount
})
// Reverts: SnapshotDoesNotExist, AlreadyClaimed, NothingToClaim
```

### 4.9 burn (ERC-7641)

Burns creator tokens and redeems a proportional share of the accumulated USDG revenue pool.
This permanently reduces total supply. The burned share is proportional to the pool balance
at the moment of burning — not historical snapshots.

```typescript
// Show the user what they will receive before they commit
const redeemable = await readContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'redeemableOnBurn',
  args: [burnAmountBigInt],  // 18-decimal creator tokens
})
// redeemable: bigint — 6-decimal USDG

writeContract({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  functionName: 'burn',
  args: [burnAmountBigInt],
})
// Reverts: ZeroAmount, InsufficientBalance
// Note: burns reduce totalSupply permanently. After burn, remaining holders'
// share of future snapshots increases proportionally.
```



---

## 5. Deposit and Redemption Math

### 5.1 Estimated basket tokens from deposit

```typescript
const usdgBig     = BigInt(usdgAmountRaw)
const feeBps      = await readContract({ functionName: 'managementFeeBps' })
const netUsdg     = usdgBig - (usdgBig * feeBps / 10_000n)
const totalValue  = await readContract({ address: basketAddress, functionName: 'totalValueUsdg' })
const totalSupply = await readContract({ address: basketAddress, functionName: 'totalSupply' })

let estimated: bigint
if (totalSupply === 0n) {
  // First depositor edge case — contract establishes 1 USDG = 1 basket token
  // USDG is 6-decimal, basket tokens are 18-decimal — multiply by 1e12 to normalise
  estimated = netUsdg * 1_000_000_000_000n
} else {
  estimated = netUsdg * totalSupply / totalValue
}

const minBasketTokensOut = estimated * 95n / 100n  // 5% slippage
```

When `totalSupply === 0n`, show "Initial price: 1 USDG = 1 basket token" in the UI.

### 5.2 Estimated USDG from redemption

```typescript
const navBig = BigInt(navPerToken)  // 18-decimal
const tokensBig = BigInt(basketTokenAmount)  // 18-decimal
// currentValue = tokensBig * navBig / 1e18
const estimatedUsdg = tokensBig * navBig / 10n**18n
```

Apply 5% slippage for `minUsdgOut`:
```typescript
const minUsdgOut = estimatedUsdg * 95n / 100n
```

### 5.3 Management fee display

```typescript
const feeBps = await readContract({ functionName: 'managementFeeBps' })
// fee = amount * feeBps / 10000
const feeAmount = usdgAmountBigInt * feeBps / 10000n
// 80% goes to creator, 20% to protocol
const creatorShare = feeAmount * 8000n / 10000n
const protocolShare = feeAmount * 2000n / 10000n
```

---

## 6. Revert Reason Decoding

Every custom error is verified from the actual Solidity source.

```typescript
import { decodeErrorResult } from 'viem'

try {
  await writeContract({ ... })
} catch (error) {
  if (error.cause?.data) {
    const decoded = decodeErrorResult({
      abi: [...BasketProxyABI, ...BasketFactoryABI, ...CreatorTokenABI, ...WeaveRegistryABI],
      data: error.cause.data,
    })
    showToast(ERROR_MESSAGES[decoded.errorName] ?? 'Transaction failed.')
  }
}
```

**BasketFactory errors:**

| Error | User-Facing Message |
|---|---|
| `TooFewConstituents(uint256)` | "A basket needs at least 3 constituents." |
| `TooManyConstituents(uint256, uint256)` | "Maximum number of constituents exceeded." |
| `ArrayLengthMismatch()` | "Constituent and weight arrays must be the same length." |
| `WeightSumInvalid(uint256)` | "Constituent weights must sum to exactly 100%." |
| `WeightTooLow(address, uint256)` | "Each constituent must have at least 1% weight." |
| `WeightTooHigh(address, uint256)` | "No single constituent can exceed 50% weight." |
| `AssetNotActive(address)` | "One or more constituents are not active in the catalogue." |
| `InitialDepositTooLow(uint256, uint256)` | "Initial deposit is below the minimum required." |
| `InvalidDriftThreshold()` | "Drift threshold must be between 1 and 5000 bps (0.01%–50%)." |
| `DuplicateConstituent(address)` | "Duplicate constituent found in basket composition." |

**BasketImplementation errors:**

| Error | User-Facing Message |
|---|---|
| `BasketSuspended()` | "This basket is suspended and cannot accept deposits or redemptions." |
| `ProtocolPaused()` | "The protocol is temporarily paused. Please try again later." |
| `RebalancingNotEnabled()` | "This basket does not have auto-rebalancing enabled." |
| `DriftThresholdNotMet()` | "This basket does not currently need rebalancing." |
| `InsufficientSlippage()` | "Price moved too much during execution. Please try again." |
| `ZeroAmount()` | "Amount cannot be zero." |

**WeaveRegistry errors:**

| Error | User-Facing Message |
|---|---|
| `StalePrice(address)` | "Price data is temporarily unavailable. Please try again." |
| `NegativePrice(address)` | "Invalid price data from oracle. Please try again." |
| `AssetNotActive(address)` | "This asset is no longer active in the catalogue." |
| `AssetNotFound(address)` | "Asset not found in the catalogue." |
| `InvalidFeeBps()` | "Fee exceeds the maximum allowed (10%)." |
| `InvalidFeeSplit()` | "Fee split is invalid." |
| `InvalidParameter()` | "Invalid parameter value." |
| `ProtocolPaused()` | "The protocol is temporarily paused. Please try again later." |

**CreatorToken errors:**

| Error | User-Facing Message |
|---|---|
| `AlreadyClaimed(address, uint256)` | "You have already claimed this snapshot." |
| `SnapshotDoesNotExist(uint256)` | "This snapshot does not exist." |
| `NothingToClaim()` | "No claimable revenue for your current creator token balance." |
| `InsufficientBalance(uint256, uint256)` | "Insufficient creator token balance for this burn amount." |
| `ZeroAmount()` | "Amount cannot be zero." |

**SwapRouter errors (surfaces through deposit/redeem/rebalance):**

| Error | User-Facing Message |
|---|---|
| `InsufficientLiquidity(address, uint256, uint256)` | "Insufficient liquidity for this constituent. Please try a smaller deposit or try again later." |
| `InsufficientOutput(uint256, uint256)` | "Swap output is below the minimum acceptable. Please try again." |

---


---

## 7. Number Display Rules

Never use JavaScript `number` for financial arithmetic. All values from the API are strings representing big integers.

```typescript
// WRONG
const price = parseFloat(asset.currentPriceUsdg) / 1e8

// RIGHT — only convert to float at the final display step
const priceDisplay = (BigInt(asset.currentPriceUsdg) * 100n / 100000000n)
// then format: "$" + (Number(priceDisplay) / 100).toFixed(2)
```

Formatting rules:
- USDG amounts (6-decimal): divide by 1e6, display as `$X,XXX.XX`
- Oracle prices (8-decimal): divide by 1e8, display as `$X,XXX.XX`
- Basket token amounts (18-decimal): divide by 1e18, display with 4 decimal places
- Percentage strings: already formatted — prepend `+` when positive, colour green/red
- All monetary values use comma separators

---

## 8. Polling and Cache Behaviour

| Data | Freshness | Recommendation |
|------|-----------|----------------|
| Catalogue prices | Updated every 60 seconds by backend price poller | Refetch /catalogue every 60s on price-sensitive pages |
| Basket NAV | Updated every 5 minutes by backend NAV poller | Refetch /baskets/:address every 30s (backend cache is 30s) |
| Basket list NAVs | Same as above | Refetch /baskets every 60s on marketplace |
| Position values | Derived from NAV — same freshness | Refetch /positions every 60s |
| Creator claimable | Cached in backend for 60s | Refetch /creator every 60s |
| Performance history | Grows every 5 minutes | Refetch on page focus |

---

## 9. BasketCreated Event Listening

After `createBasket` is confirmed, listen for the `BasketCreated` event from the factory to get the deployed basket address:

```typescript
const unwatch = watchContractEvent({
  address: BASKET_FACTORY_ADDRESS,
  abi: BasketFactoryABI,
  eventName: 'BasketCreated',
  onLogs: (logs) => {
    const log = logs.find(l => l.args.creator?.toLowerCase() === walletAddress.toLowerCase())
    if (log) {
      unwatch()
      router.push(`/baskets/${log.args.basket}`)
    }
  },
})
```

The event emits: `basket` (proxy address), `creatorToken`, `creator`.

---

## 10. Suspended Basket Behaviour

When `suspended = true` on a basket:
- Show a prominent warning banner on the basket detail page
- Disable the deposit form entirely
- Disable the redeem form (redemption may also be blocked at contract level)
- Show a warning on the portfolio position card
- Still show the basket in the marketplace but mark it visually

---

## 11. Static vs Auto-Rebalancing Baskets

When `rebalancingEnabled = false`:
- `driftThresholdBps` is 0 — ignore it
- Do not show DriftIndicator or RebalanceNowButton
- Show "Static" badge

When `rebalancingEnabled = true`:
- `driftThresholdBps` is the configured threshold in bps
- Show DriftIndicator using `maxDriftBps` vs `driftThresholdBps`
- Show RebalanceNowButton only when `needsRebalancing = true`
- Show "Auto-Rebalancing" badge

---

## 12. Event Listening

All events verified from the actual Solidity source.

```typescript
// BasketFactory — BasketCreated
// Fired when a new basket is deployed. Use to navigate after createBasket confirms.
// event BasketCreated(
//   address indexed basket,
//   address indexed creatorToken,
//   address indexed creator,
//   string name,
//   string symbol,
//   string thesis,
//   address[] constituents,
//   uint256[] targetWeightsBps,
//   bool rebalancingEnabled
// )
watchContractEvent({
  address: BASKET_FACTORY_ADDRESS,
  abi: BasketFactoryABI,
  eventName: 'BasketCreated',
  onLogs: (logs) => {
    const log = logs.find(l => l.args.creator?.toLowerCase() === walletAddress.toLowerCase())
    if (log) router.push(`/baskets/${log.args.basket}`)
  },
})

// BasketImplementation — Deposited
// event Deposited(address indexed investor, uint256 usdgAmount, uint256 basketTokensMinted, uint256 feeUsdg)
watchContractEvent({
  address: basketAddress,
  abi: BasketProxyABI,
  eventName: 'Deposited',
  args: { investor: walletAddress },
  onLogs: () => { /* refresh basket token balance and portfolio */ },
})

// BasketImplementation — Redeemed
// event Redeemed(address indexed investor, uint256 basketTokensBurned, uint256 usdgReturned, uint256 feeUsdg)
watchContractEvent({
  address: basketAddress,
  abi: BasketProxyABI,
  eventName: 'Redeemed',
  args: { investor: walletAddress },
  onLogs: () => { /* refresh basket token balance and USDG balance */ },
})

// BasketImplementation — Rebalanced
// event Rebalanced(address indexed triggeredBy)
watchContractEvent({
  address: basketAddress,
  abi: BasketProxyABI,
  eventName: 'Rebalanced',
  onLogs: () => { /* refresh basketState(), needsRebalancing() will now return false */ },
})

// BasketImplementation — Suspended
// event Suspended()
// Fired when a constituent is deactivated and the basket auto-suspends on next interaction
watchContractEvent({
  address: basketAddress,
  abi: BasketProxyABI,
  eventName: 'Suspended',
  onLogs: () => { /* show suspension banner, disable deposit button */ },
})

// CreatorToken — RevenueSnapshoted
// event RevenueSnapshoted(uint256 indexed snapshotId, uint256 usdgAmount, uint256 blockNumber)
watchContractEvent({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  eventName: 'RevenueSnapshoted',
  onLogs: () => { /* refresh creator dashboard claimable amounts */ },
})

// CreatorToken — RevenueClaimed
// event RevenueClaimed(address indexed account, uint256 indexed snapshotId, uint256 usdgAmount)
watchContractEvent({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  eventName: 'RevenueClaimed',
  args: { account: walletAddress },
  onLogs: () => { /* refresh claimable amounts */ },
})

// CreatorToken — TokensBurned
// event TokensBurned(address indexed account, uint256 amount, uint256 usdgRedeemed)
watchContractEvent({
  address: creatorTokenAddress,
  abi: CreatorTokenABI,
  eventName: 'TokensBurned',
  args: { account: walletAddress },
  onLogs: () => { /* refresh creator token balance and USDG balance */ },
})

// WeaveRegistry — ProtocolPausedEvent
// event ProtocolPausedEvent(address indexed by)
watchContractEvent({
  address: REGISTRY_ADDRESS,
  abi: WeaveRegistryABI,
  eventName: 'ProtocolPausedEvent',
  onLogs: () => { /* show protocol-wide pause banner, disable all deposit/create actions */ },
})

// WeaveRegistry — ProtocolUnpausedEvent
// event ProtocolUnpausedEvent(address indexed by)
watchContractEvent({
  address: REGISTRY_ADDRESS,
  abi: WeaveRegistryABI,
  eventName: 'ProtocolUnpausedEvent',
  onLogs: () => { /* hide pause banner, re-enable actions */ },
})
```

---


---

## 13. Call Patterns by Page

Quick reference for which calls each page needs.

**Marketplace (`/`)** — no wallet required, no contract calls. All data from `GET /baskets`.

**Basket Detail (`/baskets/:address`):**
```
On load (no wallet):
  GET /baskets/:address
  paused()                    → show protocol pause banner if true
  basketState()               → live weights, NAV, drift
  needsRebalancing()          → show/hide Rebalance Now button

On load (wallet connected, additionally):
  balanceOf(walletAddress)                         → basket token balance
  GET /baskets/:address/positions/:wallet          → cost basis and PnL
  allowance(wallet, basketAddress)                 → pre-check for deposit form

Deposit flow:
  allowance() → if insufficient: approve() → wait → deposit() → wait → refresh balanceOf()

Redeem flow:
  balanceOf() → populate max → redeem() → wait → refresh balanceOf()

Rebalance:
  needsRebalancing() → constituents() → rebalance() → wait → refresh basketState()
```

**Create Basket (`/create`):**
```
Step 1 — Thesis:   POST /ai/compose
Step 2 — Compose:  GET /catalogue · minWeightBps() · maxConstituents()
Step 3 — Config:   managementFeeBps() · protocolShareBps() · minFirstDepositUsdg()
Step 4 — Deploy:   allowance() → approve() → wait → createBasket() → listen BasketCreated
```

**Portfolio (`/portfolio`):**
```
GET /positions/:wallet
balanceOf(wallet) per basket  → live balance cross-check
navPerToken() per basket      → current value cross-check
```

**Creator Dashboard (`/creator`):**
```
GET /creator/:wallet
balanceOf(wallet) per creatorToken    → ownership percentage
snapshotCount() per creatorToken      → total snapshot count
claimableRevenue(wallet, snapshotId)  → claimable per unclaimed snapshot

Claim flow:
  claimAll() per creator token → show progress "Claiming N of M" → refresh after each
```

---

## 14. Implementation Notes

**BigInt everywhere.** All `uint256` values from contracts come back as `bigint` in viem. Never cast to `number` until the final display step — `Number(bigint)` loses precision for values above `Number.MAX_SAFE_INTEGER`.

**Transaction confirmation.** Always `waitForTransactionReceipt` before updating state or navigating. A transaction hash being available does not mean the transaction succeeded.

**Multicall batching.** Use viem's `useContractReads` hook to batch multiple read calls into one RPC request. Do this on the basket detail page where `basketState()`, `needsRebalancing()`, `balanceOf()`, and `allowance()` are all needed on load.

**Polling intervals.** Poll `basketState()` and `needsRebalancing()` every 30 seconds on the basket detail page. Do not poll more frequently — oracle prices update every 60 seconds. Match the backend cache TTL.

**ABI generation.** Generate ABIs from Solidity source at build time:
```bash
forge inspect src/WeaveRegistry.sol:WeaveRegistry abi > abis/WeaveRegistry.json
forge inspect src/BasketImplementation.sol:BasketImplementation abi > abis/BasketImplementation.json
forge inspect src/BasketFactory.sol:BasketFactory abi > abis/BasketFactory.json
forge inspect src/CreatorToken.sol:CreatorToken abi > abis/CreatorToken.json
```

---

## 15. Contract Addresses (Robinhood Chain Testnet — Chain ID 46630)

| Contract | Address |
|---|---|
| WeaveRegistry | `0x19Ab3408af6503a7D4BeC255b064f8B02A345D04` |
| BasketFactory | `0xE9854c4734cd4A9dbC5086398A11df3c11f40b21` |
| BasketImplementation | `0x1aceE18129477c0312228d306fD02313E9767F4E` |
| SwapRouter | `0x2953A82d44fDACfa7a49BfFF24f7Cc5879F10805` |
| WeaveAutomation | `0xCc10bF2f35ae47B4F946db8dc31f0f56b5b7D186` |
| OracleAdapter (TSLA) | `0xb4cCD5Aff61fCc5f87b2C79A777992561C4d5BD9` |
| OracleAdapter (AMZN) | `0xe2a8fa094812435B74c1b2081Ba35630b9b1Cbb7` |
| OracleAdapter (PLTR) | `0x0B01F4D56b39c534cDab9eCe15708E249bA8FC36` |
| OracleAdapter (NFLX) | `0xb98Fb1AC54dc33d33F6937b3ce8B93baE4f4F005` |
| OracleAdapter (AMD) | `0xDaf7e6168A748A0348e8392d31377B486D9278Ab` |

**Robinhood Chain stock tokens (canonical, not Weave-deployed):**

| Token | Address |
|---|---|
| USDG | `0x7E955252E15c84f5768B83c41a71F9eba181802F` |
| TSLA | `0xC9f9c86933092BbbfFF3CCb4b105A4A94bf3Bd4E` |
| AMZN | `0x5884aD2f920c162CFBbACc88C9C51AA75eC09E02` |
| PLTR | `0x1FBE1a0e43594b3455993B5dE5Fd0A7A266298d0` |
| NFLX | `0x3b8262A63d25f0477c4DDE23F83cfe22Cb768C93` |
| AMD | `0x71178BAc73cBeb415514eB542a8995b82669778d` |

**Block explorer:** `https://explorer.testnet.chain.robinhood.com`

Transaction link pattern: `https://explorer.testnet.chain.robinhood.com/tx/${txHash}`

---

## 16. Admin Dashboard (Governance — Separate Frontend)

The following functions are restricted to the governance address (`0x4e4B989abE79381C1B8a4871d6aF481B175F4865` for this deployment). They belong in a separate admin interface, not the product frontend.

**WeaveRegistry governance writes (verified from source):**

```typescript
// Pause all basket deposits and redemptions — emergency use only
// Rebalancing is NOT paused (reducing drift during a pause is safe)
writeContract({ functionName: 'pauseAll' })
// event: ProtocolPausedEvent(address indexed by)

// Resume normal protocol operation
writeContract({ functionName: 'unpauseAll' })
// event: ProtocolUnpausedEvent(address indexed by)

// Add a new asset to the catalogue
writeContract({
  functionName: 'addAsset',
  args: [{
    tokenAddress,  // address
    oracle,        // address — OracleAdapter or Chainlink aggregator
    symbol,        // string
    name,          // string
    sector,        // string
    active,        // bool — true
  }],
})
// event: AssetAdded(address indexed token, string symbol, string name, string sector, address oracle)

// Deactivate an asset — suspends all baskets holding it on next interaction
writeContract({ functionName: 'deactivateAsset', args: [tokenAddress] })
// event: AssetDeactivated(address indexed token)

writeContract({ functionName: 'reactivateAsset', args: [tokenAddress] })
// event: AssetReactivated(address indexed token);

// Set management fee (max 1000 bps = 10%)
writeContract({ functionName: 'setManagementFee', args: [feeBps] })

// Set protocol/creator fee split
writeContract({ functionName: 'setFeeSplit', args: [protocolShareBps] })
// e.g. 2000 = protocol gets 20%, creator gets 80%

// Set minimum AUM for protocol-funded Chainlink automation
writeContract({ functionName: 'setMinAUM', args: [minAumUsdg] })

// Set oracle staleness threshold (must be > 0)
writeContract({ functionName: 'setOracleStaleness', args: [seconds] })

// Set minimum rebalance trade leg size
writeContract({ functionName: 'setMinRebalanceTradeSize', args: [usdgAmount] })

// Set maximum per-leg swap slippage (max 1000 bps = 10%)
writeContract({ functionName: 'setMaxSwapSlippage', args: [bps] })

// Set minimum first deposit amount (must be > 0)
writeContract({ functionName: 'setMinFirstDeposit', args: [usdgAmount] })

// Set maximum number of constituents per basket (must be > 0)
writeContract({ functionName: 'setMaxConstituents', args: [max] })

// Set minimum weight per constituent in bps (must be > 0 and < 10000)
writeContract({ functionName: 'setMinWeightBps', args: [bps] })

// Set swap router address
writeContract({ functionName: 'setSwapRouter', args: [routerAddress] })

// Set basket factory address
writeContract({ functionName: 'setBasketFactory', args: [factoryAddress] })

// Set automation contract address
writeContract({ functionName: 'setAutomationContract', args: [automationAddress] })

// Set protocol treasury address
writeContract({ functionName: 'setProtocolTreasury', args: [treasuryAddress] })

// Two-step governance transfer — step 1: nominate
writeContract({ functionName: 'nominateGovernance', args: [nomineeAddress] })

// Two-step governance transfer — step 2: accept (must be called by nominee)
writeContract({ functionName: 'acceptGovernance' })
```

**WeaveAutomation governance writes:**

```typescript
// Tune how many baskets are checked per upkeep call (default 50)
writeContract({ address: AUTOMATION_ADDRESS, abi: WeaveAutomationABI, functionName: 'setBatchSize', args: [newSize] })

// Set maximum slippage on automation-triggered rebalance sell legs (max 500 bps = 5%, default 50 bps)
writeContract({ address: AUTOMATION_ADDRESS, abi: WeaveAutomationABI, functionName: 'setMaxRebalanceSlippage', args: [bps] })
```

**SwapRouter owner writes (not governance — owner is the deployer EOA):**

```typescript
// Fund the router treasury with a token
writeContract({ address: SWAP_ROUTER_ADDRESS, abi: SwapRouterABI, functionName: 'fund', args: [tokenAddress, amount] })

// Withdraw a specific amount of a token back to owner
writeContract({ address: SWAP_ROUTER_ADDRESS, abi: SwapRouterABI, functionName: 'withdraw', args: [tokenAddress, amount] })

// Withdraw all token balances back to owner
writeContract({ address: SWAP_ROUTER_ADDRESS, abi: SwapRouterABI, functionName: 'withdrawAll', args: [tokenAddressArray] })

// Set swap spread in bps (max 500 bps = 5%, default 30 bps = 0.3%)
writeContract({ address: SWAP_ROUTER_ADDRESS, abi: SwapRouterABI, functionName: 'setSpread', args: [bps] })
```
