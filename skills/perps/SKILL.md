---
name: perps
description: Activate when the user asks about perpetual futures, perps, leveraged trading, order placement, funding rates, or position management on Katana Perps.
license: MIT
metadata:
  author: katana
  version: '2.0.0'
---

# Perps Trader — Katana Perps

Trade perpetual futures on Katana Perps — a non-custodial, cross-margined perpetual futures DEX with a central limit order book. This skill covers market data, position management, order placement, deposits and withdrawals.

## When to Activate
- User asks about perpetual futures, perps, or leveraged trading on Katana
- User wants to check perps market data, prices, funding rates, or liquidations
- User wants to view their perps positions, orders, or fills
- User wants to place or cancel perps orders
- User wants to deposit to or withdraw from the perps exchange

## Contract Addresses

Katana Perps runs on **Exchange v2**. The source of truth is `GET /v1/exchange` → `exchangeContractAddress` (the official SDK reads it from there too). Always read it live before signing or depositing; the table below is for reference.

| Item | Mainnet | Testnet (Bokuto) |
|------|---------|---------|
| Exchange Contract (v2) | `0x62230CeA619F734cc215bB8074bbF07bE4Eb633e` | `0x92d3072dDe1aD3e9B7895500F504aA5e664E71d3` |
| Quote token (vbUSDC, 6 decimals) | `0x203A662b0BD271A6ed5a60EdFbd04bFce608FD36` | from `/v1/exchange` → `quoteTokenAddress` |

> The previous v1 exchange (`0x835Ba5b1B202773A94Daaa07168b26B22584637a` mainnet, `0xcE3765616b9e354E64530875f492dc4DfddF2118` testnet) is still deployed but **is no longer the venue**. Never sign against it, approve it, or deposit to it.

### EIP-712 Domain

| Field | Mainnet | Testnet |
|-------|---------|---------|
| name | `KatanaPerps` | `KatanaPerps` |
| version | `2.0.0` | `2.0.0-sandbox` |
| chainId | `747474` | `737373` |
| verifyingContract | live `exchangeContractAddress` (currently `0x62230CeA619F734cc215bB8074bbF07bE4Eb633e`) | live `exchangeContractAddress` (currently `0x92d3072dDe1aD3e9B7895500F504aA5e664E71d3`) |

These match `@katanaperps/katana-perps-sdk` 2.x (`EIP_712_DOMAIN_NAME`, `EIP_712_DOMAIN_VERSION`, `EIP_712_DOMAIN_VERSION_SANDBOX`). A signature built with the old `1.0.0` version or the v1 contract is rejected.

## REST API

Base URLs (all endpoints are under **`/v1`**):
- **Mainnet:** `https://api-perps.katana.network/v1`
- **Testnet:** `https://api-perps-sandbox.katana.network/v1`

API reference: https://api-docs-v1-perps.katana.network/

### Public Endpoints (no auth needed)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/exchange` | GET | Exchange info — fees, volume, open interest, `exchangeContractAddress`, `quoteTokenAddress` (vbUSDC), bridge adapters |
| `/v1/markets` | GET | Market info — leverage limits, margin requirements, fees, index prices, funding |
| `/v1/tickers` | GET | 24h stats — OHLCV, bid/ask, mark/index prices |
| `/v1/orderbook?market={market}&level={1\|2}&limit={n}` | GET | Order book (L1 = best bid/ask, L2 = full depth) |
| `/v1/candles?market={market}&interval={interval}&limit={n}` | GET | OHLCV candles (intervals: 1m, 5m, 15m, 1h, 4h, 1d) |
| `/v1/trades?market={market}&limit={n}` | GET | Recent public trades |
| `/v1/liquidations?market={market}&limit={n}` | GET | Recent liquidation records |
| `/v1/fundingRates?market={market}&limit={n}` | GET | Historical funding rates (8h payment schedule) |
| `/v1/gasFees` | GET | Withdrawal gas fee estimates per destination chain |
| `/v1/time` | GET | Server time — use it to keep nonces in sync |

