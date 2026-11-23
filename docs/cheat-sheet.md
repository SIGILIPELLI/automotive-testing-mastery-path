---
title: "Automotive Testing Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Automotive Testing cheat sheet: 39 quick-reference tables from every level of the Automotive Testing Mastery Path, each linked to its full lesson."
---

# Automotive Testing Cheat Sheet

Quick-reference tables from **39 lessons** of the free Automotive Testing Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Is Automotive Testing?](level-1/01-what-is-automotive-testing.md)

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

### [02 · CAN Bus Fundamentals](level-1/02-can-bus-fundamentals.md)

| Item | Notes |
|---|---|
| Physical layer | Twisted pair CAN_H/CAN_L, differential, 120 Ω termination at both ends |
| Dominant / recessive | Dominant (0) always wins on the wire — the basis for arbitration and ACK |
| No addresses | Frames carry a message ID describing content, not a destination |
| Every node sees every frame | Filtering happens at the receiver, not the network |
| Classic CAN speed | Up to 1 Mbit/s; 500 kbit/s is the most common vehicle rate |
| Common vehicle rates | 125 kbit/s (body), 500 kbit/s (powertrain/chassis), 1 Mbit/s (diagnostic) |
| ACK | Any node with a valid CRC drives dominant — not proof the *intended* consumer got it |
| Error counters | TX errors cost more than RX errors; both decrement on success |
| Error active → passive → bus-off | Escalating self-isolation of a faulty node |
| Common testing fault sources | Missing/duplicate termination, mismatched bit rate, long stubs, shorted/open bus wires |

### [03 · CAN Frame Structure](level-1/03-can-frame-structure.md)

| Item | Notes |
|---|---|
| Frame skeleton | SOF · ID(+RTR/IDE) · DLC · Data(0-8B) · CRC · ACK · EOF |
| Standard ID | 11 bits, 0x000-0x7FF |
| Extended ID | 29 bits, 0x00000000-0x1FFFFFFF (used by e.g. J1939) |
| DLC | 4-bit field, 0-8 data bytes on classic CAN |
| Arbitration | Bitwise, non-destructive; **lowest numeric ID always wins** |
| ID = priority | Safety/critical messages get low IDs by design, not accident |
| ACK slot | Transmitter sends recessive; any valid-CRC receiver drives it dominant |
| ACK proves | *Someone* got the frame — never proof the intended consumer did |
| No ACK at all | Strong signal of a disconnected bus or transceiver fault |
| Error flag | 6 dominant bits, destroys the frame for everyone, triggers automatic retry |
| Bit stuffing | One opposite bit inserted after 5 identical bits, to preserve sync |

### [04 · Introduction to Vector CANoe](level-1/04-introduction-to-canoe.md)

| Item | Notes |
|---|---|
| Vendor | Vector Informatik |
| Core roles | Simulation, analysis, stimulation/test execution |
| Configuration file | `.cfg` — ties network setup, database, CAPL/test logic together |
| Database | DBC or ARXML — decodes raw bytes into named signals |
| Trace window | Live timestamped log of bus traffic — the primary analysis view |
| Restbus simulation | Auto-generated simulation of every ECU except the DUT |
| Test module | Structured, automated, pass/fail test case |
| Hardware interface | VN16xx/VN50xx series — does the actual electrical bus connection |
| Scripting language | CAPL — C-like, event-driven (Module 5) |
| Portability | Same config can run against bench hardware, HIL, or pure offline simulation |

### [05 · CAPL Scripting Basics](level-1/05-capl-scripting-basics.md)

| Element | Purpose |
|---|---|
| `variables { }` | Declares globals for the whole CAPL program |
| `on start` | Runs once when the measurement begins |
| `on message <Name>` | Runs every time that message is received; `this` refers to it |
| `on timer <name>` | Runs once when a named `msTimer` expires |
| `output(msg)` | Transmits a message immediately |
| `setTimer(t, ms)` | Arms/re-arms a timer for `ms` milliseconds from now |
| `cancelTimer(t)` | Stops a pending timer before it fires |
| `write("...", args)` | Prints to the CANoe Write window — the basic debugging tool |
| `message <Type> var` | Declares a variable typed as a specific database message |
| Alive counter pattern | Increment and wrap (`% N`) each cycle so receivers detect a frozen sender |
| Cyclic sender pattern | `on timer` handler re-arms itself with `setTimer()` at the end |
| Timeout/watchdog pattern | `on message` cancels+rearms; `on timer` fires only on real silence |

