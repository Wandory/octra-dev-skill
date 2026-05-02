# Octra Network Overview

## Network Status (May 2026)

| | |
|---|---|
| **Mainnet Alpha** | Live since December 2025 |
| **ICO** | Completed April 2026, Uniswap CCA |
| **Peak TPS** | 17,000 tx/s |
| **OCT price** | ~$0.02 (ATH $0.08, April 22, 2026) |
| **Market cap** | ~$20M |
| **Total supply** | 1,000,000,000 OCT (fully unlocked) |

## Architecture

**4 layers:**
1. **Octra Network** — L1 blockchain + encrypted middleware
2. **Circles (IEEs)** — Isolated Execution Environments (up to 32MB state per Circle, can cluster)
3. **DSN** — Decentralised Storage Network (24 copies per fragment)
4. **Octra Protocol** — actor-model P2P communications

**Node types:** bootstrap, standard, light

**Consensus:** hybrid ABFT + Proof-of-Useful-Work (30+ parameters for validator scoring)

**FHE scheme:** HFHE (Hypergraph Fully Homomorphic Encryption) — proprietary, built from scratch

## OCT Token

- **Ticker:** $OCT
- **Decimals:** 6
- **1 OCT = 1,000,000 ou** (operational units)
- **Max supply:** 1,000,000,000 OCT
- **Genesis supply:** 630,000,000 OCT
- **Circulating at genesis:** 580,000,000 OCT

Supply APIs:
- `https://octra.network/circulating-supply`
- `https://octra.network/total-supply`
- `https://octra.network/max-supply`

## Token Allocation

| Allocation | % | Amount |
|---|---|---|
| Validator Rewards | 37% | 370,000,000 |
| Early Investors | 18.5% | 185,000,000 |
| Octra Labs | 15% | 150,000,000 |
| Ecosystem Fund | 10% | 100,000,000 |
| Uniswap CCA | 10% | 100,000,000 |
| Echo/Juicebox contributors | 4.87% | 48,700,000 |
| Faucet | 4.63% | 46,300,000 |

## Ethereum Bridge

**How it works:**
- OCT → wOCT: lock OCT on Octra → mint wOCT on Ethereum
- wOCT → OCT: burn wOCT on Ethereum → unlock OCT on Octra
- Bridge time: ~2 minutes
- Bridge fee: **0** (only normal gas)
- Ratio: 1:1, both use 6 decimals

**Octra-side functions:**
- `lock_to_eth` — locks OCT, initiates bridge to Ethereum
- `unlock_trusted` — releases OCT when bridge confirms burn

**wOCT on Ethereum:** standard ERC-20, no encrypted balances, usable in normal DeFi.

## Contract Addresses

### Ethereum
| Contract | Address |
|---|---|
| wOCT | `0x4647e1fE715c9e23959022C2416C71867F5a6E80` |
| EthereumBridge | `0xE7eD69b852fd2a1406080B26A37e8E04e7dA4caE` |
| OctraLightClient | `0xC01cA57dc7f7C4B6f1B6b87B85D79e5ddf0dF55d` |
| CCA (Uniswap auction) | `0xb3079ec6b82f22a1abfdca1a22659ab07cdf2f0f` |

### Octra
| Contract | Address |
|---|---|
| BridgeVault | `oct6tRiKpxxp62x8iRcaXtSj1u9EYQsZNYnJWZ3RmwceLi5` |

## Development Resources

| Resource | URL |
|---|---|
| Docs | `https://docs.octra.org` |
| Explorer (mainnet) | `https://octrascan.io` |
| Explorer (devnet) | `https://devnet.octrascan.io` |
| Wallet generator | `https://wallet.octra.org` |
| Bridge | `https://docs.octra.org/oct-docs/bridging` |
| Webcli (IDE) | `https://github.com/octra-labs/webcli` |
| Contract examples | `https://github.com/octra-labs/contract-examples` |
| Faucet (devnet) | `https://faucet.octra.network` |
| Dev email | `dev@octra.org` |
| Discord | `discord.gg/octra` |
| GitHub | `github.com/octra-labs` |

## Live Programs on Mainnet (May 2026)

- **AMM (Spot DEX)** — written entirely in AppliedLang by @wycfwycf
  - Working spot DEX with price chart, swaps, LP
- **FHE ML Contract** — by @lambda0xE (devnet)
  - On-chain AI inference, governance, treasury, ZK verification
- **SmolLM2-135M** — AI model deployed directly on-chain (April 2026)
- **OCS01 Standard Contract** — `octBUHw585BrAMPMLQvGuWx4vqEsybYH9N7a3WNj1WBwrDn`

## Programming Languages Supported

| Language | Status |
|---|---|
| **AppliedML (.aml)** | Native — available now |
| **Rust** | Via WASM — coming soon in webcli update |
| **C++** | Via WASM — coming soon |
| **Zig** | Via WASM — coming soon |
| **OCaml** | Used internally |
| **ReasonML** | Smart contracts interface |

## Investors

Finality Capital, Karatage, Amber Group, Big Brain Holdings, Presto Labs, Stratos, Curiosity Capital, Atka, Wise3 Ventures, Ivangbi (Ethereum Foundation), Scott Moore (Gitcoin), Vadim Koleoshkin (Zerion), Artemy Parshakov (P2P.org)

## Team

Founded 2021 by "Alex" and "David". David previously was database lead at VK (Vkontakte) under Pavel Durov. Many team members are ex-VK/Telegram contributors. Based in Zug, Switzerland.

## OCS01 Standard

OCS01 is Octra's token standard (analogous to ERC-20 on Ethereum).

Required interface:
```
fn transfer(to: address, amount: int): bool
fn grant(spender: address, amount: int): bool
fn pull(from: address, amount: int): bool
fn balance_of(addr: address): int
fn allowance(owner: address, spender: address): int
fn get_name(): string
fn get_symbol(): string
fn get_total_supply(): int
```

Test tool: `https://github.com/octra-labs/ocs01-test` (Rust CLI)
