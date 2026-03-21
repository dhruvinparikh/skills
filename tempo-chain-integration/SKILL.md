# Tempo Chain Integration

## Overview

Tempo (chainId **4217**, LZ eid **40322**) is a purpose-built Layer 1 blockchain optimized for payments. It uses **ERC-20 tokens (TIP-20) for gas** instead of native ETH — there is no native coin. It has a custom EIP-2718 transaction envelope type (`0x76`) with support for batch calls, fee sponsorship, configurable fee tokens, concurrent nonces, access keys, and scheduled execution.

Frax tokens on Tempo are deployed as LayerZero OFTs bridging to/from Fraxtal (hub model). frxUSD is special: it's a native TIP-20 precompile token on Tempo with a `FraxOFTMintableAdapterUpgradeableTIP20` OFT adapter.

## When to Use

- Adding Tempo chain support to a frontend or dApp
- Deploying OFT contracts to Tempo
- Bridging tokens to/from Tempo via LayerZero
- Handling Tempo's TIP-20 token standard
- Integrating with Tempo's stablecoin DEX, Fee AMM, or fee system

## Chain Details

| Property | Value |
|----------|-------|
| Chain ID | 4217 |
| LZ Endpoint ID | 40322 |
| Chain name (viem/wagmi) | `tempoModerato` |
| TX type | `0x76` (Tempo Transaction) |
| Gas model | ERC-20 (TIP-20 stablecoins), no native coin |
| Block explorer | https://explorer.tempo.xyz |
| RPC | https://rpc.tempo.xyz |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |

## Key Concepts

### TIP-20 Token Standard

Tempo has a protocol-level token standard called **TIP-20**. These tokens:
- Have addresses in the `0x20C0...` range (e.g., `0x20C0000000000000000000003554d28269E0f3c2` for frxUSD)
- Support ERC-20 interface (`balanceOf`, `approve`, `transferFrom`)
- Have native protocol actions via viem: `token.getBalance`, `token.approve`, `token.transfer`, `token.getMetadata`
- Can be used to pay transaction fees via the `feeToken` field
- Support mint/burn, pause, transfer policies (blacklist/whitelist), rewards, and role-based access
- Roles: `ISSUER_ROLE` (mint/burn), `BURN_BLOCKED_ROLE`, `PAUSE_ROLE`, `UNPAUSE_ROLE`

### Tempo Transaction Type (0x76)

Tempo has a custom EIP-2718 transaction type (`0x76`) that supports:

| Field | Type | Description |
|-------|------|-------------|
| `feeToken` | `Address` | Pay gas in any USD-denominated TIP-20 token |
| `feePayer` | `Address \| Account \| true` | Sponsor fees for another account |
| `calls` | `Array<{to, data, value}>` | Batch multiple operations atomically |
| `nonceKey` | `bigint` | Concurrent transactions with different nonce keys |
| `validAfter` | `number` | Earliest unix timestamp for tx inclusion |
| `validBefore` | `number` | Latest unix timestamp for tx inclusion |
| `keyAuthorization` | `Hex` | Access key delegation (e.g., WebCrypto keys) |

### ERC20 Gas Model

Tempo has **no native ETH-like gas token**. Gas is paid in TIP-20 stablecoins.
- `nativeCurrency: { name: "USD", symbol: "USD", decimals: 6 }`
- The wallet pays fees in frxUSD (or another configured `feeToken`)
- `msg.value` is always `0n` for LZ bridge calls on Tempo
- User's gas token preference is managed via the `TIP_FEE_MANAGER` precompile
- Tempo's built-in Fee AMM automatically swaps between user's token and validator's preferred token

## Contract Architecture on Tempo

### OFT Inheritance Hierarchy

All Tempo OFTs inherit from `TempoAltTokenBase`, which provides ERC-20 gas payment via `EndpointV2Alt`:

```
TempoAltTokenBase
├── FraxOFTUpgradeableTempo (sfrxUSD, frxETH, sfrxETH, WFRAX, FPI)
│   └── Standard OFT: lock/unlock on send/receive, IS the ERC20 token
└── FraxOFTMintableAdapterUpgradeableTIP20 (frxUSD only)
    └── Adapter OFT: mint/burn the underlying TIP-20 precompile
```

### TempoAltTokenBase Key Functions

| Function | Purpose |
|----------|---------|
| `_resolveUserToken()` | Get user's gas token: `TIP_FEE_MANAGER.userTokens(msg.sender)` → fallback to `PATH_USD` |
| `_findSwapTarget()` | Find cheapest swap path via `STABLECOIN_DEX.quoteSwapExactAmountOut()` |
| `_validateQuoteSwapPath()` | Called from `_quote()` override to verify swap path exists |
| `_payNativeAltToken()` | Pull ERC-20 fee from user, swap if needed, wrap via `LZEndpointDollar`, send to endpoint |
| `quoteUserTokenFee()` | External view for UIs to estimate gas cost in any TIP-20 token |

### FraxOFTUpgradeableTempo

Used for: sfrxUSD, frxETH, sfrxETH, WFRAX, FPI

These contracts **ARE the ERC20 token** combined with the OFT. `balanceOf()` works directly on the OFT address.

Overrides from base:
- `send()` — rejects `msg.value > 0` (gas is ERC-20, not ETH)
- `_quote()` → `_validateQuoteSwapPath(super._quote(...))` — validates swap path exists
- `_payNative()` → `_payNativeAltToken(_nativeFee, address(endpoint))` — pays LZ fee in ERC-20

```
User calls OFT.send() → _debit() calls _burn(msg.sender, amount)
```

### FraxOFTMintableAdapterUpgradeableTIP20

Used for: **frxUSD only**

This is an OFT **adapter** that wraps a pre-existing TIP-20 token. The adapter is NOT an ERC20 — `balanceOf()` reverts on it.

```
Addresses:
- TIP-20 token (actual frxUSD): 0x20C0000000000000000000003554d28269E0f3c2
- OFT adapter (bridge):          0x00000000D61733e7A393A10A5B48c311AbE8f1E5

User approves adapter → adapter.send() → _debit() calls:
  ITIP20(innerToken).transferFrom(msg.sender, address(this), amount)
  ITIP20(innerToken).burn(amount)
```

**Critical distinction:**
| Token | ERC20 address (balanceOf) | OFT/Bridge address (send/quoteSend) |
|-------|---------------------------|-------------------------------------|
| frxUSD | `0x20C0...3c2` (TIP-20) | `0x0000...1E5` (adapter) |
| sfrxUSD | `0x0000...a71` (same) | `0x0000...a71` (same) |
| frxETH | `0x0000...9a4c` (same) | `0x0000...9a4c` (same) |
| Others | OFT address | OFT address |

### Gas Payment Flow (Bridge Send from Tempo)

```
User calls OFT.send(params, fee, refundAddress) with msg.value == 0
  ├── _debit(): transferFrom user → adapter/lock (or burn for frxUSD)
  ├── _quote() → _validateQuoteSwapPath(): verify swap route exists
  └── _payNative() → _payNativeAltToken():
      1. Get userToken from TIP_FEE_MANAGER.userTokens(msg.sender)
      2. Check if userToken is whitelisted in LZEndpointDollar:
         ├── YES: transferFrom(user), approve(LZEndpointDollar), wrap() directly
         └── NO: transferFrom(user), swap to PATH_USD via DEX, then wrap()
      3. Wrapped tokens sent to LZ endpoint as gas
```

### LZEndpointDollar

Wrapper contract for LayerZero EndpointV2Alt's native token. Manages whitelisted stablecoins for gas payment:
- `wrap(token, to, amount)` — wraps a whitelisted token and sends to recipient
- `unwrap(token, to, amount)` — unwraps back to the original token
- `isWhitelistedToken(token)` — checks if a token is whitelisted for wrapping
- `getWhitelistedTokens()` — returns all whitelisted token addresses