### [06 · UDS Diagnostics Basics](level-1/06-uds-diagnostics-basics.md)

| Item | Notes |
|---|---|
| Standard | ISO 14229 (UDS) |
| Model | Strict client (tester) / server (ECU) request-response |
| Positive response | SID + 0x40 |
| Negative response | `7F`, echoed SID, NRC |
| `0x10` | Diagnostic Session Control |
| `0x11` | ECU Reset |
| `0x14` | Clear Diagnostic Information |
| `0x19` | Read DTC Information |
| `0x22` | Read Data By Identifier |
| `0x27` | Security Access (seed/key) |
| `0x2E` | Write Data By Identifier |
| `0x31` | Routine Control |
| `0x3E` | Tester Present (keeps a session alive) |
| NRC `0x78` | "Still working" — a test must keep waiting, not fail |
| NRC `0x33`/`0x35` | Security access denied / invalid key |
| Scaling of DID values | Project-specific, not defined by UDS itself |

### [07 · Introduction to HIL Testing](level-1/07-introduction-to-hil-testing.md)

| Term | Meaning |
|---|---|
| **HIL** | Hardware-in-the-Loop: real ECU + simulated environment |
| **Plant model** | The real-time physics/system simulation (engine, vehicle dynamics, etc.) |
| **DUT** | Device Under Test — the real ECU connected to the rig |
| **I/O interface** | Hardware converting simulated values to real electrical signals |
| **Load box** | Simulates realistic electrical load for actuator outputs |
| **FIU (Fault Insertion Unit)** | Switching matrix for injecting shorts, opens, resistance |
| **Real-time step** | The fixed, guaranteed-deadline cycle time of the simulation |
| **Open-loop test** | Pre-scripted inputs, ECU output doesn't affect the simulation |
| **Closed-loop test** | ECU outputs feed back into the simulation live |
| Why HIL matters | Vehicle-realistic, fully repeatable, safe to fault-inject, cheaper/faster than real vehicles |

### [08 · Test Case Design for ECUs](level-1/08-test-case-design-for-ecus.md)

| Technique | What it does | Automotive application |
|---|---|---|
| Equivalence partitioning | Group inputs that should behave identically | Partition a signal's physical range into valid/invalid/behaviorally distinct bands |
| Boundary value analysis | Test exactly at and adjacent to partition edges | Threshold-triggered warnings, DTC set/clear points, range limits |
| Bus-level test data | Specify exact raw bytes or signal+scale | Never just "send 120 °C" — always the actual frame/DID content |
| Timing in expected results | State the allowed latency, not just the final value | ECU reactions are rarely instantaneous — spec the deadline |
| Hysteresis check | Test both the "on" and "off" thresholds separately | Prevents missing chatter/flicker defects at a boundary |
| Environment field | Record ECU build, database version, CANoe config version | A stale DBC or wrong ECU build is a very common false failure |

### [09 · Automotive Test Standards Overview](level-1/09-automotive-test-standards-overview.md)

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

## Level 2 · Intermediate

### [01 · Advanced CAPL Scripting](level-2/01-advanced-capl-scripting.md)

| Element | Purpose |
|---|---|
| `returnType name(params) { }` | User-defined function, C-like calling convention |
| `byte buf[N];` | Fixed-size array; no dynamic memory in CAPL |
| `this.byte(i)` | Raw payload byte access, independent of database signals |
| `@Namespace::Var` | Read/write a system variable |
| `on sysvar_update X` | Fires when sysvar `X` changes, from any source |
| `elcount(arr)` | Declared element count of an array — use instead of literals |
| `snprintf(buf, elcount(buf), fmt, ...)` | Safe formatted string building |
| `testcase`, `testStepPass/Fail` | Entry points for CANoe test modules (Module 2) |

### [02 · CANoe Test Modules &amp; Test Cases](level-2/02-canoe-test-modules.md)

| Element | Purpose |
|---|---|
| `testcase name() { }` | One independently reportable test unit |
| `testCaseTitle("...")` | Human-readable title shown in the report |
| `testStep("...")` | Documents/labels one check within a test case |
| `testStepPass("...")`, `testStepFail("...")` | Record a step's verdict |
| `testWaitForTimeout(ms)` | Blocks until a timeout, used to bound how long a check waits |
| Pass / Fail / Inconclusive / Not run | The four verdict states, not interchangeable |
| Report (HTML/XML) | Structured, machine-consumable output of a test module run |

