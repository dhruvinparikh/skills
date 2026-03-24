# Verify LayerZero OFT Setup Calldata in Safe Batch Files

## Overview

Decode and verify every function argument in Gnosis Safe batch JSON files that configure LayerZero V2 OFT cross-chain pathways. Ensures correct contracts are called with correct values — peers, DVNs, libraries, enforced options — without requiring on-chain simulation.

## When to Use

- After generating Safe batch JSON files from Foundry deploy/setup scripts
- Before importing batch files into Safe UI for multisig signing
- Auditing existing batch files for correctness
- Complementing Anvil fork simulation with argument-level verification

## Prerequisites

- Foundry (`cast`, `jq`)
- Safe batch JSON files (standard Gnosis Safe format)
- Knowledge of expected values:
  - OFT/lockbox addresses
  - Remote chain EID
  - DVN addresses for the route
  - Send/receive library addresses
  - Endpoint address

## Standard OFT Setup Transaction Order

Each per-OFT batch file contains **6 transactions** in this order:

| Index | Function | Target | Purpose |
|-------|----------|--------|---------|
| 0 | `setEnforcedOptions` | Lockbox/OFT | Set min gas for SEND + SEND_AND_CALL |
| 1 | `setPeer` | Lockbox/OFT | Register remote chain's OFT as peer |
| 2 | `setConfig` | Endpoint | Configure send-side DVNs |
| 3 | `setConfig` | Endpoint | Configure receive-side DVNs |
| 4 | `setSendLibrary` | Endpoint | Set send library for route |
| 5 | `setReceiveLibrary` | Endpoint | Set receive library for route |

## Function Selectors

| Function | Selector |
|----------|----------|
| `setEnforcedOptions((uint32,uint16,bytes)[])` | `0xb98bd070` |
| `setPeer(uint32,bytes32)` | `0x3400288b` |
| `setConfig(address,address,(uint32,uint32,bytes)[])` | `0x6dbd9f90` |
| `setSendLibrary(address,uint32,address)` | `0x9535ff30` |
| `setReceiveLibrary(address,uint32,address,uint256)` | `0x6a14d715` |

## Verification Procedure

### 1. Verify File Metadata

```bash
for f in "${FILES[@]}"; do
  chainId=$(jq -r '.chainId' "$f")
  txCount=$(jq '.transactions | length' "$f")
  echo "$(basename "$f"): chainId=$chainId txs=$txCount"
done
```

Expect: correct chain ID, exactly 6 transactions per file.

### 2. Verify Target Contracts

```bash
for f in "${FILES[@]}"; do
  for i in 0 1; do  # setEnforcedOptions, setPeer → lockbox
    to=$(jq -r ".transactions[$i].to" "$f" | tr '[:upper:]' '[:lower:]')
    echo "tx$i target=$to (should be lockbox)"
  done
  for i in 2 3 4 5; do  # setConfig, setSendLib, setRecvLib → endpoint
    to=$(jq -r ".transactions[$i].to" "$f" | tr '[:upper:]' '[:lower:]')
    echo "tx$i target=$to (should be endpoint)"
  done
done
```

### 3. Verify Function Selectors

```bash
for f in "${FILES[@]}"; do
  EXPECTED_SELS=("0xb98bd070" "0x3400288b" "0x6dbd9f90" "0x6dbd9f90" "0x9535ff30" "0x6a14d715")
  for i in $(seq 0 5); do
    data=$(jq -r ".transactions[$i].data" "$f")
    sel="${data:0:10}"
    if [ "$sel" != "${EXPECTED_SELS[$i]}" ]; then
      echo "FAIL $(basename "$f") tx$i selector=$sel expected=${EXPECTED_SELS[$i]}"
    fi
  done
done
```

### 4. Decode and Verify setEnforcedOptions (tx0)

```bash
data=$(jq -r '.transactions[0].data' "$f")
cast decode-calldata "setEnforcedOptions((uint32,uint16,bytes)[])" "$data"
```

