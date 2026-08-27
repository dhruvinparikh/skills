# skills
Skills that I taught LLM....

## verify-foundry-broadcast-deployment
End-to-end verification that a Foundry `broadcast/.../run-latest.json` deployment landed
on-chain and matches the compiled artifact. Covers CREATE/CREATE2 bytecode + codehash
checks, deterministic address derivation, and (for upgrade scripts) EIP-1967 impl-slot
checks to determine whether the proxy upgrade has been applied by the multisig yet.

## oklink-contract-verification
Source code verification on OKLink explorer (xLayer / OKX chains). Uses the v5 API
directly via curl (no API key needed — dummy `Ok-Access-Key: test` works). Covers
standard JSON input submission, polling, and proxy verification. Documents the critical
optimizer-runs mismatch gotcha between `forge verify-contract --show-standard-json-input`
(which ignores `FOUNDRY_PROFILE`) and the actual deployment settings.

## lz-manual-compose-retry
Diagnose and manually retry a LayerZero V2 `lzCompose` delivery that failed because the
Executor's compose gas budget was too low (visible as an `LzComposeAlert` event on an
otherwise-successful tx). Covers decoding the exact `_message` bytes from raw calldata
(not the explorer UI, which can mangle it), confirming the message is still queued via
`composeQueue`, simulating with `cast call --trace` to find the real gas requirement, and
broadcasting the permissionless retry. Also flags that some public RPCs return
prompt-injection-style payment demands in `eth_call` error responses.
