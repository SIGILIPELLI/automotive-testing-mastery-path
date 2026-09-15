---
description: "Advanced CAPL Scripting — Level 1 Module 5 covered CAPL's event model — on start, on message, on timer, output(). This module builds on that with the…"
---

# 01 · Advanced CAPL Scripting

Level 1 Module 5 covered CAPL's event model — `on start`, `on message`,
`on timer`, `output()`. This module builds on that with the constructs
you need once a script grows past a toy demo: functions, arrays and
structured data, signal-level access, environment variables, and the
`testcase`/`testfunction` building blocks that Module 2 turns into full
test modules.

!!! note "About this module"
    All syntax below (function declarations, `sysvar`, arrays, `strncpy`
    and friends, `@sysvar` change handlers) is genuine, documented CAPL.
    As in Level 1, nothing here was compiled against a live CANoe
    instance — treat every snippet as a reviewed reference, not a
    captured run.

## User-defined functions

CAPL supports ordinary functions, declared outside any event block, with
C-like typing:

```c
variables
{
  byte lastAliveCounter;
}

byte computeChecksum(byte b0, byte b1, byte b2, byte b3)
{
  // Simple additive checksum - many real ECUs use CRC8/CRC16 instead,
  // but the calling convention is the same either way.
  return (b0 + b1 + b2 + b3) & 0xFF;
}

on message SensorNodeStatus
{
  byte calc;
  calc = computeChecksum(this.byte(0), this.byte(1), this.byte(2), this.byte(3));
  if (calc != this.byte(4))
  {
    write("Checksum mismatch: calc=%d, received=%d", calc, this.byte(4));
  }
}
```

Functions can return `void`, `int`, `long`, `float`, `byte`, `char[]`, or
a `message` type, and can take any of those as parameters. Unlike C,
CAPL has no pointers and no dynamic memory — every array is fixed-size
and declared up front, which matters for the buffer patterns below.

## Arrays and byte buffers

Raw frame payloads are commonly handled as byte arrays, especially for
UDS multi-frame reassembly or CRC calculations that need to walk the
whole payload:

```c
variables
{
  byte rxBuffer[64];
  dword rxLength;
}

on message DiagResponse
{
  dword i;
  rxLength = this.dlc;
  for (i = 0; i < rxLength; i++)
  {
    rxBuffer[i] = this.byte(i);
  }
  write("Captured %d bytes, first byte = 0x%X", rxLength, rxBuffer[0]);
}
```

`this.byte(i)` reads a raw payload byte by index regardless of whether a
database defines named signals for the message — the fallback you reach
for whenever you need generic frame access (diagnostics, unknown/unDBC'd
traffic, checksum/CRC routines).

## System variables (sysvars): state shared across the whole configuration

A **system variable** (`sysvar`) is a named, typed value that lives
outside any one CAPL block and can be read or written from multiple
nodes, panels, and test modules in the same CANoe configuration —
CAPL's mechanism for state that needs to outlive a single script or be
shared with, say, a Panel UI button.

```c
variables
{
  // Declared in the configuration's system variable namespace,
  // referenced here as Namespace::VariableName.
}

on sysvar_update Diag::ResetRequested
{
  if (@Diag::ResetRequested == 1)
  {
    write("Operator requested a diagnostic reset via the panel.");
    @Diag::ResetRequested = 0;   // consume the request
  }
}
```

`@Namespace::Name` reads or writes a sysvar's current value; `on
sysvar_update` fires whenever it changes, regardless of what wrote it.
This is the standard way to let a human-facing panel button or another
CAPL node trigger behavior in your script without a CAN message ever
being involved.

## Signal-level access vs. raw byte access

Once a database (DBC/ARXML) is loaded, CAPL gives you two ways to touch
the same payload:

| Access style | Example | When to use |
|---|---|---|
| Signal-level | `this.CoolantTemp` | Normal use — DBC scaling/offset applied automatically |
| Raw byte | `this.byte(0)` | Checksums, CRCs, generic/undefined payloads, bit-level manipulation |

Mixing the two on the same message is fine and common — read a
checksum byte raw while reading every other field by signal name.

## String and diagnostic-ID formatting helpers

Building UDS requests (Level 1 Module 6) and log messages often needs
string/byte-array conversion:

```c
char logLine[128];

void logDtc(dword dtcCode, byte statusMask)
{
  snprintf(logLine, elcount(logLine),
           "DTC 0x%06X, status mask 0x%02X", dtcCode, statusMask);
  write(logLine);
}
```

`elcount()` returns the declared element count of an array — always
preferred over a hardcoded buffer size, since CAPL arrays are fixed and
a mismatched literal is a classic off-by-one bug source.

## `testcase`/`testfunction`: the bridge to Module 2

CAPL code destined for a CANoe **test module** (not a plain simulation
node) is organized differently — into `testcase` blocks with verdict
calls like `testStepPass()`/`testStepFail()`. Module 2 covers this in
full; the shape to recognize now:

```c
testcase TC_CoolantTempInRange()
{
  testStep("Check coolant temperature signal is within calibration range");
  if (SensorNodeStatus::CoolantTemp >= -400 && SensorNodeStatus::CoolantTemp <= 1500)
  {
    testStepPass("CoolantTemp within [-40.0, 150.0] C");
  }
  else
  {
    testStepFail("CoolantTemp out of range");
  }
}
```

## Cheat sheet

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

## How It Actually Works: why sysvar updates queue instead of racing

`on sysvar_update` looks like it should behave the same as `on message`
— an event fires, a handler runs — and structurally it does, because
Module 5's single-threaded, run-to-completion event queue (Level 1)
governs sysvars exactly the same way it governs messages and timers.
But sysvars have one behavior that trips people up precisely because
they *don't* travel over the bus: **a sysvar write takes effect in
CANoe's shared variable store immediately and synchronously**, while the
resulting `on sysvar_update` notification to every other CAPL block
watching that variable is queued as a separate event, delivered in
FIFO order alongside every pending message and timer event.

The practical consequence: if your node script and a Panel button both
write to `Diag::ResetRequested` in the same instant (a human clicking
the panel while your handler is also about to reset it), there's no
race in the multithreaded sense — one write happens, then the other,
strictly in event-queue order — but the *value* a late `on sysvar_update`
handler reads via `@Diag::ResetRequested` is whatever the variable holds
**at the moment that specific handler runs**, not at the moment the
write that triggered it occurred. If a second write lands on the same
sysvar before the first update notification is processed, only one `on
sysvar_update` event may end up representing two logical changes (CANoe
coalesces rapid updates rather than guaranteeing one notification per
write) — so a script relying on sysvar updates to count discrete events
(rather than reflect current state) will silently undercount under
bursty conditions. This is exactly why the reset-request pattern above
"consumes" the flag by writing it back to 0 inside the handler: it
converts an edge-triggered assumption into explicit level-based state
management, sidestepping the coalescing behavior entirely rather than
depending on one-notification-per-write, which CANoe never actually
promises.

## Exercise

Write a CAPL function `bool isValidAliveCounter(byte previous, byte
current)` that returns true if `current` is exactly `previous + 1`
modulo 16 (matching the 4-bit alive-counter pattern from Level 1 Module
5), and false otherwise. Then write an `on message` handler for
`SensorNodeStatus` that calls this function every time a frame arrives,
using a `variables`-block byte to remember the previous value, and logs
a raw byte dump of the whole payload via `this.byte(i)` in a loop
whenever the counter check fails. Finally, explain in a short paragraph
why raw byte access (rather than signal-level access) is the right
choice for that failure-diagnostics log line specifically.