### [03 · CAN-FD Basics](level-2/03-can-fd-basics.md)

| Concept | Classic CAN | CAN-FD |
|---|---|---|
| Payload | 8 bytes | up to 64 bytes |
| Bit rate | single rate | dual: arbitration + data phase |
| New signaling bits | none | FDF, BRS, ESI |
| DLC 9–15 | not used | maps to 12/16/20/24/32/48/64 bytes |
| DBC tagging | implicit | `BA_ "VFrameFormat"` attribute |
| CAPL access | `msg.byte(n)`, `msg.dlc` | adds `msg.fdf`, `msg.brs`, `msg.esi` |

### [04 · LIN Bus Basics](level-2/04-lin-bus-basics.md)

| Concept | CAN | LIN |
|---|---|---|
| Bus access | Multi-master, arbitration | Single master, schedule-driven |
| Physical layer | 2-wire differential | 1-wire, ground-referenced |
| Frame structure | ID + DLC + data + CRC | Break/Sync/PID (master) + data + checksum (header owner) |
| Database file | DBC | LDF |
| Checksum types | Single (15-bit CRC) | Classic vs. Enhanced (version-dependent) |
| CAPL object | `message` | `linFrame` |

### [05 · Diagnostic Trouble Codes (DTCs) Deep Dive](level-2/05-dtc-deep-dive.md)

| Concept | Key point |
|---|---|
| Status byte | 8 independent bits, not a pass/fail flag |
| Pending vs. confirmed | Pending = failed once; confirmed = met maturation criteria |
| MIL off ≠ DTC cleared | Lamp can go off after N good cycles while the DTC stays stored |
| `0x19` | ReadDTCInformation — always specify the right status mask/subfunction |
| `0x14` | ClearDiagnosticInformation — test clearing under an active fault, not just a resolved one |
| Snapshot data | Freeze-frame values must reflect the actual fault moment, not stale cache |

### [06 · Database Files (DBC) &amp; Signal Definitions](level-2/06-dbc-signal-definitions.md)

| DBC element | Test relevance |
|---|---|
| `SG_` start bit + length + byte order | Must match exactly to decode/encode a signal correctly |
| Scale/offset | physical = raw × scale + offset — derive test vectors from this |
| `[min\|max]` | Drives plausibility/boundary test partitions |
| `VAL_` | Enumerated states — verify every raw value is either mapped or explicitly "reserved" |
| `CM_` | Often the only place real update-timing/precondition info lives |
| Bit-map overlap | Not caught by value-only tests — needs an explicit layout review |

### [07 · Automated Test Sequences in CANoe](level-2/07-automated-test-sequences.md)

| Construct | Runs | Purpose |
|---|---|---|
| `testpreparation` | Once, before all testcases | Shared setup |
| `testcase` | Once each, order-independent | The actual check |
| `testcasefinalization` | Once, after all testcases | Shared teardown |
| `testStepCheck` | Per assertion | Condition-driven pass/fail |
| `testCaseInconclusive` | On unmet preconditions | Keeps skipped tests out of pass/fail stats |
| Fail-fast vs. fail-forward | Test Configuration level | Smoke vs. full regression strategy |

### [08 · Restbus Simulation](level-2/08-restbus-simulation.md)

| Concept | Key point |
|---|---|
| Restbus purpose | Stand in for every node not physically under test |
| DBC cycle-time attribute | Basis for auto-generated baseline restbus |
| Meaningful vs. default values | Auto-generated restbus often needs enrichment for real test scenarios |
| `export`ed control functions | Decouple restbus timing logic from test scenario logic |
| Timing fidelity | Critical specifically for timeout/failsafe tests — verify via trace |
| Restbus vs. real hardware | Restbus for fast/frequent logic tests; real ECUs for integration fidelity |

### [09 · Basic HIL Test Bench Components](level-2/09-hil-test-bench-components.md)

