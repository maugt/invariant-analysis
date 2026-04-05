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
  (* Agent MUST add code that guarantees this — guard clauses, clamps, validation *)

VERIFY invariant_name:
  property description
  (* Agent MUST add tests that check this — no code enforcement needed *)

CHECK invariant_name:
  property description
  (* Agent must add BOTH defensive code AND tests *)
```

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

VERIFY PhaseFromResult:
  table-driven test covering every WHEN case above

CHECK NilSafety:
  parseAgentResult(nil) and determinePhase(nil, ...) never panic

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
