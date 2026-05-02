# AppliedML Language Reference

AppliedML (AML) is Octra's high-level program language. Source compiles to OCTB bytecode for the OVM.

## Design Philosophy

AML hides boilerplate but NOT execution behavior. Developers can always inspect:
- Storage layout (predictable key construction)
- Event emission
- Caller/origin semantics
- Bytecode via disassembly tab

## Types

### Primitive
```
int       -- integer only, NO floats
bool      -- true / false
string    -- text
address   -- Octra address (oct...)
bytes     -- raw bytes
```

### Composite
```
map[K]V             -- key-value map
map[K]map[K2]V      -- nested map (approvals, grants)
list[T]             -- ordered list
option[T]           -- nullable value
struct { ... }      -- named field grouping
enum { A, B, C }    -- variants
(T1, T2)            -- tuple
```

## State Block

```aml
state {
  owner: address
  paused: int                          // int used as bool (0/1)
  total_supply: int
  balances: map[address]int
  grants: map[address]map[address]int  // nested: owner → spender → amount
  status: Status                       // enum type
  items: list[string]
  maybe_addr: option[address]
}
```

State does NOT need initialization in the state block — maps return 0 for unset keys.

## Storage Key Construction

| State field | Storage key pattern |
|---|---|
| `self.owner` | `"owner"` |
| `self.balances[addr]` | `"balances:" + addr` |
| `self.grants[a][b]` | `"grants:" + a + ":" + b` |
| `self.items[0]` | `"items:0"` |
| `self.items` length | `"items:len"` |

This is important for auditing and understanding storage layout.

## Execution Context (built-in values)

```aml
caller      // address of direct caller (most common)
origin      // address of original tx sender
self_addr   // this program's own address
value       // OCT attached to call, in raw ou (int)
            // 1 OCT = 1,000,000 ou
epoch       // current epoch number (int)
```

## Functions

```aml
// Read-only — cannot modify state
view fn get_balance(addr: address): int {
  return self.balances[addr]
}

// State-changing — public entrypoint
fn transfer(to: address, amount: int): bool {
  // ... mutate state, emit events
  return true
}

// Internal helper — not callable from outside
private fn only_owner() {
  require(caller == self.owner, "not owner")
}
```

## Checks and Reverts

```aml
require(condition, "message")  // revert with message if false
assert condition               // hard assertion, no message
revert                         // unconditional rollback

// ALL state changes in the call are rolled back on revert
// No partial commits
```

## Local Variables

```aml
fn example(): bool {
  let bal = self.balances[caller]    // immutable local
  let mut count = self.count         // mutable local (if supported)
  return true
}
```

## Arithmetic

```aml
// Integer only — no floats
self.total_staked += value           // increment
self.total_staked -= amount          // decrement
let result = a * b / c               // multiply BEFORE divide (avoid truncation)
let fee = amount * fee_rate / 10000  // basis points pattern
```

## Events

```aml
// Declaration (top-level in program body):
event Transfer(from: address, to: address, amount: int)
event Staked(from: address, oct_in: int, st_oct_out: int, epoch: int)

// Emission (inside function):
emit Transfer(caller, to, amount)
emit Staked(caller, value, minted, epoch)
```

Events are observable — external tools and indexers can listen to them.

## Interfaces

```aml
// interfaces/IOCS01.aml
interface IOCS01 {
  fn transfer(to: address, amount: int): bool
  fn grant(spender: address, amount: int): bool
  fn pull(from: address, amount: int): bool
  fn balance_of(addr: address): int
  fn allowance(owner: address, spender: address): int
  fn get_name(): string
  fn get_symbol(): string
  fn get_total_supply(): int
}

// main.aml
import IOCS01 from "interfaces/IOCS01.aml"

program Token implements IOCS01 {
  // ... must implement all interface functions
}
```

## Inter-Program Calls

```aml
// Call another deployed program
call(other_contract_address, "method_name", [param1, param2])

// Deploy a child program
let new_addr = deploy(bytecode)

// Send native OCT to an address
transfer(recipient_address, amount_in_ou)
```

## Lowered Assembler Output (for inspection)

After compiling, the Assembly tab shows how AML becomes OVM instructions:

```asm
LDI r2, "transfer"         // load method name
EQ r3, r1, r2              // check dispatch
JIF r3, 800                // jump if match
CALLER r4                  // load caller address
LDI r5, "balances:"        // storage key prefix
CONCAT r5, r5, r4          // build full key
SLOAD r6, r5               // load from storage
EMIT "Transfer", r4, r1, r2  // emit event
SSTORE r5, r6              // write to storage
```

This is visible to help with auditing and debugging.

## Encrypted Computation (advanced)

AML supports FHE operations for private programs:
- Public-key loading
- Homomorphic arithmetic on ciphertexts
- Proof verification
- Commitments and ciphertext serialization

For encrypted programs: clients encrypt/decrypt, programs operate on ciphertexts, network verifies state transitions.

## Complete Example: Vault

```aml
program Vault {
  state {
    owner: address
    guardian: address
    paused: int
    total_locked: int
    lock_nonce: int
    balances: map[address]int
  }

  event Locked(sender: address, amount: int, nonce: int)
  event Released(to: address, amount: int)
  event Paused(by: address)
  event Unpaused(by: address)

  constructor(guardian_addr: address) {
    require(is_address(guardian_addr), "invalid guardian")
    self.owner = caller
    self.guardian = guardian_addr
    self.paused = 0
    self.total_locked = 0
    self.lock_nonce = 0
  }

  private fn only_owner() {
    require(caller == self.owner, "not owner")
  }

  private fn not_paused() {
    require(self.paused == 0, "paused")
  }

  fn lock(): bool {
    not_paused()
    require(value > 0, "no value attached")
    self.total_locked += value
    self.lock_nonce += 1
    self.balances[caller] += value
    emit Locked(caller, value, self.lock_nonce)
    return true
  }

  fn release(to: address, amount: int): bool {
    only_owner()
    require(amount <= self.total_locked, "insufficient vault balance")
    self.total_locked -= amount
    transfer(to, amount)
    emit Released(to, amount)
    return true
  }

  fn pause(): bool {
    only_owner()
    self.paused = 1
    emit Paused(caller)
    return true
  }

  fn unpause(): bool {
    only_owner()
    self.paused = 0
    emit Unpaused(caller)
    return true
  }

  view fn get_total_locked(): int {
    return self.total_locked
  }

  view fn get_balance(addr: address): int {
    return self.balances[addr]
  }
}
```
