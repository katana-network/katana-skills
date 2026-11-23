---
name: kat
description: Activate when the user asks about the KAT token, staking KAT, the avKAT vault, avKAT rewards or KAT buybacks, unstaking or exiting avKAT, exit fees, the 7-day cooldown, converting vKAT to avKAT, legacy/sunset vKAT positions or gauge voting, or building applications that interact with the KAT token ecosystem on Katana Network.
license: MIT
metadata:
  author: katana
  version: '2.1.0'
---

# KAT Token — Katana Network

The KAT token ecosystem on Katana Network: staking KAT into avKAT, how avKAT earns from revenue-funded KAT buybacks, exiting through the cooldown, and converting legacy vKAT positions.

> **vKAT staking, gauge voting and vote incentives were sunset on November 17, 2026.** avKAT is now the only way to stake KAT. Do not recommend creating new vKAT locks, voting on gauges, or claiming vote incentives. Existing vKAT holders should convert to avKAT (no deadline) or exit.

## KAT Token Overview

KAT is the native ERC-20 token of Katana, deployed on Katana Network (chain ID 747474). 18 decimals, fixed supply of 10 billion. KAT has been fully transferable since March 18, 2026.

## The Two Token Types

| Token | Standard | Transferable | Rewards |
|-------|----------|-------------|---------|
| **KAT** | ERC-20 | Yes | None |
| **avKAT** | ERC-4626 vault shares | Yes (liquid, tradeable, usable in DeFi) | Auto-compounded every epoch from KAT buybacks — nothing to claim |

**vKAT** (ERC-721 soulbound NFT in the VotingEscrow) still exists at the contract level, but it is **sunset** and **earns no rewards**. It now appears only as (a) legacy positions from before November 17, 2026, and (b) the temporary NFT created when unstaking avKAT through the vault.

### How avKAT earns

A share of the revenue from **Katana Swap** (built on Sushi), **Katana Perps**, **Vault Bridge**, and **sequencer fees** is used to buy KAT on the open market. **100% of that bought-back KAT is compounded into avKAT every epoch** via the CompoundStrategy.

- The avKAT balance stays the same; the amount of KAT each avKAT is redeemable for (`convertToAssets`) goes up.
- The exchange rate is measured in KAT — it says nothing about the market price of KAT or avKAT.
- Rewards depend on network usage (more trading, bridging and activity → more buybacks). They are variable and not guaranteed.
- Buybacks currently follow a scheduled process; fully automated onchain buybacks are planned.
- Rewards come from revenue-funded buybacks, **not** from token emissions, gauge voting, or vote incentives.
- Don't describe avKAT as guaranteed yield. Say rewards for KAT stakers are "compounded into avKAT".
- Epochs are timed by the Clock contract (`0x17049d374A2bcdA70F8939C21ad92bcF6B2A95ab`): `epochDuration()` = 1,209,600 s (**14 days**) at time of writing — read it live before quoting.

**Current revenue share to KAT buybacks** (current split; may change, and actual buybacks vary with network activity):

| Source | Share to buybacks |
|---|---|
| Katana Swap | 50% |
| Katana Perps | 5% |
| Vault Bridge | 5% |
| Sequencer fees | 30% |

**Old vs new avKAT** (the avKAT contract and address did not change):

| | Before Nov 17, 2026 | Now |
|---|---|---|
| How it earns | Voted through vKAT to earn fees | Protocol revenue, no voting |
| Yield sources | Sushi (now Katana Swap) fees only | Katana Swap, Katana Perps, Vault Bridge, sequencer fees |
| Yield handling | Fees autocompounded into avKAT | Revenue buys KAT on the open market; 100% compounded into avKAT |
| Exit cooldown | 45 days | 7 days |

## Contract Addresses (Mainnet)

| Contract | Address | Purpose |
|----------|---------|---------|
| KAT Token | `0x7F1f4b4b29f5058fA32CC7a97141b8D7e5ABDC2d` | ERC-20 token |
| avKAT Vault | `0x7231dbaCdFc968E07656D12389AB20De82FbfCeB` | ERC-4626 staking vault |
| VotingEscrow | `0x4d6fC15Ca6258b168225D283262743C623c13Ead` | Holds the vault's KAT; exits (begin/withdraw/cancel) go through it |
| NFT Lock | `0x106F7D67Ea25Cb9eFf5064CF604ebf6259Ff296d` | ERC-721 contract for vKAT NFTs |
| Exit Queue | `0x6dE9cAAb658C744aD337Ca5d92D084c97ffF578d` | Withdrawal queue, cooldown & exit fee |
| CompoundStrategy | `0x60233D1c150F9C08D886906d597aA79a205b0463` | Compounds bought-back KAT into the vault |
| Clock | `0x17049d374A2bcdA70F8939C21ad92bcF6B2A95ab` | Epoch timing |

