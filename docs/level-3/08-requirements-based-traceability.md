---
description: "Requirements-Based Test Traceability — A test suite that isn't linked back to requirements can't answer the one question every audit, safety assessment…"
---

# 08 · Requirements-Based Test Traceability

A test suite that isn't linked back to requirements can't answer the
one question every audit, safety assessment, or release decision asks:
*have we tested everything we were supposed to?* This module covers
the traceability model — requirement → test case → result — and how
to build and maintain it without it becoming pure bureaucratic
overhead.

!!! note "About this module"
    No specific ALM/requirements tool (DOORS, Polarion, Jama, etc.)
    was used to produce this content. The traceability model and
    CAPL/metadata patterns below reflect common industry practice and
    the documented structure ISO 26262 and ASPICE both expect —
    verify exact tool integration against your project's actual ALM
    system.

## The traceability chain

```text
Stakeholder need
   -> System requirement
        -> Software requirement
             -> Test case (this is where CAPL testcases attach)
                  -> Test result (pass/fail, with evidence)
```

Traceability means every link in this chain is explicit and queryable
in both directions:

- **Forward**: "Requirement SW-REQ-042 — show me every test case that
  verifies it, and their latest results."
- **Backward**: "Test case `tc_DtcSetsAfterPersistentFault` — show me
  which requirement(s) it verifies, so I know why it exists and
  whether it's still needed if that requirement changes."

Without backward traceability specifically, a suite accumulates
testcases nobody can explain — Module 7's "suite rot" — because no one
can tell whether a testcase is still validating something that
matters.

## Coverage: not just "do we have a test," but "how well"

| Coverage question | What it checks |
|---|---|
| Requirement coverage | Does every requirement have at least one test case? |
| Verdict coverage | Of covered requirements, how many have a *passing* latest result, versus never-run or failing? |
| ASIL-weighted coverage | For safety-relevant requirements (Module 4), is the required rigor (independent review, specific test technique) actually met, not just "a test exists"? |
| Equivalence-class coverage | For a given requirement, does the test suite cover its boundary/edge cases (Level 1 exercise technique), or just one nominal case? |

A requirement with "1 test case, currently failing" is very different
from "1 test case, currently passing, but it only exercises the
nominal case and never the boundary the requirement explicitly calls
out" — a naive coverage dashboard showing "100% requirements covered"
can hide the second problem entirely.

## Tagging CAPL testcases for traceability

```c
// Illustrative — the exact mechanism for attaching requirement IDs
// to CAPL testcases varies by CANoe version and ALM integration;
// a title-embedded ID is the lowest-common-denominator approach that
// works with any downstream tooling that can parse test reports.
testcase tc_DtcSetsAfterPersistentFault()
{
  testCaseTitle("[SW-REQ-042] DTC sets after fault persists past debounce time");
  // ... test body from Level 2 Module 5
}

testcase tc_DtcDoesNotSetOnTransientFault()
{
  testCaseTitle("[SW-REQ-042][SW-REQ-043] Transient fault under debounce time does not set DTC");
  // one testcase can legitimately cover multiple related requirements
}
```

A parsing script in the Module 5 orchestration layer can then extract
`[SW-REQ-###]` tokens from every test report's testcase titles and
build the coverage matrix automatically, rather than maintaining it by
hand in a spreadsheet that inevitably drifts out of sync with the
actual CAPL source.

```python
import re

REQ_PATTERN = re.compile(r"\[(SW-REQ-\d+)\]")

def build_coverage_matrix(test_report_results):
    matrix = {}
    for testcase_name, title, verdict in test_report_results:
        for req_id in REQ_PATTERN.findall(title):
            matrix.setdefault(req_id, []).append((testcase_name, verdict))
    return matrix

# matrix["SW-REQ-042"] -> [("tc_DtcSetsAfterPersistentFault", "Passed"),
#                          ("tc_DtcDoesNotSetOnTransientFault", "Passed")]
```

## Requirements that changed but tests didn't

The most common traceability failure isn't missing links — it's
**stale** links: a requirement changes (a debounce time moves from
100ms to 150ms) but the linked test case still asserts the old value
and still passes, because it was never touched. This is a false
positive of the worst kind — the dashboard shows green, but the test
no longer verifies the current requirement.

