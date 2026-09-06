# 07 · Supplier & OEM Test Collaboration Models

Level 4 Module 1 flagged the supplier/OEM split as a core test
strategy decision. This module goes deeper into how that
collaboration actually works day to day: what evidence crosses the
boundary, how disagreements over a failing test get resolved, and the
contractual/process concepts (like ASPICE) that structure the
relationship.

!!! note "About this module"
    No specific OEM-supplier contract or ASPICE assessment was used to
    produce this content. The collaboration patterns below reflect
    common industry practice around ASPICE and automotive supplier
    quality processes — actual contractual terms vary significantly by
    OEM and program.

## The fundamental tension

An OEM needs confidence that a supplier's ECU meets requirements
without re-doing all of the supplier's testing itself (expensive,
slow, and redundant per Level 4 Module 1). A supplier needs to protect
proprietary implementation details (the actual CAPL test source, the
security-key algorithm from Level 4 Module 4) while still providing
enough evidence to earn that confidence. Every practice in this module
exists to resolve that tension.

## ASPICE: the process-maturity language both sides share

Automotive SPICE (ASPICE) is a process assessment model many OEMs
require suppliers to demonstrate capability against. For testing
specifically, the relevant process areas are:

| ASPICE process | What it assesses | Test-relevant work product |
|---|---|---|
| SWE.4 — Software unit verification | Unit-level test coverage and results | Unit test reports |
| SWE.5 — Software integration/integration testing | Integration test strategy and execution | Integration test plan/report — overlaps directly with this course's Level 2-3 CAPL/CANoe testing |
| SWE.6 — Software qualification testing | Full software-level requirement verification | Traceability matrix (Level 3 Module 8), test reports |
| SYS.4/SYS.5 — System integration/qualification testing | Multi-ECU, system-level testing | Often OEM-owned per Level 4 Module 1's split |

An ASPICE capability level (0–5) rating isn't a pass/fail grade on
individual tests — it rates whether the *process* around testing
(planning, traceability, defect management, review) is mature and
repeatable. A supplier can have genuinely good engineers writing good
CAPL tests and still score poorly on ASPICE if the traceability and
review process around that work isn't documented and consistently
followed — which is precisely why Level 3 Module 8's traceability
discipline is not optional formality.

## What evidence actually crosses the boundary

| Evidence type | Typical exchange |
|---|---|
| Test reports (pass/fail + verdict) | Always exchanged — the OEM's primary confidence signal |
| Traceability matrix (Module 8 style) | Usually exchanged, at least in summary/coverage-percentage form |
| Raw CAPL test source | Sometimes withheld as supplier IP — OEM may instead require a description of test technique used, not the exact script |
| Raw bus traces from a specific failure | Exchanged on request, especially for a defect under joint investigation |
| Fault-injection matrix (Level 3 Module 6) | Usually exchanged in summary — full coverage grid, not necessarily every underlying testcase |

The "test technique described, source withheld" pattern is common and
legitimate — an OEM reviewing a supplier's test strategy needs to know
*that* boundary values and fault injection were applied to a given
requirement, not necessarily the exact CAPL implementation, in the
same way a customer trusts a vendor's quality process without auditing
every line of their source code.

## Handling a disputed test result

A recurring, structurally important scenario: the OEM's own
integration testing shows a failure the supplier's unit-level testing
never caught, or the two sides get different results running what's
supposed to be the same test.

```text
Structured dispute-resolution flow:
  1. Confirm the exact configuration baseline (Level 4 Module 5) each
     side used -- version skew between OEM and supplier test
     environments (different DBC/A2L/software build versions) is the
     single most common cause of "different results, same test."
  2. If baselines genuinely match, reproduce the OEM's failing
     scenario in the supplier's own environment -- does it reproduce?
  3. If it reproduces: joint root-cause investigation, defect filed
     against the supplier's software (or, less commonly, the OEM's
     test setup) with the specific baseline and trace evidence.
  4. If it does NOT reproduce: this usually signals an environmental
     difference (a rig-specific timing characteristic, a load
     condition unique to the OEM's integration rig) that itself
     becomes an investigation item -- not a reason to dismiss either
     side's result.
```

Step 1 alone resolves a large fraction of real cross-organization
disputes — Level 4 Module 5's baseline discipline exists specifically
to make step 1 fast and unambiguous instead of a multi-day
finger-pointing exercise.

## Contractual test milestones

