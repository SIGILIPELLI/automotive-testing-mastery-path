# 04 · Cybersecurity Testing for Vehicles (ISO 21434)

Connected ECUs — telematics units, Ethernet-based domain controllers,
anything reachable from outside the vehicle's trusted boundary — carry
a threat model functional and safety testing don't cover. ISO 21434
governs automotive cybersecurity engineering; this module covers its
testing-relevant pieces: TARA-driven test scoping, the UDS security
access mechanism, and how penetration-testing thinking applies to an
ECU test bench.

!!! note "About this module"
    No physical penetration-testing toolchain or live vehicle network
    was used to produce this content. The TARA model, UDS security
    access flow, and CAPL sketches below reflect documented ISO 21434
    and UDS (ISO 14229) standards — actual penetration testing against
    a real vehicle network requires specific authorization and
    specialized tooling well beyond this course's scope.

## TARA: the cybersecurity equivalent of the FMEA

Threat Analysis and Risk Assessment (TARA) is ISO 21434's structured
process for identifying threats and rating their risk — the
cybersecurity parallel to the FMEA that feeds Level 3 Module 6's fault
matrix. A TARA identifies:

| TARA element | Example |
|---|---|
| Asset | The ECU's ability to control braking torque |
| Threat scenario | An attacker on the vehicle's Ethernet/CAN network sends a spoofed braking-torque request |
| Attack path | Compromised infotainment unit → gateway → braking ECU, if the gateway doesn't segment traffic |
| Impact rating | Safety, financial, operational, privacy — same S/E/C-style structured rating spirit as ASIL |
| Cybersecurity Assurance Level (CAL) | Drives how much testing rigor the corresponding requirement needs, the security analog of ASIL |

Just as Level 3 Module 4 tied test rigor to ASIL, cybersecurity test
rigor should tie to CAL — a low-CAL informational ECU doesn't need the
same penetration-testing depth as a high-CAL gateway or braking ECU.

## UDS security access: the mechanism test engineers touch most

Most ECUs restrict sensitive diagnostic services (calibration writes,
reflashing, certain control commands) behind a **Security Access**
(UDS service `0x27`) seed/key exchange:

```text
1. Tester requests a Security Access "request seed" for a given level.
2. ECU responds with a pseudo-random seed.
3. Tester computes a key from the seed using an algorithm (often a
   proprietary or cryptographic function shared out-of-band with the
   supplier — never guessable from the protocol alone).
4. Tester sends the computed key back.
5. ECU verifies it; on success, restricted services unlock for that
   session. On repeated failure, the ECU should enforce an escalating
   lockout delay.
```

## CAPL testing of the security access mechanism

```c
// Illustrative -- CAPL UDS security access flow. The actual key
// algorithm is supplier-specific and provided separately; this
// example uses a placeholder function.
testcase tc_SecurityAccess_CorrectKeyUnlocks()
{
  testCaseTitle("[SEC-REQ-101] Valid key unlocks restricted diagnostic services");
  DiagRequest seedReq = new DiagSecurityAccessRequestSeed(0x03);
  byte seed[4], key[4];

  DiagSendRequest(seedReq);
  testWaitForDiagResponse(seedReq, 1000);
  DiagGetResponseData(seedReq, seed, 4);

  ComputeSecurityKey(seed, key); // supplier-provided algorithm, not shown
  DiagRequest keyReq = new DiagSecurityAccessSendKey(0x03, key);
  DiagSendRequest(keyReq);
  testWaitForDiagResponse(keyReq, 1000);

  testStepCheck("security access granted", DiagGetLastResponseType(keyReq) == POSITIVE_RESPONSE);
}

testcase tc_SecurityAccess_LockoutAfterRepeatedFailures()
{
  testCaseTitle("[SEC-REQ-102] ECU enforces escalating lockout after repeated invalid keys");
  int i;
  byte badKey[4] = {0x00, 0x00, 0x00, 0x00};

  for (i = 0; i < 3; i++)
  {
    DiagRequest seedReq = new DiagSecurityAccessRequestSeed(0x03);
    DiagSendRequest(seedReq);
    testWaitForDiagResponse(seedReq, 1000);

    DiagRequest keyReq = new DiagSecurityAccessSendKey(0x03, badKey);
    DiagSendRequest(keyReq);
    testWaitForDiagResponse(keyReq, 1000);
    testStepCheck("invalid key correctly rejected", DiagGetLastResponseType(keyReq) == NEGATIVE_RESPONSE);
  }

  // A 4th attempt, per most UDS security implementations, should now
  // be rejected immediately with a "required time delay not expired"
  // negative response code, without even issuing a new seed.
  DiagRequest lockedReq = new DiagSecurityAccessRequestSeed(0x03);
  DiagSendRequest(lockedReq);
  testWaitForDiagResponse(lockedReq, 1000);
  testStepCheck("lockout delay enforced on 4th attempt",
                 DiagGetNegativeResponseCode(lockedReq) == NRC_REQUIRED_TIME_DELAY_NOT_EXPIRED);
}
```

