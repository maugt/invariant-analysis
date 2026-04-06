# Experiment 7: Full Pipeline Spec Rerun of VY-194

## Hypothesis

A transformation spec with postconditions-as-tests, enumerations, and prescribed patterns prevents all known bugs from the original agent run.

## Test case

VY-194: Add idle timeout for stuck Claude processes. Originally merged as PR #122 with 4 known bugs discovered by various review methods across the session.

## Method

Rerun from the pre-change commit (847fcd0) with a spec that incorporates every technique from the pipeline:

1. **Transformation** (current state matrix → exact delta)
2. **Enumerations** (all 4 exit paths × resources, all config validation cases)
3. **Postconditions as tests** (verify properties algebraically, not hand-computed values)
4. **Patterns** (defer for cleanup, exec sleep in tests)
5. **PathSymmetry invariant** (every Kill() must have a matching Wait())

The spec explicitly called out:
- The pre-existing ctx.Done() bug (Kill without Wait)
- The config validation requirement (reject zero/negative, don't silently fallback)
- The test pattern (use exec sleep, not sleep, to prevent orphan pipes)
- The exit path matrix with "Must be Yes" for every Kill→Wait cell

## Results

| Known bug | Original agent (goal spec) | Rerun (full pipeline spec) |
|---|---|---|
| cmd.Wait() missing after Kill() | Introduced | **Avoided** |
| Config validation (0/-5 silent fallback) | Introduced | **Avoided** |
| Flaky test timing (sleep 0.04 with 80ms timeout) | Introduced | **Avoided** |
| exec sleep missing in test scripts | Introduced | **Avoided** |

**4/4 bugs avoided.** Every bug the original run introduced was prevented by the full pipeline spec.

## Cost comparison

| Spec type | Cost | Bugs introduced | Bugs avoided |
|---|---|---|---|
| Original goal spec (PR #122) | $0.72 | 4 | 0/4 |
| Full pipeline spec (rerun) | ~$0.50 (sonnet, worktree) | 0 | 4/4 |

## What prevented each bug

| Bug | Spec technique that prevented it |
|---|---|
| cmd.Wait() missing | **Enumeration**: exit path matrix with "Must be Yes" for Wait column |
| Config validation | **Transformation**: explicit "reject zero and negative" vs ambiguous "default 10" |
| Flaky test timing | **Pattern**: "use exec sleep in test fake scripts" prescribed explicitly |
| exec sleep orphan | **Pattern**: same — pattern section is where codebase conventions live |

## Key insights

### The spec IS the bug prevention

Every bug was preventable at spec time. The original spec said "add idle timeout" — leaving 4 decisions to the agent, all of which it got wrong. The full pipeline spec made those 4 decisions explicitly:

1. "Kill → Wait on every path" (not "clean up properly")
2. "Reject zero/negative" (not "default to 10 minutes")
3. "exec sleep in tests" (not "write tests")
4. "Exit path matrix with verification" (not "handle all cases")

### Transformation spec anchoring matters

The current state matrix showed the agent that ctx.Done already had a bug (Kill without Wait). The original spec didn't mention the existing code at all — the agent copied the buggy pattern.

### Patterns encode codebase-specific knowledge

"Use exec sleep in test scripts" is not obvious from first principles. It's a lesson learned from a previous failure (VY-192 hung pod). The pattern section is where institutional knowledge lives — things the agent can't derive from the problem description.

### Postconditions prevent the most common test bug

Not tested in this case (the original VY-194 tests didn't have hand-computation errors), but the Verina experiment (experiment 6) showed that postconditions-as-tests eliminate the #1 source of test bugs.

## Implications

The full pipeline spec is strictly better than goal specs for tasks where we know the failure modes. The question is whether the pipeline can discover failure modes for NEW tasks where we don't have prior data. That's what the invariant discovery and exhaustive verification skills are for — they find the constraints that the spec should include.

The workflow:
1. `/write-spec` generates the initial spec (with invariants, enumerations, patterns)
2. `/invariant-discover` finds additional cross-component constraints
3. `/exhaustive-verify` finds additional intra-function path issues
4. Human reviews and adds codebase-specific patterns
5. Agent executes the spec
6. `/invariant-verify` + `/exhaustive-verify` check the output

Steps 2-3 replace the human knowledge that went into the rerun spec. If they work, the pipeline is self-improving — each bug found adds a pattern to the library, which prevents it in future specs.
