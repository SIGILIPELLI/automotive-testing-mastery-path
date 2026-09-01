# 01 · What Is Automotive Testing?

Automotive testing is software and systems testing applied to **electronic
control units (ECUs)** — the 70-150+ small computers in a modern vehicle
that manage everything from the engine and brakes to the infotainment
system and power windows. Every general testing principle you might
already know still applies (see the
[C/C++ Testing Mastery Path](https://sigilipelli.github.io/cpp-testing-skillmastery/)
for those fundamentals) — but automotive adds three things that change
how testing is actually done: a network of ECUs talking over shared
buses, hardware in the loop, and safety standards with legal teeth.

## Why automotive testing is its own discipline

| General software testing | Automotive ECU testing |
|---|---|
| Runs on the machine you built it for | Target is often a different CPU architecture, no OS, no filesystem |
| One program, mostly self-contained | Dozens of ECUs cooperating over CAN/LIN/Ethernet — a bug can live in the *interaction* |
| A failure is usually recoverable (crash, retry) | A failure can mean a stuck throttle, a brake that doesn't respond, or a door that unlocks at speed |
| Regulatory pressure is often light | ISO 26262 (functional safety) and ISO 21434 (cybersecurity) are legally binding in most markets |
| Test environment = your laptop | Test environment = simulation → bench → Hardware-in-the-Loop (HIL) → real vehicle, each stage adding realism and cost |

The consequence: an automotive tester needs to think in terms of
**messages on a bus**, not just function calls, and needs tools built for
that world. This course teaches those tools — CAN, Vector CANoe, CAPL,
UDS diagnostics, and HIL — from the ground up.

## The V-model

Automotive software development almost universally follows some version
of the **V-model**: development activities on the left side, matched by
increasingly integrated test activities on the right side, connected by
horizontal arrows that say "this test level verifies that development
activity."

```text
Requirements  \                                      / Acceptance Test
               \  System Design                      / System Test
                \    Architecture Design            / Integration Test
                 \      Detailed Design            / Unit Test
                  \________________________________/
                         Implementation (code)
```

Read top to bottom on the left, then bottom to top on the right:

| Development activity (left) | Verifies against | Test level (right) |
|---|---|---|
| Requirements (what the vehicle/feature must do) | — | **Acceptance test** — does the finished vehicle/feature satisfy the original requirement? |
| System design (how ECUs and features cooperate) | System requirements | **System test** — the whole vehicle network, exercised end-to-end |
| Architecture design (how one ECU's software is structured — modules, interfaces) | Architecture spec | **Integration test** — do modules/ECUs work correctly *together*? |
| Detailed design (algorithms, data structures inside one module) | Detailed design spec | **Unit test** — does one function/module do what its own spec says? |
| Implementation | — | (code is written here, at the bottom of the V) |

The point of drawing it as a V, not a straight line, is that **each test
level has a specific, pre-written thing it is checking against** — you
don't invent what "correct" means at test time, you inherit it from the
matching left-side artifact. This is exactly the "test against
expectations" principle from general testing, made concrete for a
multi-ECU, standards-driven world.

### The four levels, in automotive terms

**Unit test** — one function or module, in isolation, usually with the
rest of the ECU's code stubbed out. Example: a function that converts a
raw ADC reading into a temperature in °C. Runs on a PC (host testing) or
on the real target; automotive teams commonly require **statement and
branch coverage** here, sometimes MC/DC for the highest safety levels
(more in Module 9).

**Integration test** — modules inside one ECU wired together, or several
ECUs wired together over their real buses. Example: does the engine
ECU's "requested torque" CAN message actually get correctly interpreted
by the transmission ECU? This is where CAN/CAPL tooling (Modules 2-5)
becomes the primary testing instrument, because the thing under test
*is* the message traffic.

**System test** — the vehicle network as a whole, or a full HIL rig
standing in for it (Module 7), exercised through realistic scenarios:
key-on sequences, drive cycles, fault injection (disconnect a sensor —
does the system degrade safely?).

**Acceptance test** — does the finished vehicle/feature satisfy what the
OEM (the vehicle manufacturer) or the end customer actually asked for?
Often includes homologation and regulatory sign-off (Level 4 covers this).

!!! tip "Verification vs. validation, restated for a vehicle"
    **Verification**: "did we build the ECU right?" — does the brake ECU's
    software match its own detailed design spec. **Validation**: "did we
    build the right vehicle?" — does the braking *feel* right and meet the
    actual safety requirement, which the detailed design spec might have
    gotten subtly wrong. A unit test is verification. A test-track brake
    evaluation is validation. Both matter; they catch different classes of
    problem.

## Where this course's tools fit in the V

| Tool / topic | Test level it primarily serves |
|---|---|
| CAN fundamentals & frame structure (Modules 2-3) | Foundation for integration and system test |
| Vector CANoe & CAPL (Modules 4-5) | Integration test — simulating missing ECUs, injecting/monitoring CAN traffic |
| UDS diagnostics (Module 6) | Integration and system test — every ECU must respond correctly to diagnostic requests |
| HIL testing (Module 7) | System test — a full electrical/network environment without the physical vehicle |
| Test case design for ECUs (Module 8) | All levels — equivalence partitioning and boundary analysis on signal ranges |
| ASPICE / ISO 26262 (Module 9) | Governs *how rigorously* every level above must be done |

## A worked example: one requirement through the V

Take a concrete requirement for a body control module (BCM):

> **REQ-BCM-014**: When vehicle speed exceeds 10 km/h, the BCM shall lock
> all doors within 500 ms of the speed threshold being crossed, by
> transmitting a lock command on the door-lock actuator CAN message.

- **Unit test**: the function that decides "should I command a lock?"
  given a speed input — call it with 9.9, 10.0, 10.1 km/h and check the
  boolean result (this is boundary analysis, Module 8).
- **Integration test**: does the BCM's CAN message actually reach the
  door actuator ECUs, and do they respond by locking? A CANoe test
  module (Module 4) can simulate the speed sensor's CAN message and
  watch for the lock command with a timer.
- **System test**: on a HIL rig or real vehicle, physically accelerate
  past 10 km/h and confirm the doors lock within 500 ms, including under
  degraded conditions (a noisy CAN bus, a slow-responding ECU).
- **Acceptance test**: does this actually satisfy the OEM's real safety
  intent — is 500 ms fast enough, and does the *feature* behave sensibly
  (no false locks at exactly 10 km/h during normal driving)?

Every module from here builds toward being able to execute the middle two
rows of that example for real.

## Cheat sheet

| Term | Meaning |
|---|---|
| **ECU** | Electronic Control Unit — one of many small computers in a vehicle |
| **V-model** | Development activity on the left, matched test level on the right |
| **Unit test** | One function/module in isolation |
| **Integration test** | Modules or ECUs working together, often over CAN |
| **System test** | The whole vehicle network / a HIL rig, end-to-end |
| **Acceptance test** | Does the finished vehicle/feature meet the original ask |
| **Verification** | Built it right? (matches its own spec) |
| **Validation** | Built the right thing? (matches real intent) |
| **OEM** | Original Equipment Manufacturer — the vehicle brand |
| **ASPICE** | Automotive SPICE — a process-maturity standard (Module 9) |
| **ISO 26262** | The automotive functional-safety standard (Module 9) |

## Exercise

Take this requirement for a driver-side power window:

> **REQ-PW-007**: If an obstruction is detected while the window is
> closing (measured by a rise in motor current above 8 A sustained for
> more than 50 ms), the window shall stop and reverse by 100 ms of
> down-travel within 200 ms of detection.

Write down, for each of the four V-model levels (unit, integration,
system, acceptance):

1. **One concrete test** you would run at that level, stated precisely
   enough that two different people would agree whether it passed.
2. **What tool or environment** you would need to run it (a bench PC, a
   CANoe setup with a simulated motor-current signal, a HIL rig, a real
   vehicle on a test track, etc.) — you don't need to know the tools in
   detail yet, just reason about what class of environment each level
   needs.
3. One sentence distinguishing whether your system-level test is really
   **verification** or **validation** of REQ-PW-007, and why.

Keep your answers — Module 8 will have you turn the unit-level case into
a fully documented test case with boundary values.
