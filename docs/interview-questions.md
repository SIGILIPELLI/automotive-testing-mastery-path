---
title: "Automotive Testing Interview Questions and Answers (179 Questions)"
description: "179 Automotive Testing interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Automotive Testing Mastery Path."
---

# Automotive Testing Interview Questions and Answers

**179 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Automotive Testing Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Automotive Testing?](level-1/01-what-is-automotive-testing.md)

**Q1.** What is automotive testing?

**Answer:** Software and systems testing applied to ECUs, the 70–150+ small computers in a vehicle

General testing principles still apply.

**Q2.** What three things does automotive add to general testing?

**Answer:** ECUs talking over shared buses, hardware in the loop, and safety standards with legal teeth

ISO 26262 and ISO 21434 apply.

**Q3.** Why does a tester think in terms of bus messages?

**Answer:** Dozens of ECUs cooperate over CAN/LIN/Ethernet, so a bug can span ECUs

Not just function calls.

**Q4.** What does the V-model connect?

**Answer:** Development activities on the left to increasingly integrated test levels on the right (unit, integration, system, acceptance)

Horizontal arrows say 'this test verifies that activity'.

### [02 · CAN Bus Fundamentals](level-1/02-can-bus-fundamentals.md)

**Q5.** Why a shared bus instead of point-to-point wiring?

**Answer:** With 70–150+ ECUs, wiring every pair needs an unserviceable harness; CAN uses one pair of wires (CAN_H and CAN_L)

Every node listens and can transmit.

**Q6.** Why does differential signalling tolerate noise?

**Answer:** Noise hits both wires equally and cancels when the receiver reads the difference

Important in noisy vehicles.

**Q7.** What is the termination rule?

**Answer:** 120 Ω at both ends (about 60 Ω across the bus); missing or duplicate termination is a common fault

Check termination before blaming software when scope signals look reflected.

**Q8.** Dominant versus recessive?

**Answer:** Dominant (logical 0) is actively driven and always wins; recessive (logical 1) is the passive state

This one fact explains arbitration.

### [03 · CAN Frame Structure](level-1/03-can-frame-structure.md)

**Q9.** What does a classic CAN data frame carry?

**Answer:** SOF, arbitration (ID), control (DLC), data (0–8 bytes), CRC, ACK and EOF

Every tool is a view onto these bits.

**Q10.** Does a CAN frame carry a destination address?

**Answer:** No: an identifier describing what the data is; each node filters

You observe and inject on the shared bus.

**Q11.** What do arbitration and acknowledgment fall out of?

**Answer:** The dominant/recessive rule

Lower ID wins arbitration.

**Q12.** What's the data field size in classic CAN?

**Answer:** 0–8 bytes

CAN FD allows more.

### [04 · Introduction to Vector CANoe](level-1/04-introduction-to-canoe.md)

**Q13.** What three roles does CANoe play at once?

**Answer:** Simulation (act as missing ECUs), analysis (trace, decode, timing, bus load) and stimulation/test execution

One PC with a Vector interface connected to the bus.

**Q14.** Why simulate other ECUs?

**Answer:** Early in development or on a bench most neighbouring ECUs don't exist, or you don't want to risk expensive prototypes

The ECU under test believes it's on a complete network.

**Q15.** What does a database (DBC/ARXML) provide?

**Answer:** Maps raw CAN IDs and bytes to named, scaled signals

Used to decode raw bytes in the trace.

**Q16.** What does the lesson state about CANoe?

**Answer:** It's commercial software the course can't run, so it describes documented capabilities as a conceptual map

Use vendor training when you have access.

### [05 · CAPL Scripting Basics](level-1/05-capl-scripting-basics.md)

**Q17.** What is CAPL's execution model?

**Answer:** A collection of event procedures invoked by CANoe's runtime, not a single main()

C-like syntax, event-driven.

**Q18.** What does `on message SensorNodeStatus` do?

**Answer:** Runs every time a frame matching that message arrives, with `this` referring to it

Matched by symbolic name or raw ID.

**Q19.** What does `on start` do?

**Answer:** Fires once at the beginning of a measurement: initialise variables, arm timers, send an initial state

on preStart and on stopMeasurement are setup/teardown.

**Q20.** How do timers work?

**Answer:** Declare an msTimer, start it with setTimer(), and `on timer` fires once when it expires

The lesson notes snippets weren't run on live CANoe.

### [06 · UDS Diagnostics Basics](level-1/06-uds-diagnostics-basics.md)

**Q21.** What is UDS?

**Answer:** Unified Diagnostic Services (ISO 14229), the request/response diagnostic protocol ECUs speak

Client-server: tester requests, ECU responds.

**Q22.** What does a positive response's SID look like?

**Answer:** Request SID + 0x40 (e.g. 0x22 → 0x62)

It makes responses unmistakable.

**Q23.** What shape does a negative response have?

**Answer:** 0x7F, the request SID, then an NRC

NRC is the negative response code.

**Q24.** Which SID reads a live value by identifier, and which keeps a session alive?

**Answer:** 0x22 Read Data By Identifier; 0x3E Tester Present

