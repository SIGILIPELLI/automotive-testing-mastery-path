# 06 · Fault Injection Testing

Module 4 tested safety mechanisms by manipulating signal *values* on
the bus — a valid technique, but it can't reach every fault class a
real ECU must survive. This module covers the broader taxonomy of
fault injection — electrical, communication, and software-level — and
where each technique's limits are, especially the limits of what CAPL
alone can simulate.

!!! note "About this module"
    No physical fault-injection hardware (relay boxes, breakout boxes,
    programmable power supplies) was used to produce this content. The
    taxonomy and CAPL patterns below reflect documented HIL/fault-
    injection practice — treat every snippet as an architectural
    reference, and note explicitly where real hardware is required and
    CAPL cannot substitute.

## The fault taxonomy

| Class | Example | Can CAPL alone inject it? |
|---|---|---|
| Signal-level (message content) | A CAN signal reports an implausible value | Yes — `setSignal()` on the simulated node |
| Communication-level | A frame stops arriving, arrives late, or is malformed (bad CRC, bad DLC) | Partially — CANoe can suppress/delay frames and corrupt payload; some corruption (physical-layer bit errors) needs a real network fault injector |
| Electrical-level | A sensor line opens (open circuit), shorts to ground, shorts to battery voltage | No — needs a physical relay/switch box (a "breakout box") wired between the ECU and its harness |
| Power-level | Voltage sag, brownout, ignition-cycle glitch during a write operation | No — needs a programmable power supply capable of scripted voltage profiles |
| Software/timing-level | A task misses its deadline, a watchdog isn't fed | Sometimes, if the ECU exposes a debug/test hook; otherwise needs instrumented target debugging, out of scope for bus-level testing |

The taxonomy matters because a test plan that only ever exercises
signal-level faults (the easiest to script) can systematically miss
entire classes of real-world failure — an open sensor wire behaves
very differently from that same sensor reporting an implausible value,
and a safety mechanism designed for one may not catch the other.

## Communication-level fault injection in CAPL

```c
// Illustrative — CANoe's fault-injection block/CAPL functions vary
// by version; verify exact function names against your installation.
testcase tc_MissingFrameTriggersTimeout()
{
  testCaseTitle("ECU flags communication loss when expected frame stops arriving");
  dword t0, t1;

  BlockMessage(WheelSpeedFL); // suppress transmission entirely, simulating a dead sender
  t0 = timeNowNS() / 1000000;

  testWaitForSignal(CommTimeoutFault_WheelSpeedFL, 1, 500);
  t1 = timeNowNS() / 1000000;

  testStepCheck("comm timeout fault flagged within spec", (t1 - t0) <= 500);
  UnblockMessage(WheelSpeedFL); // always restore normal traffic before the next testcase
}

testcase tc_CorruptedDlcRejected()
{
  testCaseTitle("Frame with wrong DLC vs. DBC definition is rejected, not misparsed");
  message WheelSpeedFL msg;
  msg.DLC = 4; // DBC defines 8 — deliberately wrong
  msg.byte(0) = 0xFF;
  output(msg);

  testWaitForTimeout(100);
  testStepCheck("no valid wheel speed update accepted from malformed frame",
                 getSignal(WheelSpeedFL_Value) == gLastKnownGoodValue);
}
```

`tc_MissingFrameTriggersTimeout` restores normal traffic explicitly at
the end — the same "always undo what you injected" discipline as
Module 4's calibration restore, generalized to any fault-injection
technique: leaving a blocked message in place corrupts every
subsequent testcase in the run.

## Why electrical-level faults need real hardware

A sensor reporting `0xFF 0xFF` (an implausible value CAPL can easily
simulate) and a sensor whose signal wire has physically opened produce
different electrical symptoms an ECU's front-end circuitry may
distinguish — pull-up/pull-down behavior, voltage rail sag, ADC
saturation versus floating input. A safety mechanism specifically
designed to detect an open circuit via electrical characteristics
(not just implausible values) cannot be validated by CAPL signal
manipulation alone — it requires a breakout box or fault-injection
relay bank physically wired into the ECU's harness, under HIL rig
control, to open/short/ground specific pins on command.

```text
Typical breakout-box fault types (hardware-level, NOT reachable from CAPL):
  - Open circuit      (pin disconnected)
  - Short to ground   (pin pulled to 0V)
  - Short to battery  (pin pulled to Vbatt, ~12V/24V)
  - Short to adjacent pin (cross-short between two harness pins)
```

A well-structured HIL test plan (Module 10) explicitly separates test
cases that a CAPL-only rig can execute from those that require this
class of hardware, so gaps in physical fault-injection coverage are
visible and tracked rather than silently absent.

## Fault injection campaigns: coverage over one-off checks

A mature fault-injection practice doesn't test faults ad hoc — it
maintains a **fault matrix**: every safety-relevant signal or
interface, crossed with every applicable fault type, with a tracked
pass/fail/not-yet-tested status:

| Signal/Interface | Implausible value | Frame timeout | Open circuit | Short to ground | Short to battery |
|---|---|---|---|---|---|
| WheelSpeedFL | Tested — Pass | Tested — Pass | Not yet tested | Not yet tested | Not yet tested |
| BrakePedalPosition | Tested — Pass | Tested — Fail (bug filed) | Tested — Pass | Not yet tested | N/A (digital signal) |

This matrix is itself a traceability artifact (Module 8) and typically
maps directly to entries in the safety case's FMEA (Failure Mode and
Effects Analysis) — each cell should trace back to a specific failure
mode the FMEA identified as needing a detection mechanism.

## Cheat sheet

| Concept | Key point |
|---|---|
| Fault taxonomy | Signal, communication, electrical, power, software/timing — each needs different tooling |
| CAPL's ceiling | Signal and much communication-level faults; not electrical or power-level |
| Restore after injecting | Every fault injection needs an explicit undo before the next testcase |
| Fault matrix | Track signal × fault-type coverage explicitly, tied to the FMEA |

## Exercise

1. For `BrakePedalPosition` in the fault matrix, explain why "short to
   ground" and "short to battery" are meaningful hardware-level faults
   to test but "N/A" is marked for one of them if the signal is
   digital (0/1) rather than analog — under what physical
   circumstance would a digital signal's short-to-battery still be
   meaningful to test?
2. `tc_CorruptedDlcRejected` assumes `gLastKnownGoodValue` is tracked
   elsewhere. Write the CAPL to maintain it correctly (updated only on
   an ECU-*acknowledged* valid frame, not merely a well-formed one).
3. Given the taxonomy table, design a one-page test plan section
   listing, for a fictional tire-pressure-sensor ECU, which fault
   types you'd request breakout-box hardware for and which you can
   cover with CAPL alone — with a one-line justification per row.
