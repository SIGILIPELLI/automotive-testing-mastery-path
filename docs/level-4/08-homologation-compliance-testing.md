---
description: "Homologation & Compliance Testing Overview — Everything so far validated that an ECU meets an OEM's own requirements. Homologation is a different gate…"
---

# 08 · Homologation & Compliance Testing Overview

Everything so far validated that an ECU meets an OEM's own
requirements. Homologation is a different gate entirely: proving to a
regulatory authority that a vehicle (and the ECUs that implement
regulated functions) meets legally mandated standards before it can be
sold in a given market. This module gives a test engineer's-eye
overview of what that involves and how it connects to the testing
skillset already built.

!!! note "About this module"
    This module is a general overview based on publicly documented
    regulatory frameworks (UN ECE regulations, FMVSS) — it is not
    legal or regulatory compliance advice, and any real homologation
    program must be run by qualified regulatory/compliance
    specialists with jurisdiction-specific expertise.

## Homologation vs. internal validation

| | Internal test strategy (Level 4 Module 1) | Homologation |
|---|---|---|
| Who defines the requirement | The OEM (informed by market/competitive needs) | A regulatory body (UN ECE, NHTSA/FMVSS, regional equivalents) |
| Consequence of failure | Program delay, rework | Vehicle cannot be legally sold/registered in that market |
| Evidence formality | Internal review standards | Often requires accredited test labs, specific approved procedures, government-witnessed testing |
| Applies to | Whatever the OEM chooses to build | Specific regulated functions only (braking, lighting, emissions, certain ADAS functions in some markets) |

Not every ECU or feature is homologation-relevant — a courtesy-light
control module's timing has no regulatory test requirement, while an
AEB system's activation performance may be directly regulated in
markets that mandate it (e.g., under relevant UN ECE regulations for
advanced emergency braking systems).

## Where CAN/CANoe/HIL testing supports (but doesn't replace) homologation

| Activity | Role of this course's skillset |
|---|---|
| Pre-homologation internal validation | HIL scenario testing (Level 3) against the *anticipated* regulatory test procedure, to catch failures before the costly formal test | 
| Regression after a late software change | Confirming a change doesn't regress previously-homologated behavior, using the same regression-suite discipline (Level 3 Module 7) |
| Supporting evidence for a compliance dossier | Internal test reports and traceability (Level 3 Module 8) often form supporting technical documentation, even though they don't substitute for the formal regulatory test itself |
| The formal homologation test | Typically performed by or witnessed by an accredited/government-recognized test facility, following a specific mandated procedure — outside the scope of an internal CANoe/HIL rig |

The practical value of Level 3's HIL skillset here is **derisking**: a
team that has already run a AEB HIL scenario matrix closely modeled on
the anticipated regulatory test procedure walks into the formal,
expensive, hard-to-repeat homologation test with far higher confidence
of passing on the first attempt.

## A worked example: modeling an anticipated regulatory scenario in HIL

```c
// Illustrative -- an internal HIL scenario modeled after a publicly
// documented AEB regulatory test scenario category (a lead vehicle
// decelerating scenario), for PRE-homologation internal confidence
// only. This is NOT a substitute for the actual accredited
// regulatory test procedure, which specifies exact vehicle/target
// parameters, approach speeds, and pass criteria in far more detail
// than this simplified sketch.
testcase tc_PreHomologation_LeadVehicleDecelerationScenario()
{
  testCaseTitle("[PRE-HOMOL-AEB-03] Internal rehearsal of anticipated lead-vehicle deceleration AEB scenario");
  HilSetParameter("EgoSpeed_kph", 50.0);
  HilSetParameter("LeadVehicleSpeed_kph", 50.0);
  HilSetParameter("LeadVehicleDeceleration_mps2", -6.0); // sudden hard braking ahead
  testWaitForTimeout(500);

  testStepCheck("AEB activated before simulated collision point",
                 getSignal(AEB_BrakeCommand) == 1);
  // Real homologation pass criteria are typically speed-reduction or
  // collision-avoidance thresholds defined precisely by the regulation
  // -- this internal rehearsal should mirror those thresholds as
  // closely as the team's regulatory-affairs function can specify,
  // not invent its own.
}
```

The comment matters more than the code here: the single biggest risk
in "internal rehearsal" testing is quietly drifting from the actual
regulatory procedure and pass criteria, producing false confidence.
The internal test's parameters should be sourced from and reviewed by
whoever on the program owns actual regulatory-affairs expertise, not
assumed or approximated by the test team alone.