0x27 is Security Access seed/key.

### [07 · Introduction to HIL Testing](level-1/07-introduction-to-hil-testing.md)

**Q25.** What is HIL testing?

**Answer:** A real physical ECU connected to a simulated environment (sensors, actuators, other ECUs, electrical environment) without a real vehicle

It's the dominant system-test technique.

**Q26.** What does HIL add beyond bench network testing?

**Answer:** Physical realism: sensor curves, actuator loads, supply transients and physical faults like a shorted wire

A pure CAN simulation can't represent them.

**Q27.** Why is HIL the sweet spot?

**Answer:** Real enough that the ECU can't tell it isn't in a vehicle, yet fully controllable and repeatable, at lower cost and safety risk

You can deliberately break things safely.

**Q28.** What runs the plant model in a rig?

**Answer:** A real-time simulation computer with a guaranteed fixed cycle (commonly 1 ms or faster)

The course describes architecture, not hands-on operation.

### [08 · Test Case Design for ECUs](level-1/08-test-case-design-for-ecus.md)

**Q29.** What's partitioned in ECU test design?

**Answer:** Signal ranges: the physical values a CAN signal, DID or sensor input can carry

Each case specifies the bus-level or diagnostic representation too.

**Q30.** With a 120 °C overheat threshold ('above 120'), what are the four partitions of CoolantTemp (−40 to 150 °C)?

**Answer:** Below −40 invalid, −40..120 normal, &gt;120..150 overheat, above 150 invalid

Representative values: −45, 90, 135, 155.

**Q31.** Which boundary values do you test for 'strictly greater than 120'?

**Answer:** 119.9 (no warning), 120.0 (no warning), 120.1 (warning)

120.0 is where a &gt; versus &gt;= bug lives.

**Q32.** Why not sweep every 16-bit raw value?

**Answer:** It's impossible and pointless once you've reasoned about which values behave alike

Equivalence partitioning needs one representative per class.

### [09 · Automotive Test Standards Overview](level-1/09-automotive-test-standards-overview.md)

**Q33.** What are the two standards behind much automotive rigour?

**Answer:** ASPICE (process maturity) and ISO 26262 (functional safety)

ISO 21434 (cybersecurity) comes in Level 4.

**Q34.** What are ASPICE capability levels 0 to 5?

**Answer:** Incomplete, performed, managed, established, predictable, innovating

Most OEM contracts target level 2 or 3.

**Q35.** Which ASPICE processes are testing-relevant?

**Answer:** SWE.4 unit verification, SWE.5 integration test, SWE.6 qualification test, and SYS.4/SYS.5 system levels

The course practices map onto them.

**Q36.** Why is ASPICE 'as much a commercial gate as a technical one'?

**Answer:** OEMs require a specific capability level from suppliers as a condition of business

Traceability and documented results become obligations.

### [10 · Project — Design a CAPL Test Script for a Simulated ECU Signal](level-1/10-project-capl-test-script.md)

**Q37.** When does the fuel node raise and clear LowFuelWarning?

**Answer:** Raises below 10%, clears only above 15% (hysteresis)

warningActive holds the hysteresis state.

**Q38.** What does the DiagnosticResetRequest do?

**Answer:** Resets the node's alive counter to 0, standing in for a UDS-style diagnostic trigger

Received, not sent, by the node.

**Q39.** How often does FuelLevelStatus transmit?

**Answer:** Every 100 ms, with FuelLevelPct, LowFuelWarning and a 4-bit AliveCounter

A msTimer cycleTimer drives it.

**Q40.** How honest is the project about verification?

**Answer:** It was reviewed for accuracy against CAPL conventions but not compiled or run in live CANoe

Treat it as a template to adapt.

## Level 2 · Intermediate

### [01 · Advanced CAPL Scripting](level-2/01-advanced-capl-scripting.md)

**Q41.** What does CAPL lack compared with C?

**Answer:** Pointers and dynamic memory: every array is fixed-size and declared up front

Functions return void, int, long, float, byte, char[] or a message.

**Q42.** What does `this.byte(i)` give?

**Answer:** A raw payload byte by index, regardless of whether a database defines signals

The fallback for diagnostics, unknown traffic and checksum/CRC routines.

**Q43.** What is a system variable (sysvar)?

**Answer:** State shared across the whole CANoe configuration

Used for panels and cross-node state.

**Q44.** What does the additive checksum example compute?

**Answer:** (b0 + b1 + b2 + b3) &amp; 0xFF

Many real ECUs use CRC8 or CRC16 with the same calling convention.

### [02 · CANoe Test Modules &amp; Test Cases](level-2/02-canoe-test-modules.md)

**Q45.** Why use a test module instead of a simulation node?

**Answer:** It produces a structured verdict (Pass/Fail/Inconclusive) and report, which makes suites auditable and machine-consumable for CI

write() text can't answer 'did it pass?'.

**Q46.** What does `testStepFail(...)` do?

**Answer:** Records a failing step verdict with a message

testStepPass records a pass.

**Q47.** What does testWaitForTimeout(500) do?