| Component | Analogy to pure simulation |
|---|---|
| Real-time simulation computer | Replaces "the rest of the world exists only as CAPL logic" with actual physics |
| I/O interface hardware | New — pure simulation has no real electrical signals at all |
| ECU under test | The one thing that was previously *also* simulated, now real |
| Bus interface | Same CANoe/CAPL concepts as Modules 1–8, now on a real transceiver |
| Fault-injection unit | New — real electrical faults, not simulated bad values |
| Closed loop | The defining property distinguishing HIL from open-loop bench testing |

### [10 · Project — Automated CAN Signal Validation Suite](level-2/10-project-can-signal-validation.md)

| Module | Concept exercised here |
|---|---|
| 3 — CAN-FD | DLC/length assertion on `SensorFusionFrame` |
| 5 — DTCs | Two-cycle maturation test via `0x19` status mask |
| 6 — DBC | Range assertions derived directly from `[min\|max]` fields |
| 7 — Test sequences | `testpreparation`/`testcasefinalization` structure, independent testcases |
| 8 — Restbus | `export`ed `SetCoolantTemp`/`SetEngineRPM` scenario control |

## Level 3 · Advanced

### [01 · Advanced HIL Test Automation](level-3/01-advanced-hil-automation.md)

| Concept | Key point |
|---|---|
| Two-layer split | Real-time plant model vs. non-real-time orchestration |
| Parameter/signal interface | Orchestration never touches model internals directly |
| Scenario parameterization | One testcase body, many representative operating points |
| Settle waits | Always wait for transients before asserting or starting the next scenario |
| Loop-closure sanity check | A standing testcase proving the rig is actually closed-loop |

### [02 · CANape &amp; Measurement/Calibration Basics](level-3/02-canape-measurement-calibration.md)

| Concept | Key point |
|---|---|
| A2L file | Maps symbol name → address/type/scaling; must match the exact ECU software build |
| MEASUREMENT | Read-only live variable, safe to observe |
| CHARACTERISTIC | Writable tunable value — always restore after use |
| DAQ list | Configure once, stream many samples — avoid per-address polling |
| XCP master/slave | Tool is master, ECU is slave; CONNECT/DAQ/DISCONNECT lifecycle |

### [03 · Ethernet/Automotive Ethernet Basics](level-3/03-automotive-ethernet-basics.md)

| Concept | Key point |
|---|---|
| 100BASE-T1 | Single-pair physical layer for automotive Ethernet; same frame format above it |
| SOME/IP | Service-oriented middleware: services, methods, events, eventgroups |
| SOME/IP-SD | Discovery/offer/subscribe sub-protocol — verify separately from eventing |
| No built-in reliability | SOME/IP over UDP can silently drop events — test for loss explicitly if it matters |

### [04 · Functional Safety Testing (ISO 26262)](level-3/04-functional-safety-testing.md)

| Concept | Key point |
|---|---|
| ASIL | Attaches to a safety goal (via S/E/C), not to a whole ECU; drives required test rigor |
| Safety mechanism | The specific design element that detects a fault and forces a safe reaction — the actual test target |
| Recovery testing | Test fault-clear behavior explicitly, not just fault-detect |
| Single-point vs. latent fault | Latent faults need periodic self-test evidence, not just one-shot fault injection |
| Traceability | Every safety test needs a requirement ID and its ASIL recorded |

### [05 · Test Automation Frameworks for ECU Testing](level-3/05-test-automation-frameworks.md)

| Concept | Key point |
|---|---|
| Execution vs. orchestration | CAPL/CANoe executes in real time; Python/COM orchestrates across tools and variants |
| COM interface | The standard way to drive CANoe programmatically from outside CAPL |
| Data-driven testing | Keep scenario data in external files, not hardcoded in test logic |
| JUnit XML | Common bridge from rig-level verdicts to CI dashboards |

### [06 · Fault Injection Testing](level-3/06-fault-injection-testing.md)

| Concept | Key point |
|---|---|
| Fault taxonomy | Signal, communication, electrical, power, software/timing — each needs different tooling |
| CAPL's ceiling | Signal and much communication-level faults; not electrical or power-level |
| Restore after injecting | Every fault injection needs an explicit undo before the next testcase |
| Fault matrix | Track signal × fault-type coverage explicitly, tied to the FMEA |

### [07 · Regression Test Suites for Automotive Software](level-3/07-regression-test-suites.md)