Supplier programs typically gate payment or program progression on
named test milestones, each with defined entry/exit criteria similar
in spirit to Level 3 Module 10's project exit criteria, but scoped
across the whole supplier relationship:

```text
Milestone: Software Release Candidate Test Complete
  Entry criteria:
    - All SW-REQ traceability links closed (Module 8)
    - No open ASIL C/D defects
  Exit criteria:
    - Test report package delivered to OEM in agreed format
    - Traceability matrix coverage >= agreed threshold (e.g. 100% for
      ASIL C/D, negotiated threshold for QM)
  Evidence delivered to OEM:
    - Summary test report, traceability matrix, fault-injection
      coverage matrix, open-defect list with severity/ASIL
```

## Cheat sheet

| Concept | Key point |
|---|---|
| Core tension | OEM needs confidence without redundant re-testing; supplier needs to protect proprietary detail |
| ASPICE | Rates process maturity around testing, not individual test correctness |
| Evidence boundary | Reports/traceability/coverage typically exchanged; raw test source sometimes withheld as IP |
| Dispute resolution | Always check configuration baseline match first — the most common real cause of "different results" |
| Contractual milestones | Named entry/exit criteria gate program progression, mirroring Level 3 Module 10's plan structure at supplier-relationship scale |

## How It Actually Works

**Why "same test, different result" is usually a configuration
problem, mechanically, not a testing-competence problem.** When an
OEM and a supplier each claim to run "the same" CAPL testcase against
"the same" ECU and get different verdicts, the actual CAPL source
files can be byte-identical and still produce different results if
the DBC signal scaling loaded on either side differs even slightly
(Level 4 Module 5) — a `setSignal(ForwardDistance_m, 15.0)` call
writes a genuinely different raw CAN payload value depending on which
DBC's scaling factor is active, so the ECU under test is receiving a
different physical stimulus than either side believes. This is why
Step 1 of the dispute flow — comparing configuration baselines field
by field — resolves most disputes before any actual root-cause
investigation begins: the two "identical" tests were frequently never
actually identical below the CAPL source level.

**Why "test technique described, source withheld" still gives an
assessor enough to certify against.** An ASPICE SWE.6 or ISO 26262-8
independent review needs to confirm specific things about a test: that
it exercises boundary values, that it traces to a requirement, that a
qualified reviewer other than the author checked its adequacy. None of
that inherently requires reading the CAPL statements themselves — a
supplier can satisfy the requirement with a structured technique
description (e.g., "requirement SW-REQ-202 verified via equivalence
partitioning and boundary analysis on ForwardDistance_m; independent
review completed by [named reviewer] on [date]; verdict Passed")
alongside the traceability matrix, because what's being assessed is
whether the *method* was rigorous and independently checked, not
whether the OEM personally re-derives the test logic. The OEM's real
leverage point if it doubts the description's honesty isn't demanding
the source — it's requesting a live demonstration or witnessed
re-execution of the test, which proves the technique was genuinely
applied without requiring IP disclosure.

**Why milestone entry criteria have to be checked before exit criteria
are even attempted.** A milestone like "Software Release Candidate
Test Complete" listing "no open ASIL C/D defects" as an *entry*
criterion (not just exit) is a deliberate ordering: if a supplier
starts the release-candidate test campaign while ASIL C/D defects are
still open, every test result produced during that campaign is
evidence about a software state that's about to change again once
those defects are fixed — meaning some or all of the campaign's
evidence becomes stale and has to be re-run against the fixed build.
Structuring entry criteria to block starting the expensive, formal
test campaign until the software is actually in a stable, defect-clear
state (for the ASIL tiers that matter most) is what prevents wasted
program-scale test effort, not just a compliance formality.

## Exercise

1. An OEM integration test shows an intermittent failure the supplier
   cannot reproduce after 50 attempts in their own lab. Using the
   dispute-resolution flow, list the specific baseline fields (Level 4
   Module 5) you'd request from the OEM before concluding this is a
   genuine environmental difference rather than a flaky test (Level 3
   Module 7).
2. A supplier wants to withhold their CAPL fault-injection test source
   as proprietary but still needs to satisfy an OEM's ASIL D
   independent-review requirement (Level 3 Module 4). Propose an
   evidence format that satisfies the review requirement without
   disclosing the exact CAPL implementation.
3. Draft the entry/exit criteria for a "Fault Injection Coverage
   Complete" contractual milestone, referencing Level 3 Module 6's
   fault matrix concept, including what coverage percentage or gap-
   acceptance language you'd require before sign-off.