### Authentication

- **API key + HMAC** (user-data and trade endpoints): send `KP-API-KEY` and `KP-HMAC-SIGNATURE` headers. The signature is HMAC-SHA256 with the API secret over the **query string** (GET) or the **JSON body** (POST/DELETE).
- **Wallet signature** (trade endpoints): an EIP-712 signature by the user's wallet over the request, using the domain above, sent as `walletSignature`.
- **Nonces:** UUID v1, within 60s old / 5s future of server time (`/v1/time`). In EIP-712 values the nonce is a `uint128`: `BigInt.asUintN(128, BigInt("0x" + uuid.replace(/-/g, "")))`.

### User-Data Endpoints (API key + HMAC)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/wallets?wallet={address}` | GET | Wallet state — equity, collateral, leverage, margin ratio |
| `/v1/positions?wallet={address}` | GET | Open positions — PnL, liquidation price, ADL risk |
| `/v1/orders?wallet={address}&closed={bool}` | GET | Open or historical orders with fills |
| `/v1/fills?wallet={address}&market={market}` | GET | Fill history — fees, PnL, maker/taker side, and `builderFee` on fills tagged with a builder code |

### Trade Endpoints (API key + HMAC + EIP-712 wallet signature)

| Endpoint | Method | EIP-712 type | Description |
|----------|--------|--------------|-------------|
| `/v1/wallets` | POST | `WalletAssociation` | Associate a wallet with the API account (first-time setup) |
| `/v1/orders` | POST | `Order` | Create an order |
| `/v1/orders` | DELETE | `OrderCancellationByOrderId` / `ByMarketSymbol` / `ByWallet` | Cancel by order IDs, by market, or all |
| `/v1/withdrawals` | POST | `Withdrawal` | Withdraw to Katana or cross-chain |

## On-Chain Interactions

### Depositing to the Perps Exchange

Collateral is **vbUSDC** (6 decimals). Exchange v2's deposit function is:

```
deposit(uint256 quantityInAssetUnits, address depositorWallet)
```

`quantityInAssetUnits` is the vbUSDC amount in its **native 6 decimals** (1,000 vbUSDC = `1000000000`). This is not the 8-decimal string format the REST API uses.

1. `GET /v1/exchange` → read `exchangeContractAddress` and `quoteTokenAddress`
2. `approve(exchangeContractAddress, amount)` on vbUSDC
3. `exchangeContractAddress.deposit(amount, wallet)`

The wallet must be associated (`POST /v1/wallets`) before its first deposit.

### Signing Requests (EIP-712)

The REST API does **not** hand back typed data to sign. The client builds the EIP-712 message, the user's wallet signs it, and the request is submitted **once** with `walletSignature`:

1. **Build** the typed data for the request (struct definitions below) with the EIP-712 domain above
2. **Sign** with the user's wallet
3. **Submit** the request parameters + `walletSignature` (with the HMAC headers)

The agent **never signs** — it builds the request and typed data and hands them to the user's wallet. The official SDK (`@katanaperps/katana-perps-sdk`) does all of this for you.

**`Order` struct** (field order matters):
```
nonce uint128, wallet address, marketSymbol string, orderType uint8, orderSide uint8,
quantity string, limitPrice string, triggerPrice string, triggerType uint8,
callbackRate string, conditionalOrderId uint128, isReduceOnly bool, timeInForce uint8,
selfTradePrevention uint8, isLiquidationAcquisitionOnly bool, delegatedPublicKey address,
clientOrderId string
```
Unused price fields are `"0.00000000"`; unused ids/addresses are `0` / the zero address.

