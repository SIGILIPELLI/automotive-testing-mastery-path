# 01 · Advanced HIL Test Automation

Level 2 Module 9 mapped the physical components of a HIL rig. This
module covers how you actually automate test execution against one —
scenario scripting, real-time synchronization concerns, and the
practical differences from the pure-CAPL automation you've built so
far once a real-time simulation computer is in the loop.

!!! note "About this module"
    The automation patterns and API shapes below (CAPL-driven
    orchestration talking to a real-time simulation host) reflect
    genuine, documented practice across common HIL platforms. No
    specific rig was used to produce this content — treat every
    snippet as an architectural reference to adapt to your actual
    platform's real API, not a captured run.

## The orchestration split

A HIL test automation stack usually has two cooperating layers:

| Layer | Responsibility | Typical tooling |
|---|---|---|
| **Real-time layer** | Runs the plant model deterministically, every fixed timestep, no exceptions | dSPACE/NI real-time OS, model built from Simulink or similar |
| **Test orchestration layer** | Decides *what* scenario to run, sets model parameters, reads results, makes pass/fail calls | CAPL/CANoe, Python, or vendor test-automation frameworks, running non-real-time |

The orchestration layer never touches the real-time loop's internals
directly — it reads and writes named **parameters** and **signals**
exposed by the real-time model through a defined interface, exactly
the same "don't poke internals directly" discipline as Level 2 Module
8's `export`ed restbus control functions, just one layer further out.

```c
// CAPL orchestration talking to a HIL real-time model's parameter interface
// (illustrative — the real API name/shape depends on your platform's CANoe integration)
on start
{
  HilSetParameter("VehicleSpeed_Setpoint", 0.0);
  HilSetParameter("RoadFriction_Coefficient", 0.9);   // dry asphalt
}

testcase tc_AbsActivatesOnLowFriction()
{
  testCaseTitle("ABS ECU activates within spec when friction drops sharply");
  HilSetParameter("VehicleSpeed_Setpoint", 80.0);
  testWaitForTimeout(2000);   // allow the plant model to reach steady state
  HilSetParameter("RoadFriction_Coefficient", 0.15);  // simulated ice
  HilApplyBrakePedal(0.8);

  testWaitForTimeout(300);
  testStepCheck("ABS active flag set", HilGetSignal("ABS_Active") == 1);
}
```

## Scenario scripting: parameterize, don't hardcode

Every HIL testcase worth keeping should separate **scenario
parameters** (speed, friction, load, ambient temperature) from the
**test logic** that drives and checks them, so the same testcase body
can run a matrix of conditions:

```c
void RunAbsActivationScenario(float speedKph, float friction, float pedalForce, float expectedMaxMs)
{
  dword t0, t1;
  HilSetParameter("VehicleSpeed_Setpoint", speedKph);
  testWaitForTimeout(2000);
  HilSetParameter("RoadFriction_Coefficient", friction);
  HilApplyBrakePedal(pedalForce);
  t0 = timeNowNS() / 1000000;

  testWaitForSignal("ABS_Active", 1, 1000);
  t1 = timeNowNS() / 1000000;
  testStepCheck("ABS activation within spec time", (t1 - t0) <= expectedMaxMs);
}

testcase tc_AbsMatrix()
{
  testCaseTitle("ABS activation across speed/friction matrix");
  RunAbsActivationScenario(60.0, 0.15, 0.8, 250);
  RunAbsActivationScenario(120.0, 0.15, 0.8, 250);
  RunAbsActivationScenario(60.0, 0.05, 0.8, 300);   // near-ice, slightly relaxed spec
}
```

This is the same "cheat sheet" discipline as Level 1's exercise
technique (equivalence partitioning applied to scenario space, not just
signal values) — a matrix of representative operating points, not an
exhaustive and impractical sweep.

## Real-time synchronization pitfalls

| Pitfall | Cause | Mitigation |
<br>|---|---|---|
| Reading a signal mid-update | Orchestration layer polls at a moment between the real-time step writing a new value and it becoming "settled" | Read on a model-step-boundary event/callback rather than an arbitrary wall-clock timer, if your platform exposes one |
| Setting a parameter that only takes effect next step | Parameter writes are often latched, not applied instantaneously | Always wait at least one model timestep after a parameter write before asserting dependent behavior |
| Wall-clock test timeouts drifting from simulation time | Orchestration host isn't itself real-time; its clock can lag under load | Prefer waiting on model-reported simulation time or explicit signal conditions over fixed wall-clock sleeps where the platform allows it |
| Overlapping scenarios | A new scenario's setpoints applied before the previous scenario's transients settle | Always include an explicit settle/return-to-baseline step between scenarios, as in `RunAbsActivationScenario`'s initial 2000ms speed-settle wait |

## Closed-loop validation: proving the loop is actually closed

Level 2 Module 9 stressed that HIL must be closed-loop. Advanced
automation should include an explicit **loop-closure sanity check** as
a standing testcase, not just assume the rig is wired correctly:

```c
testcase tc_ClosedLoopSanityCheck()
{
  testCaseTitle("ECU's actuator output measurably changes plant model feedback");
  float before, after;
  before = HilGetSignal("SteeringRackPosition");
  HilInjectTorqueCommand(5.0);   // command the ECU to request 5Nm of assist
  testWaitForTimeout(200);
  after = HilGetSignal("SteeringRackPosition");
  testStepCheck("rack position changed in response to ECU command", after != before);
}
```

Running this before every scenario suite catches a mis-wired or
mis-configured rig (open-loop by accident) immediately, rather than
producing a suite full of misleadingly "passing" tests that never
actually exercised the ECU's control behavior at all.

## Cheat sheet

| Concept | Key point |
|---|---|
| Two-layer split | Real-time plant model vs. non-real-time orchestration |
| Parameter/signal interface | Orchestration never touches model internals directly |
| Scenario parameterization | One testcase body, many representative operating points |
| Settle waits | Always wait for transients before asserting or starting the next scenario |
| Loop-closure sanity check | A standing testcase proving the rig is actually closed-loop |

## Exercise

1. `RunAbsActivationScenario` above waits a fixed 1000ms for
   `ABS_Active` regardless of scenario. Rewrite its signature and body
   to make that timeout itself a parameter, and explain why a single
   fixed timeout across a friction matrix (0.05 to 0.9) is a design
   smell.
2. Design a `tc_ClosedLoopSanityCheck`-style test for a simulated
   engine-cooling HIL rig, where the real ECU commands a cooling-fan
   duty cycle that should measurably change a simulated coolant
   temperature over time. Name the two signals/parameters involved and
   the settle time you'd choose, with reasoning.
3. Explain, using the synchronization pitfalls table, what could go
   wrong if a testcase reads `ABS_Active` on a plain 10ms wall-clock
   polling loop instead of a model-step-boundary event, on a heavily
   loaded orchestration host.
