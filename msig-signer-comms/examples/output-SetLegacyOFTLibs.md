## SetLegacyOFTLibs — Msig Signatures Required

Explicitly set `setSendLibrary` and `setReceiveLibrary` on legacy (non-upgradeable) Frax OFTs across **Metis**, **Blast**, **Base**, and **Ethereum**. These legacy OFTs were previously relying on the LZ endpoint default library and need explicit library assignments to continue operating properly.

All calls target the **LZ EndpointV2** contract (`0x1a44076050125825900e736c501f859c50fE728c`) — same address on all chains.

---

### Key Addresses (Common Across All Chains)

| Role | Address |
|------|---------|
| LZ EndpointV2 (all chains) | `0x1a44076050125825900e736c501f859c50fE728c` |
| FRAX legacy OFT | `0x23432452b720c80553458496d4d9d7c5003280d0` |
| frxETH legacy OFT | `0xf010a7c8877043681d59ad125ebf575633505942` |
| sfrxETH legacy OFT | `0x1f55a02a049033e3419a8e2975cf3f572f4e6e9a` |
| FPI legacy OFT | `0x6eca253b102d41b6b69ac815b9cc6bd47ef1979d` |

> The FRAX, frxETH, sfrxETH, and FPI legacy OFT addresses are identical on Metis, Blast, Base, and Ethereum (deterministic deployments).

---

### Route EID Reference

| Chain | LZ EID | Hex |
|-------|--------|-----|
| Ethereum | 30101 | `0x7595` |
| Metis | 30151 | `0x75C7` |
| Blast | 30243 | `0x7623` |
| Base | 30184 | `0x75E8` |

---

## 1. Metis Msig

### Metis Msig : https://metissafe.tech/transactions/queue?safe=metis-andromeda:0xF4A4F32732F9B2fB84Ee28c58616946F3bF80F7d

#### Key Addresses (Metis)

| Role | Address |
|------|---------|
| Metis Safe (signer) | `0xF4A4F32732F9B2fB84Ee28c58616946F3bF80F7d` |
| LZ SendUln302 (Metis) | `0x63e39ccb510926d05a0ae7817c8f1cc61c5bdd6c` |
| LZ ReceiveUln302 (Metis) | `0x5539eb17a84e1d59d37c222eb2cc4c81b502d1ac` |

#### Batches (4 separate Safe uploads)

| File | OFT | Tx Count | Routes Set |
|------|-----|----------|------------|
| `1778202385-SetLegacyOFTLibs-1088-FRAX(FXS).json` | FRAX | 6 | → Ethereum, Blast, Base |
| `1778202385-SetLegacyOFTLibs-1088-frxETH.json` | frxETH | 6 | → Ethereum, Blast, Base |
| `1778202385-SetLegacyOFTLibs-1088-sfrxETH.json` | sfrxETH | 6 | → Ethereum, Blast, Base |
| `1778202409-SetLegacyOFTLibs-1088-FPI.json` | FPI | 6 | → Ethereum, Blast, Base |

#### What Each Batch Does

Each batch sets **send and receive libraries** for one OFT across all 3 outbound routes.

For each OFT × each destination EID (30101, 30243, 30184):

```
setSendLibrary(
    _oapp:   <OFT address>
    _eid:    <destination EID>
    _newLib: 0x63e39ccb510926d05a0ae7817c8f1cc61c5bdd6c   ← Metis SendUln302
)

setReceiveLibrary(
    _oapp:        <OFT address>
    _eid:         <destination EID>
    _newLib:      0x5539eb17a84e1d59d37c222eb2cc4c81b502d1ac  ← Metis ReceiveUln302
    _gracePeriod: 0
)
```

<details>
<summary>Cast verification — Metis FRAX batch (example txs 1 & 2 — Ethereum route)</summary>

**Decode tx #1 — `setSendLibrary`:**
```bash
cast calldata-decode \
  "setSendLibrary(address,uint32,address)" \
  0x9535ff3000000000000000000000000023432452b720c80553458496d4d9d7c5003280d0000000000000000000000000000000000000000000000000000000000000759500000000000000000000000063e39ccb510926d05a0ae7817c8f1cc61c5bdd6c
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0x63e39ccb510926d05a0ae7817c8f1cc61c5bdd6c   ← _newLib (Metis SendUln302)
```

**Decode tx #2 — `setReceiveLibrary`:**
```bash
cast calldata-decode \
  "setReceiveLibrary(address,uint32,address,uint256)" \
  0x6a14d71500000000000000000000000023432452b720c80553458496d4d9d7c5003280d000000000000000000000000000000000000000000000000000000000000075950000000000000000000000005539eb17a84e1d59d37c222eb2cc4c81b502d1ac0000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0x5539eb17a84e1d59d37c222eb2cc4c81b502d1ac   ← _newLib (Metis ReceiveUln302)
0                                             ← _gracePeriod
```
</details>

---

## 2. Blast Msig

