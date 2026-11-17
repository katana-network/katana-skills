---
name: lending
description: Activate when the user asks about lending, borrowing, Morpho markets, vaults, positions, leverage, looping strategies, or yield farming on Katana Network.
license: MIT
metadata:
  author: katana
  version: '2.0.0'
---

# Lending — Morpho on Katana

Lending, borrowing, leveraged loops, and vault deposits on Morpho Blue on Katana Network.

## Morpho Blue Overview

Morpho Blue is a permissionless lending protocol with two modes:

1. **Direct Markets** — isolated pairs with a specific collateral token, loan token, oracle, IRM (interest rate model), and LLTV (liquidation loan-to-value). Users supply loan tokens to earn interest or deposit collateral to borrow.
2. **Vaults** — ERC-4626 vaults managed by curators that allocate deposits across markets. Simpler: just deposit and earn. Two generations exist on Katana:
   - **MetaMorpho (v1)** — allocates directly across Morpho Blue markets.
   - **Vault V2** — allocates through *adapters* (e.g. a MorphoMarketV1 adapter = Morpho Blue markets) under per-adapter, per-collateral and per-market caps.

**Read data from the Morpho API first** (see [Morpho API](#morpho-api-recommended-for-reads)): one request returns every Katana market or vault with USD values, APYs, listing status and risk warnings. On-chain reads remain the source of truth for building transactions and the fallback when the API is unavailable.

### Key Concepts

- **Market ID**: bytes32 hex string uniquely identifying a market (derived from `keccak256(abi.encode(MarketParams))`)
- **LLTV**: Liquidation Loan-to-Value ratio. If `borrowValue / collateralValue > LLTV`, the position is liquidatable. Higher LLTV = more leverage capacity but more risk.
- **Health Factor**: `(collateralValue × LLTV) / debtValue`. Below 1.0 = liquidatable. **Keep above 1.5 for safety.**
- **Utilization**: `totalBorrow / totalSupply`. Higher utilization = higher borrow rates but less available liquidity.

### MarketParams Struct

All market-specific functions require a `MarketParams` struct:

```
struct MarketParams {
  address loanToken;
  address collateralToken;
  address oracle;
  address irm;
  uint256 lltv;
}
```

### Contract Addresses (Mainnet)

| Contract | Address |
|----------|---------|
| Morpho Core | `0xD50F2DffFd62f94Ee4AEd9ca05C61d0753268aBc` |
| Bundler3 | `0xA8C5e23C9C0DF2b6fF716486c6bBEBB6661548C8` |
| GeneralAdapter1 | `0x916Aa175C36E845db45fF6DDB886AE437d403B61` |
| MetaMorphoFactory | `0x1c8de6889acee12257899bfeaa2b7e534de32e16` |
| MetaMorphoV1_1Factory | `0xd3f39505d0c48AFED3549D625982FdC38Ea9904b` |
| AdaptiveCurveIrm | `0x4F708C0ae7deD3d74736594C2109C2E3c065B428` |
| OracleFactory | `0x7D047fB910Bc187C18C81a69E30Fa164f8c536eC` |
| PublicAllocator | `0x39EB6Da5e88194C82B13491Df2e8B3E213eD2412` |

## Contracts & Functions

### Market Discovery

**Preferred:** query the Morpho API `markets` (see [Morpho API](#morpho-api-recommended-for-reads)) — it includes state, USD values, APYs, `listed` and `warnings` in one request.

**On-chain fallback:** scan `CreateMarket` events on Morpho Core to discover all markets:

```
event CreateMarket(bytes32 indexed id, MarketParams marketParams)
```

Read market state:
```
market(bytes32 id) → (
  uint128 totalSupplyAssets,
  uint128 totalSupplyShares,
  uint128 totalBorrowAssets,
  uint128 totalBorrowShares,
  uint128 lastUpdate,
  uint128 fee
)
```

Read market parameters:
```
idToMarketParams(bytes32 id) → MarketParams
```

Filter out dead markets (0 supply + 0 borrow). Calculate utilization as `totalBorrowAssets / totalSupplyAssets`.

### Vault Discovery

**Preferred:** query the Morpho API `vaults` (v1) and `vaultV2s` (V2). The on-chain factory scan below only finds v1 vaults.

**On-chain fallback (v1):** scan `CreateMetaMorpho` events from MetaMorphoFactory and MetaMorphoV1_1Factory to find all vaults. Vaults are standard ERC-4626, so call:
- `asset() → address` — the underlying token
- `totalAssets() → uint256` — TVL
- `name() → string` / `symbol() → string`

### Position Queries

Call on Morpho Core:
```
position(bytes32 id, address user) → (
  uint256 supplyShares,
  uint128 borrowShares,
  uint128 collateral
)
```

Convert shares to assets using market state:
- `supplyAssets = supplyShares × totalSupplyAssets / totalSupplyShares`
- `borrowAssets = borrowShares × totalBorrowAssets / totalBorrowShares`

**If health factor < 1.0, the position is liquidatable.**

### Supply (Lend)

Call on Morpho Core:
```
supply(
  MarketParams memory marketParams,
  uint256 assets,
  uint256 shares,     // pass 0 to supply by asset amount
  address onBehalf,
  bytes memory data   // pass empty bytes
) → (uint256 assetsSupplied, uint256 sharesSupplied)
```

**Requires approval:** User must call `approve()` on the loan token for Morpho Core (`0xD50F...8aBc`).

### Withdraw

```
withdraw(
  MarketParams memory marketParams,
  uint256 assets,
  uint256 shares,     // pass 0 to withdraw by asset amount
  address onBehalf,
  address receiver
) → (uint256 assetsWithdrawn, uint256 sharesWithdrawn)
```

May fail if insufficient liquidity in the market (totalSupply - totalBorrow < withdrawal amount).

### Borrow

```
borrow(
  MarketParams memory marketParams,
  uint256 assets,
  uint256 shares,     // pass 0 to borrow by asset amount
  address onBehalf,
  address receiver
) → (uint256 assetsBorrowed, uint256 sharesBorrowed)
```

Requires sufficient collateral already deposited. The LLTV constrains maximum borrow. **Check health factor after borrowing.**

### Supply Collateral

```
supplyCollateral(
  MarketParams memory marketParams,
  uint256 assets,
  address onBehalf,
  bytes memory data   // pass empty bytes
)
```

**Requires approval** for the collateral token to Morpho Core.

### Vault Detail & Risk Inspection

MetaMorpho vaults allocate across multiple Morpho Blue markets. To understand a vault's composition and withdrawal risk, read these on-chain:

```
withdrawQueueLength() → uint256
withdrawQueue(uint256 index) → bytes32 marketId
supplyQueueLength() → uint256
supplyQueue(uint256 index) → bytes32 marketId
config(bytes32 id) → (uint184 cap, bool enabled, uint64 removableAt)
totalAssets() → uint256
fee() → uint256
curator() → address
timelock() → uint256
guardian() → address
```

For each market in the withdraw queue, read its `position()` on Morpho Core (with the vault address as user) and `market()` state to calculate:
- **Vault supply per market**: convert the vault's supply shares to assets
- **Market utilization**: `totalBorrow / totalSupply`
- **Vault withdrawable per market**: `min(vaultSupplyAssets, marketLiquidity)` where `marketLiquidity = totalSupply - totalBorrow`
- **Aggregate withdrawal liquidity**: sum of per-market withdrawable amounts vs vault TVL

### Morpho API (recommended for reads)

Morpho's public API indexes every Katana market, MetaMorpho (v1) vault and Vault V2, with USD values, APYs (current and averaged, with and without rewards), listing status, risk warnings and user positions. Morpho recommends it as the default for reads ([docs](https://docs.morpho.org/api)).

```
GraphQL endpoint: https://api.morpho.org/graphql     (keyless)
```

**Always filter to Katana: `chainId: 747474`.** The API is multi-chain, so an unfiltered query returns markets and vaults from other networks. Bokuto testnet is **not** indexed — use on-chain reads on testnet. (`https://blue-api.morpho.org/graphql` is the older URL for the same API; prefer `api.morpho.org`.)

**Limits:** 750 requests/minute per IP and a per-query complexity budget (keep large list queries to one per request); abuse can trigger a 7-day IP block. Cache results (≈60s is plenty for dashboards) and don't poll. Values are fractions: `0.0273` = 2.73%. Token amounts are raw integer strings — divide by `10^decimals`.

**All markets on Katana:**
```graphql
{
  markets(first: 1000, where: { chainId_in: [747474] }, orderBy: SupplyAssetsUsd, orderDirection: Desc) {
    items {
      marketId listed lltv irmAddress
      oracle { address }
      loanAsset { address symbol decimals }
      collateralAsset { address symbol decimals }
      state {
        supplyAssets borrowAssets liquidityAssets utilization
        supplyAssetsUsd borrowAssetsUsd
        supplyApy borrowApy netSupplyApy netBorrowApy
      }
      warnings { type level }
    }
  }
}
```
Specific markets: add `uniqueKey_in: ["0x…", "0x…"]` to `where`, or use `marketById(marketId: "0x…", chainId: 747474)`.

**All vaults on Katana** — v1 and V2 are separate queries. Send them as **two requests**: combining both lists (with `first: 1000` each) in one request exceeds the API's query-complexity limit and is rejected with "Query is too complex".

MetaMorpho (v1):
```graphql
{
  vaults(first: 1000, where: { chainId_in: [747474] }, orderBy: TotalAssetsUsd, orderDirection: Desc) {
    items {
      address name symbol listed
      asset { address symbol decimals }
      state { totalAssets totalAssetsUsd apy netApy fee curator owner }
      warnings { type level }
    }
  }
}
```

Vault V2:
```graphql
{
  vaultV2s(first: 1000, where: { chainId_in: [747474] }, orderBy: TotalAssetsUsd, orderDirection: Desc) {
    items {
      address name symbol listed
      asset { address symbol decimals }
      totalAssets totalAssetsUsd liquidity liquidityUsd apy netApy performanceFee managementFee
      curator { address } owner { address }
      warnings { type level }
    }
  }
}
```

**One vault in detail:**
- v1: `vaultByAddress(address: "0x…", chainId: 747474)` → `state { allocation { market { marketId lltv loanAsset { symbol } collateralAsset { symbol } state { supplyAssets borrowAssets liquidityAssets utilization } } supplyAssets supplyCap } }`
- V2: `vaultV2ByAddress(address: "0x…", chainId: 747474)` → `adapters { items { address type assets assetsUsd } }`, `caps { items { type absoluteCap relativeCap allocation } }`, `idleAssets`, `liquidity`

**A user's risk on a market:**
```graphql
{
  marketPosition(userAddress: "0x…", marketUniqueKey: "0x…", chainId: 747474) {
    healthFactor
    priceVariationToLiquidationPrice
    state { collateral collateralUsd borrowAssets borrowAssetsUsd supplyAssets supplyAssetsUsd }
  }
}
```
Returns a `NOT_FOUND` error when the user has no position. Indexed data can lag a few blocks — read balances on-chain (`position()`) when exactness matters.

#### Reading listings, warnings and APYs

- **`listed`** means Morpho curates the market or vault in its app. Markets and vaults are permissionless, so unlisted ones can be legitimate, but on Katana most unlisted vaults are small personal or test vaults (as of Nov 2026: 5 of 44 v1 vaults and 9 of 21 V2 vaults are listed). Prefer listed ones when recommending, and say when something is unlisted.
- **`warnings`** are Morpho's risk flags with a `level`. Treat any **RED** (e.g. `oracle_unusable`, `oracle_price_derivation`, `bad_debt_unrealized`, `sustained_low_liquidity`, `deposit_disabled`, `short_timelock`) as a reason **not** to deposit or borrow without explicitly warning the user. Mention **YELLOW** ones (e.g. `low_liquidity`, `unrecognized_collateral_asset`, `not_whitelisted`, `bad_debt_realized`).
- **Implausible APYs.** Treat a net APY above 100% as unverified — on Katana these come from tiny personal vaults (some report ~300,000%). Never present it as a real yield.
- `net*` APYs include rewards (and vault fees for vaults); APYs move with utilization.

#### Verifying market params from the API

A market's ID is `keccak256(abi.encode(loanToken, collateralToken, oracle, irm, lltv))`. Before building a transaction with `MarketParams` taken from the API, recompute the ID and check it matches — then the params can be used without trusting the API, and cached forever (they are immutable). If it doesn't match, read `idToMarketParams(id)` on-chain. Idle markets have no collateral: use the zero address for `collateralToken` and `oracle`.

```ts
const id = keccak256(encodeAbiParameters(
  [{ type: "address" }, { type: "address" }, { type: "address" }, { type: "address" }, { type: "uint256" }],
  [loanToken, collateralToken, oracle, irm, BigInt(lltv)]
));
```

### Vault Deposits (ERC-4626)

Standard ERC-4626 interface on vault addresses:
```
deposit(uint256 assets, address receiver) → uint256 shares
redeem(uint256 shares, address receiver, address owner) → uint256 assets
convertToShares(uint256 assets) → uint256
convertToAssets(uint256 shares) → uint256
```

**Requires approval** for the underlying asset to the vault address.

Vault V2 uses the same ERC-4626 `deposit`/`withdraw`/`redeem` interface. Its withdrawable liquidity comes from its adapters, so check `liquidity` (Morpho API) or `maxWithdraw(owner)` before withdrawing.

### Vault Withdrawals (ERC-4626)

Two modes for exiting a vault position:

```
withdraw(uint256 assets, address receiver, address owner) → uint256 shares
```
Specify the exact asset amount to receive. The vault burns the proportional shares.

```
redeem(uint256 shares, address receiver, address owner) → uint256 assets
```
Specify shares to burn. Receive the proportional asset amount.

Use preview and max functions before building the transaction:
```
previewWithdraw(uint256 assets) → uint256 shares   // how many shares will be burned
previewRedeem(uint256 shares) → uint256 assets      // how many assets will be received
maxWithdraw(address owner) → uint256                 // max assets withdrawable by owner
maxRedeem(address owner) → uint256                   // max shares redeemable by owner
```

**Withdrawals may partially fail** if underlying markets have high utilization locking liquidity. Always check `maxWithdraw()` or `maxRedeem()` first, and inspect per-market liquidity via the vault detail reads above.

### Authorization for Bundler3

One-time authorization for leverage loops. Call on Morpho Core:
```
setAuthorization(address authorized, bool newIsAuthorized)
```

Authorize GeneralAdapter1 (`0x916Aa175C36E845db45fF6DDB886AE437d403B61`) to act on behalf of the user.

Check current status:
```
isAuthorized(address authorizer, address authorized) → bool
```

### Leverage Loops

Leverage loops use Bundler3 (`0xA8C5...48C8`) to execute an atomic multicall:

```
multicall(bytes[] memory data)
```

The multicall encodes a sequence of actions via GeneralAdapter1:
1. **Pull collateral** from user into Bundler3
2. **Flash loan** additional loan tokens from Morpho
3. **Swap** loan tokens → collateral on Katana Swap (Sushi V3) (see dex skill for swap mechanics)
4. **Supply collateral** to Morpho market
5. **Borrow** loan tokens to repay the flash loan

**Two prerequisites:**
1. Morpho authorization for GeneralAdapter1 (via `setAuthorization`)
2. ERC20 approval for **Bundler3** (`0xA8C5...48C8`) to pull initial collateral — **not Morpho Core**

**Always simulate before executing.** Use the Katana Swap (Sushi V3) QuoterV2 (see dex skill) to estimate swap slippage across each loop iteration, then calculate the resulting health factor and unwind cost.

#### Loops need a reliable RPC

A loop simulation makes dozens of swap-quote and on-chain calls (several fee tiers per swap, per iteration, plus the unwind). The free public Katana RPC (`https://rpc.katana.network/`, backup `https://rpc.katanarpc.com/`) rate-limits that kind of burst partway through.

- **Tell the user before running loop simulations or builds** that reliable results need their own RPC provider — a dedicated/keyed Katana mainnet endpoint.
- **Quote direct pools first**, and only search multi-hop routes if no direct pool exists. Cap concurrency (≈3 parallel calls on the public RPC) and retry rate-limited calls with backoff (e.g. 1s, 2.5s, 5s). Never retry contract reverts.
- **A rate-limited quote is not "no liquidity".** Only conclude a pair has no route when every quote genuinely *reverts*. If quotes fail with HTTP 429 / "Rate Limit Exceeded" / timeouts, say the RPC is limiting the request and that the user needs their own RPC provider — don't tell them the market can't be looped.

## Workflows

### Explore and Supply (3 Steps)
1. Query Morpho API `markets` for Katana (or scan `CreateMarket` events) — compare supply APY, utilization, liquidity, `listed` and `warnings`
2. Call `approve()` for the loan token → Morpho Core (`0xD50F...8aBc`)
3. Call `supply()` on Morpho Core

### Passive Vault Deposit
1. Query Morpho API `vaults` and `vaultV2s` for Katana — compare net APY, TVL and fees; prefer `listed` vaults with no RED `warnings`
2. Inspect vault allocations (v1) or adapters and caps (V2) via `vaultByAddress` / `vaultV2ByAddress`, or on-chain via `withdrawQueue` + per-market positions
3. Call `approve()` for the underlying asset → vault address
4. Call `deposit()` on the vault (returns preview of shares received)

### Vault Withdrawal
1. Check withdrawal liquidity — read per-market positions and utilization. If `maxWithdraw()` is less than desired amount, some markets have high utilization locking funds.
2. Call `withdraw(assets, receiver, owner)` to withdraw a specific asset amount, or `redeem(shares, receiver, owner)` to burn a specific share amount.

### Vault Risk Analysis
1. Read vault allocations — inspect per-market supply, caps, and utilization to understand concentration risk
2. Check DEX exit liquidity — for each collateral token in the vault's markets, check if there's a Katana Swap pool with sufficient depth (see dex skill). If a collateral token has no pool or thin liquidity, liquidations may be difficult, increasing vault risk.
3. Estimate liquidation slippage — use QuoterV2 (see dex skill) to simulate liquidation-sized trades for each collateral token. High slippage = higher risk for the vault's markets.

### Leverage Loop (5 Steps)
0. Tell the user that loop simulations and builds need many RPC requests, and that reliable results need their own Katana RPC provider (see [Loops need a reliable RPC](#loops-need-a-reliable-rpc))
1. Discover markets — find a suitable market (good LLTV, sufficient liquidity, no RED warnings)
2. **Always simulate first** — use QuoterV2 to estimate swap slippage per iteration, calculate effective leverage, health factor, and unwind cost
3. Call `setAuthorization(GeneralAdapter1, true)` on Morpho Core (one-time)
4. Call `approve()` for collateral token → **Bundler3** (`0xA8C5...48C8`)
5. Call `multicall()` on Bundler3 with the encoded loop actions

### Monitor and Manage Position
1. Call `position()` on Morpho Core — check supply, borrow, collateral (exact, on-chain); get `healthFactor` and `priceVariationToLiquidationPrice` from the Morpho API `marketPosition` query
2. If health factor is dropping: supply more collateral or repay debt
3. Call `withdraw()` on Morpho Core when ready to exit

## Safety Notes

- **Health factor:** Always check after borrowing or looping. Keep above **1.5** for safety margin (1.0 = liquidation threshold). Enforce a 5% safety margin on max leverage: `maxSafeLeverage = 1/(1-LLTV) × 0.95`.
- **LLTV is NOT a safe borrow ratio.** Borrowing at exactly LLTV means instant liquidation risk. Stay well below.
- **Simulate before looping.** Calculate real swap slippage and unwind costs. If unwinding the loop costs extra due to slippage, warn the user.
- **Market liquidity.** Available borrow = totalSupply - totalBorrow. Warn the user if utilization is very high (>90%) — they may have trouble withdrawing later.
- **Leverage loops are complex.** Make sure the user understands: they are taking a leveraged position that amplifies both gains and losses. Liquidation risk increases with leverage.

- **Approval targets differ by operation:**
  - Supply/borrow → approve **Morpho Core** (`0xD50F...8aBc`)
  - Leverage loops → approve **Bundler3** (`0xA8C5...48C8`)
- **Authorization is separate from approval.** `setAuthorization` grants GeneralAdapter1 permission to act in Morpho on the user's behalf. `approve()` grants ERC20 token spending. Both are needed for loops.
- **On-chain discovery is slow and RPC-heavy** — it scans events from genesis and then reads every market/vault. Prefer the Morpho API; cache on-chain results when you must use them.

## Data Sources

The lending skill uses two complementary data sources:

1. **Morpho API** (`https://api.morpho.org/graphql`, always filtered to `chainId: 747474`) — markets, v1 vaults, Vault V2s, positions, USD values, APYs, rewards, listing status and risk warnings. Use it for discovery, analytics and risk metrics.
2. **Katana RPC** (`https://rpc.katana.network/`, backup `https://rpc.katanarpc.com/`) — Direct on-chain reads via `eth_call`. Authoritative for exact balances and transaction building, required on Bokuto testnet, and the fallback when the Morpho API is unavailable. The public endpoints are rate-limited; heavy workloads (loops, full on-chain scans) need the user's own RPC provider.

The Katana Swap (Sushi) DEX contracts (see dex skill) provide additional on-chain data for swap quotes, pool discovery, and liquidity analysis — essential for assessing collateral exit liquidity and liquidation risk.

## Common Mistakes

- **Confusing market IDs with token symbols.** Market-specific functions require a `marketId` (bytes32 hex, exactly 66 characters: `0x` + 64 hex). To find the right market ID, scan `CreateMarket` events and match by loan/collateral token pair and LLTV.
- **Truncated or malformed market IDs.** Market IDs must be exactly 66 characters. A shorter string will fail with a bytes size mismatch error.
- **Wrong approval target.** Supply/withdraw/borrow → approve Morpho Core. Leverage loops → approve Bundler3. Mixing these up will revert.
- **Unfiltered Morpho API queries.** Always pass `chainId_in: [747474]` (or `chainId: 747474`); otherwise results include other networks.
- **Using API params without checking them.** Verify `MarketParams` from the API hash to the market ID before encoding a transaction.
- **Reporting a rate limit as missing liquidity.** RPC 429s and timeouts during quoting mean the RPC is limiting you, not that a pool doesn't exist.

## Cross-References

- **wallet-manager**: `approve()` patterns for Morpho Core or Bundler3 approvals, balance checks
- **dex**: Katana Swap (Sushi V3) provides the swap leg inside leverage loops. Use QuoterV2 to preview swap rates. Use pool reads to check DEX exit liquidity for vault collateral tokens — essential for assessing liquidation risk.
- **merkl**: check reward incentives on Morpho markets/vaults before entering positions
- **analytics**: on-chain price derivation for USD position values, gas cost estimates for complex transactions
