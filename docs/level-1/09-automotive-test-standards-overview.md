# 09 · Automotive Test Standards Overview

Automotive testing doesn't just happen to be rigorous — much of that
rigor is a direct, traceable consequence of two standards every serious
automotive supplier and OEM works against: **ASPICE** (process maturity)
and **ISO 26262** (functional safety). This module gives you enough of
each to understand *why* the practices in Modules 1-8 (traceability,
boundary testing, documented expected results, the V-model itself) are
not just good habits — they're often contractual and legal obligations.
Later levels go deeper (Level 3, Module 4 covers ISO 26262 testing
requirements in detail; Level 4 covers ISO 21434 cybersecurity).

## ASPICE: Automotive SPICE

**ASPICE** (Automotive Software Process Improvement and Capability
Determination) is a process-assessment model, based on the international
standard ISO/IEC 33020, adapted specifically for automotive software
and system engineering. OEMs frequently require suppliers to demonstrate
a specific ASPICE **capability level** on relevant processes as a
condition of doing business — it is as much a commercial gate as a
technical one.

### The capability levels

| Level | Name | Meaning |
|---|---|---|
| 0 | Incomplete | The process isn't implemented, or fails to achieve its purpose |
| 1 | Performed | The process achieves its purpose, but work products aren't managed |
| 2 | Managed | Performed **and** planned, monitored, and adjusted; work products controlled |
| 3 | Established | Managed **and** implemented using a defined, tailored standard process |
| 4 | Predictable | Established **and** operates within defined limits, quantitatively managed |
| 5 | Innovating | Predictable **and** continuously improved |

Most OEM contracts target **Level 2 or 3** on the processes most
relevant to safety and quality — and the testing-relevant processes are
exactly the ones this course's practices map onto directly.

### The testing-relevant ASPICE processes

| Process ID | Name | What it assesses |
|---|---|---|
| **SWE.4** | Software Unit Verification | Are unit tests planned, executed, and their results recorded against defined criteria? |
| **SWE.5** | Software Integration and Integration Test | Is integration tested against the architecture, with documented test cases and results? |
| **SWE.6** | Software Qualification Test | Is the complete software tested against its requirements? |
| **SYS.4** | System Integration Test | Are integrated systems (multiple ECUs) tested against system architecture? |
| **SYS.5** | System Qualification Test | Is the complete system tested against system requirements? |

Notice the direct mapping onto the V-model from Module 1: SWE.4 is
unit test, SWE.5/SYS.4 are integration test, SWE.6/SYS.5 are system
test. ASPICE doesn't invent new testing concepts — it requires that the
V-model's test levels are actually **planned, executed, and evidenced**,
with traceability from requirement to test case to result. This is the
direct organizational reason Module 8's traceability matrix and fully
documented test-case fields aren't bureaucratic overhead — an assessor
will ask to see exactly that evidence.

## ISO 26262: functional safety

**ISO 26262** ("Road vehicles — Functional safety") governs the
development of electrical/electronic systems where a malfunction could
cause harm — brakes, steering, airbags, and increasingly, driver-assist
features. Its core testing-relevant idea is the **Automotive Safety
Integrity Level (ASIL)**.

### ASIL: risk-based rigor

Every safety-relevant function is assigned an ASIL — **A** (lowest),
**B**, **C**, or **D** (highest), or **QM** (Quality Managed — no
special safety process required) — derived from a formal risk
assessment considering:

| Factor | Meaning |
|---|---|
| **Severity (S)** | How bad is the harm if it happens? (S0 none → S3 life-threatening/fatal) |
| **Exposure (E)** | How often is the vehicle in the situation where this matters? (E0 incredible → E4 high probability) |
| **Controllability (C)** | Can the driver/situation reasonably avoid harm even if the malfunction occurs? (C0 controllable → C3 difficult/uncontrollable) |

Higher S×E×C combinations drive a higher ASIL, and **higher ASIL directly
mandates more rigorous testing** — this is the single most important
practical fact to take from this standard:

| ASIL | Typical unit test coverage expectation | Typical test technique rigor |
|---|---|---|
| QM | Good practice, no mandated coverage metric | Standard test-design techniques |
| A | Statement coverage | Equivalence partitioning, boundary analysis |
| B | Statement + branch coverage | + robustness/error-handling test |
| C | Branch coverage, MC/DC recommended | + resource usage/fault injection test |
| D | MC/DC (Modified Condition/Decision Coverage) | + highest rigor: independent test team, formal methods considered |