**Answer:** Blocks up to 500 ms waiting for the next assertion

Used in the startup sequence example.

**Q48.** What does testCaseTitle give?

**Answer:** A human-readable name for the case in the report

Test cases roll up into a report.

### [03 · CAN-FD Basics](level-2/03-can-fd-basics.md)

**Q49.** What's the max payload in CAN-FD versus classic CAN?

**Answer:** 64 bytes versus 8 bytes

The data phase can run at 5–8 Mbit/s.

**Q50.** What is the byte length for DLC 13 in CAN-FD?

**Answer:** 32 bytes

DLC 9→12, 10→16, 11→20, 12→24, 13→32, 14→48, 15→64.

**Q51.** What new control bits does CAN-FD add?

**Answer:** FDF (FD format), BRS (bit-rate switch) and ESI (error state indicator)

BRS switches to the faster data-phase rate.

**Q52.** What happens to the arbitration bit rate?

**Answer:** Unchanged, up to 1 Mbit/s; only the data phase speeds up

CRC grows to 17 or 21 bits.

### [04 · LIN Bus Basics](level-2/04-lin-bus-basics.md)

**Q53.** What is LIN's topology?

**Answer:** Master/slave: the master owns a schedule table of when every message is sent

A low-cost bus for simple devices.

**Q54.** What does a LIN header consist of?

**Answer:** Break (≥13 bit-times low), Sync 0x55 and the Protected ID

The PID is a 6-bit ID plus 2 parity bits.

**Q55.** What does the Sync field 0x55 do?

**Answer:** Lets slaves calibrate their baud rate

Sent by the master.

**Q56.** Classic versus enhanced checksum?

**Answer:** Classic covers data only; enhanced covers data plus the PID, depending on LIN version

Sent by the node that provides the data.

### [05 · Diagnostic Trouble Codes (DTCs) Deep Dive](level-2/05-dtc-deep-dive.md)

**Q57.** What does the DTC status byte's confirmedDTC bit (bit 3) mean?

**Answer:** The fault met confirmation criteria: the 'real' stored DTC

pendingDTC (bit 2) means failed but not yet confirmed.

**Q58.** What does testFailed (bit 0) mean?

**Answer:** The fault is failing right now, this operation cycle

testFailedSinceLastClear is bit 5.

**Q59.** What is bit 7 warningIndicatorRequested?

**Answer:** Whether the malfunction indicator lamp should be on because of this DTC

Lamp state follows DTC status.

**Q60.** What do the testNotCompleted bits mean?

**Answer:** The diagnostic test hasn't run since the last clear or this operation cycle

Absence of a failure isn't proof the test ran.

### [06 · Database Files (DBC) &amp; Signal Definitions](level-2/06-dbc-signal-definitions.md)

**Q61.** Which DBC blocks define nodes, messages and signals?

**Answer:** BU_ nodes, BO_ messages, SG_ signals

CM_ comments often carry the real spec; VAL_ gives enumerations; BA_ attributes.

**Q62.** A signal has scale 1, offset −40 and raw 130. What's the physical value?

**Answer:** 90

physical = raw × 1 + (−40) = 90.

**Q63.** What do @1 and @0 mean in a signal definition?

**Answer:** @1 little-endian (Intel); @0 big-endian (Motorola)

The sign column is + unsigned or − signed.

**Q64.** Why does the valid range [-40|215] matter?

**Answer:** It's used for validation and plausibility checks

Raw 8-bit values 0–255 with offset −40 give −40..215.

### [07 · Automated Test Sequences in CANoe](level-2/07-automated-test-sequences.md)

**Q65.** What does `testpreparation` do?

**Answer:** Runs once before any testcase in the module (shared setup)

Teardown is test completion.

**Q66.** Why chain cases into sequences?

**Answer:** Shared setup/teardown and reporting that survives one case failing without aborting the whole run

Orchestration layer for real suites.

**Q67.** What does the boundary test of 120.0 then 120.1 check?

**Answer:** Warning stays 0 at 120.0 and goes to 1 at 120.1 (strictly above)

testStepCheck asserts each.

**Q68.** Why must cases be independent and well ordered?

**Answer:** A failure in one shouldn't corrupt the others, and state must be reset

Ordering and independence section.

### [08 · Restbus Simulation](level-2/08-restbus-simulation.md)

**Q69.** What is restbus simulation?

**Answer:** Simulating all the other ECUs' bus traffic around the ECU under test so it sees a complete network

The missing ECUs may not exist yet.

**Q70.** How can a restbus be generated quickly?

**Answer:** Automatically from a DBC (cycle times and messages per node)

Attributes like cycle time drive transmission.

**Q71.** Why is it needed at all?

**Answer:** ECUs monitor neighbours' messages and set faults or fallback modes when they're missing

A silent bus triggers timeout DTCs.

**Q72.** What should the simulated node do besides send?

**Answer:** React to messages sent to it as the real ECU would

Simulation is both transmit and react.

### [09 · Basic HIL Test Bench Components](level-2/09-hil-test-bench-components.md)

