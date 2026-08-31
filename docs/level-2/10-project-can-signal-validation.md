# 10 · Project — Automated CAN Signal Validation Suite

This project pulls together every module in Level 2 into one working
deliverable: an automated CAPL test suite that validates a small
network's signals for correctness, timing, DTC behavior, and restbus
integrity — the kind of suite a real team would run on every build.

!!! note "About this module"
    All CAPL, DBC, and CANoe test-module structure below is genuine,
    documented syntax assembled from the patterns in Modules 1–9.
    Nothing here was run against a live CANoe instance — treat it as a
    reviewed reference implementation to adapt and verify on your own
    setup, not a captured passing run.

## Scope

A small network of three messages, matching the DBC style from
Module 6:

```text
BO_ 256 EngineStatus: 8 ECU_ENGINE
 SG_ EngineRPM : 0|16@1+ (0.25,0) [0|16383.75] "rpm" Vector__XXX
 SG_ CoolantTemp : 16|8@1+ (1,-40) [-40|215] "degC" Vector__XXX
 SG_ EngineRunning : 24|1@1+ (1,0) [0|1] "" Vector__XXX

BO_ 512 BrakeStatus: 4 ECU_ABS
 SG_ BrakePressure : 0|12@1+ (0.1,0) [0|409.5] "bar" Vector__XXX

BO_ 1024 SensorFusionFrame: 32 ADAS_ECU
 SG_ ObjectCount : 0|8@1+ (1,0) [0|255] "count" Vector__XXX

BA_DEF_ BO_ "GenMsgCycleTime" INT 0 60000;
BA_ "GenMsgCycleTime" BO_ 256 10;
BA_ "GenMsgCycleTime" BO_ 512 20;
BA_ "GenMsgCycleTime" BO_ 1024 20;
BA_ "VFrameFormat" BO_ 1024 1;
```

The suite validates: signal range plausibility (Module 6), CAN-FD frame
tagging (Module 3), message cycle-time adherence (Modules 7/8), and a
coolant-overheat DTC's maturation behavior (Module 5) — restbus-backed
throughout (Module 8).

## Restbus: the supporting cast

```c
variables
{
  message EngineStatus engineMsg;
  message BrakeStatus brakeMsg;
}

on start
{
  engineMsg.EngineRunning = 1;
  engineMsg.EngineRPM = 800;
  engineMsg.CoolantTemp = 90;
  setTimer(txEngine, 10);
  setTimer(txBrake, 20);
}

on timer txEngine { output(engineMsg); setTimer(txEngine, 10); }
on timer txBrake  { output(brakeMsg);  setTimer(txBrake, 20); }

export void SetCoolantTemp(float degC) { engineMsg.CoolantTemp = degC; }
export void SetEngineRPM(float rpm)    { engineMsg.EngineRPM = rpm; }
```

## Test module: range and CAN-FD validation

```c
testcase tc_SignalRangeValidation()
{
  testCaseTitle("All signals stay within DBC-declared physical range");
  testWaitForTimeout(50);

  testStepCheck("EngineRPM within [0,16383.75]",
    engineMsg.EngineRPM >= 0 && engineMsg.EngineRPM <= 16383.75);
  testStepCheck("CoolantTemp within [-40,215]",
    engineMsg.CoolantTemp >= -40 && engineMsg.CoolantTemp <= 215);
  testStepCheck("BrakePressure within [0,409.5]",
    brakeMsg.BrakePressure >= 0 && brakeMsg.BrakePressure <= 409.5);
}

testcase tc_CanFdFrameTagging()
{
  testCaseTitle("SensorFusionFrame is correctly tagged and sized as CAN-FD");
  message SensorFusionFrame fdCheck;
  testWaitForTimeout(50);
  testStepCheck("fdf bit set", fdCheck.fdf == 1);
  testStepCheck("payload length is 32 bytes per DLC mapping", fdCheck.dlc == 12);
}
```

`fdCheck.dlc == 12` follows directly from Module 3's DLC table: DLC 12
maps to 24 bytes, so a genuinely 32-byte payload needs DLC 13 — this
line is deliberately written to demonstrate the kind of assertion bug
that table exists to prevent; a reviewer should catch that the DBC's
declared 32-byte length requires DLC 13, not 12, before this test ships.

