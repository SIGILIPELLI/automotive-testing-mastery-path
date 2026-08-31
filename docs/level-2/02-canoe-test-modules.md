# 02 · CANoe Test Modules & Test Cases

Everything in Level 1 was a **simulation** — CAPL nodes that send,
receive, and react, but never formally declare pass/fail. A **test
module** is CANoe's structured alternative: an explicit sequence of test
cases, each producing a verdict, rolled up into a report. This is how
CAN testing actually scales past "watch the Write window and eyeball
it."

!!! note "About this module"
    The test-module structure, verdict functions, and report concepts
    below are genuine, documented CANoe/CAPL features. No test module
    here was executed against real CANoe — every example is a reviewed
    reference for the syntax and structure.

## Why test modules exist

A plain CAPL simulation node can call `write()` when something looks
wrong, but "wrong" is just text in a log — there is no structured
verdict, no report, and no way for a CI pipeline (Level 3-4) to ask
"did it pass?" without parsing free-text output. A test module instead
produces a structured **test report** with a verdict (Pass/Fail/Inconclusive)
per test case and per step, which is what makes automotive test suites
auditable and machine-consumable.

## Anatomy of a test module

```c
variables
{
}

testcase TC_EngineStartupSequence()
{
  testCaseTitle("Engine startup: coolant sensor reports valid value within 500 ms");

  testStep("Wait for SensorNodeStatus after ignition-on");
  testWaitForTimeout(500);   // block up to 500 ms waiting for the next assertion

  if (SensorNodeStatus::SensorValid == 1)
  {
    testStepPass("SensorValid=1 observed before timeout");
  }
  else
  {
    testStepFail("SensorValid never went to 1 within 500 ms");
  }
}

testcase TC_CoolantTempInRange()
{
  testCaseTitle("Coolant temperature is within calibration range");
  testStep("Check current CoolantTemp signal value");

  if (SensorNodeStatus::CoolantTemp >= -400 && SensorNodeStatus::CoolantTemp <= 1500)
  {
    testStepPass("CoolantTemp within [-40.0, 150.0] C");
  }
  else
  {
    testStepFail(str_printf("CoolantTemp out of range: %d", SensorNodeStatus::CoolantTemp));
  }
}
```

Each `testcase` function is one independently reportable unit. Inside it,
`testStep()` documents what's being checked (shows up in the report as a
labeled row), and `testStepPass()`/`testStepFail()` record the verdict
for that step. A test case's overall verdict is typically the worst
verdict of its steps — one failed step fails the whole case.

## Verdicts and the verdict hierarchy

| Verdict | Meaning |
|---|---|
| **Pass** | Every step in the test case passed |
| **Fail** | At least one step explicitly failed |
| **Inconclusive** | The test couldn't reach a definite answer (e.g., a precondition timed out before the actual check could run) |
| **None/Not run** | The test case was skipped entirely (e.g., a prior setup step failed) |

Inconclusive is not the same as Fail — it flags "this test didn't prove
anything either way," which matters a great deal in a report someone
will triage: a genuine functional failure and "the rig wasn't ready"
should never look identical.

## Preconditions and test case ordering

Real test modules almost always need setup/teardown around each case:

```c
testcase TC_ObstructionReversal()
{
  testCaseTitle("Power window reverses on obstruction (Level 1 Module 1 requirement)");

  testStep("Precondition: window fully closed");
  // ... commands to close the window, wait, verify closed state ...

  testStep("Simulate obstruction: motor current step to 9 A for 60 ms");
  // ... drive the plant/simulation to the fault condition ...

  testStep("Verify reversal begins within 200 ms");
  testWaitForTimeout(200);
  if (WindowStatus::MotorDirection == eDIRECTION_DOWN)
  {
    testStepPass("Reversal direction observed");
  }
  else
  {
    testStepFail("No reversal observed within 200 ms window");
  }
}
```

Test modules run their test cases in a defined order (by default, the
order they're listed), and CANoe's test module editor lets you group
related cases, mark some as dependent on others, and reuse setup logic
across cases — the practical unit for the "automated test sequence"
Module 7 builds on.

## Reports

Running a test module produces a report (HTML/XML by default) listing
every test case, every step, its verdict, and a timestamp — the
artifact that gets archived, attached to a build, or fed into a
regression dashboard (Level 3 Module 7). This structured, exportable
report is the single biggest practical difference between "a simulation
node that logs text" and "a test module."

## Cheat sheet

| Element | Purpose |
|---|---|
| `testcase name() { }` | One independently reportable test unit |
| `testCaseTitle("...")` | Human-readable title shown in the report |
| `testStep("...")` | Documents/labels one check within a test case |
| `testStepPass("...")`, `testStepFail("...")` | Record a step's verdict |
| `testWaitForTimeout(ms)` | Blocks until a timeout, used to bound how long a check waits |
| Pass / Fail / Inconclusive / Not run | The four verdict states, not interchangeable |
| Report (HTML/XML) | Structured, machine-consumable output of a test module run |

## Exercise

Write two `testcase` functions for a simulated seatbelt-warning ECU:

1. `TC_SeatbeltWarningOnStart` — verifies that if `SeatbeltStatus ==
   UNBUCKLED` and vehicle speed exceeds 15 km/h, a `SeatbeltWarningLamp`
   signal goes active within 3 seconds; use `testWaitForTimeout` and give
   it a clear `testCaseTitle`.
2. `TC_SeatbeltWarningClearsOnBuckle` — verifies that once buckled, the
   warning lamp signal goes inactive within 1 second.

For each, write at least one step that could plausibly return
**Inconclusive** rather than Fail, and explain in a sentence why that
distinction matters to whoever triages the report.
