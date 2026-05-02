---
name: octra-dev
description: >
  Complete guide for building, compiling, deploying, and interacting with programs on the Octra FHE blockchain network.
  Use this skill whenever the user wants to: write a program or smart contract for Octra, use AppliedML (.aml) language,
  deploy to Octra mainnet or devnet, interact with Octra RPC, build DeFi protocols (staking, DEX, tokens, vaults) on Octra,
  write OCS01-standard tokens, understand how Circles or OVM work, use the Octra webcli IDE, bridge OCT to Ethereum (wOCT),
  or ask anything about developing on Octra. Trigger on keywords: Octra, AppliedLang, .aml, OVM, OCS01, stOCT, StakeOctra,
  webcli, octrascan, OCT token program, Circles, HFHE program.
---

# Octra Developer Skill

Full reference for building programs on the Octra FHE L1 blockchain using AppliedML (`.aml`).

For deep reference on a topic, read the relevant file in `references/`:
- `references/appliedml-language.md` — Full AML syntax, types, patterns
- `references/programs-structure.md` — Program anatomy, state, events, constructors
- `references/deploy-workflow.md` — IDE setup, compile, deploy, call, verify
- `references/rpc-api.md` — Full JSON-RPC 2.0 API reference
- `references/network-overview.md` — Network, bridge, tokenomics, addresses

---

## 1. Quick Mental Model

Octra is an FHE L1 blockchain. Programs run in isolated execution environments called **Circles** on the **OVM** (Octra Virtual Machine). The native language is **AppliedML (.aml)**. Programs compile to **OCTB bytecode**.

Key terms:
| Term | Meaning |
|---|---|
| **Circle** | Isolated execution environment (like a smart contract + server) |
| **OVM** | Octra Virtual Machine — executes OCTB bytecode |
| **AppliedML / AML** | Native high-level language for Octra programs |
| **OCTB** | Compiled bytecode format |
| **OCS01** | Standard token interface (like ERC-20) |
| **OCT** | Native token, 6 decimals, 1 OCT = 1,000,000 ou |
| **ou** | Operational unit — smallest OCT unit (like wei) |
| **epoch** | Network time unit — available in programs via `epoch` |
| **caller** | Address of the direct caller |
| **origin** | Address of the original transaction sender |
| **value** | OCT amount attached to the call (in raw ou units) |

---

## 2. AppliedML Language — Core Syntax

### Minimal program shape

```aml
import IOCS01 from "interfaces/IOCS01.aml"

program Token implements IOCS01 {
  state {
    owner: address
    balances: map[address]int
    grants: map[address]map[address]int
    total_supply: int
    paused: int
  }

  event Transfer(from: address, to: address, amount: int)
  event Paused(by: address)

  constructor(name: string, symbol: string, supply: int, decimals: int) {
    self.owner = caller
    self.total_supply = supply
    self.balances[origin] = supply
    self.paused = 0
  }

  private fn only_owner() {
    require(caller == self.owner, "not owner")
  }

  view fn balance_of(addr: address): int {
    return self.balances[addr]
  }

  fn transfer(to: address, amount: int): bool {
    let bal = self.balances[caller]
    require(bal >= amount, "insufficient balance")
    self.balances[caller] = bal - amount
    self.balances[to] = self.balances[to] + amount
    emit Transfer(caller, to, amount)
    return true
  }
}
```

### Types

```
int          -- integer (no float in AML)
bool
string
address      -- Octra address (oct...)
bytes

map[K]V      -- e.g. map[address]int
map[K]map[K2]V  -- nested map (for approvals/grants)
list[T]
option[T]
struct
enum
tuples
```

### Execution context builtins

```aml
caller        // address that called this function
origin        // original transaction sender
self_addr     // this program's own address
value         // OCT attached to call (in raw ou units, int)
epoch         // current epoch number (int)
```

### Control flow and checks

```aml
require(condition, "error message")   // reverts if false
assert condition                       // hard assumption
revert                                 // unconditional rollback

// All state changes roll back on revert/require failure
```

### Events

```aml
// Declaration (in program body):
event Staked(from: address, amount: int, minted: int)

// Emission (in function body):
emit Staked(caller, value, st_oct_amount)
```

### Functions

```aml
view fn get_total(): int { ... }       // read-only, no state change
fn stake(): bool { ... }               // state-changing, public
private fn only_owner() { ... }        // internal helper
```

### Variables and maps

```aml
let bal = self.balances[caller]        // local variable
self.total_staked += value             // field update (+= works)
self.balances[caller] = bal - amount   // map write
self.grants[owner][spender] = amount   // nested map write
```