Legacy (sunset) contracts — only relevant for old vKAT positions: GaugeVoter `0x5e755A3C5dc81A79DE7a7cEF192FFA60964c9352`, Curve `0x38b8B74330b2F918C22F7936aCf773C6D963C73c`, Swapper `0x92D2e00b6D2BB50B87a9BE971a82B1F00ac44768`.

## Token Flows

```
KAT   ──approve vault──→ avKATVault.deposit(assets, receiver) ──→ avKAT

avKAT ──avKATVault.withdrawTokenId(assets, receiver, owner)──→ new vKAT NFT
      ──VotingEscrow.beginWithdrawal(tokenId)──→ [cooldown]
      ──VotingEscrow.withdraw(tokenId)──→ KAT (no fee after full cooldown, up to 25% earlier)

avKAT ──swap on a DEX (Katana Swap)──→ KAT   (instant, market rate, trading fees + slippage)

legacy vKAT ──approve NFT to vault──→ avKATVault.depositTokenId(tokenId, receiver) ──→ avKAT
legacy vKAT ──beginWithdrawal → wait → withdraw──→ KAT
```

## Exit Fee Schedule

The new schedule (from the November 2026 tokenomics update) is a **7-day cooldown** with a fee that starts at **25%** and decreases linearly to **0%**.

| Time left in cooldown | Fee | 1,000 KAT → receive |
|---|---|---|
| 7 days (instant exit) | 25% | 750 KAT |
| 6 days | 21% | 790 KAT |
| 5 days | 18% | 820 KAT |
| 4 days | 14% | 860 KAT |
| 3 days | 11% | 890 KAT |
| 2 days | 7% | 930 KAT |
| 1 day | 4% | 960 KAT |
| Cooldown complete | 0% | 1,000 KAT |

```
feeBps   = MAX_FEE_BPS - (MAX_FEE_BPS - MIN_FEE_BPS) × elapsed / COOLDOWN   (elapsed clamped to COOLDOWN)
MAX_FEE_BPS = 2500   MIN_FEE_BPS = 0   COOLDOWN = 604800 s (7 days)   BPS_DENOMINATOR = 10000
```

**The on-chain ExitQueue is the source of truth — never hardcode the table above.**

- **Current parameters** for new exits: `cooldown()` (seconds), `feePercent()` (max, bps), `minFeePercent()` (min, bps). Read them before quoting a fee or cooldown. If they differ from the schedule above (for example around the November 2026 transition), tell the user and use the live values with the same linear formula.
- **Each exit locks its terms when it starts.** `queue(tokenId)` stores the cooldown and fee parameters in effect at `beginWithdrawal` time. An exit started under older terms (e.g. the pre-November 17 60-day cooldown with a 2.5% minimum fee) keeps them; it is not moved onto the new schedule.
- **Exact fee for a queued exit:** `calculateFee(tokenId)` returns the fee in KAT (wei) if withdrawn now. `canExit(tokenId)` says whether it can be withdrawn.

**Cancel**: during the cooldown, `VotingEscrow.cancelWithdrawalRequest(tokenId)` stops the exit. This leaves the user holding a **vKAT NFT, which earns nothing** — deposit it back into the vault (`depositTokenId`) to resume earning.

## Workflows

### Check KAT position
1. `balanceOf(address)` on KAT and on the avKAT vault
2. `VotingEscrow.ownedTokens(address)` — any vKAT NFTs (legacy positions or NFTs mid-exit); `locked(tokenId)` for each
3. For avKAT → KAT value, `avKATVault.convertToAssets(shares)` (each avKAT is currently worth more than 1 KAT)
4. KAT price: derive from the Katana Swap (Sushi V3) pool (see analytics skill)

### Stake KAT → avKAT
1. `approve(avKATVault, amount)` on the KAT token — spender is the avKAT Vault (`0x7231dbaCdFc968E07656D12389AB20De82FbfCeB`)
2. `avKATVault.deposit(amount, receiver)` — returns avKAT shares
3. Optionally preview with `previewDeposit(assets)` — the user receives fewer avKAT than KAT deposited because the exchange rate is above 1

