---
name: kat
description: Activate when the user asks about the KAT token, staking KAT, vKAT locks, avKAT vault, gauge voting, exit fees, cooldown periods, or building applications that interact with the KAT token ecosystem on Katana Network.
license: MIT
metadata:
  author: katana
  version: '1.0.0'
---

# KAT Token — Katana Network

The KAT token ecosystem on Katana Network, including staking (vKAT), auto-compounding vaults (avKAT), gauge voting, and exit mechanics.

## KAT Token Overview

KAT is the native ERC-20 token of the Katana Foundation, deployed on Katana Network (chain ID 747474). It has 18 decimals.

**KAT is fully transferable.** All token flows — `transfer()`, `transferFrom()`, staking (KAT → vKAT), and vault deposits (KAT → avKAT) — are enabled.

KAT itself has **no voting power**. Users must convert it into one of two staking derivatives to participate in governance and earn rewards.

## The Three Token Types

| Token | Standard | Transferable | Voting Power | Rewards |
|-------|----------|-------------|-------------|---------|
| **KAT** | ERC-20 | Yes | None | None |
| **vKAT** | ERC-721 (non-transferable NFT) | No (non-transferable) | Yes — 1:1 with locked amount | Manual claim via Merkl |
| **avKAT** | ERC-4626 vault shares | Yes (liquid, tradeable) | Delegated to CompoundStrategy | Auto-compounded |

### vKAT (Active Staking)
A non-transferable NFT representing a locked KAT position in the VotingEscrow contract. Each lock is a separate NFT with its own token ID. Users can hold multiple vKAT NFTs. Grants direct gauge voting power. Exiting requires a cooldown period with a sliding exit fee — parameters are read live from the on-chain ExitQueue contract and may change over time.

### avKAT (Passive Staking)
A liquid, transferable ERC-4626 vault token. The vault holds a "master vKAT" internally. The CompoundStrategy handles voting and reward reinvestment automatically. The exchange rate appreciates as rewards compound. Users can exit instantly by selling avKAT on SushiSwap (no cooldown, no protocol fee, subject to market slippage) or redeem through the vault (triggers the standard cooldown period).

## Contract Addresses (Mainnet)

| Contract | Address | Purpose |
|----------|---------|---------|
| KAT Token | `0x7F1f4b4b29f5058fA32CC7a97141b8D7e5ABDC2d` | ERC-20 token |
| VotingEscrow | `0x4d6fC15Ca6258b168225D283262743C623c13Ead` | Lock KAT → mint vKAT NFT |
| NFT Lock | `0x106F7D67Ea25Cb9eFf5064CF604ebf6259Ff296d` | ERC-721 contract for vKAT positions |
| avKAT Vault | `0x7231dbaCdFc968E07656D12389AB20De82FbfCeB` | ERC-4626 auto-compounding vault |
| GaugeVoter | `0x5e755A3C5dc81A79DE7a7cEF192FFA60964c9352` | Vote vKAT power on gauges |
| CompoundStrategy | `0x60233D1c150F9C08D886906d597aA79a205b0463` | Auto-compounds avKAT vault rewards |
| Exit Queue | `0x6dE9cAAb658C744aD337Ca5d92D084c97ffF578d` | Manages cooldown & exit fee calculation |
| Curve | `0x38b8B74330b2F918C22F7936aCf773C6D963C73c` | Voting power decay & exit fee curve |
| Clock | `0x17049d374A2bcdA70F8939C21ad92bcF6B2A95ab` | Epoch timing |
| Swapper | `0x92D2e00b6D2BB50B87a9BE971a82B1F00ac44768` | Swaps non-KAT rewards to KAT for compounding |
| Merkl Distributor | `0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae` | Merkle-based reward claiming |

## Token Conversion Flows

```
KAT ──approve──→ VotingEscrow.createLock(amount) ──→ vKAT (NFT)
KAT ──approve──→ avKATVault.deposit(amount, receiver) ──→ avKAT (shares)

vKAT ──approve NFT──→ avKATVault (deposit NFT) ──→ avKAT   (one-way, NFT consumed)

vKAT ──beginWithdrawal──→ [cooldown period] ──withdraw──→ KAT (minus exit fee)
vKAT ──beginWithdrawal + withdraw (same block)──→ KAT     (rage quit, max fee)

avKAT ──sell on DEX──→ KAT                                 (instant, no protocol fee)
avKAT ──vault.redeem──→ triggers standard cooldown ──→ KAT
avKAT ──vault.withdrawTokenId──→ vKAT (new NFT) ──→ then standard exit
```

