# 03 · Advanced Fault Injection & Robustness Testing

Level 3 Module 6 built the fault taxonomy and single-fault CAPL
patterns. This module goes further: combined/cascading faults,
robustness testing philosophy (searching for the system's actual
breaking point rather than just confirming it survives one known
fault), and how to reason about fault injection when the number of
possible combinations becomes too large to test exhaustively.

!!! note "About this module"
    No physical multi-channel fault-injection hardware was used to
    produce this content. The combinatorial and robustness-testing
    patterns below reflect documented HIL and robustness-testing
    practice — verify exact fault-injection hardware capability
    against your rig before assuming any specific combination is
    physically reproducible.

## Single faults vs. cascading faults

A safety mechanism proven to catch fault A alone, and separately
proven to catch fault B alone, is not automatically proven to catch A
and B **together** — and ISO 26262's multi-point fault concept (Level
3 Module 4) exists precisely because some hazards only emerge from
combinations:

```c
// Illustrative -- testing a combination Level 3's single-fault tests
// did not cover: a wheel-speed plausibility fault occurring WHILE a
// communication timeout is already active on a different signal.
testcase tc_CombinedFault_PlausibilityDuringCommTimeout()
{
  testCaseTitle("[SW-REQ-310] Plausibility fault correctly flagged even while an unrelated comm timeout is active");

  BlockMessage(SteeringAngleSensor); // fault 1: unrelated signal already down
  testWaitForSignal(CommTimeoutFault_SteeringAngle, 1, 500);

  setSignal(WheelSpeedFL, 60.0);
  setSignal(WheelSpeedFR, 60.0);
  testWaitForTimeout(200);
  setSignal(WheelSpeedFR, 10.0); // fault 2: independent plausibility fault, injected while fault 1 is still active

  testWaitForSignal(PlausibilityFault_WheelSpeed, 1, 500);
  testStepCheck("second, independent fault still correctly detected under existing fault load",
                 getSignal(PlausibilityFault_WheelSpeed) == 1);
  testStepCheck("first fault still correctly flagged, not masked by the second",
                 getSignal(CommTimeoutFault_SteeringAngle) == 1);

  UnblockMessage(SteeringAngleSensor);
}
```

The second assertion is the one single-fault testing structurally
cannot catch: a fault-handling implementation that only tracks "one
active fault at a time" (a single global fault-state variable, say)
would silently lose the first fault's flag when the second one fires —
a real and common defect class in fault-management software.

## Combinatorial explosion, and how to manage it

With N independent fault types across M signals, exhaustive
pairwise-or-worse combination testing grows combinatorially and
quickly becomes infeasible. Two standard techniques narrow this:

| Technique | How it works | Trade-off |
|---|---|---|
| Risk-prioritized pairing | Only combine faults where the FMEA (Level 3 Module 6) or safety case flags a plausible common-cause or interaction risk | Requires good FMEA input; won't catch an interaction nobody anticipated |
| Pairwise (all-pairs) combinatorial testing | A test-design technique guaranteeing every *pair* of fault conditions is covered at least once, without covering every full combination | Well-established for reducing test count while retaining strong defect-finding power for pairwise interactions specifically; does not guarantee catching 3-way-or-higher interactions |

```python
# Illustrative -- generating a minimal pairwise combination set for
# 3 fault dimensions using a simple (non-optimal) approach; production
# pairwise testing typically uses a dedicated tool/algorithm (e.g. an
# orthogonal-array or IPOG-based generator) for larger fault spaces.
import itertools

fault_dims = {
    "wheel_speed_fault": ["none", "implausible", "timeout"],
    "steering_fault": ["none", "timeout"],
    "brake_pedal_fault": ["none", "stuck_mid_travel"],
}

all_combinations = list(itertools.product(*fault_dims.values()))
print(f"Full combinatorial set: {len(all_combinations)} cases")
# A real pairwise generator would reduce this while still covering
# every (wheel_speed_fault, steering_fault) pair, every
# (wheel_speed_fault, brake_pedal_fault) pair, and every
# (steering_fault, brake_pedal_fault) pair at least once.
```

## Robustness testing: finding the breaking point, not confirming survival

Fault injection (Level 3 Module 6) proves specific, anticipated faults
are handled. **Robustness testing** asks a broader question: pushed
hard enough, in ways not specifically anticipated, does the system
degrade gracefully or fail catastrophically?

| Technique | What it stresses |
|---|---|
| Boundary value flooding | Rapidly oscillate a signal across its valid/invalid boundary many times per second — does the fault-detection debounce logic hold up, or does rapid flapping confuse it? |
| Resource exhaustion | Flood the bus with high-priority traffic to see if the ECU under test still meets its timing budgets under contention, not just nominal load |
| Long-duration soak | Run a nominal scenario for many hours — does a slow resource leak or counter overflow eventually cause a failure a short test would never see? |

```c
// Illustrative -- boundary-flooding robustness test.
testcase tc_Robustness_RapidBoundaryFlapping()
{
  testCaseTitle("[ROBUSTNESS-014] Rapid oscillation across plausibility boundary does not corrupt fault state");
  int i;
  for (i = 0; i < 200; i++)
  {
    setSignal(WheelSpeedFR, 60.0);
    testWaitForTimeout(5);
    setSignal(WheelSpeedFR, 10.0);
    testWaitForTimeout(5);
  }
  testWaitForTimeout(500);
  // The specific expected end-state depends on the debounce design --
  // the point of this test is that SOME well-defined, documented
  // state is reached, not that the ECU crashes or hangs.
  testStepCheck("ECU still responsive to diagnostics after flapping", EcuRespondsToDiagnostics() == 1);
}
```

Note the assertion style: robustness testing often can't assert one
specific "correct" outcome the way functional testing does — it
asserts the system didn't fail catastrophically (crash, hang,
unresponsive) even if the exact fault-state outcome under extreme
input is a secondary concern.

## Cheat sheet

| Concept | Key point |
|---|---|
| Cascading/combined faults | Single-fault-clean does not imply combination-clean; test independent-fault tracking explicitly |
| Combinatorial explosion | Exhaustive combination testing is usually infeasible — use risk-prioritized or pairwise selection |
| Pairwise testing | Covers every pair of conditions with far fewer cases than full combinatorics; doesn't guarantee 3-way interactions |
| Robustness testing | Looks for graceful degradation under unanticipated stress, not confirmation of one anticipated fault |
| Robustness assertions | Often "didn't crash/hang," not one single expected end-state |

## Exercise

1. `tc_CombinedFault_PlausibilityDuringCommTimeout` tests faults on
   two different signals. Write a second combined-fault testcase
   where both faults land on related redundant signals (e.g., two
   different wheel-speed sensors both going implausible at once,
   30ms apart) and explain what different failure mode this
   specifically targets versus the original testcase.
2. Using the pairwise-generation sketch, extend `fault_dims` with a
   fourth dimension (`comm_load: ["nominal", "high"]`) and explain, in
   plain terms, why a full combinatorial set grows faster than a
   pairwise set as dimensions are added.
3. Design a long-duration soak robustness test for the fault-flagging
   mechanism from Level 3 Module 4 (wheel-speed plausibility), running
   for a simulated 24 hours of intermittent fault/clear cycles.
   State what specific counter or state variable you'd watch for
   overflow or drift, and why a short test would miss it.
