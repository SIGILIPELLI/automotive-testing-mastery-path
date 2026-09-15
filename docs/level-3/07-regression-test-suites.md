---
description: "Regression Test Suites for Automotive Software — Every module so far has built individual testcases. This module covers what happens once you have…"
---

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

## How It Actually Works: why bisection finds the culprit in log₂(N) builds, and why order-dependent flakes cluster around shared CAPL state

Two of this module's practices deserve their actual mechanism rather
than treatment as folklore.

**Bisection's efficiency is a real algorithmic guarantee, not just
"try some old builds."** If a regression suite was green at commit A
and red at commit B, with N commits in between, checking the exact
midpoint commit and recursing into whichever half still spans
green→red halves the search space every iteration — the same binary
search structure underlying `git bisect` — so the number of builds you
actually need to test is `ceil(log2(N))`, not N. For a 200-commit range,
that's 8 builds, not 200 — the concrete reason "bisect against recent
changes" (this module's triage step 4) is tractable even against a
large commit window, as long as each build/run cycle is itself fast
enough (which is exactly why the tiering discipline earlier in this
module matters for triage speed, not just for gating speed).

**Order-dependent flakes have a specific mechanical cause traceable
back to Level 1 Module 5's event-queue model**: two testcases running
in the same CANoe measurement share the *same* CAPL global `variables`
block, the same sysvars, and the same restbus simulation state, all
living in one process for the whole test module's execution — nothing
is torn down and recreated between testcases by default. When a
testcase like `tc_ClosedLoopSanityCheck` (Level 3 Module 1) sets a
signal or sysvar and doesn't explicitly reset it in
`testcasefinalization`, that value persists in memory exactly as any
CAPL global would, and the next testcase's `testpreparation` step
either does or doesn't re-establish its own expected baseline for that
same variable. This is precisely why the fix is never "run testcases in
a fixed safe order" (which just hides the dependency until someone
reorders the suite) but always "make every testcase's setup
self-sufficient" — the state genuinely is shared process memory, and
the only way to make execution order irrelevant is for each testcase to
overwrite every piece of that shared state it depends on, every time,
rather than trusting whatever a previous testcase happened to leave.

This mechanism also explains why grouping failures by root cause (triage
step 2) is more than a time-saving heuristic: dozens of testcases
failing after one shared setup step breaks look, superficially, like
independent bugs, but they all trace back to the *same* corrupted shared
state — a log-message fingerprint clustering heuristic (grouping
failures whose log excerpts mention the same signal, sysvar, or restbus
node name) reliably surfaces this because the shared root cause leaves
the same textual fingerprint across every testcase it poisons.

## 🔀 Related lessons on other tracks

- [Cpp Testing — 09 · Test Builds & Running Suites](https://sigilipelli.github.io/cpp-testing-mastery-path/level-1/09-test-builds-running-suites/)

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
