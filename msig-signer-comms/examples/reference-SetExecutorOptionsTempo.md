## Tempo HopV2 msig Signature required

### Fraxtal Msig : https://safe.mainnet.frax.com/transactions/queue?safe=fraxtal:0x5f25218ed9474b721d6a38c115107428E832fA2E
#### #185 - Connect FraxtalHopV2 to RemoteHopV2Tempo
#### #186 — Set Tempo Executor Options on all Remote HopV2 Instances

**Single multisig batch** containing **21 `sendOFT` calls** on Fraxtal HopV2.
Each call relays `setExecutorOptions(30410, opts)` to a remote HopV2 instance via LayerZero compose, setting the Tempo receive gas to **2,500,000**.

##### Key Addresses

| Role | Address |
|------|---------|
| Fraxtal Safe (sender) | `0x5f25218ed9474b721d6a38c115107428E832fA2E` |
| Fraxtal HopV2 (target of all txs) | `0x00000000e18aFc20Afe54d4B2C8688bB60c06B36` |
| frxUSD Lockbox (messaging rail, amount=0) | `0x96A394058E2b84A89bac9667B19661Ed003cF5D4` |
| Default Remote HopV2 | `0x0000006D38568b00B457580b734e0076C62de659` |
| Default RemoteAdmin | `0x954286118E93df807aB6f99aE0454f8710f0a8B9` |

##### Per-Chain Transaction Table

Each tx calls `sendOFT(_token, _dstEid, _to, 0, 400000, composeData)` on Fraxtal HopV2 with native FRAX fee.

| Tx# | Chain | EID | Remote Admin | Fee (FRAX) |
|-----|-------|-----|--------------|-----------|
| 1 | Arbitrum | 30110 | `0x9542...f0a8B9` | 0.2703 |
| 2 | Aurora | 30211 | `0x9542...f0a8B9` | 0.7009 |
| 3 | Avalanche | 30106 | `0x9542...f0a8B9` | 0.1212 |
| 4 | Berachain | 30362 | `0x9542...f0a8B9` | 0.0483 |
| 5 | BSC | 30102 | `0x9542...f0a8B9` | 0.2552 |
| 6 | Hyperliquid | 30367 | `0x9542...f0a8B9` | 0.0596 |
| 7 | Ink | 30339 | `0x9542...f0a8B9` | 0.2473 |
| 8 | Katana | 30375 | `0x9542...f0a8B9` | 0.0540 |
| 9 | Mode | 30260 | `0x9542...f0a8B9` | 0.0786 |
| 10 | Optimism | 30111 | `0x9542...f0a8B9` | 0.1254 |
| 11 | Sei | 30280 | `0x9542...f0a8B9` | 0.0586 |
| 12 | Sonic | 30332 | `0x9542...f0a8B9` | 0.2483 |
| 13 | Unichain | 30320 | `0x9542...f0a8B9` | 0.0571 |
| 14 | Worldchain | 30319 | `0x9542...f0a8B9` | 0.0544 |
| 15 | X-Layer | 30274 | `0x9542...f0a8B9` | 0.0538 |
| 16 | Abstract | 30324 | `0x0E0E...6fdadb` | 0.5594 |
| 17 | Base | 30184 | `0x07dB...5715AA` | 0.1607 |
| 18 | Ethereum | 30101 | `0x181E...390dFa` | 0.5434 |
| 19 | Linea | 30183 | `0xfa80...177447` | 0.8449 |
| 20 | Scroll | 30214 | `0x1dE5...3d01` | 0.9057 |
| 21 | ZkSync | 30165 | `0x0E0E...6fdadb` | 5.0619 |
| | | | **Total** | **~10.51 FRAX** |

##### What Each Transaction Does

Every `sendOFT` call sends a compose message (0 tokens) to the remote chain's HopV2, which executes:

```
setExecutorOptions(
    _eid:     30410   (Tempo)
    _options: 0x01001101000000000000000000000000002625a0
)
```

Options breakdown: `type=1 | len=17 | optionType=1 (lzReceive) | gas=0x2625a0 (2,500,000)`

This sets the executor gas limit for messages **arriving from Tempo** on each remote HopV2 to 2,500,000 gas.

##### Compose Data (same for all 21 txs)

The `_data` param in each `sendOFT` encodes:
```
abi.encode(
    0x0000006D38568b00B457580b734e0076C62de659,   // remote HopV2 address
    abi.encodeCall(IHopV2.setExecutorOptions, (30410, options))
)
```

<details>
<summary>Cast verification commands</summary>

