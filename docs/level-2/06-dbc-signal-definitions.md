# 06 · Database Files (DBC) & Signal Definitions

Every module so far has referenced "the DBC" as the source of truth for
signal names, scaling, and ranges without dissecting the file itself.
This module opens it up: the actual syntax, the fields most likely to
be misread, and the review checklist that catches the DBC bugs which
cause the most confusing downstream test failures.

!!! note "About this module"
    The DBC syntax below matches Vector's widely-adopted (if informally
    specified) DBC format, as consumed by CANoe, CANdb++, and most
    open-source parsers (e.g. `cantools`). Nothing here was generated
    from a live tool session — cross-check against your actual DBC
    editor's export before relying on exact formatting.

## Anatomy of a DBC file

```text
VERSION ""

NS_ :
  NS_DESC_
  CM_
  BA_DEF_

BS_:

BU_: BCM ECU_ENGINE ECU_ABS

BO_ 256 EngineStatus: 8 ECU_ENGINE
 SG_ EngineRPM : 0|16@1+ (0.25,0) [0|16383.75] "rpm" BCM
 SG_ CoolantTemp : 16|8@1+ (1,-40) [-40|215] "degC" BCM,ECU_ABS
 SG_ EngineRunning : 24|1@1+ (1,0) [0|1] "" Vector__XXX

BO_ 512 BrakeStatus: 4 ECU_ABS
 SG_ BrakePressure : 0|12@1+ (0.1,0) [0|409.5] "bar" BCM

CM_ SG_ 256 EngineRPM "Crankshaft speed, updated every 10ms.";
CM_ BO_ 256 "Engine status broadcast, cycle time 10ms.";

BA_DEF_ SG_ "GenSigStartValue" FLOAT 0 100000;
BA_ "GenSigStartValue" SG_ 256 EngineRPM 0;

VAL_ 256 EngineRunning 0 "Off" 1 "Running";
```

| Block | Purpose |
|---|---|
| `BU_` | Node (ECU) list |
| `BO_` | Message: ID, name, DLC, transmitting node |
| `SG_` | Signal: start bit, length, byte order, sign, scale/offset, min/max, unit, receiving node(s) |
| `CM_` | Comments — often carries the *real* spec (units caveats, update conditions) that the raw fields don't capture |
| `BA_DEF_` / `BA_` | Custom attributes (cycle time, start value, FD frame format from Module 3) |
| `VAL_` | Enumeration — maps raw integer values to human-readable states |

## Decoding a signal by hand

Take `CoolantTemp : 16\|8@1+ (1,-40) [-40\|215] "degC"`:

| Field | Value | Meaning |
|---|---|---|
| Start bit | 16 | Bit 16 of the 64-bit (8-byte) frame |
| Length | 8 | 8 bits wide |
| Byte order | `@1` | Little-endian (Intel); `@0` would be big-endian (Motorola) |
| Sign | `+` | Unsigned |
| Scale, offset | `(1, -40)` | physical = raw × 1 + (−40) |
| Range | `[-40\|215]` | Valid physical range, for validation/plausibility checks |
| Unit | `"degC"` | Celsius |

A raw byte value of `0x64` (100 decimal) at bit 16 decodes to
`100 × 1 + (−40) = 60 °C`. This is exactly the calculation Level 1's
coolant boundary test (equivalence partitioning module) depended on
without spelling out the arithmetic — now you can derive any test
vector's raw bytes directly from a target physical value instead of
guessing.

### Byte order is the most common decode mistake

`@1+` (Intel/little-endian) and `@0+` (Motorola/big-endian) place the
same start-bit number in physically different locations within the
frame's byte layout. Hand-decoding a Motorola signal using Intel rules
(or vice versa) produces a value that's wrong but often still
*plausible-looking* — the classic false-negative trap where a test
"passes" by coincidence on one value and silently breaks on another.
Always decode via your DBC-aware tool (CANoe, `cantools`), never by
manual bit-shifting, once byte order is in play.

## Review checklist for a new or changed DBC