### Unstake avKAT → KAT (cooldown)
1. **Redeem to NFT:** `avKATVault.withdrawTokenId(assets, receiver, owner)`. **The first argument is the KAT (assets) amount, not avKAT shares** — compute it with `convertToAssets(shares)` / `previewRedeem(shares)`. This burns avKAT and mints a **new** vKAT NFT. Get the token ID from the tx logs or `ownedTokens(owner)`.
   - **Minimum:** the KAT amount must be at least `VotingEscrow.minDeposit()` (230 KAT at time of writing). Smaller amounts revert with `AmountTooSmall()` — for small balances the only exit is a DEX swap.
2. `VotingEscrow.beginWithdrawal(tokenId)` — starts the cooldown and locks the exit terms
3. Wait. Track it with `ExitQueue.queue(tokenId)`, `canExit(tokenId)` and `calculateFee(tokenId)`
4. `VotingEscrow.withdraw(tokenId)` — receive KAT minus the fee at that moment (none after the full cooldown under the new schedule)

For an **instant exit through the vault** (maximum fee), do steps 2 and 4 back to back. Always tell the user the fee in KAT (`calculateFee`) before they choose this.

### Exit avKAT instantly via DEX
1. Quote avKAT → KAT on Katana Swap (see dex skill) — check output and price impact
2. Compare against `convertToAssets(shares)`: if the DEX output is well below the redeemable KAT, the cooldown is likely better
3. Approve the router for avKAT, then swap

### Convert legacy vKAT → avKAT
1. `VotingEscrow.ownedTokens(address)` and `locked(tokenId)` — list the user's vKAT NFTs and locked amounts
2. Approve the vault for the NFT on the NFT Lock (`0x106F7D67Ea25Cb9eFf5064CF604ebf6259Ff296d`): `approve(vault, tokenId)` or `setApprovalForAll(vault, true)`
3. `avKATVault.depositTokenId(tokenId, receiver)` — one-way; the NFT is consumed
4. No deadline. Warn the user they will receive **fewer avKAT than the KAT in the lock** (because 1 avKAT > 1 KAT); the avKAT is backed by the same amount of KAT

### Exit legacy vKAT → KAT
1. `VotingEscrow.isVoting(tokenId)` — check for pre-sunset gauge votes
2. If voting, use `VotingEscrow.resetVotesAndBeginWithdrawal(tokenId)` (atomic), or `GaugeVoter.reset(tokenId)` then `beginWithdrawal(tokenId)`
3. Wait out the cooldown, then `VotingEscrow.withdraw(tokenId)`

### Legacy vote incentives
Vote incentives ended on November 17, 2026. If a user has unclaimed pre-sunset Merkl rewards, they can still claim them (see merkl skill). avKAT has nothing to claim.

## Key Function Signatures

### avKAT Vault (ERC-4626)
```
deposit(uint256 assets, address receiver) → uint256 shares
depositTokenId(uint256 tokenId, address receiver) → uint256 shares     // vKAT → avKAT
withdrawTokenId(uint256 assets, address receiver, address owner) → uint256 tokenId   // avKAT → new vKAT NFT; assets = KAT amount, ≥ VotingEscrow.minDeposit()
previewDeposit(uint256 assets) → uint256 shares
previewRedeem(uint256 shares) → uint256 assets
convertToShares(uint256 assets) → uint256 shares
convertToAssets(uint256 shares) → uint256 assets
totalAssets() → uint256
balanceOf(address) → uint256
```

### VotingEscrow
```
beginWithdrawal(uint256 tokenId)
withdraw(uint256 tokenId)
cancelWithdrawalRequest(uint256 tokenId)
resetVotesAndBeginWithdrawal(uint256 tokenId)   // legacy vKAT that still has gauge votes
isVoting(uint256 tokenId) → bool
locked(uint256 tokenId) → (uint256 amount, uint256 start)
ownedTokens(address owner) → uint256[]
minDeposit() → uint256                          // minimum KAT per lock (230 KAT at time of writing)
```

### Exit Queue
```
cooldown() → uint256          // seconds, for new exits
feePercent() → uint256        // max fee, bps, for new exits
minFeePercent() → uint256     // min fee, bps, for new exits
queue(uint256 tokenId)        // exit entry: holder, queued timestamp, and the cooldown/fee params locked at queue time
canExit(uint256 tokenId) → bool
calculateFee(uint256 tokenId) → uint256   // fee in KAT (wei) if withdrawn now
```
There is **no** `getQueueEntry` function — calls to it revert.

