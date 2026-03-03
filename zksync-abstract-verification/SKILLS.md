# Contract Verification on zkSync ERA and Abstract

## Overview

zkSync ERA and Abstract (both zkSync-stack chains) support both zkEVM bytecode and standard EVM bytecode. The bytecode type depends on **how the contract was deployed**, not which chain it's on. This distinction is critical for verification — each type requires different tools, API parameters, and explorers.

---

## Key Concept: EVM vs zkEVM Bytecode

| Property | EVM Bytecode | zkEVM Bytecode |
|----------|-------------|----------------|
| Deployed via | Arachnid CREATE2 factory (`0x4e59b448...`) or direct EVM tx replay | `forge script --zksync --broadcast`, `forge create --zksync`, zkSync system contracts |
| Bytecode prefix | `0x6080604052` (PUSH1 80 PUSH1 40 MSTORE) | `0x0001` or `0x0003` (zkEVM-specific structure) |
| Compiler | Standard `solc` only | `zksolc` wrapping `solc` |
| ERA explorer payload | `compilerSolcVersion` only, **no** `compilerZksolcVersion` | Both `compilerSolcVersion` AND `compilerZksolcVersion` |
| Abscan verification | `forge verify-contract` (no `--zksync`) | `forge verify-contract --zksync` |

**Check bytecode type:**
```bash
cast code <address> --rpc-url <rpc> | head -c 12
# 0x6080604052 = EVM bytecode
# 0x0001000000 or 0x0003000000 = zkEVM bytecode
```

**If you submit EVM bytecode with a `zk_compiler_version`, the ERA explorer rejects with: `"zk compiler version specified for EVM bytecode"`.**

---

## Verification by Explorer

### 1. zkSync ERA Explorer (explorer.zksync.io)

**Works for both EVM and zkEVM bytecode.** This is the most reliable verifier for zkSync.

**API endpoint:** `https://zksync2-mainnet-explorer.zksync.io/contract_verification`

#### For EVM bytecode (Arachnid-deployed):

```bash
# 1. Generate standard JSON (no --zksync flag)
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true FOUNDRY_BYTECODE_HASH=none FOUNDRY_CBOR_METADATA=false \
forge verify-contract --show-standard-json-input \
  <address> path/to/Contract.sol:ContractName > /tmp/std_json.json

# 2. Build payload (NO compilerZksolcVersion)
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

# 3. Submit
curl -s -X POST "https://zksync2-mainnet-explorer.zksync.io/contract_verification" \
  -H "Content-Type: application/json" -d @/tmp/payload.json
# Returns numeric ID, e.g. 105504

# 4. Poll (wait ~15s)
curl -s "https://zksync2-mainnet-explorer.zksync.io/contract_verification/105504"
# {"status":"successful"}
```

#### For zkEVM bytecode (forge --zksync deployed):

```bash
# 1. Generate standard JSON (WITH --zksync flag)
forge verify-contract --show-standard-json-input --zksync \
  <address> path/to/Contract.sol:ContractName > /tmp/std_json_zk.json

# 2. Build payload (WITH compilerZksolcVersion + constructorArguments)
python3 << 'EOF'
import json
with open('/tmp/std_json_zk.json') as f:
    std_json = json.load(f)
payload = {
    'contractAddress': '<address>',
    'sourceCode': std_json,
    'codeFormat': 'solidity-standard-json-input',
    'contractName': 'path/to/Contract.sol:ContractName',
    'compilerSolcVersion': '0.8.23',
    'compilerZksolcVersion': 'v1.5.11',
    'optimizationUsed': True,
    'constructorArguments': '0x<abi-encoded-args>'
}
with open('/tmp/payload.json', 'w') as f:
    json.dump(payload, f)
EOF

# 3. Submit + poll same as above
```

**Key differences for zkEVM:**
- Use `--zksync` when generating standard JSON (adds `codegen`, `enableEraVMExtensions`, `forceEVMLA` settings)
- Include `compilerZksolcVersion` in payload (e.g., `"v1.5.11"`)
- Include `constructorArguments` as hex string
- The `FOUNDRY_*` env vars for optimizer/evmVersion are **not needed** — the `--zksync` flag uses the correct settings
- Find zksolc version: `zksolc --version` or check `cache/zksync-solidity-files-cache.json`

