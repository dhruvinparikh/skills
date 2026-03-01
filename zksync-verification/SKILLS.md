# Contract Verification on zkSync ERA

## Overview

zkSync ERA supports both zkEVM bytecode and standard EVM bytecode. Contracts deployed via the Arachnid deterministic CREATE2 deployer (`0x4e59b44847b379578588920cA78FbF26c0B4956C`) produce **standard EVM bytecode** (starts with `0x6080604052`), NOT zkEVM bytecode. This distinction is critical for verification.

---

## Key Discovery: EVM vs zkEVM Bytecode

- Contracts deployed through `forge create --zksync` or zkSync system contracts produce **zkEVM** bytecode and require `zksolc` for verification.
- Contracts deployed via the Arachnid CREATE2 factory (`0x4e59b44847b379578588920cA78FbF26c0B4956C`) using `safeCreate2(bytes32,bytes)` (selector `0x4e59b448`) produce **standard EVM** bytecode.
- The on-chain bytecode type can be checked with `cast code <address> --rpc-url <rpc>`:
  - EVM bytecode starts with `6080604052` (PUSH1 80 PUSH1 40 MSTORE)
  - zkEVM bytecode has a different structure

**If you submit EVM bytecode for verification with a `zk_compiler_version`, the ERA explorer will reject it with: `"zk compiler version specified for EVM bytecode"`.**

---

## Verification APIs

### 1. zkSync ERA Explorer API (Recommended for zkSync)

**Endpoint:** `https://zksync2-mainnet-explorer.zksync.io/contract_verification`

**Submit verification:**
```bash
curl -s -X POST "https://zksync2-mainnet-explorer.zksync.io/contract_verification" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

**Payload format (standard-json-input):**
```json
{
  "contractAddress": "0x...",
  "sourceCode": { "<standard JSON input object>" },
  "codeFormat": "solidity-standard-json-input",
  "contractName": "src/contracts/path/Contract.sol:ContractName",
  "compilerSolcVersion": "0.8.23",
  "optimizationUsed": true
}
```

**Response:** Returns a numeric verification ID (e.g., `105504`).

**Check status:**
```bash
curl -s "https://zksync2-mainnet-explorer.zksync.io/contract_verification/<ID>"
# Returns: {"status":"successful"} or {"status":"failed","error":"..."}
```

**Important:** Do NOT include `compilerZksolcVersion` for EVM bytecode contracts.

### 2. Blockscout v2 API

**Endpoint:** `https://zksync.blockscout.com/api/v2/smart-contracts/<address>/verification/via/standard-input`

```bash
curl -s -X POST "https://zksync.blockscout.com/api/v2/smart-contracts/<address>/verification/via/standard-input" \
  -F "compiler_version=v0.8.23+commit.f704f362" \
  -F "contract_name=ContractName" \
  -F "input=@/tmp/std_json.json;type=application/json"
```

**Caveats:**
- Returns `"Bad request"` (400) if the address is locked from a previous failed attempt.
- Once a verification attempt is submitted, the address may be locked for a period — retry later or use the ERA explorer API instead.
- Do NOT include `zk_compiler_version` for EVM bytecode.

### 3. Blockscout v1 API (block-explorer-api.mainnet.zksync.io)

**Avoid this endpoint.** It accepts submissions silently but verification often doesn't complete. The standard JSON input can be too large (error: `"expected source code data at line 1 column 378455"`).

### 4. `forge verify-contract`

`forge verify-contract --zksync` generates standard JSON with zksolc-specific fields (codegen, enableEraVMExtensions) that may cause issues. For EVM bytecode on zkSync, generate the standard JSON **without** `--zksync` but with the correct compilation settings, then submit manually via the ERA explorer API.

---

## Matching Compilation Settings

The most common reason verification fails is a **bytecode mismatch** — the locally compiled bytecode doesn't match on-chain. You must use the exact same compiler settings that produced the deployed bytecode.

### Step 1: Identify the deployer and factory

```bash
# Find contract creator and creation tx
curl -s "https://block-explorer-api.mainnet.zksync.io/api?module=contract&action=getcontractcreation&contractaddresses=<address>"

# Inspect the creation transaction
cast tx <txHash> --rpc-url https://mainnet.era.zksync.io
```

- If `to` is `0x4e59b44847b379578588920cA78FbF26c0B4956C` → Arachnid CREATE2 factory → EVM bytecode
- The `input` starts with `0x4e59b448` (safeCreate2 selector) followed by salt + init code

### Step 2: Compare bytecodes to find correct settings

```python
import json, subprocess

with open('out/Contract.sol/Contract.json') as f:
    d = json.load(f)
compiled_runtime = d['deployedBytecode']['object'][2:]

result = subprocess.run(['cast', 'code', '<address>', '--rpc-url', 'https://mainnet.era.zksync.io'],
                       capture_output=True, text=True)
onchain_runtime = result.stdout.strip()[2:]

print('Match:', compiled_runtime == onchain_runtime)
```

### Step 3: Try different settings via environment overrides

```bash
# Override foundry.toml settings via environment
FOUNDRY_EVM_VERSION=shanghai \
FOUNDRY_OPTIMIZER=true \
FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true \
forge build

# Then compare bytecodes again
```

### Common settings to try