Verify:
- Two entries: msgType 1 (SEND) and msgType 2 (SEND_AND_CALL)
- EID matches remote chain
- Options bytes match expected gas values

**Common enforced options patterns:**

| Route | msgType 1 (SEND) | msgType 2 (SEND_AND_CALL) |
|-------|-------------------|---------------------------|
| Fraxtal→Somnia | `0x000301001101000000000000000000000000000f4240` (1M gas) | `0x0003010011010000000000000000000000000016e360` (1.5M gas) |
| Somnia→Fraxtal | `0x00030100110100000000000000000000000000030d40` (200k gas) | `0x0003010011010000000000000000000000000003d090` (250k gas) |

### 5. Decode and Verify setPeer (tx1)

```bash
data=$(jq -r '.transactions[1].data' "$f")
cast decode-calldata "setPeer(uint32,bytes32)" "$data"
```

Verify:
- EID matches remote chain
- bytes32 is the **left-padded** remote OFT address

**bytes32 left-padding format:**

```
Address: 0x00000000E9CE0f293D1Ce552768b187eBA8a56D4
bytes32: 0x00000000000000000000000000000000e9ce0f293d1ce552768b187eba8a56d4
```

> **Gotcha:** `cast to-bytes32` produces **right-padded** output. For setPeer verification, manually left-pad: `printf "0x%064s" "${addr#0x}" | tr ' ' '0'`

### 6. Decode and Verify setConfig (tx2, tx3)

```bash
data=$(jq -r '.transactions[2].data' "$f")
cast decode-calldata "setConfig(address,address,(uint32,uint32,bytes)[])" "$data"
```

Verify for tx2 (send DVNs):
- First arg = oapp (lockbox address)
- Second arg = send library
- Config tuples contain 3 DVN addresses in the bytes field

Verify for tx3 (receive DVNs):
- First arg = oapp (lockbox address)
- Second arg = receive library
- Config tuples contain 3 DVN addresses in the bytes field

**DVN addresses appear ABI-encoded inside the bytes field.** Search for lowercase DVN addresses in the decoded output.

### 7. Decode and Verify setSendLibrary (tx4)

```bash
data=$(jq -r '.transactions[4].data' "$f")
cast decode-calldata "setSendLibrary(address,uint32,address)" "$data"
```

Verify:
- First arg = oapp (lockbox address)
- Second arg = remote EID
- Third arg = send library address (e.g., `sendLib302`)

### 8. Decode and Verify setReceiveLibrary (tx5)

```bash
data=$(jq -r '.transactions[5].data' "$f")
cast decode-calldata "setReceiveLibrary(address,uint32,address,uint256)" "$data"
```

Verify:
- First arg = oapp (lockbox address)
- Second arg = remote EID
- Third arg = receive library address (e.g., `receiveLib302`)
- Fourth arg = grace period (typically `0`)

## Full Automated Verification Script

A comprehensive verification script that checks all arguments across all files:

