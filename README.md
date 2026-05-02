# octra-dev-skill

A skill for [Claude Code](https://claude.ai/code) that provides full knowledge of the Octra FHE blockchain developer stack — including AppliedML language syntax, program deployment workflow, RPC API, and network overview.

## What it does

When installed, this skill gives Claude Code complete context for:

- Writing programs in **AppliedML (.aml)** — Octra's native language
- Building **OCS01 tokens**, **staking pools**, **AMMs**, **vaults** on Octra
- Compiling, deploying, and testing programs via the **Octra webcli IDE**
- Calling contracts via **JSON-RPC 2.0**
- Understanding the **bridge** between Octra and Ethereum (OCT ↔ wOCT)
- Network addresses, tokenomics, live program examples

## Structure

```
SKILL.md                          ← Main skill file (loaded on trigger)
references/
  appliedml-language.md           ← Full AML syntax, types, patterns, examples
  deploy-workflow.md              ← IDE setup, compile → deploy → call → verify
  rpc-api.md                      ← Complete JSON-RPC 2.0 API reference
  network-overview.md             ← Network info, bridge, tokenomics, addresses
```

## Install in Claude Code

```bash
claude skill install https://github.com/Wandory/octra-dev-skill
```

Or manually — copy the repo and point Claude Code to the `SKILL.md`.

## Triggers

The skill activates when you mention:
`Octra`, `AppliedLang`, `.aml`, `OVM`, `OCS01`, `StakeOctra`, `webcli`, `OCT token`, `Circles`, `stOCT`, `octrascan`

## Example prompts

```
Write a liquid staking contract for Octra in AppliedML
Deploy an OCS01 token on Octra mainnet
Build an AMM DEX program in .aml
How do I call a contract via Octra RPC?
What is the exchange rate formula for StakeOctra?
How do I bridge OCT to Ethereum?
```

## About Octra

[Octra](https://octra.org) is an FHE (Fully Homomorphic Encryption) L1 blockchain with support for isolated execution environments (Circles). Programs run on the OVM (Octra Virtual Machine) and are written in AppliedML. Mainnet alpha is live since December 2025.

- Docs: https://docs.octra.org
- Explorer: https://octrascan.io
- GitHub: https://github.com/octra-labs
- Discord: https://discord.gg/octra

## Based on

All content is derived from the official [Octra documentation](https://docs.octra.org) (May 2026).
