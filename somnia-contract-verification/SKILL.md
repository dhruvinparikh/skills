# Somnia Smart Contract Verification

## Overview

This skill covers smart contract verification on **Somnia mainnet** using the explorer API at:

```text
https://explorer.somnia.network/api/
```

Somnia's explorer is **Etherscan-style**, not Sourcify-style. In practice, Foundry's built-in verifier modes may fail against it even when the explorer itself supports verification.

No API key was required in the successful Somnia verification flow used here.

## When to Use

- Verifying Solidity contracts deployed on Somnia mainnet
- Verifying contracts from `frax-oft-upgradeable` broadcasts on chain ID `5031`
- Verifying upgradeable proxies and implementations on Somnia
- Recovering from failed `forge script --verify` or `forge verify-contract` runs on Somnia

## Key Chain Details

| Property | Value |
|----------|-------|
| Chain ID | `5031` |
| Explorer | `https://explorer.somnia.network` |
| API base | `https://explorer.somnia.network/api/` |

## Primary Gotcha

Foundry verifier flows such as:

```bash
forge verify-contract ... --verifier blockscout --verifier-url https://explorer.somnia.network/api
```

may fail with:

```text
Params 'module' and 'action' are required parameters
```

This means the explorer expects a direct **Etherscan-style** request with explicit `module=contract` and `action=verifysourcecode` parameters.

Also note:

- Use the **trailing-slash API URL**: `https://explorer.somnia.network/api/`
- `https://explorer.somnia.network/api` redirects and may confuse automated verifier flows

## Verification Workflow

### 1. Generate standard JSON input with Forge

Use the fully qualified contract identifier:

```bash
forge verify-contract <ADDRESS> <SOURCE_PATH>:<CONTRACT_NAME> \
  --chain-id 5031 \
  --show-standard-json-input > /tmp/standard_json.json
```

Example:

```bash
forge verify-contract 0xa1fec5eab3a09fd12b4fddb2a81f62915a8fa8f0 \
  ./contracts/mocks/ImplementationMock.sol:ImplementationMock \
  --chain-id 5031 \
  --show-standard-json-input > /tmp/standard_json.json
```

For contracts in `node_modules`, use the full source path.

### 2. Submit verification manually to Somnia explorer

#### Contracts without constructor args

```bash
curl -sS -X POST 'https://explorer.somnia.network/api/' \
  --data-urlencode module=contract \
  --data-urlencode action=verifysourcecode \
  --data-urlencode contractaddress=<ADDRESS> \
  --data-urlencode sourceCode@/tmp/standard_json.json \
  --data-urlencode codeformat=solidity-standard-json-input \
  --data-urlencode contractname='<SOURCE_PATH>:<CONTRACT_NAME>' \
  --data-urlencode compilerversion='v0.8.22+commit.4fc1097e'
```

#### Contracts with constructor args

```bash
ARGS=$(cast abi-encode 'constructor(address)' 0x1234...)
ARGS=${ARGS#0x}

curl -sS -X POST 'https://explorer.somnia.network/api/' \
  --data-urlencode module=contract \
  --data-urlencode action=verifysourcecode \
  --data-urlencode contractaddress=<ADDRESS> \
  --data-urlencode sourceCode@/tmp/standard_json.json \
  --data-urlencode codeformat=solidity-standard-json-input \
  --data-urlencode contractname='<SOURCE_PATH>:<CONTRACT_NAME>' \
  --data-urlencode compilerversion='v0.8.22+commit.4fc1097e' \
  --data-urlencode constructorArguments="$ARGS"
```

Expected response:

```json
{"message":"OK","result":"<GUID>","status":"1"}
```

The `result` field is the verification GUID.

## Polling for Status

```bash
curl -sS 'https://explorer.somnia.network/api/?module=contract&action=checkverifystatus&guid=<GUID>'
```

Successful response:

```json
{"message":"OK","result":"Pass - Verified","status":"1"}
```

Common intermediate response:

```json
{"message":"OK","result":"Pending in queue","status":"1"}
```

## Checking Existing Verification

To probe whether a contract is already verified:

```bash
curl -sS 'https://explorer.somnia.network/api/?module=contract&action=getabi&address=<ADDRESS>'
```

If not verified yet, the explorer returns:

```json
{"message":"Contract source code not verified","result":null,"status":"0"}
```

## Constructor Args

Use `cast abi-encode`, then strip the `0x` prefix:

```bash
cast abi-encode 'constructor(address)' 0x6F475642a6e85809B1c36Fa62763669b1b48DD5B | sed 's/^0x//'
```

Proxy example:

```bash
cast abi-encode 'constructor(address,address,bytes)' <IMPLEMENTATION> <ADMIN> 0x | sed 's/^0x//'
```

## Compiler Version Format

Somnia's explorer expects the Etherscan-style compiler version string with a leading `v`:

```text
v0.8.22+commit.4fc1097e
```

Do not switch this to the Sourcify-style form without the `v` prefix.

## Submission Compatibility Note

If constructor-arg submissions behave inconsistently, it is safe to send both:

- `constructorArguments`
- `constructorArguements`

The second spelling is the legacy typo seen on some Etherscan-style explorers. Sending both is harmless and can help with explorer compatibility.

## Upgradeable Proxy Gotcha

For `TransparentUpgradeableProxy`, verify against the **creation-time constructor args**, not the implementation address after later upgrades.

In `broadcast/<SCRIPT>.s.sol/5031/run-latest.json`, use:

- `.transactions[].contractName`
- `.transactions[].contractAddress`
- `.transactions[].arguments`

If the deployment script created the proxy with an initial placeholder implementation and upgraded it later, the constructor args must still reference that **initial** implementation.

## Practical Somnia Pattern

For Somnia verification in this workspace, the reliable pattern is:

1. Run the deployment or use the existing broadcast artifact
2. Extract deployed addresses and constructor args from `broadcast/.../run-latest.json`
3. Generate standard JSON input with `forge verify-contract --show-standard-json-input`
4. Submit via `curl` to `https://explorer.somnia.network/api/`
5. Poll `checkverifystatus` until `Pass - Verified`

## When Forge Direct Verification Is Safe to Skip

If you already know Somnia's explorer is reachable and you're on chain `5031`, it is usually faster to skip `forge verify-contract --watch` entirely and go straight to:

1. `forge verify-contract --show-standard-json-input`
2. manual `curl` submission
3. manual polling

This avoids false failures caused by Foundry's verifier integration rather than by the contract metadata itself.