**Decode any tx's outer `sendOFT` call** (example: tx #1 — Arbitrum):
```bash
cast calldata-decode \
  "sendOFT(address,uint32,bytes32,uint256,uint128,bytes)" \
  0x4198dcf400000000000000000000000096a394058e2b84a89bac9667b19661ed003cf5d4000000000000000000000000000000000000000000000000000000000000759e000000000000000000000000954286118e93df807ab6f99ae0454f8710f0a8b900000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000061a8000000000000000000000000000000000000000000000000000000000000000c000000000000000000000000000000000000000000000000000000000000001000000000000000000000000000000006d38568b00b457580b734e0076c62de659000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000845135db4600000000000000000000000000000000000000000000000000000000000076ca0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000001401001101000000000000000000000000002625a000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x96A394058E2b84A89bac9667B19661Ed003cF5D4          ← _token (frxUSD lockbox)
30110                                                 ← _dstEid (Arbitrum)
0x000000000000000000000000954286118e93df807ab...      ← _to (remoteAdmin)
0                                                     ← _amountLD
400000                                                ← _composeGas
0x0000000000000000000000000000006d38568b00b4...       ← _data (compose)
```

**Decode the compose `_data` (same for all txs) — `abi.encode(address hop, bytes remoteCall)`:**
```bash
cast abi-decode --input \
  "f(address,bytes)" \
  0000000000000000000000000000006d38568b00b457580b734e0076c62de659000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000845135db4600000000000000000000000000000000000000000000000000000000000076ca0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000001401001101000000000000000000000000002625a000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x0000006D38568b00B457580b734e0076C62de659   ← remote HopV2 (DEFAULT_HOP)
0x5135db4600000000...                        ← setExecutorOptions(30410, options)
```

**Decode the inner `setExecutorOptions` call:**
```bash
cast calldata-decode \
  "setExecutorOptions(uint32,bytes)" \
  0x5135db4600000000000000000000000000000000000000000000000000000000000076ca0000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000001401001101000000000000000000000000002625a000000000000000000000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
30410                                         ← _eid (Tempo)
0x01001101000000000000000000000000002625a0     ← _options (gas = 2,500,000)
```

</details> 
  * Sim - https://dashboard.tenderly.co/public/safe/safe-apps/simulator/d45b382a-8469-4865-a494-ae229696cac0 

### Tempo Msig : https://app.test.safe.protofire.io/transactions/queue?safe=tempo:0x1Ba19a54a01AE967f5E3895764Caaa6919FD2bEe

> **Tempo Safe credentials**
> - username: `Z7JOjyA2INpoWplGm5mxDEZrJDkNRTHO`
> - password: `pIgsmsZ0U7BkYCn5zLoRm2uHYJguq9Gt`

---

#### Key Addresses (Tempo)

| Role | Address |
|------|---------|
| Tempo Safe (sender) | `0x1Ba19a54a01AE967f5E3895764Caaa6919FD2bEe` |
| RemoteHopV2 (proxy) | `0x0000006D38568b00B457580b734e0076C62de659` |
| ProxyAdmin | `0x000000dbfaA1Fb91ca46867cE6D41aB6da4f7428` |
| LZ EndpointV2 (Tempo) | `0x20Bb7C2E2f4e5ca2B4c57060d1aE2615245dCc9C` |
| Fraxtal HopV2 | `0x00000000e18aFc20Afe54d4B2C8688bB60c06B36` |
| Old (non-deterministic) RemoteAdmin | `0xe764367bf30ef6e66918ea595e2ee621c7ad4a2a` |
| Standard deterministic RemoteAdmin | `0x954286118E93df807aB6f99aE0454f8710f0a8B9` |
| Correct Tempo RemoteAdmin (w/ correct frxUSD OFT) | `0x05b4a311Aac6658C0FA1e0247Be898aae8a8581f` |

---

#### #1 — Revoke DEFAULT_ADMIN_ROLE from old non-deterministic RemoteAdmin

**Target:** RemoteHopV2 (`0x0000006D...de659`)

```
revokeRole(
    role:    0x0000...0000   (DEFAULT_ADMIN_ROLE)
    account: 0xe764367bf30ef6e66918ea595e2ee621c7ad4a2a
)
```

Removes admin access from the old, non-deterministically deployed RemoteAdmin.

<details>
<summary>Cast verification</summary>

```bash
cast calldata-decode \
  "revokeRole(bytes32,address)" \
  0xd547741f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e764367bf30ef6e66918ea595e2ee621c7ad4a2a
```

Expected:
```
0x0000000000000000000000000000000000000000000000000000000000000000   ← DEFAULT_ADMIN_ROLE
0xe764367bf30ef6e66918ea595e2ee621c7ad4a2a                           ← old RemoteAdmin
```
</details>

---

#### #2 — Grant DEFAULT_ADMIN_ROLE to standard deterministic RemoteAdmin

**Target:** RemoteHopV2 (`0x0000006D...de659`)

