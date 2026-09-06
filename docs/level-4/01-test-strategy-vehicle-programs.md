# 01 · Test Strategy for Vehicle Programs

Everything through Level 3 operated at the scale of one ECU, one
feature, one rig. A vehicle program test strategy operates at a
different scale entirely — dozens of ECUs, multiple suppliers,
years-long timelines, and a budget that has to be justified against
program risk. This module covers how a test strategy document gets
built and why its structure matters as much as its technical content.

!!! note "About this module"
    No specific OEM's actual program test strategy was used to
    produce this content. The structure and reasoning below reflect
    common automotive V-model/ASPICE program practice — adapt the
    template to your program's actual governance and supplier model.

## What a test strategy is (and isn't)

A test strategy is not a test plan (Level 3 Module 10 built one of
those, scoped to a single feature). A test strategy is the layer
above: it decides **how testing is organized across the whole
program** — which level of the V-model tests what, which
organization (OEM vs. supplier) owns which tests, and how much of the
program's risk budget goes to testing versus other assurance
activities.

| Document | Scope | Owner |
|---|---|---|
| Test strategy | Whole vehicle program, all ECUs | Program test lead / systems engineering |
| Test plan (Level 3 Module 10 style) | One feature or ECU | Feature/component test lead |
| Test case / testcase (CAPL) | One specific check | Test engineer |

## The V-model as the strategy's backbone

```text
Requirements level          Corresponding test level
--------------------------  --------------------------------
Vehicle requirements    <->  Vehicle-level validation (proving ground, fleet)
System requirements     <->  System integration testing (multi-ECU HIL)
Software requirements   <->  Software/unit + component HIL testing (this course's core)
```

A test strategy's central job is deciding, level by level: what gets
tested at that level, by whom, with what tooling, and — critically —
what does NOT need to be re-tested at the next level up because it was
already proven down here. Without that explicit "don't re-test"
decision, programs drift toward expensive, redundant testing at every
level for the same behavior.

## Supplier vs. OEM test responsibility split

Most ECUs on a vehicle program are supplier-built. The test strategy
must state, per ECU or ECU family, who runs what:

| Test level | Typical owner | Typical evidence exchanged |
|---|---|---|
| Component/software-level (CAPL/CANoe testcases like this course's) | Supplier | Test reports, traceability matrix (Level 3 Module 8 style), often mapped to ASPICE work products |
| System integration (multi-ECU bus interaction) | OEM (or a Tier 1 integrator) | Integration test reports, defect logs against supplier ECUs |
| Vehicle-level validation | OEM | Proving-ground/fleet validation reports |

A test strategy that doesn't specify this split explicitly leads to
the two most common program-level test failures: **gaps** (each side
assumes the other tested a specific interaction) and **duplication**
(both sides run near-identical tests, wasting budget). Module 7 covers
the supplier/OEM collaboration model in more depth.

## Risk-based test allocation

A program cannot test everything with equal rigor — the strategy must
allocate test effort by risk, not evenly:

| Risk driver | Test allocation implication |
|---|---|
| ASIL (Level 3 Module 4) | Higher ASIL requirements get more test techniques (boundary, fault injection, independent review), not just "a test exists" |
| New vs. carryover technology | A genuinely new ECU/feature gets full-depth testing; a carryover component with an unchanged software baseline may only need regression confirmation |
| Program timeline pressure | Late-program schedule risk should shift effort toward automated regression (Level 3 Module 7) that can absorb late changes cheaply, not toward expanding manual test scope |
| Known field-issue history (for a carryover platform) | Prior recalls/campaigns on a similar system should pull disproportionate test attention onto the analogous new-program feature |

## Tooling and environment strategy

The strategy document should also commit the program to a toolchain
consistent across suppliers where interoperability matters — a
program with 12 suppliers each choosing incompatible test-report
formats cannot build the single traceability view Level 3 Module 8
depends on. Common commitments:

```text
- Common signal database format (DBC/ARXML) and version-control process
  across all suppliers touching a shared bus
- Common diagnostic test evidence format (UDS test report schema)
- A single traceability tool (or defined export format) all suppliers
  feed into, so the OEM can build one program-wide coverage view
- Defined HIL rig availability windows per program milestone
  (Level 4 Module 5 covers scaling this further)
```

## A worked mini-strategy excerpt

```text
Program: Model-Year X Mid-Cycle Refresh — ADAS Domain Controller

Test strategy summary:
  - Software-level CAPL/CANoe testing: supplier-owned, ASIL D
    requirements require independent test review per ISO 26262-8.
  - System integration (ADAS DC <-> body/chassis ECUs): OEM-owned,
    executed on a shared multi-ECU HIL rig at the OEM's integration
    lab, twice per major milestone.
  - Vehicle-level AEB/lane-keep validation: OEM-owned, proving ground,
    scenario set drawn from both known-safe regression (SOTIF Area 1)
    and newly mined known-unsafe scenarios (SOTIF Area 2) per Level 3
    Module 9.
  - Carryover components (unchanged from prior model year): regression
    confirmation only, no new test-case development, unless a
    supplier software change is introduced mid-program.
```

## Cheat sheet

| Concept | Key point |
|---|---|
| Strategy vs. plan | Strategy governs the whole program's test organization; a plan governs one feature |
| V-model backbone | Explicitly decide what's tested at each level and what's NOT re-tested at the next |
| Supplier/OEM split | Must be explicit per ECU — ambiguity causes gaps or duplication |
| Risk-based allocation | ASIL, new-vs-carryover, and field history should drive where test effort concentrates |
| Toolchain commitments | Needed for a program-wide traceability/coverage view to be possible at all |

## How It Actually Works

**Why "don't re-test" is a technical decision, not just a scheduling
one.** When a test strategy says software-level CAPL testing "proves"
a requirement so system integration doesn't need to re-prove it, that
claim only holds if the software-level test environment's stimulus is
representative of what the ECU actually sees in the vehicle. A common
real defect: a supplier's CAPL testcase stimulates a signal directly
on the bus (`setSignal(WheelSpeedFL, 60.0)`), but the real vehicle
signal arrives through a gateway ECU that applies its own scaling,
rate-limiting, or timestamp translation. If the gateway's behavior
differs even slightly from a raw bus write, the software-level test
proved something true of the isolated ECU under idealized stimulus,
not of the ECU as it behaves inside the actual signal chain — which is
exactly the gap system integration testing exists to close, and why a
strategy that eliminates integration testing purely on the strength of
software-level pass rates is making an unverified representativeness
assumption.

**Why ASIL-driven allocation isn't just "more tests for higher ASIL."**
ISO 26262 ties ASIL to specific *techniques*, not merely test volume.
An ASIL D requirement is expected to show evidence of methods like
boundary-value analysis, fault injection, and independent test review
(a second qualified engineer, not the author, confirming test adequacy)
per ISO 26262-8's confirmation measures. An ASIL QM requirement may be
adequately covered by a single nominal-path functional test. A test
strategy document that just says "ASIL D gets 3x as many testcases as
ASIL B" without specifying *which techniques* those testcases must
collectively exercise is not actually implementing risk-based
allocation — it's just spending more effort on the same shallow
technique repeated, which an assessor reviewing the ISO 26262-8 work
products will flag as a gap regardless of raw testcase count.

**Why the supplier/OEM evidence exchange format matters mechanically,
not just administratively.** When a supplier's test report and the
OEM's traceability tool use different requirement ID schemes or
different pass/fail terminology, the OEM's program-wide coverage view
either silently drops the supplier's evidence (a requirement shows as
"untested" in the OEM's tool even though the supplier tested it) or
double-counts it against the wrong requirement. This is why the
tooling-commitment section isn't a bureaucratic nicety — a single
canonical requirement-ID scheme, referenced identically in the
supplier's CAPL `testCaseTitle` tags (as Level 3 Module 8 established)
and the OEM's traceability import, is the only thing that makes
automated coverage rollup across a multi-supplier program mechanically
possible rather than a manually reconciled spreadsheet exercise that
falls out of date the moment a milestone passes.

## Exercise

1. A supplier and the OEM both ran near-identical CAN-timing
   regression tests for the same ECU interface, discovered only after
   both teams presented results at a milestone review. Identify which
   part of the strategy template above should have prevented this,
   and rewrite that section to close the gap.
2. For the worked mini-strategy excerpt, a mid-program software change
   is introduced to a "carryover, regression-only" component.
   Describe how the strategy should require this to be re-classified,
   and what test activities re-open as a result.
3. Design the risk-based allocation table for a fictional new seat-
   occupancy-classification ECU (new technology, ASIL B, no field
   history) versus a carryover power-window ECU (unchanged for 3
   model years, ASIL QM) — state concretely how test effort should
   differ between the two.