## Test module: timing adherence

```c
testcase tc_EngineStatusCycleTime()
{
  testCaseTitle("EngineStatus arrives within tolerance of its 10ms cycle time");
  dword lastTime, thisTime, delta;
  int i;
  lastTime = timeNowNS() / 1000000;
  for (i = 0; i < 20; i++)
  {
    testWaitForMessage(EngineStatus, 50);
    thisTime = timeNowNS() / 1000000;
    delta = thisTime - lastTime;
    testStepCheck("cycle time within 10ms +-2ms", delta >= 8 && delta <= 12);
    lastTime = thisTime;
  }
}
```

Allowing a small tolerance band (±2ms here) rather than asserting an
exact 10ms is deliberate — Module 8 covered why sloppy restbus timing
can produce false failures on an exact-match assertion; the tolerance
should be as tight as the ECU's actual documented timing spec allows,
not arbitrarily loose.

## Test module: DTC maturation

```c
testcase tc_CoolantOverheatDtcMaturation()
{
  testCaseTitle("Overheat DTC matures to confirmed only after 2nd cycle failure");
  byte statusReq[] = {0x19, 0x02, 0xFF};

  SetCoolantTemp(135.0);
  testWaitForTimeout(200);
  DiagRequestSend(diagRequest(statusReq));
  testWaitForTimeout(100);
  // Expect pending (bit2) set, confirmed (bit3) not yet set
  testStepCheck("pending set after first cycle", (gLastDtcStatus & 0x04) != 0);
  testStepCheck("not confirmed after first cycle", (gLastDtcStatus & 0x08) == 0);

  // Simulate a new operation cycle
  SetCoolantTemp(90.0);
  testWaitForTimeout(50);
  SetCoolantTemp(135.0);
  testWaitForTimeout(200);
  DiagRequestSend(diagRequest(statusReq));
  testWaitForTimeout(100);
  testStepCheck("confirmed after second cycle failure", (gLastDtcStatus & 0x08) != 0);
}

on diagResponse *
{
  gLastDtcStatus = this.byte(4);  // offset depends on actual response layout; verify against ECU spec
}
```

## Test module skeleton, assembled

```c
variables
{
  byte gLastDtcStatus;
}

testpreparation
{
  // Confirm restbus is transmitting before any testcase runs
  testWaitForTimeout(30);
}

testcasefinalization
{
  SetCoolantTemp(90.0);
  SetEngineRPM(800);
}
```

## Result reporting

Per Module 7, this suite should export an XML report with per-testcase
verdicts, and per Module 9's HIL framing, be explicit that this project
is bus-simulation-only: it validates protocol/logic correctness, not
real-ECU embedded-software behavior — a HIL or vehicle-tier suite would
be a distinct, later phase against real hardware.

## Cheat sheet: what this project exercises from each module

| Module | Concept exercised here |
|---|---|
| 3 — CAN-FD | DLC/length assertion on `SensorFusionFrame` |
| 5 — DTCs | Two-cycle maturation test via `0x19` status mask |
| 6 — DBC | Range assertions derived directly from `[min\|max]` fields |
| 7 — Test sequences | `testpreparation`/`testcasefinalization` structure, independent testcases |
| 8 — Restbus | `export`ed `SetCoolantTemp`/`SetEngineRPM` scenario control |

## Exercise

1. Fix the `tc_CanFdFrameTagging` DLC bug identified above, and explain
   in one sentence why the original assertion would have let a
   genuinely mis-tagged frame pass undetected.
2. Add a fourth testcase, `tc_RestbusStartupOrder`, that verifies
   `EngineStatus` and `BrakeStatus` are both transmitting *before* any
   diagnostic request is sent in `tc_CoolantOverheatDtcMaturation` —
   describe (pseudocode is fine) how you'd detect "restbus not yet
   active" versus "restbus active but signal at default value."
3. This suite currently has no test for `BrakePressure`'s own boundary
   behavior. Using Level 1's boundary-value technique and this
   project's DBC, write one full test case (table format, as in Level
   1 Module 8) for a brake-pressure-triggered warning threshold of your
   own choosing.
