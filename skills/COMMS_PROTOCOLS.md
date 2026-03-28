# Communication Protocols & Inter-Device APIs

> All communication channels between AndroidAPS and external devices/apps/services.

## Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         AndroidAPS (Phone)                              │
│                                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ BLE GATT │  │ BLE      │  │ Broadcast│  │ REST API │  │ Wearable │ │
│  │          │  │ RFCOMM   │  │ Intents  │  │ HTTP     │  │ DataLayer│ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
└───────┼──────────────┼──────────────┼──────────────┼──────────────┼─────┘
        │              │              │              │              │
   ┌────▼────┐   ┌─────▼───┐   ┌─────▼────┐   ┌────▼─────┐  ┌────▼────┐
   │ Insulin │   │ Combo   │   │ CGM Apps │   │Nightscout│  │ Wear OS │
   │ Pumps   │   │ Pump    │   │ xDrip+   │   │ Tidepool │  │ Watch   │
   │ (most)  │   │ (RFCOMM)│   │ Dexcom   │   │ Garmin   │  │         │
   └─────────┘   └─────────┘   └──────────┘   └──────────┘  └─────────┘
```

---

## 1. Bluetooth Low Energy (BLE GATT)

### Used By
Dana RS/i, Omnipod Dash, Diaconn G8, EOPatch2, Medtrum, Equil, Insight

### Protocol
- Standard BLE GATT (Generic Attribute Profile)
- Service/Characteristic UUID-based communication
- Notifications for async pump responses
- Connection interval: typically 7.5-30ms

### Android API
```kotlin
BluetoothGatt.connectGatt(context, autoConnect, callback)
gatt.writeCharacteristic(characteristic)
gatt.setCharacteristicNotification(characteristic, true)
```

### Key Files
- `pump/danars/services/BLEComm.kt` — Dana RS BLE implementation
- `pump/omnipod-dash/` — Dash BLE manager
- `pump/medtrum/comm/` — Medtrum BLE communication

### Common Issues
- Android 16 BLE optimization conflicts (#37)
- "Nearby devices" permission required (Android 12+)
- Bond/pairing issues — Dash requires Bond OFF on Android 15+
- Background BLE scanning from other apps causes interference

---

## 2. Bluetooth RFCOMM (Serial)

### Used By
Accu-Chek Combo (via comboctl library)

### Protocol
- Classic Bluetooth RFCOMM (serial port emulation)
- Older protocol than BLE GATT
- Higher power consumption but more reliable for some use cases

### Android API
```kotlin
BluetoothDevice.createRfcommSocketToServiceRecord(uuid)
socket.connect()
socket.inputStream / socket.outputStream
```

### Key Files
- `pump/combov2/comboctl/` — Combo communication library
- `pump/combov2/comboctl/base/PumpIO.kt` — Low-level I/O

### Known Issues
- W6 warnings from timing mismatches on Android 15+ (#4479)
- RFCOMM less well-supported on modern Android versions

---

## 3. RileyLink RF Communication

### Used By
Omnipod Eros, Medtronic pumps

### Protocol
```
Phone ──BLE GATT──▶ RileyLink/OrangeLink ──RF──▶ Pump
                    (433MHz for Eros)
                    (916MHz for Medtronic)
