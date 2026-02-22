# Sourcify-Compatible Contract Verification

## Overview

This skill covers contract verification against any **Sourcify-compatible** verification API. These APIs do **not** support Etherscan-style verification.

## API Details

- **Verification submit**: `POST /v2/verify/{chainId}/{address}`
- **Status polling**: `GET /v2/verify/{verificationId}`
- **Contract lookup**: `GET /v2/contract/{chainId}/{address}`

### Endpoints NOT implemented on some Sourcify instances

These may return 404 or "not implemented" depending on the instance:

- `/verify` (legacy Sourcify)
- `/verify/solc-json`
- `/session/*` (all session-based endpoints)
- `/v2/verify/metadata/{chainId}/{address}`
- `/v2/verify/similarity/{chainId}/{address}`

## Verification Methods

### Method 1: Forge Directly (Preferred)

`forge verify-contract` with `--verifier sourcify` hits the v2 API directly:

```bash
forge verify-contract <ADDRESS> <PATH>:<CONTRACT_NAME> \
  --chain-id <CHAIN_ID> \
  --verifier sourcify \
  --verifier-url "<VERIFIER_URL>" \
  --constructor-args "<ABI_ENCODED_ARGS>" \
  --watch
```

For contracts in `node_modules`, use the full path:

```bash
forge verify-contract <ADDRESS> \
  node_modules/@fraxfinance/layerzero-v2-upgradeable/messagelib/contracts/upgradeable/proxy/TransparentUpgradeableProxy.sol:TransparentUpgradeableProxy \
  --chain-id <CHAIN_ID> \
  --verifier sourcify \
  --verifier-url "<VERIFIER_URL>" \
  --constructor-args "<ABI_ENCODED_ARGS>" \
  --watch
```

**Note**: `--watch` has a limited retry count (8 retries × 15s = 2 min). If server-side compilation takes longer, the command exits but the job continues. Poll manually if needed (see "Polling for Results" below).

### Method 2: Manual cURL with Standard JSON Input

Use this when `forge verify-contract` fails or when you need more control over the payload.

#### Generate Standard JSON Input

```bash
forge verify-contract <ADDRESS> <PATH>:<CONTRACT_NAME> \
  --show-standard-json-input 2>/dev/null > /tmp/standard_json.json
```

#### Build & Submit Payload

Use `jq --slurpfile` to avoid "Argument list too long" errors with large standard JSON files:

```bash
jq -n --slurpfile stdJson /tmp/standard_json.json \
  --arg compilerVersion "0.8.22+commit.4fc1097e" \
  --arg contractIdentifier "<SOURCE_PATH>:<CONTRACT_NAME>" \
  '{
    stdJsonInput: $stdJson[0],
    compilerVersion: $compilerVersion,
    contractIdentifier: $contractIdentifier
  }' > /tmp/verify_payload.json

curl -s -X POST "<VERIFIER_URL>/v2/verify/<CHAIN_ID>/<ADDRESS>" \
  -H "Content-Type: application/json" \
  -d @/tmp/verify_payload.json
```

Optional payload fields:
- `creationTransactionHash` — hash of the deployment tx
- `constructorArguments` — ABI-encoded constructor args (hex, **without** `0x` prefix)

Returns: `{"verificationId": "<UUID>"}`

## Polling for Results

```bash
curl -s "<VERIFIER_URL>/v2/verify/<VERIFICATION_ID>" | jq .
```

Successful response:

```json
{
  "isJobCompleted": true,
  "verificationId": "...",
  "contract": {
    "match": "exact_match",
    "creationMatch": "match",
    "runtimeMatch": "exact_match",
    "chainId": "<CHAIN_ID>",
    "address": "0x...",
    "verifiedAt": "...",
    "name": "ContractName"
  }
}
```

## Checking Existing Verification

```bash
curl -s "<VERIFIER_URL>/v2/contract/<CHAIN_ID>/<ADDRESS>" | jq .
```

## Critical Gotchas

### Compiler Version Format (Manual Method Only)

- **Correct**: `0.8.22+commit.4fc1097e` (no `v` prefix)
- **Wrong**: `v0.8.22+commit.4fc1097e` → error: `"Unsupported compilerVersion"`

### Contract Identifier (Manual Method Only)

The v2 API uses `contractIdentifier` (fully qualified `path:name`), **not** `contractName`.

### API Deduplication by Address

The API returns the **same `verificationId`** for repeated submissions to the same address. If a previous job is stuck/failed, new submissions get the old stuck ID — there is no way to force a fresh job from the client side.

### Large Contract Compilation Timeout

Contracts with many source files (e.g., 72 sources / ~320KB standard JSON) may cause the server to time out during compilation, resulting in:

```json
{
  "isJobCompleted": true,
  "error": {
    "customCode": "internal_error",
    "message": "Job timed out - container connection hung"
  }
}
```

Or the job may get permanently stuck with `"isJobCompleted": false`.

**Workaround**: Contact the chain's verification team to clear the stuck job or increase server compilation timeout. There is currently no client-side workaround when the metadata and similarity endpoints are not implemented.

### Constructor Arguments Encoding

Use `cast abi-encode` to get the constructor args, then strip the `0x` prefix:

```bash
# Single address arg
cast abi-encode "constructor(address)" 0x20Bb7C2E2f4e5ca2B4c57060d1aE2615245dCc9C | sed 's/^0x//'

# Multiple args (e.g., TransparentUpgradeableProxy)
cast abi-encode "constructor(address,address,bytes)" <IMPL> <ADMIN> 0x | sed 's/^0x//'
```

Constructor args can also be extracted from the broadcast JSON: `jq '.transactions[N].arguments'`.

## Deployment Broadcast Reference

Deployment data is in `broadcast/<SCRIPT>.s.sol/<CHAIN_ID>/run-latest.json`:

- `.transactions[].contractName` — contract name
- `.transactions[].contractAddress` — deployed address
- `.transactions[].hash` — creation tx hash
- `.transactions[].arguments` — constructor arguments (human-readable)
- `.transactions[].transaction.input` — full tx input (bytecode + encoded constructor args)
