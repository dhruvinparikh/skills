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
