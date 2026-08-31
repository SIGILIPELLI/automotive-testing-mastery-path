# 04 · LIN Bus Basics

Not every signal in a car needs CAN's arbitration, error handling, or
wiring cost. Window lifters, mirror motors, seat controllers, rain
sensors — low-speed, low-criticality, cost-sensitive nodes — typically
sit on **LIN** (Local Interconnect Network), a single-wire,
master/slave bus that trades CAN's peer-to-peer arbitration for a much
cheaper physical layer. This module covers LIN's frame model and the
CAPL/tooling differences from the CAN work in Level 1 and this level's
earlier modules.

!!! note "About this module"
    LIN 2.x frame structure, checksum rules, and the CAPL syntax below
    are genuine, documented behavior (LIN Specification 2.2A). Nothing
    here was captured from a live LIN bus — treat every snippet as a
    reviewed reference and verify against your own hardware.

## Master/slave, not peer-to-peer

CAN nodes all arbitrate for the bus independently. LIN has exactly one
**master** node, which owns a **schedule table** of when every message
gets sent, and one or more **slave** nodes that only ever respond when
the master asks. There is no arbitration, no collision handling, and no
node priority beyond "whatever order the master's schedule puts it in."

This has direct test consequences: you cannot "just transmit" a LIN
frame the way you can a CAN frame from a test node — you either *are*
the master (running the schedule) or you're a slave responding to a
header the master sends, and CANoe's LIN interface models this
distinction explicitly (Master simulation vs. Slave simulation nodes).

## Frame anatomy

A LIN frame is two parts sent by two different roles:

1. **Header** (always sent by the master): break field, sync byte,
   protected identifier (PID).
2. **Response** (sent by whichever node — master or a slave — owns
   that identifier): 1–8 data bytes, then a checksum.

| Field | Sent by | Size | Purpose |
|---|---|---|---|
| Break | Master | ≥13 bit-times low | Signals start of frame |
| Sync | Master | 0x55 | Lets slaves calibrate their baud rate |
| Protected ID (PID) | Master | 6-bit ID + 2 parity bits | Identifies which frame; parity catches bit errors on the ID itself |
| Data | Header owner | 1–8 bytes | The payload |
| Checksum | Same node as data | 1 byte | Classic (data-only) or enhanced (data+PID), depending on LIN version |

### Checksum: classic vs. enhanced

LIN 1.x used a **classic checksum** (sum of data bytes only). LIN 2.x
introduced the **enhanced checksum**, which folds the protected ID into
the sum as well, catching a class of errors the classic checksum
missed. A LIN network with mixed-version nodes must configure each
frame's checksum type individually to match what that frame's
publisher/subscriber pair actually implements — a very common
integration bug is one node using enhanced and another expecting
classic for the same frame ID, which silently drops every frame as a
checksum failure.

## CAPL for LIN

CAPL treats LIN messages similarly to CAN messages but through
LIN-specific objects (`linFrame`, `on linFrame`, `LinFrameSendRequest`):

```c
variables
{
  linFrame MirrorControl mcFrame;
}

on preStart
{
  // As LIN master simulation: request this frame be sent by the schedule
  LinFrameSendRequest(mcFrame);
}

on linFrame MirrorControl
{
  write("MirrorControl: X=%d, Y=%d, checksum_ok=%d",
        this.byte(0), this.byte(1), this.CheckSumOK);
}

on linSchedulerModeChange
{
  write("Schedule table changed to: %d", this.mode);
}
```

Key differences from the CAN handlers in Level 1 Module 5 and this
level's Module 1:

- `this.CheckSumOK` is a LIN-specific flag your test scripts should
  assert on directly — a frame with a bad checksum still arrives at the
  `on linFrame` handler, unlike a CAN frame that fails CRC (which is
  typically dropped at the controller level before your script sees it).
- LIN frames are driven by a **schedule table**, configured separately
  from the CAPL script (in the LDF — LIN Description File — and CANoe's
  scheduler configuration), not by ad-hoc `output()` calls at arbitrary
  times.

## LDF vs. DBC

LIN's equivalent of a DBC is the **LDF** (LIN Description File) —
similar goal (signals, frames, scaling, nodes) but LIN-specific syntax
and mandatory schedule-table definitions:

```text
LIN_description_file;
LIN_protocol_version = "2.2A";
LIN_language_version = "2.2";
LIN_speed = 19.2 kbps;

Nodes {
  Master: BodyControlModule, 5 ms, 0.1 ms;
  Slaves: MirrorLeft, MirrorRight;
}

Signals {
  MirrorX: 8, 0, MirrorLeft, BodyControlModule;
  MirrorY: 8, 0, MirrorLeft, BodyControlModule;
}

Frames {
  MirrorControl: 0x10, MirrorLeft, 2 {
    MirrorX, 0;
    MirrorY, 8;
  }
}

Schedule_tables {
  Normal {
    MirrorControl delay 10 ms;
  }
}
```

A test suite that reads a DBC but ignores the corresponding LDF for a
LIN sub-network will miss schedule-table timing entirely — the LDF is
the source of truth for *when* a LIN signal actually updates, which a
DBC has no concept of.

## Test implications

| Risk | Cause | Test approach |
|---|---|---|
| Silent data drop between mismatched nodes | Classic vs. enhanced checksum mismatch | Assert `CheckSumOK` on every received frame in a restbus/simulation test, not just decode the bytes |
| Slave not responding | Slave not present in the active schedule table, or wrong sleep/wake state | Verify schedule table membership and node wake state before asserting on a slave's response |
| Wrong signal timing assumption | LDF schedule interval treated like a CAN cycle time | Confirm actual update interval against the LDF's schedule table, not an assumed fixed rate |
| Bus stuck after a slave error | LIN has weaker built-in error recovery than CAN | Include a wake-up/sleep-cycle test (`go-to-sleep` frame ID `0x3C`, wake-up via bus activity) in your test plan |

## Cheat sheet

| Concept | CAN | LIN |
|---|---|---|
| Bus access | Multi-master, arbitration | Single master, schedule-driven |
| Physical layer | 2-wire differential | 1-wire, ground-referenced |
| Frame structure | ID + DLC + data + CRC | Break/Sync/PID (master) + data + checksum (header owner) |
| Database file | DBC | LDF |
| Checksum types | Single (15-bit CRC) | Classic vs. Enhanced (version-dependent) |
| CAPL object | `message` | `linFrame` |

## Exercise

A body-control LIN network has a `WindowControl` frame owned by the
`WindowMotor` slave, LIN 2.1 enhanced checksum, on a 10 ms schedule
slot.

1. Explain, in terms of header vs. response, exactly which bytes the
   master sends and which bytes the slave sends for this frame.
2. A field report says `WindowControl` values are read correctly about
   half the time and drop the rest. List two plausible causes rooted in
   this module (checksum type, schedule table, wake state) and how you
   would distinguish between them using `CheckSumOK` and schedule-table
   inspection.
3. Write a CAPL `on linFrame WindowControl` handler that counts and
   reports the checksum failure rate over a 1000-frame sample.
