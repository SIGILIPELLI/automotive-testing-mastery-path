---
description: "Test Data Management at Scale — A single test suite has a handful of DBC files, one A2L, and a spreadsheet of scenario data. A vehicle program has dozens…"
---

# 05 · Test Data Management at Scale

A single test suite has a handful of DBC files, one A2L, and a
spreadsheet of scenario data. A vehicle program has dozens of ECU
variants, software builds shipping monthly, and years of accumulated
test results — and none of that is useful if nobody can answer "which
exact DBC version, A2L, and software build produced this specific
passing result six months ago?" This module covers versioning,
configuration management, and results-data practice at program scale.

!!! note "About this module"
    No specific ALM/data-management platform was used to produce this
    content. The versioning and configuration-management patterns
    below reflect common industry practice building on the DBC/A2L
    pairing concerns raised in Levels 2–3 — adapt to your program's
    actual tooling.

## What has to be versioned together

The single biggest test-data-management failure mode at scale is
**version skew** between artifacts that must match exactly:

| Artifact | Must match |
|---|---|
| DBC file | The exact CAN network/signal definition the ECU software build implements |
| A2L file | The exact software build's memory layout (Level 3 Module 2) |
| ECU software build | The specific binary flashed to the rig |
| Test case source (CAPL) | The requirement version it was written against (Level 3 Module 8) |
| Fault-injection matrix | The FMEA revision it traces to (Level 3 Module 6) |

A **configuration baseline** — a single record naming the exact
version of every one of these that belongs together — is the artifact
that makes "reproduce this result" possible at all:

```text
Baseline: AEB-DomainController-v2.4.1-baseline
  ECU software build:      2.4.1-rc3
  DBC:                     ADAS_Network_v18.dbc  (checksum: a1b2c3...)
  A2L:                     AEB_DC_2.4.1.a2l      (checksum: d4e5f6...)
  CAPL test suite tag:     test-suite-v2.4.1-align
  Requirements baseline:   SW-REQ-set-2024-Q3-rev4
  FMEA revision:           FMEA-AEB-rev7
```

Without this record, a test result from six months ago is nearly
worthless as evidence — you cannot tell an assessor (Level 3 Module 8)
what, precisely, was tested.

## Version control for non-code artifacts

DBC, A2L, and CAPL test source are all text-ish or structured-binary
files that benefit from the same discipline as software source code —
version control, code review, and a clear branching/tagging model —
but teams sometimes treat them as loose files on a shared drive
instead, which is the single most common root cause of the skew
problem above.

```bash
# Illustrative -- tagging a configuration baseline in a repo that
# holds DBC/A2L/CAPL artifacts alongside the ECU software build ID.
git tag -a "AEB-DomainController-v2.4.1-baseline" \
  -m "DBC=ADAS_Network_v18, A2L=AEB_DC_2.4.1, SW=2.4.1-rc3"
git push origin "AEB-DomainController-v2.4.1-baseline"
```

Checksumming the DBC/A2L files (as in the baseline record above) adds
a cheap integrity check: a test orchestration script (Level 3 Module
5) can verify the checksum of the DBC actually loaded into CANoe
matches the baseline's recorded checksum before running anything,
catching an accidental substitution immediately rather than producing
silently wrong results.

## Test result data: what to retain, and for how long

| Data class | Retention driver | Typical retention |
|---|---|---|
| Raw signal/bus traces from failed tests | Root-cause investigation | Until the defect is closed, often longer for safety-relevant failures |
| Pass/fail verdicts + baseline reference | Traceability/audit evidence (Level 3 Module 8) | Program lifetime, often beyond — safety cases can be revisited years later |
| Full raw traces from passing tests | Storage cost vs. value — rarely needed once passing | Short retention, or summary-only, is common |
| Calibration/CHARACTERISTIC values used during a test (Level 3 Module 2) | Reproducibility of any test that swept a tunable value | Same as the pass/fail verdict retention — a passing threshold-sweep result is meaningless without the exact values swept |

The asymmetry in the table (keep failure traces longer/richer than
passing-test traces) is a deliberate storage-cost trade-off — full bus
traces from thousands of passing HIL runs are expensive to retain
indefinitely and rarely re-examined, while a failure's trace is often
the only evidence available for root-causing an intermittent defect
weeks later.

## A results database schema sketch

```sql
-- Illustrative minimal schema linking a result to its full context.
CREATE TABLE test_result (
  id INTEGER PRIMARY KEY,
  testcase_name TEXT NOT NULL,
  requirement_ids TEXT,          -- e.g. "SW-REQ-201,SW-REQ-202"
  baseline_tag TEXT NOT NULL,    -- e.g. "AEB-DomainController-v2.4.1-baseline"
  verdict TEXT NOT NULL,         -- Passed / Failed / Inconclusive
  run_timestamp TIMESTAMP NOT NULL,
  trace_file_path TEXT,          -- NULL if not retained (e.g., a routine pass)
  rig_id TEXT
);
```

