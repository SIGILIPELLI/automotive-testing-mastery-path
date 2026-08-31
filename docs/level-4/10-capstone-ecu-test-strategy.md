# 10 · Capstone — Full Test Strategy for a New ECU Program

This capstone integrates every module across all four levels into one
deliverable: a program-level test strategy for a new ECU, written the
way a test architect would actually present it before a program
kickoff review. Where Level 3 Module 10 scoped one feature's HIL test
plan, this capstone scopes an entire new ECU program end to end.

!!! note "About this module"
    No physical rig, real program, or real OEM/supplier relationship
    was used to produce this capstone. Every technique referenced
    traces back to a specific earlier module — treat this as a
    template demonstrating how those pieces compose, not a captured
    real program's strategy.

## The program: a new Battery Management System (BMS) ECU

Chosen deliberately as a new (not carryover) ECU with genuine safety
relevance (thermal runaway, overcharge protection are ASIL-relevant
hazards) and genuine cybersecurity relevance (a networked ECU
controlling energy storage), so the capstone can exercise the full
breadth of the course.

## 1. Scope and V-model placement (Level 4 Module 1)

```text
In scope: BMS software-level test strategy (CAPL/CANoe/HIL), owned by
the supplier per the collaboration model in Module 7; system
integration with the vehicle's powertrain/charging ECUs, OEM-owned.

Out of scope, explicitly: cell-chemistry-level electrochemical
validation (owned by a dedicated battery test lab, not signal-level
HIL); formal homologation testing for any regulated charging-safety
requirement (Module 8) -- this strategy produces PRE-homologation
rehearsal evidence only.
```

## 2. Requirements and ASIL allocation (Level 3 Module 4, 8)

| Requirement | Summary | ASIL |
|---|---|---|
| SW-REQ-501 | BMS shall disconnect the pack contactor within 200ms of detecting a cell overvoltage condition | ASIL D |
| SW-REQ-502 | BMS shall reject implausible cell-voltage/temperature sensor readings rather than acting on them | ASIL D |
| SW-REQ-503 | BMS shall authenticate charging-station communication before accepting a charge-current setpoint | ASIL B / CAL-relevant |

## 3. Safety mechanism and fault matrix (Level 3 Modules 4, 6; Level 4 Module 3)