```bash
#!/bin/bash
set -euo pipefail

# === Configuration ===
REMOTE_EID=30380  # Somnia EID
ENDPOINT="0x1a44076050125825900e736c501f859c50fe728c"  # Fraxtal endpoint
SEND_LIB="0x377530cda84dfb2673bf4d145dcf0c4d7fdcb5b6"
RECV_LIB="0x8bc1e36f015b9902b54b1387a4d733cebc2f5a4e"

# DVNs for the route
DVN_FRAX="0x26cd5abadf7ec3f0f02b48314bfca6b2342cddd4"
DVN_LZ="0xcce466a522984415bc91338c232d98869193d46e"
DVN_HORIZEN="0xdd7b5e1db4aafd5c8ec3b764efb8ed265aa5445b"

# OFT → Lockbox mapping
declare -A LOCKBOXES=(
  [wfrax]="0xd86fbbd0c8715d2c1f40e451e5c3514e65e7576a"
  [sfrxusd]="0x88aa7854d3b2daa5e37e7ce73a1f39669623a361"
  [sfrxeth]="0x999dfabe3b1cc2ef66eb032eea42fea329bba168"
  [frxusd]="0x96a394058e2b84a89bac9667b19661ed003cf5d4"
  [frxeth]="0x9abfe1f8a999b0011ecd6116649aee8d575f5604"
  [fpi]="0x75c38d46001b0f8108c4136216bd2694982c20fc"
)

declare -A PEERS=(
  [wfrax]="0x00000000e9ce0f293d1ce552768b187eba8a56d4"
  [sfrxusd]="0x00000000fd8c4b8a413a06821456801295921a71"
  [sfrxeth]="0x00000000883279097a49db1f2af954ead0c77e3c"
  [frxusd]="0x00000000d61733e7a393a10a5b48c311abe8f1e5"
  [frxeth]="0x000000008c3930dca540bb9b3a5d0ee78fca9a4c"
  [fpi]="0x00000000bc4aef4ba6363a437455cb1af19e2aeb"
)

# Expected function selectors
EXPECTED_SELS=("0xb98bd070" "0x3400288b" "0x6dbd9f90" "0x6dbd9f90" "0x9535ff30" "0x6a14d715")

PASS=0
FAIL=0

check() {
  local desc="$1" got="$2" exp="$3"
  got_l=$(echo "$got" | tr '[:upper:]' '[:lower:]')
  exp_l=$(echo "$exp" | tr '[:upper:]' '[:lower:]')
  if [[ "$got_l" == *"$exp_l"* ]]; then
    PASS=$((PASS + 1))
  else
    echo "FAIL $desc: expected=$exp got=$got"
    FAIL=$((FAIL + 1))
  fi
}

for token in wfrax sfrxusd sfrxeth frxusd frxeth fpi; do
  f="scripts/DeployFraxOFTProtocol/txs/5031-252-${token}.json"
  lockbox="${LOCKBOXES[$token]}"
  peer="${PEERS[$token]}"
  
  # File metadata
  check "$token/txCount" "$(jq '.transactions | length' "$f")" "6"
  
  # Target addresses
  for i in 0 1; do
    to=$(jq -r ".transactions[$i].to" "$f" | tr '[:upper:]' '[:lower:]')
    check "$token/tx$i/target" "$to" "$lockbox"
  done
  for i in 2 3 4 5; do
    to=$(jq -r ".transactions[$i].to" "$f" | tr '[:upper:]' '[:lower:]')
    check "$token/tx$i/target" "$to" "$ENDPOINT"
  done
  
  # Function selectors
  for i in $(seq 0 5); do
    data=$(jq -r ".transactions[$i].data" "$f")
    sel="${data:0:10}"
    check "$token/tx$i/selector" "$sel" "${EXPECTED_SELS[$i]}"
  done
  
  # setEnforcedOptions — EID
  d0=$(jq -r '.transactions[0].data' "$f")
  decoded=$(cast decode-calldata "setEnforcedOptions((uint32,uint16,bytes)[])" "$d0" 2>&1)
  check "$token/tx0/eid" "$decoded" "$REMOTE_EID"
  
  # setPeer — exact bytes32
  d1=$(jq -r '.transactions[1].data' "$f")
  got_peer=$(cast decode-calldata "setPeer(uint32,bytes32)" "$d1" | tail -n1 | tr -d '[:space:]' | tr '[:upper:]' '[:lower:]')
  addr=$(echo "$peer" | tr '[:upper:]' '[:lower:]' | sed 's/^0x//')
  exp_peer=0x$(printf "%064s" "$addr" | tr ' ' '0')
  check "$token/tx1/peer" "$got_peer" "$exp_peer"
  
  # setConfig tx2 — send DVNs
  d2=$(jq -r '.transactions[2].data' "$f")
  decoded2=$(cast decode-calldata "setConfig(address,address,(uint32,uint32,bytes)[])" "$d2" 2>&1 | tr '[:upper:]' '[:lower:]')
  check "$token/tx2/oapp" "$decoded2" "$lockbox"
  check "$token/tx2/sendlib" "$decoded2" "$SEND_LIB"
  check "$token/tx2/dvn_frax" "$decoded2" "$DVN_FRAX"
  check "$token/tx2/dvn_lz" "$decoded2" "$DVN_LZ"
  check "$token/tx2/dvn_horizen" "$decoded2" "$DVN_HORIZEN"
  
  # setConfig tx3 — receive DVNs
  d3=$(jq -r '.transactions[3].data' "$f")
  decoded3=$(cast decode-calldata "setConfig(address,address,(uint32,uint32,bytes)[])" "$d3" 2>&1 | tr '[:upper:]' '[:lower:]')
  check "$token/tx3/oapp" "$decoded3" "$lockbox"
  check "$token/tx3/recvlib" "$decoded3" "$RECV_LIB"
  check "$token/tx3/dvn_frax" "$decoded3" "$DVN_FRAX"
  check "$token/tx3/dvn_lz" "$decoded3" "$DVN_LZ"
  check "$token/tx3/dvn_horizen" "$decoded3" "$DVN_HORIZEN"
  
  # setSendLibrary tx4
  d4=$(jq -r '.transactions[4].data' "$f")
  decoded4=$(cast decode-calldata "setSendLibrary(address,uint32,address)" "$d4" 2>&1 | tr '[:upper:]' '[:lower:]')
  check "$token/tx4/oapp" "$decoded4" "$lockbox"
  check "$token/tx4/sendlib" "$decoded4" "$SEND_LIB"
  
  # setReceiveLibrary tx5
  d5=$(jq -r '.transactions[5].data' "$f")
  decoded5=$(cast decode-calldata "setReceiveLibrary(address,uint32,address,uint256)" "$d5" 2>&1 | tr '[:upper:]' '[:lower:]')
  check "$token/tx5/oapp" "$decoded5" "$lockbox"
  check "$token/tx5/recvlib" "$decoded5" "$RECV_LIB"
  check "$token/tx5/grace" "$decoded5" "0"
done

echo "========================================="
echo "PASSED=$PASS FAILED=$FAIL"
[ "$FAIL" -eq 0 ] && echo "ALL CHECKS PASSED" || echo "SOME CHECKS FAILED"
```