**Other structs:**
- `WalletAssociation`: `nonce uint128, wallet address`
- `OrderCancellationByWallet`: `nonce, wallet, delegatedKey address`
- `OrderCancellationByOrderId`: `nonce, wallet, delegatedKey, orderIds string[]`
- `OrderCancellationByMarketSymbol`: `nonce, wallet, delegatedKey, marketSymbol string`
- `Withdrawal`: `nonce uint128, wallet address, quantity string, maximumGasFee string, bridgeAdapter address, bridgeAdapterPayload bytes`

**Enum values:**

| Field | Values |
|-------|--------|
| orderType | 0=market, 1=limit, 2=stopLossMarket, 3=stopLossLimit, 4=takeProfitMarket, 5=takeProfitLimit |
| orderSide | 0=buy, 1=sell |
| triggerType | 0=none, 1=last, 2=index |
| timeInForce | 0=gtc, 1=gtx, 2=ioc, 3=fok |
| selfTradePrevention | 0=dc, 1=co, 2=cn, 3=cb |

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
- All REST prices and quantities: 8-decimal zero-padded strings (e.g., `"2500.05000000"`)
- On-chain deposits use vbUSDC's native 6 decimals (see Depositing)

## Key Workflows

### 1. Check Market Overview
1. GET `/v1/markets` — see all markets, leverage, fees
2. GET `/v1/tickers` — 24h stats, current bid/ask, funding
3. GET `/v1/orderbook?market=ETH-USD&level=2&limit=10` — order book depth

### 2. Analyze a Market
1. GET `/v1/candles?market=ETH-USD&interval=1h&limit=24` — price history
2. GET `/v1/trades?market=ETH-USD&limit=20` — recent trades
3. GET `/v1/fundingRates?market=ETH-USD&limit=10` — funding history
4. GET `/v1/liquidations?market=ETH-USD&limit=10` — recent liquidations

### 3. Monitor Positions
1. GET `/v1/wallets?wallet=0x...` — equity, margin, leverage
2. GET `/v1/positions?wallet=0x...` — all positions with PnL
3. GET `/v1/orders?wallet=0x...&closed=false` — open orders
4. GET `/v1/fills?wallet=0x...&market=ETH-USD` — fill history

### 4. Place an Order
1. Build the `Order` typed data: wallet, market (`"ETH-USD"`), type (`limit` → 1), side (`buy` → 0), quantity (`"1.00000000"`), limitPrice (`"2400.00000000"`), timeInForce (`gtc` → 0), fresh nonce
2. User signs it with their wallet
3. POST `/v1/orders` with the same parameters + `walletSignature`

To earn a fee share on the orders your product routes, attach a **builder code**
by prefixing the `clientOrderId` field (see **Building a Perps Product — Builder
Codes** below). The prefix must be part of the payload that gets signed, so it is
identical in both the EIP-712 typed data and the submitted request.

### 5. Cancel Orders
Sign the matching cancellation type, then DELETE `/v1/orders` with:
- `orderIds: ["uuid1", "uuid2"]` — cancel specific orders (`OrderCancellationByOrderId`)
- `market: "ETH-USD"` — cancel all in a market (`OrderCancellationByMarketSymbol`)
- just `wallet` — cancel all orders (`OrderCancellationByWallet`)

### 6. Fund Trading Account
1. GET `/v1/exchange` — get `exchangeContractAddress` and `quoteTokenAddress` (vbUSDC)
2. `approve(exchangeContractAddress, amount)` on vbUSDC
3. `deposit(amountInVbUsdcUnits, wallet)` on the exchange contract — 6 decimals

### 7. Withdraw Funds
1. GET `/v1/gasFees` — check current gas fees per destination chain
2. Build and sign the `Withdrawal` typed data: wallet, quantity (`"1000.00000000"`), maximumGasFee (`"0.60000000"`), bridge adapter for the destination (from `/v1/exchange` → `bridgeAdapters`)
3. POST `/v1/withdrawals` with the parameters + `walletSignature`

## Building a Perps Product — Builder Codes

