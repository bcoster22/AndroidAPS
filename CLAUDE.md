# CLAUDE.md — AndroidAPS Project Intelligence

> This file is automatically read by Claude Code agents working on this repository.
> It provides project context, conventions, safety rules, and workflow guidance.

## Project Overview

**AndroidAPS** is an open-source **artificial pancreas system** (APS) for Android.
It reads CGM (continuous glucose monitor) data, runs a closed-loop algorithm,
and commands an insulin pump to automate insulin delivery for people with Type 1 diabetes.

**This is medical device software. Safety is paramount.**

- **Fork**: `bcoster22/AndroidAPS` (forked from `nightscout/AndroidAPS`)
- **Version**: 3.4.1.0 (versionCode 1500)
- **Language**: Kotlin (JDK 21, compileSdk 36, minSdk 31)
- **Architecture**: Plugin-based, 49 Gradle modules
- **DI**: Dagger 2.57
- **Reactive**: RxJava3 + RxBus event bus
- **Database**: Room 2.8 (SQLite, version 31)
- **Build**: Gradle 8.13 with KSP, Kotlin 2.2

---

## Safety Rules (MANDATORY)

These rules are **non-negotiable** for any agent working on this codebase:

1. **NEVER weaken safety constraints** — Do not modify `PluginConstraints` implementations
   to be less restrictive without explicit human approval
2. **NEVER bypass ConstraintsChecker** — All insulin delivery values (basal, bolus, SMB)
   MUST pass through `ConstraintsChecker` before reaching the pump
3. **NEVER modify insulin delivery logic** without explicit human approval — This includes
   `LoopPlugin.invoke()`, `DetermineBasalSMB`, `CommandQueueImplementation`, and pump drivers
4. **NEVER commit secrets** — No API keys, tokens, passwords, or keystores in code
5. **NEVER remove or reduce test coverage** — Tests are a safety net for medical software
6. **ALWAYS preserve existing behavior** when refactoring — Use existing tests to verify
7. **ALWAYS add tests** for any bug fix or new logic in safety-critical paths
8. **ALL pump commands** must go through `CommandQueue` — never call `Pump` methods directly

### Safety-Critical Paths (require extra care)

```
core/interfaces/constraints/          — Safety constraint chain
core/interfaces/aps/Loop.kt           — Loop coordinator
core/interfaces/pump/Pump.kt          — Pump driver contract
core/interfaces/pump/PumpSync.kt      — Pump data sync
core/interfaces/queue/CommandQueue.kt  — Pump command serialization
plugins/aps/loop/LoopPlugin.kt        — Main loop implementation
plugins/aps/openAPSSMB/               — APS algorithm (DetermineBasalSMB)
plugins/constraints/                   — Safety constraint plugins
implementation/queue/                  — CommandQueue implementation
pump/*/                                — All pump drivers
```

---

## Quick Reference — Build & Test Commands

```bash
# Compile (fast check)
./gradlew compileFullDebugSources

# Unit tests
./gradlew testFullDebugUnitTest

# Instrumentation tests (requires emulator)
./gradlew connectedFullDebugAndroidTest

# Code coverage
./gradlew jacocoAllDebugReport
# Report: build/reports/jacoco/jacocoAllDebugReport/

# Lint / style
./gradlew ktlintCheck                    # Check Kotlin style
./gradlew ktlintFormat                   # Auto-fix style issues
./gradlew :app:lintFullDebug             # Android Lint

# detekt (static analysis)
./gradlew detekt                         # Run detekt analysis
# Config: config/detekt/detekt.yml

# Build APK
./gradlew :app:assembleFullDebug
./gradlew :app:assembleFullRelease       # Needs signing config

# Clean
./gradlew clean
```

### Gradle Performance Flags

Already configured in `gradle.properties`:
- 8GB heap, 12 workers, parallel compilation
- Use `--build-cache` and `--configuration-cache` for speed

---

## Architecture Quick Reference

### Layered Architecture
```
UI Layer         → Activities, Fragments, Wear, Widgets
Application      → MainApp (DaggerApplication), ConfigBuilder, ActivePlugin
Plugin Layer     → APS, Pump, Source, Insulin, Sensitivity, Constraints, Sync
Core Layer       → Interfaces, Data Models, Utilities
Data Layer       → Room Database, Repository, PersistenceLayer
```

### Key Interfaces (in `core/interfaces/`)

| Interface | Purpose |
|-----------|---------|
| `Pump` | Pump driver contract |
| `APS` | Algorithm plugin contract |
| `Loop` | Loop coordinator |
| `BgSource` | CGM data source |
| `CommandQueue` | Serialized pump commands |
| `IobCobCalculator` | IOB/COB computation |
| `PluginConstraints` | Safety constraint chain |
| `ActivePlugin` | Active plugin registry |
| `PumpSync` | Pump <-> AAPS data sync |
| `RxBus` | Event bus (publish-subscribe) |
| `PersistenceLayer` | Database operations |

