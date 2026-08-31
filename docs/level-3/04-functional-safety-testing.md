# 04 · Functional Safety Testing (ISO 26262)

Everything so far has tested *whether an ECU does the right thing*.
ISO 26262 asks a different, harder question: what happens when
something goes wrong — a sensor fails, a microcontroller bit-flips, a
communication link drops — and does the system fail into a safe state
rather than a dangerous one? This module covers the ASIL model, the
safety lifecycle's testing implications, and how fault-injection style
thinking connects to the CAPL patterns already built.

!!! note "About this module"
    No certified safety-case tooling or physical fault-injection rig
    was used to produce this content. The ASIL/safety-lifecycle model
    and CAPL patterns below reflect the documented ISO 26262 standard
    and common practice — a real safety case requires a qualified
    safety engineer and traceable evidence, which this module does not
    substitute for.

## ASIL: risk-graded requirements, not a pass/fail badge

ISO 26262 assigns an **Automotive Safety Integrity Level** (QM, A, B,
C, D — D is the strictest) to each hazard based on three factors:

| Factor | Question |
|---|---|
| Severity (S0–S3) | How bad is the injury if the hazard occurs? |
| Exposure (E0–E4) | How often is the vehicle in the situation where this hazard could occur? |
| Controllability (C0–C3) | Can the driver/system reasonably avoid harm once the hazard starts? |

A steering-assist ECU losing torque request capability at highway
speed might be ASIL D (high severity, common exposure, low
controllability); a courtesy-light ECU failing might be QM (no safety
relevance at all — regular quality management is sufficient). The
ASIL doesn't attach to the whole ECU — it attaches to specific
**safety goals**, and different functions on the same ECU can carry
different ASILs.

**Testing implication:** the rigor, independence, and coverage
expected of a test scales with the ASIL of the safety goal under test.
An ASIL D safety mechanism needs traceable, reviewed, often
independently-verified test evidence; a QM function can be tested with
the lighter-weight practices from earlier modules.

## Safety mechanisms: what you're actually testing

A safety goal is met through **safety mechanisms** — specific,
identifiable pieces of design meant to detect a fault and force a safe
reaction. Common patterns:

| Mechanism | Detects | Safe reaction |
|---|---|---|
| Range check | Sensor value outside physically plausible bounds | Ignore reading, fall back to default/degraded mode |
| Plausibility/cross-check | Two independent sensors disagree beyond tolerance | Flag both suspect, degrade |
| Watchdog timeout | Software task hangs or stops running | Force ECU reset |
| E2E (end-to-end) protection | CAN/CAN-FD message corrupted, stale, or out of sequence | Reject message, treat signal as invalid |
| Diagnostic coverage via redundant computation | A calculation's result diverges from a redundant/simplified check | Trip a fault, transition to safe state |

Functional safety testing is, concretely, **testing that these
mechanisms fire correctly when the fault they're designed for actually
happens** — not testing the nominal function again.

## CAPL-driven fault injection for a safety mechanism

```c
// Illustrative — CAPL fault-injection idioms for a plausibility check.
// The two "independent" sensor signals are simulated on the bus;
// real systems may also require electrical-level fault injection
// (open/short) that CAPL alone cannot produce — see Module 6.
testcase tc_WheelSpeedPlausibilityFaultDetected()
{
  testCaseTitle("Diverging wheel-speed sensors trigger plausibility fault within spec");
  dword t0, t1;

  setSignal(WheelSpeedFL, 60.0);
  setSignal(WheelSpeedFR, 60.0);
  testWaitForTimeout(200); // let the ECU settle on plausible baseline

  t0 = timeNowNS() / 1000000;
  setSignal(WheelSpeedFR, 10.0); // inject an implausible divergence
  testWaitForSignal(PlausibilityFault_WheelSpeed, 1, 500);
  t1 = timeNowNS() / 1000000;

  testStepCheck("fault flagged within 500ms of divergence", (t1 - t0) <= 500);
  testStepCheck("ECU entered degraded mode", getSignal(SteeringAssistMode) == DEGRADED);
}

testcase tc_PlausibilityFaultClearsOnRecovery()
{
  testCaseTitle("Fault clears once sensors agree again, per the safety concept's recovery rule");
  setSignal(WheelSpeedFL, 60.0);
  setSignal(WheelSpeedFR, 10.0);
  testWaitForSignal(PlausibilityFault_WheelSpeed, 1, 500);

  setSignal(WheelSpeedFR, 60.0); // sensors agree again
  testWaitForSignal(PlausibilityFault_WheelSpeed, 0, 1000);
  testStepCheck("fault cleared per recovery timing spec", 1 == 1);
}
```

The second testcase matters as much as the first: a safety mechanism
that never clears a fault (or clears it too eagerly, before the fault
condition is genuinely gone) is its own defect, and the recovery
behavior is usually spelled out explicitly in the safety concept
document, not left to inference.

## Diagnostic coverage and single-point vs. latent faults

ISO 26262 distinguishes:

| Fault class | Meaning | Test relevance |
|---|---|---|
| Single-point fault | A fault with no independent safety mechanism protecting against it — directly violates the safety goal | Highest testing priority; the mechanism protecting it (if any) must be proven to work |
| Multi-point fault | Requires a second, independent fault to become dangerous | Test the combination, not just each fault alone, where feasible |
| Latent fault | A fault that doesn't manifest immediately but defeats a safety mechanism silently (e.g., the plausibility checker's own comparator is broken) | Needs periodic self-test evidence, not just reactive fault injection |

A latent-fault example: if the plausibility checker in the example
above has a bug that always reports "plausible" regardless of input,
`tc_WheelSpeedPlausibilityFaultDetected` catches it directly — but a
checker that *usually* works and fails only under a specific timing
race needs a dedicated latent-fault test, often a self-test or built-in
diagnostic the ECU runs on its own schedule.

## Traceability: why every safety test needs a requirement ID

Unlike a Level 1 exploratory test, a functional-safety test is
worthless as *safety evidence* unless it traces to a specific safety
requirement, and that requirement traces to a specific safety goal.
Module 8 covers traceability tooling in depth; the safety-specific
addition is that the trace must also record **which ASIL** the
requirement carries, since audit and assessment activities scale with
ASIL.

## Cheat sheet

| Concept | Key point |
|---|---|
| ASIL | Attaches to a safety goal (via S/E/C), not to a whole ECU; drives required test rigor |
| Safety mechanism | The specific design element that detects a fault and forces a safe reaction — the actual test target |
| Recovery testing | Test fault-clear behavior explicitly, not just fault-detect |
| Single-point vs. latent fault | Latent faults need periodic self-test evidence, not just one-shot fault injection |
| Traceability | Every safety test needs a requirement ID and its ASIL recorded |

## Exercise

1. Classify a brake-pedal-position sensor failing to a fixed
   mid-travel value (rather than an obviously out-of-range value) as a
   single-point, multi-point, or latent fault risk, and explain what
   safety mechanism would need to exist to catch it.
2. `tc_PlausibilityFaultClearsOnRecovery` above has a weak assertion
   (`1 == 1`) after the `testWaitForSignal` call. Rewrite it with a
   meaningful check, and explain why relying solely on
   `testWaitForSignal`'s built-in timeout-as-pass/fail is a common but
   fragile pattern.
3. Pick a safety mechanism from the table above other than
   plausibility checking, and write one testcase (detection) and one
   testcase (recovery) for it, stating what ASIL you'd assume and why
   that changes how rigorously you'd document the result.