| Concept | Key point |
|---|---|
| Tiering | Smoke/functional/full-regression, matched to how often each can run |
| Independence | Every testcase must reset its own preconditions, not rely on run order |
| Flaky tests | Triage, don't reflexively silence — may be a real intermittent bug |
| Failure triage | Group by root cause before fixing individually |
| Suite health metrics | Pass rate, flake rate, runtime trend, test-to-feature ratio |

### [08 · Requirements-Based Test Traceability](level-3/08-requirements-based-traceability.md)

| Concept | Key point |
|---|---|
| Forward/backward traceability | Requirement → tests AND test → requirement, both queryable |
| Coverage ≠ existence | A passing test that only covers the nominal case isn't full coverage of a requirement with boundary conditions |
| Tag testcases explicitly | Embed requirement IDs in titles/metadata so coverage matrices can be generated, not hand-maintained |
| Stale links | The most common real failure mode — requirement changed, test didn't |
| Audit readiness | The traceability artifact itself is often literally what an assessor asks to see |

### [09 · SOTIF &amp; Testing for Autonomous Features](level-3/09-sotif-autonomous-testing.md)

| Concept | Key point |
|---|---|
| SOTIF vs. functional safety | SOTIF addresses hazards from insufficient performance with no malfunction, not from failures |
| Four scenario areas | Known-safe, known-unsafe, unknown-unsafe, unknown-safe — the goal is shrinking unknown-unsafe |
| CANoe/HIL's role | Strong for Area 1 regression and Area 2 mitigation confirmation; not a substitute for Area 3 scenario discovery |
| Mitigation side-effects | Always test that a fix for a known-unsafe scenario didn't create a new insufficient-performance gap |
| ODD boundary testing | The transition/disengagement behavior at an ODD limit is directly testable even when the underlying perception isn't |

### [10 · Project — HIL Test Plan for an ECU Feature](level-3/10-project-hil-test-plan.md)

| Section | What it forces you to make explicit |
|---|---|
| Scope / out-of-scope | What this plan does NOT claim to cover, and why |
| Requirements traced | Every testcase ties to a requirement ID and ASIL |
| Safety goals/mechanisms | What specifically each fault-injection testcase is proving |
| Fault matrix | Coverage gaps are visible as empty cells, not silently absent |
| Known gaps | Honest, dated, risk-accepted — never hidden |
| Exit criteria | An unambiguous definition of "done" for the plan itself |

## Level 4 · Master

### [01 · Test Strategy for Vehicle Programs](level-4/01-test-strategy-vehicle-programs.md)

| Concept | Key point |
|---|---|
| Strategy vs. plan | Strategy governs the whole program's test organization; a plan governs one feature |
| V-model backbone | Explicitly decide what's tested at each level and what's NOT re-tested at the next |
| Supplier/OEM split | Must be explicit per ECU — ambiguity causes gaps or duplication |
| Risk-based allocation | ASIL, new-vs-carryover, and field history should drive where test effort concentrates |
| Toolchain commitments | Needed for a program-wide traceability/coverage view to be possible at all |

### [02 · Continuous Testing in Automotive](level-4/02-continuous-testing-automotive.md)

| Concept | Key point |
|---|---|
| SIL-before-HIL | Catch cheap defects in simulation before spending scarce rig time |
| Rig as bottleneck | Physical rig capacity, not compute, caps parallelism — schedule, don't just queue-and-hope |
| Queue vs. fail | No rig capacity is a scheduling state, never a false test failure |
| State reset between jobs | Explicit relay/build-verification teardown and setup on every job, not assumed |
| Feedback tiering | Fast SIL/smoke feedback for developers; thorough nightly regression for full confidence |

### [03 · Advanced Fault Injection &amp; Robustness Testing](level-4/03-advanced-fault-injection.md)

| Concept | Key point |
|---|---|
| Cascading/combined faults | Single-fault-clean does not imply combination-clean; test independent-fault tracking explicitly |
| Combinatorial explosion | Exhaustive combination testing is usually infeasible — use risk-prioritized or pairwise selection |
| Pairwise testing | Covers every pair of conditions with far fewer cases than full combinatorics; doesn't guarantee 3-way interactions |
| Robustness testing | Looks for graceful degradation under unanticipated stress, not confirmation of one anticipated fault |
| Robustness assertions | Often "didn't crash/hang," not one single expected end-state |

### [04 · Cybersecurity Testing for Vehicles (ISO 21434)](level-4/04-cybersecurity-testing.md)

