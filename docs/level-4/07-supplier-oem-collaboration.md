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