**Q73.** Where does HIL sit among test tiers?

**Answer:** Between pure simulation (nothing physical) and vehicle test (everything real): the ECU is real, the rest simulated

It catches defects only visible with real hardware and software.

**Q74.** What does the real-time simulation computer do?

**Answer:** Runs the plant model in hard real time, completing each step within its cycle

Missing a cycle breaks determinism.

**Q75.** What does I/O interface hardware do?

**Answer:** Converts between model values and real electrical signals the ECU expects

Sensor emulation and actuator loads.

**Q76.** What is the ECU under test in a HIL?

**Answer:** The real, unmodified production or pre-production hardware and software

Bus interfaces connect it to the network.

### [10 · Project — Automated CAN Signal Validation Suite](level-2/10-project-can-signal-validation.md)

**Q77.** What are the three messages in the project?

**Answer:** EngineStatus (256), BrakeStatus (512) and SensorFusionFrame (1024, CAN-FD)

Cycle times 10, 20 and 20 ms via GenMsgCycleTime.

**Q78.** What does the suite validate?

**Answer:** Signal range plausibility, CAN-FD frame properties and cycle timing, with a restbus as supporting cast

Pulls every Level 2 module together.

**Q79.** How is the 32-byte frame marked as FD?

**Answer:** A VFrameFormat attribute on the message

DLC 13 means 32 bytes.

**Q80.** What does the restbus provide in this project?

**Answer:** The other ECUs' traffic so the ECU under test sees a complete network

It's the supporting cast.

## Level 3 · Advanced

### [01 · Advanced HIL Test Automation](level-3/01-advanced-hil-automation.md)

**Q81.** What are the two HIL automation layers?

**Answer:** A real-time layer (plant model each fixed timestep) and a test orchestration layer (what scenario, parameters, results, pass/fail)

The orchestration layer never touches the real-time loop's internals.

**Q82.** What does 'parameterize, don't hardcode' mean for scenarios?

**Answer:** Drive scenario variations from parameters/tables instead of copying scripts

It scales coverage.

**Q83.** What must the real-time layer guarantee?

**Answer:** Deterministic execution every fixed timestep, no exceptions

Examples: dSPACE or NI real-time OS.

**Q84.** Which layer makes pass/fail calls?

**Answer:** Test orchestration

It reads results.

### [02 · CANape &amp; Measurement/Calibration Basics](level-3/02-canape-measurement-calibration.md)

**Q85.** How do CANoe and CANape differ?

**Answer:** CANoe sees bus traffic and signals; CANape sees ECU-internal variables (RAM addresses) via XCP/CCP

CANoe needs a DBC; CANape needs an A2L.

**Q86.** What is an A2L (ASAP2) file?

**Answer:** The map from symbol name to ECU memory address, like a DBC for calibration

It describes internal symbols and addresses.

**Q87.** Which two A2L categories matter for testing?

**Answer:** MEASUREMENT (read live values) and CHARACTERISTIC (calibration parameters)

Only measurement is read-only.

**Q88.** Which protocol does CANape use?

**Answer:** ASAM XCP (or CCP, its predecessor)

Over CAN or Ethernet.

### [03 · Ethernet/Automotive Ethernet Basics](level-3/03-automotive-ethernet-basics.md)

**Q89.** Why Ethernet in vehicles?

**Answer:** CAN tops out around 1 Mbit/s (5 for CAN-FD data phase); vehicle Ethernet runs 100 Mbit/s to multi-gigabit for cameras, radar and infotainment

ADAS and infotainment need the bandwidth.

**Q90.** What's automotive Ethernet's physical layer?

**Answer:** 100BASE-T1 / 1000BASE-T1: a single unshielded twisted pair, chosen to cut harness weight and cost

Office Ethernet uses 100BASE-TX.

**Q91.** What is SOME/IP?

**Answer:** An AUTOSAR-standard service-oriented middleware over IP, with services and methods

CAN signals are static; Ethernet comms are often service-oriented.

**Q92.** Which protocols ride on automotive Ethernet?

**Answer:** SOME/IP, DoIP (diagnostics) and AVB/TSN for time-sync media

TCP/IP underneath.

### [04 · Functional Safety Testing (ISO 26262)](level-3/04-functional-safety-testing.md)

**Q93.** What is ASIL?

**Answer:** Automotive Safety Integrity Level (QM, A–D, D strictest) assigned per hazard from severity, exposure and controllability

It's risk-graded, not a pass/fail badge.

**Q94.** What does ASIL attach to?

**Answer:** Specific safety goals/functions, not the whole ECU

Higher ASIL means more rigorous traceable, reviewed verification.

**Q95.** What are the three ASIL factors?

**Answer:** Severity (S0–S3), exposure (E0–E4), controllability (C0–C3)

Each hazard is rated on all three.

**Q96.** What are safety mechanisms?

**Answer:** Specific detections with safe reactions, e.g. a range check that triggers a fallback

You test that they work when faults occur.

### [05 · Test Automation Frameworks for ECU Testing](level-3/05-test-automation-frameworks.md)

**Q97.** Why wrap CAPL in a framework?

