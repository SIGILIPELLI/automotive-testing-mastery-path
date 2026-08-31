# 09 · Career Growth in Automotive Test Engineering

Level 4 Module 6 covered building a career ladder from a team lead's
perspective. This module flips the view to the individual: what the
career paths out of automotive test engineering actually look like,
what skills each path leverages from this course, and how to make
deliberate choices rather than drifting.

!!! note "About this module"
    This module is general career guidance based on common automotive
    industry patterns, not a guarantee of any specific outcome — titles,
    ladders, and market conditions vary by company and region.

## The paths, and what they build on

| Path | What it emphasizes | Course foundation it builds on |
|---|---|---|
| Deepen as a test specialist (functional safety, cybersecurity, SOTIF) | Deep expertise in one high-value, scarce-skill area | Level 3 Modules 4, 9; Level 4 Module 4 |
| Test architecture / program leadership | Breadth across strategy, supplier collaboration, team building | Level 4 Modules 1, 6, 7 |
| Move into development (ECU software engineering) | Understanding of requirements/testing informs better-designed, more testable software | The CAPL/testcase design intuition built across all four levels transfers directly to "how would I test this" thinking while writing production code |
| Move into systems engineering | Requirements definition and cross-ECU interaction understanding | Level 2's DBC/restbus work, Level 3's traceability and multi-ECU HIL exposure |
| Tooling/framework specialist | Deep automation/infrastructure skill, often the scarcest role on a team (Level 4 Module 6) | Level 3 Module 5, Level 4 Module 2 |

None of these paths make the others unavailable later — a test
architect who later moves into systems engineering carries genuinely
useful judgment about what's testable and what isn't, which is a
frequently underrated skill in requirements writers who've never had
to test their own requirements.

## Signals that indicate readiness to move up a level

| From | To | Readiness signal |
|---|---|---|
| Test Engineer (Level 1-2 skillset) | Senior Test Engineer (Level 3 skillset) | Independently designs a fault-injection matrix or test strategy for a feature, not just executes handed-down testcases |
| Senior Test Engineer | Test Architect/Lead (Level 4 skillset) | Has navigated a real supplier/OEM disagreement or built a working CI-integrated regression suite end to end, not just contributed to one |
| Individual contributor track | Specialist track (safety/security/SOTIF) | Sought out specifically for judgment calls in one domain, beyond just executing that domain's tests |

These are deliberately behavioral, not tenure-based — years of
experience writing similar CAPL testcases doesn't by itself indicate
readiness for a broader-scope role; demonstrated judgment on
harder, more ambiguous problems does.

## Building a portfolio that demonstrates this course's skills

A concrete artifact set (not just a resume line) that maps to what
this course covered:

```text
- A written test strategy document for a fictional or real feature
  (Level 4 Module 1 style) -- demonstrates strategic thinking, not
  just execution.
- A fault-injection matrix with explicit, honestly-stated gaps
  (Level 3 Module 6 and Module 10's project style) -- demonstrates
  the professional honesty employers specifically look for over a
  suspiciously "complete" plan.
- A small CI-integrated regression suite (even a toy one, Level 3
  Module 7 + Level 4 Module 2) -- demonstrates automation/framework
  capability beyond hand-run testcases.
- A traceability matrix linking requirements to test cases (Level 3
  Module 8) -- demonstrates audit/process maturity awareness.
```

Interviewers in this domain consistently value the *reasoning* behind
a test design (why this fault, why this boundary, what's explicitly
out of scope and why) over a large volume of testcases with no
visible design rationale — the exercises throughout this course have
been asking for exactly that reasoning for a reason.

## Certifications and external credentials worth knowing about

| Credential | Relevance |
|---|---|
| ISTQB (foundation and specialist automotive extension) | General test-process vocabulary and recognition, useful early-career |
| ASPICE assessor training | Valuable for test architect/lead track, given how much of Level 4 touches process maturity |
| ISO 26262 / functional safety specific training | Directly relevant for the safety-specialist track (Level 3 Module 4) |
| Vendor-specific tool certifications (CANoe, CANape) | Practical, often directly requested in job postings, complements but doesn't replace the conceptual depth this course builds |

None of these substitute for demonstrated, reasoned work — they
signal baseline vocabulary and can open interview doors, but the
portfolio items above are what actually differentiate in a technical
interview for this field.

## Cheat sheet

| Concept | Key point |
|---|---|
| Multiple paths | Specialist, architect/leadership, development, systems engineering, tooling — none preclude the others later |
| Readiness signals | Behavioral (independent judgment on ambiguous problems), not tenure-based |
| Portfolio over resume lines | Strategy documents, honest gap-stated fault matrices, working CI regression suites, traceability matrices |
| Certifications | Useful door-openers, not substitutes for demonstrated reasoning |

## Exercise

1. Pick one of the five paths in the table and write a concrete
   6-month plan (specific projects or credentials, not vague goals)
   to move toward it from a Level 1-2 skillset today.
2. Using the Level 3 Module 10 project (HIL test plan for AEB) as a
   base, describe what you'd add or change to turn it into a portfolio
   artifact specifically aimed at demonstrating readiness for a Test
   Architect role.
3. A colleague argues certifications alone (without portfolio
   projects) are sufficient to move into a specialist safety-testing
   role. Using the readiness-signal table, construct the counter-
   argument for why demonstrated judgment matters more, and what
   specific evidence you'd ask them to produce instead.