### Inter-program primitives

```aml
call(contract_address, method, params)  // call another program
deploy(bytecode)                        // deploy child program
transfer(address, amount)               // send native OCT
```

### Interfaces

```aml
// interfaces/IOCS01.aml
interface IOCS01 {
  fn transfer(to: address, amount: int): bool
  fn balance_of(addr: address): int
  fn grant(spender: address, amount: int): bool
  fn pull(from: address, amount: int): bool
  fn allowance(owner: address, spender: address): int
  fn get_name(): string
  fn get_symbol(): string
  fn get_total_supply(): int
}

// Import in main program:
import IOCS01 from "interfaces/IOCS01.aml"
program MyToken implements IOCS01 { ... }
```

---

## 3. Storage Model

State lowers into **string-addressed storage keys**:
- Simple fields → stored by field name (`"owner"`, `"total_supply"`)
- Map entries → composite keys (`"balances:" + address`)
- Nested maps → extended key path (`"grants:" + owner + ":" + spender`)
- Lists → indexed entries + length field

This means storage layout is predictable and auditable.

---

## 4. Standard Token (OCS01) — Constructor Params

When deploying the `ocs-01 token` template from the IDE, constructor params are:

```json
["MyToken", "MTK", 1000000000000, 6]
```

- `"MyToken"` — token name
- `"MTK"` — token symbol  
- `1000000000000` — raw total supply (with 6 decimals = 1,000,000 whole tokens)
- `6` — decimals

Initial supply is assigned to `origin` (deployer wallet) at construction.

---

## 5. Staking Pattern (for StakeOctra / liquid staking)

```aml
program StakingPool {
  state {
    owner: address
    total_staked: int           // total OCT locked
    total_st_oct: int           // total stOCT minted
    st_oct_balances: map[address]int   // stOCT per user
    unbonding_amount: map[address]int  // OCT queued for withdrawal
    unbonding_epoch: map[address]int   // epoch when can claim
    paused: int
  }

  event Staked(from: address, oct_in: int, st_oct_out: int)
  event UnstakeRequested(from: address, st_oct_in: int, epoch_unlock: int)
  event Claimed(to: address, oct_amount: int)

  constructor(owner_addr: address) {
    self.owner = caller
    self.total_staked = 0
    self.total_st_oct = 0
    self.paused = 0
  }

  // Exchange rate: how much OCT per 1 stOCT
  view fn exchange_rate(): int {
    if self.total_st_oct == 0 {
      return 1000000   // 1:1 at start (raw units)
    }
    return self.total_staked / self.total_st_oct
  }

  fn stake(): bool {
    require(self.paused == 0, "paused")
    require(value > 0, "no OCT attached")
    // OCT is received via `value`
    let st_oct_amount = value * 1000000 / self.exchange_rate()
    self.total_staked += value
    self.total_st_oct += st_oct_amount
    self.st_oct_balances[caller] += st_oct_amount
    emit Staked(caller, value, st_oct_amount)
    return true
  }

  fn request_unstake(st_oct_amount: int): bool {
    require(self.st_oct_balances[caller] >= st_oct_amount, "insufficient stOCT")
    let oct_amount = st_oct_amount * self.exchange_rate() / 1000000
    self.st_oct_balances[caller] -= st_oct_amount
    self.total_st_oct -= st_oct_amount
    self.total_staked -= oct_amount
    self.unbonding_amount[caller] += oct_amount
    self.unbonding_epoch[caller] = epoch + 604800  // ~7 days in epochs
    emit UnstakeRequested(caller, st_oct_amount, self.unbonding_epoch[caller])
    return true
  }

  fn claim(): bool {
    require(epoch >= self.unbonding_epoch[caller], "still bonding")
    let amount = self.unbonding_amount[caller]
    require(amount > 0, "nothing to claim")
    self.unbonding_amount[caller] = 0
    transfer(caller, amount)
    emit Claimed(caller, amount)
    return true
  }
}
```

**Key:** `value` is how users send OCT to a program — no separate transfer needed. The user attaches OCT to the `stake()` call.

---

## 6. AMM / DEX Pattern (from live mainnet example)

