# RxJava3 Patterns in AndroidAPS

> Project-specific patterns for reactive programming. Follow these exactly.

## RxBus — Event Bus

### Publishing Events
```kotlin
rxBus.send(EventNewBgReading())
rxBus.send(EventRefreshOverview("reason"))
rxBus.send(EventLoopUpdateGui())
```

### Subscribing to Events
```kotlin
// CORRECT pattern — always include error handler
disposable += rxBus
    .toObservable(EventNewBG::class.java)
    .observeOn(aapsSchedulers.io)         // or .main for UI
    .subscribe({ handleEvent(it) }, fabricPrivacy::logException)

// With operators
disposable += rxBus
    .toObservable(EventNewBG::class.java)
    .observeOn(aapsSchedulers.io)
    .debounce(1L, TimeUnit.SECONDS)       // Debounce rapid events
    .subscribe({ load() }, fabricPrivacy::logException)
```

### ANTI-PATTERN: Missing Error Handler
```kotlin
// BAD — silent failures
disposable += rxBus
    .toObservable(EventXxx::class.java)
    .subscribe { doSomething() }          // NO error handler!

// GOOD — always handle errors
disposable += rxBus
    .toObservable(EventXxx::class.java)
    .subscribe({ doSomething() }, fabricPrivacy::logException)
```

## CompositeDisposable Lifecycle

### In Plugins (onStart/onStop)
```kotlin
private val disposable = CompositeDisposable()

override fun onStart() {
    super.onStart()
    disposable += rxBus
        .toObservable(EventXxx::class.java)
        .observeOn(aapsSchedulers.io)
        .subscribe({ ... }, fabricPrivacy::logException)
}

override fun onStop() {
    disposable.clear()  // Always clear on stop
    super.onStop()
}
```

### In Fragments (onResume/onPause)
```kotlin
private val disposable = CompositeDisposable()

override fun onResume() {
    super.onResume()
    disposable += rxBus
        .toObservable(EventXxx::class.java)
        .observeOn(aapsSchedulers.main)   // Main thread for UI
        .subscribe({ updateGui() }, fabricPrivacy::logException)
}

override fun onPause() {
    disposable.clear()  // Always clear on pause
    super.onPause()
}
```

### The += Operator
```kotlin
import io.reactivex.rxjava3.kotlin.plusAssign  // Required import

disposable += observable.subscribe(...)  // Adds to composite
```

## AapsSchedulers — Thread Management

```kotlin
interface AapsSchedulers {
    val main: Scheduler      // UI thread (AndroidSchedulers.mainThread())
    val io: Scheduler        // I/O operations (network, database, file)
    val cpu: Scheduler       // CPU-bound computation
    val newThread: Scheduler // New thread per subscriber
}
```

### When to Use Each
| Scheduler | Use For | Example |
|-----------|---------|---------|
| `aapsSchedulers.main` | UI updates | `updateGui()`, view binding |
| `aapsSchedulers.io` | Database, network, file I/O | `persistenceLayer.insert()` |
| `aapsSchedulers.cpu` | Heavy computation | Algorithm calculations |

### Common Pattern
```kotlin
// Database query → UI update
disposable += persistenceLayer
    .getBgReadingsDataFromTime(startTime, false)
    .observeOn(aapsSchedulers.main)           // Switch to main for UI
    .subscribe { list -> adapter.submitList(list) }
```

## Database Operations (RxJava3)

### Repository Transaction Patterns
```kotlin
// Fire-and-forget (Completable)
appRepository.runTransaction(InsertBolusTransaction(bolus))
    .subscribe()  // OK for fire-and-forget only

// Need result (Single)
appRepository.runTransactionForResult(InsertBolusTransaction(bolus))
    .subscribe(
        { result -> handleResult(result) },
        { error -> handleError(error) }
    )
```

### PersistenceLayer (preferred API)
```kotlin
// Insert with user entry logging
disposable += persistenceLayer.insertOrUpdateBolus(bolus)
    .subscribe(
        { aapsLogger.debug("Bolus saved") },
        { aapsLogger.error("Failed to save bolus", it) }
    )
```

## Common Patterns

### Combining Multiple Observables
```kotlin
// Subscribe to multiple events in onStart
override fun onStart() {
    super.onStart()
    disposable += rxBus.toObservable(EventA::class.java)
        .observeOn(aapsSchedulers.io)
        .subscribe({ processActions() }, fabricPrivacy::logException)
    disposable += rxBus.toObservable(EventB::class.java)
        .observeOn(aapsSchedulers.io)
        .subscribe({ processActions() }, fabricPrivacy::logException)
    disposable += rxBus.toObservable(EventC::class.java)
        .observeOn(aapsSchedulers.io)
        .subscribe({ processActions() }, fabricPrivacy::logException)
}
```

### Error Handling Standard
```kotlin
// Standard error handler — log to Fabric/Crashlytics
fabricPrivacy::logException

// For database operations where errors matter
.subscribe(
    { result -> /* success */ },
    { error -> aapsLogger.error(LTag.DATABASE, "Operation failed", error) }
)
```

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `.subscribe()` without error handler | Silent failures | Add `fabricPrivacy::logException` |
| `observeOn(main)` for I/O work | Main thread blocking | Use `aapsSchedulers.io` |
| Missing `disposable.clear()` | Memory leaks | Clear in onStop/onPause |
| `blockingGet()` on main thread | ANR crash | Use async subscribe |
| Creating Observables in loops | Memory pressure | Collect data, emit once |
| Not importing `plusAssign` | Compile error on `+=` | `import io.reactivex.rxjava3.kotlin.plusAssign` |
