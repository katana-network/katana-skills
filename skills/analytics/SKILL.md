---
name: analytics
description: Activate when the user asks about token prices, gas costs, transaction status, or general Katana Network chain data.
license: MIT
metadata:
  author: katana
  version: '1.0.0'
---

# Analytics — Katana Network

Read-only on-chain queries for market data, gas estimation, transaction debugging, and contract reference on Katana.

## Katana Chain Facts

- Mainnet chain ID: **747474** / Testnet (Bokuto): **737373**
- Gas token: ETH
- 1-second block times, EIP-1559 gas pricing
- RPC: `https://rpc.katana.network/` (mainnet) / `https://rpc-bokuto.katanarpc.com` (testnet)
- Explorer: katanascan.com (mainnet), bokuto.katanascan.com (testnet)

## Known Tokens

| Symbol | Address | Decimals |
|--------|---------|----------|
| KAT | `0x7F1f4b4b29f5058fA32CC7a97141b8D7e5ABDC2d` | 18 |
| WETH | `0xEE7D8BCFb72bC1880D0Cf19822eB0A2e6577aB62` | 18 |
| WBTC | `0x0913DA6Da4b42f538B445599b46Bb4622342Cf52` | 8 |
| USDC | `0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36` | 6 |
| USDT | `0x2DCa96907fde857dd3D816880A0df407eeB2D2F2` | 6 |
| USDS | `0x62D6A123E8D19d06d68cf0d2294F9A3A0362c6b3` | 18 |
| AUSD | `0x00000000eFE302BEAA2b3e6e1b18d08D69a9012a` | 18 |
| LBTC | `0xecAc9C5F704e954931349Da37F60E39f515c11c1` | 8 |
| weETH | `0x9893989433e7a383Cb313953e4c2365107dc19a7` | 18 |
| wstETH | `0x7Fb4D0f51544F24F385a421Db6e7D4fC71Ad8e5C` | 18 |
| MORPHO | `0x1e5eFCA3D0dB2c6d5C67a4491845c43253eB9e4e` | 18 |
| SUSHI | `0x17BFF452dae47e07CeA877Ff0E1aba17eB62b0aB` | 18 |
| vKAT | `0x106F7D67Ea25Cb9eFf5064CF604ebf6259Ff296d` | — |
| avKAT | `0x7231dbaCdFc968E07656D12389AB20De82FbfCeB` | 18 |

Decimals: USDC/USDT = 6, WBTC/LBTC = 8, all others = 18. Stablecoins (USDC, USDT, USDS, AUSD) can be assumed $1 for quick estimates.

## On-Chain Queries

### Token Prices

Derive USD spot prices from Sushi V3 pool state. For each token:

1. Get pool address from V3 Factory (`0x203e8740894c8955cB8950759876d7E7E45E04c1`):
   ```
   getPool(address tokenA, address tokenB, uint24 fee) → address pool
   ```
   Check across fee tiers (100, 500, 3000, 10000) and use the pool with most liquidity.

2. Read `slot0()` on the pool contract:
   ```
   slot0() → (uint160 sqrtPriceX96, int24 tick, ...)
   ```

3. Calculate price:
   ```
   price = (sqrtPriceX96 / 2^96)^2
   ```
   Adjust for decimal differences between token0 and token1: `price × 10^(decimals0 - decimals1)`.

4. **Routing:** Tokens without a direct stablecoin pool should route through WETH (e.g., KAT → WETH → USDC).

**Note:** These are on-chain spot prices, not aggregated oracle prices. Thin pools may show significant deviation from CEX prices. If a token has no Sushi V3 pool, it returns no price — this does not mean the token is worthless.

### Gas Prices

Use JSON-RPC calls to estimate current gas:

- `eth_gasPrice` — returns current gas price in wei
- `eth_feeHistory(blockCount, "latest", [25, 50, 75])` — EIP-1559 base fee + priority fee history

**Typical gas estimates for common operations:**

| Operation | Gas Units |
|-----------|-----------|
| ETH transfer | ~21,000 |
| ERC-20 transfer | ~65,000 |
| ERC-20 approve | ~46,000 |
| V3 swap | ~185,000 |
| Morpho supply | ~150,000 |
| Morpho borrow | ~200,000 |
| V3 LP mint | ~500,000 |
| Bundler3 loop (3 iterations) | ~1,500,000 |

**Cost formula:** `gasCostETH = gasUnits × maxFeePerGas / 10^18`

### Transaction Lookup

Use `eth_getTransactionReceipt(txHash)` to check transaction status:

- `status`: `0x1` = success, `0x0` = reverted
- `blockNumber`, `gasUsed`, `effectiveGasPrice`
- `logs` — decoded events

Also `eth_getTransactionByHash(txHash)` for: `from`, `to`, `value`, `input` data.

**Explorer URL:** `https://katanascan.com/tx/{txHash}`

On Katana with 1-second blocks, pending state is very brief.

## Contract Reference

### Sushi DEX Contracts (Mainnet)

| Contract | Address |
|----------|---------|
| V3 Factory | `0x203e8740894c8955cB8950759876d7E7E45E04c1` |
| V3 SwapRouter | `0x4e1d81A3E627b9294532e990109e4c21d217376C` |
| V3 QuoterV2 | `0x92dea23ED1C683940fF1a2f8fE23FE98C5d3041c` |
| V3 PositionManager | `0x2659C6085D26144117D904C46B48B6d180393d27` |
| V3 TickLens | `0x35DC3E13469E980c37b6F288BBb9822B1f9bD435` |
| V2 Factory | `0x72D111b4d6f31B38919ae39779f570b747d6Acd9` |
| V2 Router | `0x69cC349932ae18ED406eeB917d79b9b3033fB68E` |
| RouteProcessor7 | `0x3Ced11c610556e5292fBC2e75D68c3899098C14C` |

### Morpho Contracts (Mainnet)

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

### Merkl (Rewards)

| Contract | Address |
|----------|---------|
| Distributor | `0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae` |

### KAT Token Ecosystem

| Contract | Address |
|----------|---------|
| KAT Token | `0x7F1f4b4b29f5058fA32CC7a97141b8D7e5ABDC2d` |
| VotingEscrow | `0x4d6fC15Ca6258b168225D283262743C623c13Ead` |
| avKAT Vault | `0x7231dbaCdFc968E07656D12389AB20De82FbfCeB` |
| GaugeVoter | `0x5e755A3C5dc81A79DE7a7cEF192FFA60964c9352` |

### Bridge

| Contract | Address |
|----------|---------|
| Unified Bridge | `0x2a3DD3EB832aF982ec71669E178424b10Dca2EDe` |
| Bridge & Call | `0x64B20Eb25AEd030FD510EF93B9135278B152f6a6` |

### Infrastructure

| Contract | Address |
|----------|---------|
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |
| EntryPoint (ERC-4337) | `0x4337084D9E255Ff0702461CF8895CE9E3b5Ff108` |

### Price Feeds

| Provider | Address / Link |
|----------|----------------|
| Chainlink VerifierProxy | `0x2a644E5AC685112A7Eff0c4d73CD0260546D366F` |
| API3 Market | `https://market.api3.org/katana` |

### Perps Exchange

| Item | Mainnet | Testnet |
|------|---------|---------|
| Exchange Contract | `0x835Ba5b1B202773A94Daaa07168b26B22584637a` | `0xcE3765616b9e354E64530875f492dc4DfddF2118` |
| REST API | `https://api-perps.katana.network` | `https://api-perps-sandbox.katana.network` |
| WebSocket | `wss://websocket-perps.katana.network/v1` | `wss://websocket-perps-sandbox.katana.network/v1` |

## Common Uses

| User request | Action |
|---|---|
| "What's the price of KAT?" | Read Sushi V3 pool `slot0()` for KAT/WETH then WETH/USDC |
| "How much will gas cost?" | Call `eth_gasPrice` + multiply by gas estimate from table above |
| "Did my transaction go through?" | Call `eth_getTransactionReceipt(txHash)` |
| Show USD values for a portfolio | Derive prices from V3 pools then multiply by token balances |
| "What contracts are on Katana?" | See Contract Reference tables above |

## Common Mistakes

- **Truncated transaction hash.** Transaction hashes must be exactly 66 characters (`0x` + 64 hex digits). A shorter or malformed hash will fail.
- **Assuming "No liquidity found" means a token is worthless.** Price derivation from V3 pools only works if a pool exists. No pool ≠ zero value. Note this to the user — external price sources may be needed.
- **Wrong decimal adjustment for prices.** When computing price from `sqrtPriceX96`, you must adjust for the decimal difference between token0 and token1. Getting this wrong produces prices off by orders of magnitude.

## Cross-References

- All other skills can use the price derivation approach to add USD context to balances, positions, and rewards.
- Use gas estimates before constructing expensive transactions (leverage loops, multi-token operations).
- Use transaction lookup after the user sends any transaction to verify success.
