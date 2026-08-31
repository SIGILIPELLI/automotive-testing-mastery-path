# 09 · Basic HIL Test Bench Components

Everything through Module 8 has run against pure bus simulation — no
physical ECU, no real wiring, all CAPL and CANoe. **Hardware-in-the-Loop
(HIL)** testing puts a real, physical ECU on a bench, wired to
simulated versions of everything *around* it (sensors, actuators, other
ECUs' signals), so the ECU's actual embedded software runs on its
actual silicon while the rest of the world is emulated. This module
maps the components of a typical HIL rig conceptually, ahead of Level
3's deeper automation work.

!!! note "About this module"
    The component roles and architecture described below reflect
    standard, documented HIL practice across common platforms (dSPACE,
    NI, Vector VT System). No specific rig was used to produce this
    content, and CANoe/CAPL are shown only where they're the genuine
    interface layer — treat every claim about a specific vendor's
    exact configuration as something to verify against that vendor's
    documentation.

## Why HIL exists as a distinct test tier

| Tier | What's real | What's simulated | Typical use |
|---|---|---|---|
| Pure simulation (Modules 1–8) | Nothing physical | Everything, including the "ECU" | Fast logic/protocol tests, run constantly |
| HIL | The ECU itself (its real hardware and embedded software) | Sensors, actuators, other ECUs, the physical plant | Real embedded-software validation without a real vehicle |
| Vehicle test | Everything | Nothing | Final validation, most expensive and slowest tier |

HIL sits deliberately in the middle: catches defects that only show up
in the real embedded binary (timing, memory, real ADC/DAC behavior,
real CAN transceiver quirks) without the cost, safety risk, and
scheduling bottleneck of a full vehicle.

## Core components of a HIL rig

| Component | Role |
|---|---|
| **Real-time simulation computer** | Runs the plant model (e.g. engine dynamics, vehicle dynamics) in hard real time — must complete each simulation step within its fixed time budget, every time, or the ECU sees discontinuous/invalid physics |
| **I/O interface hardware** | Converts between the simulation computer's digital model values and the real electrical signals the ECU expects — analog voltages, PWM, digital I/O, resistor/load emulation |
| **ECU under test** | The real, physical target — unmodified production or pre-production hardware and software |
| **Bus interface (CAN/CAN-FD/LIN/Ethernet)** | Connects the ECU's real bus interfaces to the simulation, often via the same CANoe/CAPL tooling used in pure simulation, now driving physical transceivers |
| **Power supply & load emulation** | Provides the ECU's actual operating voltage/current profile, including realistic transients (cranking dip, load dump) the ECU's power-management logic must survive |
| **Fault-injection unit** | Hardware relays/switches capable of injecting real electrical faults — opens, shorts to ground/battery — onto real wiring, not just simulated bus values (Level 3 Module 6 covers this in depth) |
| **Test automation host** | Runs the actual test scripts/sequences, orchestrating the simulation computer, I/O, and bus interface together |

## Signal path: from model to ECU and back

```
[Plant model, real-time computer]
        │  (digital values, every fixed timestep)
        ▼
[I/O interface: DAC / PWM / digital out]
        │  (real electrical signals)
        ▼
[ECU under test — reads sensors, runs its embedded software]
        │  (real electrical signals: actuator commands, CAN frames)
        ▼
[I/O interface: ADC / digital in]  +  [CAN/LIN transceiver]
        │
        ▼
[Plant model, real-time computer]  (closes the loop)
```

The word "loop" matters: a HIL rig is a **closed loop** — the ECU's
outputs feed back into the plant model, which changes the next set of
inputs the ECU sees. A test rig that only injects fixed, pre-recorded
inputs regardless of the ECU's own outputs is open-loop, and open-loop
rigs cannot test genuine feedback-control behavior (e.g. an ABS ECU
whose braking commands should change the simulated wheel-speed signals
it then reads back).

## CAN interface layer: same tooling, real transceivers

The CAPL/CANoe layer you've used throughout this level doesn't
disappear on a HIL rig — it typically becomes the bus-side test
interface, now talking to a real CAN transceiver connected to the real
ECU, instead of a purely simulated bus:

```c
// Identical CAPL pattern to Module 8's restbus, now driving a real
// transceiver toward a physical ECU on the bench
on start
{
  setTimer(txWheelSpeed, 5);
}

on timer txWheelSpeed
{
  wheelSpeedMsg.FL = ReadPlantModelSignal("WheelSpeed_FL");
  output(wheelSpeedMsg);
  setTimer(txWheelSpeed, 5);
}
```

The conceptual shift is that `ReadPlantModelSignal()` now pulls from a
real-time physics simulation running on separate hardware rather than
a value your test script set arbitrarily — which is exactly why HIL
catches timing and closed-loop defects that pure bus simulation
structurally cannot.

## Real-time determinism: the property pure simulation doesn't need

Pure CANoe/CAPL simulation runs on a general-purpose OS with no hard
real-time guarantee — acceptable because nothing downstream depends on
a simulated signal arriving within a guaranteed microsecond window. A
HIL plant model **must** complete its physics calculation within each
fixed timestep (commonly 1ms or faster for engine/vehicle dynamics) or
the ECU under test receives a stale or skipped value at a moment when
its own control loop is running at full speed — producing artifacts
that look like ECU defects but are actually simulation-fidelity
failures. This is why HIL simulation computers run dedicated real-time
operating systems rather than the same machine that runs your test
scripts.

## Cheat sheet

| Component | Analogy to pure simulation |
|---|---|
| Real-time simulation computer | Replaces "the rest of the world exists only as CAPL logic" with actual physics |
| I/O interface hardware | New — pure simulation has no real electrical signals at all |
| ECU under test | The one thing that was previously *also* simulated, now real |
| Bus interface | Same CANoe/CAPL concepts as Modules 1–8, now on a real transceiver |
| Fault-injection unit | New — real electrical faults, not simulated bad values |
| Closed loop | The defining property distinguishing HIL from open-loop bench testing |

## Exercise

You're scoping a HIL rig to test an electronic power-steering ECU. The
ECU needs: a simulated steering-wheel torque sensor input (analog
voltage), a simulated motor position feedback (digital pulse train),
real CAN traffic for `VehicleSpeed` and `EngineRunning`, and its actual
power-assist output must feed back into a simulated steering-rack model
so subsequent torque-sensor readings reflect the assist actually given.

1. Map each of the four I/O needs above to the correct HIL component
   category from the table.
2. Explain, specifically for the "power-assist output feeds back into
   the steering-rack model" requirement, why this rig cannot be
   open-loop and what would go wrong functionally if it were.
3. Identify one property of the real-time simulation computer that
   would need to be verified (not assumed) before trusting a test that
   measures the ECU's steering-response latency to within a few
   milliseconds.
