# 03 · CAN-FD Basics

Classic CAN (Level 1) tops out at 1 Mbit/s and 8-byte payloads — fine for
most body and chassis signals, tight for the sensor-fusion and ADAS
data that modern ECUs exchange. **CAN-FD** ("Flexible Data-rate") keeps
CAN's arbitration and error-handling model but relaxes both limits, and
it's now the default bus on most new zonal and domain-controller
architectures. This module covers what actually changes at the frame
level, and what that means for test scripts and DBC files you already
know how to write.

!!! note "About this module"
    The frame layout, bit-rate-switching rules, and CAPL/DBC syntax
    below are genuine, documented CAN-FD (ISO 11898-1:2015) and Vector
    tooling behavior. Nothing was captured from a live CANoe/CAN-FD
    transceiver session — treat every snippet as a reviewed reference,
    verify against your own bus before trusting it in a released test
    suite.

## What actually changes

| Property | Classic CAN | CAN-FD |
|---|---|---|
| Max payload | 8 bytes | 64 bytes |
| Arbitration bit rate | up to 1 Mbit/s | up to 1 Mbit/s (unchanged) |
| Data-phase bit rate | same as arbitration | up to 5–8 Mbit/s (implementation-dependent) |
| DLC → byte-length mapping | linear, DLC 0–8 = 0–8 bytes | non-linear above 8 (see table below) |
| CRC | 15-bit | 17-bit (≤16 byte payload) or 21-bit (>16 byte payload) |
| New control bits | — | `FDF` (FD format), `BRS` (bit-rate switch), `ESI` (error state indicator) |

The two ideas to hold onto: **more bytes per frame**, and **two bit
rates in one frame** (slower arbitration phase so bus access still
resolves correctly, faster data phase once one node has won
arbitration and is transmitting alone).

### DLC to length mapping

Below 9 bytes, CAN-FD's DLC field means exactly what classic CAN's
does. Above that, the 4-bit DLC field (still only 16 possible values)
has to represent up to 64 bytes, so the mapping becomes non-linear:

| DLC | Data length |
|---|---|
| 0–8 | 0–8 bytes (same as classic CAN) |
| 9 | 12 bytes |
| 10 | 16 bytes |
| 11 | 20 bytes |
| 12 | 24 bytes |
| 13 | 32 bytes |
| 14 | 48 bytes |
| 15 | 64 bytes |

This table is the single most common source of "off by N bytes" bugs
when someone hand-decodes a CAN-FD trace without tooling: DLC 13 is
**not** 13 bytes, it's 32.

### The new control bits

- **FDF (FD Format)** — set to 1 on a CAN-FD frame; a classic-CAN-only
  node sees this as a reserved bit and typically errors out or ignores
  the frame depending on controller mode, which is exactly why mixed
  classic/FD buses require every node to at least tolerate FD frames
  even if it doesn't decode their extended payload.
- **BRS (Bit Rate Switch)** — set to 1 if the data phase (data field +
  CRC) actually switches to the higher bit rate; a CAN-FD-capable
  frame can still be sent with `BRS = 0`, running the whole frame at
  the arbitration rate — useful for compatibility testing.
- **ESI (Error State Indicator)** — reflects the transmitter's own
  error-active/error-passive state, letting receivers detect a
  degrading node without waiting for a bus-off.

## DBC changes for CAN-FD

Vector's DBC format is extended (not replaced) for CAN-FD via an
`BA_ "VFrameFormat"` attribute on the message, plus support for the
wider `ByteOrder`/length fields. A minimal CAN-FD message definition:

```text
BO_ 1024 SensorFusionFrame: 32 ADAS_ECU
 SG_ ObjectCount : 0|8@1+ (1,0) [0|255] "count" Vector__XXX
 SG_ ObjectDistance_m : 8|16@1+ (0.01,0) [0|655.35] "m" Vector__XXX
 SG_ ObjectVelocity_mps : 24|16@1- (0.01,-100) [-100|100] "m/s" Vector__XXX

BA_ "VFrameFormat" BO_ 1024 1;
```

`VFrameFormat` values you'll see in real DBCs:

| Value | Meaning |
|---|---|
| 0 | StandardCAN |
| 1 | StandardCAN_FD |
| 2 | ExtendedCAN |
| 3 | ExtendedCAN_FD |

A DBC editor (or CANdb++) that doesn't understand this attribute will
still open the file — it just won't validate the 32-byte length against
classic CAN's 8-byte ceiling, so a mis-tagged message is a common
integration bug: the frame transmits fine but a legacy tool silently
truncates or misparses it.

## CAPL for CAN-FD

CAPL's FD support adds an `isFDMsg` framework and dedicated frame type,
plus a `msg.brs`/`msg.esi` accessor pair:

```c
variables
{
  message SensorFusionFrame fdMsg;
}

on start
{
  fdMsg.dlc = 15;        // DLC 15 = 64 bytes, per the table above
  fdMsg.fdf = 1;         // mark as CAN-FD format
  fdMsg.brs = 1;         // request bit-rate switch on the data phase
  fdMsg.byte(0) = 3;     // ObjectCount = 3
}

on timer sendFusionFrame
{
  output(fdMsg);
  setTimer(sendFusionFrame, 20);
}

on message SensorFusionFrame
{
  if (this.fdf == 0)
  {
    write("Warning: received SensorFusionFrame as classic CAN, expected FD");
  }
  write("ObjectCount=%d, ESI=%d, BRS=%d", this.byte(0), this.esi, this.brs);
}
```

A CAPL test module verifying that BRS is actually being used (not just
requested) belongs in your restbus-simulation and signal-validation
suites once real signals depend on the higher data rate — see Module 8
and Module 10 of this level.

## Test implications: what a CAN-FD move breaks if you don't check it

| Risk | Why it happens | Test to catch it |
|---|---|---|
| Legacy node bus-off on FD traffic | A classic-CAN-only controller in "classic" mode treats `FDF=1` as a form error | Inject one FD frame on a bus with a legacy node present, in a bench/simulation test, and confirm the legacy node doesn't error-frame or bus-off |
| Truncated payload in a mixed toolchain | An older DBC consumer or logger only reads 8 bytes | Round-trip a >8-byte signal through every tool in the actual toolchain (CANoe, a data logger, a diagnostic tool) and diff the decoded value |
| Wrong DLC→length assumption | Hand-decoding DLC 9–15 as literal byte counts | Always decode via DBC/CANoe, never by eyeballing DLC above 8; add an assertion test comparing decoded length against the table above |
| BRS requested but not actually achieved | Transceiver/bus doesn't support the higher data-phase rate reliably at the wiring length used | A signal-integrity/timing test (bench, not simulation) measuring actual data-phase bit time — flagged here as needing real hardware, out of scope for a script-only test |

## Cheat sheet

| Concept | Classic CAN | CAN-FD |
|---|---|---|
| Payload | 8 bytes | up to 64 bytes |
| Bit rate | single rate | dual: arbitration + data phase |
| New signaling bits | none | FDF, BRS, ESI |
| DLC 9–15 | not used | maps to 12/16/20/24/32/48/64 bytes |
| DBC tagging | implicit | `BA_ "VFrameFormat"` attribute |
| CAPL access | `msg.byte(n)`, `msg.dlc` | adds `msg.fdf`, `msg.brs`, `msg.esi` |

## How It Actually Works: how a frame can legally change speed mid-transmission

Switching bit rate partway through a single frame sounds like it should
break the bit-timing/sample-point mechanism from Level 1 Module 2 — and
it would, if it happened anywhere arbitrary. It doesn't: the switch is
only permitted at two precisely defined, fixed points, and CAN-FD adds
extra **resynchronization edges** exactly there to make the transition
safe.

The BRS bit itself is transmitted at the *arbitration* rate, immediately
followed by a fixed-stuff bit and the **FD CRC delimiter's leading
edge**, which every receiver uses as a hard resynchronization point
before sampling the first data-phase bit — this is why the standard
mandates that edge be non-stuffable and always present, unlike ordinary
bit-stuffing which is content-dependent. The reverse transition, back
down to the arbitration rate for the ACK/EOF/IFS fields at the end of
the frame, gets the same treatment at the CRC delimiter on the way out.
Between those two edges, every node on the bus — including one that
lost arbitration and is only listening — has already agreed on exactly
where the data phase begins and ends, because arbitration itself
(Level 1 Module 3) only ever happens at the slower, common rate; by the
time BRS appears, exactly one node is driving the bus and the receivers
have nothing left to arbitrate, only to stay synchronized to.

The wider CRC (17-bit for payloads ≤16 bytes, 21-bit above that,
computed with different generator polynomials than classic CAN's 15-bit
one) exists because burst-error detection guarantees scale with message
length: a 15-bit CRC's protection weakens on a 64-byte payload relative
to an 8-byte one, so CAN-FD widens the check specifically to hold the
same practical undetected-error probability at the much larger frame
sizes Module 3's payload table allows. This is the concrete reason a
"BRS requested but not achieved" fault (this module's test-implications
table) is dangerous specifically at the data-phase/arbitration-phase
boundary: a transceiver or wiring length that can't actually settle to
the higher rate in time corrupts bits right around that resynchronization
edge, and the wider CRC is what still catches it when a shorter classic
CRC might not.

## Exercise

A supplier hands you a DBC with a new message `BO_ 2048 BatteryPackStatus`
that has a byte length of 24 and no `VFrameFormat` attribute at all.

1. What does the missing attribute imply about how a strict CAN-FD-aware
   tool will treat this message, and why is a 24-byte length combined
   with no FD tag an inconsistency worth flagging back to the supplier?
2. Write the DLC value (from the mapping table) that a CAN-FD frame
   carrying exactly 24 bytes of payload must use.
3. Sketch a short CAPL `on message BatteryPackStatus` handler that logs
   a warning if the frame's `fdf` bit is 0 despite its declared 24-byte
   length — explain in a comment why that combination should never
   happen on a correctly configured bus.
