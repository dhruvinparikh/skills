# Metadata-Tail Ethereum Contract Verification

## Overview

This skill is for cases where contract verification fails with bytecode mismatch even though constructor args and source look correct.

Core idea:

- Treat deployment broadcast as source of truth
- Use on-chain bytecode tail to drive setting selection
- Separate logic-bytecode matching from metadata-hash matching

This method is especially useful for Foundry projects using imported dependencies and multiple remappings.

## When to Use

- Etherscan V2 verification fails with: compiled deployment bytecode does NOT match
- Contract logic appears correct but verification still fails
- Same contract address exists on another chain, but cross-chain settings do not directly verify
- Need deterministic replay of historical deployment build profile

## Ground Truth Inputs

Use deployment artifact first:

- broadcast/<SCRIPT>.s.sol/<CHAIN_ID>/run-latest.json

Extract:

- transactions[].hash (deployment tx)
- transactions[].contractName
- transactions[].contractAddress
- transactions[].arguments
- transactions[].transaction.input (init code + constructor args)

## Core Diagnostic Flow

### 1. Confirm constructor arguments from broadcast

- Do not rely on memory or docs
- Use ABI encode exactly from recorded arguments

### 2. Compare runtime bytecode first

Take deployed runtime bytecode from explorer and compare against local compiled deployedBytecode under multiple settings.

Useful sweep dimensions:

- contract target symbol (wrapper and imported base)
- via_ir true/false
- optimizer runs
- evm version

Goal:

- Find setting combo where executable/runtime shape matches
- If runtime logic matches but verification still fails, metadata mismatch is likely

### 3. Inspect metadata tail

Solidity appends metadata to bytecode tail, typically ending with a marker similar to:

- a26469706673582212...64736f6c63...

If only tail differs, executable logic can still be identical while verification fails.

### 4. Align metadata-sensitive compiler settings

In Foundry, these flags are commonly decisive:

- optimizer = true
- optimizer_runs = <exact>
- evm_version = <exact>
- via_ir = <exact>
- bytecode_hash = ipfs
- cbor_metadata = true
- use_literal_content = true

Note:

- use_literal_content can change metadata hash even when logic is unchanged
- source path identity and remapping resolution also affect metadata

### 5. Verify via Etherscan V2 endpoint

Use:

- --verifier-url https://api.etherscan.io/v2/api

and pass explicit compiler version and constructor args.

## Practical Command Pattern

Generate constructor args:

- cast abi-encode 'constructor(address)' <OWNER>

Verification command:

- forge verify-contract <ADDRESS> <PATH>:<CONTRACT> --chain-id 1 --constructor-args "$(cast abi-encode 'constructor(address)' <OWNER>)" --compiler-version v0.8.22+commit.4fc1097e --etherscan-api-key "$ETHERSCAN_API_KEY" --verifier-url "https://api.etherscan.io/v2/api" --watch

## Important Lessons

- Same address on another chain is not proof of identical deploy bytecode
- Broadcast file is stronger evidence than explorer UI assumptions
- Runtime match and metadata match are different problems
- Verification can fail solely due to metadata-tail divergence

## Case Outcome Reference

In this workspace, successful verification required metadata-sensitive alignment in Foundry config after runtime/constructor validation from broadcast.