| Setting | Env Variable | Common Values |
|---------|-------------|---------------|
| EVM version | `FOUNDRY_EVM_VERSION` | `paris`, `shanghai`, `cancun` |
| Optimizer | `FOUNDRY_OPTIMIZER` | `true`, `false` |
| Optimizer runs | `FOUNDRY_OPTIMIZER_RUNS` | `200`, `1000000` |
| Via IR | `FOUNDRY_VIA_IR` | `true`, `false` |

**Hint:** If the on-chain bytecode uses `5f` (PUSH0 opcode), the EVM version is `shanghai` or later, not `paris`.

---

## Generating Standard JSON Input

```bash
# Generate standard JSON with specific settings (no --zksync for EVM bytecode)
FOUNDRY_EVM_VERSION=shanghai \
FOUNDRY_OPTIMIZER=true \
FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true \
forge verify-contract --show-standard-json-input \
  <address> \
  src/contracts/path/Contract.sol:ContractName \
  > /tmp/std_json.json
```

Verify the settings in the generated JSON:
```bash
python3 -c "
import json
d = json.load(open('/tmp/std_json.json'))
s = d.get('settings', {})
print('optimizer:', s.get('optimizer', {}))
print('evmVersion:', s.get('evmVersion', ''))
print('viaIR:', s.get('viaIR', ''))
"
```

---

## End-to-End Verification Workflow

```bash
# 1. Identify on-chain bytecode type
cast code <address> --rpc-url https://mainnet.era.zksync.io | head -c 20
# 0x6080604052 = EVM bytecode

# 2. Find correct compilation settings by comparing bytecodes
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 FOUNDRY_VIA_IR=true forge build
# Compare out/Contract.sol/Contract.json deployedBytecode with on-chain

# 3. Generate standard JSON input
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 FOUNDRY_VIA_IR=true \
  forge verify-contract --show-standard-json-input <address> path/to/Contract.sol:ContractName > /tmp/std_json.json

# 4. Build ERA explorer payload
python3 << 'EOF'
import json
with open('/tmp/std_json.json') as f:
    std_json = json.load(f)
payload = {
    'contractAddress': '<address>',
    'sourceCode': std_json,
    'codeFormat': 'solidity-standard-json-input',
    'contractName': 'path/to/Contract.sol:ContractName',
    'compilerSolcVersion': '0.8.23',
    'optimizationUsed': True
}
with open('/tmp/payload.json', 'w') as f:
    json.dump(payload, f)
EOF

# 5. Submit to ERA explorer
curl -s -X POST "https://zksync2-mainnet-explorer.zksync.io/contract_verification" \
  -H "Content-Type: application/json" \
  -d @/tmp/payload.json
# Returns verification ID

# 6. Poll for result
sleep 15
curl -s "https://zksync2-mainnet-explorer.zksync.io/contract_verification/<ID>"
# {"status":"successful"}
```

---

## Proxy Verification

For `FraxUpgradeableProxy` or similar proxies:
- Constructor args are ABI-encoded at the end of the creation tx input data
- Typical args: `(address _implementation, address _admin, bytes _data)`
- The ERA explorer API handles constructor args automatically from the standard JSON input — no need to supply them separately
- Use the same compilation settings as the implementation

---

## zksolc Version Gotchas

- `zksolc --version` on the system PATH may differ from what foundry-zksync actually uses
- Foundry-zksync stores its zksolc at `~/.zksync/zksolc-*` (e.g., `zksolc-linux-arm64-musl-v1.5.15`)
- The cache file `cache/zksync-solidity-files-cache.json` records the actual `zksolcVersion` used
- **None of this matters for EVM bytecode** — only relevant for zkEVM bytecode verification

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `"Deployed bytecode is not equal to generated one from given source"` | Wrong compilation settings | Try different optimizer/viaIR/evmVersion combinations |
| `"zk compiler version specified for EVM bytecode"` | Submitted zksolc version for EVM contract | Remove `compilerZksolcVersion` from payload |
| `"expected source code data at line 1 column N"` | Standard JSON too large for blockscout v1 | Use ERA explorer API or blockscout v2 instead |
| `"Bad request"` from blockscout v2 | Address locked from previous attempt | Wait and retry, or use ERA explorer API |
| Bytecode uses `5f` but compiled doesn't | EVM version mismatch | Set `FOUNDRY_EVM_VERSION=shanghai` |
| Submitted OK but `is_verified: None` | Blockscout accepted but silently failed | Verify via ERA explorer API instead |
| contractName format wrong | ERA explorer expects full path | Use `src/contracts/path/File.sol:ContractName` format |

---

## Reference: Verified Contracts (hop-v2 on zkSync ERA)

| Contract | Address | Settings | Verification ID |
|----------|---------|----------|-----------------|
| RemoteHopV2 (impl) | `0x0000000087ED0dD8b999aE6C7c30f95e9707a3C6` | solc 0.8.23, shanghai, optimizer=true, runs=1M, viaIR=true | 105504 |
| FraxUpgradeableProxy | `0x0000006D38568b00B457580b734e0076C62de659` | solc 0.8.23, shanghai, optimizer=true, runs=1M, viaIR=true | 105505 |

Deployer: `0x31562ae726AFEBe25417df01bEdC72EF489F45b3` via Arachnid CREATE2 factory (`0x4e59b44847b379578588920cA78FbF26c0B4956C`).
