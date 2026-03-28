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
| Extended Bolus | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| History Read | Yes | Partial | Partial | Yes | Via display | Yes | Yes | Partial | Yes | Partial |
| TDD Read | Yes | No | No | Yes | Via display | Yes | Yes | No | Yes | No |
| Battery Level | Yes | No | No | Yes | No | Yes | Yes | No | No | No |
| DST Auto | No | Yes | Yes | Some | No | No | No | No | No | No |
| Bolus Progress | Yes | Yes | Yes | No | Via display | Yes | Yes | Yes | Yes | Yes |
| Custom Actions | No | Yes | Yes | No | No | No | No | No | Yes | No |