### FrxUSDPolicyAdminTempo (Freeze/Thaw)

Admin contract for managing freeze/thaw operations via TIP-403 Registry (BLACKLIST policy):
- `freeze(account)` / `freezeMany(accounts[])` — add to blacklist (blocks all transfers)
- `thaw(account)` / `thawMany(accounts[])` — remove from blacklist
- `addFreezer(addr)` / `removeFreezer(addr)` — delegate freeze authority
- `isFrozen(account)` — queries `TIP403_REGISTRY.isAuthorized()`
- Uses ERC-7201 namespaced storage

### Tempo Precompile Addresses

| Precompile | Address Pattern | Purpose |
|------------|----------------|---------|
| TIP403_REGISTRY | `0xfeEC...0403` | Transfer policy management (whitelist/blacklist) |
| STABLECOIN_DEX | `0xDEc0...` | Swap between TIP-20 stablecoins |
| TIP_FEE_MANAGER | `0xfeEC...` | User gas token preference |
| TIP20_FACTORY | `0x20Fc...` | Create new TIP-20 tokens |
| PATH_USD (default gas token) | `0x20C0...` | Default whitelisted gas token |

## Frontend Integration Patterns

### Wagmi/Viem Configuration

```typescript
import { tempo } from "viem/chains";

// DO NOT add feeToken to chain config — causes 0x76 MetaMask incompatibility
const tempoChain = {
  ...tempo,
  contracts: {
    multicall3: {
      address: "0xcA11bde05977b3631167028862bE2a173976CA11" as `0x${string}`,
    },
  },
};

// RPC requires Authorization header
const transport = http(tempo.rpcUrls.default.http[0], {
  fetchOptions: {
    headers: {
      Authorization: "Bearer <token>",
    },
  },
});
```

### Token List Entry for frxUSD

The token address in the UI token list MUST be the **TIP-20 address** (for `balanceOf` to work), NOT the OFT adapter:

```typescript
// ✅ Correct — TIP-20 address, balanceOf works
"4217~0x20c0000000000000000000003554d28269e0f3c2": {
  address: "0x20c0000000000000000000003554d28269e0f3c2",
  symbol: "frxUSD",
  // ...
}

// ❌ Wrong — OFT adapter, balanceOf reverts
"4217~0x00000000d61733e7a393a10a5b48c311abe8f1e5": {
  address: "0x00000000d61733e7a393a10a5b48c311abe8f1e5",
  // ...
}
```

### OFT Address Resolution for Bridge Transactions

When constructing `send()` or `quoteSend()` calls, resolve the OFT address via `layerZeroLockBox`:

```typescript
const oftAddressForTx =
  chainId === TEMPO_MAINNET_INFO.id &&
    ALL_TOKENS[symbol]?.layerZeroLockBox?.tempo?.address
    ? ALL_TOKENS[symbol].layerZeroLockBox.tempo.address  // OFT adapter for frxUSD
    : tokenAddress; // Direct OFT for sfrxUSD, frxETH, etc.
```

### Bridge Value Field

Tempo has no native token, so LZ `value` must be `0n`:

```typescript
value: swapTokenA.chainId === TEMPO_MAINNET_INFO.id ? 0n : layerZeroNativeFee,
```

### Gas Token Approval for LZ Fees

On Tempo, LZ messaging fees are paid in frxUSD (ERC20), not native ETH. A separate approval step is required:

```typescript
// Approve the remote hop (or OFT) to spend frxUSD for LZ fees
const tempoGasToken = TEMPO_MAINNET_TOKENS["4217~0x20c0000000000000000000003554d28269e0f3c2"];
const tempoGasSpender = isUsingRemoteHop
  ? REMOTE_HOP_ADDRESSES.tempo
  : swapTokenA.address; // The OFT itself
```

### Native Drop Guard

Tempo as source should never use native drop options (no native token to drop):

