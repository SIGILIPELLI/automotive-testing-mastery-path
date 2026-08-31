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
