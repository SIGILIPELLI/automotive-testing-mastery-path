# 06 · UDS Diagnostics Basics

**UDS** (Unified Diagnostic Services, standardized as **ISO 14229**) is
the request/response protocol every modern automotive ECU speaks for
diagnostics: reading fault codes, reading live sensor values,
reprogramming flash, running self-tests, and much more. Every diagnostic
tool a dealership plugs in, every OTA update process, and every HIL
diagnostic test (Level 2 covers this in depth) is built on UDS. This
module covers the message format and the handful of services you'll see
constantly.

## The request/response shape

UDS is strictly **client-server**: a tester (client — a diagnostic tool,
or a CAPL script acting as one) sends a **request**, and the ECU
(server) sends back either a **positive response** or a **negative
response**. Every request and positive response begins with a **Service
Identifier (SID)**; every negative response has a fixed shape.

```text
Request:            SID  [sub-function]  [parameters...]
Positive response:  SID+0x40  [sub-function]  [data...]
Negative response:  0x7F  SID  NRC
```

The **+0x40** on a positive response is a simple, deliberate encoding
trick: it makes a positive response to service `0x22` come back as
`0x62`, unmistakably different from any other service's response, so a
tester can tell at a glance which request a response belongs to even
without matching request/response pairs explicitly.

## Common services (real SIDs)

| SID | Name | What it does |
|---|---|---|
| `0x10` | **Diagnostic Session Control** | Switch diagnostic session (default, programming, extended) |
| `0x11` | **ECU Reset** | Request the ECU reset (hard reset, key-off-on reset, soft reset) |
| `0x14` | **Clear Diagnostic Information** | Clear stored DTCs |
| `0x19` | **Read DTC Information** | Read stored/pending/confirmed diagnostic trouble codes |
| `0x22` | **Read Data By Identifier** | Read a live value by a 2-byte identifier (a "DID") |
| `0x27` | **Security Access** | Seed/key challenge to unlock protected services |
| `0x2E` | **Write Data By Identifier** | Write a value by DID (e.g., VIN, calibration data) |
| `0x31` | **Routine Control** | Start/stop/get results of a defined diagnostic routine (e.g., a self-test) |
| `0x3E` | **Tester Present** | Keep a non-default session alive by periodic "I'm still here" pings |

### Worked example: Read Data By Identifier (0x22)

Suppose DID `0xF190` is defined as the vehicle's VIN, and DID `0x1234`
is a manufacturer-specific "coolant temperature, live value" DID.

```text
Request  (tester -> ECU):  22 12 34
Response (ECU -> tester):  62 12 34 55        ; positive: SID+0x40, DID echoed, 1 data byte
```

The single data byte `0x55` (85 decimal) here would be interpreted per
that DID's defined scaling — for example, a common automotive scaling
of `physical = raw - 40` would mean 85 raw = 45 °C. **The scaling itself
is not part of UDS** — it's defined per-project in the ECU's diagnostic
specification, the same way a DBC file defines CAN signal scaling
(Level 2, Module 6).

### Worked example: negative response

If the tester asks for a DID the ECU doesn't support:

```text
Request  (tester -> ECU):  22 99 99
Response (ECU -> tester):  7F 22 31
```

Reading that: `0x7F` marks it as a negative response, `0x22` echoes the
service that was rejected, and `0x31` is the **Negative Response Code
(NRC)** — `requestOutOfRange`. A handful of NRCs come up constantly in
real diagnostic testing:

| NRC | Name | Common meaning |
|---|---|---|
| `0x11` | `serviceNotSupported` | The ECU doesn't implement this SID at all |
| `0x12` | `subFunctionNotSupported` | The SID exists, but not this sub-function |
| `0x13` | `incorrectMessageLengthOrInvalidFormat` | Malformed request |
| `0x22` | `conditionsNotCorrect` | The service exists but preconditions aren't met (e.g., vehicle must be stationary) |
| `0x31` | `requestOutOfRange` | Bad parameter value, e.g. unknown DID |
| `0x33` | `securityAccessDenied` | The session isn't unlocked via `0x27` for this operation |
| `0x35` | `invalidKey` | Security access key didn't match the expected value |
| `0x78` | `requestCorrectlyReceived-ResponsePending` | "Got it, still working" — expect the real response shortly after |