```typescript
const options =
  swapTokenA.chainId !== TEMPO_MAINNET_INFO.id &&  // Guard: no native drop from Tempo
  swapTokenB.chainId === FXTL_MAINNET_INFO.id &&
  tokens["252~0x0000000000000000000000000000000000000000"].balanceOf === 0n
    ? Options.newOptions().addExecutorNativeDropOption(...).toHex()
    : "0x";
```

## Viem Tempo SDK

### Client Setup

```typescript
import { createClient, http } from 'viem'
import { tempoModerato } from 'viem/chains'
import { tempoActions } from 'viem/tempo'

const client = createClient({
  chain: tempoModerato,
  transport: http('https://rpc.tempo.xyz'),
}).extend(tempoActions())
```

### Default Fee Token (chain-level)

```typescript
// Set a default feeToken globally via chain extension
const chain = tempoModerato.extend({
  feeToken: '0x20c0000000000000000000000000000000000001',
})
```

### Tempo Transaction Examples

```typescript
// Pay fees with a stablecoin
const receipt = await client.sendTransactionSync({
  data: '0xdeadbeef',
  feeToken: '0x20c0000000000000000000000000000000000001',
  to: '0xcafebabecafebabecafebabecafebabecafebabe',
})

// Batch calls (atomic)
const receipt = await client.sendTransactionSync({
  calls: [
    { to: '0x...', data: '0x...' },
    { to: '0x...', data: '0x...' },
  ],
})

// Fee sponsorship (local account)
const feePayer = privateKeyToAccount('0x...')
const receipt = await client.sendTransactionSync({
  data: '0xdeadbeef',
  feePayer,
  to: '0x...',
})

// Concurrent transactions (parallel nonce lanes)
const [r1, r2] = await Promise.all([
  client.sendTransactionSync({ data: '0x...', to: '0x...' }),
  client.sendTransactionSync({ data: '0x...', to: '0x...' }),
])

// Explicit nonce keys
await client.sendTransaction({ data: '0x...', to: '0x...', nonceKey: 567n })
await client.sendTransaction({ data: '0x...', to: '0x...', nonceKey: 789n })

// Scheduled execution
const receipt = await client.sendTransactionSync({
  data: '0xdeadbeef',
  to: '0x...',
  validAfter: Math.floor(Date.now() / 1000) + 3600,
  validBefore: Math.floor(Date.now() / 1000) + 7200,
})
```

### Fee Payer Transport

```typescript
import { withFeePayer } from 'viem/tempo'

const client = createWalletClient({
  account: privateKeyToAccount('0x...'),
  chain: tempoModerato,
  transport: withFeePayer(
    http(),                                // Default transport
    http('https://sponsor.example.com'),   // Fee payer relay
  ),
})

// Sponsored tx — fee payer relay signs and optionally broadcasts
const receipt = await client.sendTransactionSync({
  feePayer: true,
  to: '0x...',
})
```

Policy options for `withFeePayer`: `'sign-only'` (default — relay co-signs, client broadcasts) or `'sign-and-broadcast'` (relay handles both).

### Tempo Account Types

| Type | Import | Description |
|------|--------|-------------|
| `Account.fromSecp256k1(privateKey)` | `viem/tempo` | Standard Ethereum private key |
| `Account.fromWebAuthnP256(credential)` | `viem/tempo` | WebAuthn passkeys |
| `Account.fromWebCryptoP256(keyPair, opts)` | `viem/tempo` | WebCrypto P256 key pairs (access keys) |
| `Account.fromP256(privateKey)` | `viem/tempo` | Raw P256 private key |

### Access Key Delegation

```typescript
import { Account, WebCryptoP256 } from 'viem/tempo'

const account = Account.fromSecp256k1('0x...')
const keyPair = await WebCryptoP256.createKeyPair()
const accessKey = Account.fromWebCryptoP256(keyPair, { access: account })

const keyAuthorization = await account.signKeyAuthorization(accessKey)

// Attach key auth to first tx, then use access key for all future txs
const receipt = await client.sendTransactionSync({
  account: accessKey,
  keyAuthorization,
  data: '0xdeadbeef',
  to: '0x...',
})
```