### NFT Lock (ERC-721)
```
approve(address to, uint256 tokenId)
setApprovalForAll(address operator, bool approved)
```

### KAT Token
```
transfer(address to, uint256 amount)
approve(address spender, uint256 amount)
```

## Common Mistakes

- **Recommending vKAT staking or gauge voting.** Both were sunset on November 17, 2026. Don't suggest `createLock`, gauge votes, `merge` or `split` for new activity. Stake into avKAT instead.
- **Telling users to claim avKAT rewards.** avKAT compounds automatically every epoch; there is nothing to claim or restake.
- **Describing rewards as emissions or vote incentives.** avKAT rewards come from revenue-funded KAT buybacks (Katana Swap, Katana Perps, Vault Bridge, sequencer fees).
- **Using `redeem()` to unstake.** The unstake path is `withdrawTokenId` → `beginWithdrawal` → wait → `withdraw`.
- **Passing avKAT shares to `withdrawTokenId`.** Its first argument is the KAT (assets) amount. Passing a share count silently unstakes less than intended.
- **Unstaking less than the minimum.** Below `minDeposit()` (230 KAT at time of writing) of KAT value, `withdrawTokenId` reverts with `AmountTooSmall()`. Swap on the DEX instead.
- **Quoting the new schedule for an exit already in progress.** Each exit keeps the terms it was queued with — read `queue(tokenId)` / `calculateFee(tokenId)` for that token.
- **Calling `getQueueEntry`.** It doesn't exist on the ExitQueue. Use `queue`, `canExit` and `calculateFee`.
- **Not mentioning the fee-free option.** Exiting before the full cooldown costs up to 25%. Always state the fee in KAT and that waiting the full cooldown is free (under the new schedule).
- **Calling a DEX exit "free".** There's no protocol fee, but the swap is at the market rate and can be below the redeemable KAT value due to fees, price impact and thin liquidity. Quote first.
- **Leaving a cancelled exit as vKAT.** A cancelled withdrawal leaves a vKAT NFT that earns nothing — redeposit it with `depositTokenId`.
- **Surprise at fewer avKAT tokens.** Depositing KAT or converting vKAT returns fewer avKAT than KAT because each avKAT is worth more than 1 KAT. The underlying KAT is the same.
- **Approving the wrong contract.** Depositing KAT requires approving the avKAT Vault (`0x7231...CeB`) on the KAT token. Converting vKAT requires approving the vault on the NFT Lock (`0x106F...296d`), not the VotingEscrow.
- **Assuming the exchange rate is a price.** `convertToAssets` is measured in KAT; it says nothing about KAT's or avKAT's market price.
- **Relying on the block explorer.** `explorer.katana.network` sits behind Cloudflare and may return 403/1000 errors. Don't rely on it for ABI lookups — use this reference instead.

## Risks to Mention

- Smart contract risk in the vault and supporting contracts
- Variable rewards that depend on protocol revenue and network activity
- avKAT DEX liquidity risk (slippage, price below underlying KAT value)
- Exit fee of up to 25% before the cooldown completes
- The staking and buyback program may be modified or ended

## Legacy Reference: Gauge Voting (sunset)

Only needed to interpret or unwind pre-sunset vKAT positions. Katana's GaugeVoter uses non-standard names — do **not** assume Solidly/Velodrome ABIs:

```
reset(uint256 tokenId)
isVoting(uint256 tokenId) → bool
getAllGauges() → address[]
gauges(address gauge) → bool            // NOT isAlive/isGauge
gaugeVotes(address gauge) → uint256     // NOT weights
totalVotingPowerCast() → uint256        // NOT totalWeight
```

`weights()`, `totalWeight()`, `isAlive()`, `isGauge()`, `getActiveGauges()` and `poolForGauge()` do not exist. Gauge addresses are the pool addresses themselves (`token0()`, `token1()`, `fee()` work on them; `fee()` reverts on V2 pools).

LP incentives for Katana Swap pools are now allocated by the **Katana Foundation**, not by gauge votes. Links to pools: `https://app.katana.network/pools`.

## Cross-References

- **wallet-manager**: `approve()` patterns for the avKAT Vault and NFT Lock, balance checks for KAT/avKAT
- **dex**: swap avKAT ↔ KAT on Katana Swap for an instant exit (quote first)
- **merkl**: claim any leftover pre-sunset vote incentives; LP incentives on Katana Swap pools
- **analytics**: on-chain price derivation for KAT, contract reference for all KAT ecosystem addresses
- **lending**: Morpho markets may accept avKAT as collateral
