---
name: perps
description: Activate when the user asks about perpetual futures, perps, leveraged trading, order placement, funding rates, or position management on Katana Perps.
license: MIT
metadata:
  author: katana
  version: '1.0.0'
---

# Perps Trader — Katana Perps

Trade perpetual futures on Katana Perps — a non-custodial, cross-margined perpetual futures DEX with a central limit order book. This skill covers market data, position management, order placement, and withdrawals.

## When to Activate
- User asks about perpetual futures, perps, or leveraged trading on Katana
- User wants to check perps market data, prices, funding rates, or liquidations
- User wants to view their perps positions, orders, or fills
- User wants to place or cancel perps orders
- User wants to withdraw from the perps exchange

## Contract Addresses

| Item | Mainnet | Testnet |
|------|---------|---------|
| Exchange Contract | `0x835Ba5b1B202773A94Daaa07168b26B22584637a` | `0xcE3765616b9e354E64530875f492dc4DfddF2118` |

### EIP-712 Domain

| Field | Mainnet | Testnet |
|-------|---------|---------|
| name | `KatanaPerps` | `KatanaPerps` |
| version | `1.0.0` | `1.0.0-sandbox` |
| chainId | `747474` | `737373` |
| verifyingContract | `0x835Ba5b1B202773A94Daaa07168b26B22584637a` | `0xcE3765616b9e354E64530875f492dc4DfddF2118` |

## REST API

Base URLs:
- **Mainnet:** `https://api-perps.katana.network`
- **Testnet:** `https://api-perps-sandbox.katana.network`

### Public Endpoints (no auth needed)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v4/exchange` | GET | Exchange info — fees, volume, open interest, contract addresses, quote token (vbUSDC) |
| `/v4/markets` | GET | Market info — leverage limits, margin requirements, fees, index prices, funding |
| `/v4/tickers` | GET | 24h stats — OHLCV, bid/ask, mark/index prices |
| `/v4/orderbook?market={market}&level={1\|2}&limit={n}` | GET | Order book (L1 = best bid/ask, L2 = full depth) |
| `/v4/candles?market={market}&interval={interval}&limit={n}` | GET | OHLCV candles (intervals: 1m, 5m, 15m, 1h, 4h, 1d) |
| `/v4/trades?market={market}&limit={n}` | GET | Recent public trades |
| `/v4/liquidations?market={market}&limit={n}` | GET | Recent liquidation records |
| `/v4/fundingRates?market={market}&limit={n}` | GET | Historical funding rates (8h payment schedule) |
| `/v4/gasFees` | GET | Withdrawal gas fee estimates per destination chain |

### Authenticated Endpoints (require HMAC signature)

**Authentication:** Requires `PERPS_API_KEY` and `PERPS_API_SECRET`. Sign requests using HMAC-SHA256 of the request payload with the API secret.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v4/wallets?wallet={address}` | GET | Wallet state — equity, collateral, leverage, margin ratio |
| `/v4/positions?wallet={address}` | GET | Open positions — PnL, liquidation price, ADL risk |
| `/v4/orders?wallet={address}&closed={bool}` | GET | Open or historical orders with fills |
| `/v4/fills?wallet={address}&market={market}` | GET | Trade fill history — fees, PnL, maker/taker side |

### Trade Endpoints (require API key + EIP-712 wallet signature)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v4/orders` | POST | Create order — returns EIP-712 typed data for signing |
| `/v4/orders/cancel` | DELETE | Cancel by order IDs, market, or all |
| `/v4/withdrawals` | POST | Withdraw to Katana or cross-chain via Stargate |
| `/v4/wallets/associate` | POST | Associate wallet with API account (first-time setup) |

## On-Chain Interactions

### Depositing to Perps Exchange

The exchange collateral is **vbUSDC** (`0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36`). To fund a trading account:

1. Call `approve(exchangeAddress, amount)` on the vbUSDC contract for the exchange contract
2. Call the exchange deposit function with the approved amount

Get the exact `exchangeContractAddress` and `quoteTokenAddress` from the `/v4/exchange` endpoint.

### Order Signing (EIP-712)

Orders use a **two-step flow:**

1. **Build:** POST to `/v4/orders` with order parameters → returns EIP-712 typed data
2. **Sign:** User signs the typed data with their wallet using the EIP-712 domain above
3. **Submit:** Re-POST with the `walletSignature` field to submit the signed order

The agent **never signs** — it constructs the request and returns the EIP-712 data for the user's wallet to sign.

## Key Concepts

### Collateral & Margin
- **Collateral:** vbUSDC only, cross-margined across all positions
- **Initial Margin Fraction (IMF):** Margin to open a position (determines max leverage)
- **Maintenance Margin Fraction (MMF):** Margin to avoid liquidation
- **Margin Ratio:** totalMaintenanceMarginReq / equity — liquidation when > 1
- Larger positions require higher IMF (incremental margin)

