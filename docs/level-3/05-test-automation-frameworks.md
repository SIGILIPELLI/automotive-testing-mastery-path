# 05 · Test Automation Frameworks for ECU Testing

Every CAPL testcase built so far lives inside CANoe's own test module
system. At scale — hundreds of testcases, multiple ECUs, multiple
rigs, results that need to feed a dashboard — teams usually wrap that
core execution engine in a broader automation framework. This module
covers the layers such a framework has, where Python/COM fits
alongside CAPL, and how to structure a suite so it stays maintainable.

!!! note "About this module"
    No specific vendor framework installation was used to produce
    this content. The architecture and COM-interface patterns below
    reflect documented CANoe COM API and common industry test-
    framework practice — verify exact API names against your CANoe
    version's COM documentation.

## Why wrap CAPL at all

CAPL testcases are excellent at driving bus traffic and checking
signal-level behavior close to real time. They're less good at:

- Orchestrating tests across **multiple tools** (CANoe + CANape + a
  power supply + a thermal chamber)
- Producing results in a format a CI system or management dashboard
  can consume
- Parameterizing large data-driven matrices from an external file
  (a spreadsheet of pass/fail limits per variant) without hand-editing
  CAPL source
- Managing test *selection* — running only a subset relevant to a
  specific ECU variant or change

A test automation framework sits one layer above CANoe/CAPL,
typically driving it via its **COM (Component Object Model)
interface**, and handles orchestration, reporting, and environment
setup that CAPL alone isn't designed for.

## The layered architecture

| Layer | Responsibility | Typical tech |
|---|---|---|
| Execution core | Actually send/receive frames, run testcases in real time | CANoe + CAPL test modules |
| Orchestration | Sequence testcases, manage rig state, select which suite to run | Python (or similar) via CANoe's COM API |
| Data layer | Test parameters, expected results, traceability IDs | Spreadsheet/CSV, database, or requirements tool |
| Reporting | Aggregate results, produce human- and CI-readable output | JUnit XML, HTML report, dashboard integration |

## Driving CANoe from Python via COM

```python
# Illustrative — exact COM interface names depend on your CANoe
# version. Requires a Windows host with CANoe installed and the
# pywin32 package for COM access.
import win32com.client

app = win32com.client.Dispatch("CANoe.Application")
app.Open(r"C:\Projects\ECU_TestSuite\ECU_TestSuite.cfg")

measurement = app.Measurement
measurement.Start()

test_env = app.Configuration.TestSetup.TestEnvironments.Item(1)
test_module = test_env.TestModules.Item("ABS_FunctionalTests")
test_module.Start()

while test_module.IsRunning:
    pass  # a real framework polls with a timeout and progress callback, not a busy loop

report = test_module.Report
print(f"Verdict: {report.Verdict}")   # e.g. "Passed" / "Failed"
measurement.Stop()
app.Quit()
```

This gives an orchestration layer the hooks CAPL alone can't:
selecting which test module to run based on an external parameter,
looping across a matrix of `.cfg` variants for different ECU
configurations, and pulling a machine-readable verdict out for a CI
pipeline — all without touching the CAPL test logic itself.

## Data-driven testing: separating data from logic

The same discipline from Level 3 Module 1's scenario parameterization
extends to the framework layer — keep the **matrix of test data**
outside the code entirely:

```python
import csv

with open("abs_activation_matrix.csv") as f:
    for row in csv.DictReader(f):
        run_abs_scenario(
            speed_kph=float(row["speed_kph"]),
            friction=float(row["friction"]),
            expected_max_ms=float(row["expected_max_ms"]),
        )
```

```text
speed_kph,friction,expected_max_ms
60,0.15,250
120,0.15,250
60,0.05,300
```

Adding a new scenario becomes a spreadsheet edit, not a code change —
important when the people who know the right test values (calibration
or systems engineers) aren't the people writing CAPL or Python.

## Result reporting for CI

A framework only earns its keep if results land somewhere useful.
JUnit XML is a common lowest-common-denominator format most CI systems
(Jenkins, GitLab CI, Azure DevOps) already understand:

```python
def write_junit_report(results, path):
    with open(path, "w") as f:
        f.write('<?xml version="1.0" encoding="UTF-8"?>\n')
        f.write(f'<testsuite name="ABS_FunctionalTests" tests="{len(results)}">\n')
        for name, verdict, duration_s in results:
            f.write(f'  <testcase name="{name}" time="{duration_s}">\n')
            if verdict != "Passed":
                f.write(f'    <failure message="verdict={verdict}"/>\n')
            f.write('  </testcase>\n')
        f.write('</testsuite>\n')
```

Feeding this into CI turns a rig run into a pass/fail gate on a merge
request, the same role unit test reports play in pure-software
projects — the ECU-testing equivalent connects CANoe's CAPL verdicts
to the same visibility developers already expect.

## Structuring a suite that scales

| Anti-pattern | Better structure |
|---|---|
| One giant CAPL test module with 200 testcases | Multiple test modules grouped by feature/ECU area, orchestrated together by the Python layer |
| Hardcoded rig IP/COM port in every script | A single environment-config file the orchestration layer reads once |
| Pass/fail limits buried in CAPL source | External data file (CSV/spreadsheet), read at test-setup time |
| No traceability from testcase name to requirement | Testcase names or metadata carry a requirement ID, feeding Module 8's traceability practice |

## Cheat sheet

| Concept | Key point |
|---|---|
| Execution vs. orchestration | CAPL/CANoe executes in real time; Python/COM orchestrates across tools and variants |
| COM interface | The standard way to drive CANoe programmatically from outside CAPL |
| Data-driven testing | Keep scenario data in external files, not hardcoded in test logic |
| JUnit XML | Common bridge from rig-level verdicts to CI dashboards |

## Exercise

1. Rewrite the busy-wait `while test_module.IsRunning: pass` loop
   above to include a timeout and a periodic progress log, and explain
   what happens to a CI pipeline if a hung test module is polled with
   no timeout at all.
2. Design the CSV schema for a data-driven suite testing DTC
   set/clear thresholds (Level 2 Module 5) across three different
   fault durations, and write the Python loop that would drive it.
3. A team's CAPL test module names are `Test1`, `Test2`, `Test3` with
   no link to requirements. Propose a naming or metadata convention
   that would let the JUnit report above double as traceability
   evidence, without changing the CAPL execution logic itself.
