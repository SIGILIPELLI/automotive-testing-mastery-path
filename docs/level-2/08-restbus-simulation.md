# 08 · Restbus Simulation

Testing a single ECU on a bench almost never means that ECU is alone on
the bus in the real vehicle. It expects dozens of other messages from
other nodes — wheel speed, ignition state, gear position — to keep
arriving on schedule. **Restbus simulation** ("rest of bus") is the
practice of using CANoe/CAPL to stand in for every node you're *not*
physically testing, so the ECU under test believes it's in a complete,
normal vehicle network.

!!! note "About this module"
    The restbus generation and CAPL simulation patterns below are
    genuine, documented CANoe behavior. Nothing here was captured from
    a live bench — treat every snippet as a reviewed reference and
    verify actual cycle-time fidelity against your own hardware before
    trusting timing-sensitive results.

## Why you need it at all

An ECU under test frequently has plausibility and timeout logic that
depends on *other* nodes' signals arriving on schedule (Level 1's
watchdog/failsafe pattern, generalized across the whole network). If
you bench-test that ECU alone, with no other traffic on the bus, one of
two things happens depending on the ECU's design:

- It correctly detects the network as incomplete and enters a fault
  mode — which is the *right* behavior in real life, but means your
  test bench can never observe normal operation without a restbus.
- It silently free-runs without those signals, masking a real
  dependency your test would otherwise never catch until vehicle
  integration.

Either way, a restbus simulation that faithfully reproduces the
missing nodes' cycle times, startup behavior, and signal ranges is a
prerequisite for almost any bench-level functional test beyond the
simplest cases.

## Generating a restbus from a DBC automatically

CANoe can auto-generate a baseline restbus simulation directly from a
DBC's declared cycle-time attributes (Module 6):

```text
BA_DEF_ BO_ "GenMsgCycleTime" INT 0 60000;
BA_ "GenMsgCycleTime" BO_ 256 10;   -- EngineStatus every 10ms
BA_ "GenMsgCycleTime" BO_ 512 20;   -- BrakeStatus every 20ms
```

CANoe reads these attributes and can generate a default network node
that transmits every message at its declared cycle time with default
(usually zero or start-value) signal content. This gets you a
network that's *present* but not necessarily *meaningful* — the
ECU under test sees traffic arriving on schedule, but every signal
sitting at a default value (`EngineRunning=0`, `VehicleSpeed=0`) may
itself trip preconditions your test actually needs varied.

## Writing a meaningful restbus in CAPL

```c
variables
{
  message EngineStatus engineMsg;
  message BrakeStatus brakeMsg;
  int gVehicleSpeedKph;
}

on start
{
  engineMsg.EngineRunning = 1;
  engineMsg.EngineRPM = 800;   // idle
  gVehicleSpeedKph = 0;
  setTimer(txEngine, 10);
  setTimer(txBrake, 20);
}

on timer txEngine
{
  output(engineMsg);
  setTimer(txEngine, 10);   // reschedule for continuous cycle-accurate transmission
}

on timer txBrake
{
  brakeMsg.BrakePressure = 0.0;
  output(brakeMsg);
  setTimer(txBrake, 20);
}

// Test scripts drive scenario changes through exported functions,
// not by directly touching the raw message objects
export void SetVehicleSpeed(int kph)
{
  gVehicleSpeedKph = kph;
  // ... update the relevant signal in whichever message actually carries speed
}
```

Exposing `SetVehicleSpeed()` (and similar functions for every scenario
variable a test needs to control) as an `export`ed interface — rather
than having each test module poke at `engineMsg` fields directly — is
what keeps the restbus and the test logic decoupled: the restbus owns
timing fidelity, the test owns scenario intent.

## Cycle-time fidelity matters more than it looks

A restbus node that transmits "roughly every 10ms" using a naive timer
re-arm can drift under system load, and CANoe's own timer resolution
has practical limits. For most functional tests this drift is
harmless, but for anything testing timeout/failsafe logic specifically
(Level 1's stale-signal watchdog), sloppy restbus timing can produce
false positives — the ECU under test flags a timeout not because of a
real defect but because the restbus itself missed its window. Any
timeout-adjacent test result should be corroborated against a bus
trace of the restbus's actual transmission times before being trusted.

## Restbus vs. real other-ECU hardware: know which you're using

| | Restbus simulation | Real other ECUs |
|---|---|---|
| Cost/setup | Cheap, fast, scriptable | Requires physical units, harnessing |
| Fidelity | As good as the DBC + your CAPL logic | Ground truth |
| Fault injection | Trivial (Level 3 Module 6) — just send a bad value | Requires actually breaking the other ECU or its wiring |
| Startup/sleep behavior | Only as faithful as you script it | Authentic, including quirks not in any spec |

A mature test strategy (Level 4 Module 1) is explicit about which
tests run against a restbus (fast, run constantly, catch most logic
defects) versus which require real other-ECU hardware in the loop
(slower, run less often, catch integration-specific surprises a
simulation can't reproduce by definition).

## A worked restbus-dependent test case

| Field | Value |
|---|---|
| Test Case ID | TC-RESTBUS-004 |
| Title | ECU enters degraded mode within 500ms if EngineStatus stops arriving |
| Preconditions | Full restbus active, ECU under test in normal operation |
| Steps | 1. Confirm ECU in normal mode.<br>2. Stop the `txEngine` timer (simulate EngineStatus loss).<br>3. Wait 600ms.<br>4. Read ECU mode signal. |
| Expected result | ECU mode == Degraded within 500ms of the last EngineStatus frame, per the ECU's documented timeout spec. |
| Notes | This test is only meaningful *because* the restbus was providing EngineStatus reliably beforehand — verify via bus trace that EngineStatus was indeed arriving every 10ms before the timer was stopped, or a passing result proves nothing. |

## Cheat sheet

| Concept | Key point |
|---|---|
| Restbus purpose | Stand in for every node not physically under test |
| DBC cycle-time attribute | Basis for auto-generated baseline restbus |
| Meaningful vs. default values | Auto-generated restbus often needs enrichment for real test scenarios |
| `export`ed control functions | Decouple restbus timing logic from test scenario logic |
| Timing fidelity | Critical specifically for timeout/failsafe tests — verify via trace |
| Restbus vs. real hardware | Restbus for fast/frequent logic tests; real ECUs for integration fidelity |

## Exercise

You're testing an ABS ECU that requires `VehicleSpeed`, `WheelSpeedFL`,
`WheelSpeedFR`, `WheelSpeedRL`, `WheelSpeedRR`, and `BrakePedalPosition`
from four other nodes, all with different cycle times (5ms, 5ms, 5ms,
5ms, 10ms respectively).

1. Sketch the CAPL structure (timers and `export`ed functions only, not
   full implementation) you'd use to simulate these five signals with
   correct independent cycle times while letting a test script easily
   set up an "all four wheels agree, vehicle braking normally" scenario
   with one call.
2. Design a fault-scenario extension: one wheel-speed signal freezes
   (stops updating) while the other three continue normally. Explain
   what this restbus needs to support to make that scenario simple to
   trigger from a testcase.
3. Explain why a test asserting the ABS ECU detects the frozen-wheel
   fault within a specific time window is only trustworthy if you've
   independently verified restbus timing fidelity — referencing the
   worked test case above.