That last one, `0x78`, is important for test automation: a test script
waiting for a diagnostic response must be written to **keep waiting**
after receiving `0x78`, not treat it as a final answer or a failure.

## Diagnostic sessions

UDS services are gated by **session state**. On power-up, an ECU is
normally in the **default session**. Services like reprogramming or
writing calibration data require switching sessions first via `0x10`:

```text
Request:   10 03                    ; switch to Extended Diagnostic Session
Response:  50 03 00 32 01 F4        ; positive (SID+0x40), session echoed, timing parameters
```

Non-default sessions typically time out and fall back to default if the
tester goes quiet — which is exactly why `0x3E` (Tester Present) exists:
a test script or diagnostic tool that needs to hold a session open sends
a periodic `0x3E` request (often with a "suppress positive response"
sub-function, so it doesn't clutter the bus with responses) purely to
reset the ECU's session timeout.

## Security access: the seed/key pattern

Some services (writing calibration data, unlocking certain routines)
additionally require **Security Access (0x27)** on top of the right
session:

```text
Request:   27 01                             ; request seed, level 1
Response:  67 01 3A 7C F0 91                 ; ECU-generated seed (4 bytes here)

           [tester computes key from seed using an algorithm shared
            with the ECU, but not sent over the bus]

Request:   27 02 5B A1 3D 40                 ; send computed key
Response:  67 02                             ; positive: access granted
```

The seed/key **algorithm itself is confidential and project-specific**
— UDS defines the message exchange, not the cryptography. A test
engineer working against a real project needs the actual algorithm (or
a test-mode bypass key, common on development ECUs) supplied by the
project, never invented.

## Reading DTCs (0x19)

Diagnostic Trouble Codes are how an ECU reports "something is wrong" in
a standardized, stored way. A common sub-function reads all confirmed
DTCs:

```text
Request:   19 02 08          ; sub-function 0x02 = reportDTCByStatusMask, mask 0x08 = confirmedDTC
Response:  59 02 FF  01 23 45 08  01 24 10 08
```

Reading the response: `0x59` (positive), sub-function echoed, a
status-availability mask, then repeating groups of **3-byte DTC + 1-byte
status**. `01 23 45` and `01 24 10` here are example 3-byte DTC values
(the exact numeric meaning is defined by the vehicle manufacturer's DTC
list), each followed by a status byte (`0x08` here indicating the
confirmed bit is set). Level 2, Module 5 covers DTC status bytes and
lifecycle in depth.

## Cheat sheet

| Item | Notes |
|---|---|
| Standard | ISO 14229 (UDS) |
| Model | Strict client (tester) / server (ECU) request-response |
| Positive response | SID + 0x40 |
| Negative response | `7F`, echoed SID, NRC |
| `0x10` | Diagnostic Session Control |
| `0x11` | ECU Reset |
| `0x14` | Clear Diagnostic Information |
| `0x19` | Read DTC Information |
| `0x22` | Read Data By Identifier |
| `0x27` | Security Access (seed/key) |
| `0x2E` | Write Data By Identifier |
| `0x31` | Routine Control |
| `0x3E` | Tester Present (keeps a session alive) |
| NRC `0x78` | "Still working" — a test must keep waiting, not fail |
| NRC `0x33`/`0x35` | Security access denied / invalid key |
| Scaling of DID values | Project-specific, not defined by UDS itself |

## Exercise

You are writing a test plan for an ECU's `0x22` (Read Data By
Identifier) support on DID `0x4A10` ("battery state of charge, 0-100%,
1 byte, no scaling").

1. Write out, in the request/response byte notation used above, what a
   correct positive response for a state-of-charge reading of 73%
   should look like.
2. Write out three distinct negative-response scenarios you should test
   for (choose from the NRC table), each with the request bytes and the
   expected negative response bytes, and a one-sentence justification
   for why that scenario is worth testing.
3. Explain, referencing the session and security-access material above,
   under what circumstances reading this DID might legitimately require
   a non-default session or a security unlock first, even though "just
   reading a value" sounds harmless — give one concrete automotive
   reason a manufacturer might restrict read access to a value like
   this.
