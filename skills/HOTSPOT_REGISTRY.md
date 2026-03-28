# Hotspot Registry — Complex Files Requiring Attention

> Auto-generated registry of the 96 files over 500 lines.
> These are complexity hotspots — treat with extra care when modifying.

## Critical Hotspots (Safety-Critical + High Complexity)

| Lines | File | Domain | Notes |
|-------|------|--------|-------|
| 1,960 | `database/persistence/PersistenceLayerImpl.kt` | Database | Massive API surface (100+ transaction types). Consider splitting by entity type. |
| 1,444 | `core/interfaces/db/PersistenceLayer.kt` | Interface | Matches PersistenceLayerImpl. Same splitting opportunity. |
| 1,608 | `pump/omnipod-dash/OmnipodDashPumpPlugin.kt` | Pump | Complex BLE + pod state management. Active bug area (#37, #4522). |
| 2,319 | `pump/combov2/ComboV2Plugin.kt` | Pump | Stuck-in-error-state bug (#32). State machine needs recovery path. |
| 3,594 | `pump/combov2/comboctl/main/Pump.kt` | Pump | Core Combo pump protocol. Very complex state machine. |
| 2,084 | `pump/combov2/comboctl/base/PumpIO.kt` | Pump | Low-level I/O. Fragile BLE/RFCOMM code. |
| 1,575 | `pump/insight/InsightPlugin.kt` | Pump | Insight pump driver. Complex connection handling. |
| 1,434 | `pump/medtronic/data/MedtronicHistoryData.kt` | Pump | History parsing. Medtronic-specific data formats. |

## High Complexity (Non-Safety but Important)

| Lines | File | Domain | Notes |
|-------|------|--------|-------|
| 1,928 | `plugins/sync/wear/DataHandlerMobile.kt` | Wear | Handles all Wear OS data exchange. Many message types. |
| 1,419 | `plugins/configuration/maintenance/cloud/GoogleDriveManager.kt` | Config | Google Drive backup. OAuth flow complexity. |
| 1,537 | `plugins/main/test/AutosensDataStoreTest.kt` | Test | Large test file — good coverage, may need splitting for maintainability. |
| 1,438 | `shared/impl/test/DateUtilImplTest.kt` | Test | Comprehensive date utility tests. Well-structured. |

## Combo V2 — Largest Module (Highest Risk)

The `pump/combov2/` module contains the most complex code:

| Lines | File | Concern |
|-------|------|---------|
| 12,368 | `comboctl/jvmTest/TestDisplayFrames.kt` | Test data — display frame bitmaps |
| 3,594 | `comboctl/main/Pump.kt` | Main pump protocol |
| 2,319 | `ComboV2Plugin.kt` | AAPS integration — **error state bug #32** |
| 2,084 | `comboctl/base/PumpIO.kt` | Low-level I/O |
| 2,043 | `comboctl/base/Pattern.kt` | Display pattern matching |
| 1,929 | `comboctl/jvmTest/ParserTest.kt` | Parser tests |
| 1,882 | `comboctl/base/ApplicationLayer.kt` | App protocol layer |
| 1,819 | `comboctl/parser/Parser.kt` | Display screen parsing |

**Recommendation**: The Combo driver is the riskiest module. Changes require extra review and testing. The error-state recovery bug (#32) should be fixed with extreme care.

## Refactoring Opportunities

### Immediate (small, safe)
1. **PersistenceLayerImpl** (1,960 lines): Extract into category-specific files:
   - `PersistenceLayerBolus.kt` — bolus operations
   - `PersistenceLayerTempBasal.kt` — TBR operations
   - `PersistenceLayerGlucose.kt` — BG operations
   - Keep main class as facade delegating to these

2. **DataHandlerMobile** (1,928 lines): Extract message handlers into separate classes per message type

### Future (large, needs design)
3. **OmnipodDashPumpPlugin** (1,608 lines): Extract pod state management into separate class
4. **ComboV2Plugin** (2,319 lines): State machine could use a formal FSM pattern

## Metrics Summary

```
Total Kotlin files:    3,493
Total Kotlin lines:    359,287
Files > 500 lines:     96 (2.7% of files, ~40% of code)
Files > 1000 lines:    30
Files > 2000 lines:    5
Largest file:          12,368 lines (test data)
Largest source file:   3,594 lines (Combo Pump.kt)

Lines by module:
  pump/         172,762 (48% — half the codebase)
  plugins/       86,915 (24%)
  core/          27,394 (8%)
  database/      20,649 (6%)
  wear/          13,331 (4%)
  implementation/11,525 (3%)
  ui/            10,320 (3%)
  app/            9,403 (3%)
  shared/         4,695 (1%)
  workflow/       2,198 (<1%)
```

## Agent Guidelines for Hotspots

When modifying files in this registry:
1. **Read the full file first** — understand context before changing
2. **Run existing tests** for that module before and after changes
3. **Keep changes minimal** — surgical fixes, not rewrites
4. **Flag for review** if change exceeds 100 lines in a hotspot file
5. **Add tests** for any logic change in these files
6. **Document** the change in PR description with before/after behavior