```
grantRole(
    role:    0x0000...0000   (DEFAULT_ADMIN_ROLE)
    account: 0x954286118E93df807aB6f99aE0454f8710f0a8B9
)
```

Grants admin access to the standard deterministic RemoteAdmin (`0x9542...`). This is the same RemoteAdmin used on all other chains — however it was later discovered that this one was deployed with an incorrect `frxUSDoft` for Tempo (see #4).

<details>
<summary>Cast verification</summary>

```bash
cast calldata-decode \
  "grantRole(bytes32,address)" \
  0x2f2ff15d0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000954286118e93df807ab6f99ae0454f8710f0a8b9
```

Expected:
```
0x0000000000000000000000000000000000000000000000000000000000000000   ← DEFAULT_ADMIN_ROLE
0x954286118E93df807aB6f99aE0454f8710f0a8B9                           ← deterministic RemoteAdmin
```
</details>

---

#### #3 — Upgrade RemoteHopV2Tempo to new implementation + reinitialize

**Target:** ProxyAdmin (`0x000000dbfaA1Fb91ca46867cE6D41aB6da4f7428`)

Calls `upgradeAndCall` on the ProxyAdmin, which upgrades the RemoteHopV2 proxy to a new implementation and calls `initialize` on it.

```
upgradeAndCall(
    proxy:          0x0000006D38568b00B457580b734e0076C62de659   (RemoteHopV2)
    implementation: 0xa4dCb15E851cC4F14498cc9B0158c317672Df7b5   (new RemoteHopV2Tempo)
    data:           initialize(...)
)
```

**Inner `initialize` call:**

```
initialize(
    _localEid:     30410                                           (Tempo)
    _endpoint:     0x20Bb7C2E2f4e5ca2B4c57060d1aE2615245dCc9C     (LZ EndpointV2)
    _fraxtalHop:   0x00000000e18aFc20Afe54d4B2C8688bB60c06B36     (bytes32)
    _numDVNs:      3
    _executor:     0xf851abCa1d0fD1Df8eAba6de466a102996b7d7B2
    _dvn:          0x76FaFF60799021B301B45dC1BbEDE53F261F9961
    _treasury:     0x1deB70e45c2399a4aBEf19E9B1643F2670f892d0
    _approvedOfts: [
        0x00000000D61733e7A393A10A5B48c311AbE8f1E5   (frxUSD)
        0x00000000fD8C4B8A413A06821456801295921a71   (sfrxUSD)
        0x000000008c3930dCA540bB9B3A5D0ee78FcA9A4c   (frxETH)
        0x00000000883279097A49dB1f2af954EAd0C77E3c   (sfrxETH)
        0x00000000E9CE0f293D1Ce552768b187eBA8a56D4   (wFRAX)
        0x00000000bC4aEF4bA6363a437455Cb1af19e2aEb   (FPI)
    ]
)
```

> New implementation is `RemoteHopV2Tempo` — optimized `_sendToDestination` that sets native token as user token and does a single `_payNativeAltToken` to collect/swap. More efficient when users have a gas token not in LZD whitelist.

<details>
<summary>Cast verification</summary>

**Decode outer `upgradeAndCall`:**
```bash
cast calldata-decode \
  "upgradeAndCall(address,address,bytes)" \
  0x9623609d0000000000000000000000000000006d38568b00b457580b734e0076c62de659000000000000000000000000a4dcb15e851cc4f14498cc9b0158c317672df7b5000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000001e4df1cf4a100000000000000000000000000000000000000000000000000000000000076ca00000000000000000000000020bb7c2e2f4e5ca2b4c57060d1ae2615245dcc9c00000000000000000000000000000000e18afc20afe54d4b2c8688bb60c06b360000000000000000000000000000000000000000000000000000000000000003000000000000000000000000f851abca1d0fd1df8eaba6de466a102996b7d7b200000000000000000000000076faff60799021b301b45dc1bbede53f261f99610000000000000000000000001deb70e45c2399a4abef19e9b1643f2670f892d00000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000d61733e7a393a10a5b48c311abe8f1e500000000000000000000000000000000fd8c4b8a413a06821456801295921a71000000000000000000000000000000008c3930dca540bb9b3a5d0ee78fca9a4c00000000000000000000000000000000883279097a49db1f2af954ead0c77e3c00000000000000000000000000000000e9ce0f293d1ce552768b187eba8a56d400000000000000000000000000000000bc4aef4ba6363a437455cb1af19e2aeb00000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x0000006D38568b00B457580b734e0076C62de659   ← proxy (RemoteHopV2)
0xa4dCb15E851cC4F14498cc9B0158c317672Df7b5   ← new implementation
0xdf1cf4a1...                                 ← initialize(...)
```

**Decode inner `initialize`:**
```bash
cast calldata-decode \
  "initialize(uint32,address,bytes32,uint32,address,address,address,address[])" \
  0xdf1cf4a100000000000000000000000000000000000000000000000000000000000076ca00000000000000000000000020bb7c2e2f4e5ca2b4c57060d1ae2615245dcc9c00000000000000000000000000000000e18afc20afe54d4b2c8688bb60c06b360000000000000000000000000000000000000000000000000000000000000003000000000000000000000000f851abca1d0fd1df8eaba6de466a102996b7d7b200000000000000000000000076faff60799021b301b45dc1bbede53f261f99610000000000000000000000001deb70e45c2399a4abef19e9b1643f2670f892d00000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000d61733e7a393a10a5b48c311abe8f1e500000000000000000000000000000000fd8c4b8a413a06821456801295921a71000000000000000000000000000000008c3930dca540bb9b3a5d0ee78fca9a4c00000000000000000000000000000000883279097a49db1f2af954ead0c77e3c00000000000000000000000000000000e9ce0f293d1ce552768b187eba8a56d400000000000000000000000000000000bc4aef4ba6363a437455cb1af19e2aeb
```

Expected:
```
30410                                                             ← _localEid (Tempo)
0x20Bb7C2E2f4e5ca2B4c57060d1aE2615245dCc9C                       ← _endpoint
0x00000000000000000000000000000000e18afc20afe54d4b2c8688bb60c06b36 ← _fraxtalHop
3                                                                 ← _numDVNs
0xf851abCa1d0fD1Df8eAba6de466a102996b7d7B2                       ← _executor
0x76FaFF60799021B301B45dC1BbEDE53F261F9961                       ← _dvn
0x1deB70e45c2399a4aBEf19E9B1643F2670f892d0                       ← _treasury
[frxUSD, sfrxUSD, frxETH, sfrxETH, wFRAX, FPI]                   ← _approvedOfts
```
</details>

---

#### #4 — Fix RemoteAdmin: revoke `0x9542...` (wrong frxUSD OFT), grant `0x05b4...` (correct)

**Target:** RemoteHopV2 (`0x0000006D...de659`) via Safe `multiSend`

The deterministic RemoteAdmin at `0x9542...` (granted in #2) was deployed with an incorrect `frxUSDoft` address. Tempo's frxUSD OFT address differs from other chains. This `multiSend` atomically:

1. **`grantRole(DEFAULT_ADMIN_ROLE, 0x05b4a311Aac6658C0FA1e0247Be898aae8a8581f)`** — grant to new RemoteAdmin (deployed with correct Tempo frxUSD OFT)
2. **`revokeRole(DEFAULT_ADMIN_ROLE, 0x954286118E93df807aB6f99aE0454f8710f0a8B9)`** — revoke from old RemoteAdmin (wrong frxUSD OFT)

Both calls target RemoteHopV2 at `0x0000006D38568b00B457580b734e0076C62de659`.

> After #4, only `0x05b4...` holds DEFAULT_ADMIN_ROLE on RemoteHopV2Tempo.

<details>
<summary>Cast verification</summary>

**Decode outer `multiSend`:**
```bash
cast calldata-decode \
  "multiSend(bytes)" \
  0x8d80ff0a00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000132000000006d38568b00b457580b734e0076c62de659000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000442f2ff15d000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005b4a311aac6658c0fa1e0247be898aae8a8581f000000006d38568b00b457580b734e0076c62de65900000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000044d547741f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000954286118e93df807ab6f99ae0454f8710f0a8b90000000000000000000000000000
```

**Inner call 1 — `grantRole`:**
```bash
cast calldata-decode \
  "grantRole(bytes32,address)" \
  0x2f2ff15d000000000000000000000000000000000000000000000000000000000000000000000000000000000000000005b4a311aac6658c0fa1e0247be898aae8a8581f
```

Expected:
```
0x0000...0000                                 ← DEFAULT_ADMIN_ROLE
0x05b4a311Aac6658C0FA1e0247Be898aae8a8581f    ← correct Tempo RemoteAdmin
```

**Inner call 2 — `revokeRole`:**
```bash
cast calldata-decode \
  "revokeRole(bytes32,address)" \
  0xd547741f0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000954286118e93df807ab6f99ae0454f8710f0a8b9
```

Expected:
```
0x0000...0000                                 ← DEFAULT_ADMIN_ROLE
0x954286118E93df807aB6f99aE0454f8710f0a8B9    ← old RemoteAdmin (wrong frxUSD)
```
</details>

#### #5 - set executor options for Somnia (30380). Options : `0x01001101000000000000000000000000000f4240` . 

(0F4240)16 = (1000000)10

