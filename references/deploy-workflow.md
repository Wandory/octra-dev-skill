# Deploy Workflow — Octra Webcli IDE

## Setup: Install Webcli

```bash
# macOS
brew install openssl@3
git clone https://github.com/octra-labs/webcli
cd webcli
chmod +x setup.sh && ./setup.sh
./octra_wallet  # opens at http://127.0.0.1:8420

# Linux (Debian/Ubuntu)
sudo apt install g++ libssl-dev make
chmod +x setup.sh && ./setup.sh
./octra_wallet

# Windows — run setup.bat as Administrator, then open http://127.0.0.1:8420
```

Requirements: C++17 compiler, OpenSSL 3.x, libpvac (in pvac/ dir).

## First Launch

1. Open `http://127.0.0.1:8420`
2. Import private key (base64) or create new wallet
3. Set 6-digit PIN — used to encrypt wallet (AES-256-GCM)
4. Wallet stored at `data/wallet.oct`

## Wallet Setup (if using pre-client)

```json
{
  "priv": "your-base64-private-key",
  "addr": "octYOURADDRESS...",
  "rpc": "https://octra.network"
}
```

Wallet generator: https://wallet.octra.org

## IDE Layout

Open **Dev Tools** from top navigation.

```
Left panel:   File tree (main.aml, interfaces/, etc.)
Right panel:  Editor
Bottom tabs:  ABI | Assembly | Console | Storage

Below editor: Compile → Deploy → Call sections
```

## Create a Project

**From template:**
- `blank` — empty program
- `ocs-01 token` — standard token with IOCS01 interface
- `vault` — lock/release vault
- `escrow` — escrow logic
- `multisig` — multi-signature wallet

**Import existing:**
- `import files` — single or multiple files
- `import folder` — preferred for multi-file projects with interfaces/

## Compile

1. Set language to **AppliedML (.aml)**
2. Click **Compile**
3. Check output tabs:
   - **ABI** — callable methods and their signatures
   - **Assembly** — lowered OVM instructions (for audit/debug)
   - **Console** — compiler messages

If compilation fails, fix errors before continuing. Do NOT deploy with a failed compile.

## Preview Address

Before deploying, click **Preview Address**.

This computes the deterministic deployment address from:
- Your wallet address
- Current nonce
- Bytecode

Use this to:
- Know the address in advance
- Verify you're deploying from the right wallet
- Prepare subsequent cross-contract calls

Preview does NOT send a transaction.

## Deploy

Fill in:
- **Bytecode** — auto-filled after successful compile
- **Constructor params** — JSON array in positional order
- **Fee** — use recommended default

Click **Deploy**.

Deployment = uploading bytecode + running constructor. If constructor params are wrong, deployment fails.

### OCS01 Token Constructor

```json
["TokenName", "SYM", 1000000000000, 6]
```

Fields: name, symbol, raw supply, decimals.
With 6 decimals: 1,000,000,000,000 raw = 1,000,000 whole tokens.

### Staking Pool Constructor Example

```json
["octYOUR_GUARDIAN_ADDRESS..."]
```

## Call a Method

After deployment, use **Call Contract** section:

| Field | Value |
|---|---|
| Contract address | Paste deployed address |
| Method name | Exact method name (case-sensitive) |
| Params | JSON array `[]` or `["arg1", 100]` |
| Amount (OCT) | Only if method receives OCT (like `stake()`) |
| Fee | Use recommended |

### View (read-only) calls

Use **view (read-only)** button:
- No transaction
- No fee
- No nonce consumed
- Returns result immediately

Examples:
```
get_name          []
get_symbol        []
get_total_supply  []
balance_of        ["oct...address"]
exchange_rate     []
get_total_locked  []
```

### State-changing calls

Use **Send Call Tx** button:
- Creates on-chain transaction
- Consumes nonce, pays fee
- State changes persist

Examples:
```
transfer          ["oct...to", 1000]       amount: 0
stake             []                       amount: 5.0  (attach OCT)
request_unstake   [1000000]               amount: 0
claim             []                       amount: 0
pause             []                       amount: 0
```

**Important:** For token transfers, amount = 0. Only set amount when the program is designed to receive OCT (like a staking contract).

## Verify Source

After deployment:
1. Keep the same project open in IDE
2. Go to **Verify Contract Source**
3. Enter deployed address
4. Click verify

Verification links deployed bytecode back to source files. For multi-file projects, all files (main.aml + interfaces/) must be present.

## Check Results

After any call tx:
1. Copy the transaction hash
2. Check **History** tab in wallet
3. Look up on explorer: `https://octrascan.io`
4. Use **Last Receipt** in IDE to see execution result:
   - success/fail
   - effort consumed
   - emitted events
   - error message if failed

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| Compilation failed | Syntax error, missing interface file | Check console, keep interfaces/ intact |
| Deployment failed | Wrong constructor params | Check param types and order |
| `not owner` | Calling owner-only function from wrong address | Use deployer wallet |
| `insufficient balance` | Not enough token balance | Check `balance_of` first |
| `no value attached` | Called `stake()` without attaching OCT | Set Amount field > 0 |
| `still bonding` | Claiming before unbonding period ends | Wait until epoch >= unbonding_epoch |
| Method has no effect | Used view instead of send call tx | Switch to Send Call Tx |

## Full Flow: First Token Deploy

```
1. New project → ocs-01 token
2. Compile (AppliedML)
3. Preview address
4. Constructor: ["MyToken", "MTK", 1000000000000, 6]
5. Deploy
6. Copy address
7. View: get_name []  → should return "MyToken"
8. View: balance_of ["your_address"]  → should return 1000000000000
9. Send: transfer ["recipient_address", 1000]  amount: 0
10. View: balance_of ["recipient_address"]  → should return 1000
11. Verify source
```

## Full Flow: Staking Pool Deploy

```
1. New project → blank
2. Write StakingPool program in main.aml
3. Compile
4. Constructor: ["oct...guardian_address"]
5. Deploy
6. View: exchange_rate []  → should return 1000000 (1:1)
7. Send: stake []  amount: 1.0   (attach 1 OCT)
8. View: get_st_oct_balance ["your_address"]  → should return > 0
9. Send: request_unstake [1000000]  amount: 0
10. (wait unbonding period)
11. Send: claim []  amount: 0
```
