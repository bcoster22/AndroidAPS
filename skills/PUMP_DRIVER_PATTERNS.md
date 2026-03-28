# Pump Driver Development Patterns

> How to read, write, and maintain the 14 pump drivers (172K lines — half the codebase).

## Architecture

```
CommandQueue (orchestrates)
    ↓ calls Pump interface methods
PumpPlugin (extends PumpPluginBase, implements Pump)
    ↓ delegates to
BLE Service / Manager (handles Bluetooth communication)
    ↓ reports results via
PumpSync (stores delivery data back to AAPS database)
```

## Class Hierarchy

```
PluginBase
  └── PluginBaseWithPreferences
        └── PumpPluginBase (adds Handler thread, CommandQueue ref)
              ├── VirtualPumpPlugin       (no BLE — testing)
              ├── DanaRSPlugin            (Service-based BLE)
              ├── OmnipodDashPumpPlugin   (Manager-based BLE)
              ├── MedtronicPumpPlugin     (RileyLink relay)
              ├── ComboV2Plugin           (RFCOMM BLE)
              └── ... (9 more drivers)
```

## Starting Point: VirtualPumpPlugin

The simplest driver — no BLE, immediate success. Use as a template:
- `pump/virtual/src/main/kotlin/app/aaps/pump/virtual/VirtualPumpPlugin.kt`

## Pump Interface Methods (Must Implement)

### Connection Lifecycle
```kotlin
fun isInitialized(): Boolean    // Pump status read and ready
fun isConnected(): Boolean      // BLE connection active
fun isConnecting(): Boolean     // BLE connection in progress
fun isSuspended(): Boolean      // Pump suspended (no delivery)
fun isBusy(): Boolean           // Pump busy (don't send commands)
fun connect(reason: String)     // Initiate BLE connection
fun disconnect(reason: String)  // Close BLE connection
fun stopConnecting()            // Abort connection attempt
fun getPumpStatus(reason: String) // Force-read full pump status
```

### Insulin Delivery
```kotlin
fun deliverTreatment(detailedBolusInfo: DetailedBolusInfo): PumpEnactResult
fun stopBolusDelivering()
fun setTempBasalAbsolute(rate, duration, profile, enforceNew, tbrType): PumpEnactResult
fun setTempBasalPercent(percent, duration, profile, enforceNew, tbrType): PumpEnactResult
fun cancelTempBasal(enforceNew: Boolean): PumpEnactResult
fun setExtendedBolus(insulin, duration): PumpEnactResult
fun cancelExtendedBolus(): PumpEnactResult
```

### State Properties
```kotlin
val baseBasalRate: Double       // Profile basal (not affected by TBR)
val reservoirLevel: Double      // Remaining insulin [U]
val batteryLevel: Int?          // Battery [%]
val lastDataTime: Long          // Last successful connection [ms]
val pumpDescription: PumpDescription  // Capabilities
```

## PumpEnactResult Pattern

Every command returns a `PumpEnactResult`:
```kotlin
return pumpEnactResultProvider.get()
    .success(true)          // Command succeeded
    .enacted(true)          // Change was made (false if already at requested state)
    .bolusDelivered(5.0)    // Amount delivered
    .comment("OK")          // Human-readable result
```

## PumpSync — Reporting Data Back to AAPS

**Critical**: After every successful pump command, report it via PumpSync:

```kotlin
// After bolus delivery
pumpSync.syncBolusWithPumpId(
    timestamp = System.currentTimeMillis(),
    amount = deliveredAmount,
    type = BS.Type.NORMAL,          // or SMB, PRIMING
    pumpId = pumpHistoryRecordId,   // Unique ID from pump
    pumpType = PumpType.YOUR_PUMP,
    pumpSerial = serialNumber()
)

// After setting TBR
pumpSync.syncTemporaryBasalWithPumpId(
    timestamp = System.currentTimeMillis(),
    rate = absoluteRate,
    duration = T.mins(durationMinutes.toLong()).msecs(),
    isAbsolute = true,
    type = PumpSync.TemporaryBasalType.NORMAL,
    pumpId = pumpHistoryRecordId,
    pumpType = PumpType.YOUR_PUMP,
    pumpSerial = serialNumber()
)

// Query expected pump state
val state = pumpSync.expectedPumpState()
val currentTBR = state.temporaryBasal  // null if no TBR running
val lastBolus = state.bolus
```

## BLE Communication Patterns

### Pattern A: Service-Based (DanaRS)
```
Plugin ←binds→ DanaRSService ←BLE→ Pump
```
- Service runs in background, survives activity lifecycle
- Plugin binds via `ServiceConnection`
- Service handles BLE state machine

### Pattern B: Manager-Based (OmnipodDash)
```
Plugin ←calls→ OmnipodDashManager ←BLE→ Pod
```
- Manager injected directly
- Plugin delegates all BLE operations
- State tracked in `PodStateManager`

### Pattern C: RileyLink Relay (Medtronic)
```
Plugin ←calls→ RileyLinkService ←BLE→ RileyLink ←RF→ Pump
```
- Two-hop communication (BLE to relay device, RF to pump)
- Higher latency and failure rate

## Error Handling

```kotlin
// Always return PumpEnactResult, even on failure
override fun deliverTreatment(dbi: DetailedBolusInfo): PumpEnactResult {
    return try {
        val result = bleService.sendBolusCommand(dbi.insulin)
        if (result.success) {
            pumpSync.syncBolusWithPumpId(...)  // Report to AAPS
            pumpEnactResultProvider.get()
                .success(true).enacted(true).bolusDelivered(dbi.insulin)
        } else {
            pumpEnactResultProvider.get()
                .success(false).enacted(false).comment(result.errorMessage)
        }
    } catch (e: Exception) {
        aapsLogger.error(LTag.PUMP, "Bolus failed", e)
        pumpEnactResultProvider.get()
            .success(false).enacted(false).comment(e.message ?: "Unknown error")
    }
}
```

## Common Pitfalls

| Pitfall | Consequence | Prevention |
|---------|-------------|------------|
| Not calling PumpSync after delivery | AAPS doesn't know about insulin | Always sync after success |
| Reporting delivery before confirmed | IOB inflated if delivery fails | Only sync after pump confirms |
| Not handling BLE disconnect mid-command | Uncertain delivery state | Use temp IDs for unconfirmed boluses |
| Keeping stale TBR on disconnect | Pump runs TBR indefinitely | Handle disconnect events, consider fake TBR |
| Thread blocking on main thread | ANR / UI freeze | Use Handler thread from PumpPluginBase |

## Key Files Reference

| File | Purpose |
|------|---------|
| `core/interfaces/pump/Pump.kt` | Interface contract |
| `core/interfaces/pump/PumpPluginBase.kt` | Base class with Handler |
| `core/interfaces/pump/PumpSync.kt` | Data sync interface |
| `core/interfaces/pump/PumpEnactResult.kt` | Command result |
| `core/interfaces/pump/DetailedBolusInfo.kt` | Bolus request data |
| `core/data/pump/defs/PumpType.kt` | Pump type registry |
| `pump/virtual/` | Template driver (simplest) |
| `pump/danars/` | Real BLE driver (service pattern) |
| `pump/omnipod-dash/` | Real BLE driver (manager pattern) |