| Practice | How it catches staleness |
|---|---|
| Requirement change triggers a review flag on linked test cases | Most ALM tools support this if the link is bidirectional and maintained |
| Test case includes the requirement's specific numeric value in its title/assertion, not just the ID | A reviewer diffing the requirement text against the test spots the mismatch directly |
| Periodic trace audit (e.g., every release) | Manually sample-check that N random links still match current requirement text |

## Traceability at review/audit time

For ASIL-rated requirements (Module 4) or ASPICE-assessed processes,
an assessor typically asks for exactly this artifact: pick a
requirement, show its test case(s), show the latest passing evidence,
show the test case's review record. A traceability matrix that only
exists informally in engineers' heads fails this check immediately,
regardless of how good the actual testing was — the paper trail is
part of the deliverable, not an afterthought to it.

## Cheat sheet

| Concept | Key point |
|---|---|
| Forward/backward traceability | Requirement → tests AND test → requirement, both queryable |
| Coverage ≠ existence | A passing test that only covers the nominal case isn't full coverage of a requirement with boundary conditions |
| Tag testcases explicitly | Embed requirement IDs in titles/metadata so coverage matrices can be generated, not hand-maintained |
| Stale links | The most common real failure mode — requirement changed, test didn't |
| Audit readiness | The traceability artifact itself is often literally what an assessor asks to see |

## How It Actually Works: traceability as a bipartite graph, and where the regex approach breaks

The `build_coverage_matrix` function above is doing something worth
naming precisely: it's constructing a **bipartite graph** — one node
set for requirements, one for testcases, an edge for every `[SW-REQ-###]`
tag found — and every coverage question this module asks is really a
graph-traversal query over that structure. "Forward traceability" is
following edges from a requirement node outward; "backward
traceability" is following them from a testcase node back; an "orphan"
requirement (no test cases at all) is simply a requirement node with
**degree zero** — no edges — trivially detectable by one pass over the
graph without any special-casing. This is why a graph representation,
even an informal one built from regex-extracted tags, scales far better
than a spreadsheet: spreadsheet-based traceability tends to represent
only one direction explicitly (a "requirement → test IDs" column) and
leaves the reverse direction to manual cross-referencing, which is
exactly where staleness and orphaned links first go unnoticed.

But the regex-tagging approach has a real, mechanical blind spot worth
naming: `REQ_PATTERN.findall(title)` can only discover an edge that
someone remembered to type into a `testCaseTitle()` string. If
SW-REQ-042's debounce value changes from 100 ms to 150 ms and a
developer edits the CAPL assertion but never touches the title string,
the graph edge stays exactly as it was — the parser has no way to know
the *assertion content* underneath that edge drifted out of sync with
the requirement, because it never reads assertion values, only title
text. This is the concrete, structural reason this module's second
staleness-catching practice ("include the requirement's specific
numeric value in the title itself, not just the ID") works where a
periodic manual audit alone is unreliable: putting `150ms` in the title
string turns a value that would otherwise be invisible to the
graph-extraction tooling into another parseable token — a reviewer (or
even an automated diff against the requirements document's own stated
value) can now catch a title that says `100ms` next to a requirement
document that says `150ms`, entirely mechanically, without executing the
test or reading the CAPL body at all.

## 🔀 Related lessons on other tracks

- [Cpp Testing — 03 · Requirements-Based Testing & Traceability](https://sigilipelli.github.io/cpp-testing-mastery-path/level-4/03-requirements-based-testing/)
- [S32K Automotive — Manufacturing, EOL Test & Traceability](https://sigilipelli.github.io/s32k-mastery-path/level-4/09-manufacturing-eol-test/)

## Exercise

1. `tc_DtcSetsAfterPersistentFault` asserts a 100ms debounce time.
   SW-REQ-042 changes to 150ms. Describe, step by step, how this
   staleness would be caught (or wouldn't) under each of the three
   staleness-catching practices in the table above.
2. Design a coverage report format (columns/fields) that distinguishes
   "requirement has a passing nominal-case test only" from
   "requirement has passing nominal AND boundary-case tests," using
   the equivalence-class coverage idea.
3. An assessor asks for evidence that every ASIL C or D requirement's
   test cases were independently reviewed. What metadata would your
   tagging scheme need to add to `tc_DtcSetsAfterPersistentFault`'s
   title or test report to answer that question automatically?