### Plugin System
- All features are plugins extending `PluginBase`
- One active plugin per type (pump, APS, BG source, etc.)
- Plugins registered via Dagger modules in `app/src/main/kotlin/app/aaps/di/`
- Lifecycle: `NOT_INITIALIZED` → `ENABLED` (onStart) / `DISABLED` (onStop)

### Data Flow
```
CGM → BgSource plugin → Database → RxBus(EventNewBgReading) → LoopPlugin
→ APS algorithm → ConstraintsChecker → CommandQueue → Pump driver → Physical pump
```

### Event Bus (RxBus)
- Publish: `rxBus.send(EventXxx())`
- Subscribe: `rxBus.toObservable(EventXxx::class.java).subscribe { ... }`
- Always use `.observeOn(aapsSchedulers.main)` for UI updates

---

## Documentation

| Document | Location | Contents |
|----------|----------|----------|
| Architecture | `docs/ARCHITECTURE.md` | System diagrams, data flow, plugin system |
| Project Structure | `docs/PROJECT_STRUCTURE.md` | All 49 modules explained |
| Database | `docs/DATABASE.md` | ER diagram, repository pattern, sync flow |
| Plugin Development | `docs/PLUGIN_DEVELOPMENT.md` | How to create plugins |
| This File | `CLAUDE.md` | Agent context and conventions |

### Skillset Files (Agent Training)

Read the relevant skill file before working in that domain:

| Skill | Location | Read When |
|-------|----------|-----------|
| Diabetes Domain | `skills/DIABETES_DOMAIN.md` | Touching ANY insulin/algorithm/constraint code |
| Pump Drivers | `skills/PUMP_DRIVER_PATTERNS.md` | Working on pump/ modules |
| RxJava3 Patterns | `skills/RXJAVA_PATTERNS.md` | Adding/modifying reactive code |
| Code Review Guide | `skills/CODE_REVIEW_GUIDE.md` | Reviewing PRs or deciding auto-fix vs flag |
| Testing Strategy | `skills/TESTING_STRATEGY.md` | Writing or reviewing tests |
| Dagger DI | `skills/DAGGER_DI_PATTERNS.md` | Adding injectable components |
| Database Ops | `skills/DATABASE_OPERATIONS.md` | Working with Room/transactions |
| Hotspot Registry | `skills/HOTSPOT_REGISTRY.md` | Modifying files >500 lines |

### Auto-Fix Thresholds

```
Change Size     Safety Level        Action
1-100 lines     Non-critical        AUTO-FIX: Implement, test, create PR
1-100 lines     Safety-critical     FLAG: Create PR but request human review
100-300 lines   Non-critical        IMPLEMENT + FLAG: Detailed PR explanation
100-300 lines   Safety-critical     ANALYSIS ONLY: Create issue with proposal
300+ lines      Any level           PLAN ONLY: Issue with design, wait for approval
```

---

## Code Quality Standards

### Medical Device Quality Requirements

This project follows **IEC 62304** software lifecycle principles for medical device software:
- All changes must be tested
- Safety-critical code requires additional review
- Traceability from requirements to tests
- No unreviewed code in safety-critical paths

### Kotlin Style
- **ktlint** enforced via `org.jlleitschuh.gradle.ktlint` plugin (v14.0.1)
- EditorConfig: `.editorconfig` (wildcard imports allowed)
- Run `./gradlew ktlintFormat` before committing

### Static Analysis
- **detekt** for code smells, complexity, and potential bugs
  - Config: `config/detekt/detekt.yml`
  - Run: `./gradlew detekt`
- **Android Lint** for Android-specific issues
  - Run: `./gradlew :app:lintFullDebug`

### Testing
- **Framework**: JUnit 5 + Mockito 5.21
- **Base classes**: `TestBase` (standard), `TestBaseWithProfile` (algorithm tests)
- **Mocks**: `SharedPreferencesMock`, `BundleMock`, `TestPumpPlugin`
- **Location**: `src/test/kotlin/` (unit), `src/androidTest/kotlin/` (instrumentation)
- **Coverage**: Jacoco — target minimum 60% for new code

### Code Review Checklist (for agents and humans)
- [ ] Does it compile? (`./gradlew compileFullDebugSources`)
- [ ] Do tests pass? (`./gradlew testFullDebugUnitTest`)
- [ ] Does ktlint pass? (`./gradlew ktlintCheck`)
- [ ] Does detekt pass? (`./gradlew detekt`)
- [ ] Are safety constraints preserved?
- [ ] Are new tests added for new logic?
- [ ] Is KDoc added for new public interfaces?
- [ ] No secrets or credentials in code?

