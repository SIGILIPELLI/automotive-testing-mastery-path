# 10 · Project — Design a CAPL Test Script for a Simulated ECU Signal

This capstone pulls together every Level 1 module into one deliverable:
a CAPL script that simulates a sensor node, a threshold-driven warning
signal with hysteresis, a UDS-style diagnostic reset, and a documented
test case set proving it all behaves correctly — the same shape of work
a junior automotive test engineer does in their first real project.

!!! note "About this project"
    As with every hands-on module in this course, this CAPL script is a
    carefully reviewed worked example for technical accuracy against
    real, documented CAPL conventions — it has not been compiled or run
    against a live CANoe instance. Treat it as a template to adapt once
    you have real tool access, and verify it against your own project's
    CANoe/CAPL version and database before relying on it.

## The scenario

You're testing a **fuel level sensor node** that:

- Transmits `FuelLevelStatus` every 100 ms: a `FuelLevelPct` signal
  (0-100%, 1% per raw unit), a `LowFuelWarning` flag, and a 4-bit
  `AliveCounter`.
- Raises `LowFuelWarning` when `FuelLevelPct` drops **below 10%**, and
  clears it only once `FuelLevelPct` rises **above 15%** (hysteresis,
  per Module 8's exercise on exactly this pattern).
- Listens for a `DiagnosticResetRequest` message (standing in for a
  UDS-style diagnostic trigger from Module 6) and, on receiving it,
  resets its internal alive counter to 0.

## Step 1 — the CAPL simulation node

```c
variables
{
  message FuelLevelStatus       statusMsg;
  message DiagnosticResetRequest resetReq;   // received, not sent by this node

  msTimer cycleTimer;
  byte    aliveCounter;
  int     fuelLevelPct;     // current simulated fuel level, 0-100
  byte    warningActive;    // 0 or 1 - holds hysteresis state
}

on start
{
  aliveCounter  = 0;
  fuelLevelPct  = 50;     // start at a nominal, safe level
  warningActive = 0;

  setTimer(cycleTimer, 100);
}

on timer cycleTimer
{
  // Hysteresis logic: activate below 10, deactivate only above 15.
  if (fuelLevelPct < 10)
  {
    warningActive = 1;
  }
  else if (fuelLevelPct > 15)
  {
    warningActive = 0;
  }
  // Between 10 and 15 inclusive: warningActive keeps its previous value.

  statusMsg.FuelLevelPct  = fuelLevelPct;
  statusMsg.LowFuelWarning = warningActive;
  statusMsg.AliveCounter  = aliveCounter;
  output(statusMsg);

  aliveCounter = (aliveCounter + 1) % 16;
  setTimer(cycleTimer, 100);
}

on message DiagnosticResetRequest
{
  write("DiagnosticResetRequest received - resetting alive counter.");
  aliveCounter = 0;
}
```

A CAPL test harness would drive `fuelLevelPct` directly (as shown below)
to control the scenario deterministically; a real sensor node would
instead compute it from a simulated physical model or a real ADC
reading, per the S32K course's ADC module.

## Step 2 — the CAPL test script

A separate CAPL program, acting as the tester, drives `fuelLevelPct`
through the exact boundary sequence from Module 8's exercise and checks
`LowFuelWarning` at each step:

```c
variables
{
  msTimer stepTimer;
  int     testStep;
}

on start
{
  testStep = 0;
  setTimer(stepTimer, 500);   // allow one full cycle to settle before checking
}

on timer stepTimer
{
  switch (testStep)
  {
    case 0:
      write("Step 0: set fuel level to 9%% (expect warning ON)");
      fuelLevelPct = 9;
      break;

    case 1:
      if (warningActive == 1)
        write("PASS: TC-FUEL-01 - warning ON at 9%%");
      else
        write("FAIL: TC-FUEL-01 - warning OFF at 9%%, expected ON");

      write("Step 1: set fuel level to 12%% (expect warning still ON - hysteresis)");
      fuelLevelPct = 12;
      break;

    case 2:
      if (warningActive == 1)
        write("PASS: TC-FUEL-02 - warning still ON at 12%% (hysteresis holds)");
      else
        write("FAIL: TC-FUEL-02 - warning OFF at 12%%, expected ON (hysteresis broken)");

      write("Step 2: set fuel level to 16%% (expect warning OFF)");
      fuelLevelPct = 16;
      break;

    case 3:
      if (warningActive == 0)
        write("PASS: TC-FUEL-03 - warning OFF at 16%%");
      else
        write("FAIL: TC-FUEL-03 - warning still ON at 16%%, expected OFF");

      write("Test sequence complete.");
      return;   // stop re-arming the timer - test finished
  }

  testStep = testStep + 1;
  setTimer(stepTimer, 500);
}
```

This mirrors exactly the "trickiest boundary case" from Module 8's
exercise — 9% → 12% → 16% — with the middle step being the one that
actually proves the hysteresis logic works, not just the two clear-cut
thresholds.

## Step 3 — the documented test cases (traceability)

| Test Case ID | Title | Steps | Expected result | Requirement |
|---|---|---|---|---|
| TC-FUEL-01 | Warning activates below 10% | Set `FuelLevelPct = 9` | `LowFuelWarning == 1` within 100 ms | REQ-FUEL-002 |
| TC-FUEL-02 | Warning stays active in the hysteresis band | Set `FuelLevelPct = 12` after TC-FUEL-01 | `LowFuelWarning == 1` (unchanged) | REQ-FUEL-003 |
| TC-FUEL-03 | Warning deactivates above 15% | Set `FuelLevelPct = 16` after TC-FUEL-02 | `LowFuelWarning == 0` within 100 ms | REQ-FUEL-003 |
| TC-FUEL-04 | Alive counter resets on diagnostic request | Send `DiagnosticResetRequest`, then observe next `FuelLevelStatus` | `AliveCounter == 0` in the next transmitted frame | REQ-FUEL-005 |
| TC-FUEL-05 | Alive counter increments and wraps | Observe 20 consecutive frames | Counter increments by 1 each frame, wraps `15 -> 0` | REQ-FUEL-006 |

A minimal traceability matrix:

| Req ID | Requirement summary | Test cases | Status |
|---|---|---|---|
| REQ-FUEL-002 | Warning activates below 10% | TC-FUEL-01 | Covered |
| REQ-FUEL-003 | Warning deactivates only above 15% (hysteresis) | TC-FUEL-02, TC-FUEL-03 | Covered |
| REQ-FUEL-005 | Alive counter resets on diagnostic reset request | TC-FUEL-04 | Covered |
| REQ-FUEL-006 | Alive counter increments and wraps correctly | TC-FUEL-05 | Covered |
| REQ-FUEL-004 | Warning state must be reported correctly to the instrument cluster | — | **GAP — not testable at this node's level alone; needs integration test with the cluster ECU** |

That last row is deliberate: a good traceability matrix surfaces real
gaps, including ones that are out of scope for a single node's test
script and need to move up the V-model to integration test.

## What this project demonstrates, mapped to Level 1

| Module | Concept used here |
|---|---|
| 1 — V-model | This script sits at unit/component test; TC-FUEL-04's gap points at integration test |
| 2-3 — CAN fundamentals & frames | The signals being tested are carried in real CAN frame fields |
| 4 — CANoe | This script and its test harness would run inside a CANoe configuration |
| 5 — CAPL | Event handlers, `on timer`, `on message`, cyclic sender, alive counter pattern |
| 6 — UDS diagnostics | `DiagnosticResetRequest` stands in for a UDS-style diagnostic trigger |
| 8 — Test case design | Boundary value analysis with hysteresis, fully documented test cases, traceability |
| 9 — Standards | This is exactly the kind of evidence an ASPICE SWE.4 assessment or an ISO 26262 safety case would ask to see |

## Exercise

Extend this project with one more requirement: **REQ-FUEL-007** — if the
sensor node's `FuelLevelStatus` frame stops arriving for more than
300 ms (simulating a bus fault or node failure), a *separate* consumer
node (e.g., the instrument cluster) shall set an internal
`fuelSignalLost` flag and display a generic "check fuel system" fault
rather than trusting a frozen `LowFuelWarning` value.

1. Write the CAPL `variables` block and `on message` / `on timer`
   handlers for the consumer node's signal-timeout watchdog, following
   the exact pattern from Module 5.
2. Write two fully documented test cases: one proving the flag sets
   correctly after 300 ms of silence, and one proving it clears
   correctly the moment `FuelLevelStatus` resumes.
3. Add both new test cases to the traceability matrix above, and
   identify whether REQ-FUEL-007 is better tested as a unit test on the
   consumer node alone, or requires the sensor node included too —
   justify your answer using the V-model framing from Module 1.
