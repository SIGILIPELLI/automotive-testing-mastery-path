# 09 · SOTIF & Testing for Autonomous Features

ISO 26262 (Module 4) covers hazards caused by *system malfunction* —
something broke. ISO 21448, SOTIF (Safety Of The Intended
Functionality), covers a different and harder problem: hazards that
occur even when everything works exactly as designed, because the
design's *intended* behavior wasn't good enough for a situation it
encountered. This module covers what SOTIF actually asks for and how
testing for it differs fundamentally from functional-safety testing.

!!! note "About this module"
    No physical ADAS rig, simulation environment, or scenario-mining
    dataset was used to produce this content. The SOTIF concepts and
    scenario-classification model below reflect the documented ISO
    21448 standard and common industry practice — a real SOTIF case
    requires far more extensive simulation/road-data evidence than
    this module can substitute for.

## The core distinction: malfunction vs. insufficient performance

| | ISO 26262 (functional safety) | ISO 21448 (SOTIF) |
|---|---|---|
| Trigger | A component fails or malfunctions | The system works exactly as designed, but the design is insufficient for a real-world situation |
| Example | A radar sensor's ECU crashes | A radar sensor correctly reports what it sees, but fails to classify a stationary object crossing an unusual profile (e.g., a jackknifed trailer) because the perception algorithm was never trained/validated for that shape |
| Root fix | Redundancy, fault detection, safe-state transition | Broaden the validated operating envelope, add complementary sensing, restrict where the feature is allowed to operate |

SOTIF matters most for features with a learned or complex perception
component — ADAS (AEB, lane-keeping, adaptive cruise) and autonomous
driving functions — where "the software has no bug" and "the system
is still unsafe in some situation" can both be true simultaneously.

## The SOTIF scenario categories

ISO 21448 organizes the operating space into four areas, often drawn
as overlapping regions:

| Area | Meaning | Testing goal |
|---|---|---|
| **Area 1** — Known safe | Scenarios understood, and shown safe | Regression-test to keep it that way |
| **Area 2** — Known unsafe | Scenarios understood, and shown to cause unacceptable risk | Design the system to avoid/restrict them (e.g., geofencing, operational design domain limits); test that avoidance/restriction works |
| **Area 3** — Unknown unsafe | Scenarios not yet identified, but unsafe if encountered | The hardest and most important area to shrink — found through scenario mining, edge-case simulation, and field data, not designed-in-advance test cases |
| **Area 4** — Unknown safe | Scenarios not yet identified, and would be safe anyway | Lower priority but shrinking Area 3 usually also surfaces some of Area 4 |

The entire practical goal of SOTIF validation activity is **moving
scenarios from Area 3 (unknown-unsafe) into Area 2 (known-unsafe,
addressed) or Area 1 (known-safe)** — you cannot write test cases
against scenarios you haven't identified yet, so a large part of the
practice is scenario *discovery*, not scenario *execution*.

## Where CANoe/HIL fits, and where it clearly doesn't

| Activity | Suitable tooling |
|---|---|
| Replaying a known critical scenario (Area 2) against the ECU in HIL to confirm mitigation works | CANoe/HIL, same closed-loop discipline as Level 2 Module 9 |
| Regression-testing Area 1 scenarios stay safe after a software change | HIL scenario matrix, similar structure to Module 1's ABS matrix |
| Discovering unknown-unsafe scenarios (Area 3) | Large-scale simulation, scenario mining from field/fleet data, statistical/ML-based edge-case search — largely outside CAPL/CANoe's scope and outside this course's covered tooling |
| Perception-algorithm-level validation (does the object classifier correctly label an edge-case shape) | Specialized ML validation tooling, not signal-level bus testing |

A test engineer coming from the CAN/CANoe/HIL background this course
has built should recognize their tooling's ceiling honestly: it
excels at **Area 2 confirmation and Area 1 regression**, but Area 3
discovery is a different discipline (often owned by a dedicated
validation/simulation team) that this course does not cover in depth.

