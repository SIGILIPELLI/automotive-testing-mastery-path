# 07 · Regression Test Suites for Automotive Software

Every module so far has built individual testcases. This module
covers what happens once you have hundreds of them: how to structure a
regression suite that runs reliably and quickly enough to gate every
software change, how to triage failures at scale, and how to keep a
growing suite from becoming untrustworthy.

!!! note "About this module"
    No specific CI/regression infrastructure was used to produce this
    content. The suite-structuring and flake-triage practices below
    reflect common industry regression-testing practice adapted to the
    CAPL/CANoe tooling from earlier modules.

## What makes a good regression suite (versus a pile of testcases)

| Property | Why it matters |
|---|---|
| Deterministic | Same input, same result, every run — a suite that's sometimes right is worse than no suite, because it erodes trust |
| Fast enough to run on every change | A suite that takes 6 hours doesn't gate a merge request; it gates a nightly build at best |
| Independent testcases | One testcase's failure or leftover state must not affect the next (Module 6's "always restore" discipline, applied suite-wide) |
| Clear failure attribution | A failing testcase name/log should point at *what broke*, not just "suite failed" |
| Maintained, not just accumulated | Testcases for removed features get removed; testcases for changed behavior get updated, not left silently wrong |

## Tiering: not every testcase belongs in every run

```text
Smoke tier   (2–5 min):  Does the ECU boot, respond to diagnostics, transmit its core signals?
                         Run on every build, every commit.

Functional tier (30–60 min): Full feature-level testcases — DTC behavior, restbus
                         interactions, CAPL test modules from Levels 1–2.
                         Run on every merge request / pull request.

Full regression (hours): Fault injection, HIL scenario matrices, endurance/soak.
                         Run nightly or on a fixed cadence, not per-commit.
```

Tiering solves the "fast enough to gate every change" tension directly
— the smoke tier catches the most common and cheapest-to-detect
breakage immediately, while expensive fault-injection matrices still
run regularly without blocking every small change.

## Structuring the CAPL side for tiering

```c
// Illustrative — tagging testcases so an orchestration layer (Module 5)
// can select a tier without touching CAPL test logic per run.
testcase tc_Smoke_EcuRespondsToDiagnosticSession()
{
  testCaseTitle("[SMOKE] ECU accepts UDS diagnostic session request");
  DiagRequest req = new DiagSessionControl(EXTENDED);
  DiagSendRequest(req);
  testWaitForDiagResponse(req, 1000);
  testStepCheck("positive response received", DiagGetLastResponseType(req) == POSITIVE_RESPONSE);
}

testcase tc_Functional_DtcSetsAfterPersistentFault()
{
  testCaseTitle("[FUNCTIONAL] DTC sets after fault persists past debounce time");
  // ... full DTC debounce logic from Level 2 Module 5
}
```

The `[SMOKE]`/`[FUNCTIONAL]` prefix in the title is a simple, low-tech
tiering mechanism the Python orchestration layer can filter on via the
COM API's report data — teams with more mature tooling use dedicated
metadata fields instead, but the principle (tier is explicit and
machine-readable, not tribal knowledge) is the same either way.

## Flaky tests: the regression suite's silent killer

A **flaky test** passes and fails intermittently with no code change
— usually a timing race (a fixed wait that's sometimes too short), a
leftover state from a prior testcase, or a genuine intermittent bug in
the system under test. Flaky tests are worse than simply missing tests
because engineers learn to ignore red results ("oh that one's just
flaky"), which quietly disables the suite's ability to catch real
regressions.

| Flaky-test cause | Fix |
|---|---|
| Fixed wait too short under load | Wait on a signal condition (`testWaitForSignal`) instead of a fixed `testWaitForTimeout` wherever possible |
| Testcase order dependency | Ensure every testcase resets shared state on entry, not just on exit |
| Shared rig resource contention | Serialize testcases that touch the same physical resource (e.g., a single power supply) rather than assuming parallel safety |
| Genuine intermittent bug | Do not suppress or delete the test — file the bug; a flaky test is often reporting a real, rare defect |

The last row matters: the instinct to delete or `@skip` a flaky test
to unblock CI is sometimes right (a badly-written test) and sometimes
exactly wrong (a genuine race condition in the ECU under test that the
test is the only thing catching) — triage each flaky failure rather
than reflexively silencing it.

## Failure triage workflow

```text
1. Suite reports N failures.
2. Group failures by testcase → are multiple testcases failing on the
   SAME root cause (e.g., a shared setup step broke)? Fix once.
3. For each distinct failure: reproduce in isolation. Does it fail
   alone, outside the full suite? If not, suspect order-dependency.
4. Bisect against recent changes if the failure is new since last
   green run — which commit introduced it?
5. File or update a defect with the specific testcase name, log
   excerpt, and suspected commit — not just "suite is red."
```

Step 2 is the highest-leverage step at scale: a single broken shared
fixture (say, a restbus simulation node that stopped initializing
correctly) can cause dozens of unrelated-looking testcase failures,
and triaging each individually wastes far more time than recognizing
the shared root cause first.

## Suite health metrics worth tracking

| Metric | What it reveals |
|---|---|
| Pass rate over time | Sudden drops correlate with specific changes; slow decline suggests suite rot |
| Flake rate (pass after retry with no code change) | Rising flake rate erodes trust before anyone notices explicitly |
| Suite runtime trend | Growing runtime eventually breaks the "fast enough to gate" property — needs re-tiering |
| Testcases added vs. features shipped | A ratio dropping toward zero suggests test debt accumulating |

## Cheat sheet

| Concept | Key point |
|---|---|
| Tiering | Smoke/functional/full-regression, matched to how often each can run |
| Independence | Every testcase must reset its own preconditions, not rely on run order |
| Flaky tests | Triage, don't reflexively silence — may be a real intermittent bug |
| Failure triage | Group by root cause before fixing individually |
| Suite health metrics | Pass rate, flake rate, runtime trend, test-to-feature ratio |

## Exercise

1. A 45-minute functional-tier suite has grown to 3 hours over 6
   months with steady testcase additions. Propose a concrete re-
   tiering plan, including how you'd decide which existing testcases
   move to the nightly full-regression tier.
2. A testcase fails only when run after `tc_ClosedLoopSanityCheck`
   from Level 3 Module 1, never when run alone. Diagnose the likely
   cause using the order-dependency row above, and write the CAPL
   fix.
3. Design the failure-triage grouping step (step 2 above) as a small
   script's logic: given a list of failing testcase names and their
   log excerpts, what heuristic would you use to flag "these 12
   failures likely share one root cause" versus "these are 12
   independent failures"?
