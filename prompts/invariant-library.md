# Standard Invariant Library

Named, reusable invariant patterns. Reference by name in ticket specs.

## BackwardCompat(oldFormat, newFormat)
The wire format (JSON, YAML, API response) must not change. Old inputs
produce identical outputs after refactoring.
- **Common violation**: Renaming JSON keys, changing field types, reordering
- **Check method**: Unmarshal old format into new struct, compare field values

## TotalCoverage(sourceSet, targetSet)
Every item in the source set has a corresponding entry in the target set.
No orphans, no gaps.
- **Common violation**: Adding a new config field but forgetting to add its env override
- **Check method**: Count items in both sets, verify equality

## NilSafety(function)
The function handles nil/empty/zero inputs without panicking.
All pointer dereferences are guarded.
- **Common violation**: Assuming a pointer is non-nil after a type assertion
- **Check method**: Call with nil for each pointer parameter, verify no panic

## BehaviorPreservation(oldFunc, newFunc)
For all valid inputs, the old and new function produce identical outputs.
Pure refactoring — no behavior change.
- **Common violation**: Changing default values, reordering conditions, dropping edge cases
- **Check method**: Table-driven test with same inputs, compare outputs

## RoundTrip(encode, decode)
Encoding then decoding (or vice versa) preserves all data. No information
loss in the conversion.
- **Common violation**: Serializing a float to a string with fixed precision, losing digits
- **Check method**: Encode→decode→compare with original

## NoFalsePositive(validator, validInputs)
Valid inputs must never be rejected. The validator may miss some invalid
inputs (false negatives are acceptable), but must never reject good ones.
- **Common violation**: Over-constraining validation (requiring checkboxes when header suffices)
- **Check method**: Feed known-valid inputs, verify all pass

## NoFalseNegative(validator, invalidInputs)
Invalid inputs must always be caught. Complementary to NoFalsePositive.
- **Common violation**: Only checking for section headers, not content
- **Check method**: Feed known-invalid inputs, verify all fail

## IdempotentOperation(operation)
Running the operation twice produces the same result as running it once.
Important for reconcile loops and retry logic.
- **Common violation**: Appending to a list on each call instead of checking existence
- **Check method**: Run twice, compare state after each run

## ParseValidateConsistency(parser, validator)
The parser and validator agree on what's valid. If the validator says OK,
the parser must extract meaningful content. If the parser returns empty,
the validator should have rejected.
- **Common violation**: Validator checks headers, parser checks content — gap between them
- **Check method**: Generate inputs at the boundary, verify both agree

## AtomicGuard(sideEffect, annotation)
A side effect (API call, child task creation, external post) that is
guarded by an annotation must not be repeated if the annotation write
fails. The guard and the effect must be effectively atomic.
- **Common violation**: Post comment → update annotation. If update fails
  (resource version conflict), next reconcile re-posts the comment.
- **Check method**: Trace the failure path: what happens if the side effect
  succeeds but the annotation persistence fails? Three safe patterns:
  1. Write annotation first (optimistic), perform effect, revert on failure
  2. Use retry-on-conflict for the annotation write
  3. Make the side effect itself idempotent (check if already done)

## TypePreservation(source, destination)
When converting between representations, type information is not lost.
Integers stay integers, booleans stay booleans.
- **Common violation**: Pre-computing `retriesRemaining > 0` to bool, losing the count
- **Check method**: Compare types in source and destination signatures
