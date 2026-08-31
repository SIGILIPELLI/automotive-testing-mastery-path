# 07 · Automated Test Sequences in CANoe

Module 2 introduced CANoe's Test Module structure (`testcase`,
`testfunction`) for individual checks. Real suites chain dozens of
these into ordered **test sequences** with shared setup, teardown, and
pass/fail reporting that survives a single case failing without
aborting the whole run. This module covers that orchestration layer.

!!! note "About this module"
    The CAPL test-sequence constructs, XML test report structure, and
    execution-control APIs below are genuine, documented CANoe
    behavior. Nothing here was captured from a live CANoe session —
    treat every snippet as a reviewed reference and verify exact
    report formatting against your installed CANoe version.

## Test module structure, end to end

```c
variables
{
  int gSetupOk;
}

testpreparation
{
  gSetupOk = 0;
  // Runs once, before any testcase in this module
  if (InitBusSimulation() == 1)
  {
    gSetupOk = 1;
  }
}

testcase tc_CoolantOverheatBoundary()
{
  testCaseTitle("Coolant overheat activates above 120C, not at/below");
  if (gSetupOk == 0)
  {
    testStepFail("Preparation failed, skipping");
    return;
  }
  testStep("Set CoolantTemp=120.0");
  SetSignalValue("CoolantTemp", 120.0);
  testWaitForTimeout(150);
  testStepCheck("OverheatWarning stays 0", GetSignalValue("OverheatWarning") == 0);

  testStep("Set CoolantTemp=120.1");
  SetSignalValue("CoolantTemp", 120.1);
  testWaitForTimeout(150);
  testStepCheck("OverheatWarning goes to 1", GetSignalValue("OverheatWarning") == 1);
}

testcase tc_LowFuelHysteresis()
{
  testCaseTitle("Low fuel warning hysteresis 10%/15%");
  // ... independent test body, order-independent of tc_CoolantOverheatBoundary
}

testcasefinalization
{
  // Runs once after all testcases, regardless of individual pass/fail
  ResetBusSimulation();
}
```

The structural rule that matters most: **`testpreparation` and
`testcasefinalization` always run exactly once per module execution**,
while every `testcase` function runs independently and a failure in
one must not prevent the others from executing — CANoe's test module
runner already guarantees this as long as you don't hand-roll your own
early-exit logic that skips sibling testcases.

## Ordering and independence

A frequent design mistake is writing testcases that depend on each
other's side effects (e.g. `tc_02` assumes `tc_01` left a DTC set).
This breaks the moment someone reorders the module, runs a single
testcase in isolation for debugging, or a test-selection tool runs a
subset in CI (Level 4's continuous-testing module). Each `testcase`
should:

- Establish its own preconditions explicitly (even if that duplicates
  a `SetSignalValue` call another testcase also makes).
- Clean up any state it changed that isn't already handled by
  `testcasefinalization`, if that state could leak into a differently
  ordered run.

## Reporting and pass/fail semantics

CANoe's test report distinguishes several outcome levels per step, not
just pass/fail:

| Step API | Meaning | Effect on testcase result |
|---|---|---|
| `testStepPass(msg)` | Explicit pass | Testcase passes if all steps pass |
| `testStepFail(msg)` | Explicit fail | Testcase fails, execution of *that* testcase can continue or `return` |
| `testStepCheck(msg, cond)` | Pass/fail based on `cond` | Same as above, condition-driven |
| `testCaseFatalError(msg)` | Unrecoverable for this testcase | Aborts the testcase immediately, module continues |
| `testCaseInconclusive(msg)` | Neither pass nor fail | Common for skipped/preconditions-not-met tests, kept out of pass-rate stats |

Using `testCaseInconclusive` correctly matters for CI dashboards
(Level 4): a testcase that couldn't run because bench hardware wasn't
present should never silently count as either a pass (false confidence)
or a fail (false alarm) — inconclusive is the honest third option.

## Building a sequence across multiple test modules

CANoe's **Test Configuration** (`.testcfg` / Test Environment)
orders multiple test modules and controls whether a module's failure
halts the whole configuration or just gets recorded and moves on. The
two common patterns:

| Pattern | Configuration | Use case |
|---|---|---|
| Fail-fast | Configuration halts on first module failure | Smoke tests, gating a build before spending time on the full suite |
| Fail-forward | All modules run regardless of earlier failures | Full regression run (Level 3 Module 7) where you want a complete picture in one pass |

A well-organized suite usually runs a small fail-fast smoke sequence
first, then the full fail-forward regression sequence only if the smoke
sequence passes — avoiding a 45-minute full run against a build that's
obviously broken in the first two minutes.

## XML/report output

CANoe test modules can export results as XML (and HTML) reports
suitable for CI ingestion:

```xml
<TestModule name="CoolantAndFuelTests">
  <TestCase name="tc_CoolantOverheatBoundary" verdict="passed">
    <TestStep title="Set CoolantTemp=120.0" verdict="passed"/>
    <TestStep title="Set CoolantTemp=120.1" verdict="passed"/>
  </TestCase>
  <TestCase name="tc_LowFuelHysteresis" verdict="failed">
    <TestStep title="Verify warning off at 16%" verdict="failed"
              detail="Expected 0, got 1"/>
  </TestCase>
</TestModule>
```

This structure is exactly what a CI pipeline (Level 4 Module 2) parses
to compute pass rates and flag regressions — designing testcase titles
and step messages to be self-explanatory in this report is what makes
a failed nightly run debuggable without re-running it locally first.

## Cheat sheet

| Construct | Runs | Purpose |
|---|---|---|
| `testpreparation` | Once, before all testcases | Shared setup |
| `testcase` | Once each, order-independent | The actual check |
| `testcasefinalization` | Once, after all testcases | Shared teardown |
| `testStepCheck` | Per assertion | Condition-driven pass/fail |
| `testCaseInconclusive` | On unmet preconditions | Keeps skipped tests out of pass/fail stats |
| Fail-fast vs. fail-forward | Test Configuration level | Smoke vs. full regression strategy |

## Exercise

You're assembling a nightly sequence with three existing test modules:
`SmokeTests` (5 basic sanity testcases, ~1 minute), `CANSignalSuite`
(40 testcases, ~10 minutes), and `DTCLifecycleSuite` (15 testcases,
~15 minutes, requires a bench with real hardware that's occasionally
unavailable).

1. Propose an ordering and a fail-fast/fail-forward policy for these
   three modules, explaining your reasoning for each choice.
2. `DTCLifecycleSuite` sometimes can't run because the bench is
   occupied by another team. Which CAPL construct from this module
   should its testcases use in that situation, and why is it wrong to
   have them report a hard pass or fail instead?
3. Sketch (as pseudocode, not full CAPL) one `testcase` from
   `CANSignalSuite` that demonstrates proper independence — i.e., it
   sets up its own preconditions rather than assuming a prior testcase
   already did.
