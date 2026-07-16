# Verify Foundry Broadcast Deployment on a Live Chain

## Overview

End-to-end verification that a Foundry `broadcast/<SCRIPT>.s.sol/<CHAIN_ID>/run-latest.json`
deployment was actually executed on-chain and matches the locally compiled artifacts. This
covers CREATE / CREATE2 deployments and, for upgrade scripts, confirms whether the proxy
upgrade has been applied by the multisig yet.

The workflow treats the **broadcast file as the source of truth** and cross-checks it
against:

1. The on-chain transaction receipt (status, from, to, contractAddress).
2. The on-chain deployed bytecode (keccak256 of runtime code).
3. The locally compiled artifact (`out/<Contract>.sol/<Contract>.json`).
4. For CREATE2: the deterministic address derivation (factory + salt + init code hash).
5. For upgrade scripts: the proxy's EIP-1967 implementation slot vs. the deployed impl,
   and the generated Safe batch file (`txs/<chainId>-<msig>.json`).

## When to Use

- After running `forge script ... --broadcast` and wanting to confirm the deployment landed.
- Verifying a CREATE2 deployment derived from the Arachnid factory
  (`0x4e59b44847b379578588920cA78FbF26c0B4956C`).
- Auditing an upgrade script's broadcast to determine whether the **implementation was only
  deployed** vs. **deployed AND the proxy was upgraded**.
- Producing evidence (codehash match, address derivation match) for a deployment review.

## Prerequisites

- Foundry (`cast`, `forge`, `jq`)
- RPC URL for the target chain
- The broadcast directory: `broadcast/<SCRIPT>.s.sol/<CHAIN_ID>/run-latest.json`
- A compiled `out/<Contract>.sol/<Contract>.json` artifact (run `forge build` first)

## Procedure

### 1. Inspect the broadcast summary

```bash
jq '{
  chain: .chain,
  transactions: [.transactions[] | {
    hash,
    transactionType,
    contractName,
    contractAddress,
    arguments: (.arguments // null)
  }]
}' broadcast/<SCRIPT>.s.sol/<CHAIN_ID>/run-latest.json
```

Capture:

- `transactions[].hash` — the on-chain tx hash to verify
- `transactions[].contractAddress` — the deployed address
- `transactions[].transactionType` — `CREATE` or `CREATE2`
- `transactions[].transaction.to` — for CREATE2 this is the factory address

### 2. Fetch the on-chain receipt

```bash
RPC="<RPC_URL>"
TX="<TX_HASH>"

cast receipt "$TX" --rpc-url "$RPC"
```

Confirm:

- `status` == `1` (success)
- `from` matches the broadcaster / `--sender`
- `contractAddress` (for CREATE) or `to` == factory (for CREATE2)
- `blockNumber` is reasonable (not 0 / not pending)

### 3. Compare deployed bytecode keccak

This is the strongest single check: the keccak256 of the on-chain runtime bytecode must
equal the keccak256 of the compiled artifact's `deployedBytecode.object`.

```bash
RPC="<RPC_URL>"
ADDR="<DEPLOYED_ADDRESS>"
ARTIFACT="out/<Contract>.sol/<Contract>.json"

# On-chain runtime code hash
cast code "$ADDR" --rpc-url "$RPC" | cast keccak

# Compiled runtime code hash
jq -r '.deployedBytecode.object' "$ARTIFACT" | cast keccak
```

**Expect both hashes to be identical.** If they differ:

- Rebuild (`forge build`) — the artifact may be stale.
- Check whether the script selected a different contract variant (e.g. `RemoteHopV201`
  vs. `RemoteHopV201Tempo`) based on `block.chainid`.
- See the `metadata-tail-ethereum-verification` skill if only the metadata tail differs.

### 4. (CREATE2 only) Verify deterministic address derivation

For CREATE2 deployments, recompute the expected address from the factory, salt, and init
code hash, then confirm it matches the broadcast's `contractAddress`.

```bash
FACTORY="<factory address, e.g. 0x4e59b44847b379578588920cA78FbF26c0B4956C>"
SALT="<salt from script>"
ARTIFACT="out/<Contract>.sol/<Contract>.json"

INIT_HASH=$(jq -r '.bytecode.object' "$ARTIFACT" | cast keccak)

cast create2 \
  --init-code-hash "$INIT_HASH" \
  --salt "$SALT" \
  --deployer "$FACTORY"
```

**Expect** the derived address to equal:

- `transactions[].contractAddress` in the broadcast, AND
- the `require(newImplementation == 0x..., "...")` assertion in the script (if present).

The salt and factory are typically hardcoded in the deploy script — read the script source
to extract them. For the standard Arachnid factory, the salt often begins with
`0x4e59b44847b379578588920ca78fbf26c0b4956c...` (the factory address prefix).

### 5. (Upgrade scripts only) Check whether the proxy was actually upgraded

A common point of confusion: an "upgrade" broadcast often only **deploys the new
implementation**. The actual `upgradeAndCall` is performed later by the multisig via a
Safe batch. To determine the real state:

#### 5a. Read the proxy's EIP-1967 implementation slot