### Tempo Actions Catalog (via `tempoActions()`)

| Namespace | Key Actions |
|-----------|-------------|
| `client.token.*` | `getBalance`, `getMetadata`, `transfer`, `approve`, `getAllowance`, `mint`, `burn`, `create`, `pause`, `unpause`, `grantRoles`, `revokeRoles`, `changeTransferPolicy`, `setSupplyCap` |
| `client.fee.*` | `getUserToken`, `setUserToken` |
| `client.dex.*` | `buy`, `sell`, `place`, `placeFlip`, `cancel`, `getBalance`, `getBuyQuote`, `getSellQuote`, `getOrder`, `getTickLevel`, `createPair`, `withdraw` |
| `client.amm.*` | `mint`, `burn`, `getPool`, `getLiquidityBalance`, `rebalanceSwap` |
| `client.nonce.*` | `getNonce` (with nonceKey for concurrent tx lanes) |
| `client.policy.*` | `create`, `getData`, `isAuthorized`, `modifyBlacklist`, `modifyWhitelist`, `setAdmin` |
| `client.reward.*` | `claim`, `distribute`, `getUserRewardInfo`, `setRecipient` |
| `client.validator.*` | `add`, `get`, `getByIndex`, `getCount`, `list`, `update`, `changeOwner`, `changeStatus` |
| `client.accessKey.*` | `authorize`, `revoke`, `getMetadata`, `getRemainingLimit`, `signAuthorization`, `updateLimit` |
| `client.faucet.*` | `fund` (testnet only) |

All write actions have `*Sync` variants that wait for tx inclusion (e.g., `client.token.transferSync`).

### TIP-20 Token Operations

```typescript
// Get balance (native Tempo action)
const balance = await client.token.getBalance({
  account: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEbb',
  token: '0x20c0000000000000000000000000000000000000',
})

// Get metadata
const metadata = await client.token.getMetadata({
  token: '0x20c0000000000000000000000000000000000000',
})
// Returns: { name, symbol, decimals, currency, totalSupply, paused?, quoteToken?, supplyCap?, transferPolicyId? }

// Transfer with optional memo
const { receipt } = await client.token.transferSync({
  amount: parseUnits('10.5', 6),
  to: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEbb',
  token: '0x20c0000000000000000000000000000000000000',
  memo: '0xcafebabe', // optional
})

// Set gas token preference
await client.fee.setUserTokenSync({
  token: '0x20c0000000000000000000000000000000000001',
})

// Read gas token preference
const result = await client.fee.getUserToken({
  account: '0x742d35Cc6634C0532925a3b844Bc9e7595f0bEbb',
})
// Returns: { address, id } | null
```

### Tempo Address Utilities

```typescript
import { TempoAddress } from 'viem/tempo'

TempoAddress.format(address)
TempoAddress.parse(address)
TempoAddress.validate(address)
```

## Wagmi Tempo SDK

### Config Setup

```typescript
import { createConfig, http } from 'wagmi'
import { tempoModerato } from 'wagmi/chains'
import { KeyManager, webAuthn } from 'wagmi/tempo'

export const config = createConfig({
  connectors: [
    webAuthn({
      keyManager: KeyManager.localStorage(), // Use KeyManager.http(url) for production
    }),
  ],
  chains: [tempoModerato],
  multiInjectedProviderDiscovery: false,
  transports: {
    [tempoModerato.id]: http(),
  },
})
```

### Wagmi Connectors

| Connector | Import | Description |
|-----------|--------|-------------|
| `webAuthn({ keyManager })` | `wagmi/tempo` | WebAuthn passkey connector (recommended for Tempo-native apps) |
| `dangerous_secp256k1(privateKey)` | `wagmi/tempo` | Private key connector (dev/testing only) |

### Key Managers (for WebAuthn)

