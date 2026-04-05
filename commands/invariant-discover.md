Discover cross-component invariants for a task. Takes file paths as arguments.

## What to look for

Focus on **boundaries** — places where one component assumes something about another:

- Function A produces output that function B consumes — what format contract exists?
- A config value is read in one place and used in another — are both consistent?
- A state machine has transitions — can any transition lead to an invalid state?
- A field is set in one file and read in another — can it ever be nil/empty unexpectedly?
- An error in component A is classified by component B — do they agree on the taxonomy?

## What NOT to do

- Do not review code quality or suggest refactors
- Do not find bugs in the current code (only contracts that must be preserved)
- Do not state obvious invariants ("the function should return no error on valid input")
- Focus on invariants that span TWO OR MORE components/functions/files

## Steps

1. Read each file path provided as arguments
2. Identify cross-component invariants — properties that span TWO OR MORE functions/files/components
3. For each invariant, classify as ENFORCE (add guard code), VERIFY (add tests), or CHECK (both)
4. Flag any open decisions the spec doesn't address

## Choosing the verb

- **ENFORCE** when the property must be guaranteed by defensive code (nil checks, validation, clamping)
- **VERIFY** when the property should be checked by tests (behavioral correctness, edge cases)
- **CHECK** when both — the property is critical enough to need code guards AND test coverage

## Output format

For each invariant found:

```
ENFORCE InvariantName:
  property description
  why it matters (what breaks if violated)
  which components it spans

VERIFY InvariantName:
  ...

CHECK InvariantName:
  ...

DECISION: description of choice the spec doesn't make
  options with tradeoffs (not recommendations)
  which component is affected
```

Aim for 5-10 invariants, not 50. Prioritize the ones most likely to catch real bugs.

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
- **ParseValidateConsistency(parser, validator)** — parser and validator agree on what's valid
- **TypePreservation(source, destination)** — type information is not lost when converting between representations
