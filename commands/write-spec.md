Write a dispatchable specification for an agent task. Takes a description of what needs to be done as arguments.

Transform a rough task description into a structured, agent-ready specification. Use the codebase to fill in concrete file paths, existing patterns, and testable criteria.

## Steps

1. **Understand the task**: Read the arguments. If vague, ask one clarifying question — don't guess.

2. **Explore the codebase**: Find the files that need to change. Use Glob and Grep to locate:
   - Functions/types that will be modified
   - Test files that will need updating
   - Related code the agent should read for context
   - Configuration that must not be hardcoded

3. **Check scope**: If the task touches more than 5 files or requires changes across multiple modules, recommend decomposition. Large tasks burn tokens on incremental build failures. Suggest how to split.

4. **Identify invariants** (cross-component boundaries):
   - What does this component assume about the other's output?
   - Could a refactor break this assumption silently?
   - Name using ENFORCE/VERIFY/CHECK

5. **Select design patterns** that enforce constraints structurally:
   For each invariant or enumeration discovered, ask: **is there a pattern that makes this hold by construction?**
   
   If a pattern exists, prescribe it in the spec — the agent should implement the pattern, not a one-off fix. Common mappings:
   
   | Constraint | Pattern | Why |
   |---|---|---|
   | PathSymmetry (resource cleanup on all paths) | `defer` or extract `cleanup()` helper | Enforced by construction — can't forget a path |
   | AtomicGuard (side effect + annotation) | Extract `guardedEffect()` with retry-on-conflict | One call site, atomicity guaranteed |
   | TotalCoverage (all fields handled) | Reflection-based test | Catches drift automatically, no manual enumeration |
   | AnnotationConsistency (constants centralized) | Single const block, linter rule | Convention enforced by location |
   | BehaviorPreservation (same output after refactor) | Table-driven test with before/after | Regression caught mechanically |
   
   Check if the codebase already has an instance of the pattern. If yes, reference it ("follow the pattern in `step_review_dispatch.go`"). If no, the spec should introduce the pattern explicitly.
   
   Use `/design-patterns` to identify existing patterns in the affected code if unsure.

6. **Identify enumerations** (things the agent must list exhaustively before coding):
   For each modified function, ask: **what needs to be enumerated?**
   
   - **Exit paths × resources**: How many ways can this function return? What resources (processes, files, locks, connections, timers) does it acquire? Every exit path must handle every resource.
   - **Enum values × switch cases**: Does the function switch on a type/enum? List every value. What happens when a new value is added later?
   - **Config fields × coverage**: Are new fields added to a struct? List every place that struct is read — do all consumers handle the new field?
   - **Callers × new behavior**: Is the function signature or contract changing? List every caller. Do they all handle the new behavior?
   - **Error types × handling**: What errors can this function produce? Is each one returned, logged, or silently swallowed? Is handling consistent?
   - **State mutations × failure paths**: If step A succeeds and step B fails, is the state left consistent?
   
   The bug is always in case N when only N-1 are handled. If the spec forces the agent to list all N, it can't silently skip one.

7. **Write the spec** with these sections:

```markdown
## Files to modify
- `path/to/file.go` — what changes and why (use - or * list markers)

## Context files
- `path/to/related.go` — why the agent should read this

## Acceptance criteria
- [ ] Testable criterion 1
- [ ] Testable criterion 2
(Each must be verifiable by running tests or reading code — not aspirational)

## Build/test
```
exact commands to build and test
```

## Tests
- Specific test name or behavior to implement
- Edge cases to cover

## Invariants
ENFORCE InvariantName: property spanning components A and B
VERIFY InvariantName: property spanning components A and B

## Patterns
Use these design patterns to enforce constraints by construction:
- PatternName for constraintName — "follow the pattern in existing_file.go:functionName"
(If no existing pattern: describe the abstraction to introduce)

## Enumerations
Before implementing, the agent MUST list all cases for each enumeration below.
The implementation must handle every listed case — no silent gaps.

- Exit paths in functionName: [list them, or "agent must enumerate before coding"]
- Switch cases for TypeName: [list values, or "agent must list all variants"]
- Callers of changedFunction: [list them, or "agent must grep and list"]
- Config fields affected: [list them]

## DO NOT change
- Files or behaviors that must not be touched
- Guardrails to prevent scope creep

## Configuration
- Values that must come from config, not hardcoded
```

## Rules

- **File paths must exist**. Verify with Glob before listing. If a file doesn't exist and needs to be created, use `## Files to create/modify`.
- **Acceptance criteria must be testable**. "Works correctly" is not testable. "Returns error for nil input" is.
- **Build/test must be exact commands**. Not "run the tests" — the actual `cd foo && go test ./...` command.
- **DO NOT change must be specific**. Not "don't break anything" — name the specific files, functions, or behaviors.
- **Invariants check cross-component contracts**. Boundaries between files/modules where one component assumes something about another.
- **Enumerations force exhaustive coverage**. The bug is always in case N when only N-1 are handled. If the spec lists all N, the agent can't skip one. Apply to: exit paths, switch cases, config fields, callers, error types, state mutation steps.
- **Include the dangerous commands warning** if the task involves process execution, file watching, or streaming: agent must not use `tail -f`, `watch`, or other blocking commands.

## Anti-patterns to flag

If the task description has any of these, warn the user:
- **Mostly prose, no code structure** → needs more concrete specification
- **"Explore" or "investigate" language** → this is research, not a code task
- **No file paths** → agent will burn tokens exploring
- **Touches `.github/workflows/`** → agents can't push workflow files (GitHub PAT lacks workflow scope)
- **Single task touching 10+ files** → decompose into smaller tasks
- **"Make it configurable"** without specifying where config lives → specify the config layer

## Output

Present the spec as a single markdown block the user can paste into a Linear ticket. Do not create files or tickets — just output the spec.
