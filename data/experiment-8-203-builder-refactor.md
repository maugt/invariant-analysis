# Experiment 8: SandboxBuilder Refactor A/B Test (VY-203)

## Setup

Unbiased A/B test on a non-trivial refactor. Neither spec author knew what bugs would appear.

- **Task**: Migrate 300+ line BuildSandbox function into SandboxBuilder methods
- **Complexity**: 3 files, 32 existing tests, cross-file changes (sandbox_template + step_pending)
- **Model**: Sonnet for both, $5 budget

## Specs

**A (Goal)**: "Move internals into builder methods, make independently testable"
**B (Transform)**: Current state table, step-by-step migration plan, postconditions, enumerations, prescribed builder pattern

## Results

| Metric | Goal (#134) | Transform (#135) |
|---|---|---|
| Cost | $1.36 | $0.74 (46% cheaper) |
| Diff size | 524 lines | 388 lines (26% smaller) |
| Public methods | 7 (stateless) | 4 (stateful, fluent) |
| Test coverage | 237 lines | 92 lines |
| OverrideEnvVar safety | Silent no-op on unknown var | Appends unknown var (safer) |
| Builds + passes | Yes | Yes |

## Bugs found by exhaustive verification

### In both:
- Zero timeout: resolvedTimeoutSeconds returns 0 when both task and config are zero
- No nil guard on task parameter in NewSandboxBuilder

### Goal spec only:
- OverrideEnvVar silently ignores unknown env var names — token injection can fail without error

### Transform spec only:
- Less test coverage (92 vs 237 lines)

## Adversarial review accuracy

Both reviews produced false positives on this larger refactor:
- #134: Said code changes were "missing" — they weren't
- #135: Said slice mutation after Build() — but code correctly sequences override before build

The adversarial prompt ("finding zero bugs is suspicious, try harder") causes the reviewer to hallucinate bugs on complex PRs where the diff is large and harder to reason about.

## Key findings

### Transform spec cost advantage scales with task size
- VY-200 (small fix): 16% cheaper
- VY-203 (medium refactor): 46% cheaper
- Hypothesis: larger tasks benefit more because the transform spec eliminates more exploration

### Transform spec produces better API design
The transform spec prescribed the fluent builder pattern with pre-Build overrides. The goal spec produced a stateless design with post-Build mutation — functionally correct but less safe (silent failures on unknown vars).

### Goal spec produces more tests
The goal spec agent wrote 237 lines of tests vs 92. The vague "make independently testable" criterion led the agent to write more granular tests. The transform spec's precise migration steps left less room for the agent to add test creativity.

### Adversarial review is unreliable on large PRs
Both reviews produced false positives. The prompt that works for 50-line PRs breaks on 400-line refactors — the reviewer can't hold the full diff in context and hallucinates problems.

### Exhaustive verification is reliable at all scales
Found the same bugs in both PRs (zero timeout, nil guard) plus the goal-specific OverrideEnvVar safety issue. No false positives.

## Implications

1. **Transform spec for implementation quality** — cheaper, smaller, safer API design
2. **Goal spec for test coverage** — the vagueness produces more creative tests
3. **Exhaustive verify > adversarial review for large PRs** — no false positives at any scale
4. **Review prompt needs size-awareness** — different prompt for <100 line vs >100 line diffs
