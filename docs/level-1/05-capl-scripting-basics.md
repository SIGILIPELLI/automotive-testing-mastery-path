# 05 · CAPL Scripting Basics

**CAPL** (Communication Access Programming Language) is Vector CANoe's
built-in scripting language: C-like syntax, event-driven execution, and
first-class support for CAN/LIN/Ethernet messages as native types. This
module covers the real, documented core of the language — event
handlers, message send/receive, and timers — the building blocks every
later module (test modules, restbus simulation, fault injection) is
built from.

!!! note "About this module"
    The syntax and event-handler names below (`on start`, `on message`,
    `on timer`, `output()`, `setTimer()`, message struct access) are
    genuine, documented CAPL conventions. This course cannot compile or
    execute CAPL against a live CANoe instance — every snippet here is a
    faithful, reviewed example to study, not a transcript of an actual
    run.

## CAPL's execution model: event handlers, not a single main()

Unlike a normal C program with one `main()`, a CAPL program is a
collection of **event procedures** — blocks that CANoe's runtime invokes
when a specific event occurs. The three you'll use constantly:

```c
variables
{
  // Global variables live here, declared once for the whole program.
  msTimer commandTimeout;
  byte    aliveCounter;
}

on start
{
  // Runs once when the measurement (CANoe's term for a test run) begins.
  write("Node simulation started.");
}

on message SensorNodeStatus
{
  // Runs every time a frame matching this message is received.
  write("Coolant temp raw = %d", this.CoolantTemp);
}

on timer commandTimeout
{
  // Runs when the named timer expires.
  write("Command timed out - entering failsafe.");
}
```

- `on start` — fires once at the beginning of a measurement. Typical
  use: initialize variables, arm timers, send an initial state message.
- `on message <Name or ID>` — fires every time a frame matching that
  message (by symbolic name from the loaded database, or by raw ID)
  arrives on the bus. `this` refers to the message that triggered the
  handler.
- `on timer <name>` — fires once when a `msTimer` (millisecond-resolution
  timer) you started with `setTimer()` expires.

Other common handlers: `on key` (keyboard input, useful for manual test
panels), `on preStart`/`on stopMeasurement` (setup/teardown around a
measurement), and `on signal` (fires on a *signal* change rather than a
raw message — useful once a database is loaded).

## Sending a message

CAPL represents each CAN message from the loaded database as a global
struct-like variable, matching its DBC/ARXML definition. You set its
signal fields, then call `output()` to actually transmit it:

```c
variables
{
  message SensorNodeStatus statusMsg;   // declared from the database
  byte aliveCounter;
}

on start
{
  statusMsg.CoolantTemp   = 850;   // raw units - scaling is defined in the DBC
  statusMsg.SupplyVoltage = 13800; // e.g. millivolts, per the signal's DBC scale
  statusMsg.SensorValid   = 1;
  statusMsg.AliveCounter  = aliveCounter;

  output(statusMsg);
  write("Sent SensorNodeStatus, alive=%d", aliveCounter);
}
```

`output()` queues the message for transmission on the bus (or the
simulated bus, if running offline) immediately. Whether the values you
write are "raw" or "physical" depends on how the message was declared —
CANoe supports both raw byte-level messages and signal-based access
where scaling is applied automatically; check which style a given
project uses before assuming units.

## Sending a message periodically

Automotive networks are full of **cyclic messages** — status frames sent
every fixed interval regardless of anything else happening. A minimal
periodic transmitter using `setTimer()` and re-arming itself:

```c
variables
{
  message SensorNodeStatus statusMsg;
  msTimer  cycleTimer;
  byte     aliveCounter;
}

on start
{
  aliveCounter = 0;
  setTimer(cycleTimer, 100);   // fire once, 100 ms from now
}

on timer cycleTimer
{
  statusMsg.CoolantTemp  = 850;
  statusMsg.SensorValid  = 1;
  statusMsg.AliveCounter = aliveCounter;
  output(statusMsg);

  aliveCounter = (aliveCounter + 1) % 16;   // 4-bit alive counter wraps at 16
  setTimer(cycleTimer, 100);                // re-arm for the next cycle
}
```

This pattern — fire, do the work, re-arm — is the standard way to build
a cyclic sender in CAPL; there is no built-in "repeat every N ms"
primitive, so re-arming inside the handler itself is the documented
idiom.

## Reacting to a received message

A command/response pattern, receiving a command frame and responding
with an acknowledgment on a different message ID:

```c
variables
{
  message MotorCommand   cmdMsg;      // received
  message MotorStatus    statusReply; // sent in response
}

on message MotorCommand
{
  write("Received MotorCommand: TargetSpeed=%d", this.TargetSpeed);

  if (this.TargetSpeed > 4000)
  {
    write("Rejecting out-of-range TargetSpeed request");
    statusReply.CommandAccepted = 0;
  }
  else
  {
    statusReply.CommandAccepted = 1;
  }

  output(statusReply);
}
```

`this` is only valid inside the `on message` block it belongs to — it
gives you read access to every signal/byte of the specific frame that
triggered the handler.

## A signal-loss timeout pattern

A very common real test/simulation pattern: detect that a periodic
message has stopped arriving, and react. This combines `on message` (to
reset a timeout) with `on timer` (to detect its expiry):

```c
variables
{
  msTimer signalTimeout;
}

on start
{
  setTimer(signalTimeout, 300);   // arm: 300 ms of silence = fault
}

on message SensorNodeStatus
{
  cancelTimer(signalTimeout);     // frame arrived - message is alive
  setTimer(signalTimeout, 300);   // re-arm the watchdog
}

on timer signalTimeout
{
  write("SensorNodeStatus timed out - no frame for 300 ms");
  // In a real test module this would set a FAIL verdict (Level 2, Module 2)
}
```

This exact shape — cancel-and-rearm on every valid receipt, react on
expiry — is how CAPL simulations and test modules implement the kind of
signal-timeout failsafe logic described in Module 1's power-window
example.

## Cheat sheet

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

## Exercise

Design (in CAPL, following the patterns above — you do not need to run
it) a script for a simulated **wheel speed sensor node** that:

1. Sends a `WheelSpeedStatus` message every 20 ms containing a speed
   value and a 4-bit alive counter that increments and wraps correctly.
2. Listens for a `DiagnosticResetRequest` message and, on receiving it,
   resets its alive counter to 0 and writes a message to the Write
   window confirming the reset.
3. Implements a 100 ms timeout watchdog on an incoming `BrakeApplied`
   message from another node — if it stops arriving, write a fault
   message and set a flag variable `brakeSignalLost` to 1; clear the
   flag the moment the message resumes.

Write out the full `variables` block and all three `on ...` handlers.
Then, in one paragraph, explain what would happen to your alive counter
logic if two separate `on timer` handlers in the same program both
happened to call `output()` on the exact same message — is that a
realistic mistake, and why or why not?