### Order Types
| Type | Description |
|------|-------------|
| `market` | Execute immediately at best price |
| `limit` | Execute at specified price or better |
| `stopLossMarket` | Market order triggered at stop price |
| `stopLossLimit` | Limit order triggered at stop price |
| `takeProfitMarket` | Market order triggered at take profit price |
| `takeProfitLimit` | Limit order triggered at take profit price |

### Time In Force
| Value | Description |
|-------|-------------|
| `gtc` | Good-til-canceled (default, rests on book) |
| `gtx` | Post-only / maker only (canceled if crosses spread) |
| `ioc` | Immediate-or-cancel (fill what you can, cancel rest) |
| `fok` | Fill-or-kill (entire quantity or nothing) |

### Funding Payments
- Every 8 hours: 00:00, 08:00, 16:00 UTC
- Positive rate → longs pay shorts; negative → shorts pay longs
- Incentivizes order book price to converge with index price

### Precision
- All prices and quantities: 8 decimal zero-padded strings (e.g., `"2500.05000000"`)

## Key Workflows

### 1. Check Market Overview
1. GET `/v4/markets` — see all markets, leverage, fees
2. GET `/v4/tickers` — 24h stats, current bid/ask, funding
3. GET `/v4/orderbook?market=ETH-USD&level=2&limit=10` — order book depth

### 2. Analyze a Market
1. GET `/v4/candles?market=ETH-USD&interval=1h&limit=24` — price history
2. GET `/v4/trades?market=ETH-USD&limit=20` — recent trades
3. GET `/v4/fundingRates?market=ETH-USD&limit=10` — funding history
4. GET `/v4/liquidations?market=ETH-USD&limit=10` — recent liquidations

### 3. Monitor Positions
1. GET `/v4/wallets?wallet=0x...` — equity, margin, leverage
2. GET `/v4/positions?wallet=0x...` — all positions with PnL
3. GET `/v4/orders?wallet=0x...&closed=false` — open orders
4. GET `/v4/fills?wallet=0x...&market=ETH-USD` — fill history

### 4. Place an Order
1. POST `/v4/orders` with: wallet, market (`"ETH-USD"`), type (`"limit"`), side (`"buy"`), quantity (`"1.00000000"`), price (`"2400.00000000"`), timeInForce (`"gtc"`)
2. Returns EIP-712 typed data → user signs with wallet
3. Re-POST with `walletSignature` to submit

### 5. Cancel Orders
- DELETE `/v4/orders/cancel` with `orderIds: "uuid1,uuid2"` — cancel specific orders
- DELETE `/v4/orders/cancel` with `market: "ETH-USD"` — cancel all in market
- DELETE `/v4/orders/cancel` with just `wallet` — cancel all orders

### 6. Fund Trading Account
1. GET `/v4/exchange` — get `quoteTokenAddress` (vbUSDC) and `exchangeContractAddress`
2. Call `approve(exchangeContractAddress, amount)` on vbUSDC
3. Call deposit on the exchange contract

### 7. Withdraw Funds
1. GET `/v4/gasFees` — check current gas fees per destination chain
2. POST `/v4/withdrawals` with: wallet, quantity (`"1000.00000000"`), maximumGasFee (`"0.60000000"`), destinationChain (`"arbitrum"`)
3. Returns EIP-712 typed data → user signs → submit

## Common Mistakes

- **Wrong market ID format.** Markets use the `"BASE-QUOTE"` format (e.g., `"ETH-USD"`, `"BTC-USD"`), NOT `"ETH"`, `"ETHUSD"`, or `"ETH/USD"`. Invalid market strings will return empty results or errors.
- **Wrong price/quantity precision.** All prices and quantities must be 8-decimal zero-padded strings (e.g., `"2500.05000000"`, `"1.00000000"`). Passing a plain number like `2500` or a string like `"2500.05"` will fail validation. Always pad to 8 decimal places.
- **Forgetting the two-step order flow.** Creating an order does NOT submit it. It returns EIP-712 typed data that the user must sign with their wallet. The signed result must then be re-submitted with the `walletSignature` parameter. Telling the user "your order is placed" after the first call is incorrect.
- **Not checking gas fees before withdrawal.** Withdrawals require a `maximumGasFee` parameter. Always query `/v4/gasFees` first to get current estimates per destination chain. Using a stale or too-low gas fee will cause the withdrawal to fail.
- **Confusing wallet address with API account.** Authenticated endpoints need both the wallet `0x` address AND valid API credentials. First-time users must call the associate endpoint before any authenticated reads or trades will work.

## Safety Notes
- Trade operations return EIP-712 typed data — they never sign or hold private keys
- Always check margin ratio before placing leveraged orders
- Check gas fees before withdrawals to set `maximumGasFee` accurately
- Keep margin ratio well below 1 to avoid liquidation
- Stop loss orders are not guaranteed to fill at the stop price
- Funding payments can accumulate — check `totalFunding` on positions

## Cross-References

- **wallet-manager**: `approve()` pattern for exchange deposit approvals, balance checks for vbUSDC
- **analytics**: on-chain spot price comparison with perps mark prices, contract reference for exchange addresses
