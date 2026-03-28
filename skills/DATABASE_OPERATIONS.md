# Database Operations Patterns

> Room database patterns, transaction system, and migration strategy.

## Architecture

```
Plugin/Implementation code
    ↓ calls
PersistenceLayer (interface — public API)
    ↓ delegates to
AppRepository (transaction executor)
    ↓ wraps in
DelegatedAppDatabase (change tracking)
    ↓ executes via
Room DAO (SQL operations)
    ↓ reads/writes
SQLite (AppDatabase v31)
```

## Reading Data

### Via PersistenceLayer (preferred)
```kotlin
// Single result
disposable += persistenceLayer.getNewestBolus()
    .observeOn(aapsSchedulers.main)
    .subscribe(
        { bolus -> updateUi(bolus) },
        { error -> aapsLogger.error("Query failed", error) }
    )

// List result
disposable += persistenceLayer.getBolusesFromTime(startTime, includeInvalid = false)
    .observeOn(aapsSchedulers.main)
    .subscribe { list -> adapter.submitList(list) }
```

### Subscribing to Changes
```kotlin
// React to any database change
disposable += appRepository.changeObservable()
    .filter { it.filterBolus() }        // Filter for bolus changes
    .observeOn(aapsSchedulers.main)
    .subscribe { refreshBolusUi() }
```

## Writing Data

### Insert via Transaction
```kotlin
// Fire-and-forget
disposable += persistenceLayer.insertOrUpdateBolus(bolus)
    .subscribe()

// With result handling
disposable += persistenceLayer.insertOrUpdateBolus(bolus)
    .subscribe(
        { aapsLogger.debug("Bolus saved successfully") },
        { aapsLogger.error("Failed to save bolus", it) }
    )
```

### Custom Transaction
```kotlin
class InsertMyDataTransaction(
    private val data: MyData
) : Transaction<TransactionResult>() {

    override fun run(): TransactionResult {
        val result = TransactionResult()

        // 1. Validate
        require(data.timestamp > 0) { "Invalid timestamp" }

        // 2. Check for existing
        val existing = database.myDao.findByPumpId(
            data.interfaceIDs.pumpId!!,
            data.interfaceIDs.pumpType!!,
            data.interfaceIDs.pumpSerial!!
        )

        // 3. Insert or update
        if (existing == null) {
            database.myDao.insertNewEntry(data)
            result.inserted.add(data)
        } else {
            // Already exists — skip or update
            result.notUpdated.add(data)
        }

        return result
    }
}
```

### Sync Transaction (from Nightscout)
```kotlin
class SyncNsMyDataTransaction(
    private val records: List<MyData>
) : Transaction<TransactionResult>() {

    override fun run(): TransactionResult {
        val result = TransactionResult()

        for (record in records) {
            // Check by NS ID
            val existing = database.myDao.findByNsId(record.interfaceIDs.nightscoutId)

            when {
                existing != null && !record.isValid -> {
                    // Invalidation from NS
                    existing.isValid = false
                    database.myDao.updateExistingEntry(existing)
                    result.invalidated.add(existing)
                }
                existing == null -> {
                    // New record from NS
                    database.myDao.insertNewEntry(record)
                    result.inserted.add(record)
                }
                else -> {
                    // Existing, update NS ID
                    existing.interfaceIDs.nightscoutId = record.interfaceIDs.nightscoutId
                    database.myDao.updateExistingEntry(existing)
                    result.updatedNsId.add(existing)
                }
            }
        }

        return result
    }
}
```

## Transaction Execution in Repository

```kotlin
// Completable (no result needed)
fun <T> runTransaction(transaction: Transaction<T>): Completable {
    val changes = mutableListOf<DBEntry>()
    return Completable.fromCallable {
        database.runInTransaction {
            transaction.database = DelegatedAppDatabase(changes, database)
            transaction.run()
        }
    }
    .subscribeOn(Schedulers.io())
    .doOnComplete { changeSubject.onNext(changes) }  // Emit changes
}

// Single (result needed)
fun <T : Any> runTransactionForResult(transaction: Transaction<T>): Single<T> {
    val changes = mutableListOf<DBEntry>()
    return Single.fromCallable {
        database.runInTransaction(Callable {
            transaction.database = DelegatedAppDatabase(changes, database)
            transaction.run()
        })
    }
    .subscribeOn(Schedulers.io())
    .doOnSuccess { changeSubject.onNext(changes) }  // Emit changes
}
```

## Entities (Data Models)

### Core Pattern
```kotlin
@Entity(
    tableName = "boluses",
    indices = [
        Index("timestamp"),
        Index("referenceId"),
        Index("id"),
        Index("isValid")
    ]
)
data class Bolus(
    @PrimaryKey(autoGenerate = true) var id: Long = 0,
    var timestamp: Long,
    var amount: Double,
    var type: Type,
    var isBasalInsulin: Boolean = false,
    var isValid: Boolean = true,
    var referenceId: Long? = null,
    var interfaceIDs_backing: InterfaceIDs? = null
) : DBEntry {
    enum class Type { NORMAL, SMB, PRIMING }
}
```

### InterfaceIDs (cross-system identification)
```kotlin
data class InterfaceIDs(
    var nightscoutSystemId: String? = null,
    var nightscoutId: String? = null,
    var pumpType: PumpType? = null,
    var pumpSerial: String? = null,
    var pumpId: Long? = null,
    var startId: Long? = null,
    var endId: Long? = null
)
```

## Migrations

### Adding a Migration
```kotlin
// In DatabaseModule.kt
private val migration31to32 = object : Migration(31, 32) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE boluses ADD COLUMN newField TEXT DEFAULT NULL")
    }
}

// Register in database builder
Room.databaseBuilder(context, AppDatabase::class.java, DB_NAME)
    .addMigrations(migration31to32)
    .build()
```

### Rules
- **Never use destructive migration** — data loss is unacceptable for medical device
- Increment database version in `AppDatabase` `@Database(version = XX)`
- Test migration with Room's `MigrationTestHelper`
- Handle nullable new columns with DEFAULT values

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Querying on main thread | ANR crash | Always use `Schedulers.io()` |
| Not using transactions for multi-step writes | Data inconsistency | Wrap in Transaction class |
| Forgetting `isValid` filter | Include soft-deleted records | Filter `isValid = true` |
| Not tracking changes | UI doesn't update | Use DelegatedAppDatabase |
| Destructive migration | Data loss | Always write migration scripts |
| Missing index | Slow queries | Add `@Index` on frequently queried columns |

## Key Files

| File | Purpose |
|------|---------|
| `database/impl/.../AppDatabase.kt` | Room DB definition (v31) |
| `database/impl/.../AppRepository.kt` | Transaction execution |
| `database/impl/.../transactions/` | All transaction classes |
| `database/impl/.../daos/` | Room DAO interfaces |
| `database/impl/.../entities/` | Room entity classes |
| `database/persistence/.../PersistenceLayerImpl.kt` | Public API (1960 lines) |
| `core/interfaces/.../db/PersistenceLayer.kt` | Interface (1444 lines) |
| `database/impl/.../di/DatabaseModule.kt` | DB creation + migrations |
