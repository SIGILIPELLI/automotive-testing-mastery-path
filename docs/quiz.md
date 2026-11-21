---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Automotive Testing Mastery Path with a 19-question final quiz."
---

# Final Quiz

19 questions drawn from all four levels of this course. You need **70% (14 out of 19)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "Two ECUs transmit at the same instant. ECU A sends ID 0x100 and ECU B sends 0x200. Who wins arbitration?",
    "options": [
      "ECU A: the lower ID has dominant bits first and dominant always wins; B backs off and retries",
      "Whichever sent more data",
      "ECU B",
      "Both are lost"
    ],
    "correctIndex": 0,
    "explanation": "Arbitration falls out of the dominant/recessive rule."
  },
  {
    "question": "The scope shows reflected, garbled CAN signals. What do you check first?",
    "options": [
      "The ECU firmware",
      "Termination: 120 Ω at both ends (about 60 Ω across), since missing or duplicate termination is a common fault",
      "The CAPL script",
      "The DBC"
    ],
    "correctIndex": 1,
    "explanation": "Don't blame software before checking the physical layer."
  },
  {
    "question": "A CAN-FD frame has DLC 13. How many data bytes?",
    "options": [
      "24",
      "13",
      "32",
      "48"
    ],
    "correctIndex": 2,
    "explanation": "DLC 9 to 15 map to 12, 16, 20, 24, 32, 48, 64 bytes."
  },
  {
    "question": "A signal is defined with start bit 16, length 8, scale 1, offset −40 (Intel). Raw byte 2 is 130. What is the physical value?",
    "options": [
      "90 °C",
      "170",
      "130",
      "−40"
    ],
    "correctIndex": 0,
    "explanation": "physical = raw × 1 − 40. Verified by decoding a test frame."
  },
  {
    "question": "A signal with scale 0.1 and raw 0x012C (300) means:",
    "options": [
      "300",
      "30.0 physical units",
      "0.3",
      "3.0"
    ],
    "correctIndex": 1,
    "explanation": "300 × 0.1 = 30.0. Verified by computation."
  },
  {
    "question": "A UDS request is 22 F1 90. What does a positive response start with?",
    "options": [
      "62 F1 90 followed by the data",
      "F1 90 62",
      "7F 22",
      "22 F1 90"
    ],
    "correctIndex": 0,
    "explanation": "Positive response SID = request SID + 0x40; the DID is echoed. Verified with a small responder."
  },
  {
    "question": "What's the response to 22 12 34 when DID 0x1234 isn't supported?",
    "options": [
      "22 12 34",
      "62 12 34",
      "7F 22 11",
      "7F 22 31 (negative response, requestOutOfRange)"
    ],
    "correctIndex": 3,
    "explanation": "Negative responses are 7F, the SID, then the NRC."
  },
  {
    "question": "What's the response to an unknown service 0x99?",
    "options": [
      "No reply",
      "7F 99 31",
      "99 40",
      "7F 99 11 (serviceNotSupported)"
    ],
    "correctIndex": 3,
    "explanation": "Verified with the responder."
  },
  {
    "question": "Which UDS service keeps a non-default session alive?",
    "options": [
      "0x14",
      "0x3E Tester Present",
      "0x10",
      "0x11"
    ],
    "correctIndex": 1,
    "explanation": "0x10 changes session; 0x3E pings."
  },
  {
    "question": "A CoolantTemp warning must trigger when temperature is strictly above 120 °C. Which boundary tests are best?",
    "options": [
      "100 and 150",
      "Only 120",
      "119.9 (no warning), 120.0 (no warning), 120.1 (warning)",
      "0 and 255"
    ],
    "correctIndex": 2,
    "explanation": "120.0 is where > versus >= bugs hide."
  },
  {
    "question": "A low-fuel warning raises below 10% and clears above 15%. At 12% after having been below 10%, is the warning on?",
    "options": [
      "No",
      "Depends on timer",
      "Yes: hysteresis keeps it on until above 15%",
      "Only at exactly 12"
    ],
    "correctIndex": 2,
    "explanation": "warningActive holds the state."
  },
  {
    "question": "In CAPL, which handler runs each time a particular frame arrives?",
    "options": [
      "on timer",
      "on message <name or ID>",
      "on key",
      "on start"
    ],
    "correctIndex": 1,
    "explanation": "on start fires once; on timer when a timer expires."
  },
  {
    "question": "Why use a CANoe test module rather than a simulation node with write()?",
    "options": [
      "It's faster",
      "It needs no database",
      "It uses less memory",
      "It produces structured verdicts and reports that CI and auditors can consume"
    ],
    "correctIndex": 3,
    "explanation": "Text in a log isn't a verdict."
  },
  {
    "question": "A requirement has one test case which is currently failing. What does coverage say?",
    "options": [
      "Not covered",
      "Requirement covered but verdict not passing; verdict coverage separates covered from passing",
      "Fully verified",
      "Unneeded"
    ],
    "correctIndex": 1,
    "explanation": "Requirement coverage alone isn't enough."
  },
  {
    "question": "Which standard covers hazards when the system works as designed but the design is insufficient (e.g. perception limits)?",
    "options": [
      "ISO 21448 SOTIF",
      "ISO 21434",
      "ASPICE",
      "ISO 26262"
    ],
    "correctIndex": 0,
    "explanation": "ISO 26262 covers malfunctions; 21434 cybersecurity."
  },
  {
    "question": "Why test combined (cascading) faults?",
    "options": [
      "It replaces ASIL",
      "Single faults can't be injected",
      "Mechanisms proven separately for faults A and B may still fail when both occur",
      "It's cheaper"
    ],
    "correctIndex": 2,
    "explanation": "Single-fault testing can't find that."
  },
  {
    "question": "A test result from six months ago can't be reproduced. What was missing?",
    "options": [
      "A bigger rig",
      "A configuration baseline naming the exact DBC, A2L, ECU build, test source and fault matrix versions",
      "A faster bus",
      "More tests"
    ],
    "correctIndex": 1,
    "explanation": "Artifacts must be versioned together."
  },
  {
    "question": "A HIL rig is shared by 10 pipelines. What is the automotive-specific bottleneck?",
    "options": [
      "Network speed",
      "CPU cores",
      "Disk space",
      "Rig time: parallelism is capped by the number of physical rigs, so reservations are scheduled"
    ],
    "correctIndex": 3,
    "explanation": "Run SIL first to save rig time."
  },
  {
    "question": "An ASPICE assessor scores a supplier low although all CAPL tests pass. Why can that happen?",
    "options": [
      "DBC was big",
      "Tests were too many",
      "ASPICE assesses process maturity and evidence such as traceability, not just test outcomes",
      "Rigs were old"
    ],
    "correctIndex": 2,
    "explanation": "Capability levels 0–5 grade process."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### Decode a DBC signal

Write decode(raw_bytes, start_bit, length, scale, offset) that extracts a little-endian (Intel) signal and returns physical = raw × scale + offset. Decode CoolantTemp (start 16, length 8, scale 1, offset −40) from raw byte 130, and a 16-bit signal with scale 0.1 from 0x012C.

**Starter code**

```python
def decode(raw_bytes, start_bit, length, scale, offset, little_endian=True):
    # extract `length` bits at `start_bit` (Intel/little-endian) and apply physical = raw * scale + offset
    pass

frame = bytes([0x00, 0x00, 130, 0, 0, 0, 0, 0])
print(decode(frame, 16, 8, 1, -40))
print(decode(bytes([0x2C, 0x01]) + bytes(6), 0, 16, 0.1, 0))
```

**Expected output**

```text
90
30.0
```

<details><summary>Show solution and explanation</summary>

<pre><code>def decode(raw_bytes, start_bit, length, scale, offset, little_endian=True):
    value = int.from_bytes(raw_bytes, &quot;little&quot; if little_endian else &quot;big&quot;)
    raw = (value &gt;&gt; start_bit) &amp; ((1 &lt;&lt; length) - 1)
    return raw * scale + offset

frame = bytes([0x00, 0x00, 130, 0, 0, 0, 0, 0])
print(decode(frame, 16, 8, 1, -40))
print(decode(bytes([0x2C, 0x01]) + bytes(6), 0, 16, 0.1, 0))
</code></pre>

<p>A DBC maps bit position, length, endianness, scale and offset to a physical value. Verified by running the solution.</p>
</details>

### A tiny UDS responder

Write respond(req) for three services: 0x10, 0x22 (Read Data By Identifier with DID 0xF190 = a VIN) and 0x3E. Positive response = SID + 0x40 (for 0x22 echo the DID then the data). Unknown service: 7F SID 11. Unknown DID: 7F 22 31. Print each response as hex.

**Starter code**

```python
SERVICES = {0x10: "Session", 0x22: "ReadDID", 0x3E: "TesterPresent"}
DIDS = {0xF190: b"WVWZZZ1JZXW000001"}

def respond(req):
    # positive: SID+0x40 (+ DID echo + data for 0x22); negative: 7F SID NRC
    # NRC 0x11 serviceNotSupported, 0x31 requestOutOfRange
    pass

for r in ([0x22, 0xF1, 0x90], [0x22, 0x12, 0x34], [0x3E, 0x00], [0x99]):
    print(respond(bytes(r)).hex(" "))
```

**Expected output**

```text
62 f1 90 57 56 57 5a 5a 5a 31 4a 5a 58 57 30 30 30 30 30 31
7f 22 31
7e 00
7f 99 11
```

<details><summary>Show solution and explanation</summary>

<pre><code>SERVICES = {0x10: &quot;Session&quot;, 0x22: &quot;ReadDID&quot;, 0x3E: &quot;TesterPresent&quot;}
DIDS = {0xF190: b&quot;WVWZZZ1JZXW000001&quot;}

def respond(req):
    sid = req[0]
    if sid not in SERVICES:
        return bytes([0x7F, sid, 0x11])          # serviceNotSupported
    if sid == 0x22:
        did = (req[1] &lt;&lt; 8) | req[2]
        if did not in DIDS:
            return bytes([0x7F, sid, 0x31])      # requestOutOfRange
        return bytes([sid + 0x40, req[1], req[2]]) + DIDS[did]
    return bytes([sid + 0x40]) + req[1:2]

for r in ([0x22, 0xF1, 0x90], [0x22, 0x12, 0x34], [0x3E, 0x00], [0x99]):
    print(respond(bytes(r)).hex(&quot; &quot;))
</code></pre>

<p>UDS positive responses add 0x40 to the SID; negative responses are 0x7F, the SID and an NRC. Verified by running the solution.</p>
</details>