**Answer:** A layer above CANoe handles sequencing, rig state, data, traceability and CI-readable reporting beyond CAPL's strengths

CAPL is great at driving bus traffic.

**Q98.** What are the framework layers?

**Answer:** Execution core, orchestration, data layer, reporting

Orchestration is often Python via CANoe's COM API.

**Q99.** What does driving CANoe via COM require?

**Answer:** A Windows host with CANoe installed and pywin32 (exact interface names vary by version)

The snippet is illustrative.

**Q100.** Which report formats suit CI?

**Answer:** JUnit XML and HTML dashboards

Machine-consumable outputs.

### [06 · Fault Injection Testing](level-3/06-fault-injection-testing.md)

**Q101.** What are the fault classes in the taxonomy?

**Answer:** Signal-level, communication-level, electrical-level, power-level, software/timing-level

A test plan covering only signal values misses the rest.

**Q102.** Which class can CAPL alone inject?

**Answer:** Signal-level: e.g. setSignal() to an implausible value

Communication faults are partially possible by suppressing frames.

**Q103.** What do electrical faults need?

**Answer:** A physical relay/fault-insertion unit in the HIL rig

Open, short to ground, short to battery.

**Q104.** What do power-level faults need?

**Answer:** A programmable power supply to create sags, brownouts and ignition glitches

For example a glitch during a flash write.

### [07 · Regression Test Suites for Automotive Software](level-3/07-regression-test-suites.md)

**Q105.** Which property makes a regression suite trustworthy?

**Answer:** Deterministic: same input, same result every run; a flaky suite erodes trust and is worse than none

Also fast enough to gate changes and independent testcases.

**Q106.** How are tiers arranged?

**Answer:** Smoke (2–5 min), functional (30–60 min), full regression (hours: fault injection, HIL matrices, soak)

Tiering solves the speed-versus-depth tension.

**Q107.** Why 'clear failure attribution'?

**Answer:** A failing name/log should point at what broke, not just 'suite failed'

It shortens triage.

**Q108.** What does 'maintained, not just accumulated' mean?

**Answer:** Remove tests for removed features and update tests for changed behaviour

Otherwise the suite rots.

### [08 · Requirements-Based Test Traceability](level-3/08-requirements-based-traceability.md)

**Q109.** What's forward versus backward traceability?

**Answer:** Forward: requirement → its tests; backward: test → the requirement(s) it verifies

Without backward traceability a suite accumulates orphan tests.

**Q110.** What does verdict coverage measure?

**Answer:** Of covered requirements, how many have a passing latest result versus never-run or failing

A requirement with one failing test is very different from a passing one.

**Q111.** What does ASIL-weighted coverage check?

**Answer:** For safety-relevant requirements, that the required rigour (review, test techniques) was applied

Higher ASIL demands more.

**Q112.** Why traceability at all?

**Answer:** It lets a suite answer 'is this requirement verified?' and supports audits (ASPICE, ISO 26262)

The chain runs from stakeholder need to test result.

### [09 · SOTIF &amp; Testing for Autonomous Features](level-3/09-sotif-autonomous-testing.md)

**Q113.** What does SOTIF (ISO 21448) address?

**Answer:** Hazards when the system works exactly as designed but the design is insufficient for the real world

ISO 26262 covers malfunctions.

**Q114.** Example of a SOTIF problem?

**Answer:** A radar correctly reports what it sees but fails to classify a stationary object

Root fix: broaden the validated operating envelope or add complementary sensing.

**Q115.** What are the SOTIF areas?

**Answer:** Known safe, known unsafe, unknown unsafe, unknown safe (four areas of the operating space)

The goal is to shrink the unknown-unsafe area.

**Q116.** For Area 1 (known safe), what's the testing goal?

**Answer:** Regression-test to keep it that way

Area 2 needs design restrictions.

### [10 · Project — HIL Test Plan for an ECU Feature](level-3/10-project-hil-test-plan.md)

**Q117.** What feature does the project plan test?

**Answer:** Automatic Emergency Braking (AEB), scoped down to one clear feature

Scoping keeps it concrete.

**Q118.** Why list out-of-scope items with reasons?

**Answer:** It's part of the plan: it states what isn't being verified and why

Explicit boundaries prevent false assurance.

**Q119.** What's SW-REQ-201?

**Answer:** AEB commands full braking within 150 ms of a validated imminent-collision condition (ASIL D)

SW-REQ-202: no activation on implausible signals.

**Q120.** What does the plan trace each requirement to?

**Answer:** Test cases and ASIL, so rigour and coverage are explicit

From Module 8.

## Level 4 · Master

### [01 · Test Strategy for Vehicle Programs](level-4/01-test-strategy-vehicle-programs.md)

**Q121.** How does a test strategy differ from a test plan?

**Answer:** Strategy covers the whole vehicle program and all ECUs (owned by the program test lead); a plan covers one feature or ECU

A testcase is one check by a test engineer.

**Q122.** What backs the strategy?

**Answer:** The V-model: vehicle, system and software requirements each paired with a test level

