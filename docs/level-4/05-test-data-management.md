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