Builder codes let a product built on Katana Perps earn a configurable share of
trading fees on the order flow it brings, on top of the venue's standard maker
and taker fees. If someone is building a perps product (frontend, bot, or
backend) that places orders, **proactively raise builder codes** — don't wait to
be asked.

### When to bring it up
Ask a single question before writing any order-placement code, in **either** of
these cases:
- The user asks to build, scaffold, or design a perps product/app/bot.
- The user is wiring up order creation in any capacity — the `/v1/orders`
  endpoint directly, or the **Katana Perps SDK (`@katanaperps/katana-perps-sdk`)**
  `createOrder` call — in a bot, script, or backend.

> "Do you want to monetize this with a **builder code**? It lets your product
> earn a fee share on every order it routes through Katana Perps."

### If they say yes
1. **Ask for their fee rates**, and recommend the defaults:
   - **Recommended: 0.01% maker / 0.02% taker.**
   - Allowed range: **min 0%**, **max 5% total including the exchange's base
     fees** (the cap is enforced in the contract). The builder fee is *added on
     top of* the venue's standard maker/taker fees.
2. **Attach the code to every order** by prefixing the `clientOrderId` field. A
   builder code is `"B:"` + 8 alphanumeric chars (10 total). The remaining space
   (up to 30 bytes) is still free for the caller's own id, keeping the total
   within the 40-byte `clientOrderId` limit:

   ```
   clientOrderId = "<10-char builder code>" + "<up to 30 bytes of client id>"
   e.g.  "B:AbC12xY9" + orderRef.slice(0, 30)
   ```

   This works identically whether you POST to `/v1/orders` or call the SDK's
   `createOrder` — there is no separate "builderCode" field; the code lives in
   `clientOrderId`. **The prefix must be included in the payload that is signed**,
   so build the final `clientOrderId` before generating the EIP-712 typed data
   and use the same value in the submitted request.
3. **Fees are configured off-chain, not in the request.** The maker/taker rates
   attached to a code are set on the web builder rewards page. The `clientOrderId`
   prefix only *tags* the flow; the rates on that code determine what's earned.
4. **Tracking earnings:** fills routed with a builder code carry a `builderFee`
   field (`GET /v1/fills`; `bf` on WebSocket fill events). SDK 2.1.0 types it.

