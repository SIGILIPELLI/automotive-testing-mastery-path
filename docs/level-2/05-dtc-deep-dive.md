# 05 · Diagnostic Trouble Codes (DTCs) Deep Dive

Level 1 introduced UDS services and touched DTCs as "the thing that
sets when something fails." Real ECU test suites spend a disproportionate
amount of effort on DTC behavior specifically, because a DTC's job is to
be trustworthy evidence of a fault long after the fault itself is gone —
which means its state machine, not just its final value, is what needs
testing.

!!! note "About this module"
    DTC status byte semantics and UDS service `0x19`/`0x14` behavior
    below are genuine, documented ISO 14229-1 (UDS) and ISO 15031/SAE
    J2012 DTC conventions. Nothing here was captured from a live ECU —
    treat every snippet as a reviewed reference and verify against your
    target ECU's diagnostic spec, since manufacturers customize DTC
    behavior within the standard's allowed variation.

## The DTC status byte

Every DTC carries an 8-bit **status byte**, not just a pass/fail flag.
Each bit is independently meaningful:

| Bit | Name | Meaning |
|---|---|---|
| 0 | testFailed | Fault is failing right now, this operation cycle |
| 1 | testFailedThisOperationCycle | Fault has failed at least once since power-up |
| 2 | pendingDTC | Fault failed but hasn't yet met the confirmation criteria (Module varies by OEM — often 2 consecutive drive cycles) |
| 3 | confirmedDTC | Fault has met confirmation criteria — this is the "real" stored DTC |
| 4 | testNotCompletedSinceLastClear | The diagnostic test for this DTC hasn't run since the last clear |
| 5 | testFailedSinceLastClear | Fault has failed at least once since the last clear |
| 6 | testNotCompletedThisOperationCycle | This cycle's diagnostic test hasn't completed yet |
| 7 | warningIndicatorRequested | Should the malfunction indicator lamp be on because of this DTC |

The single most common testing mistake is treating a DTC as binary
(set/not set). A fault can be **pending** (bit 2) without being
**confirmed** (bit 3) — e.g., it failed once but the OEM's maturation
logic requires it to fail on two separate drive cycles before it's
"real" enough to light the MIL and be reported to emissions/warranty
systems. A test that clears a fault after one failure and calls it done
never actually exercises confirmation logic at all.

## The DTC lifecycle

```
no fault ──fails──> pending (bit2=1, bit3=0)
                │
                ├─ passes before maturation ──> pending clears, back to no fault
                │
                └─ fails again next cycle (or per OEM maturation rule)
                        │
                        v
                confirmed (bit3=1), MIL possibly on (bit7)
                        │
                        ├─ passes for N consecutive cycles (OEM-defined) ──> MIL off, DTC stays stored (bit0=0, bit3=1)
                        │
                        └─ UDS 0x14 ClearDiagnosticInformation ──> DTC and status fully cleared
```

A rigorous test plan checks every transition arrow, not just the two
endpoints — especially the "passes for N cycles turns MIL off but
doesn't erase the stored DTC" transition, which is frequently missed
because it looks similar to a full clear from the driver's-seat view
(lamp off) but is diagnostically very different (history is retained).

## UDS services for DTC testing

| Service | ID | Purpose |
|---|---|---|
| ReadDTCInformation | `0x19` | Read DTCs by status mask, by severity, snapshot/extended data |
| ClearDiagnosticInformation | `0x14` | Clear DTC(s), optionally by group |

### Reading DTCs by status mask (subfunction 0x02)

```c
// CAPL: request all confirmed DTCs (status mask bit3 = 0x08)
variables
{
  byte req[] = {0x19, 0x02, 0x08};  // ReadDTCInformation, reportDTCByStatusMask, mask=confirmed
}

on key 'r'
{
  DiagRequestSend(diagRequest(req));
}

on diagResponse *
{
  write("DTC response: %s", this.GetString());
}
```

A common bug this catches: an ECU that reports a DTC under
`testFailed` (bit0) mask requests but not under `confirmedDTC` (bit3)
mask requests, when it should be confirmed — meaning the maturation
logic isn't actually running, only the raw fault detection is.

### Snapshot and extended data

Reading a DTC's raw status is only half the picture — most OEM specs
also require **freeze frame (snapshot) data**: a capture of related
signals (vehicle speed, engine RPM, other sensor values) at the moment
the fault was first detected, retrievable via `0x19` subfunction
`0x04` (reportDTCSnapshotRecordByDTCNumber). Testing this means
verifying not just that a snapshot exists, but that its captured values
are actually representative of the fault moment — a snapshot captured
from stale/cached signal values rather than the live bus at fault time
is a real defect class.

### Clearing DTCs correctly

```c
variables
{
  byte clearReq[] = {0x14, 0xFF, 0xFF, 0xFF};  // ClearDiagnosticInformation, groupOfDTC = all
}

on key 'c'
{
  DiagRequestSend(diagRequest(clearReq));
}
```

Test cases for `0x14` should include:

- Clearing while the fault condition is still physically active — per
  most OEM specs, the DTC should re-mature (go back to pending) almost
  immediately rather than staying cleared, since the underlying problem
  never went away.
- Clearing with a security-access precondition unmet, if the ECU's
  spec requires it (Level 1's session/security module) — verify the
  correct UDS negative response (`0x7F 0x14 0x33` — securityAccessDenied)
  rather than a silent no-op.
- Clearing a specific DTC group vs. all DTCs, and confirming unrelated
  DTCs in other groups are untouched.

## A worked DTC test case

| Field | Value |
|---|---|
| Test Case ID | TC-DTC-021 |
| Title | Coolant overheat DTC matures to confirmed only after two-cycle rule |
| Preconditions | No active DTCs, ECU at default session |
| Steps | 1. Trigger coolant overheat condition (Module 1's boundary test).<br>2. Power-cycle the ECU (end operation cycle).<br>3. Read DTC status via `0x19 0x02` with mask 0xFF.<br>4. Trigger the same fault again in the new cycle.<br>5. Read DTC status again. |
| Expected result | After step 3: bit2 (pending) = 1, bit3 (confirmed) = 0. After step 5: bit3 (confirmed) = 1, bit7 (MIL request) per OEM spec. |
| Notes | This is the test that actually distinguishes "detects the fault" from "correctly implements maturation" — the two are frequently conflated in under-specified test suites. |

## Cheat sheet

| Concept | Key point |
|---|---|
| Status byte | 8 independent bits, not a pass/fail flag |
| Pending vs. confirmed | Pending = failed once; confirmed = met maturation criteria |
| MIL off ≠ DTC cleared | Lamp can go off after N good cycles while the DTC stays stored |
| `0x19` | ReadDTCInformation — always specify the right status mask/subfunction |
| `0x14` | ClearDiagnosticInformation — test clearing under an active fault, not just a resolved one |
| Snapshot data | Freeze-frame values must reflect the actual fault moment, not stale cache |

## How It Actually Works: the debounce counter underneath "pending" vs. "confirmed"

The status byte's bits look like a clean state machine, but inside the
ECU they're driven by a much more mechanical piece of code: a signed
**debounce counter**, usually incremented on each diagnostic-test fail
and decremented on each pass, checked against two thresholds. A typical
implementation looks roughly like:

```text
on each diagnostic test result:
    if failed:  counter = min(counter + failStep,  matureThreshold)
    if passed:  counter = max(counter - healStep,  healThreshold)

    pendingDTC   = (counter > 0)
    confirmedDTC = (counter >= matureThreshold)
```

The "2 of 3 cycles" rule in the exercise below is a direct consequence
of choosing `failStep`, `healStep`, and `matureThreshold` values — for
instance `matureThreshold = 2`, `failStep = 1`, `healStep = 1` produces
exactly "confirmed once two net failures have accumulated, tolerating
one intervening pass," which behaves differently from a naive
"confirmed after 2 *consecutive* failures" implementation the moment a
pass appears between two failures: the counter-based version keeps its
accumulated progress (net count stays at 1 after fail-pass, then reaches
2 on the next fail), while a consecutive-counter implementation resets
to zero on any pass and needs two failures *in a row* afterward. This is
precisely the difference this module's exercise asks you to design a
test sequence to expose — and it's why the right test sequence is
fail → pass → fail (three cycles, two failures, one pass in between),
not fail → fail (which both implementations would confirm identically
and therefore proves nothing about which rule is actually implemented).

The **operation cycle** boundary that resets `testFailedThisOperationCycle`
and `testNotCompletedThisOperationCycle` (bits 1 and 6) is itself just
an ECU power-state transition, not a fixed calendar concept — commonly
key-off-to-key-on, sometimes engine-off-to-running specifically for
powertrain DTCs per SAE J1979/J2012 conventions. A test that "power
cycles the ECU" (as this module's worked test case does at step 2) is
really forcing exactly this transition so the debounce counter's
per-cycle bits reset while its cross-cycle maturation counter — bits 2
and 3 — deliberately does not, which is the whole mechanical reason
pending/confirmed state can survive a power cycle while the "this
cycle" bits cannot.

## Exercise

An OEM spec states: a DTC matures to confirmed after failing on 2 of
the last 3 operation cycles, and the MIL turns off after 3 consecutive
passing cycles but the DTC remains stored until an explicit `0x14`
clear or 40 consecutive passing drive cycles (whichever comes first).

1. Draw the state machine (mirroring the lifecycle diagram above) for
   this specific OEM's rule, labeling each transition with its trigger
   condition.
2. Design a test sequence (as a numbered step list) that proves the "2
   of 3 cycles" maturation rule specifically — i.e., a sequence where a
   naive "matures after 2 consecutive failures" implementation would
   pass or fail differently than the correct "2 of 3" implementation.
3. Write the `0x19` status-mask byte value you'd use to query only DTCs
   that are confirmed AND currently failing (both bit0 and bit3 set),
   and explain why querying with mask `0xFF` and post-filtering in your
   test script is a reasonable alternative when you need to inspect
   multiple bits at once.