### Blast Msig : https://console.brahma.fi/account/0x33a133020b2c2cd41a24f74033b11ec2fc0bf97a

#### Key Addresses (Blast)

| Role | Address |
|------|---------|
| Blast Safe (signer) | `0x33a133020b2c2cd41a24f74033b11ec2fc0bf97a` |
| LZ SendUln302 (Blast) | `0xc1b621b18187f74c8f6d52a6f709dd2780c09821` |
| LZ ReceiveUln302 (Blast) | `0x377530cda84dfb2673bf4d145dcf0c4d7fdcb5b6` |

#### Batches (4 separate Safe uploads)

| File | OFT | Tx Count | Routes Set |
|------|-----|----------|------------|
| `1778202403-SetLegacyOFTLibs-81457-FRAX(FXS).json` | FRAX | 6 | → Ethereum, Metis, Base |
| `1778202407-SetLegacyOFTLibs-81457-sfrxETH.json` | sfrxETH | 6 | → Ethereum, Metis, Base |
| `1778202409-SetLegacyOFTLibs-81457-frxETH.json` | frxETH | 6 | → Ethereum, Metis, Base |
| `1778202423-SetLegacyOFTLibs-81457-FPI.json` | FPI | 6 | → Ethereum, Metis, Base |

#### What Each Batch Does

For each OFT × each destination EID (30101, 30151, 30184):

```
setSendLibrary(
    _oapp:   <OFT address>
    _eid:    <destination EID>
    _newLib: 0xc1b621b18187f74c8f6d52a6f709dd2780c09821   ← Blast SendUln302
)

setReceiveLibrary(
    _oapp:        <OFT address>
    _eid:         <destination EID>
    _newLib:      0x377530cda84dfb2673bf4d145dcf0c4d7fdcb5b6  ← Blast ReceiveUln302
    _gracePeriod: 0
)
```

<details>
<summary>Cast verification — Blast FRAX batch (example txs 1 & 2 — Ethereum route)</summary>

**Decode tx #1 — `setSendLibrary`:**
```bash
cast calldata-decode \
  "setSendLibrary(address,uint32,address)" \
  0x9535ff3000000000000000000000000023432452b720c80553458496d4d9d7c5003280d00000000000000000000000000000000000000000000000000000000000007595000000000000000000000000c1b621b18187f74c8f6d52a6f709dd2780c09821
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0xc1b621b18187f74c8f6d52a6f709dd2780c09821   ← _newLib (Blast SendUln302)
```

**Decode tx #2 — `setReceiveLibrary`:**
```bash
cast calldata-decode \
  "setReceiveLibrary(address,uint32,address,uint256)" \
  0x6a14d71500000000000000000000000023432452b720c80553458496d4d9d7c5003280d00000000000000000000000000000000000000000000000000000000000007595000000000000000000000000377530cda84dfb2673bf4d145dcf0c4d7fdcb5b60000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0x377530cda84dfb2673bf4d145dcf0c4d7fdcb5b6   ← _newLib (Blast ReceiveUln302)
0                                             ← _gracePeriod
```
</details>

---

## 3. Base Msig

### Base Msig : https://app.safe.global/transactions/queue?safe=base:0xCBfd4Ef00a8cf91Fd1e1Fe97dC05910772c15E53

#### Key Addresses (Base)

| Role | Address |
|------|---------|
| Base Safe (signer) | `0xCBfd4Ef00a8cf91Fd1e1Fe97dC05910772c15E53` |
| LZ SendUln302 (Base) | `0xb5320b0b3a13cc860893e2bd79fcd7e13484dda2` |
| LZ ReceiveUln302 (Base) | `0xc70ab6f32772f59fbfc23889caf4ba3376c84baf` |

#### Batches (4 separate Safe uploads)

| File | OFT | Tx Count | Routes Set |
|------|-----|----------|------------|
| `1778202405-SetLegacyOFTLibs-8453-FRAX(FXS).json` | FRAX | 6 | → Ethereum, Metis, Blast |
| `1778202423-SetLegacyOFTLibs-8453-sfrxETH.json` | sfrxETH | 6 | → Ethereum, Metis, Blast |
| `1778202429-SetLegacyOFTLibs-8453-frxETH.json` | frxETH | 6 | → Ethereum, Metis, Blast |
| `1778202433-SetLegacyOFTLibs-8453-FPI.json` | FPI | 6 | → Ethereum, Metis, Blast |

#### What Each Batch Does

For each OFT × each destination EID (30101, 30151, 30243):

```
setSendLibrary(
    _oapp:   <OFT address>
    _eid:    <destination EID>
    _newLib: 0xb5320b0b3a13cc860893e2bd79fcd7e13484dda2   ← Base SendUln302
)

setReceiveLibrary(
    _oapp:        <OFT address>
    _eid:         <destination EID>
    _newLib:      0xc70ab6f32772f59fbfc23889caf4ba3376c84baf  ← Base ReceiveUln302
    _gracePeriod: 0
)
```

