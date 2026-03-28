# Dagger Dependency Injection Patterns

> How DI works across 49 modules with 100+ injectable components.

## Architecture

```
AppComponent (@Singleton, @Component)
├── AndroidInjectionModule         (Dagger Android)
├── AppModule                      (Context, SharedPrefs, Plugin list)
├── ActivitiesModule               (Activity injection)
├── ReceiversModule                (BroadcastReceiver injection)
├── DatabaseModule                 (Room DB singleton)
├── ImplementationModule           (Interface → Impl bindings)
├── SharedImplModule               (SP, Logger, RxBus, DateUtil)
├── ApsModule                      (APS plugins + Loop binding)
├── SourceModule                   (BG source plugins)
├── SyncModule                     (NS, Tidepool, xDrip, WorkManager)
├── AutomationModule               (Automation engine)
├── DanaModule, OmnipodModule...   (Pump driver modules)
└── ... (30+ more modules)
```

## Injection Patterns

### Pattern 1: Constructor Injection (PREFERRED for plugins/services)
```kotlin
@Singleton
class MyPlugin @Inject constructor(
    aapsLogger: AAPSLogger,           // Pass to PluginBase
    rh: ResourceHelper,               // Pass to PluginBase
    private val rxBus: RxBus,          // Keep as private val
    private val persistenceLayer: PersistenceLayer,
    private val dateUtil: DateUtil
) : PluginBase(
    PluginDescription()
        .mainType(PluginType.GENERAL)
        .pluginName(R.string.my_plugin),
    aapsLogger, rh
)
```

### Pattern 2: Property Injection (for Fragments/Activities)
```kotlin
class MyFragment : DaggerFragment() {
    @Inject lateinit var rxBus: RxBus
    @Inject lateinit var aapsSchedulers: AapsSchedulers
    @Inject lateinit var persistenceLayer: PersistenceLayer
    @Inject lateinit var fabricPrivacy: FabricPrivacy
}
```

### Pattern 3: Action/Worker Injection (HasAndroidInjector)
```kotlin
class MyAction(injector: HasAndroidInjector) : Action(injector) {
    @Inject lateinit var rxBus: RxBus
    @Inject lateinit var dateUtil: DateUtil

    init {
        injector.androidInjector().inject(this)  // Manual injection
    }
}
```

## Module Patterns

### Pattern A: Abstract Module with @Binds (interface → impl)
```kotlin
@Module(includes = [MyModule.Bindings::class])
abstract class MyModule {

    @ContributesAndroidInjector
    abstract fun contributesMyFragment(): MyFragment

    @Module
    interface Bindings {
        @Binds fun bindMyInterface(impl: MyImpl): MyInterface
    }
}
```

### Pattern B: Concrete Module with @Provides
```kotlin
@Module
open class SharedImplModule {

    @Provides
    @Singleton
    fun provideRxBus(schedulers: AapsSchedulers, logger: AAPSLogger): RxBus =
        RxBusImpl(schedulers, logger)

    @Provides
    @Singleton
    fun provideSchedulers(): AapsSchedulers = AapsSchedulersImpl()
}
```

### Pattern C: Plugin Module with Workers
```kotlin
@Module(includes = [SyncModule.Binding::class, SyncModule.Provide::class])
abstract class SyncModule {

    @ContributesAndroidInjector abstract fun contributesFragment(): NSClientFragment
    @ContributesAndroidInjector abstract fun contributesWorker(): DataSyncWorker
    @ContributesAndroidInjector abstract fun contributesService(): NSClientV3Service

    @Module
    open class Provide {
        @Reusable
        @Provides
        fun providesWorkManager(context: Context) = WorkManager.getInstance(context)
    }

    @Module
    interface Binding {
        @Binds fun bindSyncData(impl: StoreDataForDbImpl): StoreDataForDb
    }
}
```

## Adding a New Injectable Component

### Step 1: Define interface in `core/interfaces/`
```kotlin
interface MyService {
    fun doWork(): Result
}
```

### Step 2: Implement in `implementation/` or `plugins/`
```kotlin
@Singleton
class MyServiceImpl @Inject constructor(
    private val logger: AAPSLogger
) : MyService {
    override fun doWork(): Result = ...
}
```

### Step 3: Bind in appropriate module
```kotlin
// In ImplementationModule or your plugin module
@Binds fun bindMyService(impl: MyServiceImpl): MyService
```

### Step 4: Inject wherever needed
```kotlin
class Consumer @Inject constructor(
    private val myService: MyService  // Dagger resolves MyServiceImpl
)
```

## Registering a New Plugin

1. Add module to `app/src/main/kotlin/app/aaps/di/AppComponent.kt`:
```kotlin
@Component(modules = [
    ...,
    MyPluginModule::class,  // Add here
])
```

2. Add plugin to list in `PluginsListModule`:
```kotlin
@Provides
fun providePluginsList(
    ...,
    myPlugin: MyPlugin,  // Add parameter
): List<PluginBase> = listOf(
    ...,
    myPlugin,            // Add to list
)
```

## Scopes

| Scope | Meaning | Use For |
|-------|---------|---------|
| `@Singleton` | One instance per app | Plugins, services, repositories |
| `@Reusable` | May be shared, not guaranteed | Utilities, providers |
| (none) | New instance each time | Data objects, temporary |

## Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Missing `@Inject` on constructor | Dagger can't create instance | Add `@Inject constructor` |
| Missing `@ContributesAndroidInjector` | Fragment/Activity crash on inject | Add to module |
| Missing module in `AppComponent` | Unresolved binding at compile | Add module to includes |
| Circular dependency | Compile error | Use `Lazy<T>` or `Provider<T>` |
| Wrong scope (`@Singleton` on fragment) | Leak | Don't scope fragments |
| Missing `abstract` on module class | Compile error for @Binds | Make module abstract |

## Key Files

| File | Purpose |
|------|---------|
| `app/di/AppComponent.kt` | Root component — lists all modules |
| `app/di/AppModule.kt` | Context, SharedPrefs, plugin list |
| `app/di/PluginsListModule.kt` | Plugin registration |
| `app/di/ActivitiesModule.kt` | Activity/Fragment injection |
| `shared/impl/di/SharedImplModule.kt` | Core singletons (SP, RxBus, Logger) |
| `implementation/di/ImplementationModule.kt` | Interface → Impl bindings |
| `database/impl/di/DatabaseModule.kt` | Room DB creation |
