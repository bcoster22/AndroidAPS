# Diabetes Domain Knowledge for Agents

> Essential medical knowledge required to work safely on AndroidAPS code.
> **Read this before touching any insulin delivery, algorithm, or constraint code.**

## Core Concepts

### Blood Glucose (BG)
- Measured in **mg/dL** (US) or **mmol/L** (EU). Conversion: mg/dL / 18 = mmol/L
- **Normal range**: 70-180 mg/dL (3.9-10 mmol/L)
- **Hypoglycemia** (low): < 70 mg/dL — **dangerous, can be fatal**
- **Hyperglycemia** (high): > 180 mg/dL — harmful long-term
- **Target range** in AAPS: typically 80-120 mg/dL

### Insulin
- Lowers blood glucose. **Too much insulin = hypoglycemia = medical emergency**
- Delivered via insulin pump in **units (U)**
- Types of delivery:
  - **Basal**: Continuous background insulin (e.g., 0.8 U/h)
  - **Bolus**: One-time dose for meals or corrections (e.g., 5.0 U)
  - **SMB (Super Micro Bolus)**: Small automatic corrections (e.g., 0.3 U)

### Carbohydrates (Carbs)
- Raises blood glucose. Measured in **grams (g)**
- Entered by user when eating
- Can be "extended" over time for slow-absorbing foods

---

## Key Metrics (Used Throughout the Code)

### IOB — Insulin on Board
- **What**: Active insulin still working in the body from previous doses
- **Units**: U (insulin units)
- **DIA**: Duration of Insulin Action (typically 5-7 hours)
- **Why it matters**: IOB prevents stacking — the algorithm won't deliver more insulin if there's already enough active
- **Code**: `IobCobCalculator.calculateFromTreatmentsAndTemps()` → `IobTotal`

### COB — Carbs on Board
- **What**: Carbs still being absorbed from previous meals
- **Units**: g (grams)
- **Why it matters**: COB tells the algorithm food is still raising BG, affecting SMB decisions
- **Code**: `IobCobCalculator.getCobInfo()` → `CobInfo`
- **Safety note**: Issue #29 — COB doubling bug with extended carbs is safety-critical

### ISF — Insulin Sensitivity Factor
- **What**: How much 1 unit of insulin lowers BG
- **Units**: mg/dL/U (e.g., ISF=40 means 1U drops BG by 40 mg/dL)
- **Varies**: By time of day, activity, stress
- **Code**: `APS.getIsfMgdl()`, profile ISF blocks

### IC — Insulin-to-Carb Ratio
- **What**: How many grams of carbs 1 unit of insulin covers
- **Units**: g/U (e.g., IC=10 means 1U covers 10g of carbs)
- **Code**: `APS.getIc()`, profile IC blocks

### TBR — Temporary Basal Rate
- **What**: Temporary adjustment to basal rate (higher or lower than profile)
- **How**: Set as absolute (U/h) or percent (0-500% of profile)
- **Duration**: Minutes
- **0% TBR = zero temp**: No insulin delivery (used for hypo prevention)
- **Code**: `Pump.setTempBasalAbsolute()`, `CommandQueue.tempBasalAbsolute()`

### TT — Temporary Target
- **What**: Temporary BG target range (different from profile target)
- **Common uses**:
  - **Eating soon** (lower target): More aggressive insulin delivery before meals
  - **Activity** (higher target): Less insulin during exercise
  - **Hypo treatment** (higher target): Reduce insulin during low BG recovery

---

## How the Closed Loop Works (Simplified)

```
Every 5 minutes:
1. Read latest BG from CGM
2. Calculate IOB (how much insulin is still active)
3. Calculate COB (how many carbs are still absorbing)
4. Run autosensitivity (is the patient more/less sensitive today?)
5. Predict where BG is heading
6. Calculate optimal basal rate to reach target
7. Optionally calculate SMB (micro-correction bolus)
8. Apply safety constraints (max IOB, max basal, max SMB)
9. Send TBR and/or SMB command to pump
```

## Running Modes

| Mode | Behavior |
|------|----------|
| `CLOSED_LOOP` | Automatic TBR + SMB adjustments |
| `OPEN_LOOP` | Suggests actions, user must approve |
| `CLOSED_LOOP_LGS` | Low Glucose Suspend — only reduces/stops insulin when BG is dropping low |
| `DISABLED_LOOP` | No automatic actions |
| `SUSPENDED_BY_USER` | User manually paused the system |
| `DISCONNECTED_PUMP` | Pump physically disconnected |

---

## Safety Boundaries (Hard Limits)

These are absolute limits that CANNOT be exceeded regardless of algorithm output:

| Parameter | Typical Max | Why |
|-----------|------------|-----|
| Max IOB | 7-15 U | Prevents insulin stacking |
| Max basal | 2-5x profile basal | Prevents runaway delivery |
| Max SMB | ~50% of max IOB | Limits single correction dose |
| Max bolus | 10-25 U | Prevents accidental large dose |
| Max carbs | 50-150 g | Prevents entry errors |
| Min BG for SMB | ~70 mg/dL | No corrections when already low |

---

## Common Abbreviations in Code

| Abbreviation | Meaning |
|---|---|
| BG | Blood Glucose |
| CGM | Continuous Glucose Monitor |
| IOB | Insulin on Board |
| COB | Carbs on Board |
| ISF | Insulin Sensitivity Factor |
| IC / CR | Insulin-to-Carb Ratio |
| TBR / TBR | Temporary Basal Rate |
| SMB | Super Micro Bolus |
| UAM | Unannounced Meals (algorithm detects carbs without user input) |
| DIA | Duration of Insulin Action |
| TDD | Total Daily Dose |
| TT | Temporary Target |
| EPS | Effective Profile Switch |
| APS | Artificial Pancreas System |
| LGS | Low Glucose Suspend |
| CAGE | Cannula Age |
| IAGE | Insulin Age (reservoir age) |
| SAGE | Sensor Age |

---

## What Can Go Wrong (Agent Must Understand)

| Scenario | Cause | Consequence |
|----------|-------|-------------|
| **Too much insulin** | Algorithm bug, constraint bypass | Hypoglycemia → seizures, coma, death |
| **Too little insulin** | Algorithm stops working | Hyperglycemia → DKA (hours-days) |
| **Stale data** | CGM disconnected, old readings used | Wrong decisions by algorithm |
| **Double counting** | COB or IOB calculated twice | Excessive or insufficient insulin |
| **Pump communication failure** | BLE disconnect | Pump continues last TBR indefinitely |
| **Wrong profile** | ISF/IC/basal values wrong | All calculations off |

**The first scenario is the most dangerous and most immediate.** This is why the constraint system exists — it's the last line of defense.