### 2. Abscan (abscan.org) — Abstract's Etherscan-based Explorer

**Works for both EVM and zkEVM bytecode.** Uses the Etherscan V2 API.

```bash
# EVM bytecode — standard forge verify (no --zksync)
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true FOUNDRY_BYTECODE_HASH=none FOUNDRY_CBOR_METADATA=false \
forge verify-contract --chain-id 2741 --verifier etherscan \
  --verifier-url "https://api.etherscan.io/v2/api?chainid=2741" \
  --etherscan-api-key <KEY> \
  <address> path/to/Contract.sol:ContractName

# zkEVM bytecode — add --zksync and --constructor-args
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true FOUNDRY_BYTECODE_HASH=none FOUNDRY_CBOR_METADATA=false \
forge verify-contract --chain-id 2741 --verifier etherscan \
  --verifier-url "https://api.etherscan.io/v2/api?chainid=2741" \
  --etherscan-api-key <KEY> --zksync \
  --constructor-args <abi-encoded-hex> \
  <address> path/to/Contract.sol:ContractName
```

**Notes:**
- Abscan uses the **Etherscan V2 API** — the old `api.abscan.org/api` is deprecated
- URL format: `https://api.etherscan.io/v2/api?chainid=2741`
- The `"already verified"` response means success
- `forge verify-contract` with `--verifier etherscan` works out of the box for both bytecode types

### 3. zkSync Blockscout (zksync.blockscout.com)

**Only verifies zkEVM bytecode.** As of March 2026, this Blockscout instance is configured exclusively for zksolc-compiled contracts.

**Evidence:** All verified contracts on this instance have a `zk_compiler_version`. The verification config (`/api/v2/smart-contracts/verification/config`) lists `zk_compiler_versions` and `zk_optimization_modes`. EVM bytecode contracts will always fail with `"Fail - Unable to verify"`.

**For zkEVM bytecode only:**

The `forge verify-contract --verifier blockscout` command doesn't support passing `zk_compiler_version`. Use the v2 API directly:

```bash
curl -s -X POST \
  "https://zksync.blockscout.com/api/v2/smart-contracts/<address>/verification/via/standard-input" \
  -F "compiler_version=v0.8.23+commit.f704f362" \
  -F "zk_compiler_version=v1.5.11" \
  -F "contract_name=ContractName" \
  -F "files[0]=@/tmp/std_json_zk.json;type=application/json"
```

**Do NOT attempt EVM bytecode verification on zkSync Blockscout — it will silently fail.**

---

## `forge verify-contract` Gotchas

### Environment variables are REQUIRED for correct settings

`forge verify-contract` pulls compilation settings from `foundry.toml`'s **default profile**, NOT the `[profile.deploy]` that was used to compile the deployed contract. You MUST override via env vars:

```bash
FOUNDRY_EVM_VERSION=shanghai \
FOUNDRY_OPTIMIZER=true \
FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true \
FOUNDRY_BYTECODE_HASH=none \
FOUNDRY_CBOR_METADATA=false \
forge verify-contract ...
```

Without these, forge generates a standard JSON with the wrong settings (e.g., `paris`, `optimizer=false`, `via_ir=false`), the verifier compiles differently, and verification fails.

### `--show-standard-json-input` respects env vars

The `--show-standard-json-input` flag generates JSON that reflects the env vars / foundry.toml settings. Always set the correct env vars when using it.

### The `--zksync` flag and `forge verify-contract`

- For **ERA explorer** (etherscan-compatible API): `forge verify-contract --zksync` fails with `"Failed to deserialize the request"` — the ERA API can't parse the zksolc-specific JSON format that forge sends. Use curl with manual payload instead.
- For **Abscan** (Etherscan V2): `forge verify-contract --zksync` works correctly.
- For **Blockscout**: `forge verify-contract --verifier blockscout --zksync` submits but always fails because it doesn't pass `zk_compiler_version`.

### `appendCBOR: false` in metadata

