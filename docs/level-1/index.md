# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand what makes automotive ECU testing different from general
software testing, get fluent in CAN bus fundamentals and frame structure,
understand what Vector CANoe is for and how CAPL scripts drive it, learn
the shape of UDS diagnostics and Hardware-in-the-Loop (HIL) testing, apply
solid test-case design to ECU signals, and know where the industry's
standards (ASPICE, ISO 26262) touch testing — then put it together by
designing a CAPL test script for a simulated ECU signal.

!!! note "About this track"
    Vector CANoe, CAPL, and real HIL test benches are specialized
    commercial tooling and physical hardware — they cannot be installed
    or exercised on a general-purpose machine the way a compiler or
    interpreter can. Every lesson here is written and checked for
    technical accuracy against genuinely documented CAN, CAPL, and UDS
    (ISO 14229) conventions and real Vector tooling behavior, rather than
    against live execution. Where an example shows CAPL code or a CANoe
    workflow, treat it as a faithful worked example to study and adapt —
    not a transcript of a script that was actually run.

## Modules

1. [What Is Automotive Testing?](01-what-is-automotive-testing.md)
2. [CAN Bus Fundamentals](02-can-bus-fundamentals.md)
3. [CAN Frame Structure](03-can-frame-structure.md)
4. [Introduction to Vector CANoe](04-introduction-to-canoe.md)
5. [CAPL Scripting Basics](05-capl-scripting-basics.md)
6. [UDS Diagnostics Basics](06-uds-diagnostics-basics.md)
7. [Introduction to HIL Testing](07-introduction-to-hil-testing.md)
8. [Test Case Design for ECUs](08-test-case-design-for-ecus.md)
9. [Automotive Test Standards Overview](09-automotive-test-standards-overview.md)
10. [Project — CAPL Test Script for a Simulated ECU Signal](10-project-capl-test-script.md)

By the end of this level you'll be able to explain how automotive testing
maps onto the V-model, read and reason about CAN frames and arbitration,
describe what CANoe does and why teams use it, write basic CAPL event
handlers that send and receive CAN messages, describe UDS request/response
diagnostics, explain what HIL testing is and why it exists, design test
cases for ECU signals using equivalence partitioning and boundary
analysis, and connect ASPICE/ISO 26262 to concrete testing obligations.

New to CAN from the embedded-firmware side? The
[S32K Automotive Embedded Mastery Path](https://sigilipelli.github.io/s32k-skillmastery/)
covers the CAN peripheral and ECU firmware itself. New to software testing
in general? The
[C/C++ Testing Mastery Path](https://sigilipelli.github.io/cpp-testing-skillmastery/)
covers universal test methodology in depth.