### Key conversion details:
- **KAT → vKAT**: Call `approve(VotingEscrow, amount)` on KAT token, then `VotingEscrow.createLock(amount)`. Mints a non-transferable NFT.
- **KAT → avKAT**: Call `approve(vault, amount)` on KAT token, then `avKATVault.deposit(amount, receiver)`. Returns vault shares at current exchange rate.
- **vKAT → avKAT**: One-way. The NFT is consumed and merged into the vault's master position.
- **vKAT → KAT**: Call `beginWithdrawal(tokenId)`, wait for cooldown, call `withdraw(tokenId)`. Fee decays from max → min (read live from ExitQueue contract).
- **avKAT → KAT (instant)**: Sell avKAT on SushiSwap (see dex skill). No cooldown, no protocol fee, subject to slippage.
- **avKAT → KAT (redeem)**: Call `vault.redeem(shares, receiver, owner)`. Triggers the standard cooldown.

## Exit Fee Mechanics

When unstaking vKAT, a fee is charged based on how long the user waits during the cooldown period. The cooldown duration and fee bounds are **governed on-chain** via the ExitQueue contract and can change over time.

**ExitQueue contract:** `0x6dE9cAAb658C744aD337Ca5d92D084c97ffF578d`

**On-chain parameters (read via view functions):**
- `cooldown()` — cooldown period in seconds
- `feePercent()` — max exit fee in bps (rage quit fee)
- `minFeePercent()` — min exit fee in bps (after full cooldown)

**Formula:**
```
fee = maxFee - ((maxFee - minFee) × timeWaited / cooldown)
```

**Important:** Do not hardcode these values — always read them from the ExitQueue contract, as governance can update them. The MCP tools (`build_kat_begin_withdrawal`, `build_kat_withdraw`) include live `exitParams` in their responses.

**Cancel withdrawal**: During cooldown, `cancelWithdrawalRequest(tokenId)` restores the staking position and voting power.

## Gauge Voting

vKAT holders vote directly on gauges to direct KAT emission incentives to liquidity pools.

- **Vote**: `GaugeVoter.vote(tokenId, [{gauge, weight}])` — weights in basis points (10,000 = 100%)
- **Reset**: `GaugeVoter.reset(tokenId)` — clear all votes (required before withdrawing a vKAT position)
- **Combined**: `VotingEscrow.resetVotesAndBeginWithdrawal(tokenId)` — atomic reset + begin withdrawal
- Votes persist across epochs until changed or reset
- avKAT holders do **not** vote directly — the CompoundStrategy votes automatically

## vKAT Lock Management

- **Merge**: `VotingEscrow.merge(fromTokenId, toTokenId)` — combine two locks into one. Burns the "from" lock.
- **Split**: `VotingEscrow.split(tokenId, amount)` — create a new lock with a portion of KAT. Returns new token ID.

## Reward Distribution

Rewards come from three sources:
1. **KAT emission schedule** — protocol inflation
2. **Pool trading fees** — from Sushi DEX
3. **Exit fees** — from users who unstake

Distribution uses Merkle tree proofs via the Merkl Distributor. vKAT holders claim manually; avKAT holders get automatic compounding via the CompoundStrategy.

To check unclaimed rewards, query the Merkl API: `https://api.merkl.xyz/v4/users/{address}?chainId=747474` (see merkl skill for claiming workflow).

## Key Function Signatures

### VotingEscrow (vKAT)
```
createLock(uint256 amount) → uint256 tokenId
beginWithdrawal(uint256 tokenId)
withdraw(uint256 tokenId)
cancelWithdrawalRequest(uint256 tokenId)
resetVotesAndBeginWithdrawal(uint256 tokenId)
merge(uint256 fromTokenId, uint256 toTokenId)
split(uint256 tokenId, uint256 amount) → uint256 newTokenId
```

### avKAT Vault (ERC-4626)
```
deposit(uint256 assets, address receiver) → uint256 shares
redeem(uint256 shares, address receiver, address owner) → uint256 assets
convertToShares(uint256 assets) → uint256 shares
convertToAssets(uint256 shares) → uint256 assets
```

### GaugeVoter

**IMPORTANT:** Katana's GaugeVoter uses non-standard function names. Do NOT assume Solidly/Velodrome ABIs.

```
vote(uint256 tokenId, (address gauge, uint256 weight)[])
reset(uint256 tokenId)
getAllGauges() → address[]
gauges(address gauge) → bool              # check if gauge is active (NOT isAlive/isGauge)
gaugeVotes(address gauge) → uint256       # votes for a gauge (NOT weights/votes)
totalVotingPowerCast() → uint256          # total voting power (NOT totalWeight)
isVoting(uint256 tokenId) → bool
usedVotingPower(uint256 tokenId) → uint256
```

**Functions that DO NOT exist on this contract** (will revert):
- `weights(address)` — use `gaugeVotes(address)` instead
- `totalWeight()` — use `totalVotingPowerCast()` instead
- `isAlive(address)` — use `gauges(address)` instead
- `isGauge(address)` — use `gauges(address)` instead
- `getActiveGauges()` — use `getAllGauges()` + `gauges(addr)` filter
- `poolForGauge(address)` — does not exist (see Gauge = Pool below)

### ExitQueue
```
cooldown() → uint256 seconds
feePercent() → uint256 bps (max exit fee, rage quit)
minFeePercent() → uint256 bps (min exit fee, after full cooldown)
queue(uint256 tokenId) → (address holder, uint48 createdAt)
canExit(uint256 tokenId) → bool
```