Forge adds `"appendCBOR": false` to `settings.metadata` when `cbor_metadata = false` in foundry.toml. This IS a valid solc 0.8.23 option (added in 0.8.18) and does not cause verification issues with standard solc. However, Blockscout's verifier may handle it differently.

---

## Matching Compilation Settings

The most common reason verification fails is a **bytecode mismatch**.

### Quick check: does forge output match on-chain?

```bash
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true FOUNDRY_BYTECODE_HASH=none FOUNDRY_CBOR_METADATA=false \
forge build

python3 << 'PYEOF'
import json, subprocess
with open('out/Contract.sol/Contract.json') as f:
    compiled = json.load(f)['deployedBytecode']['object'][2:]
r = subprocess.run(['cast', 'code', '--rpc-url', '<rpc>', '<address>'],
                   capture_output=True, text=True, timeout=15)
onchain = r.stdout.strip()[2:]
print(f'Compiled: {len(compiled)} chars, On-chain: {len(onchain)} chars')
print(f'Match: {compiled == onchain}')
PYEOF
```

### Identifying the correct settings

| Clue | What it tells you |
|------|-------------------|
| Bytecode contains `5f` opcode | EVM version is `shanghai` or later (not `paris`) |
| `foundry.toml` has `[profile.deploy]` | Check both default and deploy profiles |
| Deployed via `forge script --zksync` | zkEVM bytecode, needs zksolc |
| Deployed via Arachnid factory | EVM bytecode, standard solc only |

### Common deploy profile (hop-v2):
```toml
[profile.deploy]
optimizer = true
optimizer_runs = 1_000_000
via_ir = true
# Also from [profile.default]:
evm_version = "paris"       # BUT actual bytecodes use shanghai (PUSH0)
bytecode_hash = "none"
cbor_metadata = false
```

**Important:** The foundry.toml says `evm_version = "paris"` in the default profile, but the actual deployed bytecodes use PUSH0 (`5f`) which means `shanghai`. Always verify by comparing bytecodes, not by trusting the config file alone.

---

## Generating Standard JSON Input

```bash
# For EVM bytecode (no --zksync):
FOUNDRY_EVM_VERSION=shanghai FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000 \
FOUNDRY_VIA_IR=true FOUNDRY_BYTECODE_HASH=none FOUNDRY_CBOR_METADATA=false \
forge verify-contract --show-standard-json-input \
  <address> path/to/Contract.sol:ContractName > /tmp/std_json.json

# For zkEVM bytecode (with --zksync):
forge verify-contract --show-standard-json-input --zksync \
  <address> path/to/Contract.sol:ContractName > /tmp/std_json_zk.json
```

**Verify the generated settings:**
```bash
python3 -c "
import json
d = json.load(open('/tmp/std_json.json'))
s = d.get('settings', {})
print('optimizer:', s.get('optimizer', {}))
print('evmVersion:', s.get('evmVersion', ''))
print('viaIR:', s.get('viaIR', ''))
print('metadata:', s.get('metadata', {}))
# zkEVM-specific fields (only present with --zksync):
for k in ['codegen', 'enableEraVMExtensions', 'forceEVMLA']:
    if k in s: print(f'{k}: {s[k]}')
"
```

---

## Proxy Verification

For `FraxUpgradeableProxy` or similar proxies:
- Constructor args: `(address _implementation, address _admin, bytes _data)`
- ERA explorer API handles constructor args automatically from standard JSON — no separate parameter needed
- Abscan (Etherscan V2) requires `--constructor-args <hex>` flag
- Use the same compilation settings as the implementation

```bash
# Encode proxy constructor args
cast abi-encode "constructor(address,address,bytes)" \
  <impl_address> <admin_address> "0x"
```

---

## zksolc Versions