---

## Conventions

### Branching
- `master` — stable, fork-specific
- `develop` — integration branch
- `feature/<name>` — new features
- `bugfix/<name>` — bug fixes
- `claude/<description>` — Claude agent work branches

### Commit Messages
Follow conventional commits:
```
type: short description

Longer explanation if needed.

https://claude.ai/code/session_XXX  (if from Claude Code)
```
Types: `feat`, `fix`, `docs`, `test`, `refactor`, `ci`, `chore`

### KDoc
- Required on all public interfaces and classes
- Include `@param`, `@return`, `@see` where helpful
- Focus on "why" not "what" — the code shows what, docs explain why

### Data Model Abbreviations
| Abbreviation | Full Name | Purpose |
|---|---|---|
| `GV` | GlucoseValue | CGM reading |
| `BS` | Bolus | Insulin bolus |
| `TB` | TemporaryBasal | Temp basal rate |
| `EB` | ExtendedBolus | Extended bolus |
| `CA` | Carbs | Carb entry |
| `TT` | TempTarget | Temporary BG target |
| `TE` | TherapyEvent | Sensor/site changes |
| `PS` | ProfileSwitch | Profile change |
| `RM` | RunningMode | Loop mode |
| `UE` | UserEntry | Audit trail |
| `TDD` | TotalDailyDose | Daily insulin totals |

---

## CI/CD Pipeline

### GitHub Actions (`fork-ci.yml`)
Triggers automatically on PRs to master/develop:
1. **Build** — compile check
2. **Unit Tests** — `testFullDebugUnitTest` with result reporting
3. **Lint** — ktlint check
4. **detekt** — static analysis

### CircleCI (`.circleci/config.yml`)
- **PR branches**: Quick compile check (`build-only`)
- **master/develop**: Full test suite + Jacoco coverage + Codecov upload

### Claude Workflows
- `claude.yml` — responds to `@claude` mentions in issues/PRs
- `claude-code-review.yml` — auto-reviews every new PR

---

## Agent Workflow

### When assigned an issue via `@claude`:
1. Read the issue description and all comments
2. Read this CLAUDE.md for context
3. Read relevant docs in `docs/`
4. Create a branch: `claude/<short-description>`
5. Make changes with tests
6. Run: `./gradlew compileFullDebugSources testFullDebugUnitTest ktlintCheck detekt`
7. Commit with conventional commit message
8. Push and create PR referencing the issue
9. Wait for CI and review

### When reviewing a PR:
1. Check the code review checklist above
2. Verify safety constraints are preserved
3. Check for test coverage on new logic
4. Verify KDoc on new public APIs
5. Post constructive feedback

---

## Key Files Quick Reference

| Purpose | Path |
|---------|------|
| App entry point | `app/src/main/kotlin/app/aaps/MainApp.kt` |
| Main activity | `app/src/main/kotlin/app/aaps/MainActivity.kt` |
| DI component | `app/src/main/kotlin/app/aaps/di/AppComponent.kt` |
| Plugin registry | `app/src/main/kotlin/app/aaps/di/PluginsListModule.kt` |
| Loop implementation | `plugins/aps/src/main/kotlin/.../loop/LoopPlugin.kt` |
| SMB algorithm | `plugins/aps/src/main/kotlin/.../openAPSSMB/DetermineBasalSMB.kt` |
| Command queue | `implementation/src/main/kotlin/.../queue/CommandQueueImplementation.kt` |
| Database | `database/impl/src/main/kotlin/.../AppDatabase.kt` |
| Repository | `database/impl/src/main/kotlin/.../AppRepository.kt` |
| Build config | `build.gradle.kts`, `gradle/libs.versions.toml` |
| CI config | `.circleci/config.yml`, `.github/workflows/fork-ci.yml` |
| Quality config | `config/detekt/detekt.yml`, `.editorconfig` |

---

## Upstream Tracking

- **Upstream**: `nightscout/AndroidAPS`
- **Fork base version**: 3.4.1.0
- **Fork-specific files**: `CLAUDE.md`, `docs/`, CI configs, KDoc additions, quality tooling
- **Nightscout original CI**: `.circleci/config.yml.nightscout`
- **Sync strategy**: Periodic manual sync via upstream sync workflow

## GitHub Issues

Use labels to find work:
- `epic` — major roadmap areas (#4-#9)
- `task` / `feature` / `research` — individual work items
- `bug` — bugs (including upstream)
- `security` — safety-critical items (highest priority)
- `upstream` — issues from nightscout/AndroidAPS
- `ci-cd` — build and CI infrastructure
- `documentation` — docs and KDoc work
- `testing` — test coverage and quality