## Gotchas

| Issue | Cause | Fix |
|-------|-------|-----|
| setPeer bytes32 comparison fails | `cast to-bytes32` right-pads, but setPeer uses left-padded bytes32 | Use `printf "0x%064s" "${addr}" \| tr ' ' '0'` for left-padding |
| DVN address not found in decoded bytes | Case mismatch | Lowercase both sides before comparing |
| `cast decode-calldata` fails | Wrong function signature string | Ensure exact Solidity signature including tuple syntax |
| Shell heredoc breaks in inline commands | Complex quoting with jq + cast | Write verification to a standalone script file |
| jq "file not found" errors | Shell cwd differs from expected | Use absolute paths or `cd` first |

## Reference: OFT Address → Lockbox Mapping (Fraxtal)

| Token | OFT (Somnia) | Lockbox (Fraxtal) |
|-------|-------------|-------------------|
| wfrax | `0x00000000E9CE0f293D1Ce552768b187eBA8a56D4` | `0xd86fBBd0c8715d2C1f40e451e5C3514e65E7576A` |
| sfrxUSD | `0x00000000fD8C4B8A413A06821456801295921a71` | `0x88Aa7854d3b2dAA5e37E7Ce73A1F39669623a361` |
| sfrxETH | `0x00000000883279097A49dB1f2af954EAd0C77E3c` | `0x999dfAbe3b1cc2EF66eB032Eea42FeA329bBa168` |
| frxUSD | `0x00000000D61733e7A393A10A5B48c311AbE8f1E5` | `0x96A394058E2b84A89bac9667B19661Ed003cF5D4` |
| frxETH | `0x000000008c3930dCA540bB9B3A5D0ee78FcA9A4c` | `0x9aBFE1F8a999B0011ecD6116649AEe8D575F5604` |
| FPI | `0x00000000bC4aEF4bA6363a437455Cb1af19e2aEb` | `0x75c38D46001b0F8108c4136216bd2694982C20FC` |
