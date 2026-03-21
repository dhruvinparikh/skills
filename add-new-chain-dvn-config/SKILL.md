# Add New Chain — DVN Config & L0Config

## Overview

When a new EVM chain is added to `frax-oft-upgradeable`, these files must be updated:

1. **DVN config files** in `config/dvn/` — one JSON file per chain (consumed by Foundry's `SetDVNs.s.sol` and TypeScript LayerZero configs)
2. **L0Config.json** in `scripts/L0Config.json` — central LayerZero config (consumed by `BaseL0Script.sol` and TS configs)
3. **hardhat.config.ts** — network entry for the new chain
4. **config/dvn/providers.json** — RPC endpoints for the Frax DVN service
5. **scripts/L0Constants.sol** — only if the chain uses non-standard OFT addresses

## When to Use

- Adding a new EVM chain to the LayerZero OFT bridge mesh
- Fixing a chain ID (renaming DVN file + updating all cross-references)
- Adding/removing a DVN provider for a chain
- Correcting DVN addresses in L0Config.json

## Important Rules

- **Never modify `non-evm.json`, `providers.json`, or `testnet-providers.json`** — these have different structures
- **No trailing newline in JSON files** — files must end with `}` (no `\n` after the closing brace)
- **Preserve original key ordering** — keys in DVN files are NOT sorted numerically; they follow a specific insertion order that must be preserved
- **4-space indentation** for all DVN JSON files
- **Self-entry must be all zeros** — each chain's DVN file has its own chain ID as a key with all DVN addresses set to `0x0000000000000000000000000000000000000000`

## DVN Config File Structure

### Location

```
frax-oft-upgradeable/config/dvn/{chainId}.json
```

### File per chain

Each chain has its own JSON file (e.g., `1.json` for Ethereum, `50312.json` for Somnia). The file contains DVN addresses for pathways from that chain to every other chain.

### Entry structure

Each entry is keyed by a destination chain ID and contains DVN provider addresses:

```json
{
    "34443": {
        "bcwGroup": "0x0000000000000000000000000000000000000000",
        "frax": "0x38654142f5e672ae86a1b21523aafc765e6a1e08",
        "horizen": "0x380275805876Ff19055EA900CDb2B46a94ecF20D",
        "lz": "0x589dedbd617e0cbcb916a9223f4d1300c294236b",
        "nethermind": "0x0000000000000000000000000000000000000000",
        "stargate": "0x0000000000000000000000000000000000000000"
    }
}
```

DVN providers (in order): `bcwGroup`, `frax`, `horizen`, `lz`, `nethermind`, `stargate`.

### Self-entry pattern

The chain's own chain ID entry has ALL DVN addresses set to zero:

```json
{
    "1": {
        "bcwGroup": "0x0000000000000000000000000000000000000000",
        "frax": "0x0000000000000000000000000000000000000000",
        "horizen": "0x0000000000000000000000000000000000000000",
        "lz": "0x0000000000000000000000000000000000000000",
        "nethermind": "0x0000000000000000000000000000000000000000",
        "stargate": "0x0000000000000000000000000000000000000000"
    }
}
```

### Key ordering

Keys are **NOT** sorted numerically or alphabetically. They follow a specific insertion order. New chains are typically inserted at the end, just before the `111111111`, `22222222`, `33333333` block. The current order (as of March 2026, 34 chains) is:

```
34443, 1329, 252, 196, 146, 57073, 42161, 10, 137, 43114,
56, 1101, 1, 81457, 8453, 80094, 59144, 2741, 324, 480,
130, 3637, 98866, 747474, 1313161554, 534352, 999, 988, 143, 4217,
50312, 111111111, 22222222, 33333333
```

When adding a new chain, insert its key **before `111111111`** in this order.

### DVN addresses per file

Each DVN file uses its **own chain's DVN contract addresses** for pathway entries, not the destination chain's addresses. The DVN addresses vary per chain — look at the most common (non-zero, non-self) entry in the file to determine the standard addresses for that chain.

To find the standard addresses for a given file, use the most common address per DVN key (excluding the all-zeros self-entry):

```python
from collections import Counter, OrderedDict
import json

DVN_KEYS = ["bcwGroup", "frax", "horizen", "lz", "nethermind", "stargate"]
ZERO = "0x0000000000000000000000000000000000000000"

with open(f"{chain_id}.json") as f:
    data = json.loads(f.read(), object_pairs_hook=OrderedDict)

addr_counts = {dk: Counter() for dk in DVN_KEYS}
for k, v in data.items():
    if all(v[dk] == ZERO for dk in DVN_KEYS):
        continue  # skip self-entry
    for dk in DVN_KEYS:
        addr_counts[dk][v[dk]] += 1

standard = {dk: addr_counts[dk].most_common(1)[0][0] for dk in DVN_KEYS}
```

## L0Config.json Structure

### Location

```
frax-oft-upgradeable/scripts/L0Config.json
```

### Top-level arrays

The file has 4 arrays: `"Legacy"`, `"Proxy"`, `"Non-EVM"`, `"Testnet"`.

- **`Proxy`** — chains where Frax deploys upgradeable OFT proxies. **New EVM chains go here.**
- **`Legacy`** — older deployments (Eth, Metis, Blast, Base) with a different deployment pattern
- **`Non-EVM`** — Solana, Movement, Aptos
- **`Testnet`** — testnet chains

Some chains appear in both `Legacy` and `Proxy` (chainids 1, 81457, 1088, 8453) — `BaseL0Script.sol` deduplicates them.

### Entry structure

Each chain has an entry in one of the arrays:

```json
{
    "RPC": "https://rpc-url",
    "chainid": 50312,
    "delegate": "0x...",
    "dvnHorizen": "0x...",
    "dvnL0": "0x...",
    "eid": 30380,
    "endpoint": "0x...",
    "proxyAdmin": "0x...",
    "receiveLib302": "0x...",
    "sendLib302": "0x..."
}
```

### Key fields

| Field | Description |
|-------|-------------|
| `chainid` | The chain's EVM chain ID |
| `eid` | LayerZero endpoint ID for this chain |
| `dvnHorizen` | Horizen DVN contract address on this chain |
| `dvnL0` | LayerZero DVN contract address on this chain |
| `delegate` | Delegate address (multisig) |
| `endpoint` | LZ endpoint contract address |
| `sendLib302` / `receiveLib302` | LZ messaging library addresses |

### Common mistake

`dvnHorizen` and `dvnL0` are easily confused with nethermind or other DVN addresses. Always verify against the [LayerZero DVN metadata API](https://metadata.layerzero-api.com/v1/metadata) or on-chain data.

## Step-by-Step: Adding a New Chain

### 1. Create the new DVN config file

Create `config/dvn/{newChainId}.json`:
- Copy key ordering from any existing mainnet file (e.g., `1.json`)
- Add the new chain ID as a key (before `111111111`)
- Set the self-entry (new chain's own ID) to all zeros
- Set all pathway entries with the new chain's DVN addresses (from LZ metadata API)
- **Do not add a trailing newline**

### 2. Add the new chain ID to all other mainnet DVN files

For each of the existing mainnet DVN files:
- Add a `"{newChainId}"` key before `"111111111"`
- Use that file's standard DVN addresses (same as other pathway entries in that file)
- Preserve original key ordering
- **Do not modify testnet files or non-evm.json**

### 3. Update L0Config.json

Add an entry to the **`"Proxy"`** array (not Legacy) with:
- Correct `chainid`
- Correct `eid` (LayerZero endpoint ID)
- Correct `dvnHorizen` and `dvnL0` addresses
- RPC URL, endpoint, delegate, proxyAdmin, send/receive lib addresses

### 4. Update hardhat.config.ts

Add a network entry for the new chain:
```typescript
newChainName: {
    eid: EndpointId.NEW_CHAIN_V2_MAINNET,
    url: "https://rpc-url",
    accounts,
}
```
This is required for HardHat/LayerZero tooling used by the TypeScript configs.

### 5. Update config/dvn/providers.json

Add RPC endpoints for the Frax DVN service:
```json
"chainname": {
    "uris": ["https://rpc-1", "https://rpc-2"],
    "quorum": 2
}
```
This file is consumed by the Frax DVN infrastructure (not by scripts in this repo).

### 6. Update scripts/L0Constants.sol (conditional)

Only needed if the new chain uses **non-standard OFT addresses** (not the pre-determined proxy addresses). If so:
- Add `address public` variables for each OFT
- Add a `chainNameProxyOfts` array
- Add a `_registerChain(chainid, chainNameProxyOfts)` call in the constructor

Standard proxy chains using `expectedProxyOfts` need **no changes** — they fall through to the default in `_getChainPeers()`.

### 7. Which files to modify

**Mainnet DVN files** (modify these):
```
1.json, 10.json, 56.json, 130.json, 137.json, 143.json, 146.json,
196.json, 252.json, 324.json, 480.json, 988.json, 999.json, 1101.json,
1329.json, 2741.json, 3637.json, 4217.json, 8453.json, 34443.json,
42161.json, 43114.json, 50312.json, 57073.json, 59144.json, 80094.json,
81457.json, 98866.json, 534352.json, 747474.json, 22222222.json,
33333333.json, 111111111.json, 1313161554.json
```

**Do NOT modify**:
- `non-evm.json` — different structure (Solana/Aptos addresses, 64-char hex), consumed by Move TS configs
- `providers.json` — update separately (step 5), different structure
- `testnet-providers.json` — testnet provider metadata
- Testnet chain files (e.g., `97.json`, `80002.json`, `11155111.json`, `11155420.json`, `421614.json`, `2522.json`, `42431.json`, `9745.json`)

### 8. Verification checklist

1. All edited JSON files parse correctly (`jq . file.json`)
2. No trailing newlines (file ends with `}`)
3. New chain's DVN file exists with correct name
4. Self-entry in new file is all zeros
5. `"{newChainId}"` key exists in all 33+ other mainnet DVN files
6. Key is positioned before `"111111111"` in every file
7. DVN addresses in pathway entries match each file's standard addresses
8. L0Config.json `"Proxy"` array has new entry with correct `chainid`, `dvnHorizen`, `dvnL0`
9. `hardhat.config.ts` has new network entry
10. `config/dvn/providers.json` has new chain RPC endpoints
11. No testnet, non-evm.json, or providers.json DVN files were accidentally modified

## How DVN Files Are Consumed

### Foundry (Solidity)

`scripts/DeployFraxOFTProtocol/inherited/SetDVNs.s.sol`:
```solidity
function getDvnStack(uint256 _srcChainId, uint256 _dstChainId) public virtual view returns (DvnStack memory dvnStack) {
    string memory path = string.concat(vm.projectRoot(), "/config/dvn/", _srcChainId.toString(), ".json");
    string memory jsonFile = vm.readFile(path);
    string memory key = string.concat(".", _dstChainId.toString());
    dvnStack = abi.decode(jsonFile.parseRaw(key), (DvnStack));
}
```
Reads `config/dvn/{srcChainId}.json`, looks up key `.{dstChainId}`, decodes into `DvnStack` struct with 6 address fields.

### TypeScript (LayerZero configs)

`*-solana-layerzero.config.ts` and `*-move.layerzero.config.ts`:
```typescript
const srcDVNConfig = JSON.parse(readFileSync(path.join(__dirname, `../../config/dvn/111111111.json`), "utf8"));
const dstDVNConfig = JSON.parse(readFileSync(path.join(__dirname, `../../config/dvn/${_chainid}.json`), "utf8"));
```
Uses `dvnKeys = ['bcwGroup', 'frax', 'horizen', 'lz', 'nethermind', 'stargate']` to read addresses.

## Python Helper Script

Use this pattern for bulk operations (preserves key order, 4-space indent, no trailing newline):

```python
import json
from collections import OrderedDict

def read_dvn(path):
    with open(path) as f:
        return json.loads(f.read(), object_pairs_hook=OrderedDict)

def write_dvn(path, data):
    with open(path, 'w') as f:
        json.dump(data, f, indent=4)
        # No trailing newline

def insert_before_key(data, new_key, new_value, before_key="111111111"):
    result = OrderedDict()
    for k, v in data.items():
        if k == before_key:
            result[new_key] = new_value
        result[k] = v
    return result
```