| Mechanism | Fault(s) tested | Combined-fault case (Level 4 Module 3) |
|---|---|---|
| Overvoltage plausibility + contactor disconnect | Implausible cell-voltage jump, frame timeout on cell-voltage bus signal | Overvoltage condition occurring WHILE a comm timeout is active on an unrelated temperature signal |
| Charging-station authentication (Level 4 Module 4's Security Access pattern) | Invalid key, replayed setpoint message | Authentication failure occurring during an active overvoltage fault (does the ECU still prioritize the safety-critical disconnect over diagnostic/security processing?) |

```c
testcase tc_ContactorDisconnectsWithinSpec()
{
  testCaseTitle("[SW-REQ-501] Contactor disconnects within 200ms of overvoltage detection");
  dword t0, t1;
  setSignal(CellVoltage_mV, 3700); // nominal
  testWaitForTimeout(200);

  t0 = timeNowNS() / 1000000;
  setSignal(CellVoltage_mV, 4300); // overvoltage threshold exceeded
  testWaitForSignal(ContactorState, 0 /* OPEN */, 200);
  t1 = timeNowNS() / 1000000;

  testStepCheck("contactor opened within 200ms", (t1 - t0) <= 200);
}

testcase tc_SafetyDisconnectPrioritizedDuringAuthFailure()
{
  testCaseTitle("[SW-REQ-501][SW-REQ-503] Overvoltage disconnect is not delayed by concurrent charging-auth handling");
  // Combined-fault case per Level 4 Module 3's cascading-fault discipline.
  StartChargingAuthenticationAttempt(INVALID_KEY);
  setSignal(CellVoltage_mV, 4300);

  testWaitForSignal(ContactorState, 0 /* OPEN */, 200);
  testStepCheck("safety-critical disconnect not delayed by concurrent security processing", 1 == 1);
  // A real implementation would assert against a captured timestamp,
  // not a placeholder -- left here to flag in the exercise below.
}
```

## 4. Test data management (Level 4 Module 5)

```text
Configuration baseline: BMS-v1.0.0-rc1-baseline
  ECU software build:  1.0.0-rc1
  DBC:                 BMS_Network_v3.dbc
  A2L:                 BMS_1.0.0.a2l
  Requirements set:    SW-REQ-set-2026-Q3-rev1
  FMEA revision:       FMEA-BMS-rev2
```

## 5. Continuous testing integration (Level 4 Module 2)

```text
SIL smoke (minutes, every commit): overvoltage/undervoltage plausibility
  logic against a simulated cell model.
HIL smoke (tens of minutes, every commit): tc_ContactorDisconnectsWithinSpec
  and equivalent core safety-mechanism checks.
Nightly full regression: full fault matrix from Section 3, including
  combined-fault cases.
```

## 6. Supplier/OEM collaboration (Level 4 Module 7)

```text
Evidence exchanged with OEM at each milestone: summary test report,
traceability matrix (100% coverage required for ASIL D requirements),
fault-injection coverage matrix. CAPL test source withheld as supplier
IP per the negotiated evidence boundary; test *technique* per
requirement documented instead.
```

## 7. Known gaps and risk acceptance (Level 3 Module 10 style, program scale)

```text
Gap 1: No breakout-box electrical fault injection available at supplier
site for cell-voltage sensor wiring -- ASIL D coverage gap, flagged to
OEM safety assessor with target hardware-acquisition date.

Gap 2: Charging-station authentication testing (SW-REQ-503) currently
covers key-validation and lockout (Level 4 Module 4 pattern) but has
NOT yet been extended to combined-fault testing against a concurrent
overvoltage condition beyond the one testcase above -- flagged as an
open item for the next test-plan revision.
```

## 8. Team and process (Level 4 Module 6)

```text
Roles assigned: 2 test engineers (CAPL authorship), 1 framework
engineer (CI/orchestration, Level 3 Module 5 style), 1 safety test
specialist (ASIL D independent review), 1 rig engineer (HIL/fault-
injection hardware). Cross-training matrix maintained to avoid single
points of failure on the BMS-specific rig.
```

## Cheat sheet — how the capstone maps to the whole course

| Section | Modules it draws on |
|---|---|
| Scope/V-model | Level 4 Module 1 |
| Requirements/ASIL | Level 3 Modules 4, 8 |
| Fault matrix, combined faults | Level 3 Module 6, Level 4 Module 3 |
| Configuration baseline | Level 4 Module 5 |
| CI integration | Level 4 Module 2 |
| Supplier/OEM evidence boundary | Level 4 Module 7 |
| Honest gap disclosure | Level 3 Module 10's pattern, applied program-wide |
| Team structure | Level 4 Module 6 |

## Exercise

1. `tc_SafetyDisconnectPrioritizedDuringAuthFailure` contains a
   placeholder assertion (`1 == 1`) exactly as flagged in its comment.
   Rewrite it with a real timing assertion that would actually catch a
   defect where security-processing load delays the safety-critical
   contactor disconnect beyond 200ms.
2. Using the homologation overview (Level 4 Module 8), identify which
   BMS requirement in Section 2 is most likely to have a real-world
   regulatory analog, and draft a one-paragraph note for the strategy
   document distinguishing this program's internal ASIL D testing from
   any future formal homologation activity for that requirement.
3. Write Section 9 of this strategy — a "Definition of Done" for the
   whole program's test strategy (not one feature, per Level 3 Module
   10's narrower exit criteria) — covering traceability coverage
   thresholds, gap risk-acceptance sign-off, and CI health criteria
   that must all be met before this ECU is considered test-complete
   for its first release candidate.
