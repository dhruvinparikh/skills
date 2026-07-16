# OKLink Contract Verification (xLayer / OKX chains)

## Overview

Verify Solidity contract source code on **OKLink** explorer (xLayer, BSC, Polygon,
Arbitrum, etc.). OKLink uses its own API — **not** Etherscan, **not** Sourcify. The
API key requirement is loosely enforced; the v5 endpoint accepts submissions with a
dummy `Ok-Access-Key` header.

## When to Use

- Verifying contracts deployed on **xLayer** (chain 196) or other OKLink-supported chains
- `forge verify-contract --verifier oklink` fails or the `OKLINK_API_KEY` env var is unset
- Source code needs to appear on the OKLink explorer contract page
- Recovering from a `--broadcast --verify` run where verification silently failed

## Key Chain Details

| Property | xLayer value |
|----------|-------------|
| Chain ID | `196` |
| chainShortName | `XLAYER` |
| Explorer | `https://www.oklink.com/x-layer/evm` |
| Verify URL | `https://www.oklink.com/api/v5/explorer/contract/verify-source-code` |
| Check URL | `https://www.oklink.com/api/v5/explorer/contract/check-verify-result` |
| Contract info URL | `https://www.oklink.com/api/v5/explorer/contract/verify-contract-info` |

**Supported chains** (chainShortName): ETH, XLAYER, XLAYER_TESTNET, BSC, POLYGON,
AVAXC, FTM, OP, ARBITRUM, LINEA, MANTA, CANTO, BASE, SCROLL, OPBNB, POLYGON_ZKEVM,
SEPOLIA_TESTNET, GOERLI_TESTNET, AMOY_TESTNET, MUMBAI_TESTNET, POLYGON_ZKEVM_TESTNET

## API Key

The docs say "Apply for OKLink API key" but the link loops back to the docs page with
no signup flow visible. In practice, the v5 API accepts a **dummy key** (`"test"`)
for contract verification submissions. Pass it as header `Ok-Access-Key: test`.

## Critical: Use the Correct Compiler Settings

The most common cause of verification failure is a mismatch between the compiler
settings in the standard JSON input and the settings used for the actual deployment.

### How to determine the correct settings

1. **Check the on-chain bytecode hash**:
   ```bash
   cast code <ADDRESS> --rpc-url <RPC> | cast keccak
   ```

2. **Check the compiled artifact's bytecode hash**:
   ```bash
   jq -r '.deployedBytecode.object' out/<Contract>.sol/<Contract>.json | cast keccak
   ```

3. **If they match**, read the compiler settings from the artifact's metadata:
   ```bash
   jq -r '.metadata' out/<Contract>.sol/<Contract>.json | \
     python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps({'compiler': d.get('compiler'), 'settings': {k: d['settings'][k] for k in ['optimizer','viaIR','evmVersion','metadata']}}, indent=2))"
   ```

4. **If they don't match**, rebuild with the correct profile:
   ```bash
   FOUNDRY_PROFILE=<profile> forge build --build-info --force
   ```
   Then check each `out/build-info/*.json` for a matching bytecode hash.

### Gotcha: `forge verify-contract --show-standard-json-input` ignores profiles

`forge verify-contract --show-standard-json-input` always uses the **default profile**,
not `FOUNDRY_PROFILE`. If the deployment used a different profile (e.g. `[profile.deploy]`
with `optimizer_runs = 1000000`), the generated standard JSON will have the wrong
optimizer runs. You must either:

- Patch the JSON manually: `jq '.settings.optimizer.runs = <correct_value>'`
- Or extract the standard JSON from the matching `build-info` file (see below)

### Gotcha: `build-info` files may not match the `out/` artifact

After multiple builds with different profiles, the `out/<Contract>.json` artifact may
match the on-chain bytecode while none of the `out/build-info/*.json` files do (the
artifact is a cumulative cache; build-info files are per-build). Always verify by
comparing bytecode hashes, not by assuming the newest build-info is correct.

## Procedure

### 1. Generate Standard JSON Input

```bash
# Generate via forge (uses default profile — verify settings match!)
forge verify-contract <ADDRESS> src/contracts/<Contract>.sol:<Contract> \
  --chain-id <CHAIN_ID> \
  --show-standard-json-input 2>/dev/null > /tmp/stdjson.json

# Check settings
jq '.settings | {optimizer, viaIR, evmVersion, metadata}' /tmp/stdjson.json
```

### 2. Verify Settings Match the Deployment

Compare the settings against the artifact metadata (see "Critical" section above).
If optimizer runs differ, patch:

```bash
jq '.settings.optimizer.runs = <correct_runs>' /tmp/stdjson.json > /tmp/stdjson_fixed.json
```

