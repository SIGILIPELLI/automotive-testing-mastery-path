# 02 · CANape & Measurement/Calibration Basics

CANoe validates bus behavior and diagnostics; CANape does something
adjacent but different — it measures and calibrates ECU-internal
signals over ASAM MCD-3MC/XCP, the protocol that reaches past the bus
into an ECU's live RAM. This module covers the A2L/XCP model, how
measurement and calibration differ, and where a test engineer touches
CANape-style tooling day to day.

!!! note "About this module"
    No CANape license or physical ECU was used to produce this
    content. The A2L/XCP concepts, memory-segment model, and workflow
    below reflect documented ASAM XCP and A2L standards and common
    calibration-tool practice — verify exact menu paths and API calls
    against your installed CANape/XCP tool version.

## Why CANape exists alongside CANoe

| | CANoe | CANape |
|---|---|---|
| Primary view | Bus traffic (frames, signals as transmitted) | ECU-internal variables (RAM addresses, live) |
| Protocol | Raw CAN/LIN/FlexRay/Ethernet + UDS | ASAM XCP (or CCP, its predecessor) over CAN/Ethernet |
| Typical use | Functional/system test, restbus simulation | Calibration engineering, measurement/rapid prototyping |
| What it needs from the ECU supplier | A DBC/signal database | An **A2L file** describing internal symbols and addresses |

A test engineer usually isn't the primary CANape user — calibration
engineers are — but functional and performance testing frequently
needs to *observe* an internal variable that never appears on the bus
(a filtered sensor value, an internal state machine variable, a PID
controller's integral term), and that's exactly what XCP measurement
gives you without instrumenting the ECU's application code.

## The A2L file: the map from name to address

An A2L (ASAP2) file is the calibration-side equivalent of a DBC file.
Where a DBC maps a CAN signal to bit position, an A2L maps a **symbol
name** to a **memory address, data type, and scaling**:

```text
/begin MEASUREMENT
  EngineSpeed_rpm
  "Filtered engine speed"
  UWORD NO_COMPU_METHOD 0 0 0 8000
  ECU_ADDRESS 0x80012A40
/end MEASUREMENT

/begin CHARACTERISTIC
  KP_ThrottleController
  "Proportional gain, throttle position controller"
  VALUE 0x80020100 DAMOS_SST 0 NO_COMPU_METHOD 0.0 4.0
/end CHARACTERISTIC
```

Two categories matter for testing:

- **MEASUREMENT** — a read-only (from the tool's perspective) live
  variable you can watch, log, and plot during a test — engine speed,
  a diagnostic session state, a filtered sensor value.
- **CHARACTERISTIC** — a tunable calibration constant (a gain, a
  threshold, a lookup table) you can *write* live, without reflashing,
  to explore how ECU behavior changes with different tuning — directly
  useful for boundary and robustness testing around a threshold.

The A2L file must match the exact software build in the ECU — a
mismatched A2L (wrong address for the software version actually
flashed) reads garbage or writes to the wrong memory location, so
build/A2L pairing is itself a configuration-management concern for a
test team, the same discipline as DBC-to-build pairing from Level 2.

## XCP: the protocol underneath

XCP (Universal Measurement and Calibration Protocol) runs a
master/slave model: the tool (CANape, or a CAPL/COM-driven XCP master)
is the **master**; the ECU is the **slave**. Core operations:

| XCP service | Purpose |
|---|---|
| `CONNECT` | Establish session, negotiate protocol version/resources |
| `GET_STATUS` | Query current session state |
| `SET_DAQ_PTR` / `WRITE_DAQ` | Configure a Data Acquisition (DAQ) list — which addresses to sample, at what rate |
| `START_STOP_DAQ_LIST` | Start/stop streaming a configured DAQ list |
| `SHORT_UPLOAD` | One-shot read of a memory address (ad hoc measurement) |
| `DOWNLOAD` | Write a value to a memory address (calibration change) |
| `DISCONNECT` | End session, release resources |

A DAQ list is the efficient path: rather than polling addresses one at
a time, you configure a list once and the ECU streams sampled values
at a fixed rate, similar in spirit to configuring a CANoe trace filter
once rather than re-querying per frame.

## A CAPL-side XCP measurement sketch

CANoe includes XCP support so you can add live-variable checks to an
existing CAPL-based test suite without switching tools entirely:

```c
// Illustrative — exact XCP driver function names depend on the CANoe
// XCP option's API surface in your installed version.
variables
{
  msTimer settleTimer;
}

testcase tc_ThrottleGainAffectsResponseTime()
{
  float originalGain, testGain, responseMs;
  testCaseTitle("Reducing KP_ThrottleController measurably slows step response");

  originalGain = XcpGetCharacteristicValue("KP_ThrottleController");
  testGain = originalGain * 0.5;
  XcpSetCharacteristicValue("KP_ThrottleController", testGain);
  testWaitForTimeout(50); // allow the write to take effect

  responseMs = MeasureThrottleStepResponseTime(); // drives pedal, times via XCP DAQ
  testStepCheck("response time increased vs. baseline", responseMs > gBaselineResponseMs);

  // Always restore — a left-over calibration change corrupts every later test.
  XcpSetCharacteristicValue("KP_ThrottleController", originalGain);
}
```

The restore step is not optional. A CHARACTERISTIC write survives
until explicitly reverted or the ECU is repowered/reflashed — leaving
a modified gain in place after a test silently corrupts every
subsequent test in the same session, session run, or shared rig.

## Measurement vs. calibration in test scope

| Activity | Typical actor | Test-relevant use |
|---|---|---|
| Measurement (read DAQ) | Test/calibration engineer | Observe internal state a bus signal can't show — root-causing a test failure, verifying an internal threshold crossing |
| Calibration (write CHARACTERISTIC) | Calibration engineer, sometimes test engineer for robustness sweeps | Boundary testing around a tunable threshold without reflashing for every value |
| Flash (new full calibration set) | Release/calibration engineer | Out of scope for most test automation — a coarser-grained, slower operation than a live CHARACTERISTIC write |

## Cheat sheet

| Concept | Key point |
|---|---|
| A2L file | Maps symbol name → address/type/scaling; must match the exact ECU software build |
| MEASUREMENT | Read-only live variable, safe to observe |
| CHARACTERISTIC | Writable tunable value — always restore after use |
| DAQ list | Configure once, stream many samples — avoid per-address polling |
| XCP master/slave | Tool is master, ECU is slave; CONNECT/DAQ/DISCONNECT lifecycle |

## Exercise

1. Explain concretely what goes wrong if a test suite runs against an
   A2L file generated for software build 2.3 while the ECU on the
   bench is actually flashed with build 2.4, using the address-mapping
   model above.
2. Design a test that uses a CHARACTERISTIC write to sweep a braking
   threshold across three values and confirms ABS activation timing
   changes accordingly — including the restore step and why it must
   run even if the test fails partway through (`finally`-style, in
   whatever cleanup mechanism your automation layer offers).
3. A DAQ list configured to sample 200 addresses at 1ms intervals
   saturates the XCP transport's bandwidth on a slow link. Propose two
   ways to reduce load without losing the specific measurement your
   test actually needs.
