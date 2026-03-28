# Pump Function Registry — All 14 Driver Capabilities

> Quick reference for every pump driver's capabilities, limits, and quirks.
> Source: `core/data/pump/defs/PumpType.kt` + individual driver implementations.

## Capability Matrix

### Insulin Delivery Capabilities

| Driver | Bolus Step | Extended Bolus | TBR Type | TBR Max | Basal Step | Basal Min | Patch Pump |
|--------|-----------|---------------|----------|---------|------------|-----------|------------|
| **Virtual** | 0.1 U | Yes | Percent | 500% | 0.01 U/h | 0.01 | No |
| **Dana R** | 0.05 U | Yes | Percent | 200% | 0.01 U/h | 0.04 | No |
| **Dana RS/i** | 0.05 U | Yes | Percent | 200% | 0.01 U/h | 0.04 | No |
| **Omnipod Eros** | 0.05 U | Yes | Absolute | 30 U/h | 0.05 U/h | 0.05 | Yes |
| **Omnipod Dash** | 0.05 U | Yes | Absolute | 30 U/h | 0.05 U/h | 0.05 | Yes |
| **Medtronic 512-754** | 0.1 U | No (TBR comp.) | Absolute | 35 U/h | 0.05 U/h | 0.05 | No |
| **Medtronic 523-Veo** | 0.05 U | No (TBR comp.) | Absolute | 35 U/h | 0.025 U/h | 0.025 | No |
| **Combo (Accu-Chek)** | 0.1 U | Yes | Percent | 500% | 0.01 U/h | 0.01 | No |
| **Insight** | 0.01 U | Yes | Percent | 250% | 0.01 U/h | 0.02 | No |
| **Diaconn G8** | 0.01 U | Yes | Absolute | 15 U/h | 0.01 U/h | 0.05 | No |
| **EOPatch2** | 0.05 U | Yes | Absolute | 15 U/h | 0.05 U/h | 0.05 | Yes |
| **Medtrum Nano** | 0.05 U | Yes | Absolute | 25 U/h | 0.05 U/h | 0.05 | Yes |
| **Medtrum 300U** | 0.05 U | Yes | Absolute | 30 U/h | 0.05 U/h | 0.05 | Yes |
| **Equil** | 0.05 U | Yes | Absolute | 35 U/h | 0.01 U/h | 0.05 | No |

### Communication & Connection

| Driver | Connection Type | Protocol | Hardware Link | Battery Report | History Support |
|--------|----------------|----------|---------------|----------------|-----------------|
| **Virtual** | None | N/A | No | Simulated | No |
| **Dana R** | BLE (GATT) | Dana serial | No | Yes | Yes |
| **Dana RS/i** | BLE (GATT) | Dana BLE | No | Yes | Yes (with events) |
| **Omnipod Eros** | BLE → RileyLink → RF | RF 433MHz | Yes (RileyLink) | No | Limited |
| **Omnipod Dash** | BLE (GATT) | Dash BLE | No | No | Limited |
| **Medtronic** | BLE → RileyLink → RF | RF 916MHz | Yes (RileyLink) | Yes | Yes |
| **Combo** | BLE (RFCOMM) | Serial/Display | No | No (external AA) | Via display parsing |
| **Insight** | BLE (custom) | Insight protocol | No | Yes | Yes |
| **Diaconn G8** | BLE (GATT) | Diaconn BLE | No | Yes | Yes |
| **EOPatch2** | BLE (GATT) | EOPatch BLE | No | No | Limited |
| **Medtrum Nano/300U** | BLE (GATT) | Medtrum BLE | No | No | Yes |
| **Equil** | BLE (GATT) | Equil encrypted | Yes | No | Limited |

### TBR Duration Capabilities

| Driver | 15-min TBR | 30-min TBR | Max TBR Duration |
|--------|-----------|-----------|-------------------|
| **Virtual** | Yes | Yes | 24h |
| **Dana R/RS/i** | Yes | Yes | 24h |
| **Omnipod Eros/Dash** | No | Yes | 12h |
| **Medtronic** | No | Yes | 24h |
| **Combo** | Yes | Yes | 12h |
| **Insight** | Yes | Yes | 24h |
| **Diaconn G8** | No | Yes | 24h |
| **EOPatch2** | No | Yes | 12h |
| **Medtrum** | No | Yes | 12h |
| **Equil** | No | Yes | 24h |

## Per-Driver Details

