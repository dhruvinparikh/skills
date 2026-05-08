# Msig Signer Comms: HackMD Doc + TG Message

## Overview

When generating a batch of Gnosis Safe transactions (Tx Builder JSONs) that require multisig signatures across multiple chains, produce two artifacts for the team:

1. **A HackMD-ready Markdown doc** — a per-msig signing guide with key addresses, batch tables, decoded calldata, and `cast` verification commands. This is what signers read before signing.
2. **A short Telegram message** — a 2–4 line nudge linking the HackMD + PR. It must NOT repeat what's in HackMD.

## Bundled Examples (this folder)

This skill is self-contained. Reference artifacts live in [`examples/`](./examples):

| File | What it is |
|------|------------|
| [`examples/reference-SetExecutorOptionsTempo.md`](./examples/reference-SetExecutorOptionsTempo.md) | Canonical signing doc to mirror — layout, table style, `<details>` cast blocks, "What Each Tx Does" pseudocode |
| [`examples/output-SetLegacyOFTLibs.md`](./examples/output-SetLegacyOFTLibs.md) | A full real output produced from this skill — 4 chains, 18 batches, 90 txs |
| [`examples/sample-input-batch-metis-1088-FRAX.json`](./examples/sample-input-batch-metis-1088-FRAX.json) | Example Safe Tx Builder input (Metis side: alternating `setSendLibrary` + `setReceiveLibrary`) |
| [`examples/sample-input-batch-ethereum-1-FRAX.json`](./examples/sample-input-batch-ethereum-1-FRAX.json) | Example Safe Tx Builder input (Ethereum lockbox side: receive-only) |
| [`examples/tg-message-template.md`](./examples/tg-message-template.md) | TG message template + real example |

## When to Use

User asks to "create a hackmd / msig doc / signer doc" for a folder of generated Safe batch JSONs (typically `*/txs/*.json` with `chainId`, `transactions[]`, `data`, `to`, `value`).

## Inputs to Gather (in this order)

1. **A reference doc to mirror.** If the user provides one, use it. Otherwise default to [`examples/reference-SetExecutorOptionsTempo.md`](./examples/reference-SetExecutorOptionsTempo.md). Mirror its structure exactly: section ordering, table style, `<details>` cast blocks, "What Each Batch/Tx Does" pseudocode.
2. **The txs folder** — list and read every JSON. From each:
   - `chainId` → maps to source chain
   - `to` → target contract (note if same across all txs)
   - `data` → first 4 bytes = function selector; decode the rest
   - Group txs by OFT/asset and by destination EID encoded in calldata
3. **The msig URL source** — usually the project's main README. Pull the exact Safe URL per chain. Don't fabricate URLs.
4. **Nonce** — the user typically says "I'll take care of nonce". Do NOT invent or guess nonces. Leave them out (or write "TBD") unless explicitly given.

## HackMD Doc Structure

Reproduce the reference doc's layout. Skeleton:

```markdown
## <FixName> — Msig Signatures Required

<one-paragraph context: what's being changed, why, what contract is targeted>

PR: <github PR URL if available>

### Key Addresses (Common Across All Chains)
| Role | Address |
|------|---------|
| <shared contract> | `0x...` |

### Route EID Reference
| Chain | LZ EID | Hex |
|-------|--------|-----|

---

## 1. <Chain> Msig

### <Chain> Msig : <safe URL with /transactions/queue?safe=...>

#### Key Addresses (<Chain>)
| Role | Address |
|------|---------|

#### Batches (N separate Safe uploads)
| File | OFT/Asset | Tx Count | Routes / Action |
|------|-----------|----------|-----------------|

#### What Each Batch Does
For each <thing> × <thing>:
\`\`\`
functionName(
    arg1: <value>
)
\`\`\`

<details>
<summary>Cast verification — <Chain> <Asset> batch (example tx #N)</summary>

\`\`\`bash
cast calldata-decode "<full signature>" <full hex calldata>
\`\`\`

Expected:
\`\`\`
<decoded args with inline comments>
\`\`\`
</details>

---

<repeat per chain>

## Summary
| Chain | Msig | Batches | Total Txs |
|-------|------|---------|-----------|
| **Total** | | **N batches** | **N txs** |
```

### Critical Conventions

- **No nonces.** User handles those.
- **One row per JSON file** in the batch table.
- **Summary table** at the bottom with batch + tx counts that match what's actually in the JSONs.
- **Cast verification: decode at least one tx per chain** as a worked example. Show full hex `data` and decoded values inline-commented.
- **Identify shared addresses** (e.g. LZ EndpointV2 same on all chains, deterministic OFT addresses) and put them in a single "Common" table at the top to avoid repetition.
- **Note chain-specific peculiarities** (e.g. "Ethereum lockboxes only need `setReceiveLibrary` — no send side"). See [`examples/output-SetLegacyOFTLibs.md`](./examples/output-SetLegacyOFTLibs.md) section 4 for the pattern.

### Common Function Selectors

| Selector | Function |
|----------|----------|
| `0x9535ff30` | `setSendLibrary(address,uint32,address)` |
| `0x6a14d715` | `setReceiveLibrary(address,uint32,address,uint256)` |
| `0x4198dcf4` | `sendOFT(address,uint32,bytes32,uint256,uint128,bytes)` |
| `0x5135db46` | `setExecutorOptions(uint32,bytes)` |
| `0x2f2ff15d` | `grantRole(bytes32,address)` |
| `0xd547741f` | `revokeRole(bytes32,address)` |

If a selector is unfamiliar, run `cast 4byte <selector>`.

## Telegram Message

See [`examples/tg-message-template.md`](./examples/tg-message-template.md) for the full template + a real example. Rules in short:

- 2–4 lines max, lowercase casual, no emojis (unless asked), no headings.
- Lead with the ask. Two links only: HackMD + PR.
- Do NOT repeat msig URLs, addresses, EIDs, or per-chain counts (those are in HackMD).

## Workflow

1. Read the reference doc (user-provided or [`examples/reference-SetExecutorOptionsTempo.md`](./examples/reference-SetExecutorOptionsTempo.md)) end-to-end.
2. List the txs folder; read every JSON in parallel.
3. Read the project README to extract Safe URLs per chain.
4. Decode at least one tx per unique selector to confirm function signatures.
5. Write the HackMD doc as a single new `.md` file under the project's docs folder. The user pastes its contents into HackMD and supplies the URL back.
6. After receiving the HackMD URL, write the short TG message using [`examples/tg-message-template.md`](./examples/tg-message-template.md).
7. If asked to commit: stage ONLY the script/txs/doc files for that op. Inspect `git status` and unstage stray modifications (e.g. `contracts/flat/*Flat.sol` regenerations) before committing. Use `commit -S` for signed commits when user asks.

## Common Pitfalls

- Including flat/generated contract diffs in the commit. Always inspect `git status` and reset stray changes.
- Inventing nonces. Don't.
- Repeating msig URL in the TG message that's already in HackMD.
- Forgetting the Safe URL on the `## <Chain> Msig :` heading line — signers grep this format.
- Wrong Safe domain (e.g., Brahma vs Safe Protofire for Blast — check README, may have been updated).
- Mixing `setSendLibrary` and `setReceiveLibrary` ordering: in batches they alternate per-route (send, receive, send, receive, ...). Confirm by decoding two consecutive txs — see [`examples/sample-input-batch-metis-1088-FRAX.json`](./examples/sample-input-batch-metis-1088-FRAX.json).