```aml
// import IBCS01 from "interface/IBCS01.aml"

contract AMM {
  deployer: address
  state {
    fee_rate: int   // basis points, 100 = 1%
    token_x: address
    token_y: address
    reserve_x: int
    reserve_y: int
    total_lp: int
    lp_balances: map[address]int
  }

  event Swap(from: address, to: address, is_buy: bool,
             amount_in: int, amount_out: int)
  event AddLiquidity(from: address, to: address, amount: int)
  event RemoveLiquidity(from: address, amount: int)

  constructor(token_x: address, token_y: address, fee_rate: int) {
    require(fee_rate >= 0 && fee_rate <= 9999, "invalid fee rate")
    self.deployer = caller
    self.token_x = token_x
    ...
  }
}
```

---

## 7. Deploy Workflow (Step by Step)

1. **Open dev tools** in webcli (`http://127.0.0.1:8420`) → Dev Tools tab
2. **New project** → choose template: `blank`, `ocs-01 token`, `vault`, `escrow`, `multisig`
3. **Edit** `main.aml` (and `interfaces/IOCS01.aml` if using OCS01)
4. **Language selector**: set to `AppliedML (.aml)`
5. **Click Compile** → check ABI, Assembly, Console tabs
6. **Preview address** → see deployment address without sending tx
7. **Enter constructor params** as JSON array: `["name", "SYM", 1000000, 6]`
8. **Click Deploy** → pays fee, runs constructor
9. **Copy deployed address**
10. **Test with view calls**:
    - Method: `get_name`, Params: `[]`
    - Method: `balance_of`, Params: `["oct...address"]`
11. **Send state-changing calls** via `send call tx` (NOT view):
    - Method: `transfer`, Params: `["oct...to", 1000]`, Amount: `0`
    - Method: `stake`, Params: `[]`, Amount: `5.0` (attach OCT)
12. **Verify source** → enter deployed address + verify project files

**Common mistakes:**
- Wrong constructor param order → deployment fails
- Using `view` for state-changing methods → no effect
- Adding OCT amount to token transfer calls (should be 0)
- Deleting `interfaces/IOCS01.aml` → breaks compilation

---

## 8. RPC Quick Reference

Endpoint: `POST /rpc` with JSON-RPC 2.0

```json
{ "jsonrpc": "2.0", "id": 1, "method": "METHOD", "params": [] }
```

**Most used methods:**

```
octra_balance(address)              → balance, nonce
octra_submit(tx_json)               → submit tx
contract_call(addr, method, params, caller?)  → read-only call
vm_contract(address)                → contract metadata
octra_contractAbi(address)          → ABI
octra_compileAml(source)            → compile single file
octra_compileAmlMulti(files, main)  → compile multi-file project
octra_computeContractAddress(bytecode_b64, deployer, nonce?)
contract_receipt(hash)              → execution result
octra_recommendedFee(op_type?)      → fee suggestion
epoch_current()                     → current epoch
```

Full RPC reference → `references/rpc-api.md`

---

## 9. Network Info

| | |
|---|---|
| **Mainnet RPC** | `https://octra.network` |
| **Explorer** | `https://octrascan.io` |
| **Devnet explorer** | `https://devnet.octrascan.io` |
| **Wallet generator** | `https://wallet.octra.org` |
| **Webcli** | `https://github.com/octra-labs/webcli` |
| **Contract examples** | `https://github.com/octra-labs/contract-examples` |
| **Dev email** | `dev@octra.org` |
| **Discord** | `discord.gg/octra` |

**Ethereum contract addresses:**
| Contract | Address |
|---|---|
| wOCT (ERC-20) | `0x4647e1fE715c9e23959022C2416C71867F5a6E80` |
| EthereumBridge | `0xE7eD69b852fd2a1406080B26A37e8E04e7dA4caE` |
| OctraLightClient | `0xC01cA57dc7f7C4B6f1B6b87B85D79e5ddf0dF55d` |
| BridgeVault (Octra) | `oct6tRiKpxxp62x8iRcaXtSj1u9EYQsZNYnJWZ3RmwceLi5` |

---

## 10. Key Gotchas

- **`value` is in raw `ou` units** (1 OCT = 1,000,000 ou). Always work in raw units in programs.
- **Revert rolls back all state** — no partial commits.
- **No floats** — all math is integer. Use basis points (10000 = 100%) for percentages.
- **Exchange rate math**: multiply before divide to avoid integer truncation.
- **`caller` vs `origin`**: `caller` = direct caller, `origin` = tx initiator. For user-facing programs, usually use `caller`.
- **Maps don't need initialization** — accessing `self.balances[newAddr]` returns `0` by default.
- **Constructor params are positional JSON array** — order matters.
- **Multi-file projects**: use `import folder` in IDE, not individual file imports, to keep the file tree intact.
- **`value` only exists when caller attaches OCT** — always `require(value > 0, ...)` before using it.
