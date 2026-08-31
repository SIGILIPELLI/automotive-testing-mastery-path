# 02 · Continuous Testing in Automotive

Level 3 Module 5 built a framework wrapping CAPL/CANoe for
orchestration; Level 3 Module 7 built a tiered regression suite. This
module covers what changes when those pieces are wired into an
always-on CI/CD pipeline against real (or virtual) ECU targets — the
biggest addition being that "the hardware" is now a shared, contended
resource that software CI pipelines don't normally have to think
about.

!!! note "About this module"
    No specific CI platform or physical rig farm was used to produce
    this content. The pipeline structure and rig-scheduling patterns
    below reflect common industry practice adapting standard CI/CD
    concepts to automotive HIL constraints — verify exact tooling
    against your project's actual CI system.

## Why automotive CI differs from pure software CI

| Pure software CI | Automotive HIL-involved CI |
|---|---|
| A build runs on a cheap, disposable, instantly-scalable VM | A HIL rig is expensive, physical, and cannot be trivially cloned |
| Tests are naturally parallelizable across many workers | Rig time is a bottleneck — parallelism is capped by the number of physical rigs |
| A "flaky" failure is almost always a test/code problem | A flaky failure can be a test problem, a rig wiring problem, or a genuine ECU timing issue — Level 3 Module 7's triage gets harder |
| Rollback = revert a commit | Rollback may mean re-flashing an ECU to a prior software build, a slower and more deliberate operation |

## The pipeline stages

```text
Commit
  -> Static analysis / build (compile the ECU software, MISRA checks)
  -> Software-in-the-loop (SIL) smoke tests (no hardware — fast, first gate)
  -> Flash to a shared HIL rig pool
  -> Smoke tier (Level 3 Module 7) on real/HIL-simulated hardware
  -> Functional tier (scheduled — rig capacity allows only so many
     concurrent full runs)
  -> Nightly/weekly full regression + fault injection
```

SIL testing before any hardware step matters specifically for
automotive CI: if the same CAPL/CANoe test logic can run against a
**simulated** ECU model before ever touching a physical rig, most
basic defects get caught in the cheap, infinitely-parallel SIL stage,
reserving scarce rig time for what actually needs real hardware
(timing-sensitive, electrical-level, or genuinely HIL-only checks).

## Rig scheduling: the automotive-specific bottleneck

```python
# Illustrative — a minimal rig-reservation scheduler concept for
# a CI pipeline with more commits than available physical rigs.
class RigPool:
    def __init__(self, rig_ids):
        self.available = set(rig_ids)
        self.reserved = {}

    def reserve(self, job_id, ecu_variant):
        # Match the job to a rig actually wired for that ECU variant --
        # not every rig in the pool carries every harness configuration.
        candidates = [r for r in self.available if rig_supports(r, ecu_variant)]
        if not candidates:
            return None  # job queues, does not fail -- capacity, not a defect
        rig = candidates[0]
        self.available.remove(rig)
        self.reserved[job_id] = rig
        return rig

    def release(self, job_id):
        rig = self.reserved.pop(job_id)
        self.available.add(rig)
```

The key design point: a job that can't get rig time should **queue**,
not report a false failure — conflating "no rig capacity right now"
with "the test failed" is a common and confusing automotive CI defect
that erodes the same trust Level 3 Module 7 warned about for flaky
tests.

## Handling flash time and rig state between runs

Unlike a software VM that resets to a known state trivially, a
physical ECU's flash memory and a HIL rig's fault-injection relays
(Level 3 Module 6) can carry state between jobs if not explicitly
reset:

| Risk | Mitigation |
|---|---|
| Job N leaves a fault-injection relay open; job N+1 runs against a mis-wired rig | Every job's teardown step must explicitly restore all relays to nominal, verified by a rig self-check before the next job starts |
| Wrong software build left flashed from a previous, unrelated job | Every job's setup step re-flashes and verifies the build ID via UDS `ReadDataByIdentifier` before running any test |
| Two jobs interleaved on the same rig due to a scheduler bug | Rig reservation must be atomic/locked, not just advisory |

```c
// CAPL setup-step build verification, run before every CI job's
// actual test suite begins.
testcase tc_Setup_VerifyCorrectSoftwareBuildFlashed(char expectedBuildId[])
{
  DiagRequest req = new DiagReadDataByIdentifier(0xF1F0); // illustrative DID for build ID
  DiagSendRequest(req);
  testWaitForDiagResponse(req, 1000);
  testStepFail_IfNot("correct build flashed",
                      strncmp(DiagGetResponseData(req), expectedBuildId, strlen(expectedBuildId)) == 0);
}
```

Failing loudly here — before running a single functional test — turns
a "why did 40 testcases fail mysteriously" investigation into a single
clear, immediate signal.

## Feedback speed vs. thoroughness

Continuous testing lives or dies on how fast a developer gets useful
feedback:

| Stage | Target feedback time | What it can afford to skip |
|---|---|---|
| SIL smoke | Minutes | Anything needing real timing/electrical behavior |
| HIL smoke | Tens of minutes | Full fault-injection matrix |
| Full HIL regression | Hours (nightly) | Nothing — this is the thorough pass |

A developer who has to wait for the nightly full regression to learn
their commit broke smoke-level behavior will simply stop trusting or
watching CI — the tiering from Level 3 Module 7 is what makes fast
feedback and thorough coverage coexist instead of trading off against
each other.

## Cheat sheet

| Concept | Key point |
|---|---|
| SIL-before-HIL | Catch cheap defects in simulation before spending scarce rig time |
| Rig as bottleneck | Physical rig capacity, not compute, caps parallelism — schedule, don't just queue-and-hope |
| Queue vs. fail | No rig capacity is a scheduling state, never a false test failure |
| State reset between jobs | Explicit relay/build-verification teardown and setup on every job, not assumed |
| Feedback tiering | Fast SIL/smoke feedback for developers; thorough nightly regression for full confidence |

## Exercise

1. A CI dashboard shows a job as "Failed" but the actual cause was no
   rig being available for two hours. Redesign the `RigPool.reserve`
   logic above so this state is visibly distinct from a genuine test
   failure in the CI UI.
2. Design the rig self-check testcase that should run at the START of
   every CI job (before `tc_Setup_VerifyCorrectSoftwareBuildFlashed`)
   to confirm all fault-injection relays are in their nominal
   (non-faulted) state, referencing Level 3 Module 6's fault types.
3. A team wants to add a 45-minute fault-injection matrix to the
   per-commit (not nightly) pipeline stage because "it caught a real
   bug once." Using the feedback-speed table, argue for or against
   this change, and propose an alternative that preserves fast
   feedback while still increasing fault-injection frequency.
