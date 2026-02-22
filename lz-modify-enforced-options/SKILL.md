# Updating LayerZero Enforced Options

## Overview

Update `enforcedOptions` on source chain OFTs/lockboxes to modify the minimum gas required for `lzReceive` execution on destination chains.

## When to Use

- Destination chain gas costs change (e.g., SSTORE repricing)
- Cross-chain transactions failing due to insufficient gas
- Proactive adjustment for upcoming chain upgrades
- Adding new destination chain pathways

## Prerequisites

- Foundry (`forge`, `cast`)
- Access to source chain RPC
- Multisig signer access (for execution)

## Procedure

### 1. Identify Impact

Determine which source→destination pathways are affected:

```bash
# Check current enforcedOptions for a specific OFT/lockbox
cast call <OFT_ADDRESS> "enforcedOptions(uint32,uint16)(bytes)" <DEST_EID> <MSG_TYPE> --rpc-url <RPC>
```

**Message Types:**
- `MSG_TYPE=1`: SEND (standard token transfer)
- `MSG_TYPE=2`: SEND_AND_CALL (transfer + compose execution)

### 2. Determine New Gas Values

Calculate appropriate gas limits for the destination chain:

| Message Type | Description | Typical Range |
|--------------|-------------|---------------|
| SEND (1) | Standard lzReceive execution | 100k - 300k |
| SEND_AND_CALL (2) | lzReceive + compose callback | 150k - 500k |

**Factors to consider:**
- Destination chain opcode costs (SSTORE, SLOAD, etc.)
- Token contract complexity
- Compose callback gas requirements
- Safety margin (recommend 20-50% buffer)

### 3. Create Fix Script

Create a script in `scripts/ops/fix/<FixName>/` that inherits your deployment base:

```solidity
// SPDX-License-Identifier: ISC
pragma solidity ^0.8.19;

import "scripts/deploy/DeployOFTProtocol.s.sol";

/**
 * @title Fix<Name>EnforcedOptions
 * @notice Updates enforcedOptions for <SOURCE> → <DEST> pathway
 * @dev Reason: <DOCUMENT WHY THIS CHANGE IS NEEDED>
 *      - Previous: SEND=Xk, SEND_AND_CALL=Yk
 *      - New: SEND=Ak, SEND_AND_CALL=Bk
 *      - Cause: <e.g., gas repricing, new chain requirements>
 */
contract Fix<Name>EnforcedOptions is DeployOFTProtocol {
    using OptionsBuilder for bytes;

    uint256 constant TARGET_EID = <DEST_EID>;
    L0Config[] targetConfigArray;

    /// @notice Override with adjusted gas limits
    function setEvmEnforcedOptions(
        address[] memory _connectedOfts,
        L0Config[] memory _configs
    ) public override {
        bytes memory optionsTypeOne = OptionsBuilder
            .newOptions()
            .addExecutorLzReceiveOption(<NEW_SEND_GAS>, 0);
        bytes memory optionsTypeTwo = OptionsBuilder
            .newOptions()
            .addExecutorLzReceiveOption(<NEW_SEND_AND_CALL_GAS>, 0);

        setEnforcedOptions({
            _connectedOfts: _connectedOfts,
            _configs: _configs,
            _optionsTypeOne: optionsTypeOne,
            _optionsTypeTwo: optionsTypeTwo
        });
    }

    function filename() public view override returns (string memory) {
        return string.concat(
            vm.projectRoot(),
            "/scripts/ops/fix/<FixName>/txs/<ScriptName>.json"
        );
    }

    function run() public override simulateAndWriteTxs(broadcastConfig) {
        // Set up source OFTs/lockboxes to update
        delete proxyOfts;
        proxyOfts.push(<OFT_OR_LOCKBOX_1>);
        proxyOfts.push(<OFT_OR_LOCKBOX_2>);
        // ... add all affected contracts

        // Find target chain config
        for (uint256 i = 0; i < proxyConfigs.length; i++) {
            if (proxyConfigs[i].eid == TARGET_EID) {
                targetConfigArray.push(proxyConfigs[i]);
            }
        }
        require(targetConfigArray.length == 1, "Config not found");

        setEvmEnforcedOptions({
            _connectedOfts: proxyOfts,
            _configs: targetConfigArray
        });
    }
}
```

### 4. Generate Multisig JSON

