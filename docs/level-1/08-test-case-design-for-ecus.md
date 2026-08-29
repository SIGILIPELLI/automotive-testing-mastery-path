# 08 · Test Case Design for ECUs

The general test-design techniques you'd learn in any software testing
course — equivalence partitioning, boundary value analysis, clear
documented expected results — apply directly to automotive ECU testing.
What changes is *what you're partitioning*: instead of arbitrary function
parameters, you're almost always partitioning **signal ranges** — the
physical values a CAN signal, a UDS DID, or a sensor input can carry —
and every case needs to specify the exact bus-level or diagnostic-level
representation of its test data, not just an abstract value.

## Equivalence partitioning, applied to a signal

**Equivalence partitioning** groups inputs into classes where every
member of a class should be treated identically by the system, so you
only need to test one representative from each class rather than every
possible value.

Take the `CoolantTemp` signal from the DBC snippet used in Module 2 of
the S32K course's CAN fundamentals lesson — physical range −40 to 150 °C,
scale 0.1 °C/bit — feeding an ECU that raises a "coolant overheat"
warning above 120 °C:

| Partition | Physical range | Representative value | Expected system behavior |
|---|---|---|---|
| Invalid — below sensor spec | < −40 °C | −45 °C | Signal treated as implausible/out-of-range fault |
| Valid — normal operation | −40 °C to 120 °C | 90 °C | No warning |
| Valid — overheat | > 120 °C to 150 °C | 135 °C | Overheat warning active |
| Invalid — above sensor spec | > 150 °C | 155 °C | Signal treated as implausible/out-of-range fault |

Four partitions, four representative test cases — not an exhaustive
sweep of every possible 16-bit raw value, which would be both impossible
and pointless once you've reasoned about which values behave alike.

## Boundary value analysis, applied to a threshold

**Boundary value analysis** focuses specifically on the edges between
partitions, because that's overwhelmingly where off-by-one and
comparison-operator (`>` vs `>=`) defects live. For the 120 °C overheat
threshold above, assuming the requirement reads "above 120 °C" (strictly
greater than):

| Test value | Expected result | Why it's chosen |
|---|---|---|
| 119.9 °C | No warning | Just below the boundary |
| 120.0 °C | No warning | Exactly at the boundary — the case a `>` vs `>=` bug lives on |
| 120.1 °C | Warning active | Just above the boundary |

Boundary pairs always come in twos or threes like this — a test suite
that only checks 90 °C and 135 °C (deep inside each partition) will
never catch a `>=` written where `>` was required, or vice versa. This
is the single most common category of real ECU logic defect, and it's
exactly why boundary cases, not just "one value per partition," are
mandatory in a serious automotive test plan.

## Writing the test case: the fields that matter for a bus-level test

Reusing the general test-case template (from general testing
methodology) with automotive-specific fields added:

| Field | Purpose | Automotive-specific note |
|---|---|---|
| Test Case ID | Unique, stable identifier | Often tied to a requirement-numbering scheme for traceability (Level 3, Module 8) |
| Requirement ID | Links back to the spec | Automotive requirements are frequently version-controlled, formally reviewed documents |
| Preconditions | State before step 1 | ECU session state (Module 6) and vehicle mode (ignition on, engine running, vehicle speed) often matter here |
| **Test data (bus-level)** | Exact bytes/signals to send | Must specify raw CAN bytes *or* the signal value plus the DBC scaling used to derive them — not just "send 120 °C" |
| Steps | Numbered, imperative | Often literally "transmit frame X with these bytes," "wait N ms," "read DID Y" |
| Expected result | Precise, observable | For ECU tests: often a specific CAN message/signal value, a UDS response, or a physical output state — with a tolerance where floating comparisons apply |
| Environment | Reproducibility | ECU hardware/software version, CANoe configuration version, database version — a stale DBC has caused many false test failures |

### A fully worked test case

For the boundary above:

| Field | Value |
|---|---|
| **Test Case ID** | TC-COOLANT-014 |
| **Title** | Coolant overheat warning activates exactly above 120 °C, not at or below |
| **Requirement ID** | REQ-COOLANT-009 |
| **Priority** | P1 |
| **Type** | Boundary |
| **Preconditions** | Ignition on, engine-running simulated via `EngineRunning = 1` signal, no other active DTCs |
| **Test data** | `SensorNodeStatus.CoolantTemp` = 1200 raw (120.0 °C at 0.1 °C/bit scale) for step 2; 1201 raw (120.1 °C) for step 4 |
| **Steps** | 1. Transmit `SensorNodeStatus` with `CoolantTemp = 1200` at the node's normal 100 ms cycle for 500 ms.<br>2. Read the `OverheatWarning` signal.<br>3. Change `CoolantTemp` to `1201` and continue transmitting for 500 ms.<br>4. Read the `OverheatWarning` signal again. |
| **Expected result** | After step 2: `OverheatWarning == 0`. After step 4: `OverheatWarning == 1` within one signal update cycle (≤ 100 ms) of the value change. |
| **Environment** | CANoe 14.x config `coolant_test.cfg`, database version `body_v3.2.dbc`, ECU software build `1.4.2` |
| **Notes** | Pair with TC-COOLANT-013 (119.9 °C, must stay warning-off) to fully bracket the boundary from both sides. |

Notice the expected result specifies **both** the value and the timing
("within one signal update cycle") — an ECU test that only checks the
final value and ignores latency can pass a design that's dangerously
slow to react.

## Common automotive-specific test data classes

Beyond the general positive/negative/boundary/error-handling families,
ECU signal testing has some recurring classes worth a permanent mental
checklist:

| Class | Example | Why it matters |
|---|---|---|
| **Implausible/out-of-sensor-range** | A temperature signal reading 300 °C | Tests the ECU's plausibility checking, not just its math |
| **Frozen/stale signal** | An alive counter that stops incrementing | Tests timeout/failsafe logic (Module 5's watchdog pattern) |
| **Rapid oscillation at a boundary** | A value bouncing between 119.9 and 120.1 °C | Tests for hysteresis — does the warning chatter on/off, or is there a debounce/hysteresis band? |
| **Simultaneous multi-signal edge cases** | Overheat *and* low coolant level at once | Tests priority/interaction between independent fault conditions |
| **Out-of-session / unauthorized diagnostic request** | A `0x2E` write attempted without security access | Tests the negative-response path from Module 6 |

That "rapid oscillation" row deserves a specific callout: a well-designed
ECU threshold usually has **hysteresis** — e.g., warning turns on above
120 °C but only turns off below 115 °C — specifically to prevent a
warning light flickering on and off around a noisy boundary. A test
plan that only checks the single 120 °C threshold both ways, without
checking whether hysteresis exists, misses a very common real
requirement.

## Cheat sheet

| Technique | What it does | Automotive application |
|---|---|---|
| Equivalence partitioning | Group inputs that should behave identically | Partition a signal's physical range into valid/invalid/behaviorally distinct bands |
| Boundary value analysis | Test exactly at and adjacent to partition edges | Threshold-triggered warnings, DTC set/clear points, range limits |
| Bus-level test data | Specify exact raw bytes or signal+scale | Never just "send 120 °C" — always the actual frame/DID content |
| Timing in expected results | State the allowed latency, not just the final value | ECU reactions are rarely instantaneous — spec the deadline |
| Hysteresis check | Test both the "on" and "off" thresholds separately | Prevents missing chatter/flicker defects at a boundary |
| Environment field | Record ECU build, database version, CANoe config version | A stale DBC or wrong ECU build is a very common false failure |

## Exercise

An ECU has a low-fuel warning requirement: "the low-fuel warning shall
activate when fuel level drops below 10%, and shall deactivate only once
fuel level rises above 15% (hysteresis to prevent flicker)."

1. Identify the **equivalence partitions** for the fuel-level signal
   with respect to this requirement (at least three), each with a
   representative value.
2. Identify **all** the boundary values worth testing, including both
   the activation boundary (10%) and the deactivation boundary (15%) —
   list each value and its expected `LowFuelWarning` result.
3. Write one fully documented test case (using the template above) for
   the single trickiest boundary case: fuel level drops to exactly 9%
   (warning activates), rises to 12% (should the warning still be on?),
   then rises further to 16%. Specify the expected `LowFuelWarning`
   value at each of the three readings, and explain in your notes field
   why the 12% reading is the one that actually proves the hysteresis
   logic works.