## Compliance documentation: what a dossier typically needs

| Document | Content | Relationship to earlier modules |
|---|---|---|
| Type-approval technical file | Design description, test results, conformity evidence | Traceability matrix (Module 8) often feeds directly into this |
| Declaration of conformity | A formal statement the vehicle/system meets the applicable regulation | Signed off by regulatory/quality function, not test engineering alone |
| Test reports from accredited facility | The actual homologation test results | Distinct from and not replaceable by internal CANoe/HIL reports |

## Cheat sheet

| Concept | Key point |
|---|---|
| Homologation vs. internal validation | Regulatory-mandated, legally consequential, distinct from OEM-internal requirements |
| Not every feature is regulated | Homologation applies to specific mandated functions, not the whole vehicle equally |
| HIL's role | Pre-homologation derisking/rehearsal, never a substitute for the formal accredited test |
| Drift risk | Internal rehearsal scenarios must be sourced from actual regulatory-affairs expertise, not approximated by the test team |
| Dossier | Traceability and internal test evidence often support, but don't replace, the formal compliance documentation |

## How It Actually Works

**Why a HIL rehearsal can pass while the formal test still fails —
the specific gap.** `tc_PreHomologation_LeadVehicleDecelerationScenario`
drives the ECU's software logic against a plant-model-simulated lead
vehicle and radar signal, exercising exactly the decision-making
algorithm the regulation cares about. But a formal regulatory AEB test
uses a physical target vehicle (or a certified soft-target surrogate),
a real radar/camera sensor suite, and real vehicle dynamics — meaning
it also exercises sensor-fusion accuracy under real-world radar
clutter, actual brake-actuator response time and vehicle deceleration
dynamics, and environmental conditions (weather, target-surface radar
reflectivity) that a signal-level HIL rehearsal cannot represent at
all, because those signals were injected directly rather than produced
by real sensors and physics. A pass in HIL proves the ECU's *decision
logic* is correct given a specific assumed input; it says nothing
about whether the *sensors and actuators* feeding that logic will
behave as assumed in the physical test — which is precisely the
distinction the module's cheat sheet insists on and Exercise 1 is
built to surface.

**Why the internal test's pass criteria have to be sourced externally,
not inferred from the regulation's plain text.** Regulatory AEB test
procedures typically specify pass criteria as precise, parameterized
formulas — e.g., a required speed reduction as a function of the
approach speed and target deceleration profile, evaluated at a
specific measurement point relative to the target — rather than a
simple binary "did it brake." A test team approximating this as "brake
command must assert before an estimated collision point," as the
worked example's comment flags, can produce a testcase that passes
reliably in HIL while not actually mirroring the regulation's real
threshold, because the approximation and the true formula diverge at
exactly the boundary conditions that matter most. This is a mechanism-
level instance of a general risk: any pass criterion derived by
engineering judgment from a regulation's summary, rather than from the
regulation's actual clause and the regulatory-affairs function's
interpretation of it, tends to be systematically optimistic precisely
where the real test is hardest to pass.

**Why traceability links have to point at both an internal requirement
and an external regulation clause, structurally.** A CAPL testcase
tagged only with an internal requirement ID (`SW-REQ-201`) gives an
OEM engineer everything they need to trace coverage internally, but
gives a regulatory-affairs reviewer building the type-approval
technical file nothing to cite — that dossier has to reference the
specific regulation clause number the evidence supports, in the
regulator's own terminology, not the OEM's internal requirement
scheme. Carrying both tags on the same testcase (as Exercise 3 asks
for) means one piece of evidence serves two structurally different
audiences without requiring a separate parallel test suite, or a
manual, error-prone remapping exercise every time the dossier is
assembled.

## Exercise

1. Explain in your own words why a passing result on
   `tc_PreHomologation_LeadVehicleDecelerationScenario` should never
   be reported to program leadership as "AEB homologation passed" —
   what specific gap exists between this test and an actual formal
   regulatory test?
2. Identify two ECUs on a typical vehicle program (one clearly
   homologation-relevant, one clearly not), and justify the
   distinction using the "specific regulated functions only" point
   above.
3. Draft the traceability link (in the Level 3 Module 8 tagging style)
   connecting an internal pre-homologation testcase to both an
   internal software requirement AND a named external regulation
   reference, and explain why both links are useful to different
   audiences (engineering vs. regulatory-affairs).
