# 03 · Ethernet/Automotive Ethernet Basics

Modern ECUs — ADAS domain controllers, infotainment head units,
gateways — increasingly carry Ethernet alongside or instead of CAN.
This module covers what's actually different about automotive
Ethernet versus office Ethernet, the SOME/IP service layer test
engineers meet most often, and how frame-level testing habits from CAN
carry over.

!!! note "About this module"
    No physical Ethernet-equipped ECU or SOME/IP stack was used to
    produce this content. The frame formats, timing figures, and
    SOME/IP concepts below reflect documented IEEE 802.3/AUTOSAR/
    SOME/IP standards — verify exact configuration and vendor-specific
    tooling against your project's actual network design.

## Why Ethernet, and what's different

CAN tops out around 1 Mbit/s (5 Mbit/s for CAN-FD's data phase);
Ethernet in a vehicle runs 100 Mbit/s to multi-gigabit — needed for
camera/radar/lidar data volumes ADAS and surround-view systems produce
that CAN/CAN-FD simply cannot carry. But vehicle wiring, EMI, and cost
constraints mean automotive Ethernet is not the same physical layer as
an office switch:

| | Office Ethernet | Automotive Ethernet |
|---|---|---|
| Physical layer | 100BASE-TX (2-pair, RJ45) | 100BASE-T1 / 1000BASE-T1 (single unshielded twisted pair) |
| Topology | Star, switches everywhere | Star or daisy-chained via switches; often one link per ECU pair |
| Wiring cost/weight driver | Not a concern | Single-pair wiring specifically to cut harness weight/cost |
| Typical protocols riding on top | TCP/IP, arbitrary | SOME/IP, DoIP (diagnostics), AVB/TSN for time-sync media |

The single-pair physical layer (`BroadR-Reach`, standardized as
100BASE-T1/1000BASE-T1) is the headline automotive-specific piece —
same Ethernet frame format above it, different electrical signaling
below it.

## SOME/IP: the service layer test engineers actually touch

Where CAN signals are static (a DBC defines every frame up front),
Ethernet-based ECU communication is frequently **service-oriented**:
an ECU offers a named service, other ECUs discover and subscribe to
it, and requests/responses or eventing happen at that service level.
SOME/IP (Scalable service-Oriented MiddlewarE over IP) is the
AUTOSAR-standard protocol for this.

Core SOME/IP concepts:

| Concept | Meaning |
|---|---|
| Service | A named capability an ECU offers (e.g., "climate control state") |
| Method | An RPC-style call — request, get a response |
| Event | A one-way notification, typically as part of an eventgroup |
| Eventgroup | A named bundle of events a client subscribes to as a unit |
| SOME/IP-SD (Service Discovery) | The sub-protocol for offering/finding/subscribing to services dynamically over UDP multicast |

This is a meaningfully different test mental model from CAN's "every
signal is always on the bus at its configured rate": a SOME/IP event
only arrives if something has subscribed, and a method call needs the
service to have been *discovered* first. A test that sends a subscribe
request but never sees any events may be failing at the discovery
step, not the eventing step — the two need to be verified separately.

## A CAPL-side SOME/IP-style test sketch

CANoe supports Ethernet and SOME/IP test modules; the syntax differs
from pure CAPL CAN but the testcase structure is familiar:

```c
// Illustrative — exact SOME/IP CAPL API surface depends on your
// CANoe Ethernet/SOME/IP option version and .fibex/service description.
testcase tc_ClimateStateEventDelivered()
{
  testCaseTitle("Subscribing to ClimateState eventgroup yields an event within spec");
  dword t0, t1;

  SomeIpSdSubscribeEventgroup("ClimateControlService", "ClimateStateGroup");
  t0 = timeNowNS() / 1000000;

  testWaitForSomeIpEvent("ClimateControlService", "ClimateStateChanged", 2000);
  t1 = timeNowNS() / 1000000;

  testStepCheck("event received within 2000ms of subscription", (t1 - t0) <= 2000);
}

testcase tc_ServiceDiscoveredBeforeSubscribe()
{
  testCaseTitle("Service must be offered/discovered before a subscribe can succeed");
  testStepCheck("service currently offered", SomeIpSdIsServiceAvailable("ClimateControlService") == 1);
}
```

Note the split into two testcases: discovery availability is checked
independently of eventgroup delivery, the same "test one thing per
testcase" discipline from Level 1 applied to a protocol where the
setup step (discovery) has its own well-defined failure mode.

## Frame- and timing-level differences from CAN testing

| Concern | CAN habit | Ethernet/SOME-IP adjustment |
|---|---|---|
| "Is it on the bus at all" | Check frame ID present at expected cycle time | Check service *offered* via SD before checking method/event traffic |
| Payload inspection | Fixed DBC-defined bit layout | Often serialized (SOME/IP payload serialization rules) — needs the service description, not just a byte offset guess |
| Timing budget | Frame cycle time from DBC | Service discovery has its own timers (initial delay, repetition, cyclic offer) separate from application-level event rate |
| Loss/retransmission | CAN handles at hardware/arbitration level | UDP-based SOME/IP has no built-in reliability — a dropped event is simply gone, so test for it explicitly if reliability matters |

## Cheat sheet

| Concept | Key point |
|---|---|
| 100BASE-T1 | Single-pair physical layer for automotive Ethernet; same frame format above it |
| SOME/IP | Service-oriented middleware: services, methods, events, eventgroups |
| SOME/IP-SD | Discovery/offer/subscribe sub-protocol — verify separately from eventing |
| No built-in reliability | SOME/IP over UDP can silently drop events — test for loss explicitly if it matters |

## Exercise

1. A test subscribes to an eventgroup and waits 2000ms for an event
   with no result. List the three distinct failure points (discovery,
   subscription, event delivery) and, for each, one way to
   distinguish it from the others in a test log.
2. Explain why a fixed byte-offset assumption from CAN DBC habits is
   unsafe when reading a SOME/IP payload, and what artifact (analogous
   to a DBC) you'd need instead.
3. SOME/IP events ride over UDP with no automatic retransmission.
   Design a test that specifically checks for event loss under a
   simulated burst of 100 rapid state changes, and state what
   pass/fail threshold you'd choose and why.
