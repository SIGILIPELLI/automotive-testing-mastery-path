---
description: "Building a Test Automation Team/Practice — Every technical module so far assumed someone capable was already writing the CAPL, building the framework…"
---

# 06 · Building a Test Automation Team/Practice

Every technical module so far assumed someone capable was already
writing the CAPL, building the framework, maintaining the traceability
matrix. This module addresses the organizational question directly:
how do you build and run the team that does this work sustainably, at
program scale, without any one person becoming a single point of
failure.

!!! note "About this module"
    This content is organizational/practice guidance based on common
    industry team-structure patterns, not a specific company's
    documented process — adapt roles and ratios to your organization's
    actual size and constraints.

## Roles in a mature test automation practice

| Role | Primary responsibility | Relationship to earlier modules |
|---|---|---|
| Test engineer | Writes and maintains CAPL testcases, executes and triages results | Levels 1–3 core skillset |
| Automation/framework engineer | Builds and maintains the orchestration layer, CI integration, reporting | Level 3 Module 5, Level 4 Module 2 |
| Test architect / lead | Owns the test strategy, traceability model, risk-based allocation | Level 3 Module 8, Level 4 Module 1 |
| Rig/infrastructure engineer | Maintains physical HIL rigs, fault-injection hardware, calibration of test equipment itself | Level 3 Modules 1, 6 |
| Safety/security test specialist | Owns ASIL/CAL-specific rigor, independent review requirements | Level 3 Module 4, Level 4 Module 4 |

Smaller teams collapse several of these into one person; the point of
naming them separately is that each is a genuinely distinct skill and
workload, and a team that never explicitly assigns the "framework
engineer" or "rig engineer" work often ends up with it done poorly,
ad hoc, by whoever happens to be blocked by its absence that week.

## The bus-factor problem, specifically in this domain

Automotive test automation has a sharper bus-factor risk than typical
software teams: CAPL expertise, specific rig wiring knowledge, and
supplier-specific tooling quirks are less commonly held skills than
general-purpose programming, and a program can genuinely stall if the
one person who understands a particular rig's fault-injection wiring
leaves.

| Mitigation | How it works |
|---|---|
| Pair test-case authorship on ASIL C/D work | Forces knowledge transfer as a side effect of the independent-review requirement (Level 3 Module 4) already mandated for high-ASIL work |
| Document rig wiring/configuration as code, not tribal knowledge | A rig's fault-injection relay mapping, HIL parameter names, and physical pin assignments belong in a versioned config file (Level 4 Module 5's baseline discipline), not one engineer's notebook |
| Rotate framework/infrastructure ownership | Even a lightweight rotation (a different engineer owns CI pipeline health each quarter) spreads familiarity beyond one person |
| Onboarding path through the course's own level structure | A new engineer following Levels 1→4 in order builds the same layered mental model this course teaches, rather than being thrown directly at Level 4 problems |

## Career progression as a retention tool

A test automation practice that has no visible growth path beyond
"senior test engineer" loses people to development or systems roles
that appear to offer more. A concrete ladder, mapped to this course's
own structure, gives a legible path:

```text
Level 1-2 skillset  -> Test Engineer I/II       (executes, maintains testcases)
Level 3 skillset     -> Senior Test Engineer     (designs test strategy for a
                                                   feature/ECU, owns fault
                                                   injection matrices)
Level 4 skillset     -> Test Architect / Lead     (owns program-level strategy,
                                                    supplier collaboration,
                                                    team practice)
```

Module 9 covers the individual career-growth side of this ladder in
more depth; this module's point is that a team lead should design the
ladder deliberately, not let it emerge accidentally from whoever has
been there longest.

## Metrics for a healthy practice (beyond suite health)

Level 3 Module 7 covered suite-health metrics (pass rate, flake rate).
A team-health view needs a complementary set:

| Metric | What it reveals |
|---|---|
| Bus-factor count per critical rig/tool | How many people could maintain this if the primary owner left tomorrow? |
| Onboarding time to first independent CAPL testcase | A rising trend suggests documentation or tooling has decayed |
| Ratio of new test-case authorship to defect-fix rework | A team spending most of its time fixing flaky/broken existing tests rather than adding coverage is signaling suite or process debt (Level 3 Module 7) |
| Cross-training coverage (rig × people matrix) | A simple grid — which people can independently operate which rigs — directly shows single-point-of-failure risk |

