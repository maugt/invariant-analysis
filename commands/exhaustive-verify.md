Exhaustive enumeration and coverage verification for a PR or set of changed files. Takes a PR number or file paths as arguments.

This skill complements `/invariant-verify` (which checks named cross-component contracts). This skill forces exhaustive enumeration — listing all cases, paths, values, or states, then checking that every one is handled. Bugs hide where enumeration is incomplete.

## Steps

1. If a PR number is provided, run `gh pr diff <number>` to get the diff
2. If file paths are provided, read those files directly

For every function that was added or substantially modified, apply ALL of the following enumerations:

### Step 1: Exit paths

Enumerate every way the function can return:
- Normal return (end of function)
- Early returns (guard clauses, error checks)
- Deferred cleanup paths
- Panic/recover paths (if applicable)

Then for each resource acquired (process, file, lock, connection, timer, channel, goroutine): is it released on EVERY path?

### Step 2: Cases and branches

For every switch/case, if/else chain, or type assertion:
- List all possible values of the switched expression
- Are all values handled? Is the default case correct or a silent swallow?
- For enum types: does the switch cover every variant? What happens when a new variant is added?

### Step 3: Error handling consistency

For each error that can occur:
- Is it returned, logged, or swallowed?
- Is the handling consistent with how other errors in the same function are handled?
- If one error path logs + returns, but another silently swallows, flag it

### Step 4: Input domain

For each parameter:
- What happens with nil/zero/empty?
- What happens at boundary values (max int, empty string, empty slice, nil map)?
- What happens with negative values (for numerics)?
- Are these cases tested?

### Step 5: State mutation

If the function modifies shared state (struct fields, maps, external resources):
- Is the modification visible on ALL exit paths, or only some?
- Could a partial mutation leave the system in an inconsistent state?
- If step A succeeds and step B fails, is the state rolled back or left inconsistent?

### Step 6: Caller impact

If the function signature or behavior changed:
- List all callers (grep for the function name)
- Does each caller handle the new behavior correctly?
- Could a caller pass an input that the function doesn't expect?

## Output format

```
# Exhaustive Verification Report

## function_name (file:line)

### Enumerations
- Exit paths: N
- Switch/case branches: N (M values possible, K handled)
- Error sources: N
- Parameters: N

### Findings
- **VIOLATED**: [category] — description
  Counterexample: [specific input/condition]

- **GAP**: [category] — description
  Missing: [what's not covered]

### Clean
[only if genuinely clean after all enumerations]

## Summary
N functions analyzed. X findings across Y functions.
```

## Rules

- The technique is **enumerate then verify coverage**. List everything first, check handling second.
- Be concrete: every finding needs a specific counterexample or a specific missing case
- Don't review style or naming — only correctness
- If a function has >5 exit paths or >5 switch cases, it's worth extra scrutiny
- The most common bug pattern: N cases exist, N-1 are handled, the Nth is silently dropped
