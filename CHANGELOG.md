# Changelog — bcoster22/AndroidAPS Fork

> Running log of all work done on this fork. Agents should read this to avoid redoing work.
> Updated each time work is merged to master.

## 2026-03-28 — Initial Fork Setup

### Documentation & Architecture
- **CLAUDE.md** — Project intelligence file for Claude agents (safety rules, build commands, architecture, conventions, auto-fix thresholds, skillset references)
- **docs/ARCHITECTURE.md** — High-level system architecture with 10+ Mermaid diagrams
- **docs/PROJECT_STRUCTURE.md** — All 49 modules with dependency graph
- **docs/DATABASE.md** — ER diagram, repository pattern, transaction system, sync flow
- **docs/PLUGIN_DEVELOPMENT.md** — Plugin development guide (pump, APS, BG source, constraints)

### Agent Skillset Files
- **skills/DIABETES_DOMAIN.md** — Medical domain knowledge (IOB, COB, ISF, IC, safety boundaries)
- **skills/PUMP_DRIVER_PATTERNS.md** — Pump driver architecture, PumpSync, BLE patterns
- **skills/RXJAVA_PATTERNS.md** — RxBus, CompositeDisposable, schedulers, anti-patterns
- **skills/CODE_REVIEW_GUIDE.md** — Review rubric, auto-fix thresholds, PR template
- **skills/TESTING_STRATEGY.md** — TestBase/TestBaseWithProfile patterns, coverage targets
- **skills/DAGGER_DI_PATTERNS.md** — DI patterns across 49 modules, registration guide
- **skills/DATABASE_OPERATIONS.md** — Room transactions, migrations, PersistenceLayer
- **skills/HOTSPOT_REGISTRY.md** — 96 complex files registry with refactoring notes
- **skills/PUMP_FUNCTION_REGISTRY.md** — Capabilities of all 14 pump drivers
- **skills/COMMS_PROTOCOLS.md** — BLE, inter-app, Nightscout, Wear OS, SMS protocols
- **skills/MODULE_API_MAP.md** — Module exports/imports and boundary rules

### KDoc Comments Added
- `PluginBase` — lifecycle, constraints, parameters
- `CommandQueue` — all 25+ methods documented
- `IobCobCalculator` — all calculation methods
- `ActivePlugin` — registry purpose, usage
- `BgSource` — data reception, implementations
- `PluginConstraints` — chain-of-responsibility pattern
- `RxBus` — usage examples, thread safety

### Code Quality Tooling
- **detekt** (v1.23.7) — static analysis with medical-grade config (`config/detekt/detekt.yml`)
- **ktlint** (existing v14.0.1) — already configured
- **Android Lint** — now runs in CI

### CI/CD Pipeline
- **CircleCI** — Updated to cloud runners (replaced nightscout's self-hosted). Two workflows: full-tests (master/develop) and pr-check (branches)
- **GitHub Actions** (`fork-ci.yml`) — 4 parallel jobs: build, unit-tests, code-quality (ktlint+detekt), android-lint
- **Claude Code** (`claude.yml`) — responds to `@claude` mentions using ANTHROPIC_API_KEY
- **Claude Code Review** (`claude-code-review.yml`) — auto-reviews every PR
- Nightscout's original CI preserved as `.circleci/config.yml.nightscout`

### GitHub Issues (49 total)
- 6 epics (#4-#9): CI/CD, Documentation, Testing, Security, Upstream Sync, Performance
- 19 roadmap tasks (#10-#28)
- 21 upstream issues imported (#29-#49) with root cause analysis and proposed solutions
- Labels created: epic, roadmap, task, feature, research, ci-cd, documentation, testing, security, upstream, bug, enhancement

### Project Management
- **CURRENT_SPRINT.md** — Active priorities for agents
- **CHANGELOG.md** — This file

---

## Upstream Base
- Forked from `nightscout/AndroidAPS` at version **3.4.1.0** (versionCode 1500)
- Commit: `23f797e5c`