### Dana RS/i (pump/danars/)
- **Manufacturer**: Sooil
- **Architecture**: Plugin → DanaRSService (bound service) → BLE GATT
- **Key Feature**: Full history read capability, bidirectional sync
- **DST Handling**: Cannot handle DST automatically
- **Max Reservoir**: 300 U
- **Fakeing TBR by Extended**: Supported (to bypass 200% limit)
- **Known Issues**: #4488 — serial number display regression in 3.4

### Omnipod Dash (pump/omnipod-dash/)
- **Manufacturer**: Insulet
- **Architecture**: Plugin → OmnipodDashManager → BLE GATT
- **Key Feature**: Patch pump, no external hardware needed
- **DST Handling**: Yes (pod handles internally)
- **Max Reservoir**: 200 U (50 reported to AAPS)
- **Known Issues**: #37 — Android 16 BLE connectivity, #4522 — PUMP UNREACHABLE
- **Bond Setting**: Must be OFF on Android 16
- **Quirks**: No battery level reporting, fake TBR when pod deactivated

### Omnipod Eros (pump/omnipod-eros/)
- **Manufacturer**: Insulet
- **Architecture**: Plugin → RileyLink Service → RF 433MHz → Pod
- **Key Feature**: Uses RileyLink hardware relay
- **Requires**: RileyLink, OrangeLink, or EmaLink device
- **Known Issues**: RileyLink battery level reporting unreliable with NiMH

### Medtronic (pump/medtronic/)
- **Manufacturer**: Medtronic
- **Architecture**: Plugin → RileyLink Service → RF 916MHz → Pump
- **Key Feature**: History read from pump memory, TDD support
- **Models**: 512/712, 515/715, 522/722, 523/723 Revel, 554/754 Veo
- **Known Issues**: No bolus progress reporting, DST handling varies
- **Quirks**: Dynamic history generation (not stored like other pumps)

### Accu-Chek Combo (pump/combov2/)
- **Manufacturer**: Roche
- **Architecture**: Plugin → comboctl library → BLE RFCOMM → Pump
- **Key Feature**: Controls pump by parsing its LCD display
- **Complexity**: Highest in codebase (3,594 lines in main Pump.kt)
- **Known Issues**: #32 — stuck in Error state, #4479 — permanent error after W6/occlusion
- **Quirks**: Uses RFCOMM (not GATT), display pattern matching, no battery reporting

### Insight (pump/insight/)
- **Manufacturer**: Roche
- **Architecture**: Plugin → Custom BLE protocol
- **Key Feature**: Fine-grained control (0.01 U bolus step)
- **Known Issues**: Complex connection handling

### Diaconn G8 (pump/diaconn/)
- **Manufacturer**: G2e
- **Architecture**: Plugin → BLE GATT
- **Key Feature**: Finest bolus step (0.01 U), absolute TBR
- **Max Basal**: 3.0 U/h (lower than most)

### EOPatch2 (pump/eopatch/)
- **Manufacturer**: Eoflow
- **Architecture**: Plugin → BLE GATT
- **Key Feature**: Disposable patch pump
- **Max Basal**: 15.0 U/h

### Medtrum Nano/300U (pump/medtrum/)
- **Manufacturer**: Medtrum
- **Architecture**: Plugin → BLE GATT
- **Key Feature**: Patch pump with history support
- **Known Issues**: #43 — CRITICAL: 10U dispensed during failed activation, #36 — wrong basal during patch change, #4373 — expiry warnings
- **Reservoir**: 200U (Nano), 300U (300U model)

### Equil (pump/equil/)
- **Manufacturer**: Equil
- **Architecture**: Plugin → BLE GATT (encrypted)
- **Key Feature**: Uses hardware link, encrypted communication
- **Known Issues**: #4505 — communication broken (firmware reports 0.0), #4485 — pin movement stops

## Quick Lookup: "Does This Pump Support...?"

| Feature | Dana | Omni Dash | Omni Eros | Medtronic | Combo | Insight | Diaconn | EOPatch | Medtrum | Equil |
|---------|------|-----------|-----------|-----------|-------|---------|---------|---------|---------|-------|
| Absolute TBR | via % | Yes | Yes | Yes | via % | via % | Yes | Yes | Yes | Yes |
| Percent TBR | Yes | No | No | No | Yes | Yes | No | No | No | No |
| Extended Bolus | Yes | Yes | Yes | No (TBR) | Yes | Yes | Yes | Yes | Yes | Yes |
| History Read | Yes | Partial | Partial | Yes | Via display | Yes | Yes | Partial | Yes | Partial |
| TDD Read | Yes | No | No | Yes | Via display | Yes | Yes | No | Yes | No |
| Battery Level | Yes | No | No | Yes | No | Yes | Yes | No | No | No |
| DST Auto | No | Yes | Yes | Some | No | No | No | No | No | No |
| Bolus Progress | Yes | Yes | Yes | No | Via display | Yes | Yes | Yes | Yes | Yes |
| Custom Actions | No | Yes | Yes | Yes (3) | No | No | No | No | Yes | No |