Vehicle validation, multi-ECU HIL, then unit and component HIL.

**Q123.** What's the strategy's central job per level?

**Answer:** Deciding what gets tested where, by whom, and with what environment

Including the OEM/supplier split.

**Q124.** Who builds most ECUs?

**Answer:** Suppliers, so the strategy must define OEM versus supplier test responsibility

Typical owner varies per level.

### [02 · Continuous Testing in Automotive](level-4/02-continuous-testing-automotive.md)

**Q125.** How does automotive CI differ from pure software CI?

**Answer:** HIL rigs are expensive and physical, so rig time caps parallelism; flaky failures may be rig wiring; rollback may mean re-flashing an ECU

Rig scheduling is the bottleneck.

**Q126.** Why run SIL before hardware steps?

**Answer:** It's cheap and fast, filtering out failures before scarce rig time is spent

Software-in-the-loop catches logic problems first.

**Q127.** What can a flaky failure be?

**Answer:** A test problem, a rig wiring problem, or a genuine intermittent ECU defect

Triage needs rig health data.

**Q128.** What does rig scheduling do?

**Answer:** Reserves physical rigs for jobs to avoid conflicts and maximise use

The pipeline queues for a rig.

### [03 · Advanced Fault Injection &amp; Robustness Testing](level-4/03-advanced-fault-injection.md)

**Q129.** Why test cascading faults, not just single faults?

**Answer:** A mechanism proven for fault A alone and fault B alone may fail when both occur together

Single-fault testing structurally can't find that.

**Q130.** What is robustness testing?

**Answer:** Exercising the ECU under combinations, timing and stress to find unexpected behaviour

Beyond the taxonomy's single faults.

**Q131.** What must a fault-injection test always do afterwards?

**Answer:** Restore the system to a known state so later tests are independent

Module 6's 'always restore' discipline.

**Q132.** What does the second assertion in the cascading example check?

**Answer:** That the combined fault still reaches the safe state

Safe state under combined faults.

### [04 · Cybersecurity Testing for Vehicles (ISO 21434)](level-4/04-cybersecurity-testing.md)

**Q133.** What is TARA?

**Answer:** Threat Analysis and Risk Assessment: ISO 21434's structured way to find assets, threats, attack paths, impact and a CAL

The cybersecurity analogue of FMEA.

**Q134.** What does the Cybersecurity Assurance Level (CAL) drive?

**Answer:** How much testing rigour a requirement needs, like ASIL for safety

Test rigour scales with risk.

**Q135.** What's an example attack path?

**Answer:** A compromised infotainment unit → gateway → braking ECU if the gateway doesn't segment traffic

Segmentation is a countermeasure.

**Q136.** What protects sensitive UDS services?

**Answer:** Security Access (0x27) seed/key, which test engineers exercise often

Calibration writes are protected.

### [05 · Test Data Management at Scale](level-4/05-test-data-management.md)

**Q137.** What must be versioned together?

**Answer:** DBC, A2L, ECU software build, test case source and fault-injection matrix, so they match

The biggest failure mode at scale is mismatched artifacts.

**Q138.** What is a configuration baseline?

**Answer:** A single record naming the exact versions of all artifacts used for a result

For example AEB-DomainController-v2.4.1-baseline.

**Q139.** Why is it needed?

**Answer:** Without it a result from six months ago is nearly meaningless

You can't reproduce what you can't identify.

**Q140.** How should DBC, A2L and CAPL be handled?

**Answer:** Version-controlled like code, even if structured-binary

Non-code artifacts need version control.

### [06 · Building a Test Automation Team/Practice](level-4/06-building-test-automation-team.md)

**Q141.** Which roles does a mature practice include?

**Answer:** Test engineer, automation/framework engineer, test architect/lead, rig/infrastructure engineer, safety/security specialist

Smaller teams collapse several into one person.

**Q142.** Why is bus-factor sharper in this domain?

**Answer:** Scarce combined skills (CAPL, rigs, safety) sit with few people

Mitigations include pairing.

**Q143.** How does pairing on ASIL C/D test authorship help?

**Answer:** It forces knowledge transfer as a side effect of the independent-review requirement

Two birds, one stone.

**Q144.** What does the rig engineer maintain?

**Answer:** Physical HIL rigs, fault-injection hardware and calibration of the test equipment itself

A key infrastructure role.

### [07 · Supplier &amp; OEM Test Collaboration Models](level-4/07-supplier-oem-collaboration.md)

**Q145.** What's the fundamental tension?

**Answer:** The OEM needs confidence a supplier's ECU meets requirements without owning the supplier's internal process

ASPICE gives both sides a shared language.

**Q146.** Which ASPICE processes cover software unit, integration and qualification testing?

**Answer:** SWE.4, SWE.5 and SWE.6

SYS.4/SYS.5 cover system levels, often OEM-owned.

**Q147.** Is an ASPICE capability level a pass/fail grade on CAPL tests?

**Answer:** No: you can pass every test and still score poorly if traceability and process evidence are weak

It assesses process maturity.

**Q148.** Which work product does SWE.6 expect?