```bash
forge script scripts/ops/fix/<FixName>/<Script>.s.sol \
    --rpc-url <SOURCE_RPC> \
    --ffi
```

Output: `scripts/ops/fix/<FixName>/txs/<ScriptName>.json`

### 5. Import to Safe

1. Go to Safe Transaction Builder
2. Import the generated JSON
3. Review transactions
4. Simulate before signing

---

## Verification

### Verify Calldata Encoding

```bash
cast decode-calldata "setEnforcedOptions((uint32,uint16,bytes)[])" <CALLDATA>
```

Expected output format:
```
[(eid, msgType, options_bytes), ...]
```

### Decode Gas from Options Bytes

The gas value is encoded in the options bytes. For type 3 executor lzReceive options:
```
0x00030100110100000000000000000000000000<GAS_HEX>
```

Decode the hex gas value:
```bash
cast to-dec 0x<GAS_HEX>
```

### Simulate on Tenderly

1. Use Safe's Tenderly integration to simulate
2. Verify state changes show storage updates for `enforcedOptions` mapping
3. Confirm correct number of contracts affected (2 slots per contract: SEND + SEND_AND_CALL)

### Post-Execution Verification

```bash
# Confirm new values are set
cast call <OFT_ADDRESS> "enforcedOptions(uint32,uint16)(bytes)" <DEST_EID> 1 --rpc-url <RPC>
cast call <OFT_ADDRESS> "enforcedOptions(uint32,uint16)(bytes)" <DEST_EID> 2 --rpc-url <RPC>
```

### Bridge Testing

**Pre-upgrade testing (before destination chain upgrade takes effect):**
1. Execute multisig transaction to update enforcedOptions
2. Test bridging in both directions (source→dest and dest→source)
3. Verify transactions complete on [LayerZeroScan](https://layerzeroscan.com)
4. Document test transaction hashes

**Post-upgrade testing (after destination chain upgrade is live):**
1. Re-test bridging after the upgrade takes effect
2. Confirm transactions complete successfully with new gas costs
3. Document final verification transaction hashes

> **Important:** If the update is for an upcoming chain upgrade, the change can be deployed proactively but must be re-tested after the upgrade goes live to confirm the new gas limits are sufficient.

---

## Reference

### Common LayerZero V2 EIDs

| Chain | EID |
|-------|-----|
| Ethereum | 30101 |
| BSC | 30102 |
| Avalanche | 30106 |
| Polygon | 30109 |
| Arbitrum | 30110 |
| Optimism | 30111 |
| Fantom | 30112 |
| Base | 30184 |
| Linea | 30183 |
| Scroll | 30214 |
| zkSync Era | 30165 |
| Mantle | 30181 |
| Blast | 30243 |

> Full EID list: https://docs.layerzero.network/v2/developers/evm/technical-reference/deployed-contracts

### Options Type 3 Format

```
0x0003        - Option type 3
  0100        - Executor option
    11        - lzReceive option type
      01      - Has params
        <128-bit gas limit, hex encoded>
```

**Example:** 200,000 gas = `0x30d40` → Full options: `0x00030100110100000000000000000000000000030d40`

### Storage Layout

`enforcedOptions` uses ERC-7201 namespaced storage:
```
keccak256("layerzerov2.storage.oappoptionstype3") - 1
= 0x8d2bda5d9f6ffb5796910376005392955773acee5548d0fcdb10e7c264ea0000
```

Mapping slots: `keccak256(msgType . keccak256(eid . baseSlot))`

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| Transaction reverts with "LZ_UnsupportedEid" | Invalid destination EID | Verify EID in LayerZero docs |
| Gas estimation fails | Options bytes malformed | Check OptionsBuilder encoding |
| Bridge tx stuck "Inflight" | Insufficient gas for lzReceive | Increase gas limits |
| Simulation shows no state changes | Wrong contract addresses | Verify OFT/lockbox addresses |

---

## Best Practices

1. **Document everything**: Include reason for change in script comments and PR description
2. **Test both directions**: Bridge source→dest and dest→source after changes
3. **Use safety margins**: Add 20-50% buffer above minimum required gas
4. **Proactive updates**: Deploy before chain upgrades when possible
5. **Monitor transactions**: Track on LayerZeroScan after deployment
6. **Keep records**: Document test transaction hashes for audit trail
