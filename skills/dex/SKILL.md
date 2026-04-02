---
name: dex
description: Activate when the user wants to swap tokens, get trade quotes, check pool liquidity, or provide liquidity on SushiSwap on Katana Network.
license: MIT
metadata:
  author: katana
  version: '1.0.0'
---

# DEX — SushiSwap on Katana

Token swaps, trade quoting, pool analysis, and liquidity provision on SushiSwap (V3 + V2) on Katana Network.

## SushiSwap Overview

SushiSwap on Katana offers both concentrated liquidity (V3) and full-range (V2) pools. V3 is generally preferred for better capital efficiency and tighter spreads.

### V3 Fee Tiers

| Fee | Bps | Tick Spacing | Typical Use |
|-----|-----|-------------|-------------|
| 100 | 0.01% | 1 | Stablecoin pairs |
| 500 | 0.05% | 10 | Correlated assets |
| 3000 | 0.3% | 60 | Most pairs (default) |
| 10000 | 1% | 200 | Exotic/volatile pairs |

### Contract Addresses (Mainnet)

| Contract | Address |
|----------|---------|
| V3 Factory | `0x203e8740894c8955cB8950759876d7E7E45E04c1` |
| V3 SwapRouter | `0x4e1d81A3E627b9294532e990109e4c21d217376C` |
| V3 QuoterV2 | `0x92dea23ED1C683940fF1a2f8fE23FE98C5d3041c` |
| V3 PositionManager | `0x2659C6085D26144117D904C46B48B6d180393d27` |
| V3 TickLens | `0x35DC3E13469E980c37b6F288BBb9822B1f9bD435` |
| V2 Factory | `0x72D111b4d6f31B38919ae39779f570b747d6Acd9` |
| V2 Router | `0x69cC349932ae18ED406eeB917d79b9b3033fB68E` |

## Contracts & Functions

### Quoting a Swap

Use the V3 QuoterV2 to get quotes without executing. Call `quoteExactInputSingle` with a static call (`eth_call`) — it reverts with the quote data.

```
quoteExactInputSingle((
  address tokenIn,
  address tokenOut,
  uint256 amountIn,
  uint24 fee,
  uint160 sqrtPriceLimitX96  // pass 0 for no limit
)) → (uint256 amountOut, uint160 sqrtPriceX96After, uint32 initializedTicksCrossed, uint256 gasEstimate)
```

**To find the best route:** Call `quoteExactInputSingle` across all four fee tiers (100, 500, 3000, 10000) and pick the one with the highest `amountOut`. Also check V2 via `getAmountsOut(uint256 amountIn, address[] path)` on the V2 Router. **Always quote before swapping.**

### Executing a Swap (V3)

Call `exactInputSingle` on SwapRouter (`0x4e1d81A3E627b9294532e990109e4c21d217376C`):

```
exactInputSingle((
  address tokenIn,
  address tokenOut,
  uint24 fee,
  address recipient,
  uint256 amountIn,
  uint256 amountOutMinimum,
  uint160 sqrtPriceLimitX96  // pass 0 for no limit
)) → uint256 amountOut
```

Wrap in `multicall(uint256 deadline, bytes[] data)` to enforce a deadline.

**Slippage:** Calculate `amountOutMinimum = expectedOutput × (10000 - slippageBps) / 10000`.

**Requires approval first.** The user must call `approve()` on `tokenIn` for the V3 SwapRouter (see wallet-manager skill).

### Executing a Swap (V2)

Call `swapExactTokensForTokens` on V2 Router (`0x69cC349932ae18ED406eeB917d79b9b3033fB68E`):

```
swapExactTokensForTokens(
  uint256 amountIn,
  uint256 amountOutMin,
  address[] path,
  address to,
  uint256 deadline
) → uint256[] amounts
```

**Requires approval** for V2 Router.

### Reading Pool State (V3)

Get V3 pool addresses from the Factory: `getPool(address tokenA, address tokenB, uint24 fee) → address pool`.

On the pool contract:
- `slot0() → (uint160 sqrtPriceX96, int24 tick, ...)` — current price and tick
- `liquidity() → uint128` — active liquidity at current tick
- `tickBitmap(int16 wordPosition) → uint256` — bitmap of initialized ticks (for analyzing liquidity distribution)

Use the TickLens (`0x35DC3E13469E980c37b6F288BBb9822B1f9bD435`) for efficient tick range queries:
```
getPopulatedTicksInWord(address pool, int16 tickBitmapIndex) → (int24 tick, int128 liquidityNet, uint128 liquidityGross)[]
```

### Adding V3 Concentrated Liquidity

Call `mint` on PositionManager (`0x2659C6085D26144117D904C46B48B6d180393d27`):

```
mint((
  address token0,
  address token1,
  uint24 fee,
  int24 tickLower,
  int24 tickUpper,
  uint256 amount0Desired,
  uint256 amount1Desired,
  uint256 amount0Min,
  uint256 amount1Min,
  address recipient,
  uint256 deadline
)) → (uint256 tokenId, uint128 liquidity, uint256 amount0, uint256 amount1)
```

**Tick ranges must be divisible by tickSpacing** (1 for 100 fee, 10 for 500, 60 for 3000, 200 for 10000). Invalid ticks will revert.