| Manager | Description |
|---------|-------------|
| `KeyManager.localStorage()` | Stores credential→publicKey on client (dev only — lost on clear) |
| `KeyManager.http(url)` | Remote server stores credential→publicKey (production) |

### Wagmi Hooks

```typescript
import { Hooks } from 'wagmi/tempo'

// Read hooks
Hooks.token.useGetBalance({ account, token })
Hooks.token.useGetMetadata({ token })
Hooks.token.useGetAllowance({ owner, spender, token })
Hooks.fee.useUserToken({ account })

// Write hooks
Hooks.token.useApprove()
Hooks.token.useTransfer()
Hooks.token.useMint()
Hooks.token.useBurn()
Hooks.fee.useSetUserToken()

// DEX hooks
Hooks.dex.useBuy()
Hooks.dex.useSell()
Hooks.dex.useBuyQuote({ pair, amount })
Hooks.dex.useSellQuote({ pair, amount })

// AMM hooks
Hooks.amm.useMint()
Hooks.amm.useBurn()
Hooks.amm.usePool({ tokenA, tokenB })
Hooks.amm.useLiquidityBalance({ account, tokenA, tokenB })

// Reward hooks
Hooks.reward.useClaim()
Hooks.reward.useDistribute()
Hooks.reward.useUserRewardInfo({ account, token })

// Standard wagmi hooks with Tempo properties
import { useSendTransactionSync } from 'wagmi'

const sendTx = useSendTransactionSync()
sendTx.mutate({
  calls: [...],
  feePayer: '0x...',
  nonceKey: 1337n,
})
```

## Key Addresses

| Contract | Address |
|----------|---------|
| frxUSD TIP-20 token (precompile) | `0x20C0000000000000000000003554d28269E0f3c2` |
| frxUSD OFT adapter | `0x00000000D61733e7A393A10A5B48c311AbE8f1E5` |
| sfrxUSD OFT | `0x00000000fD8C4B8A413A06821456801295921a71` |
| frxETH OFT | `0x000000008c3930dCA540bB9B3A5D0ee78FcA9A4c` |
| sfrxETH OFT | `0x00000000883279097A49dB1f2af954EAd0C77E3c` |
| WFRAX OFT | `0x00000000E9CE0f293D1Ce552768b187eBA8a56D4` |
| FPI OFT | `0x00000000bC4aEF4bA6363a437455Cb1af19e2aEb` |
| Remote Hop | `0x0000006D38568b00B457580b734e0076C62de659` |
| Multicall3 | `0xcA11bde05977b3631167028862bE2a173976CA11` |

## Deployment

### Deploy OFT on Tempo (Forge)

```bash
forge script scripts/FraxtalHub/1_DeployFraxOFTFraxtalHub/DeployFraxOFTFraxtalHubTempo.s.sol \
  --rpc-url $TEMPO_RPC_URL \
  --gcp \
  --sender 0x54f9b12743a7deec0ea48721683cbebedc6e17bc \
  --broadcast \
  --verify
```

The deployment script (`DeployFraxOFTFraxtalHubTempo.s.sol`):
1. Uses `FraxOFTUpgradeableTempo` implementation for standard OFTs (not `FraxOFTUpgradeable`)
2. For frxUSD: Deploys `FraxOFTMintableAdapterUpgradeableTIP20` adapter wrapping the existing TIP-20 precompile at `0x20C0...3c2`
3. Grants `ISSUER_ROLE` to the adapter on the TIP-20 token (enables mint/burn)
4. Deploys `FrxUSDPolicyAdminTempo` for freeze/thaw (creates BLACKLIST policy in TIP-403 Registry)
5. Transfers `DEFAULT_ADMIN_ROLE` and ownership to multisig, renounces deployer admin
6. Uses `msg.sender` as temporary proxy admin (GCS deployer broadcasts directly)
7. Skips wallet deployment (not needed on Tempo)

## Key Differences from Other L2s