## A HIL-level SOTIF regression testcase (Area 2 confirmation)

```c
// Illustrative — confirming a known-unsafe scenario is now correctly
// mitigated (moved from Area 2 identified-unsafe to Area 1 known-safe
// via a design change, e.g. an added ODD restriction or sensor fusion
// rule). This tests the MITIGATION, not the original perception gap
// itself, which is outside CAPL's reach.
testcase tc_SotifKnownScenario_LowSunGlareFalsePositiveSuppressed()
{
  testCaseTitle("[SOTIF-AREA2-014] AEB does not false-brake under simulated low-sun-glare radar clutter profile");
  // The specific clutter/glare profile is injected via the plant model's
  // simulated sensor feed, not via CAPL signal manipulation directly —
  // this requires HIL sensor-simulation capability beyond pure CAPL.
  HilSetParameter("SimulatedSensorProfile", "LOW_SUN_GLARE_CLUTTER_014");
  testWaitForTimeout(2000);

  testStepCheck("AEB did not command unintended braking",
                 getSignal(AEB_BrakeCommand) == 0);
  testStepCheck("real obstacle 30m ahead still correctly detected",
                 getSignal(AEB_ObstacleDetected) == 1);
}
```

Both assertions matter together: a mitigation that suppresses the
false positive by simply making the system less sensitive overall
would pass the first check while silently failing the second — a
classic SOTIF trap where fixing a known-unsafe scenario introduces a
new insufficient-performance gap elsewhere.

## Operational Design Domain (ODD) as a SOTIF testing boundary

Many SOTIF mitigations work by restricting *where* a feature is
allowed to operate rather than making the underlying perception
perfect everywhere — an ODD limit (e.g., "adaptive cruise disengages
below 10°C ambient with active precipitation detected"). Testing an
ODD restriction has two parts:

1. **Inside the ODD**: the feature performs to its validated
   standard (standard functional/HIL testing, Areas 1/2).
2. **At the ODD boundary**: the feature correctly detects it's leaving
   the ODD and disengages/hands back control gracefully — this
   boundary-transition behavior is itself testable with the CAPL/HIL
   toolset, since it's a well-defined signal-level condition
   (temperature, precipitation sensor state) triggering a well-defined
   response (disengagement).

## Cheat sheet

| Concept | Key point |
|---|---|
| SOTIF vs. functional safety | SOTIF addresses hazards from insufficient performance with no malfunction, not from failures |
| Four scenario areas | Known-safe, known-unsafe, unknown-unsafe, unknown-safe — the goal is shrinking unknown-unsafe |
| CANoe/HIL's role | Strong for Area 1 regression and Area 2 mitigation confirmation; not a substitute for Area 3 scenario discovery |
| Mitigation side-effects | Always test that a fix for a known-unsafe scenario didn't create a new insufficient-performance gap |
| ODD boundary testing | The transition/disengagement behavior at an ODD limit is directly testable even when the underlying perception isn't |

## Exercise

1. Classify each of the following into one of the four SOTIF areas,
   with justification: (a) a lane-keep system tested extensively on
   painted highway lines and known to work correctly; (b) the same
   system's behavior on a road with faded, partially-covered-by-snow
   lane markings that has never been tested; (c) a scenario the team
   has explicitly decided the vehicle should refuse to operate in at
   all (e.g., unmapped off-road terrain).
2. `tc_SotifKnownScenario_LowSunGlareFalsePositiveSuppressed` checks
   two things together. Design a third assertion that would catch a
   mitigation that works correctly in this exact glare profile but
   only by disabling AEB broadly at low sun angles regardless of
   glare severity — a narrower, more targeted regression check.
3. Explain in your own words why "we ran 10,000 simulated scenarios
   and found zero failures" is not, by itself, sufficient SOTIF
   evidence — what additional information about how those 10,000
   scenarios were chosen would you need before trusting that result?