## A minimal cross-training matrix

```text
                RigA(ABS)  RigB(AEB)  RigC(Ethernet/SOME-IP)
Engineer 1         Yes        Yes           No
Engineer 2         Yes        No            No
Engineer 3         No         No            Yes      <- single point of failure
Engineer 4 (new)   Training    -             -
```

Engineer 3 being the only person who can operate RigC is a visible,
actionable risk the moment this matrix exists — the value isn't the
spreadsheet itself, it's that the risk stops being invisible.

## Cheat sheet

| Concept | Key point |
|---|---|
| Distinct roles | Test engineer, framework engineer, rig engineer, architect, safety/security specialist — name them even on a small team |
| Bus factor | Sharper in this domain due to specialized CAPL/rig knowledge — mitigate with pairing, documentation-as-code, rotation |
| Career ladder | Map directly to this course's Level 1→4 skill progression to make growth legible |
| Team-health metrics | Bus-factor count, onboarding time, new-vs-rework ratio, cross-training matrix |

## How It Actually Works

**Why "rig wiring as code" is a specific, checkable artifact, not a
metaphor.** A HIL rig's fault-injection capability (Level 3 Module 6)
depends on a physical mapping: relay channel 7 on the fault-injection
matrix is wired to the ForwardDistance_m sensor line's power rail,
relay channel 12 to its signal line, and so on. When that mapping only
exists as a hand-drawn wiring diagram or one engineer's memory, a new
CAPL testcase written against the wrong assumed channel number will
run without any error — `InjectFault(channel=7)` executes successfully
regardless of what it's actually wired to — and silently produce a
result for the wrong fault entirely. Turning that mapping into a
versioned config file (a YAML/JSON table of `signal_name ->
relay_channel`, checked into the same repo as the baseline in Module
5) makes it something a test framework can load and validate
programmatically: the orchestration layer can assert the config file's
channel-to-signal mapping matches what the rig controller itself
reports as wired, at job setup time, turning a silent, undetectable
wrong-fault bug into a loud pre-flight failure.

**Why pairing on ASIL C/D work transfers knowledge as a side effect,
mechanically.** ISO 26262-8's independent review requirement for
high-ASIL work mandates that someone other than the author confirms
the test's adequacy before it counts as a valid confirmation measure.
The mechanical knowledge-transfer effect comes from what that review
actually has to inspect to be a real review rather than a rubber
stamp: the reviewer has to trace the CAPL testcase back to its
requirement ID, understand which fault-injection channels and rig
capabilities it depends on, and judge whether the assertions actually
prove the safety mechanism works — which requires the reviewer to
build enough of the same rig/requirement/CAPL mental model the author
has. A team that treats this review as a formality (a quick glance and
a sign-off) gets the compliance checkbox but none of the bus-factor
benefit; a team that treats it as a genuine technical walkthrough gets
both, for the same mandated review step.

**Why the cross-training matrix is a leading indicator, not a lagging
one.** Suite-health metrics (pass rate, flake rate, from Level 3
Module 7) tell you about test quality after the fact — they can't warn
you a rig is one resignation away from being unusable. The
cross-training matrix is structurally different: it measures
*capability distribution*, which changes slowly and predictably (an
engineer leaving is usually knowable weeks or months in advance, unlike
a sudden flaky-test regression), which is exactly why it functions as
an early-warning signal a team can act on — reassign RigC time to pair
Engineer 4 with Engineer 3 — before the single point of failure
actually manifests as a program-stalling gap, rather than after.

## Exercise

1. Using the cross-training matrix template, identify what concrete
   action (not just "train someone else") you'd take first for
   Engineer 3's single-point-of-failure risk on RigC, considering
   Engineer 4 is still new.
2. A team reports high suite pass rates but a rising "new test-case
   authorship vs. defect-fix rework" ratio trending toward mostly
   rework. Diagnose two plausible root causes and what Level 3 Module
   7 practice would address each.
3. Design the onboarding path for a new hire with strong general
   software skills but no CAPL/automotive background, mapped to this
   course's own Level 1→4 structure, with a rough timeline per level.