---

## BLE Service & Characteristic UUIDs

Reference for debugging BLE communication and writing driver code.

### Dana RS/i
```
UART Read:  0000fff1-0000-1000-8000-00805f9b34fb
UART Write: 0000fff2-0000-1000-8000-00805f9b34fb
CCCD:       00002902-0000-1000-8000-00805f9b34fb
Packet Format: Start 0xA5 / End 0x5A (standard), Start 0xAA / End 0xEE (BLE5)
Encryption: Custom (DEFAULT or ENHANCED mode)
Write Delay: 50ms between packets
Bonding: Required
```
Source: `pump/danars/src/main/kotlin/app/aaps/pump/danars/services/BLEComm.kt`

### Omnipod Dash
```
Scan Service:    00004024-0000-1000-8000-00805F9B34FB (16-bit: 4024)
Main Service:    1a7e-4024-e3ed-4464-8b7e-751e03d0dc5f
Secondary:       000a (unknown service)
Communication:   Async packet-based with BlockingQueue for receive
Bonding:         Must be OFF on Android 15+
```
Source: `pump/omnipod/common/src/main/kotlin/.../bledriver/comm/ServiceDiscoverer.kt`

### Medtrum Nano/300U
```
Service:    669A9001-0008-968F-E311-6050405558B3
Read Char:  669a9120-0008-968f-e311-6050405558b3
Write Char: 669a9101-0008-968f-e311-6050405558b3
CCCD:       00002902-0000-1000-8000-00805f9b34fb
Device Name Filter: "MT"
Manufacturer ID: 18305
Write Delay: 10ms between packets
Notifications: 0x10 (notification), 0x20 (indication)
```
Source: `pump/medtrum/src/main/kotlin/app/aaps/pump/medtrum/services/BLEComm.kt`

### Diaconn G8
```
Write Char: 6e400002-b5a3-f393-e0a9-e50e24dcca9e (Nordic UART TX)
Read Char:  6e400003-b5a3-f393-e0a9-e50e24dcca9e (Nordic UART RX)
Protocol:   Nordic UART Service (NUS) profile
```
Source: `pump/diaconn/src/main/kotlin/app/aaps/pump/diaconn/service/BLECommonService.kt`

### Insight (RFCOMM, not GATT)
```
RFCOMM UUID: 00001101-0000-1000-8000-00805f9b34fb (SPP UUID)
Connection:  createInsecureRfcommSocketToServiceRecord()
Security:    SATL (Secure Asynchronous Transmission Layer)
Key Exchange: RSA with nonce validation (Spongy Castle)
```
Source: `pump/insight/src/main/kotlin/app/aaps/pump/insight/connection_service/InsightConnectionService.kt`

### Combo V2 (RFCOMM)
```
Protocol: BLE GATT with frame-based ComboFrame protocol
Note: Despite using BluetoothGatt API, uses frame-based serial communication
```
Source: `pump/combov2/comboctl/src/androidMain/kotlin/.../AndroidBluetoothDevice.kt`

### Equil
```
Protocol: BLE GATT (encrypted)
Encryption: Custom encrypted commands
Firmware Query: CmdDevicesOldGet (firmware at byte positions 12-13)
Known Bug: Firmware reports 0.0 on some units (#4505)
```
Source: `pump/equil/src/main/kotlin/app/aaps/pump/equil/ble/EquilBLE.kt`

### Medtronic (via RileyLink)
```
Phone ←BLE GATT→ RileyLink ←RF 916MHz→ Pump
RileyLink handles BLE-to-RF translation
Custom binary packet protocol over RF
```
Source: `pump/rileylink/src/main/kotlin/app/aaps/pump/common/hw/rileylink/ble/RileyLinkBLE.kt`

### Omnipod Eros (via RileyLink)
```
Phone ←BLE GATT→ RileyLink ←RF 433MHz→ Pod
Same RileyLink relay as Medtronic but different RF frequency
```
Source: `pump/rileylink/` + `pump/omnipod-eros/`
