# 10 · Project — HIL Test Plan for an ECU Feature

This project pulls together every Level 3 module into one deliverable:
a real, structured HIL test plan for a single ECU feature, written the
way a test lead would actually hand it to a team before rig time is
booked. You'll produce the plan, not just talk about what one should
contain.

!!! note "About this module"
    No physical HIL rig was used to produce or validate this project.
    The plan template, CAPL sketches, and fault matrix below follow
    documented HIL/ISO 26262/SOTIF practice from Modules 1–9 — treat
    the whole deliverable as a template to adapt, not a captured real
    test campaign.

## The feature: Automatic Emergency Braking (AEB), scoped down

To keep the project concrete, scope it to one feature and one clear
boundary: an AEB ECU that receives a simulated forward-object distance
and closing-speed signal (from a radar-simulation HIL plant model) and
commands autonomous braking when a collision is imminent and the
driver hasn't reacted.

## Section 1 — Scope and test levels

```text
In scope:
  - AEB activation logic under nominal, boundary, and fault conditions
  - Closed-loop HIL validation of braking command reaching simulated
    vehicle deceleration
  - Functional-safety mechanism testing for the two identified safety
    goals (see Section 3)
  - Known-unsafe (SOTIF Area 2) scenario regression

Out of scope (explicitly, and why):
  - Radar perception algorithm accuracy itself (owned by perception
    validation team, not signal-level HIL — see Level 3 Module 9)
  - Electrical-level fault injection on radar wiring (requires a
    breakout box not assumed available for this project — flagged as
    a gap, see Section 6)
```

Stating out-of-scope items explicitly, with reasons, is itself part of
professional test-plan writing — it prevents a stakeholder from
assuming a gap was an oversight rather than a deliberate, documented
boundary.

## Section 2 — Requirements traced

| Requirement ID | Summary | ASIL |
|---|---|---|
| SW-REQ-201 | AEB shall command full braking within 150ms of a validated imminent-collision condition | ASIL D |
| SW-REQ-202 | AEB shall not activate on implausible or out-of-range distance/speed signals | ASIL D |
| SW-REQ-203 | AEB shall disengage cleanly if the driver applies throttle above a defined threshold (override) | ASIL C |

Every testcase in Section 4 carries one of these IDs in its title,
per the Level 3 Module 8 tagging convention.

## Section 3 — Safety goals and mechanisms under test

| Safety goal | Safety mechanism | Fault(s) to inject |
|---|---|---|
| Avoid failing to brake when collision is imminent | Plausibility check on distance/speed signal pair; redundant timing computation | Signal timeout (Module 6), implausible value jump |
| Avoid unintended/false braking | Confidence threshold on radar-simulated object; driver-override path | Simulated low-confidence noisy object, throttle-override signal |

## Section 4 — Representative CAPL testcases

```c
testcase tc_AebActivatesWithinSpec()
{
  testCaseTitle("[SW-REQ-201] AEB commands full braking within 150ms of imminent collision");
  dword t0, t1;
  HilSetParameter("ClosingSpeed_kph", 60.0);
  HilSetParameter("ForwardDistance_m", 15.0); // set up an imminent-collision geometry
  t0 = timeNowNS() / 1000000;

  testWaitForSignal(AEB_BrakeCommand, 1, 150);
  t1 = timeNowNS() / 1000000;
  testStepCheck("brake command issued within 150ms", (t1 - t0) <= 150);
}

testcase tc_AebIgnoresImplausibleDistance()
{
  testCaseTitle("[SW-REQ-202] AEB does not activate on an implausible negative distance reading");
  HilSetParameter("ForwardDistance_m", -5.0); // physically impossible
  testWaitForTimeout(300);
  testStepCheck("no brake command on implausible input", getSignal(AEB_BrakeCommand) == 0);
}

testcase tc_AebDisengagesOnDriverOverride()
{
  testCaseTitle("[SW-REQ-203] AEB disengages cleanly on driver throttle override above threshold");
  HilSetParameter("ClosingSpeed_kph", 60.0);
  HilSetParameter("ForwardDistance_m", 15.0);
  testWaitForSignal(AEB_BrakeCommand, 1, 150);

  setSignal(ThrottlePosition_pct, 40.0); // above the defined override threshold
  testWaitForSignal(AEB_BrakeCommand, 0, 500);
  testStepCheck("AEB cleanly released control, no residual brake command", getSignal(AEB_BrakeCommand) == 0);
}
```

## Section 5 — Fault injection matrix (Module 6 practice applied)

| Signal | Implausible value | Frame timeout | Electrical (needs breakout box) |
|---|---|---|---|
| ForwardDistance_m | Covered — `tc_AebIgnoresImplausibleDistance` | Planned, not yet written | Gap — see Section 6 |
| ClosingSpeed_kph | Planned, not yet written | Planned, not yet written | Gap — see Section 6 |

## Section 6 — Known gaps and risk acceptance

```text
Gap: No breakout-box hardware assumed available for this project, so
electrical-level fault injection (open/short on radar signal lines)
for both ForwardDistance_m and ClosingSpeed_kph is NOT covered by this
plan's test suite.

Risk: For ASIL D requirement SW-REQ-201/202, this is a real coverage
gap against the safety case, not a cosmetic one — electrical faults on
these lines are a plausible real-world failure mode. This must be
explicitly flagged to the safety assessor as an open item, with a
target date for hardware acquisition, rather than silently left out of
the plan.
```

Naming the gap explicitly — rather than a plan that looks complete
because it simply never mentions what it doesn't cover — is the
single most professionally important habit this project is meant to
practice.

## Section 7 — Exit criteria

```text
This test plan is considered executed and closeable when:
  1. All testcases in Section 4 (and remaining planned ones from
     Section 5) have a recorded verdict.
  2. Every ASIL D requirement's test cases show a Passed verdict, or
     a filed, tracked defect for each Failed one.
  3. The Section 6 gap has a documented risk acceptance signed off by
     the safety assessor, or the breakout-box testing has been
     completed and the gap closed.
```

## Cheat sheet

| Section | What it forces you to make explicit |
|---|---|
| Scope / out-of-scope | What this plan does NOT claim to cover, and why |
| Requirements traced | Every testcase ties to a requirement ID and ASIL |
| Safety goals/mechanisms | What specifically each fault-injection testcase is proving |
| Fault matrix | Coverage gaps are visible as empty cells, not silently absent |
| Known gaps | Honest, dated, risk-accepted — never hidden |
| Exit criteria | An unambiguous definition of "done" for the plan itself |

## Exercise

1. Write the two "Planned, not yet written" testcases from Section 5
   (frame timeout on `ClosingSpeed_kph`, implausible value on the
   same signal), following the pattern of Section 4's testcases.
2. `tc_AebDisengagesOnDriverOverride` doesn't check *how quickly* the
   brake command clears once override begins. Add a timing assertion,
   propose a reasonable spec value, and add it to Section 2's
   requirements table with a new requirement ID.
3. Section 6 names one gap. Identify a second plausible gap in this
   plan (consider SOTIF Area 3, Level 3 Module 9) that isn't already
   listed, and write its Section 6 entry in the same style.