| Feature | Standard L2 (Fraxtal, Arbitrum, etc.) | Tempo |
|---------|---------------------------------------|-------|
| Gas token | Native ETH | ERC-20 stablecoin (TIP-20) |
| TX type | Legacy / EIP-1559 / EIP-4844 | `0x76` (Tempo Transaction) |
| `msg.value` | Used for gas + value transfer | Must be 0 (ETH doesn't exist) |
| OFT base | FraxOFTUpgradeable | FraxOFTUpgradeableTempo |
| frxUSD OFT | FraxOFTMintableAdapterUpgradeable | FraxOFTMintableAdapterUpgradeableTIP20 |
| Gas payment in OFT | `_payNative()` sends ETH | `_payNativeAltToken()` pulls ERC-20, swaps, wraps |
| Batch calls | Not native | `calls` field in tx envelope |
| Fee sponsorship | Not native | `feePayer` field in tx envelope |
| Concurrent nonces | Not supported | `nonceKey` field for parallel tx lanes |
| Account types | secp256k1 only | secp256k1, P256, WebAuthn passkeys |
| Native token ops | ETH transfer | `token.*` actions via precompiles |
| Freeze/compliance | N/A or custom | TIP-403 BLACKLIST policy + FrxUSDPolicyAdminTempo |

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|---------|
| MetaMask "unsupported tx type" / 0x76 error | `feeToken` in chain config triggers Tempo tx type | Remove `feeToken` from wagmi chain config |
| Balance shows spinner / 0 for frxUSD | `balanceOf` called on OFT adapter (not ERC20) | Use TIP-20 address `0x20C0...3c2` for balance |
| `quoteSend` reverts on frxUSD | Calling `quoteSend` on TIP-20 instead of adapter | Resolve OFT address via `layerZeroLockBox.tempo` |
| Bridge tx reverts with value | Sending `msg.value` on Tempo (no native token) | Set `value: 0n` for all Tempo source txs |
| Native drop fails from Tempo | Tempo has no native token to drop | Guard: skip native drop when source is Tempo |
| RPC 401/403 | Tempo RPC requires auth | Add `Authorization: Basic/Bearer <token>` header |
| OFT send reverts "msg_value_not_zero" | Sending ETH value with OFT send | Ensure `msg.value = 0`; LZ fee paid in ERC-20 |

## Best Practices

1. **Always distinguish TIP-20 from OFT adapter** for frxUSD — use TIP-20 for reads, adapter for bridge calls
2. **Never add `feeToken` to wagmi chain config** — it triggers the 0x76 tx type that MetaMask can't handle
3. **Always set `value: 0n`** for bridge transactions originating from Tempo
4. **Use `layerZeroLockBox.tempo`** in master token config for tokens with separate adapter addresses
5. **Test multicall** — Tempo supports multicall3 but individual failing calls won't revert the batch
6. **Handle gas as ERC20** — LZ fees on Tempo require separate token approval, not native ETH
7. **Use `*Sync` actions** for simpler code — they wait for tx inclusion automatically
8. **Use `KeyManager.http()`** in production for WebAuthn — `localStorage` loses keys on clear/device change

## References

- Tempo Protocol Docs: https://docs.tempo.xyz
- Viem Tempo Extension: https://viem.sh/tempo
- Wagmi Tempo Extension: https://wagmi.sh/tempo
- Contract source: `frax-oft-upgradeable/contracts/FraxOFTUpgradeableTempo.sol`
- Contract source: `frax-oft-upgradeable/contracts/FraxOFTMintableAdapterUpgradeableTIP20.sol`
- Contract source: `frax-oft-upgradeable/contracts/base/TempoAltTokenBase.sol`
- Specs + architecture: `frax-oft-upgradeable/contracts/modules/tempo-specs/README.md`
- Deploy script: `frax-oft-upgradeable/scripts/FraxtalHub/1_DeployFraxOFTFraxtalHub/DeployFraxOFTFraxtalHubTempo.s.sol`
- Viem Tempo docs (local): `viem/site/pages/tempo/`
- Wagmi Tempo docs (local): `wagmi/site/tempo/`