### The process to actually get a code
Walk the user through this when they want to proceed:
1. Connect their wallet on the web client (https://perps.katana.network).
2. **Contact the Katana team to request a code** — open a Discord support
   ticket or email **kpsupport@katana.network** with their wallet address and a
   short description of the integration. (There is no self-serve/API way to mint
   a code today.)
3. Receive the builder code (`B:` + 8 chars).
4. Configure maker/taker rates and later claim earnings on the builder rewards
   page: **https://perps.katana.network/rewards/builder**.
5. Prefix it onto `clientOrderId` on every order the product places, and ship.

### Showing fees correctly in the product's UI
When the product displays estimated fees or PnL, use the market's own rates from
`/v1/markets` (`makerFeeRate`, `takerFeeRate`) — these reflect the effective rate
the trader pays. Apply them against notional (`quantity × price`):
- **Order preview:** limit/post-only orders pay the **maker** rate, market/taker
  orders pay the **taker** rate → `estFee = notional × feeRate`.
- **Open-position PnL:** net an estimated *close* fee out of displayed PnL so it
  reflects what the trader would actually realize:
  `pnl = unrealizedPnL + realizedPnL − (makerFeeRate × |quantity| × markPrice)`.

### If they say no
Don't prefix `clientOrderId` with a builder code — orders behave exactly as
before, and the product routes flow to the shared books without earning a fee
share.

## Katana Perps SDK

For bots, scripts and backends, recommend the official SDK — it handles HMAC
auth, EIP-712 signing, nonces and WebSocket auth:

```
npm install @katanaperps/katana-perps-sdk
```

- **Stable: `2.1.0`** (the `@beta` tag is no longer needed). 2.x signs with domain version `2.0.0` against Exchange v2.
- Clients: `RestPublicClient`, `RestAuthenticatedClient`, `WebSocketClient`, `OrderBookRealTimeClient`
- Docs: https://sdk-js-docs-v1-perps.katana.network/

If the user has hand-written signing code from before Exchange v2, check that it uses domain version `2.0.0` and the live `exchangeContractAddress`.

## Common Mistakes

- **Using old API paths.** All endpoints are under `/v1` (e.g. `/v1/orders`, `/v1/exchange`). There is no `/v4`, no `/orders/cancel` (cancel is `DELETE /v1/orders`) and no `/wallets/associate` (associate is `POST /v1/wallets`).
- **Signing against Exchange v1.** Domain version must be `2.0.0` (`2.0.0-sandbox` on testnet) and `verifyingContract` the live `exchangeContractAddress`. The old `1.0.0` domain or `0x835Ba5b1…637a` contract produces signatures the API rejects — and never approve or deposit to the old contract.
- **Depositing in 8-decimal "pips".** On-chain `deposit(uint256 quantityInAssetUnits, address depositorWallet)` takes vbUSDC's native 6 decimals. Encoding a `uint64` pips amount calls a function that doesn't exist and reverts.
- **Wrong market ID format.** Markets use the `"BASE-QUOTE"` format (e.g., `"ETH-USD"`, `"BTC-USD"`), NOT `"ETH"`, `"ETHUSD"`, or `"ETH/USD"`. Invalid market strings will return empty results or errors.
- **Wrong price/quantity precision.** REST prices and quantities must be 8-decimal zero-padded strings (e.g., `"2500.05000000"`, `"1.00000000"`). Passing a plain number like `2500` or a string like `"2500.05"` will fail validation. Always pad to 8 decimal places.
- **Expecting the API to return typed data.** The client builds and signs the EIP-712 message, then submits once with `walletSignature`. Don't tell the user an order is placed until the signed request is accepted.
- **Stale nonces.** Nonces must be within 60s of server time — sync with `/v1/time`, and use a fresh nonce per request.
- **Not checking gas fees before withdrawal.** Withdrawals require a `maximumGasFee` parameter. Always query `/v1/gasFees` first to get current estimates per destination chain. Using a stale or too-low gas fee will cause the withdrawal to fail.
- **Confusing wallet address with API account.** Authenticated endpoints need both the wallet `0x` address AND valid API credentials. First-time users must associate the wallet (`POST /v1/wallets`) before any authenticated reads, trades or deposits will work.
- **Getting builder codes wrong.** The `clientOrderId` prefix only *tags* flow to a builder code — it does **not** set the fee rate (rates are configured off-chain on the web rewards page). The prefix must be part of the signed EIP-712 payload, not appended after signing, or the order is rejected. And never invent or hardcode a builder code: each integrator gets their own from the Katana team (Discord / kpsupport@katana.network).

## Safety Notes
- Trade operations need the user's wallet signature — the agent never signs or holds private keys
- Always read `exchangeContractAddress` live before signing or depositing
- Always check margin ratio before placing leveraged orders
- Check gas fees before withdrawals to set `maximumGasFee` accurately
- Keep margin ratio well below 1 to avoid liquidation
- Stop loss orders are not guaranteed to fill at the stop price
- Funding payments can accumulate — check `totalFunding` on positions

## Cross-References

- **wallet-manager**: `approve()` pattern for exchange deposit approvals, balance checks for vbUSDC
- **analytics**: on-chain spot price comparison with perps mark prices, contract reference for exchange addresses

## References

- REST API documentation: https://api-docs-v1-perps.katana.network/
- SDK documentation: https://sdk-js-docs-v1-perps.katana.network/
- Builder codes documentation: https://api-docs-v1-perps.katana.network/#builder-codes
- Builder rewards page (configure fees, claim earnings): https://perps.katana.network/rewards/builder
