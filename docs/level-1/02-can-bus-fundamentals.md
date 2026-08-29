# 02 · CAN Bus Fundamentals

**CAN** (Controller Area Network), developed by Bosch in the 1980s, is the
backbone of in-vehicle communication and the network you will spend most
of this course testing. Before touching CANoe or writing a line of CAPL,
you need the physical and electrical picture straight — arbitration,
priority, and fault behavior all fall directly out of it, and a tester
who doesn't understand *why* CAN behaves as it does will misdiagnose real
bus problems as application bugs.

## Why a shared bus, not point-to-point wiring

A modern vehicle has 70-150+ ECUs. Wiring every pair that needs to talk
directly would need a harness thicker than your arm and impossible to
service. CAN's answer: **one shared pair of wires, CAN_H and CAN_L,
that every node listens to and can transmit on.**

- **Differential signaling**: CAN_H and CAN_L move in opposite directions
  from a common voltage. Electrical noise hits both wires equally and
  cancels out when a receiver reads the *difference* between them — this
  is why CAN tolerates the electrically noisy environment inside a
  vehicle (ignition systems, motors, relays) far better than a
  single-ended signal would.
- **Termination**: both physical ends of the bus carry a 120 Ω resistor
  (nominally ~60 Ω total in parallel). Missing or duplicate termination
  is one of the most common real-world CAN faults — a tester who sees
  garbled or reflected signals on a scope should check termination
  before suspecting software.
- **No addresses, only identifiers**: a CAN frame doesn't say "to ECU
  #7" — it carries an **identifier (ID)** that describes *what the data
  is* (e.g., "engine RPM"). Every node on the bus receives every frame
  electrically and decides for itself, via a filter, whether to accept
  it. This is fundamental to how CANoe test tooling works: you don't
  "connect to" one ECU, you observe and inject onto the shared bus.

## Dominant and recessive: the idea everything else follows from

CAN defines two bus states:

- **Dominant** (logical 0) — actively driven by a transmitting node.
- **Recessive** (logical 1) — the bus's passive, undriven state.

The rule that makes CAN work: **if any node drives dominant while
another drives (or leaves) recessive, the bus reads dominant.** Dominant
always wins. Two consequences follow directly from this one electrical
fact, and they explain almost everything else in this module:

1. **Arbitration** (Module 3) — when two nodes transmit simultaneously,
   the one sending the lower binary ID wins without any bus time wasted,
   because a node sending recessive that reads back dominant knows
   instantly it lost and stops.
2. **Acknowledgment** — every node that receives a frame with a correct
   CRC drives one bit dominant in the ACK slot, regardless of whether it
   *wanted* the message. The transmitter just needs to see *a* dominant
   ACK to know at least one node received the frame intact.

## Bus speed and typical rates

Classic CAN supports **up to 1 Mbit/s**. In production vehicles the most
common rates are:

| Rate | Typical use |
|---|---|
| 125 kbit/s | Body/comfort networks (low-priority, low-bandwidth: seats, mirrors) |
| 500 kbit/s | Powertrain and chassis — the most common vehicle CAN rate |
| 1 Mbit/s | High-speed diagnostic or dedicated performance networks |
| CAN FD (500 kbit/s arbitration / 2-8 Mbit/s data phase) | Modern high-bandwidth networks — Level 2 covers this |

A tester configuring a CANoe channel (Module 4) always has to match the
DUT's actual bus speed — a mismatched bit rate produces a bus that looks
completely dead or fills with error frames, which is a common early
mistake to rule out first.

## Bus topology and physical layer basics

CAN typically runs as a **linear bus** with all nodes tapped along its
length, terminated at the two physical ends — not a star and not a ring.
Each node connects through a **CAN transceiver** IC, which converts the
microcontroller's single-ended TX/RX logic signals into the differential
CAN_H/CAN_L pair. From a testing point of view, three physical-layer
facts matter:

- **Stub length matters.** A node tapped via a long stub off the main
  bus can reflect signal energy back onto the bus at higher bit rates —
  this is a real cause of intermittent frame errors that shows up only
  under specific harness routing, and testers chasing "random" CRC
  errors should ask about wiring before assuming a logic bug.
- **A shorted or open CAN_H/CAN_L is a common, testable fault.** HIL
  rigs (Module 7) deliberately include the ability to short or open bus
  wires to test an ECU's fault response.
- **Ground offset and common-mode voltage range** are why differential
  signaling has real limits — CAN transceivers specify a common-mode
  voltage range (typically around -2 V to +7 V) beyond which the
  differential trick stops working.

## Fault confinement: why CAN protects itself

CAN has a built-in mechanism so that a single malfunctioning node cannot
permanently jam the whole bus. Every node tracks a **transmit error
counter** and a **receive error counter**, incremented on detected
errors (bad CRC, bit errors, stuffing errors, form errors) and
decremented on successful transmissions/receptions. Three states follow:

| State | Condition | Behavior |
|---|---|---|
| **Error active** | Both counters < 128 | Normal — node can signal errors actively (dominant error flags) |
| **Error passive** | Either counter ≥ 128 | Node can still signal errors, but only passively (recessive) — it can no longer disrupt other traffic |
| **Bus off** | Transmit counter > 255 | Node disconnects itself from the bus entirely |

This matters enormously for testing: a node that goes **bus-off** during
a test is telling you something real happened (a wiring fault, a
babbling node, an injected error test), and a test suite should always
check bus-off state as part of its pass/fail criteria for any test
involving deliberate fault injection.

## Cheat sheet

| Item | Notes |
|---|---|
| Physical layer | Twisted pair CAN_H/CAN_L, differential, 120 Ω termination at both ends |
| Dominant / recessive | Dominant (0) always wins on the wire — the basis for arbitration and ACK |
| No addresses | Frames carry a message ID describing content, not a destination |
| Every node sees every frame | Filtering happens at the receiver, not the network |
| Classic CAN speed | Up to 1 Mbit/s; 500 kbit/s is the most common vehicle rate |
| Common vehicle rates | 125 kbit/s (body), 500 kbit/s (powertrain/chassis), 1 Mbit/s (diagnostic) |
| ACK | Any node with a valid CRC drives dominant — not proof the *intended* consumer got it |
| Error counters | TX errors cost more than RX errors; both decrement on success |
| Error active → passive → bus-off | Escalating self-isolation of a faulty node |
| Common testing fault sources | Missing/duplicate termination, mismatched bit rate, long stubs, shorted/open bus wires |

## Exercise

You're given a bus that intermittently shows CRC errors on about 2% of
frames, only when the vehicle's HVAC blower motor is running at high
speed, and only on one particular ECU's stub of the harness.

1. List three physically plausible causes, ranked by how likely each is
   given the "only during blower operation" and "only on one stub"
   clues specifically — and explain your reasoning for the ranking.
2. For each cause, describe one **measurement or test** you would perform
   to confirm or rule it out (you do not need lab equipment names beyond
   general concepts: oscilloscope, multimeter, load test).
3. Explain, in your own words, why a node experiencing intermittent CRC
   errors under this scenario would eventually show a rising **receive
   error counter** rather than a transmit error counter — tie your answer
   back to which node actually detects the corrupted frame.
