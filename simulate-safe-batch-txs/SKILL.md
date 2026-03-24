# Simulate Gnosis Safe Batch Transactions on Anvil Forks

## Overview

Fork any EVM chain using Anvil, impersonate a Gnosis Safe multisig, and replay all transactions from Safe batch JSON files to verify correctness before multisig execution. Includes post-state assertions for LayerZero OFT configuration.

## When to Use

- Validating Safe batch JSON files before importing them into Safe UI
- Testing cross-chain OFT setup transactions (source or destination side)
- Verifying that `scripts/DeployFraxOFTProtocol/txs/*.json` or `scripts/ops/fix/*/txs/*.json` files are correct
- Smoke-testing LayerZero config changes (peers, DVNs, libraries, enforced options) on a fork

## Prerequisites

- Foundry (`anvil`, `cast`, `jq`)
- RPC URL for the target chain
- Safe/multisig address that owns the transactions
- Safe batch JSON files in the standard Gnosis Safe format

## Safe Batch JSON Format

Each file follows this structure:

```json
{
  "version": "1.0",
  "chainId": "252",
  "meta": { "name": "..." },
  "transactions": [
    {
      "to": "0xTargetContract",
      "value": "0",
      "data": "0xCalldata...",
      "operation": 0
    }
  ]
}
```

Key fields per transaction:
- `to` — target contract address
- `data` — ABI-encoded calldata
- `operation` — 0 = CALL, 1 = DELEGATECALL (should always be 0 for OFT setup)

## Procedure

### 1. Start Anvil Fork

```bash
anvil --fork-url <RPC_URL> --port <PORT> --chain-id <CHAIN_ID> &
ANVIL_PID=$!
sleep 3
```

Example for Fraxtal:

```bash
anvil --fork-url https://rpc.frax.com --port 8546 --chain-id 252 &
```

Example for Somnia:

```bash
anvil --fork-url https://api.infra.mainnet.somnia.network --port 8545 --chain-id 5031 &
```

### 2. Impersonate the Multisig

```bash
SAFE="0xYourSafeAddress"
RPC="http://127.0.0.1:<PORT>"

# Impersonate
cast rpc anvil_impersonateAccount "$SAFE" --rpc-url "$RPC"

# Fund with ETH for gas
cast rpc anvil_setBalance "$SAFE" 0x56BC75E2D63100000 --rpc-url "$RPC"
```

### 3. Replay All Transactions

Loop over all Safe batch JSON files and send each transaction:

```bash
FILES=(scripts/DeployFraxOFTProtocol/txs/5031-252-*.json)
total_ok=0
total_fail=0

for f in "${FILES[@]}"; do
  n=$(jq '.transactions | length' "$f")
  echo "=== $(basename "$f") — $n txs ==="

  for i in $(seq 0 $((n - 1))); do
    to=$(jq -r ".transactions[$i].to" "$f")
    data=$(jq -r ".transactions[$i].data" "$f")

    result=$(cast send "$to" "$data" \
      --from "$SAFE" \
      --rpc-url "$RPC" \
      --unlocked 2>&1)

    if echo "$result" | grep -q "transactionHash"; then
      total_ok=$((total_ok + 1))
    else
      echo "FAIL tx $i in $(basename "$f"): $result"
      total_fail=$((total_fail + 1))
    fi
  done
done

echo "total_ok=$total_ok total_fail=$total_fail"
```

### 4. Post-State Assertions (LayerZero OFT)

After replaying, verify on-chain state matches expected values:

#### Check peer

```bash
cast call <OFT_OR_LOCKBOX> "peers(uint32)(bytes32)" <REMOTE_EID> --rpc-url "$RPC"
```

Expected: left-padded bytes32 of the remote OFT address.

#### Check send library

```bash
cast call <ENDPOINT> "getSendLibrary(address,uint32)(address)" <OFT_OR_LOCKBOX> <REMOTE_EID> --rpc-url "$RPC"
```

#### Check receive library

```bash
cast call <ENDPOINT> "getReceiveLibrary(address,uint32)(address,bool)" <OFT_OR_LOCKBOX> <REMOTE_EID> --rpc-url "$RPC"
```

#### Check enforced options

```bash
cast call <OFT_OR_LOCKBOX> "enforcedOptions(uint32,uint16)(bytes)" <REMOTE_EID> 1 --rpc-url "$RPC"
cast call <OFT_OR_LOCKBOX> "enforcedOptions(uint32,uint16)(bytes)" <REMOTE_EID> 2 --rpc-url "$RPC"
```

#### Check DVN config (ULN send + receive)

```bash
# Send config (configType=2 for UlnConfig)
cast call <ENDPOINT> "getConfig(address,address,uint32,uint32)(bytes)" \
  <OFT_OR_LOCKBOX> <SEND_LIB> <REMOTE_EID> 2 --rpc-url "$RPC"

# Receive config (configType=2 for UlnConfig)
cast call <ENDPOINT> "getConfig(address,address,uint32,uint32)(bytes)" \
  <OFT_OR_LOCKBOX> <RECEIVE_LIB> <REMOTE_EID> 2 --rpc-url "$RPC"
```

Verify DVN addresses appear in the decoded config bytes.

### 5. Cleanup

```bash
kill $ANVIL_PID
```

## Batch File Integrity Checks

Before simulation, verify file metadata:

```bash
for f in "${FILES[@]}"; do
  chainId=$(jq -r '.chainId' "$f")
  txCount=$(jq '.transactions | length' "$f")
  nonCallOps=$(jq '[.transactions[] | select(.operation != 0)] | length' "$f")
  echo "$(basename "$f"): chainId=$chainId txs=$txCount nonCallOps=$nonCallOps"
done
```

All `nonCallOps` should be 0. Chain ID should match the target chain.

## Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| `cast send --data` flag unknown | Foundry version difference | Pass calldata as 2nd positional argument: `cast send "$to" "$data"` |
| Anvil fork hangs on start | RPC rate limiting | Add `--retries 3` or use a faster RPC |
| `insufficient funds` | Forgot to set balance | Run `anvil_setBalance` before replaying |
| Transaction reverts | Wrong Safe address (not the admin/owner) | Verify the Safe is the delegate/admin for the contracts |
| DVN config bytes look wrong | Decoded as raw bytes, not structured | Use `cast abi-decode` with the UlnConfig struct to parse |

## Reference: Common Ports

| Chain | Port |
|-------|------|
| Somnia (5031) | 8545 |
| Fraxtal (252) | 8546 |
| Ethereum (1) | 8547 |

Use different ports to run multiple forks simultaneously.
