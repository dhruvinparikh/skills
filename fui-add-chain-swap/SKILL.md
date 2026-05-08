# Add New Chain to FUI Frontend Swap

## Overview

Step-by-step process to add a new EVM chain with LayerZero OFT support to the Frax FUI frontend swap interface (`fui-frontend`).

## When to Use

- Adding a new chain to the bridge/swap UI on frax.com
- Enabling LayerZero OFT bridging for Frax tokens on a newly deployed chain

## Prerequisites

Before starting, gather:

| Detail | Where to find |
|--------|---------------|
| Chain ID | L0Config.json, chainlist.org, or deployment notes |
| LayerZero Endpoint ID (eid) | L0Config.json `"eid"` field |
| RPC URL | L0Config.json `"RPC"` field or chainlist.org |
| Block explorer URL | Chain documentation |
| Native currency (name, symbol, decimals) | chainlist.org |
| OFT token addresses (frxUSD, sfrxUSD, frxETH, sfrxETH, WFRAX, FPI) | `frax-oft-upgradeable/README.md` |
| Chain logo URL | Upload to `https://static.frax.com/images/chains/{chain}.png` |

## Key Decision: viem Built-in vs Manual Chain Definition

**Check first**: Does `viem/chains` already export the chain?

```bash
ls node_modules/viem/_esm/chains/definitions/{chain}.js
```

- **If yes** → Import from `viem/chains` (e.g., `import { somnia } from "viem/chains"`). No need for a manual `*_MAINNET_INFO` constant. Use `{chain}.id`, `{chain}.name`, `{chain}.blockExplorers.default`, etc.
- **If no** → Define a `{CHAIN}_MAINNET_INFO` constant in `chain-constants.ts` following the pattern of `TEMPO_MAINNET_INFO` or `STABLE_MAINNET_INFO`.

viem `^2.43` already includes most popular chains. Always check before creating a manual definition.

## Files to Modify (7 files)

All paths relative to `fui-frontend/src/`:

### 1. `lib/chain-constants.ts` — Chain definition (only if not in viem)

Add a `{CHAIN}_MAINNET_INFO` export with `id`, `name`, `nativeCurrency`, `rpcUrls`, `blockExplorers`, `testnet`, and `contracts.multicall3`.

### 2. `lib/config.ts` — Wagmi registration

- **Import**: Add chain to either `chain-constants` import or `viem/chains` import
- **`chains` array**: Add to the wagmi `createConfig({ chains: [...] })` array
- **`transports`**: Add transport entry — use `http()` for viem built-ins (uses default RPC), or `http(URL)` for custom RPCs

### 3. `lib/tokens/master.ts` — Token address mappings

- **`NETWORK_TO_CHAIN_ID`**: Add `{chain}: {chainId}` entry (alphabetical order)
- **`ALL_TOKENS`**: Add `{chain}: "0x..."` address entry to each token's `address` map:
  - `sfrxETH`, `FPI`, `WFRAX`, `frxUSD`, `sfrxUSD`, `frxETH`
- Keep entries alphabetically sorted within each address map

### 4. `lib/tokens/tokens.mainnet.ts` — Per-chain token list

Add a `{CHAIN}_MAINNET_TOKENS` export object. Each entry follows:

```typescript
export const SOMNIA_MAINNET_TOKENS = {
  "{chainId}~{lowercaseAddress}": {
    tokenId: "{chainId}~{lowercaseAddress}",
    name: "Frax USD",
    symbol: "frxUSD",
    decimals: 18,           // 18 for most, 6 for Tempo frxUSD (TIP-20)
    address: "{lowercaseAddress}",
    chainId: {chainId},
    logoURI: "https://static.frax.com/images/tokens/{token}.png",
    balanceOf: undefined,
    xrefTokenId: undefined,
    isL0Supported: true,
    isL0FraxtalHub: true,
  },
  // ... repeat for sfrxUSD, frxETH, sfrxETH, WFRAX, FPI
} as TokensObject;
```

Standard 6 tokens: frxUSD, sfrxUSD, frxETH, sfrxETH, WFRAX, FPI.

Logo URIs:
- frxUSD → `frxusd.png`
- sfrxUSD → `sfrxusd.png`
- frxETH → `frxeth.png` (not frxeth-256x256)
- sfrxETH → `sfrxeth.png`
- WFRAX → `wfrax.png`
- FPI → `fpi.png`

### 5. `lib/utils.ts` — APPROVED_CHAINS map

- **Import**: Add chain constant + `{CHAIN}_MAINNET_TOKENS`
- **`APPROVED_CHAINS` Map**: Add entry with `id`, `name`, `blockExplorer`, `logoURI`, `tokenList`, `nativeCurrency`, `eid`

```typescript
[somnia.id, {
  id: somnia.id,
  name: somnia.name,
  blockExplorer: somnia.blockExplorers.default,
  logoURI: "https://static.frax.com/images/chains/somnia.png",
  tokenList: SOMNIA_MAINNET_TOKENS,
  nativeCurrency: somnia.nativeCurrency,
  eid: 30380,
}],
```

### 6. `features/shared/token-select/state/token-select-state.tsx` — Initial token state

- **Import**: Add `{CHAIN}_MAINNET_TOKENS` from tokens.mainnet
- **`tokens` object**: Add `...{CHAIN}_MAINNET_TOKENS` spread (alphabetical)

### 7. `features/shared/token-select/actions.tsx` — Token fetching & display

- **Import chain constant** (from chain-constants or viem/chains)
- **Import `{CHAIN}_MAINNET_TOKENS`** from tokens.mainnet
- **`mutableTokenMaps`**: Add `[{chain}.id]: { ...{CHAIN}_MAINNET_TOKENS }`
- **Chain name switch**: Add `case {chain}.id: return "{ChainName}";`
- **`chainFetches` array**: Add `{ name: "{ChainName}", promise: fetchChainBalances({chain}.id, mutableTokenMaps[{chain}.id]) }`

## Special Cases

### Chains with non-standard gas (e.g., Tempo)

Tempo pays gas in ERC-20 tokens instead of native ETH. This requires:
- Special `feeToken` in wagmi chain config
- Custom bridge logic in `lib/tempo/index.ts`
- `isTempoChainId()` checks throughout swap flow
- frxUSD has 6 decimals on Tempo (TIP-20 precompile), vs 18 on other chains

Most new chains use standard native gas and don't need this.

### Remote Hop

If the chain supports remote hop (cross-chain bridging without going through Fraxtal), add to `REMOTE_HOP_ADDDRESSES` in `lib/tokens/master.ts`:

```typescript
export const REMOTE_HOP_ADDDRESSES: Record<string, `0x${string}`> = {
  // ...existing entries
  somnia: "0x...",
};
```

This is typically added later after RemoteHop contracts are deployed.

### LayerZero Lockbox (Adapter chains)

If the chain uses a Mintable Adapter instead of a standard OFT (like Ethereum, Fraxtal, Tempo for frxUSD), add `layerZeroLockBox` entries in `master.ts` `ALL_TOKENS` and update the `TokenDetails` interface if needed.

## Verification Checklist

- [ ] Chain appears in swap source/destination dropdown
- [ ] All 6 tokens (frxUSD, sfrxUSD, frxETH, sfrxETH, WFRAX, FPI) show for the chain
- [ ] Token balances load correctly when wallet is connected to the chain
- [ ] Bridge quote works (LayerZero fee estimation)
- [ ] No TypeScript compilation errors in modified files
- [ ] Chain logo renders (or use inline SVG as fallback)
