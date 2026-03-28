# Current Sprint — Active Priorities

> Agents: Read this FIRST in every new session. It tells you what to work on.
> Last updated: 2026-03-28

## Active Priority Queue

### P0 — Safety Critical (work on these first)
| Issue | Title | Status |
|-------|-------|--------|
| #29 | **CRITICAL: COB overestimation with extended carbs** | Open — needs root cause research |
| #43 | **CRITICAL: Medtrum dispenses ~10U during failed activation** | Open — needs activation state machine audit |
| #23 | Audit constraint chain for safety completeness | Open — research task |

### P1 — High Priority
| Issue | Title | Status |
|-------|-------|--------|
| #16 | Create CLAUDE.md project intelligence file | **Done** (merged PR #50) |
| #22 | Enable Dependabot for dependency scanning | Open — quick task |
| #24 | Enable GitHub secret scanning and push protection | Open — quick task |
| #32 | Combo stuck in Error state after BT issues | Open — needs state machine fix |
| #44 | Bolus progress not rounded to 0.05U | Open — has solution from upstream |

### P2 — Medium Priority
| Issue | Title | Status |
|-------|-------|--------|
| #14 | Add KDoc to remaining core interfaces | Open |
| #15 | Create CONTRIBUTING.md | Open |
| #18 | Audit test coverage and identify gaps | Open — research task |
| #25 | Set up automated upstream sync workflow | Open |
| #33 | AAPSClient loop disable doesn't cancel TBR | Open |

### P3 — Lower Priority
| Issue | Title | Status |
|-------|-------|--------|
| #10 | Set up self-hosted CircleCI runner | Open — infrastructure |
| #21 | Set up ktlint enforcement in CI | Open |
| #27 | Profile Gradle build performance | Open — research |
| #39-#42 | Feature requests (CSV export, HeartRate sync, Wear, NFC) | Open |

## Quick Wins (agents can auto-fix, <100 lines)
- #22 — Create `.github/dependabot.yml` (10 lines)
- #24 — Enable in GitHub Settings (manual by human)
- #44 — Add `Round.roundTo()` in EventOverviewBolusProgress (5-10 lines)
- #46 — Separate validation for negative carb entries (~20 lines)

## What's Been Completed This Session
- [x] CLAUDE.md + all skillset files
- [x] docs/ (ARCHITECTURE, PROJECT_STRUCTURE, DATABASE, PLUGIN_DEVELOPMENT)
- [x] KDoc on 7 core interfaces
- [x] CI/CD pipeline (CircleCI + GitHub Actions + Claude review)
- [x] detekt static analysis setup
- [x] 49 GitHub issues created and organized
- [x] CHANGELOG.md and CURRENT_SPRINT.md

## How to Pick Up Work
1. Read `CLAUDE.md` for project context
2. Read this file for priorities
3. Pick the highest-priority open issue you can handle
4. Read the relevant `skills/` file for domain knowledge
5. Create branch `claude/<short-description>`
6. Implement with tests
7. Run quality checks: `./gradlew compileFullDebugSources testFullDebugUnitTest ktlintCheck detekt`
8. Push and create PR referencing the issue
