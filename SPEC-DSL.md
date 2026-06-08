# Spec DSL v2

A lightweight specification language for agent task tickets. Designed to be token-efficient while preserving the invariant discipline that catches cross-component bugs.

## Function specs

```
FUNCTION name(params) -> return_type
  WHEN condition => result
  WHEN condition => result
  OTHERWISE => result
```

Explicit case enumeration guides the agent toward correct types and complete branching. Every WHEN case becomes a test case.

## Invariant verbs

Three verbs, three meanings:

```
ENFORCE invariant_name:
  property description
  Assumes: environmental preconditions this depends on
  Falsified by: what input/state would break this
  (* Agent MUST add code that guarantees this — guard clauses, clamps, validation *)

VERIFY invariant_name:
  property description
  Assumes: environmental preconditions this depends on
  Falsified by: what input/state would break this
  (* Agent MUST add tests that check this — no code enforcement needed *)

CHECK invariant_name:
  property description
  Assumes: environmental preconditions this depends on
  Falsified by: what input/state would break this
  (* Agent must add BOTH defensive code AND tests *)
```

### Assumes and Falsified by

Every invariant depends on preconditions the code does not control. The `Assumes` field
makes these explicit. The `Falsified by` field applies a pre-mortem lens — naming what
would break the invariant generates the adversarial test cases needed to cover it.

```
CHECK ArtifactResolutionOrder:
  Triage stage artifact overrides requirements for target-repos
  Assumes: enrichment list ordered by stage completion time
  Falsified by: enrichment list arriving in non-chronological order
```

Writing "Assumes: enrichment list ordered by stage completion time" immediately raises
the question: is that guaranteed? If not, the implementation must enforce ordering
explicitly rather than relying on iteration order. Each `Falsified by` scenario should
produce at least one test case.

### Why three verbs

In experiments, agents treated `INVARIANT` ambiguously — natural language agents enforced defensively in code, DSL agents treated them as verification properties. The ENFORCE/VERIFY/CHECK distinction tells the agent exactly what to produce:

- `ENFORCE CostNonNegative` → add `if cost < 0 { cost = 0 }` in the function
- `VERIFY PhaseFromResult` → add table-driven test covering all cases
- `CHECK NilSafety` → add both a nil guard AND a test for nil input

## Mapping specs

```
MAPPING name: KEY_TYPE -> (VALUE_TYPE, target)
  Type1: key1 -> field1, key2 -> field2, ...
  Type2: key3 -> field3, ...
```

For bulk field-to-field mappings (config overrides, serialization). The agent can verify completeness by counting entries against the source.

## Compact reference format

When CLAUDE.md defines conventions, tickets can reference by name:

```
FILES: step_running.go, step_running_test.go
BUILD: controller  (* expands to: cd agent-sandbox/controller && go build ./... && go test ./... *)
SCOPE: step_running*.go only
```

## Example: complete spec

```
FUNCTION parseAgentResult(data: map[string]string) -> (*AgentResult, error)
  WHEN "result.json" not in data => (nil, nil)
  WHEN json.Unmarshal fails => (nil, error)
  OTHERWISE => (parsed AgentResult, nil)

FUNCTION determinePhase(result, hasResult, podSucceeded, oomKilled,
                        retriesRemaining: int) -> Phase
  WHEN !hasResult AND podSucceeded => Succeeded
  WHEN !hasResult AND !podSucceeded => Failed
  WHEN result.Status in {"failure", "error"} => Failed
  WHEN result.Status == "max_turns" AND retriesRemaining > 0 => Continuable
  WHEN result.Status == "max_turns" AND retriesRemaining == 0 => Failed
  WHEN result.Status == "success" AND result.PRURL != "" => WaitingForReview
  WHEN result.Status == "success" AND result.PRURL == "" => Succeeded

ENFORCE CostNonNegative:
  r.CostUSD >= 0 — clamp negative values to 0 in parseAgentResult
  Assumes: CostUSD field is always present in result JSON
  Falsified by: result JSON missing CostUSD field entirely

VERIFY PhaseFromResult:
  table-driven test covering every WHEN case above
  Assumes: result.Status values are a closed set
  Falsified by: unknown status string not in any WHEN case

CHECK NilSafety:
  parseAgentResult(nil) and determinePhase(nil, ...) never panic
  Assumes: callers pass Go-typed nil, not zero-value structs
  Falsified by: empty map instead of nil, empty string fields

FILES: step_running.go, step_running_test.go
BUILD: controller
SCOPE: step_running*.go only
```

## Design principles

1. **ENFORCE vs VERIFY distinction** — the agent must know whether to write code or tests
2. **Explicit WHEN/THEN cases** — guides correct types and complete branching
3. **Token-efficient** — no Unicode symbols, no box diagrams, ASCII only
4. **Invariants are named** — can be referenced in reviews ("does this PR satisfy NilSafety?")
5. **Z analysis still valuable** — use Z post-hoc to find cross-component invariants the spec missed