**Answer:** A traceability matrix linking requirements to qualification tests

From Level 3 Module 8.

### [08 · Homologation &amp; Compliance Testing Overview](level-4/08-homologation-compliance-testing.md)

**Q149.** How does homologation differ from internal validation?

**Answer:** A regulator defines the requirement and failure means the vehicle can't be legally sold; evidence often needs accredited labs

Internal failure means delay and rework.

**Q150.** Which functions are homologation-relevant?

**Answer:** Specific regulated functions: braking, lighting, emissions, certain ADAS functions

A courtesy light usually isn't.

**Q151.** How does HIL testing support homologation?

**Answer:** Pre-homologation validation against the anticipated procedure, regression after late changes, and supporting evidence

It supports but doesn't replace formal testing.

**Q152.** Who issues the requirements?

**Answer:** Regulatory bodies such as UN ECE or NHTSA/FMVSS

Market-specific.

### [09 · Career Growth in Automotive Test Engineering](level-4/09-career-growth.md)

**Q153.** Which career paths does the course name?

**Answer:** Specialist (safety, cybersecurity, SOTIF), architecture/leadership, move to ECU development, systems engineering, tooling/framework specialist

None closes off the others later.

**Q154.** Which role is often the scarcest on a team?

**Answer:** Tooling/framework specialist

Deep automation and infrastructure skill.

**Q155.** What signals readiness for Senior Test Engineer?

**Answer:** Independently designing a fault-injection matrix or similar Level 3 work

Evidence beats tenure.

**Q156.** What signals readiness for Test Architect/Lead?

**Answer:** Having navigated a real supplier/OEM disagreement or built a working strategy

Level 4 skillset.

### [10 · Capstone — Full Test Strategy for a New ECU Program](level-4/10-capstone-ecu-test-strategy.md)

**Q157.** What program does the capstone use?

**Answer:** A new Battery Management System (BMS) ECU with genuine safety and security content

Chosen as new, not carryover.

**Q158.** What does SW-REQ-501 require?

**Answer:** Disconnect the pack contactor within 200 ms of detecting cell overvoltage (ASIL D)

SW-REQ-502 rejects implausible readings.

**Q159.** What does SW-REQ-503 involve?

**Answer:** Authenticating charging-station communication before accepting a charge-current setpoint (ASIL B with a cybersecurity level)

Safety and security meet.

**Q160.** What does the fault matrix include?

**Answer:** Mechanisms, faults tested and combined-fault cases from Level 4 Module 3

Scope states what's explicitly out of scope.

## Mixed questions across all levels

**Q161.** Two ECUs transmit at the same instant. ECU A sends ID 0x100 and ECU B sends 0x200. Who wins arbitration?

**Answer:** ECU A: the lower ID has dominant bits first and dominant always wins; B backs off and retries

Arbitration falls out of the dominant/recessive rule.

**Q162.** The scope shows reflected, garbled CAN signals. What do you check first?

**Answer:** Termination: 120 Ω at both ends (about 60 Ω across), since missing or duplicate termination is a common fault

Don't blame software before checking the physical layer.

**Q163.** A CAN-FD frame has DLC 13. How many data bytes?

**Answer:** 32

DLC 9 to 15 map to 12, 16, 20, 24, 32, 48, 64 bytes.

**Q164.** A signal is defined with start bit 16, length 8, scale 1, offset −40 (Intel). Raw byte 2 is 130. What is the physical value?

**Answer:** 90 °C

physical = raw × 1 − 40. Verified by decoding a test frame.

**Q165.** A signal with scale 0.1 and raw 0x012C (300) means:

**Answer:** 30.0 physical units

300 × 0.1 = 30.0. Verified by computation.

**Q166.** A UDS request is 22 F1 90. What does a positive response start with?

**Answer:** 62 F1 90 followed by the data

Positive response SID = request SID + 0x40; the DID is echoed. Verified with a small responder.

**Q167.** What's the response to 22 12 34 when DID 0x1234 isn't supported?

**Answer:** 7F 22 31 (negative response, requestOutOfRange)

Negative responses are 7F, the SID, then the NRC.

**Q168.** What's the response to an unknown service 0x99?

**Answer:** 7F 99 11 (serviceNotSupported)

Verified with the responder.

**Q169.** Which UDS service keeps a non-default session alive?

**Answer:** 0x3E Tester Present

0x10 changes session; 0x3E pings.

**Q170.** A CoolantTemp warning must trigger when temperature is strictly above 120 °C. Which boundary tests are best?

**Answer:** 119.9 (no warning), 120.0 (no warning), 120.1 (warning)

120.0 is where &gt; versus &gt;= bugs hide.

**Q171.** A low-fuel warning raises below 10% and clears above 15%. At 12% after having been below 10%, is the warning on?

**Answer:** Yes: hysteresis keeps it on until above 15%

warningActive holds the state.

**Q172.** In CAPL, which handler runs each time a particular frame arrives?

**Answer:** on message &lt;name or ID&gt;

on start fires once; on timer when a timer expires.

**Q173.** Why use a CANoe test module rather than a simulation node with write()?