**Requires approval of BOTH tokens** for the PositionManager. Note: `token0 < token1` by address — the contracts enforce this ordering.

### Adding V2 Liquidity

Call `addLiquidity` on V2 Router (`0x69cC349932ae18ED406eeB917d79b9b3033fB68E`):

```
addLiquidity(
  address tokenA,
  address tokenB,
  uint256 amountADesired,
  uint256 amountBDesired,
  uint256 amountAMin,
  uint256 amountBMin,
  address to,
  uint256 deadline
) → (uint256 amountA, uint256 amountB, uint256 liquidity)
```

Simpler than V3 — no tick management. **Requires approval of BOTH tokens** for the V2 Router.

## Workflows

### Token Swap (3 Steps)
1. **Quote:** Call `quoteExactInputSingle` on QuoterV2 across all fee tiers — find best route and expected output
2. **Approve:** Call `approve()` on `tokenIn` for the router
   - V3: spender = `0x4e1d81A3E627b9294532e990109e4c21d217376C`
   - V2: spender = `0x69cC349932ae18ED406eeB917d79b9b3033fB68E`
3. **Swap:** Call `exactInputSingle` (V3) or `swapExactTokensForTokens` (V2) using the best fee tier from the quote

Always show the user: expected output amount, minimum output (after slippage), and which fee tier/version is being used.

### Add V3 Concentrated Liquidity (4 Steps)
1. **Analyze:** Read pool `slot0()` and `liquidity()` — check current tick, price, and liquidity concentration
2. **Choose range:** Help the user pick `tickLower` and `tickUpper` based on current tick and desired price range. Tighter ranges earn more fees but risk going out of range.
   - Ticks must be divisible by tickSpacing for the chosen fee tier
   - Current tick from `slot0()` is the center point
3. **Approve x2:** Call `approve()` for both tokens → PositionManager (`0x2659C6085D26144117D904C46B48B6d180393d27`)
4. **Mint:** Call `mint()` on PositionManager

### Add V2 Liquidity (3 Steps)
1. **Check price:** Call QuoterV2 or read pool state to determine current price ratio
2. **Approve x2:** Call `approve()` for both tokens → V2 Router (`0x69cC349932ae18ED406eeB917d79b9b3033fB68E`)
3. **Add:** Call `addLiquidity` on V2 Router

## Slippage Guidance

| Pair type | Typical slippage |
|-----------|--------------------|
| Stablecoin-stablecoin (USDC/USDT) | 10-20 bps (0.1-0.2%) |
| Major pairs (WETH/USDC) | 50 bps (0.5%) — the default |
| Volatile/thin pairs | 100-200 bps (1-2%) |

## Safety Notes

- **Always quote before swapping.** The quote tells you which fee tier is best and what output to expect.
- **Approval is required** before every swap or liquidity add. Use the correct router address as the spender.
- **Check balances first** (see wallet-manager skill) to verify the user has sufficient tokens.
- **V3 token ordering:** Token0 < token1 by address. The contracts enforce sorting, but tick ranges are relative to the token0/token1 order, not the user's input order.
- **V3 tick validation:** tickLower and tickUpper must be divisible by tickSpacing. The contract will revert if they're not.

## Common Mistakes

- **Using a fee tier without quoting first.** Always call `quoteExactInputSingle` across all fee tiers before building a swap. Hardcoding `3000` may route through a worse pool.
- **Invalid tick ranges for V3 LP.** `tickLower` and `tickUpper` must be divisible by the fee tier's tickSpacing (1 for 0.01%, 10 for 0.05%, 60 for 0.3%, 200 for 1%). Non-divisible ticks will revert on-chain. Read `slot0()` to get the current tick and calculate valid boundaries.
- **Approving the wrong router.** V3 swaps need approval for SwapRouter (`0x4e1d...376C`), V3 LP needs PositionManager (`0x2659...3d27`), V2 uses V2 Router (`0x69cC...b68E`). Mixing these up means the tx will revert with an allowance error.
- **Forgetting to approve BOTH tokens for LP.** Adding liquidity (V3 or V2) requires approving both `tokenA` and `tokenB`. Missing one will revert.

## Data Sources

SushiSwap tools interact directly with on-chain contracts via Katana RPC:

- **V3 QuoterV2** — simulates swaps across all fee tiers for accurate quotes with price impact
- **V3 Factory** — discovers pools for token pairs, checks pool existence and liquidity
- **V3 Pool contracts** — reads current price, tick, reserves, and tick concentration maps for LP analysis
- **V2 Factory / V2 Router** — full-range pool discovery and routing

These on-chain reads provide real-time DEX data. Use pool reads to check exit liquidity for any token pair (e.g., assessing whether a collateral token can be liquidated efficiently in Morpho markets). Use QuoterV2 to get real slippage estimates for any trade size.

## Cross-References

- **wallet-manager**: `approve()` patterns for router approvals, balance checks before swaps
- **analytics**: on-chain price derivation to show USD values, gas cost estimates
- **merkl**: check if a pool has reward incentives before adding LP
- **lending**: SushiSwap provides the swap leg inside Morpho leverage loops