| Concept | Key point |
|---|---|
| TARA | Cybersecurity's FMEA-equivalent — threats, attack paths, CAL ratings drive test rigor |
| CAL | The security analog of ASIL — higher CAL needs deeper testing |
| UDS Security Access (0x27) | Seed/key exchange gating sensitive diagnostic services |
| Lockout testing | As important as key-validation testing — a resettable/absent lockout defeats the mechanism |
| Adversarial habits within existing tooling | Spoofing, malformed-input, and logging checks are reachable with CAPL; deep pentesting is not |

### [05 · Test Data Management at Scale](level-4/05-test-data-management.md)

| Concept | Key point |
|---|---|
| Version skew | The most common test-data failure — DBC/A2L/software/test-source drifting out of alignment |
| Configuration baseline | One record naming the exact matched version of every artifact involved in a test run |
| Version control for non-code artifacts | DBC/A2L/CAPL deserve the same discipline as source code, not loose shared-drive files |
| Checksum verification | Cheap, automatable guard against accidental artifact substitution |
| Asymmetric retention | Keep failure evidence rich and long; passing-test raw traces can be pruned aggressively |

### [06 · Building a Test Automation Team/Practice](level-4/06-building-test-automation-team.md)

| Concept | Key point |
|---|---|
| Distinct roles | Test engineer, framework engineer, rig engineer, architect, safety/security specialist — name them even on a small team |
| Bus factor | Sharper in this domain due to specialized CAPL/rig knowledge — mitigate with pairing, documentation-as-code, rotation |
| Career ladder | Map directly to this course's Level 1→4 skill progression to make growth legible |
| Team-health metrics | Bus-factor count, onboarding time, new-vs-rework ratio, cross-training matrix |

### [07 · Supplier &amp; OEM Test Collaboration Models](level-4/07-supplier-oem-collaboration.md)

| Concept | Key point |
|---|---|
| Core tension | OEM needs confidence without redundant re-testing; supplier needs to protect proprietary detail |
| ASPICE | Rates process maturity around testing, not individual test correctness |
| Evidence boundary | Reports/traceability/coverage typically exchanged; raw test source sometimes withheld as IP |
| Dispute resolution | Always check configuration baseline match first — the most common real cause of "different results" |
| Contractual milestones | Named entry/exit criteria gate program progression, mirroring Level 3 Module 10's plan structure at supplier-relationship scale |

### [08 · Homologation &amp; Compliance Testing Overview](level-4/08-homologation-compliance-testing.md)

| Concept | Key point |
|---|---|
| Homologation vs. internal validation | Regulatory-mandated, legally consequential, distinct from OEM-internal requirements |
| Not every feature is regulated | Homologation applies to specific mandated functions, not the whole vehicle equally |
| HIL's role | Pre-homologation derisking/rehearsal, never a substitute for the formal accredited test |
| Drift risk | Internal rehearsal scenarios must be sourced from actual regulatory-affairs expertise, not approximated by the test team |
| Dossier | Traceability and internal test evidence often support, but don't replace, the formal compliance documentation |

### [09 · Career Growth in Automotive Test Engineering](level-4/09-career-growth.md)

| Concept | Key point |
|---|---|
| Multiple paths | Specialist, architect/leadership, development, systems engineering, tooling — none preclude the others later |
| Readiness signals | Behavioral (independent judgment on ambiguous problems), not tenure-based |
| Portfolio over resume lines | Strategy documents, honest gap-stated fault matrices, working CI regression suites, traceability matrices |
| Certifications | Useful door-openers, not substitutes for demonstrated reasoning |

### [10 · Capstone — Full Test Strategy for a New ECU Program](level-4/10-capstone-ecu-test-strategy.md)

| Section | Modules it draws on |
|---|---|
| Scope/V-model | Level 4 Module 1 |
| Requirements/ASIL | Level 3 Modules 4, 8 |
| Fault matrix, combined faults | Level 3 Module 6, Level 4 Module 3 |
| Configuration baseline | Level 4 Module 5 |
| CI integration | Level 4 Module 2 |
| Supplier/OEM evidence boundary | Level 4 Module 7 |
| Honest gap disclosure | Level 3 Module 10's pattern, applied program-wide |
| Team structure | Level 4 Module 6 |

---

Go deeper with the full [Automotive Testing Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
