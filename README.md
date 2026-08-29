# Automotive Testing Mastery Path

A free, structured, module-wise training program for automotive ECU
testing — entry level to master level, covering CAN bus fundamentals,
Vector CANoe, CAPL scripting, UDS diagnostics, and Hardware-in-the-Loop
(HIL) testing. This track covers specialized commercial tooling and real
ECU/HIL benches that cannot be executed on a general-purpose machine —
content is written for technical accuracy against genuinely documented
CAN/CAPL/UDS conventions rather than live execution.

**Live site:** https://sigilipelli.github.io/automotive-testing-mastery-path/

## Contents

- **Level 1 · Entry** — what automotive testing is, CAN bus fundamentals, CAN frame structure, introduction to Vector CANoe, CAPL scripting basics, UDS diagnostics basics, introduction to HIL testing, test case design for ECUs, automotive test standards overview, CAPL test script project
- **Level 2 · Intermediate** — advanced CAPL scripting, CANoe test modules & test cases, CAN-FD basics, LIN bus basics, DTC deep dive, DBC & signal definitions, automated test sequences in CANoe, restbus simulation, HIL test bench components, automated CAN signal validation project
- **Level 3 · Advanced** — advanced HIL test automation, CANape & measurement/calibration, automotive Ethernet, functional safety testing (ISO 26262), test automation frameworks, fault injection testing, regression test suites, requirements-based traceability, SOTIF, HIL test plan project
- **Level 4 · Master** — test strategy for vehicle programs, continuous testing, advanced fault injection, cybersecurity testing (ISO 21434), test data management, building a test automation team, supplier/OEM collaboration, homologation & compliance, career growth, capstone ECU test strategy

## Local development

```bash
python3 -m venv .venv
.venv/bin/pip install mkdocs-material
.venv/bin/python -m mkdocs serve
```

## Related

- [S32K Automotive Embedded Mastery Path](https://sigilipelli.github.io/s32k-mastery-path/)
- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
- [C/C++ Testing Mastery Path](https://sigilipelli.github.io/cpp-testing-mastery-path/)
- [FreeRTOS Mastery Path](https://sigilipelli.github.io/freertos-mastery-path/)
