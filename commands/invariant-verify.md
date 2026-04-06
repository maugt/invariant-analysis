Verify named invariants against a PR or set of changed files. Takes a PR number or file paths as arguments.

## Invariant verbs

- **ENFORCE**: The code MUST contain defensive logic that guarantees this property. Look for guard clauses, validation, clamping, or error returns.
- **VERIFY**: Tests MUST exist that check this property. Look for table-driven tests covering the stated cases.
- **CHECK**: BOTH code enforcement AND tests must exist.

## Steps

1. If a PR number is provided, run `gh pr diff <number>` to get the diff
2. If file paths are provided, read those files directly
3. The user will provide invariants to check (or ask you to check all relevant ones)

For each invariant:
- **ENFORCE**: trace through the code — can you construct an input that violates the property? If yes, it's VIOLATED. Show the counterexample.
- **VERIFY**: search the test files for test cases covering the property. If missing, it's VIOLATED. Say which test is missing.
- **CHECK**: both of the above.

## Output format

```
# Invariant Verification Report

## ENFORCE InvariantName: **SATISFIED** or **VIOLATED**
Finding: one-line explanation
[If violated: concrete counterexample]

## VERIFY InvariantName: **SATISFIED** or **VIOLATED**
Finding: one-line explanation
[If violated: what test case is missing]

## Summary
X/Y invariants satisfied.
```

## Rules

- Be precise: if you say VIOLATED, provide a concrete counterexample
- If you can't determine status, say UNCERTAIN with explanation
- Do NOT review code style or architecture — only check invariants
- Do NOT approve or reject the PR — only report invariant status

## Standard invariant patterns

Reference these by name when they apply:

- **BackwardCompat(oldFormat, newFormat)** — wire format must not change; old inputs produce identical outputs after refactoring
- **TotalCoverage(sourceSet, targetSet)** — every item in source has a corresponding entry in target; no orphans, no gaps
- **NilSafety(function)** — handles nil/empty/zero inputs without panicking; all pointer dereferences guarded
- **BehaviorPreservation(oldFunc, newFunc)** — for all valid inputs, old and new function produce identical outputs
- **RoundTrip(encode, decode)** — encoding then decoding preserves all data; no information loss
- **NoFalsePositive(validator, validInputs)** — valid inputs must never be rejected
- **NoFalseNegative(validator, invalidInputs)** — invalid inputs must always be caught
- **IdempotentOperation(operation)** — running twice produces same result as running once
- **AtomicGuard(sideEffect, annotation)** — if a guarded side effect succeeds but the annotation write fails, the next reconcile must not repeat the effect
- **PathSymmetry(function, resource)** — every exit path through a function must handle resources consistently; if happy path cleans up but error path doesn't, it's violated
- **DataExposure(secretRef, persistedResource)** — values referenced by secretKeyRef or equivalent must not be converted to literal strings before persistence (CRD spec, etcd, logs)
- **ParseValidateConsistency(parser, validator)** — parser and validator agree on what's valid
- **TypePreservation(source, destination)** — type information is not lost when converting between representations
