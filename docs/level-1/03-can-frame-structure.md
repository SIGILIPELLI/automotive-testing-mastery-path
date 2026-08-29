# 03 · CAN Frame Structure

Module 2 gave you the physical and electrical picture. This module opens
up the **frame** itself — the bit-level structure every CAN message
follows — and the two mechanisms, arbitration and acknowledgment, that
fall directly out of the dominant/recessive rule. Every tool you'll use
for the rest of this course (CANoe's trace window, a CAPL message
handler, a DBC signal definition) is really just a structured view onto
these bits.

## The classic CAN data frame

```text
| SOF | Arbitration field (ID + RTR/SRR + IDE) | Control (IDE, r0, DLC) | Data (0-8 bytes) | CRC (15 bit + delim) | ACK (slot + delim) | EOF | IFS |
```

Field by field:

| Field | Size | Meaning |
|---|---|---|
| **SOF** | 1 bit | Start of Frame — dominant bit that tells all nodes a frame is beginning |
| **Identifier** | 11 bits (standard) or 29 bits (extended) | What the message *is* — also doubles as its arbitration priority |
| **RTR** | 1 bit | Remote Transmission Request — recessive on a data frame, dominant would request one (remote frames are rarely used in modern designs) |
| **IDE** | 1 bit | Identifier Extension — dominant = 11-bit standard frame, recessive = 29-bit extended frame |
| **DLC** | 4 bits | Data Length Code — how many data bytes follow (0-8 for classic CAN) |
| **Data** | 0-8 bytes | The payload |
| **CRC** | 15 bits + 1 recessive delimiter | Checksum computed by hardware over SOF through the data field |
| **ACK** | 1 slot bit + 1 recessive delimiter | The transmitter sends this bit recessive; any receiver with a valid CRC overwrites it dominant |
| **EOF** | 7 recessive bits | End of Frame |
| **IFS** | 3 bits | Interframe space — minimum gap before the next frame |

### Standard vs. extended identifiers

Both formats coexist on the same physical bus:

- **Standard (11-bit) IDs**: range 0x000-0x7FF (2048 possible values).
  Most classic body/chassis/powertrain traffic uses standard IDs.
- **Extended (29-bit) IDs**: range 0x00000000-0x1FFFFFFF. Used where more
  ID space is needed — J1939 (heavy truck/commercial networks) is built
  entirely on extended IDs, for example.

A receiver's filter must be configured to accept the ID *format* it
expects, not just the numeric value — this is a common early mistake
when setting up a CANoe RX filter or a CAPL `on message` handler: an ID
that "looks right" but is the wrong format (standard vs. extended) will
never match.

## Arbitration: how simultaneous transmission resolves without collision

Any node may start transmitting the instant it observes the bus is idle.
If two or more nodes start at exactly the same time, they resolve who
gets to continue **during the arbitration field itself**, bit by bit,
using the dominant-wins rule from Module 2:

```text
Node A transmits ID 0x24C:  0 1 0 0 1 0 0 1 1 0 0
Node B transmits ID 0x24F:  0 1 0 0 1 0 0 1 1 1 …
                                              ^
     Bit 10: B sends recessive(1), reads back dominant(0) on the bus.
     B recognizes it lost, stops transmitting immediately, and will
     retry automatically once the bus is idle again.
     Node A never even notices — its frame proceeds undamaged.
```

Every transmitting node reads the bus back while it transmits its own
ID. The moment a node sends recessive but reads dominant, it knows a
higher-priority (numerically lower ID) frame is in progress and backs
off — with **zero bus time wasted**, because the losing node's bits were
identical to the winner's up to that point anyway. This is called
**CSMA/CR** (Carrier Sense Multiple Access with Collision *Resolution*)
— explicitly not collision *destruction* the way early Ethernet worked.

The direct consequence: **the numerically lowest ID always wins
arbitration, so ID assignment is a priority decision, not just a naming
scheme.** A brake-system status message should get a low ID; a
seat-heater status message should not. Networks are typically kept under
roughly 50-70% bus load partly because, under heavy load, low-priority
(high-numbered) IDs can be starved indefinitely by higher-priority
traffic — something a load test on a CANoe simulation can reproduce and
measure directly.

## Acknowledgment: what ACK actually proves

The ACK slot is one bit that the transmitter sends recessive. **Any
node — not just the intended consumer — that received the frame with a
valid CRC drives that bit dominant.** If the transmitter reads dominant
back, it knows the frame was heard correctly by at least one node on the
bus.

This is a subtlety testers frequently get wrong: **a dominant ACK does
not mean the specific ECU you care about received the message** — it
only means *some* node did. If your test needs to confirm that a
specific target ECU processed a message, you need an application-level
acknowledgment (a response message, a UDS positive response, a changed
output signal) — not just bus-level ACK. Conversely, a transmitter
seeing **no ACK** (the ACK slot stays recessive) means literally no node
on the bus received a valid frame — a strong, immediate signal that the
bus itself may be disconnected, or the transmitting node's own
transceiver is faulty (a node hears its own transmission's ACK slot the
same way as any receiver, unless self-reception is disabled).

## Error signaling within the frame

Every node checks every frame it hears for correctness: **bit
monitoring** (does what I see on the bus match what I expect during
non-arbitration fields?), **CRC** (does the checksum match?), **form**
(are fixed-format fields — CRC delimiter, ACK delimiter, EOF — actually
recessive as required?), and **bit stuffing** (after 5 consecutive
identical bits, a transmitter inserts one opposite bit so receivers stay
synchronized — a receiver expecting this and not seeing it flags a
stuffing error). Any node that detects an error transmits an **error
flag** — six dominant bits — which destroys the frame for every node on
the bus simultaneously, and the transmitter automatically retries. This
is what feeds the error counters and eventual bus-off state from
Module 2.

## Cheat sheet

| Item | Notes |
|---|---|
| Frame skeleton | SOF · ID(+RTR/IDE) · DLC · Data(0-8B) · CRC · ACK · EOF |
| Standard ID | 11 bits, 0x000-0x7FF |
| Extended ID | 29 bits, 0x00000000-0x1FFFFFFF (used by e.g. J1939) |
| DLC | 4-bit field, 0-8 data bytes on classic CAN |
| Arbitration | Bitwise, non-destructive; **lowest numeric ID always wins** |
| ID = priority | Safety/critical messages get low IDs by design, not accident |
| ACK slot | Transmitter sends recessive; any valid-CRC receiver drives it dominant |
| ACK proves | *Someone* got the frame — never proof the intended consumer did |
| No ACK at all | Strong signal of a disconnected bus or transceiver fault |
| Error flag | 6 dominant bits, destroys the frame for everyone, triggers automatic retry |
| Bit stuffing | One opposite bit inserted after 5 identical bits, to preserve sync |

## Exercise

Two nodes attempt to transmit at the same instant with standard
(11-bit) IDs `0x1A3` and `0x1A1`.

1. Write out both IDs in binary (11 bits each) and perform the
   bit-by-bit arbitration trace by hand — mark exactly which bit
   position the loser first sends recessive while the bus reads
   dominant, and state which node wins.
2. Suppose instead a third node with ID `0x0FF` also starts
   transmitting at the same instant. Redo the trace with all three IDs
   and identify the final winner — explain why the presence of the
   third node doesn't change *when* the original two nodes' arbitration
   would have resolved between themselves.
3. Design one CRC/data mismatch scenario (in words, not bits) that would
   cause a receiving node to flag a CRC error, and explain what every
   other listening node on the bus would observe happen to that frame
   as a direct result.