- `zksolc --version` on the system PATH may differ from what foundry-zksync actually uses
- Foundry-zksync stores its zksolc at `~/.zksync/zksolc-*` (e.g., `zksolc-linux-arm64-musl-v1.5.15`)
- The cache file `cache/zksync-solidity-files-cache.json` records the actual `zksolcVersion` used
- For ERA explorer payload: use format `"v1.5.11"` (with `v` prefix)
- **None of this matters for EVM bytecode** — only relevant for zkEVM bytecode verification

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `"Deployed bytecode is not equal to generated one"` | Wrong compilation settings | Check env vars — especially `FOUNDRY_EVM_VERSION=shanghai` |
| `"zk compiler version specified for EVM bytecode"` | Submitted zksolc version for EVM contract | Remove `compilerZksolcVersion` from payload |
| `"Failed to deserialize the request"` on ERA | `forge verify-contract --zksync` with ERA API | Use curl with manual JSON payload instead |
| `"Fail - Unable to verify"` on zkSync Blockscout | EVM bytecode on zkEVM-only verifier | This Blockscout only verifies zkEVM bytecode; skip it |
| `"already verified"` on Abscan | Contract was previously verified | This is success, not an error |
| `"Bad request"` from Blockscout v2 | Address locked from failed attempt | Wait and retry, or use ERA explorer API |
| Blockscout `is_verified: None` | Submitted OK but silently failed | Use ERA explorer API or Abscan instead |
| Wrong optimizer settings in standard JSON | `foundry.toml` default profile used | Set `FOUNDRY_OPTIMIZER=true FOUNDRY_OPTIMIZER_RUNS=1000000` etc. |
| `contractName` format wrong | ERA explorer expects full path | Use `src/contracts/path/File.sol:ContractName` format |

---

## Reference: Verified Contracts (hop-v2)

### zkSync ERA (chain 324)

| Contract | Address | Bytecode Type | Deployer | Verification |
|----------|---------|--------------|----------|--------------|
| RemoteHopV2 (impl) | `0x0000000087ED0dD8b999aE6C7c30f95e9707a3C6` | EVM | Arachnid CREATE2 | ERA explorer ID 105504 |
| FraxUpgradeableProxy | `0x0000006D38568b00B457580b734e0076C62de659` | EVM | Arachnid CREATE2 | ERA explorer ID 105505 |
| RemoteAdmin | `0x000000000E0E120FCAc7b4d98e9E35E1DE6fdadb` | zkEVM | forge --zksync (L2 Create2Factory) | ERA explorer ID 105506 |

**Settings for EVM contracts:** solc 0.8.23, shanghai, optimizer=true, runs=1M, viaIR=true, bytecodeHash=none, cborMetadata=false
**Settings for zkEVM contract:** solc 0.8.23, zksolc v1.5.11

### Abstract (chain 2741)

| Contract | Address | Bytecode Type | Deployer | Verification |
|----------|---------|--------------|----------|--------------|
| RemoteHopV2 (impl) | `0x0000000087ED0dD8b999aE6C7c30f95e9707a3C6` | EVM | Arachnid CREATE2 | Abscan (Etherscan V2) |
| FraxUpgradeableProxy | `0x0000006D38568b00B457580b734e0076C62de659` | EVM | Arachnid CREATE2 | Abscan (Etherscan V2) |
| RemoteAdmin | `0x000000000E0E120FCAc7b4d98e9E35E1DE6fdadb` | zkEVM | forge --zksync (L2 Create2Factory) | Abscan (Etherscan V2) |

**Same contract addresses on both chains** (same bytecode, same CREATE2 salts).

### Constructor Arguments

| Contract | Constructor Args (ABI-encoded hex) |
|----------|-----------------------------------|
| FraxUpgradeableProxy | `0x0000000000000000000000000000000087ed0dd8b999ae6c7c30f95e9707a3c600000000000000000000000054f9b12743a7deec0ea48721683cbebedc6e17bc00000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000000` |
| RemoteAdmin | `0x000000000000000000000000ea77c590bb36c43ef7139ce649cfbcfd6163170d0000000000000000000000000000006d38568b00b457580b734e0076c62de6590000000000000000000000005f25218ed9474b721d6a38c115107428e832fa2e` |

### Key Addresses

| Role | Address |
|------|---------|
| GCP KMS deployer | `0x54f9b12743a7deec0ea48721683cbebedc6e17bc` |
| Arachnid CREATE2 factory | `0x4e59b44847b379578588920cA78FbF26c0B4956C` |
| zkSync L2 Create2Factory | `0x0000000000000000000000000000000000010000` |
| Etherscan API key (Abscan) | `98K9P43YDT6K882QVCDS8UDGJ62794NYPN` |
