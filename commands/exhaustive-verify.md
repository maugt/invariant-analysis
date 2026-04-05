Exhaustive path and resource verification for a PR or set of changed files. Takes a PR number or file paths as arguments.

This skill complements `/invariant-verify` (which checks named cross-component contracts). This skill does intra-function exhaustive analysis — tracing every code path to find asymmetries.

## Steps

1. If a PR number is provided, run `gh pr diff <number>` to get the diff
2. If file paths are provided, read those files directly

For every function that was added or substantially modified:

### Step 1: List all exit paths

Enumerate every way the function can return:
- Normal return (end of function)
- Early returns (guard clauses, error checks)
- Deferred cleanup paths
- Panic/recover paths (if applicable)

### Step 2: Resource tracking

For each resource acquired in the function (process, file handle, lock, connection, timer, channel, goroutine):
- Where is it acquired?
- On which exit paths is it released?
- On which exit paths is it NOT released?
- Is `defer` used? If not, why not?

### Step 3: Error handling symmetry

For each error that can occur:
- Is it returned, logged, or swallowed?
- Is the handling consistent with how other errors in the same function are handled?
- If one error path logs + returns, but another silently swallows, flag it

### Step 4: Input edge cases

For each parameter:
- What happens with nil/zero/empty?
- What happens with maximum values?
- What happens with negative values (for numerics)?
- Are these cases tested?

### Step 5: State mutation consistency

If the function modifies shared state (struct fields, maps, external resources):
- Is the modification visible on ALL exit paths, or only some?
- Could a partial mutation leave the system in an inconsistent state?
- Is the mutation order-dependent? (e.g., update annotation THEN post comment)

## Output format

```
# Exhaustive Verification Report

## function_name (file:line)

### Exit paths: N total
1. Normal return (line X)
2. Error return: condition (line Y)
3. ...

### Resources
| Resource | Acquired | Released on path 1 | Released on path 2 | ... |
|----------|----------|-------------------|-------------------|-----|
| cmd      | line X   | Yes (Wait)        | NO — zombie leak  | ... |

### Findings
- **PathSymmetry VIOLATED**: resource not released on path N
  Counterexample: [specific input/condition that triggers the leaky path]

- **InputEdge VIOLATED**: nil input causes panic at line X
  Counterexample: call function(nil) → panic

### No issues found
[only if genuinely clean after checking all paths]

## Summary
N functions analyzed. X findings across Y functions.
```

## Rules

- Be exhaustive: check EVERY exit path, not just the obvious ones
- Be concrete: every finding needs a specific counterexample
- Don't review style or naming — only correctness
- Focus on: resource leaks, path asymmetry, unhandled edge cases, inconsistent error handling
- If a function has >5 exit paths, it's worth extra scrutiny — complexity breeds asymmetry