**MC/DC** — a coverage criterion requiring that each condition in a
complex boolean decision be shown to independently affect the outcome —
is the standard's hallmark requirement at ASIL D, and it's dramatically
more expensive to achieve than simple branch coverage, which is exactly
why ASIL assignment (not blanket maximum rigor everywhere) matters: it
concentrates the most expensive testing on the parts of the system where
a malfunction is genuinely most dangerous.

### A concrete example: ASIL assignment in practice

Compare two functions in the same vehicle:

| Function | Severity | Exposure | Controllability | Resulting ASIL (illustrative) |
|---|---|---|---|---|
| Airbag deployment decision | S3 (life-threatening) | E4 (every crash) | C3 (driver cannot control) | **ASIL D** |
| Seat-heater temperature display accuracy | S0 (no physical harm) | — | — | **QM** |

The airbag function needs MC/DC-level test rigor and independent test
review; the seat-heater display needs ordinary good testing practice.
Neither needs the other's rigor — applying ASIL D-level cost to every
line of vehicle software would be both economically impossible and a
misallocation of the actual safety effort where it matters.

### The safety case and traceability

ISO 26262 requires a **safety case** — a structured argument, backed by
evidence, that the safety goals for a function have actually been met.
Test results are primary evidence in that argument, which is why:

- Every safety requirement must trace to specific test cases (the same
  traceability matrix idea from Module 8, now with legal/audit weight).
- Test **independence** is required at higher ASILs — the person/team
  writing safety-critical tests should not be the same person who wrote
  the code being tested, specifically to avoid confirmation bias.
- Evidence must be **retained** — safety cases are audited, sometimes
  years after a vehicle ships, including after a field incident.

## How this connects back to the rest of Level 1

| Practice from Modules 1-8 | Standard it satisfies |
|---|---|
| V-model test levels (Module 1) | ASPICE SWE.4/5/6, SYS.4/5 |
| Documented test cases with traceability (Module 8) | ASPICE + ISO 26262 safety case evidence |
| Boundary value analysis (Module 8) | Baseline technique expected at every ASIL, including QM |
| UDS diagnostic testing (Module 6) | Often itself a safety-relevant function (e.g., correctly reporting a real fault) |
| HIL testing (Module 7) | Common evidence source for SYS.4/SYS.5 and safety-case validation |

## Cheat sheet

| Term | Meaning |
|---|---|
| **ASPICE** | Automotive process-maturity standard; OEMs often require a demonstrated capability level |
| **Capability level 0-5** | Incomplete → Performed → Managed → Established → Predictable → Innovating |
| **SWE.4/5/6, SYS.4/5** | The ASPICE processes mapping to unit/integration/system test |
| **ISO 26262** | Functional safety standard for automotive E/E systems |
| **ASIL A-D, QM** | Risk-derived rigor levels; D is highest, QM means no special safety process |
| **S / E / C** | Severity / Exposure / Controllability — the three factors deriving ASIL |
| **MC/DC** | Modified Condition/Decision Coverage — hallmark requirement at ASIL D |
| **Safety case** | The structured, evidence-backed argument that safety goals were met |
| **Test independence** | Higher ASILs require testers separate from the implementers |

## Exercise

A new feature: an automatic emergency braking (AEB) system that applies
the brakes if a forward collision is imminent and the driver hasn't
reacted.

1. Reason through a plausible S/E/C assessment for this function
   (state your reasoning, not just a guess) and propose an ASIL for it.
   Compare it to the airbag example above — is your reasoning consistent
   with why airbags land at ASIL D?
2. Given the ASIL you proposed, state what unit-test coverage criterion
   would be expected, and explain in your own words what MC/DC would
   require beyond simple branch coverage for a boolean decision like
   `if (closingSpeed > threshold && driverBrakeInput == 0 && radarConfidence >= minConfidence)`.
3. Explain why **test independence** matters specifically for this
   feature — describe a plausible way a developer testing their own AEB
   braking-decision code might unconsciously under-test it, that an
   independent tester would be more likely to catch.
4. Name one ASPICE process (from the table above) this feature's
   integration testing with the radar sensor ECU would fall under, and
   explain why.
