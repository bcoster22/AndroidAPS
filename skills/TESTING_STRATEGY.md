# Testing Strategy for AndroidAPS

> Medical device software testing patterns. Every fix needs a test.

## Test Infrastructure

### Base Classes

| Class | Location | Use When |
|-------|----------|----------|
| `TestBase` | `shared/tests/` | Standard unit tests |
| `TestBaseWithProfile` | `shared/tests/` | APS algorithm tests, profile-dependent logic |
| (none — lightweight mocking) | | Database transaction tests |

### TestBase Provides
- JUnit 5 + Mockito extension
- `aapsLogger` (mock)
- `aapsSchedulers` (test schedulers — synchronous)
- `rxBus` (real instance with test schedulers)
- Locale set to English for consistent formatting
- Firebase disabled

### TestBaseWithProfile Provides (on top of TestBase)
- `validProfile` — a complete, valid profile with basal/ISF/IC/target blocks
- `profileFunction` (mock)
- `dateUtil` (spy on real implementation)
- `profileUtil` (real implementation)
- `hardLimits` (mock with safe defaults)
- `iobCobCalculator` (mock)
- `activePlugin` (mock)
- `glucoseStatusCalculatorSMB` (real)
- Helper: `getValidProfileStore()`, `getInvalidProfileStore1/2()`

## Writing Unit Tests

### Basic Plugin Test
```kotlin
class MyPluginTest : TestBase() {

    @Mock lateinit var rh: ResourceHelper
    @Mock lateinit var rxBus: RxBus
    @Mock lateinit var persistenceLayer: PersistenceLayer

    private lateinit var plugin: MyPlugin

    @BeforeEach
    fun setup() {
        plugin = MyPlugin(aapsLogger, rh, rxBus, persistenceLayer)
    }

    @Test
    fun `should be disabled by default`() {
        assertFalse(plugin.isEnabled())
    }

    @Test
    fun `should handle enable lifecycle`() {
        plugin.setPluginEnabledBlocking(PluginType.GENERAL, true)
        assertTrue(plugin.isEnabled())
    }
}
```

### Algorithm Test (with profile)
```kotlin
class MyApsTest : TestBaseWithProfile() {

    private lateinit var plugin: MyApsPlugin

    @BeforeEach
    fun setup() {
        plugin = MyApsPlugin(aapsLogger, rh, ..., profileFunction, iobCobCalculator)
    }

    @Test
    fun `should not run without glucose data`() {
        // Arrange — no glucose status available
        `when`(glucoseStatusProvider.getGlucoseStatusData(true)).thenReturn(null)

        // Act
        plugin.invoke("test", false)

        // Assert — no result produced
        assertNull(plugin.lastAPSResult)
    }

    @Test
    fun `should respect max IOB constraint`() {
        // Arrange
        val constraint = ConstraintObject(10.0, aapsLogger)

        // Act
        plugin.applyMaxIOBConstraints(constraint)

        // Assert
        assertTrue(constraint.value() <= expectedMaxIOB)
    }
}
```

### Database Transaction Test
```kotlin
class MyTransactionTest {
    // No base class — lightweight mocking

    private lateinit var database: DelegatedAppDatabase
    private lateinit var dao: BolusDao

    @BeforeEach
    fun setup() {
        database = mock()
        dao = mock()
        whenever(database.bolusDao).thenReturn(dao)
    }

    @Test
    fun `should insert new bolus`() {
        // Arrange
        val bolus = createTestBolus(timestamp = 1000L, amount = 5.0)
        val transaction = InsertBolusTransaction(bolus)
        transaction.database = database

        // Act
        val result = transaction.run()

        // Assert
        verify(dao).insertNewEntry(bolus)
        assertEquals(1, result.inserted.size)
    }
}
```

## Test Categories

### Must Test (safety-critical)
- [ ] Constraint enforcement (max IOB, max basal, max bolus)
- [ ] Algorithm output for edge cases (very high/low BG, no data, stale data)
- [ ] PumpSync data reporting accuracy
- [ ] Database transaction atomicity
- [ ] Event bus event emission after state changes

### Should Test (quality)
- [ ] Plugin enable/disable lifecycle
- [ ] Preference changes affect behavior
- [ ] Error handling (what happens when things fail)
- [ ] Data model conversions (entity ↔ model)
- [ ] UI state updates after events

### Nice to Test (coverage)
- [ ] String formatting / display logic
- [ ] Date/time conversions
- [ ] Serialization/deserialization

## Testing Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Testing implementation details | Brittle tests | Test behavior/output, not internals |
| No assertion in test | Test always passes | Every test needs at least one assert |
| `Thread.sleep()` in tests | Slow, flaky | Use test schedulers |
| Mocking everything | Tests don't catch real bugs | Use real objects where practical |
| One giant test method | Hard to debug failures | Split into focused test per scenario |
| Testing private methods | Indicates design issue | Test via public API |

## Coverage Targets

| Module | Current | Target | Priority |
|--------|---------|--------|----------|
| `core/interfaces` | — | 80% | High (contracts) |
| `plugins/aps` | — | 90% | Critical (algorithms) |
| `plugins/constraints` | — | 90% | Critical (safety) |
| `implementation/queue` | — | 80% | High (pump commands) |
| `pump/*` | — | 60% | Medium (per driver) |
| `database` | — | 70% | Medium (transactions) |
| `plugins/sync` | — | 50% | Low |
| `ui` | — | 30% | Low |

## Running Tests

```bash
# All unit tests
./gradlew testFullDebugUnitTest

# Single module
./gradlew :plugins:aps:testFullDebugUnitTest

# Single test class
./gradlew :plugins:aps:testFullDebugUnitTest --tests "*.OpenAPSSMBPluginTest"

# With coverage
./gradlew jacocoAllDebugReport
# Report: build/reports/jacoco/jacocoAllDebugReport/

# Instrumentation tests (needs emulator)
./gradlew connectedFullDebugAndroidTest
```