A query like "show every Failed result for any requirement under ASIL
D, in the last 90 days, across all rigs" becomes a straightforward
join against this table plus the requirements/ASIL table from Level 3
Module 8 — the kind of program-level visibility a spreadsheet of
loose result files cannot realistically provide.

## Cheat sheet

| Concept | Key point |
|---|---|
| Version skew | The most common test-data failure — DBC/A2L/software/test-source drifting out of alignment |
| Configuration baseline | One record naming the exact matched version of every artifact involved in a test run |
| Version control for non-code artifacts | DBC/A2L/CAPL deserve the same discipline as source code, not loose shared-drive files |
| Checksum verification | Cheap, automatable guard against accidental artifact substitution |
| Asymmetric retention | Keep failure evidence rich and long; passing-test raw traces can be pruned aggressively |

## How It Actually Works

**Why a DBC checksum mismatch has to block the run, not just log a
warning.** CANoe (and most CAN tools) load a DBC by parsing its
message/signal definitions into an internal symbolic table used at
runtime to translate raw CAN payload bytes into named signals your
CAPL testcase reads and writes (`getSignal(AEB_BrakeCommand)`, etc.).
If a slightly different DBC revision is silently loaded — say, one
where `ForwardDistance_m`'s scaling factor changed from a prior
calibration round — every `setSignal`/`getSignal` call in the test
suite still executes without error, but is now writing or reading a
physically different value than the test author intended, because the
raw-to-physical conversion table changed underneath the test. There is
no runtime exception to catch here — the test appears to run and
report a verdict normally, just against the wrong physical scale. This
is precisely why the checksum check in Level 3 Module 5's orchestration
script has to be a hard pre-flight gate rather than a log line: by the
time a human would notice something's off (an oddly-scaled trace, or a
verdict that doesn't match physical intuition), potentially thousands
of test executions have already produced quietly mis-scaled results.

**Why A2L/software-build pairing failures are worse than DBC ones, and
harder to detect.** An A2L file maps CHARACTERISTIC and MEASUREMENT
names to fixed memory addresses in a *specific compiled binary*
(Level 3 Module 2). If the software build is rebuilt — even with zero
functional code changes, just a different compiler version or a
reordered source file — the linker can place variables at different
addresses, silently invalidating every address in the old A2L. Unlike
a DBC mismatch, which at least produces a wrong-but-plausible physical
value, an A2L/build mismatch used over XCP can read or write to an
address that in the new binary holds a completely unrelated variable
or unallocated memory — at best producing garbage calibration reads,
at worst corrupting unrelated ECU state during a CHARACTERISTIC write.
This is why the baseline record pins the A2L to an exact build ID, not
just a version number: two builds tagged with the "same" human-readable
version can still have diverged addresses if either was rebuilt.

**Why raw-trace retention asymmetry is a real engineering trade-off,
not just cost-cutting.** A full CAN/Ethernet bus trace captures every
frame on the bus for the test's duration — for a modern domain
controller's Ethernet/SOME-IP traffic this can run into gigabytes per
hour of test time. Retaining that at full fidelity for every one of
thousands of passing nightly-regression runs is not merely expensive
storage — it also makes the results database itself slow to query at
scale, since large binary trace blobs sitting alongside the
lightweight pass/fail row (as in the `test_result` schema) bloat
backups, replication, and any full-table scan. The asymmetric policy
— rich traces for failures, summary-only for passes — mirrors why
production systems keep verbose logs briefly but retain structured
metrics indefinitely: the artifact needed for deep debugging and the
artifact needed for long-term audit trail have fundamentally different
size/value profiles and should be stored accordingly.

## 🔀 Related lessons on other tracks

- [Java Testing — 05 · Test Data Management](https://sigilipelli.github.io/java-testing-mastery-path/level-2/05-test-data-management/)
- [Playwright — 08 · Environment & Test Data Management](https://sigilipelli.github.io/playwright-mastery-path/level-2/08-env-test-data/)
- [Python Testing — 05 · Test Data Management & Factories](https://sigilipelli.github.io/python-testing-mastery-path/level-2/05-test-data-management/)

## Exercise

1. A test run six months ago is cited in a safety audit, but the DBC
   file referenced no longer exists on the shared drive it was loaded
   from at the time. Using the configuration-baseline concept, design
   the minimum set of fields that would have prevented this from being
   unrecoverable.
2. Extend the `test_result` schema to also record the specific
   CHARACTERISTIC values used in a threshold-sweep test (Level 3
   Module 2's XCP calibration sweep), and explain why a single
   `trace_file_path` column isn't sufficient for that case.
3. Propose a concrete checksum-verification step to add to the Level
   3 Module 5 Python orchestration script, that fails the job loudly
   (not silently) if the DBC actually loaded doesn't match the
   baseline's recorded checksum.
