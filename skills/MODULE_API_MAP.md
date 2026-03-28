# Module API Map — What Each Module Exports & Imports

> Prevents agents from making wrong-layer calls or creating circular dependencies.

## Dependency Rules

1. **Never depend upward** — plugins/ cannot import from app/, pump/ cannot import from plugins/
2. **Never bypass persistence layer** — use `PersistenceLayer` interface, never import `database:impl` directly
3. **Never import pump specifics into core** — core must stay pump-agnostic
4. **Prefer interface from core:interfaces** over concrete implementations

## Layer Hierarchy (top depends on bottom, never reverse)

```
┌─────────────────────────────────────────────────┐
│  app/                  (assembles everything)    │
├─────────────────────────────────────────────────┤
│  plugins/*  pump/*  ui/  wear/  workflow/        │
├─────────────────────────────────────────────────┤
│  implementation/  database:persistence/          │
├─────────────────────────────────────────────────┤
│  core:interfaces  core:objects  shared:impl      │
├─────────────────────────────────────────────────┤
│  core:data  core:keys  core:utils  core:ui       │
├─────────────────────────────────────────────────┤
│  database:impl  core:nssdk  core:graph           │
└─────────────────────────────────────────────────┘
```

## Module Export Table

### Foundation Modules (no AAPS dependencies)

| Module | Exports | Depends On |
|--------|---------|------------|
| `core:data` | 50+ data models (GV, BS, TB, CA, TT, etc.), PumpType, PumpDescription, PluginType, enums | Nothing |
| `core:keys` | Preference key constants (BooleanKey, IntKey, StringKey, etc.) | Nothing |

### Contract Module

| Module | Exports | Depends On |
|--------|---------|------------|
| `core:interfaces` | 220+ interfaces: Pump, APS, Loop, BgSource, CommandQueue, PluginBase, ActivePlugin, PersistenceLayer, RxBus, IobCobCalculator, PluginConstraints, Profile, Insulin, Sensitivity, etc. | `core:data`, `core:keys` |

### Implementation Modules

| Module | Exports | Depends On |
|--------|---------|------------|
| `implementation` | PumpSyncImpl, CommandQueueImpl, DetailedBolusInfoStorageImpl, TddCalculatorImpl, ProfilerImpl, Logger setup | `core:interfaces`, `core:data` |
| `database:persistence` | PersistenceLayerImpl, entity converters, CompatDBHelper | `core:interfaces`, `core:data`, `database:impl` |
| `database:impl` | Room AppDatabase, DAOs, entities, transactions, AppRepository | `core:data` (only) |
| `shared:impl` | AAPSLoggerImpl, RxBusImpl, AapsSchedulersImpl, SPImpl, DateUtilImpl | `core:interfaces`, `core:data` |

### Plugin Modules

| Module | Exports | Depends On |
|--------|---------|------------|
| `plugins:aps` | OpenAPSAMA, OpenAPSSMB, LoopPlugin, AutotunePlugin | `core:interfaces`, `core:data` |
| `plugins:sync` | NSClientV3Plugin, TidepoolPlugin, GarminPlugin, XdripPlugin | `core:interfaces`, `core:nssdk` |
| `plugins:source` | DexcomPlugin, LibrePlugin, NSClientSourcePlugin, XdripSourcePlugin | `core:interfaces` |
| `plugins:automation` | AutomationPlugin, Actions, Triggers | `core:interfaces` |
| `plugins:constraints` | SafetyPlugin, ObjectivesPlugin | `core:interfaces` |
| `plugins:insulin` | RapidActingPlugin, UltraRapidPlugin, LyumjevPlugin | `core:interfaces` |
| `plugins:sensitivity` | SensitivityOref1Plugin, SensitivityAAPSPlugin, SensitivityWeightedAveragePlugin | `core:interfaces` |
| `plugins:smoothing` | AvgSmoothingPlugin, ExponentialSmoothingPlugin | `core:interfaces` |
| `plugins:main` | OverviewPlugin, ProfilePlugin, ActionsPlugin | `core:interfaces` |
| `plugins:configuration` | ConfigBuilderPlugin, MaintenancePlugin, SetupWizard | `core:interfaces` |

### Pump Modules

| Module | Exports | Depends On |
|--------|---------|------------|
| `pump:common` | PumpPluginAbstract, BLE utilities, history handling, sync storage | `core:interfaces`, `core:data` |
| `pump:virtual` | VirtualPumpPlugin | `core:interfaces`, `implementation` |
| `pump:dana` | Shared Dana code | `pump:common`, `core:interfaces` |
| `pump:danars` | DanaRSPlugin + DanaRSService | `pump:dana`, `pump:common` |
| `pump:omnipod-common` | Shared Omnipod code | `pump:common` |
| `pump:omnipod-dash` | OmnipodDashPumpPlugin | `pump:omnipod-common` |
| `pump:omnipod-eros` | OmnipodErosPumpPlugin | `pump:omnipod-common`, `pump:rileylink` |
| `pump:medtronic` | MedtronicPumpPlugin | `pump:common`, `pump:rileylink` |
| `pump:rileylink` | RileyLink BLE communication | `pump:common` |
| `pump:combov2` | ComboV2Plugin + comboctl library | `pump:common` |
| `pump:insight` | InsightPlugin | `pump:common` |
| `pump:equil` | EquilPumpPlugin | `pump:common` |
| `pump:medtrum` | MedtrumPlugin | `pump:common` |
| `pump:diaconn` | DiaconnG8Plugin | `pump:common` |
| `pump:eopatch` | EopatchPlugin | `pump:common` |

### Application Modules

| Module | Exports | Depends On |
|--------|---------|------------|
| `app` | MainApp, MainActivity, AppComponent, PluginsListModule | Everything |
| `ui` | Treatment screens, Stats, Overview fragments | `core:interfaces`, `core:graph` |
| `wear` | Wear OS app, tiles, complications | `shared:impl`, `core:interfaces` |
| `workflow` | CalculationWorkflow, Worker classes | `core:interfaces`, `database:impl` |

## Key Boundaries

### Database Access
```
WRONG:  import app.aaps.database.entities.Bolus     // Direct Room entity
RIGHT:  import app.aaps.core.data.model.BS          // Data model via core:data
RIGHT:  persistenceLayer.insertOrUpdateBolus(bolus)  // Via PersistenceLayer interface
```

### Pump Access
```
WRONG:  import app.aaps.pump.danars.DanaRSPlugin    // Direct pump import
RIGHT:  activePlugin.activePump.deliverTreatment()   // Via Pump interface
RIGHT:  commandQueue.bolus(dbi, callback)             // Via CommandQueue
```

### Plugin Access
```
WRONG:  import app.aaps.plugins.aps.openAPSSMB.OpenAPSSMBPlugin
RIGHT:  activePlugin.activeAPS.invoke()              // Via APS interface
```

## Adding a New Module

1. Create directory under appropriate parent (`plugins/`, `pump/`, `core/`)
2. Add `build.gradle.kts` with correct dependencies
3. Add to `settings.gradle`: `include ':plugins:your-module'`
4. Add Dagger module to `app/di/AppComponent.kt`
5. Follow layer rules — never depend upward
