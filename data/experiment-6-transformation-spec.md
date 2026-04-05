# Experiment 6: Goal Spec vs Transformation Spec

## Hypothesis

A spec that describes the transformation from current state produces fewer bugs than a spec describing the goal state, because it anchors the model's representation precisely and reduces exploration.

## Test case

VY-200: Add `cmd.Wait()` after `Process.Kill()` in idle timeout and context cancellation paths.

Known correct output: add `cmd.Wait()` on two paths in `runClaude`.

## Spec variants

**Goal spec** (PR #131): Describes the problem and acceptance criteria. Agent must infer current code state and compute the delta.

**Transformation spec** (PR #132): Describes current state as a resource × exit path matrix, specifies the exact transformation, includes a consistency check matrix.

## Results

| Metric | Goal Spec (#131) | Transform Spec (#132) |
|---|---|---|
| Core fix correct | Yes (both paths) | Yes (both paths) |
| Cost | $0.40 | $0.34 (16% cheaper) |
| Test quality | Racy — PID file write races with idle timeout | Clean — approved by adversarial review |
| Comment style | `//nolint:errcheck` (linter suppression) | Plain explanatory comment |
| Review result | CHANGES_REQUESTED (flaky test) | APPROVED |

## Analysis

### Both agents got the core fix right
The `cmd.Wait()` addition is a small, well-defined change. Both specs were sufficient to guide it. The difference isn't in what was changed — it's in the quality of the surrounding code (tests, comments).

### Test quality is where specs diverge
The goal spec agent wrote a test that creates a subprocess, writes its PID to a file, then checks if the process was reaped. But the PID file write races with the 100ms idle timeout — on slow hardware (Pi), the process can be killed before writing its PID.

The transformation spec agent didn't have this problem because the spec's consistency matrix framed verification as "check the exit path behavior" rather than "check the process state."

### Cost reduction is modest but consistent
16% fewer tokens. The transformation spec gives the model less room to explore — it knows exactly what to change and where.

### Comment quality differs
Goal spec: `//nolint:errcheck` — suppresses a linter warning. This is a code smell.
Transform spec: `// reap killed process; ignore error (expected for killed process)` — explains the *why*.

The transformation spec's "ignore error" language came directly from the spec itself, which said "ignore error (expected for killed process)." The goal spec didn't mention error handling at all, so the agent reached for a linter annotation.

## Key insight

**The spec shapes the test, not just the code.** When the spec describes the transformation as a matrix (resource × path → must be released), the agent writes tests that verify matrix cells. When the spec describes a goal ("no zombie processes"), the agent writes tests that check for zombies directly — which is harder and produces race conditions.

## Implications for /write-spec

Transformation specs should include:
1. Current state (anchoring the model)
2. Exact delta (minimizing exploration)
3. Consistency check (framing verification as coverage, not behavior)
4. The consistency check becomes the test specification

The test the agent writes should verify the consistency matrix, not the behavioral goal. "Every exit path calls Wait()" is easier to test than "no zombie processes exist."