<details>
<summary>Cast verification — Base FRAX batch (example txs 1 & 2 — Ethereum route)</summary>

**Decode tx #1 — `setSendLibrary`:**
```bash
cast calldata-decode \
  "setSendLibrary(address,uint32,address)" \
  0x9535ff3000000000000000000000000023432452b720c80553458496d4d9d7c5003280d00000000000000000000000000000000000000000000000000000000000007595000000000000000000000000b5320b0b3a13cc860893e2bd79fcd7e13484dda2
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0xb5320b0b3a13cc860893e2bd79fcd7e13484dda2   ← _newLib (Base SendUln302)
```

**Decode tx #2 — `setReceiveLibrary`:**
```bash
cast calldata-decode \
  "setReceiveLibrary(address,uint32,address,uint256)" \
  0x6a14d71500000000000000000000000023432452b720c80553458496d4d9d7c5003280d00000000000000000000000000000000000000000000000000000000000007595000000000000000000000000c70ab6f32772f59fbfc23889caf4ba3376c84baf0000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy OFT)
30101                                         ← _eid (Ethereum)
0xc70ab6f32772f59fbfc23889caf4ba3376c84baf   ← _newLib (Base ReceiveUln302)
0                                             ← _gracePeriod
```
</details>

---

## 4. Ethereum Msig

### Ethereum Msig : https://app.safe.global/transactions/queue?safe=eth:0xB1748C79709f4Ba2Dd82834B8c82D4a505003f27

#### Key Addresses (Ethereum)

| Role | Address |
|------|---------|
| Ethereum Safe (signer) | `0xB1748C79709f4Ba2Dd82834B8c82D4a505003f27` |
| LZ ReceiveUln302 (Ethereum) | `0xc02ab410f0734efa3f14628780e6e695156024c2` |
| sFRAX legacy lockbox | `0xe4796ccb6bb5de2290c417ac337f2b66ca2e770e` |
| LFRAX legacy lockbox | `0x909dbde1ebe906af95660033e478d59efe831fed` |

> **Note:** Ethereum lockboxes only require `setReceiveLibrary` calls — no `setSendLibrary` needed on this side.

#### Batches (6 separate Safe uploads)

| File | OFT / Lockbox | Tx Count | Routes Set |
|------|---------------|----------|------------|
| `1778204195-SetLegacyOFTLibs-1-FRAX(FXS).json` | FRAX lockbox | 3 | ← from Metis, Blast, Base |
| `1778204195-SetLegacyOFTLibs-1-sFRAX.json` | sFRAX lockbox | 3 | ← from Metis, Blast, Base |
| `1778204207-SetLegacyOFTLibs-1-FPI.json` | FPI lockbox | 3 | ← from Metis, Blast, Base |
| `1778204207-SetLegacyOFTLibs-1-LFRAX.json` | LFRAX lockbox | 3 | ← from Metis, Blast, Base |
| `1778204207-SetLegacyOFTLibs-1-frxETH.json` | frxETH lockbox | 3 | ← from Metis, Blast, Base |
| `1778204207-SetLegacyOFTLibs-1-sfrxETH.json` | sfrxETH lockbox | 3 | ← from Metis, Blast, Base |

#### What Each Batch Does

For each lockbox × each inbound source EID (30151, 30243, 30184):

```
setReceiveLibrary(
    _oapp:        <lockbox address>
    _eid:         <source EID>
    _newLib:      0xc02ab410f0734efa3f14628780e6e695156024c2  ← Ethereum ReceiveUln302
    _gracePeriod: 0
)
```

<details>
<summary>Cast verification — Ethereum FRAX batch (example tx #1 — Metis source)</summary>

**Decode tx #1 — `setReceiveLibrary`:**
```bash
cast calldata-decode \
  "setReceiveLibrary(address,uint32,address,uint256)" \
  0x6a14d71500000000000000000000000023432452b720c80553458496d4d9d7c5003280d000000000000000000000000000000000000000000000000000000000000075c7000000000000000000000000c02ab410f0734efa3f14628780e6e695156024c20000000000000000000000000000000000000000000000000000000000000000
```

Expected:
```
0x23432452b720c80553458496d4d9d7c5003280d0   ← _oapp (FRAX legacy lockbox)
30151                                         ← _eid (Metis)
0xc02ab410f0734efa3f14628780e6e695156024c2   ← _newLib (Ethereum ReceiveUln302)
0                                             ← _gracePeriod
```
</details>

---

## Summary

| Chain | Msig | Batches | Total Txs |
|-------|------|---------|-----------|
| Metis | `0xF4A4F32...F7d` | 4 | 24 |
| Blast | `0x33a133...97a` | 4 | 24 |
| Base | `0xCBfd4E...E53` | 4 | 24 |
| Ethereum | `0xB1748C...f27` | 6 | 18 |
| **Total** | | **18 batches** | **90 txs** |

Each batch file is a self-contained Gnosis Safe JSON upload with `value: "0"` on all transactions (no ETH required).