**Answer:** It produces structured verdicts and reports that CI and auditors can consume

Text in a log isn't a verdict.

**Q174.** A requirement has one test case which is currently failing. What does coverage say?

**Answer:** Requirement covered but verdict not passing; verdict coverage separates covered from passing

Requirement coverage alone isn't enough.

**Q175.** Which standard covers hazards when the system works as designed but the design is insufficient (e.g. perception limits)?

**Answer:** ISO 21448 SOTIF

ISO 26262 covers malfunctions; 21434 cybersecurity.

**Q176.** Why test combined (cascading) faults?

**Answer:** Mechanisms proven separately for faults A and B may still fail when both occur

Single-fault testing can't find that.

**Q177.** A test result from six months ago can't be reproduced. What was missing?

**Answer:** A configuration baseline naming the exact DBC, A2L, ECU build, test source and fault matrix versions

Artifacts must be versioned together.

**Q178.** A HIL rig is shared by 10 pipelines. What is the automotive-specific bottleneck?

**Answer:** Rig time: parallelism is capped by the number of physical rigs, so reservations are scheduled

Run SIL first to save rig time.

**Q179.** An ASPICE assessor scores a supplier low although all CAPL tests pass. Why can that happen?

**Answer:** ASPICE assesses process maturity and evidence such as traceability, not just test outcomes

Capability levels 0–5 grade process.

## Practical coding questions

### Decode a DBC signal

Write decode(raw_bytes, start_bit, length, scale, offset) that extracts a little-endian (Intel) signal and returns physical = raw × scale + offset. Decode CoolantTemp (start 16, length 8, scale 1, offset −40) from raw byte 130, and a 16-bit signal with scale 0.1 from 0x012C.

Starter code:

```
def decode(raw_bytes, start_bit, length, scale, offset, little_endian=True):
    # extract `length` bits at `start_bit` (Intel/little-endian) and apply physical = raw * scale + offset
    pass

frame = bytes([0x00, 0x00, 130, 0, 0, 0, 0, 0])
print(decode(frame, 16, 8, 1, -40))
print(decode(bytes([0x2C, 0x01]) + bytes(6), 0, 16, 0.1, 0))
```

Expected output:

```text
90
30.0
```

??? success "Show a solution"

    ```
    def decode(raw_bytes, start_bit, length, scale, offset, little_endian=True):
        value = int.from_bytes(raw_bytes, "little" if little_endian else "big")
        raw = (value >> start_bit) & ((1 << length) - 1)
        return raw * scale + offset

    frame = bytes([0x00, 0x00, 130, 0, 0, 0, 0, 0])
    print(decode(frame, 16, 8, 1, -40))
    print(decode(bytes([0x2C, 0x01]) + bytes(6), 0, 16, 0.1, 0))
    ```

    A DBC maps bit position, length, endianness, scale and offset to a physical value. Verified by running the solution.

### A tiny UDS responder

Write respond(req) for three services: 0x10, 0x22 (Read Data By Identifier with DID 0xF190 = a VIN) and 0x3E. Positive response = SID + 0x40 (for 0x22 echo the DID then the data). Unknown service: 7F SID 11. Unknown DID: 7F 22 31. Print each response as hex.

Starter code:

```
SERVICES = {0x10: "Session", 0x22: "ReadDID", 0x3E: "TesterPresent"}
DIDS = {0xF190: b"WVWZZZ1JZXW000001"}

def respond(req):
    # positive: SID+0x40 (+ DID echo + data for 0x22); negative: 7F SID NRC
    # NRC 0x11 serviceNotSupported, 0x31 requestOutOfRange
    pass

for r in ([0x22, 0xF1, 0x90], [0x22, 0x12, 0x34], [0x3E, 0x00], [0x99]):
    print(respond(bytes(r)).hex(" "))
```

Expected output:

```text
62 f1 90 57 56 57 5a 5a 5a 31 4a 5a 58 57 30 30 30 30 30 31
7f 22 31
7e 00
7f 99 11
```

??? success "Show a solution"

    ```
    SERVICES = {0x10: "Session", 0x22: "ReadDID", 0x3E: "TesterPresent"}
    DIDS = {0xF190: b"WVWZZZ1JZXW000001"}

    def respond(req):
        sid = req[0]
        if sid not in SERVICES:
            return bytes([0x7F, sid, 0x11])          # serviceNotSupported
        if sid == 0x22:
            did = (req[1] << 8) | req[2]
            if did not in DIDS:
                return bytes([0x7F, sid, 0x31])      # requestOutOfRange
            return bytes([sid + 0x40, req[1], req[2]]) + DIDS[did]
        return bytes([sid + 0x40]) + req[1:2]

    for r in ([0x22, 0xF1, 0x90], [0x22, 0x12, 0x34], [0x3E, 0x00], [0x99]):
        print(respond(bytes(r)).hex(" "))
    ```

    UDS positive responses add 0x40 to the SID; negative responses are 0x7F, the SID and an NRC. Verified by running the solution.

---

More free practice: the [Automotive Testing Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
