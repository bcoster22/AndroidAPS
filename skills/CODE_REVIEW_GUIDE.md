# Code Review Guide & Auto-Fix Thresholds

> Rubric for agents reviewing PRs and deciding when to auto-fix vs flag for human review.

## Auto-Fix vs Flag Decision Matrix

```
Change Size     Safety Level        Action
───────────     ─────────────       ──────────────────────────
1-100 lines     Non-critical        AUTO-FIX: Implement, test, create PR
1-100 lines     Safety-critical     FLAG: Create PR but request human review
100-300 lines   Non-critical        IMPLEMENT + FLAG: Create PR with detailed explanation
100-300 lines   Safety-critical     ANALYSIS ONLY: Create issue with proposed approach
300+ lines      Any level           PLAN ONLY: Create issue with design proposal, wait for approval
```

### What Is "Safety-Critical"?
Any code that can affect insulin delivery:
- Constraint implementations (`PluginConstraints`)
- Loop invocation (`LoopPlugin.invoke()`)
- APS algorithms (`DetermineBasalSMB`, `OpenAPSSMBPlugin`)
- Command queue execution (`CommandQueueImplementation`)
- Pump driver delivery methods (`deliverTreatment`, `setTempBasalAbsolute`)
- PumpSync data reporting
- IOB/COB calculations

### What Is "Non-Critical"?
- UI changes (fragments, activities, views)
- Documentation (KDoc, markdown)
- Build configuration (Gradle, CI)
- Test additions
- Logging improvements
- Sync/upload logic (Nightscout, Tidepool)
- Wear OS companion
- String resources

---

## Review Checklist

### Tier 1: Always Check (every PR)

- [ ] **Compiles**: `./gradlew compileFullDebugSources`
- [ ] **Tests pass**: `./gradlew testFullDebugUnitTest`
- [ ] **No secrets**: No API keys, tokens, passwords in diff
- [ ] **No removed tests**: Test count didn't decrease
- [ ] **Scope matches**: Changes match the stated intent (no scope creep)

### Tier 2: Code Quality (every PR)

- [ ] **ktlint**: `./gradlew ktlintCheck` — no new violations
- [ ] **detekt**: `./gradlew detekt` — no new issues
- [ ] **KDoc**: New public APIs have documentation
- [ ] **Naming**: Variables/functions follow Kotlin conventions
- [ ] **Error handling**: RxJava subscriptions have error handlers
- [ ] **Disposable management**: CompositeDisposable cleared in lifecycle methods
- [ ] **Thread safety**: UI updates on main, I/O on io scheduler

### Tier 3: Safety (when touching safety-critical code)

- [ ] **Constraints preserved**: No constraint bypassed or weakened
- [ ] **PumpSync called**: After every successful pump command
- [ ] **IOB/COB unchanged**: Calculation logic not accidentally altered
- [ ] **Edge cases**: What happens at BG=40? BG=400? IOB=maxIOB?
- [ ] **Null safety**: No `!!` on nullable values in delivery paths
- [ ] **Units correct**: mg/dL vs mmol/L, U/h vs U, minutes vs milliseconds
- [ ] **Regression tests**: New test covers the fix

---

## Common Issues to Flag

### Severity: CRITICAL (block merge)
- Any weakening of safety constraints
- Missing PumpSync call after delivery
- Division by zero in ISF/IC/basal calculations
- Unbounded insulin delivery (missing max check)
- `!!` operator on nullable in safety path

### Severity: HIGH (request changes)
- Missing error handler on RxJava subscription
- Missing disposable cleanup (memory leak)
- Main thread blocking (ANR risk)
- Hardcoded values that should be configurable
- Missing test for bug fix

### Severity: MEDIUM (comment, approve)
- Missing KDoc on public interface
- Inconsistent naming
- Unnecessary complexity (could be simpler)
- Copy-paste code (could be extracted)

### Severity: LOW (approve, optional comment)
- Minor style issues (ktlint will catch)
- Comment wording
- Import ordering

---

## Auto-Fix Examples (1-100 Lines)

### Type: Missing Error Handler
```kotlin
// BEFORE (anti-pattern spotted)
disposable += rxBus.toObservable(EventXxx::class.java)
    .subscribe { doSomething() }

// AFTER (auto-fix)
disposable += rxBus.toObservable(EventXxx::class.java)
    .observeOn(aapsSchedulers.io)
    .subscribe({ doSomething() }, fabricPrivacy::logException)
```

### Type: Missing Disposable Cleanup
```kotlin
// BEFORE (leak spotted)
override fun onPause() {
    super.onPause()
}

// AFTER (auto-fix)
override fun onPause() {
    disposable.clear()
    super.onPause()
}
```

### Type: Missing KDoc
```kotlin
// BEFORE
fun calculateIsfAdjustment(bg: Double, profile: Profile): Double {

// AFTER (auto-fix)
/**
 * Calculate ISF adjustment factor based on current BG and profile.
 *
 * @param bg Current blood glucose in mg/dL.
 * @param profile Active profile with ISF values.
 * @return Adjustment multiplier (1.0 = no change).
 */
fun calculateIsfAdjustment(bg: Double, profile: Profile): Double {
```

### Type: Unsafe Null Access
```kotlin
// BEFORE (risk spotted in non-safety code)
val rate = tempBasal!!.rate

// AFTER (auto-fix)
val rate = tempBasal?.rate ?: return
```

---

## PR Description Template (for agents)

```markdown
## What
[1-2 sentences: what changed]

## Why
[1-2 sentences: why this change is needed, link to issue]

## Safety Impact
- [ ] Non-critical (UI, docs, tests, build)
- [ ] Safety-adjacent (sync, data display)
- [ ] Safety-critical (insulin delivery, algorithms, constraints)

## Testing
- [ ] Unit tests added/updated
- [ ] Existing tests pass
- [ ] Manual testing done (describe)

## Checklist
- [ ] Compiles
- [ ] Tests pass
- [ ] ktlint clean
- [ ] detekt clean
- [ ] KDoc on new public APIs
```