### KAT Token
```
transfer(address to, uint256 amount)
approve(address spender, uint256 amount)
```

## Workflows

### Check KAT ecosystem state
1. Read KAT price from Sushi V3 pool (see analytics skill for price derivation)
2. Call `balanceOf(address)` on KAT token (`0x7F1f...2d`) and avKAT vault (`0x7231...eB`)
3. See Contract Addresses table above for all ecosystem contracts

### Stake KAT → vKAT (Active staking)
1. Call `approve(VotingEscrow, amount)` on KAT token — approve VotingEscrow (`0x4d6fC15Ca6258b168225D283262743C623c13Ead`)
2. Call `VotingEscrow.createLock(amount)` — mints a vKAT NFT

### Deposit KAT → avKAT (Passive staking)
1. Call `approve(avKATVault, amount)` on KAT token — approve avKAT Vault (`0x7231dbaCdFc968E07656D12389AB20De82FbfCeB`)
2. Call `avKATVault.deposit(amount, receiver)` — returns avKAT shares

### Unstake vKAT → KAT
1. Ensure votes are reset: call `GaugeVoter.reset(tokenId)` or use `resetVotesAndBeginWithdrawal(tokenId)`
2. Call `VotingEscrow.beginWithdrawal(tokenId)` — starts cooldown period
3. Wait for cooldown to pass (fee decreases over time)
4. Call `VotingEscrow.withdraw(tokenId)` — receive KAT minus exit fee

### Exit avKAT → KAT (instant, no cooldown)
1. Sell avKAT on SushiSwap — execute a swap via V3 SwapRouter (see dex skill for swap workflow)
2. No protocol fee, subject to DEX slippage

### Claim rewards (vKAT holders)
1. Query Merkl API for unclaimed rewards: `https://api.merkl.xyz/v4/users/{address}?chainId=747474`
2. Fetch proofs and call `claim()` on Merkl Distributor (see merkl skill for claiming workflow)

## Gauge = Pool (Katana-specific)

On Katana, gauge addresses **ARE** the pool addresses. There is no separate `poolForGauge()` mapping. To get pool info, call standard Uniswap V3 pool functions directly on the gauge address:

| Function | Signature | Notes |
|----------|-----------|-------|
| Token 0 | `token0() → address` | Works on gauge address directly |
| Token 1 | `token1() → address` | Works on gauge address directly |
| Fee tier | `fee() → uint24` | 100=0.01%, 500=0.05%, 3000=0.3%, 10000=1% |

**V2 vs V3 gauges:** Some gauges are V2 pools where `fee()` will revert. `token0()` and `token1()` still work. Handle gracefully and label as "V2".

## Common Mistakes

- **KAT is fully transferable.** All transfers, staking to vKAT, and depositing to avKAT are enabled.
- **Withdrawing without resetting votes.** A vKAT position with active gauge votes cannot begin withdrawal. Call `GaugeVoter.reset(tokenId)` first, or use the combined `resetVotesAndBeginWithdrawal(tokenId)`.
- **Confusing vKAT and avKAT exit paths.** vKAT requires a cooldown period with exit fee (params read from ExitQueue contract). avKAT can be sold instantly on DEX with no protocol fee. Users wanting immediate exit should use the avKAT → DEX path.
- **Expecting instant rewards.** Merkl rewards are computed offchain every ~2 hours and merkle roots are pushed onchain every ~8 hours. New stakers won't see rewards immediately.
- **Approving the wrong contract.** Staking to vKAT requires approving VotingEscrow (`0x4d6f...3Ead`). Depositing to avKAT requires approving the vault (`0x7231...CeB`). Mixing these up will revert.
- **Assuming Solidly/Velodrome ABI.** Katana's GaugeVoter uses non-standard function names. `weights()`, `totalWeight()`, `isAlive()`, `isGauge()`, `getActiveGauges()`, and `poolForGauge()` do NOT exist. See the GaugeVoter section above for correct names.
- **Looking up gauge pools separately.** Gauge addresses ARE pool addresses on Katana. Call `token0()`, `token1()`, `fee()` directly on the gauge address.
- **Relying on the block explorer.** `explorer.katana.network` sits behind Cloudflare and may return 403/1000 errors. Don't rely on it for ABI lookups — use this reference instead.
- **Calling `fee()` on V2 gauges.** Some gauges are V2 pools where `fee()` reverts. Handle gracefully.

## Cross-References

- **wallet-manager**: `approve()` patterns for VotingEscrow or avKAT Vault, balance checks for KAT/avKAT
- **dex**: sell avKAT on SushiSwap for instant exit without cooldown
- **merkl**: query Merkl API for KAT staking reward APRs, claiming workflow for vKAT rewards
- **analytics**: on-chain price derivation for KAT, contract reference for all KAT ecosystem addresses
- **lending**: Morpho markets may accept avKAT as collateral