| Check | Why it matters |
|---|---|
| Every signal's `[min\|max]` matches its documented physical spec | A mismatched range makes plausibility-check tests (Level 1) meaningless |
| Scale/offset arithmetic round-trips cleanly at the range boundaries | Catches off-by-one rounding at exactly the min/max raw values |
| No two signals in the same message overlap start-bit ranges | A silent overlap corrupts both signals' decoded values |
| Byte order is consistent with the transmitting ECU's documented convention | Mixed byte orders within one supplier's messages is a frequent copy-paste error |
| `VAL_` enumerations cover every raw value the signal can actually carry | An undocumented raw value (e.g. `0b11` meaning "invalid/error") left out of `VAL_` gets silently treated as data by naive tooling |
| Comments (`CM_`) describe update conditions, not just restate the signal name | "CoolantTemp" as a comment says nothing; "Updated only when EngineRunning=1" is the note that prevents someone from writing a test that reads it at the wrong time |
| Cycle time attribute matches the transmitting node's actual schedule | A stale cycle-time attribute breaks timing-based test assertions (Level 1's `>= 100ms` example) |

## Signal overlap: a concrete bug

```text
BO_ 768 BadExample: 8 Vector__XXX
 SG_ SignalA : 0|12@1+ (1,0) [0|4095] "" Vector__XXX
 SG_ SignalB : 8|12@1+ (1,0) [0|4095] "" Vector__XXX
```

`SignalA` occupies bits 0–11, `SignalB` occupies bits 8–19 — they
overlap on bits 8–11. Any write to one corrupts the other. This is
exactly the kind of defect that a **bit-map review** (laying every
signal in a message out against its byte/bit grid before sign-off)
catches immediately but a value-only spot-check test completely misses,
because each signal can still decode to a plausible-looking number in
isolation.

## Using the DBC directly in a test script

```c
// CAPL: signals are addressed by name, but always cross-check
// against the DBC's declared scale/offset when computing expected raw values
on message EngineStatus
{
  float coolantC = this.CoolantTemp;      // CAPL applies scale/offset automatically
  if (coolantC < -40 || coolantC > 215)
  {
    testStepFail("CoolantTemp out of DBC-declared physical range");
  }
}
```

Relying on CAPL's automatic scaling (rather than hand-computing raw
bytes) is safer for day-to-day test logic — but understanding the raw
math above is still necessary whenever you're constructing bus-level
test data manually (bit-banging a malformed frame for negative testing,
for instance), since a deliberately invalid frame often needs raw byte
values that no signal-aware API will let you set "incorrectly" on
purpose.

## Cheat sheet

| DBC element | Test relevance |
|---|---|
| `SG_` start bit + length + byte order | Must match exactly to decode/encode a signal correctly |
| Scale/offset | physical = raw × scale + offset — derive test vectors from this |
| `[min\|max]` | Drives plausibility/boundary test partitions |
| `VAL_` | Enumerated states — verify every raw value is either mapped or explicitly "reserved" |
| `CM_` | Often the only place real update-timing/precondition info lives |
| Bit-map overlap | Not caught by value-only tests — needs an explicit layout review |

## Exercise

Given `SG_ FuelLevel : 32|8@1+ (0.5,0) [0|100] "%" Vector__XXX` in an
8-byte message:

1. Compute the raw byte value that encodes a physical fuel level of
   37.5%.
2. The signal's declared range is `[0|100]` with scale 0.5 — what is
   the maximum raw value that stays within range, and what physical
   value would an 8-bit field's true maximum raw value (255) decode to?
   Explain why that out-of-declared-range possibility is exactly what a
   plausibility test (Level 1) needs to guard against.
3. A colleague proposes adding a second signal
   `SG_ FuelLevelValid : 40|1@1+ (1,0) [0|1] ""` starting at bit 40.
   Confirm there's no bit overlap with `FuelLevel` (bits 32–39) and
   explain, referencing the review checklist above, one more DBC-level
   check you'd still want to run before approving this addition.