```bash
RPC="<RPC_URL>"
HOP="<proxy address>"

cast storage "$HOP" \
  0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc \
  --rpc-url "$RPC"
```

Compare the stored address against:

- The broadcast's `contractAddress` (the freshly deployed impl), AND
- The impl address embedded in the generated Safe batch file
  (`txs/<chainId>-<msig>.json`, decoded from the `upgradeAndCall` calldata).

**Three possible states:**

| Impl slot value | Meaning |
|-----------------|---------|
| Equals broadcast `contractAddress` | Upgrade already executed ✓ |
| Equals a *different* address | An older impl is live; upgrade is **pending** |
| Zero / empty | Wrong slot read, or not an EIP-1967 proxy — re-check proxy type |

#### 5b. Identify the proxy admin and multisig

```bash
# EIP-1967 admin slot
cast storage "$HOP" \
  0xb53127684a568b3173ae13b9f8a6016e243e63b6e8eeae8d8795b881c3e1e8d2 \
  --rpc-url "$RPC"

# If admin slot is zero, the proxy may store admin elsewhere; try:
cast call "$HOP" "admin()(address)" --rpc-url "$RPC"
```

Then confirm the admin contract's owner is the multisig that the Safe batch is addressed
to:

```bash
cast call "<proxyAdmin>" "owner()(address)" --rpc-url "$RPC"
```

The owner should match the msig address in the Safe batch filename
(`txs/<chainId>-<msig>.json`).

#### 5c. Decode the generated Safe batch

```bash
cat txs/<chainId>-<msig>.json | jq '.transactions'
```

For a standard HopV201 upgrade batch, expect two transactions:

1. `upgradeAndCall(proxy, newImpl, "")` → sent to the ProxyAdmin
   - selector `0x9623609d`
   - decode the `newImpl` argument (bytes 36–68 of the data) and confirm it equals the
     broadcast's `contractAddress`
2. `grantRole(RECOVER_ETH_ROLE, msig)` → sent to the proxy
   - selector `0x2f2ff15d`
   - the role hash is the first 32-byte arg; verify it matches
     `cast keccak "RECOVER_ETH_ROLE()"`

#### 5d. Verify role grant state (if applicable)

```bash
# RECOVER_ETH_ROLE hash
ROLE=$(cast keccak "RECOVER_ETH_ROLE()")

cast call "$HOP" "hasRole(bytes32,address)(bool)" "$ROLE" "<msig>" --rpc-url "$RPC"
```

If `false`, the role grant has not been executed yet (upgrade batch still pending).

### 6. Summarize the verification

Produce a table covering:

- Chain ID + RPC used
- Tx hash + status + block
- Deployed address
- On-chain codehash vs. compiled codehash (must match)
- CREATE2 derivation (if applicable): derived address vs. broadcast address vs. script
  `require` address (all three must match)
- For upgrades: current impl slot vs. deployed impl vs. Safe-batch target impl, and
  whether the upgrade + role grant are applied or pending

## Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| `cast code` returns `0x` | Address has no code — wrong chain, wrong address, or tx not yet mined | Re-check chain ID and tx receipt |
| Codehash mismatch after `forge build` | Stale `out/` artifact or wrong contract variant selected by `block.chainid` branch | Rebuild; inspect the script's `if (block.chainid == X)` branches |
| CREATE2 derived address ≠ broadcast | Wrong salt, wrong factory, or init code hash computed from `bytecode.object` instead of the full init code including constructor args | For no-arg constructors, `bytecode.object` is sufficient; for constructors with args, use the full `transactions[].transaction.input` |
| EIP-1967 admin slot is zero | Proxy is a `TransparentUpgradeableProxy` whose admin is stored at the `0xb531...` slot but the slot read returned zero — the proxy may use a custom admin pattern, or the read hit the impl's storage | Read the slot via `cast storage` on the proxy address directly; if still zero, dump the proxy runtime bytecode and look for the admin slot constant |
| `implementation()` reverts | The proxy does not expose the `ITransparentUpgradeableProxy` interface (e.g. it's a UUPS or custom proxy) | Use the raw EIP-1967 storage slot read instead |
| Upgrade appears "not applied" | The broadcast only deployed the impl; the `upgradeAndCall` lives in the Safe batch and must be executed by the msig separately | This is expected — confirm via the Safe batch file and the on-chain impl slot |

## Reference: EIP-1967 Slot Constants

| Slot | Hash of | Used for |
|------|---------|----------|
| `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc` | `eip1967.proxy.implementation` | Implementation address |
| `0xb53127684a568b3173ae13b9f8a6016e243e63b6e8eeae8d8795b881c3e1e8d2` | `eip1967.proxy.admin` | Proxy admin address |
| `0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50` | `eip1967.proxy.beacon` | Beacon address (for beacon proxies) |

## Reference: Common CREATE2 Factories

| Factory | Address |
|---------|---------|
| Arachnid | `0x4e59b44847b379578588920cA78FbF26c0B4956C` |
| Solady CREATE3 | `0x0000000000ffe384F9ADfF8D038d6EBf8C7C0cB3` |

For the Arachnid factory, the deployer sends a tx **to** the factory address with the
salt+initcode encoded in the calldata; the factory performs the CREATE2.