`tc_SecurityAccess_LockoutAfterRepeatedFailures` matters as much as
the positive-path test — a security mechanism with no lockout, or a
lockout an attacker can trivially reset (e.g., by power-cycling the
ECU), fails its actual security purpose even if the seed/key exchange
itself is cryptographically sound.

## Penetration-testing thinking, adapted to a bus-level test bench

A test engineer without a dedicated red-team toolchain can still apply
adversarial thinking within the CAPL/CANoe skillset already built:

| Adversarial question | How to test it with existing tooling |
|---|---|
| Can an unauthorized node spoof a message from a trusted sender? | Send the exact frame ID/payload from an unauthorized simulated node and check whether the receiving ECU has any sender-authentication mechanism (e.g., a MAC/counter in the payload) that should reject it |
| Does a malformed diagnostic request crash or hang the ECU rather than rejecting it cleanly? | Fuzz UDS request payloads with malformed lengths/subfunctions (see Level 2's DTC module for well-formed request structure) and confirm the ECU always returns a clean negative response, never becomes unresponsive |
| Is a security-relevant DTC actually logged when an attack is attempted? | Verify a failed security access attempt sets an expected security-event DTC, not just a silent rejection |

## Where this module's coverage ends

Genuine automotive penetration testing — network protocol fuzzing at
scale, side-channel/timing attacks on cryptographic implementations,
firmware reverse engineering — requires specialized tooling, training,
and typically a dedicated red-team function with explicit
authorization. This module's scope is limited to the adversarial
*testing habits* a functional/HIL test engineer can and should apply
within their existing CAPL/CANoe skillset — recognizing that
boundary honestly is itself part of professional practice, the same
way Level 3 Module 9 drew an honest line around SOTIF's Area 3
scenario discovery.

## Cheat sheet

| Concept | Key point |
|---|---|
| TARA | Cybersecurity's FMEA-equivalent — threats, attack paths, CAL ratings drive test rigor |
| CAL | The security analog of ASIL — higher CAL needs deeper testing |
| UDS Security Access (0x27) | Seed/key exchange gating sensitive diagnostic services |
| Lockout testing | As important as key-validation testing — a resettable/absent lockout defeats the mechanism |
| Adversarial habits within existing tooling | Spoofing, malformed-input, and logging checks are reachable with CAPL; deep pentesting is not |

## Exercise

1. `tc_SecurityAccess_LockoutAfterRepeatedFailures` checks a 4th
   attempt is rejected. Extend it to check that power-cycling the ECU
   between the 3rd and 4th attempt does NOT reset the lockout counter
   — write the testcase structure (the actual power-cycle mechanism
   depends on rig capability, so describe what HIL primitive from
   Level 3 Module 1 you'd need).
2. Design a spoofing test for a message that includes a rolling
   counter or MAC field intended to prevent replay — what exactly
   would you send, and what response from the ECU would indicate the
   anti-replay mechanism is actually working versus merely present in
   the DBC/A2L definition but unenforced?
3. Explain, referencing the CAL concept, why a low-CAL courtesy-light
   ECU and a high-CAL telematics gateway should not receive the same
   security test depth, and sketch what a CAL-based test-tiering table
   (parallel to Level 3 Module 4's ASIL-based rigor table) would look
   like.