### 3. Build the API Payload

```python
import json

stdjson = json.load(open('/tmp/stdjson.json'))  # or stdjson_fixed.json
payload = {
    'chainShortName': '<CHAIN_SHORT_NAME>',  # e.g. XLAYER
    'contractAddress': '<ADDRESS>',
    'contractName': '<ContractName>',
    'sourceCode': json.dumps(stdjson),
    'codeFormat': 'solidity-standard-json-input',
    'compilerVersion': 'v0.8.23+commit.f704f362',  # from artifact metadata
    'optimization': '1',
    'optimizationRuns': '200',  # must match actual deployment
    'evmVersion': 'shanghai',   # must match
    'viaIr': True                # must match
}
with open('/tmp/oklink_payload.json', 'w') as f:
    json.dump(payload, f)
```

### 4. Submit Verification

```bash
curl -s -X POST "https://www.oklink.com/api/v5/explorer/contract/verify-source-code" \
  -H "Content-Type: application/json" \
  -H "Ok-Access-Key: test" \
  -d @/tmp/oklink_payload.json | python3 -m json.tool
```

Response:
```json
{
    "code": "0",
    "msg": "",
    "data": ["<GUID>"]
}
```

### 5. Poll for Result

Wait 10–60 seconds, then check:

```bash
curl -s -X POST "https://www.oklink.com/api/v5/explorer/contract/check-verify-result" \
  -H "Content-Type: application/json" \
  -H "Ok-Access-Key: test" \
  -d '{"chainShortName":"<CHAIN_SHORT_NAME>","guid":"<GUID>"}' | python3 -m json.tool
```

Response:
```json
{"code": "0", "msg": "", "data": ["Success"]}    # or "Fail" / "Pending"
```

### 6. Confirm via API

```bash
curl -s "https://www.oklink.com/api/v5/explorer/contract/verify-contract-info?chainShortName=<CHAIN_SHORT_NAME>&contractAddress=<ADDRESS>" \
  -H "Ok-Access-Key: test" | python3 -m json.tool
```

If verified, the response includes `contractName`, `compilerVersion`, and `sourceCode`.

## Verifying Proxy Contracts

After the implementation is verified and the proxy upgrade is executed:

```bash
curl -s -X POST "https://www.oklink.com/api/v5/explorer/contract/verify-proxy-contract" \
  -H "Content-Type: application/json" \
  -H "Ok-Access-Key: test" \
  -d '{
    "chainShortName": "XLAYER",
    "proxyContractAddress": "<PROXY_ADDRESS>",
    "expectedImplementation": "<IMPL_ADDRESS>"
  }'
```

Poll with the returned GUID:

```bash
curl -s -X POST "https://www.oklink.com/api/v5/explorer/contract/check-proxy-verify-result" \
  -H "Content-Type: application/json" \
  -H "Ok-Access-Key: test" \
  -d '{"chainShortName":"XLAYER","guid":"<GUID>"}'
```

## Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| `forge verify-contract --verifier oklink` fails with "Failed to submit" | Missing `OKLINK_API_KEY` or wrong API URL | Use the v5 API directly via curl with `Ok-Access-Key: test` |
| Verification returns "Fail" | Compiler settings mismatch (optimizer runs, evmVersion, viaIR, or solc version) | Read settings from the artifact's `.metadata` field and match exactly |
| `--show-standard-json-input` uses wrong profile | `FOUNDRY_PROFILE` is ignored by `verify-contract` | Patch the JSON with `jq` or extract from the matching `build-info` file |
| Page still shows "unverified" after API says "Success" | OKLink page cache delay | Wait 1–2 minutes and reload; the API confirms verification immediately |
| `verify-source-code-plugin/XLAYER` endpoint returns `{"status":"0","message":"NOTOK"}` | Wrong API path (plugin endpoint) | Use the v5 endpoint: `/api/v5/explorer/contract/verify-source-code` (no `-plugin/`) |
| Build-info files don't match the `out/` artifact | Multiple builds with different profiles created stale build-info files | Compare bytecode hashes across all `out/build-info/*.json` to find the match |

## Reference: API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v5/explorer/contract/verify-source-code` | POST | Submit source verification |
| `/api/v5/explorer/contract/check-verify-result` | POST | Poll verification status |
| `/api/v5/explorer/contract/verify-proxy-contract` | POST | Submit proxy verification |
| `/api/v5/explorer/contract/check-proxy-verify-result` | POST | Poll proxy verification |
| `/api/v5/explorer/contract/verify-contract-info` | GET | Query verified contract info |

All endpoints use header `Ok-Access-Key: <key>` (dummy `"test"` works).
