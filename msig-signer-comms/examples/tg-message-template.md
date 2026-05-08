# Telegram Message Template

Use this verbatim template for the short TG nudge after the HackMD doc is published. **Do not include addresses, EIDs, batch counts, or per-chain breakdowns** — those belong in HackMD.

## Template

```
hey team, need sigs to <one-sentence action> on <chain list>

<N> batches total, all value=0 — upload JSONs from PR into Safe tx builder

doc: <hackmd URL>
PR: <github PR URL>
```

## Real Example (used for SetLegacyOFTLibs)

```
hey team, need sigs to explicitly set send/receive libs on legacy OFTs (FRAX, frxETH, sfrxETH, FPI) across Metis, Blast, Base + Ethereum lockboxes

18 batches total, all value=0 — upload JSONs from PR into Safe tx builder

doc: https://hackmd.io/@dhruvin-frax/legacy-oft-lz-lib
PR: https://github.com/FraxFinance/frax-oft-upgradeable/pull/140
```

## Rules

- 2–4 lines. No headings. No emojis (unless user explicitly asks).
- Lowercase casual tone.
- Lead with the ask, not the context.
- Two links only: HackMD + PR. Do not paste msig URLs (they're in HackMD).
- If `value=0` is not true for all txs, replace with the actual fee total or omit.
