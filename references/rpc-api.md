# Octra RPC API Reference

## Transport

```
POST /rpc
Content-Type: application/json
```

JSON-RPC 2.0 format:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "METHOD_NAME",
  "params": ["positional", "args"]
}
```

Params are **positional arrays** — order matters exactly.

Main endpoint: `https://octra.network`

---

## Account Methods

```
octra_balance(address)
→ { formatted_balance, raw_balance, nonce, pending_nonce, pubkey_status }

octra_account(address, limit?)
→ account overview + recent transactions

octra_nonce(address)
→ confirmed nonce

octra_publicKey(address)
→ registered ed25519 public key

octra_validateAddress(address)
→ bool — address format check

octra_supply()
→ { total, max, burned }
```

---

## Transaction Methods

```
octra_submit(tx_json)
→ submit signed tx to staging; returns tx hash

octra_submitBatch([tx_json, ...])
→ batch submit

octra_transaction(hash)
→ { status: "pending"|"confirmed"|"rejected"|"dropped", ... }

octra_recentTransactions(limit?, offset?)
→ recent confirmed txs

octra_transactionsByAddress(address, limit?, offset?)
→ full tx history for address

octra_search(query)
→ universal lookup by hash, address, or epoch number
```

---

## Smart Contract Methods

```
vm_contract(address)
→ { owner, version, code_hash, balance }

octra_contractAbi(address)
→ ABI JSON

octra_contractStorage(address, key)
→ value at storage key

octra_listContracts()
→ list of deployed contracts

contract_receipt(hash)
→ {
    success: bool,
    effort_used: int,
    events: [...],
    error?: string
  }

contract_call(address, method, params?, caller?)
→ read-only execution result (no tx, no fee)

octra_computeContractAddress(bytecode_b64, deployer, nonce?)
→ deterministic deployment address
```

### contract_call example

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "contract_call",
  "params": [
    "octCONTRACT_ADDRESS...",
    "balance_of",
    ["octUSER_ADDRESS..."],
    "octCALLER_ADDRESS..."
  ]
}
```

---

## Compilation Methods

```
octra_compileAml(source_string)
→ {
    bytecode: "base64...",
    abi: [...],
    disassembly: "...",
    size: int,
    instruction_count: int,
    version: "..."
  }

octra_compileAmlMulti(files_object, main_filename)
→ same as above, for multi-file projects

files_object format:
{
  "main.aml": "source code...",
  "interfaces/IOCS01.aml": "interface source..."
}

octra_compileAssembly(oasm_source)
→ compiled OCTB bytecode from assembly
```

---

## Source Verification Methods

```
contract_verify(address, source_string, files?)
→ verify source against deployed bytecode

contract_saveAbi(address, abi_json)
→ store ABI for a contract

contract_source(address)
→ stored verified source and file map
```

---

## Node Methods

```
node_status()
→ { epoch, validator, roots, timestamp, network_version }

node_version()
→ node software version

node_stats()
→ { accounts, supply, tx_count, ... }

epoch_current()
→ { epoch_id, root_count }

epoch_get(epoch_id)
→ epoch summary, validator, timestamps, state root

octra_recommendedFee(op_type?)
→ { min, base, recommended, fast }
  op_types: "transfer" | "stealth" | "decrypt" | "contract_call" | "deploy"

staging_view()
→ all pending transactions in staging pool
```

---

## Common Integration Patterns

### Read contract state

```json
{
  "jsonrpc": "2.0", "id": 1,
  "method": "contract_call",
  "params": ["oct...contract", "total_staked", [], "oct...caller"]
}
```

### Compile and get ABI

```json
{
  "jsonrpc": "2.0", "id": 1,
  "method": "octra_compileAml",
  "params": ["program MyToken { state { ... } ... }"]
}
```

### Compile multi-file

```json
{
  "jsonrpc": "2.0", "id": 1,
  "method": "octra_compileAmlMulti",
  "params": [
    {
      "main.aml": "import IOCS01 from ...",
      "interfaces/IOCS01.aml": "interface IOCS01 { ... }"
    },
    "main.aml"
  ]
}
```

### Check tx status

```json
{
  "jsonrpc": "2.0", "id": 1,
  "method": "octra_transaction",
  "params": ["7dda0eae...tx_hash"]
}
```

### Get execution receipt

```json
{
  "jsonrpc": "2.0", "id": 1,
  "method": "contract_receipt",
  "params": ["7dda0eae...tx_hash"]
}
```

---

## FHE / Encryption Methods

```
octra_registerPublicKey(address, public_key, signature)
octra_registerPvacPubkey(address, pubkey_blob, aes_kat)
octra_pvacPubkey(address)
octra_encryptedBalance(address, signature, pubkey)
octra_privateTransfer(tx)
octra_viewPubkey(address)
octra_stealthOutputs(from_epoch?)
```

---

## Response Shape

Success:
```json
{ "jsonrpc": "2.0", "id": 1, "result": { ... } }
```

Error:
```json
{ "jsonrpc": "2.0", "id": 1, "error": { "code": -32600, "message": "..." } }
```