```

### Architecture
- Two-hop communication: BLE to relay device, then RF to pump
- Higher latency and failure rate than direct BLE
- Relay devices: RileyLink, OrangeLink, EmaLink

### Key Files
- `pump/rileylink/` — RileyLink communication layer
- `pump/omnipod-eros/` — Eros RF commands via RileyLink
- `pump/medtronic/` — Medtronic RF commands via RileyLink

### Known Issues
- RileyLink battery level unreliable with NiMH batteries
- Range limited by RF power (~1-3 meters typical)

---

## 4. CGM Data Sources (Broadcast Intents)

### xDrip+ → AAPS
- **Protocol**: Android Broadcast Intent
- **Action**: `com.eveningoutpost.dexdrip.BgEstimate`
- **Data**: BG value, timestamp, slope, direction
- **Plugin**: `plugins/source/XdripSourcePlugin.kt`

### Dexcom App → AAPS
- **Protocol**: Android Broadcast Intent
- **Action**: `com.dexcom.cgm.EXTERNAL_BROADCAST`
- **Data**: BG readings, sensor info, calibrations
- **Plugin**: `plugins/source/DexcomPlugin.kt`
- **Feature**: Supports advanced filtering (native mode)

### Libre (via Juggluco/xDrip+)
- **Protocol**: Broadcast Intent (relayed through xDrip+ or Juggluco)
- **Plugin**: `plugins/source/` (multiple Libre source plugins)

### Nightscout as BG Source
- **Protocol**: REST API (HTTP GET)
- **Endpoint**: `/api/v3/entries` (SGV records)
- **Plugin**: `plugins/source/NSClientSourcePlugin.kt`

### General Broadcast Receiver
- **Location**: `app/src/main/kotlin/app/aaps/receivers/DataReceiver.kt`
- **Handles**: BG data from multiple sources
- **Pattern**: Receives intent → routes to active BG source plugin → stores in DB

---

## 5. Nightscout Sync (REST API)

### Protocol
- HTTP/HTTPS REST API (v1 and v3)
- Authentication: Bearer token (API secret hash or JWT)
- Transport: Retrofit 3 + OkHttp 5

### API v3 Endpoints (NSClientV3)
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v3/entries` | GET/POST | Glucose values (SGV) |
| `/api/v3/treatments` | GET/POST | Treatments (bolus, carbs, TBR, etc.) |
| `/api/v3/devicestatus` | GET/POST | Device status (IOB, COB, pump state) |
| `/api/v3/food` | GET/POST | Food database |
| `/api/v3/profile` | GET/POST | Profile data |

### Sync Architecture
```
Local DB change → DataSyncWorker (WorkManager) → NSClientV3Service
    → Retrofit POST to Nightscout
    → Response with NS ID → Update local record with NS ID

Nightscout change → LoadBgWorker / LoadTreatmentsWorker
    → Retrofit GET with modified_since
    → StoreDataForDb → PersistenceLayer.runTransaction()
```

### Key Files
- `core/nssdk/` — Nightscout SDK (Retrofit service, models, auth)
- `plugins/sync/nsclientV3/` — NSClient V3 plugin
- `plugins/sync/nsclientV3/workers/` — Background sync workers

### Known Issues
- #38 — COB/CAGE/IAGE not uploading
- NSv3 queue can accumulate 74K+ entries if stuck

---

## 6. Tidepool Sync (REST API)

### Protocol
- HTTPS REST API with OAuth2 authentication
- Batch upload of diabetes data

### Endpoints
| Endpoint | Purpose |
|----------|---------|
| `/auth/login` | Authentication |
| `/v1/users/:userId/data_sets` | Dataset management |
| `/v1/users/:userId/data_sets/:dataSetId/data` | Data upload |

### Data Types
- `BasalElement`, `BolusElement`, `BloodGlucoseElement`
- `SensorGlucoseElement`, `WizardElement`, `ProfileElement`

### Key Files
- `plugins/sync/tidepool/` — Tidepool plugin
- `plugins/sync/tidepool/comm/TidepoolUploader.kt` — Upload logic

### Known Issues
- #30 — BLOCKED state regression in 3.4.x
- #45 — Session null after establishment in 3.4.1.0-dev

---

## 7. Wear OS Communication (Google Play Services Wearable)

### Protocol
- Google Play Services Wearable Data Layer API
- Bidirectional message passing and data sync

### Communication Channels
| Channel | Direction | Purpose |
|---------|-----------|---------|
| `DataClient` | Phone ↔ Watch | Persistent data sync (BG, status) |
| `MessageClient` | Phone ↔ Watch | One-time commands (bolus, TBR) |
| Tiles | Watch → Phone | Quick actions from watch face |
| Complications | Watch → Phone | Status display on watch face |

