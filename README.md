# Katana Skills Hub

Katana Skills Hub is an open skills collection that gives AI agents native access to DeFi on [Katana Network](https://katana.network) — a high-performance L2 blockchain. Check balances, execute swaps, manage lending positions, trade perpetual futures, claim rewards, and interact with the KAT token ecosystem, all through natural language.

Built by Katana. Built for everyone.

Skills Hub is designed for the entire agent ecosystem: any agent, any framework. Whether you're building on Claude Code, OpenClaw, LangChain, CrewAI, or your own stack, your agents can plug into Katana DeFi with a few lines of config.

---

## About This Repository

Each skill lives in its own folder under `skills/` and contains a `SKILL.md` file with YAML frontmatter and structured instructions that teach AI agents **when and how** to interact with Katana Network.

Each skill teaches an agent how to reason about on-chain DeFi operations — contract addresses, function signatures, workflows, safety rules, common mistakes, and cross-references between skills. Skills can be used as reference guides by any agent framework that interacts with Katana's on-chain contracts via standard JSON-RPC or any EVM library (viem, ethers.js, web3.py, etc.).

Browse the existing skills to understand patterns and naming conventions before contributing.

---

## Skills

| Skill | Activates When |
|-------|---------------|
| [`wallet-manager`](./skills/wallet-manager) | Balances, transfers, approvals, wrap/unwrap ETH |
| [`dex`](./skills/dex) | Token swaps, quotes, pool analysis, LP provision on SushiSwap |
| [`lending`](./skills/lending) | Morpho markets, vaults, positions, leverage loops |
| [`merkl`](./skills/merkl) | Reward discovery, yield farming, claiming Merkl rewards |
| [`analytics`](./skills/analytics) | Token prices, gas costs, tx lookup, contract reference |
| [`perps`](./skills/perps) | Perpetual futures — market data, orders, positions, withdrawals, and builder codes for building fee-earning perps products |
| [`kat`](./skills/kat) | KAT token, staking (vKAT), auto-compounding vault (avKAT), gauge voting |

---

## Katana Network

| Detail | Value |
|--------|-------|
| **Mainnet chain ID** | `747474` |
| **Testnet (Bokuto) chain ID** | `737373` |
| **Gas token** | ETH |
| **Block time** | 1 second |
| **Gas model** | EIP-1559 |
| **WETH** | Yield-generating vbToken (Vault Bridge ETH) with WETH9 interface |
| **Core protocols** | SushiSwap (DEX/AMM) + Morpho (lending) + Merkl (rewards) + Katana Perps (futures) |
| **RPC** | `https://rpc.katana.network/` (mainnet) / `https://rpc-bokuto.katanarpc.com` (testnet) |
| **Explorer** | [katanascan.com](https://katanascan.com/) (mainnet) / [bokuto.katanascan.com](https://bokuto.katanascan.com/) (testnet) |

---

## Installation

Get started with Katana Skills Hub in a single command. Works with various agents such as OpenClaw and Claude Code.

### Prerequisites

* **Node.js** (version 22 or higher)

### Install Skills Hub

Run the following command to add Katana Skills Hub to your project:

```bash
npx skills add https://github.com/katana-network/katana-skills
```

---

## Contribution

We welcome contributions.

To add a new skill:

1. **Fork the repository** and create a new branch:

   ```bash
   git checkout -b feature/<skill-name>
   ```

2. **Create a new folder** under `skills/` containing a `SKILL.md` file.

3. **Follow the required format:**

   ```markdown
   ---
   name: <skill-name>
   description: A clear description of what the skill does and when to activate it.
   license: MIT
   metadata:
     author: <your-github-username>
     version: <skill-version>
   ---

   # <Skill Name>

   [Add instructions, workflows, safety rules, common mistakes, and cross-references here]
   ```

4. **Open a Pull Request** to `main` for review. Once approved, the skill will be merged.

### Skill Structure Guidelines

Each `SKILL.md` should include:

- **Overview** — what the skill covers and key concepts
- **Contract Addresses** — deployed contract addresses on Katana mainnet
- **Function Signatures** — key function signatures and parameter documentation
- **Workflows** — step-by-step sequences for common tasks
- **Safety Notes** — risk notes and guardrails
- **Common Mistakes** — pitfalls agents should avoid
- **Cross-References** — links to related skills

---

## Disclaimer

Katana Skills Hub is an informational tool only. It and its outputs are provided on an "as is" and "as available" basis, without representation or warranty of any kind. It does not constitute investment, financial, trading, or any other form of advice, and does not represent a recommendation to buy, sell, or hold any digital assets. The accuracy, timeliness, or completeness of any data or analysis presented is not guaranteed. Your use of this tool and any information it provides is at your own risk — you are solely responsible for evaluating the information and for all decisions made based on it. AI-generated information or summaries should not be solely relied on for decision making and may include errors, biases, or outdated information. Digital asset prices are subject to high market risk and price volatility; the value of your investment may go down or up, and you may not get back the amount invested. You should carefully consider your investment experience, financial situation, investment objectives, and risk tolerance, and consult an independent financial adviser prior to making any investment. This tool is not responsible for any losses or damages incurred as a result of your use of or reliance on it.