### Message Types (EventData)
```kotlin
// Phone → Watch
EventData.Status           // Current BG, IOB, COB, loop status
EventData.TreatmentData    // Treatment history
EventData.BolusProgress    // Bolus delivery progress

// Watch → Phone
EventData.ActionBolusRequest     // User requests bolus from watch
EventData.ActionTempTargetRequest // Set temp target from watch
EventData.ActionProfileSwitchRequest // Switch profile
```

### Key Files
- `plugins/sync/wear/wearintegration/DataHandlerMobile.kt` — Phone-side handler (1,928 lines)
- `wear/src/main/kotlin/` — Watch app implementation
- `core/interfaces/rx/weardata/EventData.kt` — Shared data classes

---

## 8. SMS Communicator

### Protocol
- Standard SMS text messages
- Commands protected by phone number allowlist + PIN

### Supported Commands
| Command | Action |
|---------|--------|
| `BG` | Report current BG |
| `LOOP STOP/START` | Stop/start loop |
| `LOOP SUSPEND <min>` | Suspend loop |
| `LOOP RESUME` | Resume loop |
| `TREATMENTS REFRESH` | Refresh treatments from NS |
| `NSCLIENT RESTART` | Restart NS client |
| `PUMP` | Report pump status |
| `PUMP CONNECT/DISCONNECT <min>` | Connect/disconnect pump |
| `BOLUS <amount>` | Deliver bolus (requires PIN) |
| `CARBS <grams>` | Record carbs |
| `TARGET <low> <high>` | Set temp target |
| `PROFILE <name>` | Switch profile |
| `SMS DISABLE` | Disable SMS communicator |

### Key Files
- `plugins/configuration/src/main/kotlin/.../smsCommunicator/SmsCommunicatorPlugin.kt`

---

## 9. Garmin Integration

### Protocol
- Garmin ConnectIQ SDK
- HTTP loopback for local communication (device → phone app → AAPS)

### Key Files
- `plugins/sync/garmin/` — Garmin plugin

---

## 10. AAPS Broadcast Outputs

### AAPS sends these broadcasts for other apps to consume:

| Action | Data | Consumer |
|--------|------|----------|
| `info.nightscout.androidaps.status` | Loop status, BG, IOB | xDrip+, widgets |
| Treatment updates | Via Nightscout sync | NS-connected apps |

### AAPS listens to these broadcasts:

| Action | Source | Handler |
|--------|--------|---------|
| `com.eveningoutpost.dexdrip.BgEstimate` | xDrip+ | DataReceiver |
| `com.dexcom.cgm.EXTERNAL_BROADCAST` | Dexcom | DataReceiver |
| `android.bluetooth.*` | System | BTReceiver |
| `android.net.conn.CONNECTIVITY_CHANGE` | System | NetworkChangeReceiver |
| `android.intent.action.BATTERY_CHANGED` | System | ChargingStateReceiver |
| `android.intent.action.TIME_SET` | System | TimeDateOrTZChangeReceiver |

### Key Files
- `app/src/main/kotlin/app/aaps/receivers/` — All broadcast receivers
- `core/interfaces/src/main/kotlin/.../receivers/Intents.kt` — Intent action constants

---

## Protocol Summary Table

| Protocol | Transport | Direction | Latency | Reliability |
|----------|-----------|-----------|---------|-------------|
| BLE GATT | Bluetooth LE | Bidirectional | Low (ms) | Good |
| BLE RFCOMM | Bluetooth Classic | Bidirectional | Low (ms) | Fair (Android 15+ issues) |
| RileyLink RF | BLE + RF | Bidirectional | Medium (100ms+) | Fair (range limited) |
| Broadcast Intent | Android IPC | One-way | Very low | High (same device) |
| REST API (NS) | HTTP/HTTPS | Bidirectional | High (network) | Depends on connectivity |
| Tidepool API | HTTPS | Upload only | High (network) | Fair (session issues) |
| Wearable DataLayer | Google Play Services | Bidirectional | Medium | Good |
| SMS | Cellular | Bidirectional | High (seconds) | Fair |
| Garmin | ConnectIQ + HTTP | Bidirectional | Medium | Good |
